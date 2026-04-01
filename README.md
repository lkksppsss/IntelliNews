# IntelliNews — 新機器環境建立指南

## 必要環境

| 工具 | 版本 |
|------|------|
| Python | 3.14+ |
| Poetry | 2.0+ |
| .NET SDK | 10.0+ |
| PHP | 8.2+ |
| Composer | 最新 |
| PostgreSQL | 任意新版 |
| RabbitMQ | 任意新版 |

---

## 1. Clone

```bash
git clone --recursive https://github.com/lkksppsss/IntelliNews.git
cd IntelliNews
```

---

## 2. PostgreSQL — 建立資料庫

```bash
psql -U postgres -c "CREATE DATABASE news_crawler;"
psql -U postgres -c "CREATE DATABASE news_app_db;"
psql -U postgres -c "CREATE DATABASE laravel_notify;"
```

| DB 名稱 | 用途 |
|---------|------|
| `news_crawler` | 爬蟲資料（crawler_grpc_server / FastAPI） |
| `news_app_db` | Django 用戶資料 |

---

## 3. crawler_grpc_server

```bash
cd Python/crawler_grpc_server
poetry install
```

建立 `.env`，參考以下內容填入實際值：
```
DATABASE_URL=postgresql://postgres:<password>@localhost:5432/news_crawler
```

跑 migration（Alembic）：
```bash
poetry run alembic upgrade head
```

啟動：
```bash
poetry run python server.py
```

---

## 4. intellinews_fastapi

```bash
cd Python/intellinews_fastapi
poetry install
```

建立 `.env`：
```
DATABASE_URL=postgresql://postgres:<password>@localhost:5432/news_crawler
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
```

啟動：
```bash
poetry run uvicorn src.intellinews_fastapi.main:app --reload
```

---

## 5. intellinews-app（Django）

```bash
cd Python/intellinews-app
poetry install
```

編輯 `intellinews/intellinews/settings.py`，填入 DB 密碼：
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'news_app_db',
        'USER': 'postgres',
        'PASSWORD': '<password>',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

建立資料表並建立使用者：
```bash
cd intellinews
poetry run python manage.py migrate
poetry run python manage.py createsuperuser
poetry run python manage.py shell -c "from django.contrib.auth.models import User; User.objects.create_user('service', password='unused')"
```

啟動：
```bash
poetry run python manage.py runserver 8080
```

---

## 6. IntelliNewsGateway（C#）

```bash
cd "C#/IntelliNewsGateway"
dotnet restore
```

編輯 `appsettings.json`，填入 JWT Secret（與 Django `SIMPLE_JWT` 的 `SIGNING_KEY` 相同）：
```json
"Jwt": {
  "Issuer": "intellinews-django",
  "Audience": "intellinews-gateway",
  "Secret": "<shared_secret>"
}
```

啟動：
```bash
dotnet run
```

---

## 7. laravel_notify（PHP）

```bash
cd PHP/laravel_notify
composer install
cp .env.example .env
php artisan key:generate
```

編輯 `.env`，填入以下設定：
```
DB_CONNECTION=pgsql
DB_HOST=host.docker.internal
DB_PORT=5432
DB_DATABASE=laravel_notify
DB_USERNAME=postgres
DB_PASSWORD=<password>
RABBITMQ_HOST=host.docker.internal
RABBITMQ_PORT=5672
RABBITMQ_USER=guest
RABBITMQ_PASS=guest
DISCORD_WEBHOOK_URL=<your_webhook_url>
WWWGROUP=1000
WWWUSER=1000
```

建立 DB，啟動並初始化：
```bash
# 第一次啟動需 build
docker compose up -d --build

docker compose exec laravel.test php artisan migrate
docker compose exec laravel.test php artisan db:seed
```

啟動背景服務（各開一個 terminal）：
```bash
# Queue worker
docker compose exec laravel.test php artisan queue:work

# RabbitMQ consumer
docker compose exec laravel.test php artisan rabbitmq:consume-article-created
```

---

## 啟動順序

```
1. PostgreSQL（背景服務）
2. RabbitMQ（背景服務）
3. crawler_grpc_server  → port 50051
4. intellinews_fastapi  → port 8000
5. IntelliNewsGateway   → port 5138
6. intellinews-app      → port 8080
7. laravel_notify       → port 8001（選用）
```
