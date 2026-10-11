# build

Общая база сборки проектов brzhnkv: одна версия Java и Spring Boot, одни правила сборки и один
способ доставки на сервер — вместо копий `build.gradle`, `Dockerfile.ci` и `deploy.yml` в каждом
репозитории.

| Что | Где | Для чего |
|---|---|---|
| Версии | `gradle/libs.versions.toml` | **единственное** место: Java, Spring Boot, общие библиотеки |
| BOM `dev.brzhnkv:platform` | `platform/` | версии для проектов: весь Spring Boot + общие библиотеки |
| Плагин `dev.brzhnkv.service` | `plugins/` | Spring Boot сервис: Java, BOM, тесты, исполняемый jar, воспроизводимые архивы |
| Плагин `dev.brzhnkv.keycloak-extension` | `plugins/` | расширение Keycloak: сборка JDK проекта, байткод под Java Keycloak |
| `java-build.yml` | `.github/workflows/` | тесты и образ в ghcr (общий Dockerfile внутри) |
| `compose-deploy.yml` | `.github/workflows/` | деплой на VPS: compose, ожидание healthy, проверка Flyway |

Плагины и BOM раздаются как Maven-репозиторий с GitHub Pages: `https://brzhnkv.github.io/build/maven`
(без токенов). Версия базы — тег `vX.Y.Z`; плагины, BOM и workflow одной версии.

## Подключить

`settings.gradle`:

```groovy
pluginManagement {
    repositories {
        maven { url = 'https://brzhnkv.github.io/build/maven' }
        gradlePluginPortal()
    }
}
rootProject.name = 'notify'
```

`build.gradle` — только свои зависимости, **без версий** (их даёт BOM):

```groovy
plugins {
    id 'dev.brzhnkv.service' version '1.0.0'
}

version = '0.1.0'

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webmvc'
    implementation 'io.sentry:sentry'            // общая библиотека — версия из BOM
    implementation 'com.example:only-here:1.2.3' // библиотека одного проекта — версия здесь
}
```

`.github/workflows/deploy.yml`:

```yaml
on:
  push: { branches: [ main ] }
  pull_request:
permissions:
  contents: read
  packages: write
jobs:
  build:
    uses: brzhnkv/build/.github/workflows/java-build.yml@v1.0.0
    with:
      image: ghcr.io/brzhnkv/notify
      push: ${{ github.event_name != 'pull_request' }}
  deploy:
    needs: build
    if: github.event_name != 'pull_request'
    uses: brzhnkv/build/.github/workflows/compose-deploy.yml@v1.0.0
    with: { image: ghcr.io/brzhnkv/notify, service: notify, server-dir: /opt/notify }
    secrets: inherit
```

Свой раннер вместо GitHub: `runs-on: '["self-hosted", "orchestrator"]'` в `with:` (JSON; по умолчанию
`"ubuntu-latest"`). Так CI Orchestrator и Golden идут на Mac mini и не тратят минуты Actions.

В compose-файле проекта образ берётся так: `image: ${IMAGE}:${IMAGE_TAG:?…}`. Порт для
healthcheck общего образа — `MANAGEMENT_PORT` (или `SERVER_PORT`) из `environment` compose.

## Что даёт `dev.brzhnkv.service`

- Java из каталога (toolchain), UTF-8;
- Spring Boot и `io.spring.dependency-management` с BOM `dev.brzhnkv:platform` той же версии;
- `spring-boot-starter-test` и запуск JUnit 5; 2 ГБ памяти тестам;
- только исполняемый jar (`build/libs/<имя>-<версия>.jar`), воспроизводимые архивы.

Своё, отличное от общего (имя jar, окружение `bootRun`, Lombok…), остаётся в `build.gradle` проекта.

## Правило версий

В каталог попадает только общее: Java, Spring Boot и библиотеки, которыми пользуются **два проекта и
больше**. Библиотека одного проекта живёт в нём вместе с версией; начал пользоваться второй — версия
переезжает сюда. Запись в каталоге ничего не навязывает: библиотеку получает только тот, кто её
подключил.

## Обновить

1. Правка в `gradle/libs.versions.toml` (или в плагине/workflow), PR — CI соберёт и проверит.
2. Тег `vX.Y.Z` → `release.yml` публикует плагины и BOM. Выпущенную версию не переписываем.
3. В проекте поднять версию в `build.gradle` и в `@vX.Y.Z` workflow — CI проекта прогонит тесты.

## Локально

```bash
./gradlew build publish     # плагины и BOM — в build/repo
```

Проверить проект на невыпущенной базе: временно добавить
`maven { url = 'file:///…/build/build/repo' }` в `pluginManagement` и `repositories` проекта.
