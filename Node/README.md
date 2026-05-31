# hello-express

Aplicação Node.js com Express containerizada via Docker.

## Stack

- Node.js 22
- Express 5
- Docker

## Estrutura

```
Node/
├── dockerfile        # Definição da imagem Docker
├── index.js          # Servidor Express (porta 3000)
├── package.json      # Dependências do projeto
└── node_modules/     # Dependências instaladas
```

## Dockerfile

```dockerfile
FROM node:22

WORKDIR /usr/src/app

COPY . .

EXPOSE 3000

CMD ["node", "index.js"]
```

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


