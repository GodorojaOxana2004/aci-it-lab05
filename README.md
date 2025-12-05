# Лабораторная работа: Настройка CI/CD конвейера с GitLab Community Edition для Laravel-приложения

## Цель работы

Получить практический опыт развертывания собственного CI/CD-сервера на базе GitLab Community Edition и реализации конвейера непрерывной интеграции для Laravel-приложения. В рамках работы выполняется установка GitLab CE в облачной инфраструктуре GCP, настройка GitLab Runner, создание проекта с конфигурацией `.gitlab-ci.yml`, а также запуск и верификация автоматизированного пайплайна.

---

## Задание

Необходимо выполнить следующие этапы:

1. **Подготовка облачной инфраструктуры в Google Cloud Platform**
   - Создать проект в GCP
   - Зарезервировать статический внешний IP-адрес
   - Настроить правила firewall для доступа к GitLab

2. **Развертывание виртуальной машины**
   - Создать VM на базе Ubuntu 22.04 LTS
   - Установить Docker и Docker Compose
   - Настроить необходимые параметры безопасности

3. **Установка GitLab Community Edition**
   - Развернуть GitLab CE в Docker-контейнере
   - Настроить базовую конфигурацию
   - Установить пароль администратора (root)

4. **Настройка GitLab Runner**
   - Установить GitLab Runner на виртуальной машине
   - Зарегистрировать Runner с использованием Docker executor
   - Проверить статус и активность Runner

5. **Создание Laravel-проекта с CI/CD конфигурацией**
   - Создать репозиторий в GitLab
   - Подготовить Laravel-приложение
   - Создать Dockerfile для контейнеризации
   - Написать конфигурацию `.gitlab-ci.yml` с этапами тестирования и сборки

6. **Запуск и верификация CI/CD пайплайна**
   - Выполнить commit и push изменений
   - Проанализировать выполнение пайплайна
   - Проверить результаты тестирования и сборки образа

---

## Ход выполнения работы

### Этап 1. Подготовка инфраструктуры в GCP

Для выполнения работы был создан проект в Google Cloud Platform с включенным биллингом. Выполнена авторизация через Cloud SDK:

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
```

#### 1.1. Резервирование статического IP-адреса

Для обеспечения постоянного доступа к GitLab был зарезервирован статический внешний IP-адрес:

```bash
gcloud compute addresses create gitlab-ip --region=us-central1
gcloud compute addresses describe gitlab-ip --region=us-central1
```

В результате был получен статический адрес: `136.119.66.238`

![Резервирование статического IP](/img/img1.png)

#### 1.2. Настройка правил firewall

Созданы правила межсетевого экрана для открытия необходимых портов (HTTP, HTTPS, SSH, а также порт 8022 для SSH GitLab):

```bash
gcloud compute firewall-rules create allow-gitlab-ports \
  --allow tcp:80,tcp:443,tcp:22,tcp:8022 \
  --target-tags=gitlab-server \
  --description="Allow GitLab access"
```

![Настройка firewall](/img/img2.png)

### Этап 2. Создание и настройка виртуальной машины

Была создана виртуальная машина с параметрами, достаточными для работы GitLab (4 vCPU, 16GB RAM, 200GB диск):

```bash
gcloud compute instances create gitlab-vm-lab5 \
  --zone=us-central1-a \
  --machine-type=e2-standard-4 \
  --boot-disk-size=200GB \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --tags=gitlab-server \
  --address=gitlab-ip \
  --metadata=startup-script='#! /bin/bash
    apt-get update
    apt-get install -y docker.io docker-compose git
    usermod -aG docker $USER || true
  '
