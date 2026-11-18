# 05_deployment.md

## Step 1: Выбор платформы

Выбран **Docker + docker-compose** для контейнеризации всех сервисов.

*Rationale:* Docker обеспечивает одинаковое окружение на всех стадиях (разработка, тестирование, продакшен). docker-compose позволяет запустить все сервисы одной командой.

## Step 2: Структура инфраструктуры

```
infrastructure/
├── Dockerfile.backend      # Образ Backend (Python + FastAPI)
├── Dockerfile.frontend     # Образ Frontend (Nginx)
├── nginx.conf              # Конфигурация Nginx для Frontend
└── docker-compose.yml      # Оркестрация сервисов
```

## Step 3: Dockerfile Backend

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Установка зависимостей
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копирование кода
COPY src/backend/ ./backend/
COPY data/models/ ./data/models/

# Переменные окружения
ENV PYTHONUNBUFFERED=1
ENV PORT=8000

EXPOSE 8000

CMD ["uvicorn", "backend.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

*Rationale:* Используется slim-образ для минимального размера. `PYTHONUNBUFFERED=1` гарантирует, что логи выводятся сразу.

## Step 4: Dockerfile Frontend

```dockerfile
FROM nginx:alpine

COPY frontend/ /usr/share/nginx/html/
COPY infrastructure/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 3000

CMD ["nginx", "-g", "daemon off;"]
```

*Rationale:* Nginx alpine — минимальный образ для раздачи статических файлов. Frontend состоит только из HTML/CSS/JS, поэтому сборка не требуется.

## Step 5: docker-compose.yml

```yaml
version: '3.8'

services:
  backend:
    build:
      context: .
      dockerfile: infrastructure/Dockerfile.backend
    ports:
      - "${BACKEND_PORT:-8000}:8000"
    env_file:
      - .env
    volumes:
      - ./data/models:/app/data/models
    restart: unless-stopped

  frontend:
    build:
      context: .
      dockerfile: infrastructure/Dockerfile.frontend
    ports:
      - "${FRONTEND_PORT:-3000}:3000"
    depends_on:
      - backend
    restart: unless-stopped

networks:
  default:
    name: pmi-network
```

*Rationale:*
- Порты задаются через переменные `.env` с дефолтными значениями.
- `volumes` монтируют модель в контейнер, чтобы не пересобирать образ при обновлении модели.
- `depends_on` гарантирует запуск Backend перед Frontend.

## Step 6: Переменные окружения (.env)

```env
# Backend
BACKEND_PORT=8000
MODEL_PATH=data/models/pmi_model.pkl

# Frontend
FRONTEND_PORT=3000

# API
API_URL=http://backend:8000
```

*Rationale:* Все конфигурационные параметры вынесены в `.env`. Файл не должен попадать в git (добавлен в `.gitignore`).

## Step 7: Запуск проекта

```bash
# Сборка и запуск всех сервисов
docker-compose up --build

# Запуск в фоновом режиме
docker-compose up -d

# Остановка
docker-compose down

# Просмотр логов
docker-compose logs -f backend
```

*Сервисы доступны по адресам:*
- Frontend: http://localhost:3000
- Backend API: http://localhost:8000
- Swagger Docs: http://localhost:8000/docs

## Step 8: Проверка работоспособности

```bash
# Проверка health Backend
curl http://localhost:8000/health

# Тестовый запрос к API
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "inflation": 4.0,
    "key_rate": 7.5,
    "oil_price": 80.0,
    "gdp_growth": 2.0,
    "unemployment": 5.0,
    "consumer_confidence": 65.0
  }'
```
