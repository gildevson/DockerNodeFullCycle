# hello-express

Aplicação Node.js com Express containerizada via Docker.

## Stack

- Node.js 22
- Express 5
- Docker

## Estrutura do Projeto

```
DockerNode/
├── Node/                     # Aplicação Node.js
│   ├── dockerfile            # Imagem de desenvolvimento
│   ├── dockerfile.prod       # Imagem de produção (multi-stage)
│   ├── index.js              # Servidor Express (porta 3000)
│   ├── package.json          # Dependências npm
│   ├── .gitignore            # Arquivos ignorados pelo Git
│   └── .dockerignore         # Arquivos ignorados pelo Docker
├── nginx/                    # Configuração do proxy reverso
│   ├── dockerfile.prod       # Imagem do Nginx
│   └── nginx.conf            # Regras de proxy para o Node
├── mysql/                    # Dados persistidos do banco (gerado pelo Docker)
│   ├── ibdata1               # Dados do InnoDB
│   ├── ib_logfile0/1         # Logs de transação
│   └── nodedb/               # Banco nodedb (tabelas criadas)
├── docker-compose.yml        # Orquestração desenvolvimento
├── docker-compose.prod.yml   # Orquestração produção (com MySQL)
├── .gitignore                # Ignora mysql/ e .claude/
└── README.md                 # Esta documentação
```

---

## Guia por pasta

### Pasta [Node/](Node/)

Contém toda a aplicação Express.

| Arquivo | Caminho | O que faz |
|---|---|---|
| Servidor | [Node/index.js](Node/index.js) | Aplicação Express — rotas e conexão com MySQL |
| Dockerfile dev | [Node/dockerfile](Node/dockerfile) | Imagem com Node:22 + dockerize instalado |
| Dockerfile prod | [Node/dockerfile.prod](Node/dockerfile.prod) | Multi-stage: builder (node:22) + final (alpine) |
| Dependências | [Node/package.json](Node/package.json) | Lista pacotes: express, mysql |
| Ignorados Docker | [Node/.dockerignore](Node/.dockerignore) | Exclui `node_modules/` do build |

**Como instalar pacotes:**
```bash
cd C:\fullCycle\DockerNode\Node
npm install <pacote>
```

**Como entrar no container Node:**
```bash
docker exec -it node sh
```

**Ver logs da aplicação:**
```bash
docker logs -f node
```

---

### Pasta [nginx/](nginx/)

Configura o proxy reverso que fica na frente do Node.

| Arquivo | Caminho | O que faz |
|---|---|---|
| Dockerfile | [nginx/dockerfile.prod](nginx/dockerfile.prod) | Imagem nginx:alpine com conf personalizada |
| Configuração | [nginx/nginx.conf](nginx/nginx.conf) | Redireciona porta 80 → Node:3000 |

**Como funciona o nginx.conf:**
```nginx
server {
    listen 80;
    location / {
        proxy_pass http://node:3000;  # redireciona para o container node
    }
}
```

**Como entrar no container Nginx:**
```bash
docker exec -it nginx sh
```

**Ver logs do Nginx:**
```bash
docker logs -f nginx
```

---

### Pasta [mysql/](mysql/)

Criada automaticamente pelo Docker para persistir os dados do banco. **Não edite os arquivos manualmente.**

| Arquivo/Pasta | O que contém |
|---|---|
| `ibdata1` | Dados principais do InnoDB |
| `ib_logfile0`, `ib_logfile1` | Logs de transação do MySQL |
| `auto.cnf` | UUID único do servidor MySQL |
| `nodedb/` | Pasta do banco `nodedb` com as tabelas |
| `*.pem` | Certificados SSL do MySQL |

**Esta pasta:**
- Está no [.gitignore](.gitignore) — não vai para o GitHub
- Sobrevive ao `docker compose down` — dados são mantidos
- Para resetar o banco, delete a pasta e suba novamente:

