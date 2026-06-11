# dockerfiles-labs

Laboratório prático de Dockerfiles criado durante estudos de containerização. Cada arquivo representa uma etapa evolutiva de aprendizado, partindo de imagens básicas até builds multi-estágio otimizados.

---

## Estrutura do projeto

```text
dockerfiles-labs/
└── Aulas-dias01-02/
    ├── Primeiro-dockerfile          # Nginx básico
    ├── Segundo-Dockerfile           # Nginx + boas práticas
    ├── Terceiro-Dockerfile          # Nginx + ENTRYPOINT + LABEL
    ├── Dockerfile                   # Nginx completo com HEALTHCHECK e ADD
    ├── Quarto-Dockerfile            # App Go (single-stage)
    ├── Quarto-Dockerfile-primeiro   # App Go (multi-stage com Alpine)
    ├── Quarto-Dockerfile-segundo    # App Go (multi-stage + ENV + ARG + VOLUME)
    ├── hello.go                     # Código-fonte Go usado nos builds
    └── index.html                   # Página HTML servida pelo Nginx
```

---

## Dockerfiles — evolução e conceitos

### 1. `Primeiro-dockerfile` — Nginx mínimo

```dockerfile
FROM ubuntu:18.04
RUN apt-get update && apt-get install nginx -y
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Conceitos:** `FROM`, `RUN`, `EXPOSE`, `CMD`

```bash
docker build -f Primeiro-dockerfile -t nginx-basico .
docker run -p 8080:80 nginx-basico
```

---

### 2. `Segundo-Dockerfile` — Limpeza de cache e página customizada

```dockerfile
FROM ubuntu:18.04
RUN apt-get update && apt-get install nginx -y && rm -rf /var/lib/apt/lists/*
EXPOSE 80
COPY index.html /var/www/html/
CMD ["nginx", "-g", "daemon off;"]
WORKDIR /var/www/html
ENV APP_VERSION 1.0.0
```

**Conceitos:** `COPY`, `WORKDIR`, `ENV`, limpeza do cache `apt` para reduzir tamanho da imagem

```bash
docker build -f Segundo-Dockerfile -t nginx-custom .
docker run -p 8080:80 nginx-custom
```

---

### 3. `Terceiro-Dockerfile` — ENTRYPOINT e metadados

```dockerfile
FROM ubuntu:18.04
LABEL maintainer="csar.santos18@gmail.com"
RUN apt-get update && apt-get install nginx -y && rm -rf /var/lib/apt/lists/*
EXPOSE 80
COPY index.html /var/www/html/
WORKDIR /var/www/html
ENV APP_VERSION 1.0.0
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

**Conceitos:** `LABEL`, diferença entre `ENTRYPOINT` e `CMD`  
> `ENTRYPOINT` define o executável fixo; `CMD` fornece argumentos padrão que podem ser sobrescritos em `docker run`.

```bash
docker build -f Terceiro-Dockerfile -t nginx-entrypoint .
docker run -p 8080:80 nginx-entrypoint
```

---

### 4. `Dockerfile` — Imagem completa com HEALTHCHECK e ADD

```dockerfile
FROM ubuntu:18.04
LABEL maintainer="csar.santos18@gmail.com"
RUN apt-get update && apt-get install nginx curl -y && rm -rf /var/lib/apt/lists/*
EXPOSE 80
ADD node_exporter-1.6.0.linux-amd64.tar.gz /root/node-exporter
COPY index.html /var/www/html/
WORKDIR /var/www/html
ENV APP_VERSION 1.0.0
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
HEALTHCHECK --timeout=2s CMD curl -f localhost || exit 1
```

**Conceitos:** `ADD` (extrai `.tar.gz` automaticamente), `HEALTHCHECK`

```bash
docker build -t nginx-completo .
docker run -p 8080:80 nginx-completo
docker inspect --format='{{.State.Health.Status}}' <container_id>
```

---

### 5. `Quarto-Dockerfile` — App Go (single-stage)

```dockerfile
FROM golang:1.18
WORKDIR /app
COPY . ./
RUN go mod init hello
RUN go build -o /app/hello
CMD ["/app/hello"]
```

**Conceitos:** build de aplicação Go dentro do container  
> Desvantagem: a imagem final contém todo o toolchain Go (~800 MB).

```bash
docker build -f Quarto-Dockerfile -t go-hello .
docker run go-hello
```

---

### 6. `Quarto-Dockerfile-primeiro` — Multi-stage build

```dockerfile
FROM golang:1.18 AS buildando
WORKDIR /app
COPY . ./
RUN go mod init hello
RUN go build -o /app/hello

FROM alpine:3.15.9
COPY --from=buildando /app/hello /app/hello
CMD ["/app/hello"]
```

**Conceitos:** multi-stage build — o binário compilado é copiado para uma imagem Alpine mínima (~10 MB).

```bash
docker build -f Quarto-Dockerfile-primeiro -t go-hello-alpine .
docker run go-hello-alpine
```

---

### 7. `Quarto-Dockerfile-segundo` — Multi-stage + ENV + ARG + VOLUME

```dockerfile
FROM golang:1.18 AS buildando
WORKDIR /app
COPY . ./
RUN go mod init hello
RUN go build -o /app/hello

FROM alpine:3.15.9
COPY --from=buildando /app/hello /app/hello
ENV app="hello_world"
ARG GIROPOPS="strigus"
ENV GIROPOPS=$GIROPOPS
VOLUME /app/dados
RUN echo "O giropops é: $GIROPOPS"
CMD ["/app/hello"]
```

**Conceitos:** `ARG` (variável disponível apenas em build-time), `VOLUME` para persistência de dados

```bash
# Build com ARG padrão
docker build -f Quarto-Dockerfile-segundo -t go-hello-final .

# Build sobrescrevendo o ARG
docker build -f Quarto-Dockerfile-segundo --build-arg GIROPOPS=meuvalor -t go-hello-final .

docker run go-hello-final
```

---

## Comparação de tamanho de imagem

| Dockerfile | Base | Abordagem | Tamanho estimado |
| --- | --- | --- | --- |
| Primeiro-dockerfile | ubuntu:18.04 | Single-stage | ~170 MB |
| Segundo / Terceiro / Dockerfile | ubuntu:18.04 | Single-stage | ~180 MB |
| Quarto-Dockerfile | golang:1.18 | Single-stage | ~850 MB |
| Quarto-Dockerfile-primeiro | alpine:3.15.9 | Multi-stage | ~12 MB |
| Quarto-Dockerfile-segundo | alpine:3.15.9 | Multi-stage | ~12 MB |

---

## Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/) 20.10+

---

## Referência rápida das instruções

| Instrução | Função |
| --- | --- |
| `FROM` | Define a imagem base |
| `RUN` | Executa comando durante o build |
| `COPY` | Copia arquivos do host para a imagem |
| `ADD` | Como COPY, mas também extrai arquivos `.tar.gz` |
| `EXPOSE` | Documenta a porta que o container escuta |
| `ENV` | Define variável de ambiente persistente |
| `ARG` | Define variável disponível apenas durante o build |
| `WORKDIR` | Define o diretório de trabalho |
| `ENTRYPOINT` | Executável principal do container (fixo) |
| `CMD` | Argumentos padrão para o ENTRYPOINT (sobrescrevível) |
| `LABEL` | Adiciona metadados à imagem |
| `VOLUME` | Declara ponto de montagem para persistência |
| `HEALTHCHECK` | Define verificação de saúde do container |
