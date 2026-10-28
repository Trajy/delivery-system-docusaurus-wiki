# Documentação Técnica — Delivery System

Este repositório contém a documentação técnica do **Delivery System**, desenvolvida e publicada como um site estático com [Docusaurus](https://docusaurus.io/).

## Acesse a documentação

[Abrir a documentação publicada](https://delivery-system.trajy.com)

## Sobre esta documentação

A documentação tem como objetivo registrar as informações necessárias para compreender, desenvolver e operar o Delivery System. Ela reúne e pode evoluir para cobrir:

- **Arquitetura e modelagem:** estilo arquitetural, diagramas, context map, bounded contexts, serviços e modelos de domínio.
- **Decisões de arquitetura (ADRs):** contexto, alternativas consideradas, decisão tomada e suas consequências.
- **Tecnologias e infraestrutura:** ferramentas, frameworks, serviços e ambientes utilizados pelo sistema.
- **Fluxos do sistema:** jornadas e interações entre serviços, integrações e eventos.
- **Desenvolvimento e operação:** configuração do ambiente local, execução, build, publicação e orientações operacionais.

O documento inicial de arquitetura descreve a proposta baseada em microsserviços, arquitetura orientada a eventos, Domain-Driven Design, CQRS parcial, Saga e API Gateway, além dos bounded contexts do domínio. Consulte [Arquitetura e Modelagem](docs/arquitetura-e-modelagem/arquitetura-e-modelagem.md) para os detalhes disponíveis.

## Tecnologias da documentação

- **Docusaurus 3.10.2** para gerar e servir o site de documentação.
- **Node.js 20 ou superior** para desenvolvimento local; a pipeline de publicação utiliza **Node.js 22**.
- **npm** para instalar dependências e executar scripts.
- **GitHub Actions e GitHub Pages** para build e publicação contínua.

## Executar localmente

Pré-requisitos: [Node.js](https://nodejs.org/) 20 ou superior e npm.

No terminal, a partir da pasta deste projeto (`docusaurus`):

```bash
npm ci
npm run start
```

O site ficará disponível em [http://localhost:3000](http://localhost:3000). O servidor de desenvolvimento atualiza a página automaticamente quando os arquivos da documentação são alterados.

Para validar a versão de produção localmente:

```bash
npm run build
npm run serve
```

O comando `build` gera o site estático na pasta `build`; `serve` permite visualizar esse build localmente.

## Fluxo de commit e publicação

1. Faça as alterações na documentação ou no site e valide localmente com `npm run start` e, quando apropriado, `npm run build`.
2. Crie um commit e envie as alterações para o GitHub. O fluxo recomendado é abrir um pull request e, após a revisão, integrá-lo à branch `main`.
3. Um push para `main` inicia automaticamente o workflow [Deploy Docusaurus to GitHub Pages](.github/workflows/deploy.yml). Abrir ou atualizar um pull request, por si só, não aciona este workflow.
4. A etapa de build executa em Ubuntu: faz checkout do repositório, configura Node.js 22 com cache do npm, instala as dependências com `npm ci` e gera o site com `npm run build`.
5. Se o build for concluído, o workflow configura o GitHub Pages e envia a pasta `build` como artefato.
6. O job de deploy, dependente do build, publica o artefato no GitHub Pages. O workflow também pode ser iniciado manualmente pela opção `workflow_dispatch` na aba **Actions** do GitHub.

Se a compilação falhar, o job de deploy não é executado; consulte os logs do workflow na aba **Actions** antes de tentar novamente.