```bash
docker compose -f docker-compose.prod.yml down
rm -rf ./mysql
docker compose -f docker-compose.prod.yml up -d --build
```

**Como entrar no banco:**
```bash
docker exec -it db bash
mysql -uroot -proot
use nodedb;
show tables;
```

## Dockerfile

```dockerfile
FROM node:22
WORKDIR /usr/src/app
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

### Instruções explicadas

| Instrução | O que faz |
|---|---|
| `FROM` | Define a imagem base (sistema + Node já instalado) |
| `WORKDIR` | Define a pasta de trabalho dentro do container |
| `COPY` | Copia arquivos da máquina para dentro do container |
| `RUN` | Executa um comando durante o build da imagem |
| `EXPOSE` | Documenta qual porta o container usa |
| `CMD` | Comando executado quando o container inicia |

## Como usar

### Desenvolvimento local (com volume)

Roda o projeto montando o diretório local no container — alterações nos arquivos refletem imediatamente:

```powershell
docker run --rm -it -v ${PWD}:/usr/src/app -p 3000:3000 node:22 bash
```

Dentro do container:
```bash
npm install
node index.js
```

### Build da imagem

```bash
docker build -t gildevso/hello-express .
```

### Rodar o container

```bash
docker run -p 3000:3000 gildevso/hello-express
```

Acesse: [http://localhost:3000](http://localhost:3000)

### Publicar no Docker Hub

```bash
docker push gildevso/hello-express
```

Imagem publicada em: `docker.io/gildevso/hello-express:latest`

## Endpoint

| Método | Rota | Resposta        |
|--------|------|-----------------|
| GET    | `/`  | `Hello World!`  |

## Multistage Building

Técnica para reduzir o tamanho da imagem final usando dois estágios no mesmo Dockerfile.

### O problema

Sem multistage, a imagem carrega tudo — ferramentas de build, cache do npm, arquivos desnecessários:

```
node:22 com npm install → ~1.6GB
```

### A solução

```dockerfile
# Stage 1 — "cozinha": monta o projeto
FROM node:22 AS builder
RUN npm install

# Stage 2 — "caixa final": só o que precisa rodar
FROM node:22-alpine
COPY --from=builder ...
```

O Stage 2 copia apenas o resultado do Stage 1 — a "cozinha" é descartada.

### Comparação das imagens base

| | `node:22` | `node:22-alpine` |
|---|---|---|
| Linux base | Debian completo | Alpine (mínimo) |
| Tamanho | ~1.6GB | ~60MB |
| Ferramentas | muitas | só o essencial |

### Build da imagem de produção

```bash
docker build -t gildevso/node:prod -f dockerfile.prod .
```

**Resumo:** a imagem final não carrega a "cozinha" — só o "bolo pronto" em cima de um Linux minúsculo.

## Nginx

Nginx é um servidor web que recebe requisições HTTP e decide o que fazer com elas.

### PHP vs Node.js

Com PHP, o nginx é obrigatório como intermediário:
```
Usuário → nginx (porta 80) → PHP-FPM (porta 9000) → aplicação
```

Com Node.js, o Express já é o servidor — nginx não é necessário:
```
Usuário → Express (porta 3000) → aplicação
```

### Quando usar nginx com Node.js

Em produção, o nginx ainda é útil para:

| Função | Descrição |
|---|---|
| Proxy reverso | Redireciona porta 80 → 3000 |
| SSL/HTTPS | Gerencia certificados |
| Load balancer | Distribui entre vários containers |
| Arquivos estáticos | Serve imagens, CSS, JS sem passar pelo Node |

## Erros comuns

| Erro | Causa | Solução |
|---|---|---|
| `requires 1 argument` | Faltou o `.` no build | `docker build -t nome .` |
| `invalid reference format` | Volume mal formatado | Usar `${PWD}:/usr/src/app` |
| `unknown instruction: FROM:` | Dois pontos após FROM | Usar `FROM node:22` sem `:` |
| `grep não reconhecido` | grep não existe no PowerShell | Usar `Select-String` |

---

## Docker Compose

O Docker Compose orquestra múltiplos containers ao mesmo tempo com um único comando, substituindo vários `docker run` manuais.

### Estrutura dos arquivos

```
docker-compose.yml       → ambiente de desenvolvimento
docker-compose.prod.yml  → ambiente de produção
```

---

### docker-compose.yml (Desenvolvimento)

```yaml
version: '3'

