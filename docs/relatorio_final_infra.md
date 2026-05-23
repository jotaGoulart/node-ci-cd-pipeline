# Pipeline CI/CD
**TRILHA DE INFRAESTRUTURA — COMP JÚNIOR — 2026.1**
Desenvolvido por Juliano Vieira Goulart

---

## O projeto

Para o desafio da trilha de infraestrutura, implementei uma pipeline CI/CD completa usando GitHub Actions, Docker e Render. A aplicação escolhida foi uma API REST em Node.js que já vinha com testes automatizados escritos com Jest, o que permitiu focar totalmente na construção da pipeline.

O repositório está disponível em [github.com/JotaGoulart/node-ci-cd-pipeline](https://github.com/JotaGoulart/node-ci-cd-pipeline). A aplicação está no ar em [https://node-ci-cd-pipeline.onrender.com](https://node-ci-cd-pipeline.onrender.com).

---

## Como foi desenvolvido

O ambiente de desenvolvimento foi o WSL (Ubuntu no Windows). O primeiro passo foi configurar o Git com autenticação SSH para o GitHub, pois a autenticação padrão via HTTPS não funcionou como esperado. Esse foi o primeiro obstáculo: gerei as chaves com ssh-keygen, adicionei a chave pública no GitHub e alterei o remote do repositório para SSH. Depois disso o push passou a funcionar normalmente.

Com o ambiente configurado, clonei o projeto, rodei `npm install` e confirmei que os testes passavam com `npm test`. A partir daí, o desenvolvimento da pipeline começou de verdade.

Utilizei um padrão na formatação dos commits para o GitHub, fornecido na Athena da Comp.

### Containerizando com Docker

Antes de criar a pipeline, precisei containerizar a aplicação com Docker.

Criei o Dockerfile definindo a imagem base como Node.js 18 Alpine (uma versão leve do Linux), o diretório de trabalho, a instalação das dependências e o comando de inicialização. Um detalhe importante: o arquivo principal da aplicação não é o `app.js`, e sim o `bin/www.js`, que é quem de fato sobe o servidor HTTP. Descobri isso só depois de o container não responder na primeira tentativa.

Criei também o `.dockerignore` para evitar que a pasta `node_modules` fosse copiada pro contexto de build. O impacto foi visível: o contexto caiu de 42MB para menos de 1MB, tornando o build muito mais rápido. Testei o container localmente, confirmei que a API respondia na porta 3000 e só então avancei.

### Configurando o Render

O Render é a plataforma onde a aplicação está hospedada. Conectei o repositório do GitHub, e o Render detectou o Dockerfile automaticamente. Fiz o primeiro deploy manualmente para confirmar que tudo funcionava antes de automatizar.

Para integrar o Render com o GitHub Actions, precisei de duas informações: a API Key da minha conta e o Service ID do serviço criado. Ambas foram adicionadas como `secrets` no repositório do GitHub para que a pipeline pudesse usá-las sem expor as credenciais no código.

### Criando a pipeline

A pipeline está definida no arquivo `.github/workflows/ci.yml`. Ela é acionada automaticamente a cada push na branch main e tem dois jobs: o de CI e o de CD.

O job de CI roda numa máquina virtual Ubuntu provisionada pelo próprio GitHub. Ele faz o checkout do código, instala o Node.js 18, instala as dependências com `npm ci` e executa os testes. O `npm ci` foi escolhido no lugar do `npm install` por ser mais rígido: ele reinstala as dependências do zero seguindo exatamente o `package-lock.json`, garantindo reprodutibilidade.

O job de CD só executa se o job de CI passar. Essa dependência é declarada com a propriedade `needs` no arquivo yml. O deploy em si é uma chamada à API do Render via curl, usando as credenciais armazenadas nos secrets. Além disso, o deploy só acontece em push direto na main, nunca em pull requests.

---

## Como a pipeline funciona

O fluxo completo começa com um push na branch main. O GitHub Actions detecta o evento e inicia a pipeline. O job de CI instala as dependências e roda os testes automatizados. Se algum teste falhar, a pipeline para ali mesmo e o deploy não acontece. Se todos passarem, o job de CD é acionado e faz uma chamada à API do Render. O Render então builda a imagem Docker e sobe a nova versão da aplicação automaticamente.

Isso garante que só vai pra produção o código que passou pelos testes. O processo inteiro, do push ao deploy, acontece sem nenhuma intervenção manual.

---

## Aprimorando o projeto

Com a pipeline funcionando e atendendo ao escopo do desafio, decidi ir além e adicionar três melhorias: análise de qualidade de código com SonarCloud, publicação automática da imagem no Docker Hub e um badge de status no README.

### SonarCloud

O SonarCloud é a versão em nuvem do SonarQube, citado no conteúdo do curso. Ele analisa o código em busca de bugs, vulnerabilidades e problemas de manutenção, e é gratuito para repositórios públicos. Integrei como um step do job de CI, rodando logo após os testes.

Configurei também a cobertura de testes: o Jest gera um relatório no formato lcov que é enviado automaticamente ao SonarCloud. O dashboard mostra 47.9% de cobertura, além das métricas de segurança, confiabilidade e manutenção.

O Quality Gate aparece como falho por conta de problemas que já existiam no código original. Optei por não bloquear o deploy por isso, pois o código é de terceiros e travar a pipeline impediria qualquer evolução do projeto. A decisão foi consciente: o SonarCloud está sendo usado como ferramenta de visibilidade, identificando os problemas, e não como bloqueador.

### Docker Hub

Adicionei um job dedicado para publicar a imagem Docker automaticamente no Docker Hub a cada push na main. A imagem é publicada com duas tags: `latest`, que sempre aponta para a versão mais recente, e o hash do commit, que funciona como versão imutável. Isso significa que qualquer pessoa consegue rodar a aplicação com um simples `docker pull`, sem precisar clonar o repositório.

O job do Docker Hub roda após os testes e antes do deploy no Render, garantindo que só imagens de código testado são publicadas. A imagem está disponível em [hub.docker.com/r/jotagoulart/node-ci-cd-pipeline](https://hub.docker.com/r/jotagoulart/node-ci-cd-pipeline).

### Badge de status no README

Adicionei um badge de status da pipeline no README do repositório. É uma imagem gerada automaticamente pelo GitHub que mostra em tempo real se a pipeline está passando ou falhando. É um padrão comum em projetos open source e deixa claro, logo na página principal do repositório, que o projeto tem CI/CD configurado.

---

## Dificuldades ao longo do caminho

A maior dificuldade foi entender como a pipeline funcionava como um todo. No início eu não tinha clareza sobre a relação entre os jobs, o papel dos secrets e como o GitHub Actions se comunicava com o Render. O que ajudou foi construir de forma incremental: primeiro apenas o job de CI, verificar que estava verde, e só depois adicionar o CD. Ver cada parte funcionando separadamente antes de juntar tudo tornou o processo muito mais claro.

Outro problema foi a configuração do SSH com o GitHub. A autenticação padrão via HTTPS não funcionou no WSL, e precisei entender e configurar SSH do zero. Além disso, o Docker Desktop já estava instalado no Windows (pois precisei utilizá-lo anteriormente para realizar a trilha de QA), mas não estava integrado ao WSL. A correção foi simples depois que entendi o problema: nas configurações do Docker Desktop, ativei a integração com o Ubuntu.

Por fim, o container não respondia na primeira tentativa porque o Dockerfile apontava pro arquivo errado. Investigando o código, percebi que o projeto segue o padrão do Express Generator, onde o `app.js` só configura o Express e o `bin/www.js` é quem inicia o servidor. Corrigi o Dockerfile e o container passou a funcionar normalmente.
