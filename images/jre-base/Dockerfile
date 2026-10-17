# Базовый образ JRE для сервисов, собираемых Jib (CODE_HOST_INDEPENDENCE_DESIGN §4.2): Jib не исполняет RUN,
# поэтому всё, что раньше делал `RUN` в Dockerfile сервиса, — здесь, а Jib только кладёт сверху слои с jar.
#   ghcr.io/brzhnkv/jre-base:25       — wget (для healthcheck в compose) и пользователь app
#   ghcr.io/brzhnkv/jre-base:25-heif  — плюс libheif (Inventa: HEIC/HEIF с айфонов)
# JAVA_OPTS Jib не читает (запускает java сам) — память и строки задаёт JAVA_TOOL_OPTIONS, её JVM берёт всегда.
ARG JAVA=25
FROM eclipse-temurin:${JAVA}-jre-alpine
ARG EXTRA_PACKAGES=""
ENV TZ=UTC \
    JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75 -XX:+UseStringDeduplication"
RUN apk add --no-cache wget ${EXTRA_PACKAGES} \
    && addgroup -S app && adduser -S -u 10001 -G app app
WORKDIR /opt/app
USER app