services:
  node:
    build:
      context: ./Node       # pasta onde está o Dockerfile
      dockerfile: dockerfile # usa o dockerfile de dev
    volumes:
      - ./Node:/usr/src/app         # espelha código local no container (hot reload)
      - /usr/src/app/node_modules   # preserva node_modules instalado no container
    networks:
      - node-network

  nginx:
    build:
      context: ./nginx
      dockerfile: dockerfile.prod
    ports:
      - "8080:80"   # porta do host : porta do container
    depends_on:
      - node         # nginx só sobe depois do node
    networks:
      - node-network

networks:
  node-network:
    driver: bridge  # rede isolada entre os containers
```

#### O que cada instrução faz

| Instrução | O que faz |
|---|---|
| `services` | Define os containers que serão criados |
| `build.context` | Pasta onde o Docker procura o Dockerfile e os arquivos |
| `build.dockerfile` | Nome do Dockerfile a usar |
| `volumes: ./Node:/usr/src/app` | Código local espelhado no container — alterações refletem em tempo real |
| `volumes: /usr/src/app/node_modules` | Volume anônimo que protege o `node_modules` do container de ser sobrescrito |
| `ports: "8080:80"` | Expõe a porta 80 do container como 8080 no host — acesse via `localhost:8080` |
| `depends_on` | Garante a ordem de inicialização dos serviços |
| `networks` | Rede interna compartilhada — containers se comunicam pelo nome do serviço como DNS |

---

### MySQL no Docker

#### Como funciona

O MySQL roda como um container isolado. Na primeira vez que sobe, ele:

1. Lê as variáveis de ambiente (`MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`)
2. Inicializa o banco de dados em `/var/lib/mysql` dentro do container
3. Cria o banco `nodedb` automaticamente
4. Fica disponível na porta `3306` para outros containers da mesma rede

#### Como conectar a partir do Node.js

```js
// O host é o nome do serviço no Compose, não localhost
const connection = mysql.createConnection({
  host: 'db',       // nome do serviço no docker-compose
  user: 'root',
  password: 'root',
  database: 'nodedb'
});
```

#### Vantagens de rodar MySQL no Docker

| Vantagem | Explicação |
|---|---|
| **Sem instalação local** | Não precisa instalar MySQL na máquina — roda isolado no container |
| **Versão controlada** | `mysql:5.7` garante que todos do time usam a mesma versão |
| **Ambiente idêntico** | Dev e prod rodam o mesmo banco, eliminando "funciona na minha máquina" |
| **Dados persistidos** | Volume `./mysql:/var/lib/mysql` mantém os dados após `docker compose down` |
| **Fácil de resetar** | Deletar a pasta `./mysql` e subir novamente recria o banco do zero |
| **Isolamento** | O banco não conflita com outros projetos ou com um MySQL local instalado |
| **restart: always** | Se o banco cair, reinicia sozinho sem intervenção manual |

#### Acessar o MySQL pelo terminal

**Passo 1 — Entrar no container:**

```bash
docker exec -it db bash
```

**Passo 2 — Conectar ao MySQL:**

```bash
mysql -uroot -proot
```

> A senha deve vir **sem espaço** após `-p`. `mysql -uroot -p root` (com espaço) não funciona — o MySQL ignora a senha e dá `Access denied`.

Quando conectado, o prompt muda para `mysql>`:

```
Welcome to the MySQL monitor. Commands end with ; or \g.
mysql>
```

---

**Comandos úteis dentro do MySQL:**

```sql
-- Ver todos os bancos de dados
show databases;