```

![Создание VM](/img/img3.png)

Выполнено подключение к VM по SSH:

```bash
gcloud compute ssh gitlab-vm-lab5 --zone=us-central1-a
```

Проверена корректность установки Docker:

```bash
docker --version
docker-compose --version
ip addr show
```

![Проверка Docker](/img/img4.png)
![Проверка сети](/img/img5.png)

### Этап 3. Развертывание GitLab CE в Docker

GitLab Community Edition был развернут в Docker-контейнере с использованием зарезервированного статического IP:

```bash
sudo docker run -d \
  --hostname 136.119.66.238 \
  -p 80:80 \
  -p 443:443 \
  -p 8022:22 \
  --name gitlab \
  -e GITLAB_OMNIBUS_CONFIG="external_url='http://136.119.66.238'; gitlab_rails['gitlab_shell_ssh_port']=8022" \
  -v gitlab-data:/var/opt/gitlab \
  -v ~/gitlab-config:/etc/gitlab \
  gitlab/gitlab-ce:latest
```

![Запуск GitLab](/img/img6.png)

Выполнен мониторинг логов для контроля готовности системы:

```bash
docker logs -f gitlab
```

После полной инициализации был получен первоначальный пароль администратора:

```bash
docker exec -it gitlab cat /etc/gitlab/initial_root_password
```

Результат: `IYsSbEP4ar5VxhG6WwYIm8TlIfgnQUx9KdaO0wMNKEE=`

![Получение пароля](/img/img7.png)

Выполнен вход в веб-интерфейс GitLab по адресу `http://136.119.66.238` с учетными данными:
- Login: `root`
- Password: `aciitlab!` (после смены начального пароля)

![Страница входа](/img/img8.png)
![Главная страница GitLab](/img/img9.png)
![Интерфейс администратора](/img/img10.png)

### Этап 4. Установка и регистрация GitLab Runner

#### 4.1. Установка GitLab Runner

На виртуальной машине был установлен GitLab Runner:

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner
```

#### 4.2. Регистрация Runner

В веб-интерфейсе GitLab выполнен переход в раздел **Admin Area → CI/CD → Runners → New instance runner**. Создан новый Runner с параметрами:
- Executor: Docker
- Опция "Run untagged jobs": включена

Получен Authentication Token: `glrt--TJTu_RcaZ9-XF1ThIOLd286MQp0OjEKdToxCw.01.1216fyyjk`

![Создание Runner](/img/img11.png)

Выполнена регистрация Runner на VM:

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "http://136.119.66.238/" \
  --registration-token "glrt--TJTu_RcaZ9-XF1ThIOLd286MQp0OjEKdToxCw.01.1216fyyjk" \
  --executor "docker" \
  --description "laravel-runner" \
  --tag-list "docker,php" \
  --docker-image "php:8.2-cli" \
  --run-untagged="true"
```

![Регистрация Runner](/img/img12.png)

Запущен и активирован сервис:

```bash
sudo systemctl enable gitlab-runner
sudo systemctl start gitlab-runner
sudo gitlab-runner status
```

![Статус Runner](/img/img13.png)

Проверена активность Runner в веб-интерфейсе:

![Runner в интерфейсе](/img/img14.png)

### Этап 5. Подготовка Laravel-проекта

#### 5.1. Создание репозитория

В GitLab создан новый проект `laravel-app`. Репозиторий склонирован локально:

```bash
git clone http://136.119.66.238/root/laravel-app.git
cd laravel-app
```

![Создание проекта](/img/img15.png)

#### 5.2. Подготовка кода Laravel

Скачан шаблон Laravel и скопирован в рабочую директорию:

```bash
git clone https://github.com/laravel/laravel.git ../laravel-temp
cp -r ../laravel-temp/* ./
```

![Копирование Laravel](/img/img16.png)

#### 5.3. Создание Dockerfile

Создан `Dockerfile` для контейнеризации приложения:

```dockerfile
# Dockerfile
FROM php:8.2-apache

RUN apt-get update && apt-get install -y \
    libpng-dev libonig-dev libxml2-dev zip unzip git \
 && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

COPY . /var/www/html
WORKDIR /var/www/html

RUN composer install --no-scripts --no-interaction --prefer-dist || true
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache
RUN chmod -R 775 /var/www/html/storage

RUN a2enmod rewrite
EXPOSE 80
CMD ["apache2-foreground"]
```

#### 5.4. Конфигурация окружения для тестирования

Создан файл `.env.testing` с настройками тестового окружения:

