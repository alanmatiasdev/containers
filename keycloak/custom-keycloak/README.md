# Imagem interna do Keycloak baseada no layout Bitnami

Este diretório replica a estrutura utilizada pela Bitnami para empacotar o Keycloak, mas permite que gerar a imagem localmente e publique em um registry privado. O objetivo é manter o máximo de compatibilidade possível com os contêineres originais, evitando depender das imagens públicas que foram descontinuadas.

## Estrutura

```
containers/internal-keycloak/
└── 26/debian-12/
    ├── Dockerfile          # Receita principal (base Debian 12 oficial)
    ├── docker-compose.yml  # Exemplo apontando para a imagem interna
    ├── prebuildfs/         # Ferramentas auxiliares (install_packages, libs, etc.)
    └── rootfs/             # Scripts específicos do Keycloak
```

> A cópia exata do upstream permanece em `containers/references/bitnami/`.

## Diferenças em relação à imagem pública da Bitnami

- Base: usamos `debian:12` em vez de `bitnami/minideb:bookworm`; isso mantém os mesmos pacotes, mas muda o fornecedor e a fonte dos updates.
- Metadados OCI e nome da imagem foram atualizados para apontar para o repositório interno (`alanmatias/keycloak`), evitando confusão com a tag oficial.
- O `docker-compose.yml` passou a construir localmente a imagem (build context) antes de subir o serviço, garantindo que sempre haja uma versão interna disponível.
- Todos os scripts copiados em `prebuildfs/` e `rootfs/` precisam manter o bit de execução versionado; sem isso, etapas como `install_packages` falham durante o build.

### O que observar em futuras atualizações

1. **Base image** – decidir se continua em `debian:12` ou se há necessidade de reintroduzir o `minideb`; caso mude, avalie pacotes e permissões padrão dos scripts.
2. **Permissões dos scripts** – sempre validar (ou rodar `find ... -exec chmod +x {} +`) antes de fazer o build para garantir que `install_packages`, entrypoints e hooks sejam executáveis.
3. **Metadados** – sincronizar `LABEL`, `APP_VERSION`, `IMAGE_REVISION` e nomes de componentes para refletir a nova versão e diferenciar do upstream.
4. **Downloads** – conferir se a Bitnami renomeou artefatos; se sim, atualize a lista `COMPONENTS` e, caso use um mirror interno, mantenha a convenção de nomes.

## Construção da imagem

```bash
cd containers/internal-keycloak/26/debian-12
# Necessário BuildKit por conta do --mount=type=secret
DOCKER_BUILDKIT=1 docker build \
  -t alanmatias/keycloak:26.4.0 \
  .
```

### Utilizando um mirror dos artefatos Bitnami

O Dockerfile continua buscando os artefatos oficiais (`wait-for-port`, `jre`, `keycloak`) via `downloads.bitnami.com`. Caso você espelhe esses arquivos em outro local, basta passar a URL durante o `build`:

```bash
DOCKER_BUILDKIT=1 docker build \
  --secret id=downloads_url,env=SECRET_DOWNLOADS_URL \
  -t registro.local/keycloak:26.4.0 \
  .
```

> O valor `SECRET_DOWNLOADS_URL` deve apontar para o prefixo que contém os arquivos `.tar.gz` e `.sha256` exatamente com os mesmos nomes usados pela Bitnami.

### Build multi-plataforma

O Dockerfile já respeita `TARGETARCH`. Para gerar imagens multi-arch:

```bash
DOCKER_BUILDKIT=1 docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t registro.local/keycloak:26.4.0 \
  .
```

## Teste rápido com docker-compose

O `docker-compose.yml` deste diretório já referencia a imagem interna e adiciona `build.context: .` para facilitar o desenvolvimento:

```bash
cd containers/internal-keycloak/26/debian-12
DOCKER_BUILDKIT=1 docker compose up --build
```

> O serviço de banco continua usando a imagem pública do PostgreSQL Bitnami. Altere-o conforme a sua estratégia interna (por exemplo, `postgres:16`) se quiser eliminar essa dependência também.

## Publicação em registro privado

Depois de construir e validar a imagem:

```bash
docker tag alanmatias/keycloak:26.4.0 registry.local/keycloak:26.4.0
docker push registry.local/keycloak:26.4.0
```

## Atualizando versões

1. Ajuste `org.opencontainers.image.version`, `APP_VERSION` e os nomes dos componentes na variável `COMPONENTS` do Dockerfile.
2. Atualize o `image` do `docker-compose.yml` para refletir a nova tag.
3. Documente a mudança neste README.

Com isso podemos controlar todo o ciclo de vida da imagem do Keycloak mantendo-se alinhado à estrutura Bitnami.