-- Entrar no banco nodedb
use nodedb;

-- Ver as tabelas do banco atual
show tables;

-- Sair do MySQL
exit
```

---

**Resetar o banco do zero** (apaga todos os dados):

```bash
docker compose -f docker-compose.prod.yml down
rm -rf ./mysql
docker compose -f docker-compose.prod.yml up -d --build
```

> A pasta `./mysql` guarda os dados do banco na sua máquina. Deletá-la faz o MySQL recriar tudo na próxima inicialização.

---

### docker-compose.prod.yml (Produção)

Arquivo completo com **3 serviços**: banco de dados MySQL, aplicação Node.js e proxy Nginx.

```yaml
version: '3'

services:

  db:
    image: mysql:5.7
    command: --innodb-use-native-aio=0
    container_name: db
    restart: always
    tty: true
    volumes:
      - ./mysql:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: nodedb
    networks:
      - node-network

  node:
    build:
      context: ./Node
      dockerfile: dockerfile.prod
    image: gildevso/node:prod
    container_name: node
    depends_on:
      - db
    networks:
      - node-network

  nginx:
    build:
      context: ./nginx
      dockerfile: dockerfile.prod
    image: gildevso/nginx:prod
    container_name: nginx
    ports:
      - "8080:80"
    depends_on:
      - node
    networks:
      - node-network

networks:
  node-network:
    driver: bridge
```

---

#### Serviço `db` — MySQL 5.7

```yaml
db:
  image: mysql:5.7
```
Usa a imagem oficial do MySQL versão 5.7 direto do Docker Hub — não precisa de Dockerfile, o Docker baixa pronto.

---

```yaml
  command: --innodb-use-native-aio=0
```
Desativa o **AIO (Asynchronous I/O nativo)** do InnoDB. Necessário em ambientes Windows e WSL onde o sistema de arquivos não suporta AIO nativo do Linux — sem isso o MySQL trava na inicialização.

---

```yaml
  container_name: db
```
Define um nome fixo para o container (`db`). Sem isso, o Compose gera um nome automático como `dockernode-db-1`. Com nome fixo, você pode usar `docker logs db` ou `docker exec -it db bash` diretamente.

---

```yaml
  restart: always
```
Reinicia o container automaticamente se ele cair por qualquer motivo — crash, erro, reinício da máquina. Essencial para banco de dados em produção.

---

```yaml
  tty: true
```
Mantém um terminal virtual alocado para o container. Necessário para o MySQL 5.7 permanecer rodando sem encerrar o processo principal.

---

```yaml
  volumes:
    - ./mysql:/var/lib/mysql
```
Persiste os dados do banco na pasta `./mysql` do projeto. Sem isso, ao rodar `docker compose down`, **todos os dados são perdidos**. Com o volume, os dados sobrevivem mesmo após remover o container.

```
./mysql (sua máquina) ←→ /var/lib/mysql (dentro do container MySQL)
```

---

```yaml
  environment:
    MYSQL_ROOT_PASSWORD: root
    MYSQL_DATABASE: nodedb
```
Variáveis de ambiente que o MySQL usa na primeira inicialização:

| Variável | Função |
|---|---|
| `MYSQL_ROOT_PASSWORD` | Senha do usuário `root` do MySQL |
| `MYSQL_DATABASE` | Cria automaticamente o banco `nodedb` ao subir |

---

#### Serviço `node` — Aplicação

```yaml
node:
  build:
    context: ./Node
    dockerfile: dockerfile.prod
  image: gildevso/node:prod
  container_name: node
  depends_on:
    - db
  networks:
    - node-network
