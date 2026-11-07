# Nginx que roda como usuario nao-root (UID 101) e escuta na porta 8080.
FROM nginxinc/nginx-unprivileged:1.28-alpine

USER root
# Remove os modulos amarrados à versão antiga que bloqueiam a atualização e aplica o patch
RUN apk del nginx-module-geoip nginx-module-image-filter nginx-module-njs nginx-module-xslt && \
    apk update && apk upgrade --no-cache && \
    apk add --no-cache nginx=1.28.3-r7
USER 101

COPY html/ /usr/share/nginx/html/
EXPOSE 8080