## Pizza Shop — Docker/Compose развертывание

Этот каталог содержит готовую конфигурацию Docker Compose для локальной разработки и продакшен-развертывания фронтенда и бэкенда Pizza Shop, а также Nginx и Redis.

### Состав и образы
- Backend (Django + Celery):
  - Prod: `ghcr.io/polkanoval/pizza-shop-backend:latest` (берётся из реестра)
  - Dev: собирается локально из Dockerfile `../github_pizza-shop-backend/backend/Dockerfile`
- Frontend (React, статика на Nginx):
  - Prod: `ghcr.io/polkanoval/pizza-shop-frontend:latest` (берётся из реестра)
  - Dev: собирается локально из Dockerfile `../github_pizza-shop-frontend/Dockerfile`
- Redis: `redis:7-alpine`
- Nginx: `nginx:stable-alpine` (использует конфиги из `nginx/conf`)

Dockerfile-ы в проекте:
- Backend: `../github_pizza-shop-backend/backend/Dockerfile` (порт приложения 8000)
- Frontend: `../github_pizza-shop-frontend/Dockerfile` (Nginx внутри образа слушает 80)

### Порты, доступные снаружи
- Dev (docker-compose.override.yml активируется автоматически):
  - Backend: `localhost:8000` (прямой доступ к Django)
  - Frontend: `localhost:3000` (прямой доступ к собранному фронту в контейнере)
  - Nginx: `localhost:80` (проксирует на `frontend:80` и `backend-1:8000`)
- Prod (только docker-compose.yml):
  - Nginx: `80:80`, `443:443`
  - Backend/Frontend не пробрасываются наружу напрямую, доступны лишь через Nginx

### Как выбирается режим (Dev/Prod)
- По умолчанию при запуске в этой папке Docker Compose подхватывает оба файла: `docker-compose.yml` и `docker-compose.override.yml`.
  - Это режим разработки (локальные сборки, проброс портов 3000/8000, dev nginx-конфиг).
- Для «чистого» продакшен-режима используйте только основной файл:
  - `docker compose -f docker-compose.yml up -d`

При желании можно явно указать набор файлов через `-f file1 -f file2`.

### Переменные окружения и секреты
- Backend (prod/dev) использует переменные окружения из Compose:
  - Prod: `BETTERSTACK_API_TOKEN=${BETTERSTACK_API_TOKEN}`
    - Значение берётся из переменных окружения хоста или файла `.env` в этой папке (Compose автоматически его читает).
    - Создайте `.env` рядом с этим README, например:
      - `BETTERSTACK_API_TOKEN=ваш_токен`
  - Dev: переменные заданы явно в `docker-compose.override.yml` (`DEBUG`, `PYTHONUNBUFFERED`, `PYTHONPATH`, `DJANGO_SETTINGS_MODULE`, `CELERY_BROKER_URL`, `DJANGO_ALLOWED_HOSTS`).
- Frontend (dev-сборка) может потребовать ключи карт как build-аргументы:
  - `REACT_APP_YMAPS_JS_API_KEY`
  - `REACT_APP_YMAPS_SUGGEST_API_KEY`
  - Передавайте их при сборке, например:
    - `docker compose build --build-arg REACT_APP_YMAPS_JS_API_KEY=xxx --build-arg REACT_APP_YMAPS_SUGGEST_API_KEY=yyy frontend`
  - В prod-режиме используется готовый образ из реестра — ключи уже должны быть учтены на этапе сборки образа.
- TLS (prod Nginx): ожидаются сертификаты Let’s Encrypt на хосте, монтируются в контейнер:
  - Том: `/etc/letsencrypt:/etc/letsencrypt:ro`
  - Домен настраивается в `nginx/conf/prod.conf` (директива `server_name`). Обновите её под свой домен.

### Основные команды Docker Compose
- Запуск (dev по умолчанию, берутся оба файла):
  - `docker compose up -d`
- Остановка и удаление контейнеров:
  - `docker compose down`
  - с удалением анонимных томов (статик): `docker compose down -v`
- Пересборка образов (актуально для dev):
  - `docker compose build`
  - без кэша: `docker compose build --no-cache`
- Просмотр логов:
  - `docker compose logs -f`
- Обновление prod-образов из реестра и рестарт:
  - `docker compose -f docker-compose.yml pull`
  - `docker compose -f docker-compose.yml up -d`

Альтернатива для старой утилиты (если установлена): заменяйте `docker compose` на `docker-compose`.

### Запуск разработки (рекомендуемый локальный флоу)
1) В корне этой папки (deploy) создайте `.env` при необходимости (например, если нужен `BETTERSTACK_API_TOKEN`).
2) Запустите сервисы:
   - `docker compose up -d`
3) Доступы:
   - Фронт: `http://localhost:3000` (напрямую) или `http://localhost` (через Nginx)
   - Бэк: `http://localhost:8000`
4) Статика и медиа:
   - `staticfiles` собираются в общий том `static_volume` и монтируются в Nginx как `/var/www/staticfiles` (доступны по `/djstatic/`)
   - Медиа в dev/ prod монтируются из локальной папки `./media` в `/var/www/media` (доступно по `/media/`)

### Запуск продакшен-режима (минимальный сценарий)
1) Убедитесь, что DNS домена указывает на сервер.
2) Обновите `nginx/conf/prod.conf` — задайте свой `server_name`.
3) Убедитесь, что на хосте есть сертификаты Let’s Encrypt в `/etc/letsencrypt/...`.
4) Заполните `.env` (если нужен `BETTERSTACK_API_TOKEN`).
5) Запустите только prod-файл:
   - `docker compose -f docker-compose.yml up -d`
6) Доступы:
   - `http://<домен>` и `https://<домен>` через Nginx (80/443)

### Что внутри Compose
- `docker-compose.yml` (prod):
  - Backend/Frontend — из `ghcr.io/...:latest`
  - `nginx` использует `nginx/conf/prod.conf`, пробрасывает `80:80`, `443:443`
  - Монтирует `/etc/letsencrypt` и общий том `static_volume` + локальную `./media`
- `docker-compose.override.yml` (dev):
  - Собирает backend из `../github_pizza-shop-backend/backend`
  - Собирает frontend из `../github_pizza-shop-frontend`
  - Пробрасывает `8000:8000` (backend), `3000:80` (frontend), `80:80` (nginx dev-конфиг)
  - Redis для Celery

### Полезно знать
- Переменные в виде `${NAME}` читаются из переменных окружения ОС и/или файла `.env` рядом с `docker-compose.yml`.
- При использовании и prod, и override вперемешку могут меняться имена контейнеров (в dev для backend задан `container_name: backend-1`). Это нормально.
- Для чистого продакшена всегда запускайте с явным `-f docker-compose.yml`.

Если понадобится CI/CD или кастомные профили, можно добавить профили Compose и/или отдельные файлы `docker-compose.prod.yml` и `docker-compose.dev.yml`.