```

| Instrução | O que faz |
|---|---|
| `build.context` | Pasta `./Node` onde está o código e o Dockerfile |
| `build.dockerfile` | Usa `dockerfile.prod` — multi-stage build com imagem alpine menor |
| `image: gildevso/node:prod` | Nome/tag que a imagem recebe após o build (pronta para `docker push`) |
| `container_name: node` | Nome fixo do container |
| `depends_on: db` | Garante que o MySQL sobe **antes** do Node — evita erro de conexão |

---

#### Serviço `nginx` — Proxy Reverso

```yaml
nginx:
  build:
    context: ./nginx
    dockerfile: dockerfile.prod
  image: gildevso/nginx:prod
  container_name: nginx
  ports:
    - "8080:80"
  depends_on:
    - node
  networks:
    - node-network
```

| Instrução | O que faz |
|---|---|
| `image: gildevso/nginx:prod` | Nome da imagem gerada |
| `ports: "8080:80"` | Único serviço exposto ao host — porta 8080 do PC → porta 80 do container |
| `depends_on: node` | Nginx só sobe depois que o Node estiver rodando |

---

#### Rede `node-network`

```yaml
networks:
  node-network:
    driver: bridge
```

Cria uma rede interna isolada. Todos os serviços estão nela, podendo se comunicar pelo **nome do serviço** como se fosse um DNS:

```
nginx  → chama → http://node:3000
node   → chama → mysql://db:3306
```

O host (sua máquina) só enxerga a porta `8080` — todo o resto é interno.

---

#### Ordem de inicialização

```
db (MySQL) → node (Express) → nginx (Proxy)
```

Cada serviço espera o anterior graças ao `depends_on`.

---

#### Diferenças entre dev e prod

| Item | Desenvolvimento | Produção |
|---|---|---|
| Dockerfile Node | `dockerfile` (node:22 completo) | `dockerfile.prod` (multi-stage alpine) |
| Volumes no Node | Sim — hot reload ativo | Não — código copiado na build |
| Banco de dados | Não tem | MySQL 5.7 com persistência |
| Tamanho das imagens | Maior | Menor |
| `image:` com tag | Não | Sim — pronto para Docker Hub |

---

### Fluxo completo de uma requisição

```
Você acessa → localhost:8080
                    │
              [Nginx container]
               porta 80 interna
               proxy_pass → http://node:3000
                    │
              [Node container]
               porta 3000
               Express → responde "Hello World"
```

1. Você acessa `http://localhost:8080` no navegador
2. O Nginx recebe na porta 80
3. O Nginx repassa para o serviço `node:3000` (via rede interna Docker)
4. O Express processa e retorna a resposta
5. O Nginx devolve ao navegador

> O nome `node` funciona como endereço porque os dois containers estão na mesma `node-network` — o Docker resolve o nome do serviço automaticamente como DNS interno.

---

### Comandos Docker Compose

#### Desenvolvimento

```bash
docker compose up
```
Sobe todos os serviços definidos no `docker-compose.yml`. Mostra os logs de todos os containers no terminal em tempo real. O terminal fica preso — `Ctrl+C` para parar.

---

```bash
docker compose up --build
```
Mesmo que o anterior, mas **força o rebuild das imagens** antes de subir. Use quando alterar o Dockerfile ou instalar novas dependências.

---

```bash
docker compose up -d --build
```
Sobe os containers em **background (detached)** — o terminal fica livre. A flag `-d` significa *detached*. Combine com `--build` para rebuildar antes de subir.

---

```bash
docker compose down
```
**Para e remove** todos os containers, redes e volumes anônimos criados pelo Compose. Os dados de volumes nomeados são preservados. Use para encerrar o ambiente completamente.

---

#### Tabela resumo

| Comando | O que faz |
|---|---|
| `docker compose up` | Sobe os containers e exibe logs no terminal |
| `docker compose up --build` | Reconstrói as imagens e sobe os containers |
| `docker compose up -d --build` | Reconstrói e sobe em background (terminal livre) |
| `docker compose down` | Para e remove containers e redes |

---

#### Produção

```bash
docker compose -f docker-compose.prod.yml up --build
```
A flag `-f` indica qual arquivo Compose usar. Aqui aponta para o arquivo de produção, que usa imagens menores (alpine) e sem volumes de hot reload.

---

