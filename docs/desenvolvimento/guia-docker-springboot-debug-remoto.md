---
id: meu-documento-customizado
title: Configuração Ambiente Local Com Docker
sidebar_label: Configuração Ambiente Local Com Docker
sidebar_position: 2
description: desenvolver uma aplicação Java com Docker
tags:
  - debug
  - configuração de ambiente
  - Docker
  - Java
  - Maven
---

# Guia de Configuracao: aplicação Java/Spring Boot no Docker com PostgreSQL, hot reload e debug remoto

Este guia reúne um fluxo reutilizável para desenvolver uma aplicação
Java --- especialmente Spring Boot com Maven --- inteiramente em Docker,
com PostgreSQL em um serviço separado, sincronização de código durante o
desenvolvimento e depuração remota por JDWP.

Os exemplos usam Java 21, Maven Wrapper, Spring Boot, Docker Compose e
as portas `8080` (HTTP) e `5005` (debug). Ajuste versões, caminhos,
portas, nomes de serviço e variáveis para cada projeto.

## 1. Arquitetura e conceitos

-   **Container `app`:** executa Maven/Spring Boot e a aplicação.
-   **Container `postgres`:** executa o banco de dados PostgreSQL.
-   **Volume nomeado:** persiste os dados do PostgreSQL entre
    reinicializações.
-   **Spring profile:** seleciona configurações da aplicação, por
    exemplo `docker-local`.
-   **Maven profile:** seleciona dependências/plugins de build, por
    exemplo `docker-local`. É independente do Spring profile.
-   **Compose Watch:** sincroniza arquivos e pode executar comandos ou
    reconstruir a imagem quando arquivos mudam.
-   **JDWP:** permite que uma IDE conecte-se à JVM da aplicação para
    usar breakpoints, inspecionar variáveis e percorrer a execução.

> Segurança: publique a porta de debug somente no loopback do host
> (`127.0.0.1`). Não exponha JDWP diretamente à internet; uma conexão de
> debug pode dar controle significativo sobre a JVM.

## 2. Estrutura sugerida do projeto

Exemplo de estrutura:

``` text
meu-projeto/
├── .mvn/
├── mvnw
├── pom.xml
├── src/
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
└── .dockerignore
```

Os exemplos abaixo pressupõem que os comandos são executados na raiz do
projeto, onde estão `pom.xml`, `mvnw` e `src/`.

## 3. Prepare o Dockerfile de desenvolvimento

Crie `docker/Dockerfile`:

``` dockerfile
FROM maven:3.9-eclipse-temurin-21 AS dev

WORKDIR /workspace

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw && ./mvnw -B dependency:go-offline

COPY src/ src/

EXPOSE 8080
EXPOSE 5005

CMD ["./mvnw", "-B", "-Pdocker-local", "spring-boot:run"]
```

Pontos importantes:

-   `WORKDIR /workspace` define o diretório de trabalho dentro do
    container.
-   O Maven Wrapper e o `pom.xml` são copiados antes do código para
    aproveitar o cache das dependências.
-   `-Pdocker-local` ativa um **perfil Maven** chamado `docker-local`,
    caso ele esteja definido no `pom.xml`.
-   `EXPOSE` documenta portas usadas pela imagem, mas **não publica** as
    portas no host.
-   O Spring profile será ativado pelo Compose, usando
    `SPRING_PROFILES_ACTIVE=docker-local`.
-   Não adicione JDWP via `JAVA_TOOL_OPTIONS` neste cenário com
    `mvn spring-boot:run`; veja a seção de debug para evitar que Maven e
    a JVM da aplicação tentem abrir a mesma porta.

Se o projeto não tiver o perfil Maven `docker-local`, remova
`-Pdocker-local` ou crie o perfil correspondente no `pom.xml`.

## 4. Configure o Docker Compose

Crie `docker/docker-compose.yml`:

``` yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: devdb
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpassword
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devuser -d devdb"]
      interval: 5s
      timeout: 3s
      retries: 10

  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: dev
    environment:
      SPRING_PROFILES_ACTIVE: docker-local
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: devdb
      DB_USER: devuser
      DB_PASSWORD: devpassword
    ports:
      - "127.0.0.1:8080:8080"
      # Necessária somente quando for conectar uma IDE do host ao debugger:
      - "127.0.0.1:5005:5005"
    depends_on:
      postgres:
        condition: service_healthy
    develop:
      watch:
        - action: sync+exec
          path: ./src
          target: /workspace/src
          initial_sync: true
          exec:
            command: ./mvnw -q -DskipTests compile
            working_dir: /workspace
        - action: rebuild
          path: ./pom.xml
        - action: rebuild
          path: ./.mvn
        - action: rebuild
          path: ./mvnw

volumes:
  postgres_data:
```

### Como funciona a conexão com o banco

Dentro do Compose, a aplicação deve acessar o PostgreSQL pelo nome do
serviço, `postgres`, na porta `5432`. Não use `localhost` como host do
banco dentro do container da aplicação: ali, `localhost` aponta para o
próprio container `app`.

As variáveis `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER` e `DB_PASSWORD`
precisam corresponder aos placeholders da configuração Spring, por
exemplo:

``` yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:postgres}:${DB_PORT:5432}/${DB_NAME:devdb}
    username: ${DB_USER:devuser}
    password: ${DB_PASSWORD:devpassword}
```

Coloque essa configuração no arquivo de profile apropriado, por exemplo
`application-docker-local.yml`. Ajuste-a ao modelo real de configuração
do projeto.

**Credenciais:** os valores acima são apenas para desenvolvimento local.
Não use senhas de exemplo em ambientes compartilhados ou produção;
prefira secrets/variáveis seguras.

### Sobre `develop.watch`

-   `sync+exec` sincroniza os arquivos de `src/` para `/workspace/src` e
    executa a compilação.
-   Alterações no `pom.xml`, `.mvn/` ou `mvnw` acionam rebuild da
    imagem.
-   O watch não substitui a compilação nem garante que todas as
    alterações de recursos/configuração sejam recarregadas. Configure
    regras adicionais conforme as necessidades do projeto.
-   O comando `docker compose watch` pode ser usado para iniciar o modo
    de observação, se suportado pela versão instalada do Docker Compose.

## 5. Configure os perfis Maven e Spring

São duas configurações diferentes:

-   `-Pdocker-local` ativa o perfil **Maven**.
-   `SPRING_PROFILES_ACTIVE=docker-local` ativa o perfil **Spring**.

Uma configuração típica de dependências no `pom.xml` pode separar
H2/DevTools para desenvolvimento local e PostgreSQL para o ambiente em
Docker. Exemplo ilustrativo:

``` xml
<profiles>
  <profile>
    <id>local</id>
    <dependencies>
      <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
      </dependency>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
      </dependency>
    </dependencies>
  </profile>

  <profile>
    <id>docker-local</id>
    <dependencies>
      <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
      </dependency>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
      </dependency>
    </dependencies>
  </profile>
</profiles>
```

Esse fragmento é apenas um modelo: confirme as versões gerenciadas pelo
Spring Boot e a organização atual do `pom.xml`. Se o driver PostgreSQL
estiver declarado fora dos perfis, ele continuará fazendo parte do build
de todos eles; um perfil Maven não remove automaticamente uma
dependência global. Para separar drivers, declare-os nos perfis
apropriados.

## 6. Inicie a aplicação

Como o arquivo Compose fica dentro de `docker/`, execute na raiz do
projeto:

``` bash
docker compose --project-directory . -f docker/docker-compose.yml up --build
```

Para iniciar em modo de observação (Compose Watch), use:

``` bash
docker compose --project-directory . -f docker/docker-compose.yml watch
```

