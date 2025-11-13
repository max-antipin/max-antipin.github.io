# Разработка PHP-пакетов

В этом руководстве мы создадим PHP-пакет на примере библиотеки, организуем локальную среду разработки (Xdebug, PHPUnit и линтеры), настроим конвейер CI в GitHub и GitLab, разместим библиотеку на [packagist.org](https://packagist.org/) и в GitLab package registry.

А главное — получим целостную картину взаимодействия всех используемых компонентов.

Под словом «тесты» далее для краткости подразумеваются все виды тестирования и анализа кода.

Прежде чем броситься писать код, давайте подумаем о том, как именно будет выстроен процесс разработки.
- Поскольку это библиотека, а не приложение, требуется поддерживать несколько версий PHP — то есть, запускать тесты на разных версиях PHP.
- Полный прогон тестов после каждой мельчайшей доработки неоправданно долог, поэтому чаще всего на локальной машине будут запускаться отдельные тесты или группы тестов под фичу или исправление бага.
- После завершения задачи нужно прогнать все тесты. Я предпочитаю перед отправкой кода в репозиторий сделать это на своей машине; так быстрее: у нас же библиотека, а не тяжеленное приложение.

## Настройка локального окружения

На локальной машине у нас есть две среды (окружения): development & testing.
- Development: директория проекта монтируется к контейнеру с минимальной поддерживаемой в проекте версией PHP. В нашем примере это 8.2.
- Testing: собираются образы, содержащие файлы проекта, для каждой поддерживаемой минорной версии PHP. В нашем примере это 8.2, 8.3 и 8.4.

Создадим файл `.docker/Dockerfile`:
```Dockerfile
#syntax=docker/dockerfile:1.6

ARG PHP_VERSION
ARG COMPOSER_VERSION=2.9

FROM composer:${COMPOSER_VERSION} AS composer_image

FROM php:${PHP_VERSION}-cli-alpine AS php_dev
RUN set -eux; \
    apk update; \
    apk upgrade; \
    apk add --no-cache linux-headers; \
    apk add --update --no-cache --virtual .build-dependencies $PHPIZE_DEPS; \
    pecl install xdebug; \
    docker-php-ext-enable xdebug; \
    pecl clear-cache; \
    apk del .build-dependencies; \
    mv "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini"; \
    sed -i "s|\(memory_limit =\) [1-9]\+[a-zA-Z]\+|\1 1024M|" "$PHP_INI_DIR/php.ini"; \
    php -r "xdebug_info();"
COPY --from=composer_image --link /usr/bin/composer /usr/local/bin/composer
WORKDIR /usr/src/app/
ENV COMPOSER_ROOT_VERSION=dev-main
```
Для запуска контейнера создадим файл `.docker/composer.yaml`. Использование оркестратора для единственного контейнера выглядит странно, но мы так делаем для удобства, чтобы не нужно было использовать длинную команду запуска контейнера.
```yaml
name: php-package-carcass
services:
  php-dev:
    build:
      args:
        - PHP_VERSION=${PHP_VERSION:-8.2}
      target: php_dev
    configs:
      - source: xdebug
        target: /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini
    container_name: ${COMPOSE_PROJECT_NAME}-dev
    environment:
      XDEBUG_CONFIG: client_port=${XDEBUG_CLIENT_PORT:-9003}
    image: ${COMPOSE_PROJECT_NAME}:dev
    restart: unless-stopped
    tty: true
    volumes:
      - ../.:/usr/src/app/
configs:
  xdebug:
    file: ./xdebug.ini
```

Обычно, `Dockerfile` и `compose.yaml` находятся в корне проекта, но я предпочитаю держать все файлы Docker в отдельной директории, поскольку это не совсем часть проекта — мы вполне можем обходиться без них.

Запускаем контейнер:
```shell
docker compose -f .docker/compose.yaml up -d
```
Заходим в контейнер:
```shell
docker exec -it php-package-carcass-dev sh
```

## Настройка проекта и зависимостей

Теперь нужно создать файл `composer.json`, это можно [сделать вручную](https://getcomposer.org/doc/01-basic-usage.md#introduction), но мы инициализируем проект командой `composer init`.\
`Package name (<vendor>/<name>)` не обязательно должно совпадать с именем репозитория на GitHub (пример: [github.com/PHPCSStandards/PHP_CodeSniffer](https://github.com/PHPCSStandards/PHP_CodeSniffer/) имеет название `squizlabs/php_codesniffer`), это название пакета, под которым он будет находиться в package registry.\
Package Type — library.

После успешного завершения команды `composer init` в созданном `compose.json` исправляем секцию `autoload.psr-4`, убрав оттуда `src/`.

Добавляем ограничение на минимальную версию PHP:
```shell
composer require php:^8.2
```

Поскольку наша библиотека должна работать с разными версиями PHP, то мы не должны держать в репозитории файл `composer.lock`, и поэтому добавляем его в `.gitignore`:
```
/composer.lock
/vendor/
```

```shell
composer require --dev phpunit/phpunit squizlabs/php_codesniffer phpstan/phpstan
```


[Репозиторий для этой статьи](https://github.com/max-antipin/php-package-carcass)

### To do:
- интеграция с VSCode & PHPStorm
- приватный репозиторий в GitLab
- проверка сборки и исключение ненужных файлов (тесты, Docker и т.д.).
- а можно ли мой каркас затаскивать на проект через composer create project и какой-нибудь плагин?
- надо ли для статьи делать теги: PHP, Composer, Docker, GitHub/GitLab?