```bash
docker compose -f docker-compose.prod.yml up -d --build
```
Mesmo que o anterior, mas sobe em background.

---

```bash
docker compose -f docker-compose.prod.yml down
```
Para e remove os containers do ambiente de produção.

---

## Fluxo de trabalho com múltiplos terminais

Para desenvolver e depurar o projeto, use **3 terminais abertos ao mesmo tempo**, cada um com uma função:

| Terminal | Função | Comandos |
|---|---|---|
| **Terminal 1** — Docker | Sobe todos os containers | `cd C:\fullCycle\DockerNode` → `docker compose -f docker-compose.prod.yml up -d --build` |
| **Terminal 2** — Logs do Node | Monitora a aplicação em tempo real | `docker logs -f node` |
| **Terminal 3** — MySQL | Consulta e verifica o banco de dados | `docker exec -it db bash` → `mysql -uroot -proot` → `use nodedb;` → `select * from people;` |

### Ordem correta de execução

```
1. Terminal 1 — sobe os containers
         ↓
2. Terminal 2 — acompanha os logs do Node
         ↓
3. Navegador  — acessa http://localhost:8080
         ↓
4. Terminal 3 — confere os dados no MySQL
```

### Comandos rápidos por terminal

**Terminal 1 — subir o ambiente:**
```bash
cd C:\fullCycle\DockerNode
docker compose -f docker-compose.prod.yml up -d --build
```

**Terminal 2 — ver logs do Node:**
```bash
docker logs -f node
```

**Terminal 3 — acessar o banco:**
```bash
docker exec -it db bash
mysql -uroot -proot
use nodedb;
select * from people;
```

---

## Dockerize

O `depends_on` do Docker Compose garante a **ordem de inicialização** dos containers, mas não garante que o serviço interno está pronto. O MySQL pode levar alguns segundos para aceitar conexões mesmo após o container subir.

```
Sem dockerize:
  container db sobe → Node inicia imediatamente → MySQL ainda não aceitando conexões → ERRO

Com dockerize:
  container db sobe → Node aguarda tcp://db:3306 estar acessível → MySQL pronto → Node inicia
```

### Onde está configurado

**[Node/dockerfile](Node/dockerfile)** e **[Node/dockerfile.prod](Node/dockerfile.prod)** — instalação do binário:

```dockerfile
ENV DOCKERIZE_VERSION v0.7.0

RUN apk add --no-cache wget \
    && wget -O /tmp/dockerize.tar.gz https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
    && tar -C /usr/local/bin -xzvf /tmp/dockerize.tar.gz \
    && rm /tmp/dockerize.tar.gz
```

> No `dockerfile` (dev) usa `apt-get` porque a base é Debian (`node:22`). No `dockerfile.prod` usa `apk` porque a base é Alpine (`node:22-alpine`).

---

**[docker-compose.yml](docker-compose.yml)** — uso do dockerize no serviço `node`:

```yaml
node:
  entrypoint: dockerize -wait tcp://db:3306 -timeout 20s node index.js
```

| Parte do comando | O que faz |
|---|---|
| `dockerize` | Ferramenta que aguarda serviços ficarem disponíveis |
| `-wait tcp://db:3306` | Aguarda a porta 3306 do serviço `db` aceitar conexões TCP |
| `-timeout 20s` | Tempo máximo de espera — após 20s retorna erro se o MySQL não responder |
| `node index.js` | Comando executado assim que o MySQL estiver pronto |

### `entrypoint` vs `command`

```yaml
# command → passa argumentos para o entrypoint padrão da imagem
command: dockerize -wait tcp://db:3306 -timeout 20s node index.js

# entrypoint → substitui o executável principal do container
entrypoint: dockerize -wait tcp://db:3306 -timeout 20s node index.js
```

O `entrypoint` é o correto aqui — o dockerize vira o processo principal do container, e `node index.js` é passado como argumento para ele executar após a espera.

### Verificar se o dockerize está instalado

```bash
docker exec -it node sh
dockerize --version
```