Se a versão do Compose não suportar `watch` ou a ação `sync+exec`,
atualize o Docker Compose para uma versão compatível ou adapte a
configuração às ações disponíveis.

Comandos úteis:

``` bash
# Ver serviços e estado
docker compose --project-directory . -f docker/docker-compose.yml ps

# Acompanhar logs
docker compose --project-directory . -f docker/docker-compose.yml logs -f app

# Acompanhar somente os logs do banco
docker compose --project-directory . -f docker/docker-compose.yml logs -f postgres

# Abrir um shell no container da aplicação
docker compose --project-directory . -f docker/docker-compose.yml exec app sh
```

## 7. Ative o debug remoto JDWP

Para depurar a aplicação que é iniciada com `mvn spring-boot:run`, passe
os argumentos JDWP **à JVM da aplicação**, não globalmente por
`JAVA_TOOL_OPTIONS`.

No `docker/Dockerfile`, use:

``` dockerfile
CMD ["./mvnw", "-B", "-Pdocker-local", "spring-boot:run", "-Dspring-boot.run.jvmArguments=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"]
```

Mantenha no Compose a publicação local da porta:

``` yaml
ports:
  - "127.0.0.1:8080:8080"
  - "127.0.0.1:5005:5005"
```

Reconstrua/recrie o serviço:

``` bash
docker compose --project-directory . -f docker/docker-compose.yml up --build
```

Procure nos logs uma mensagem parecida com
`Listening for transport dt_socket at address: 5005`. A redação exata
pode variar conforme a JVM.

### Significado dos parâmetros JDWP

-   `transport=dt_socket`: usa conexão TCP.
-   `server=y`: a JVM aguarda a conexão do debugger.
-   `suspend=n`: a aplicação não espera a IDE para começar.
-   `address=*:5005`: escuta na porta `5005` dentro do container.

Se quiser que a aplicação espere a IDE antes de executar o código de
inicialização, use `suspend=y` temporariamente. A aplicação ficará
parada até que o debugger conecte.

> Não configure simultaneamente JDWP em `JAVA_TOOL_OPTIONS` e em
> `spring-boot.run.jvmArguments`. Escolha uma estratégia. Com
> `spring-boot:run`, a segunda opção é a recomendada neste guia.

## 8. Configure a IDE para conectar ao debugger

Em todas as IDEs, os parâmetros essenciais são:

-   **Host:** `localhost` ou `127.0.0.1`
-   **Porta:** `5005`
-   **Tipo:** conexão remota Java/JDWP
-   **Fontes:** abra/importе o projeto local correspondente ao código
    que está no container. Os breakpoints precisam corresponder à versão
    compilada que está rodando.

### 8.1 Visual Studio Code

1.  Instale uma extensão de suporte Java, por exemplo **Extension Pack
    for Java**.
2.  Abra a raiz do projeto no VS Code.
3.  Crie ou edite `.vscode/launch.json`.
4.  Adicione uma configuração de attach:

``` json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Attach to Spring Boot in Docker",
      "request": "attach",
      "hostName": "localhost",
      "port": 5005
    }
  ]
}
```

5.  Inicie o container com JDWP habilitado.
6.  Na aba **Run and Debug**, selecione **Attach to Spring Boot in
    Docker** e inicie a conexão.
7.  Coloque um breakpoint em uma classe executada pela aplicação e faça
    uma requisição que passe por ela.

Se o VS Code não reconhecer `type: "java"`, confirme que a extensão de
suporte Java está instalada e ativa.

### 8.2 IntelliJ IDEA

1.  Abra o projeto no IntelliJ e aguarde a importação do Maven.
2.  Vá a **Run → Edit Configurations...**.
3.  Adicione uma configuração **Remote JVM Debug** (em algumas versões,
    chamada **Remote**).
4.  Configure:
    -   **Host:** `localhost`
    -   **Port:** `5005`
    -   **Debugger mode:** `Attach to remote JVM` / modo de conexão, se
        essa opção aparecer.
    -   **Module/classpath:** selecione o módulo do projeto quando
        solicitado.
