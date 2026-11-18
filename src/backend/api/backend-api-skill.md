---
name: Backend API Skill
description: Правила разработки FastAPI бэкенда для прогноза PMI
---

# Backend API Skill

## Goal (Цель)
Создание высокопроизводительного и безопасного REST API. Сервис должен принимать макроэкономические показатели, передавать их в ML-модель и возвращать прогноз PMI в формате JSON с указанием статуса экономики.

## 1. Dependency Management (Управление зависимостями)
1.  **Файл:** Использовать `requirements.txt`.
2.  **Актуальность:** Запрещено писать версии библиотек "из головы". 
    *   *Как правильно:* Активация `venv` $\rightarrow$ `pip install package` $\rightarrow$ `pip freeze > requirements.txt`.
3.  **Разделение:** 
    *   Для разработки (линтеры, тесты) используется `requirements-dev.txt`.
    *   В `requirements.txt` для продакшена должны быть только библиотеки для запуска: `fastapi`, `uvicorn`, `catboost`, `joblib`, `pydantic`, `numpy`, `pandas`.
4.  **Запрет:** Не использовать `poetry` или `pipenv`. Стандарт — чистый `requirements.txt`.

## 2. Tech Stack (Стек технологий)
*   **Framework:** FastAPI.
*   **Server:** Uvicorn (ASGI).
*   **Data Validation:** Pydantic (BaseModel).

## 3. API Endpoints (Эндпоинты)
### 3.1. POST /predict (Основной)
*   **Input:** Строгая валидация через `pydantic.BaseModel`.
*   **Output:** JSON с прогнозом и статусом.

**Пример Input (Request):**
```json
{
  "inflation": 8.5,
  "key_rate": 16.0,
  "oil_price": 75.0,
  "consumer_confidence": 105.0
}
**Пример Output (Response):**
{
  "predicted_pmi": 48.2,
  "status": "recession", 
  "probability": 0.75
}
```
*Примечание: status — это enum. Если pmi >= 50, то expansion. Если < 50, то recession.*

### 3.2. GET /health (Проверка жизни)
* **Цель:** Для Docker/Kubernetes healthcheck.
* **Output:** {"status": "ok"} со статусом 200.
### 3.3. GET /docs (Документация)
* **Правило:** Swagger (OpenAPI) должен быть включен по умолчанию (FastAPI делает это сам). 
* **Требование:** Эндпоинты должны быть задокументированы через docstrings в коде.
## 4. Architecture Rules (Архитектурные правила)
1. **Валидация:**
  - Использовать con(gt=0, le=100) для макропоказателей в Pydantic.
  - Пример: inflation: float = Field(..., description="Инфляция в процентах", ge=0, le=100).
2. **Производительность:** Время ответа эндпоинта /predict не должно превышать **200ms.**
3. **Безопасность:**
  - **Порты:** Запрещен хардкод портов в uvicorn.run() внутри Python-кода. Порт берется из переменной окружения или задается в docker-compose.
  - **Секреты:** Никаких API-ключей или паролей в коде. Только .env файл.
  - **Пользователь:** В Docker контейнере приложение должно запускаться от пользователя appuser (не от root).
4. **Загрузка модели:**
  - Модель (model.pkl) загружается **один раз** при старте сервера (глобально), а не при каждом запросе.
  - Код должен обрабатывать ошибку FileNotFoundError, если модель не найдена.
## 5. Code Style (Стиль кода)
1. **Type Hints:** Обязательное использование аннотаций типов во всех функциях.
2. **Structure:**
  - main.py — только роутинг и запуск.
  - schemas.py — только Pydantic модели.
  - Логика предсказания вынесена в отдельный сервисный класс или функцию (не писать всё внутри функции роута).
## Forbidden (Запрещено)
  - Использовать from typing import List без необходимости (использовать list из Python 3.9+).
  - Выполнять тяжелые вычисления или масштабирование данных внутри эндпоинта (это должно быть сделано в model.pkl).
  - Возвращать числовой PMI без текстового статуса (expansion/recession).