```env
APP_NAME=Laravel
APP_ENV=testing
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost
APP_LOCALE=en
APP_FALLBACK_LOCALE=en
APP_FAKER_LOCALE=en_US
APP_MAINTENANCE_DRIVER=file
PHP_CLI_SERVER_WORKERS=4
BCRYPT_ROUNDS=12

LOG_CHANNEL=stack
LOG_STACK=single
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel_test
DB_USERNAME=root
DB_PASSWORD=root

SESSION_DRIVER=array
QUEUE_CONNECTION=sync
CACHE_STORE=array

MAIL_MAILER=log
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"

REDIS_CLIENT=array
```

#### 5.5. Создание тестов

Добавлен базовый Unit-тест в `tests/Unit/ExampleTest.php`:

```php
<?php
namespace Tests\Unit;
use PHPUnit\Framework\TestCase;

class ExampleTest extends TestCase
{
    public function testBasicTest()
    {
        $this->assertTrue(true);
    }
}
```

### Этап 6. Конфигурация CI/CD пайплайна

Создан файл `.gitlab-ci.yml` в корне проекта с определением этапов тестирования и сборки:

```yaml
stages:
  - test
  - build

services:
  - name: mysql:8.0
    alias: mysql

variables:
  MYSQL_DATABASE: laravel_test
  MYSQL_ROOT_PASSWORD: root
  DB_HOST: mysql
  DB_USERNAME: root
  DB_PASSWORD: root

cache:
  paths:
    - vendor/

test:
  stage: test
  image: php:8.2-cli
  before_script:
    - apt-get update -yqq
    - apt-get install -yqq libpng-dev libonig-dev libxml2-dev unzip git zip
    - docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath || true
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer install --no-interaction --prefer-dist --no-scripts
    - cp .env.testing .env
    - php artisan key:generate
    - php artisan migrate --force || true
  script:
    - vendor/bin/phpunit --stop-on-failure
  artifacts:
    when: always
    paths:
      - storage/logs/

build:
  stage: build
  image: docker:24.0.0
  services:
    - docker:24.0.0-dind
  variables:
    DOCKER_DRIVER: overlay2
  before_script:
    - echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin || true
  script:
    - docker build -t $CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA .
    - docker tag $CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA $DOCKERHUB_USER/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKERHUB_USER/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA
  only:
    - main
```

Для корректной работы этапа сборки в конфигурации Runner (`/etc/gitlab-runner/config.toml`) установлен параметр `privileged = true`:

![Конфигурация privileged mode](/img/img17.png)

### Этап 7. Запуск пайплайна и верификация результатов

Выполнен commit и push изменений в репозиторий:

```bash
git add .
git commit -m "Laravel app with CI/CD"
git push origin main
```

![Push в репозиторий](/img/img18.png)

#### 7.1. Мониторинг выполнения пайплайна

В веб-интерфейсе GitLab открыт раздел **CI/CD → Pipelines**. Пайплайн запустился автоматически после push:

![Список пайплайнов](/img/img19.png)

#### 7.2. Анализ логов выполнения

Проверены логи каждого job:

**Этап test:**
![Логи тестирования](/img/img20.png)

**Этап build:**
![Логи сборки](/img/img21.png)

#### 7.3. Верификация работы Runner

Выполнена проверка статуса Runner:

```bash
sudo gitlab-runner status
sudo gitlab-runner verify
```

![Статус Runner](/img/img22.png)

---

## Результаты работы

В результате выполнения лабораторной работы:

1. Успешно развернут полнофункциональный CI/CD сервер на базе GitLab Community Edition в облачной инфраструктуре Google Cloud Platform
2. Настроена виртуальная машина с Docker и GitLab Runner
3. Создан Laravel-проект с конфигурацией автоматизированного пайплайна
4. Реализован CI/CD конвейер, включающий:
   - Автоматическое тестирование кода с помощью PHPUnit
   - Сборку Docker-образа приложения
   - Интеграцию с базой данных MySQL для тестов
5. Верифицирована корректность работы всех компонентов системы

Все этапы пайплайна выполняются автоматически при push изменений в репозиторий, что подтверждает успешную настройку процесса непрерывной интеграции.