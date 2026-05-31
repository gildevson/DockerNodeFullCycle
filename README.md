# DockerNode FullCycle

Estudos de Docker com Node.js — repositório do curso FullCycle.

## Projetos

### [Node/](Node/)

Aplicação Node.js com Express containerizada via Docker.

- Servidor Express na porta 3000
- Dockerfile para desenvolvimento
- Dockerfile de produção com Multistage Building + Alpine
- Imagem publicada no Docker Hub: `gildevso/hello-express`

## Conceitos abordados

- Criação e execução de containers Docker
- Mapeamento de portas e volumes
- Escrita de Dockerfiles
- Multistage Building para otimização de imagens
- Publicação de imagens no Docker Hub
- Diferença entre nginx + PHP-FPM vs Node.js + Express