5.  Salve e inicie a configuração de debug.
6.  Coloque um breakpoint e execute uma requisição que atinja aquele
    trecho.

Os nomes e a disposição dos campos podem variar entre versões do
IntelliJ IDEA.

### 8.3 Eclipse

1.  Importe o projeto como **Existing Maven Project**.
2.  Abra **Run → Debug Configurations...**.
3.  Selecione **Remote Java Application** e crie uma configuração.
4.  Selecione o projeto correto.
5.  Configure:
    -   **Connection Type:** `Standard (Socket Attach)`
    -   **Host:** `localhost`
    -   **Port:** `5005`
6.  Clique em **Debug**.
7.  Coloque um breakpoint e faça uma requisição à aplicação.

A configuração `Socket Attach` é a IDE que inicia a conexão com a JVM
que está aguardando na porta JDWP.

## 9. Problemas encontrados e como resolvê-los

### Erro A --- Docker não encontra `docker/docker`

**Sintoma:** erro semelhante a
`lstat .../identity/docker/docker: no such file or directory`.

**Causa:** o caminho do Compose e o contexto de build foram resolvidos a
partir de um diretório diferente do esperado. Como o Compose está dentro
da pasta `docker/`, usar caminhos relativos sem definir o diretório do
projeto pode fazer o Docker procurar uma pasta `docker/docker`.

**Solução recomendada:** execute da raiz do projeto:

``` bash
docker compose --project-directory . -f docker/docker-compose.yml up --build
```

E use:

``` yaml
build:
  context: .
  dockerfile: docker/Dockerfile
  target: dev
```

O `context: .` refere-se à raiz definida por `--project-directory`;
`dockerfile` aponta para o Dockerfile dentro dela.

### Erro B --- Alterar usuário/senha do PostgreSQL não surte efeito

**Sintoma:** a aplicação recebe erro de autenticação ou o banco parece
continuar usando credenciais antigas depois de alterar `POSTGRES_DB`,
`POSTGRES_USER` ou `POSTGRES_PASSWORD`.

**Causa:** a imagem oficial do PostgreSQL aplica essas variáveis para
inicializar um diretório de dados vazio. Se o volume já contém um banco
inicializado, alterar as variáveis não recria o usuário nem troca a
senha existente.

**Solução para um banco local descartável:** remova o container e o
volume e recrie o ambiente:

``` bash
docker compose --project-directory . -f docker/docker-compose.yml down --volumes --remove-orphans
docker compose --project-directory . -f docker/docker-compose.yml up --build
```

**Atenção:** `down --volumes` apaga os dados do banco guardados nos
volumes declarados pelo Compose. Faça backup antes se houver dados que
precisem ser preservados. Em ambientes persistentes, altere as
credenciais com comandos SQL em vez de apagar o volume.

### Erro C --- JDWP: `Address already in use` / `bind failed`

**Sintoma:**

``` text
Picked up JAVA_TOOL_OPTIONS: -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
ERROR: transport error 202: bind failed: Address already in use
JDWP Transport dt_socket failed to initialize
```

**Causa provável neste cenário:** `JAVA_TOOL_OPTIONS` é aplicado
automaticamente a cada processo Java. Com `mvn spring-boot:run`, o
agente JDWP pode ser aplicado tanto à JVM do Maven quanto à JVM separada
da aplicação. As duas tentam escutar na mesma porta `5005`, provocando o
conflito.

Remover somente o mapeamento `5005:5005` do Compose não resolve esse
conflito: o mapeamento publica a porta no host, enquanto o erro acontece
ao abrir a porta dentro do ambiente da JVM/container.

**Solução:** retire `JAVA_TOOL_OPTIONS` do Compose e passe o agente JDWP
somente para a JVM da aplicação:

``` dockerfile
CMD ["./mvnw", "-B", "-Pdocker-local", "spring-boot:run", "-Dspring-boot.run.jvmArguments=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"]
```

Se a IDE roda no host, mantenha a publicação:

``` yaml
- "127.0.0.1:5005:5005"
```

Se ainda ocorrer o erro, confira se outro processo dentro do container
está usando a porta e verifique se JDWP não foi configurado em mais de
um lugar.

### Erro D --- A aplicação funciona, mas a IDE não conecta

Verifique, nesta ordem:

1.  O processo da aplicação iniciou com JDWP e os logs indicam que a
    porta está escutando.
2.  A porta `5005` está publicada em `ports` no serviço `app`.
3.  A aplicação foi recriada depois da alteração do Dockerfile/Compose.
4.  A IDE conecta a `localhost:5005`, não à porta `8080`.
5.  O debugger está em modo **Attach**, não em modo de iniciar uma nova
    JVM.
6.  Não há firewall, política corporativa ou outro processo interferindo
    na conexão.
7.  A IDE abriu o mesmo código-fonte/versão compilada que está dentro do
    container.

### Erro E --- O breakpoint não é atingido ou aparece como não verificado

Causas comuns:

-   O fluxo de execução ainda não passou pela linha.
-   A classe foi compilada antes da última alteração.
-   O container está executando uma versão diferente do código aberto na
    IDE.
-   O arquivo não foi sincronizado ou o build falhou.
-   O breakpoint está em código que não está sendo carregado pela
    aplicação.

**Solução:** confira os logs do Compose Watch, confirme que a compilação
terminou sem erros, reconstrua/reinicie quando necessário e repita a
requisição que deveria atingir o breakpoint. Verifique também se o
projeto correto está selecionado na configuração de debug da IDE.

### Erro F --- `EXPOSE 5005` foi configurado, mas a conexão não funciona

**Causa:** `EXPOSE` no Dockerfile é apenas metadado; ele não publica a
porta no host.

**Solução:** declare o mapeamento em `ports` no Compose:

``` yaml
ports:
  - "127.0.0.1:5005:5005"
```

## 10. Checklist final

Antes de iniciar uma sessão de debug, confirme:

-   [ ] O comando Compose é executado com o diretório do projeto
    correto.
-   [ ] O serviço `postgres` está saudável.
-   [ ] A aplicação usa o host `postgres` para acessar o banco dentro da
    rede Docker.
-   [ ] O Spring profile e o Maven profile foram configurados
    separadamente e de forma intencional.
-   [ ] O serviço `app` está com JDWP configurado somente na JVM da
    aplicação.
-   [ ] `JAVA_TOOL_OPTIONS` não está injetando JDWP ao mesmo tempo.
-   [ ] A porta `5005` está publicada em `127.0.0.1` se a IDE conecta a
    partir do host.
-   [ ] A IDE está em modo Attach para `localhost:5005`.
-   [ ] O código-fonte local corresponde ao código compilado que está
    rodando no container.
-   [ ] O volume do banco só foi apagado quando a perda dos dados era
    aceitável.

## 11. Comandos de referência

``` bash
# Subir e construir
docker compose --project-directory . -f docker/docker-compose.yml up --build

# Modo Compose Watch
docker compose --project-directory . -f docker/docker-compose.yml watch

# Ver status
docker compose --project-directory . -f docker/docker-compose.yml ps

# Logs da aplicação
docker compose --project-directory . -f docker/docker-compose.yml logs -f app

# Parar containers mantendo o volume
docker compose --project-directory . -f docker/docker-compose.yml down

# Apagar containers e volumes (APAGA OS DADOS DO BANCO)
docker compose --project-directory . -f docker/docker-compose.yml down --volumes --remove-orphans
```

------------------------------------------------------------------------

Este documento é um ponto de partida genérico. Ajuste nomes de serviço,
portas, caminhos, profiles, propriedades Spring e versão de Docker
Compose conforme o projeto. Para produção, use imagens e configurações
específicas de produção; não habilite JDWP publicamente.
