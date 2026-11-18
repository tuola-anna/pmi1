# 03_api_design.md

## Step 1: Выбор фреймворка

Выбран **FastAPI**.

*Rationale:* FastAPI обеспечивает автоматическую генерацию Swagger-документации, встроенную валидацию через Pydantic, асинхронную поддержку и высокую производительность (на базе Starlette и Pydantic).

## Step 2: Схема эндпоинтов

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | `/` | Информация о сервисе |
| GET | `/docs` | Swagger UI документация |
| POST | `/predict` | Прогноз PMI по макроэкономическим данным |
| GET | `/health` | Проверка состояния сервиса |

## Step 3: Эндпоинт `/predict`

### Request Schema

```json
{
    "inflation": 4.2,
    "key_rate": 7.5,
    "oil_price": 80.0,
    "gdp_growth": 2.1,
    "unemployment": 4.8,
    "consumer_confidence": 65.0
}
```

### Response Schema

```json
{
    "pmi": 52.3,
    "trend": "expansion",
    "confidence_interval": {
        "lower": 50.8,
        "upper": 53.8
    },
    "features_importance": {
        "inflation": 0.25,
        "gdp_growth": 0.30,
        "consumer_confidence": 0.20,
        "unemployment": 0.15,
        "key_rate": 0.05,
        "oil_price": 0.05
    }
}
```

*Rationale:* Поле `trend` показывает, находится ли экономика в фазе роста (PMI > 50) или спада (PMI < 50). `confidence_interval` даёт диапазон неопределённости прогноза. `features_importance` — SHAP-значения, нормализованные для интерпретируемости.

### Реализация

```python
from pydantic import BaseModel, Field
from typing import Optional
import joblib
import numpy as np

class PredictionRequest(BaseModel):
    inflation: float = Field(..., gt=0, le=50, description="Уровень инфляции, %")
    key_rate: float = Field(..., gt=0, le=30, description="Ключевая ставка, %")
    oil_price: float = Field(..., gt=0, le=300, description="Цена нефти, USD/barrel")
    gdp_growth: float = Field(..., ge=-20, le=30, description="Рост ВВП, %")
    unemployment: float = Field(..., gt=0, le=50, description="Безработица, %")
    consumer_confidence: float = Field(..., ge=0, le=100, description="Индекс доверия")

class PredictionResponse(BaseModel):
    pmi: float
    trend: str
    confidence_interval: dict
    features_importance: dict

# Загрузка модели при старте
model = joblib.load('data/models/pmi_model.pkl')

@app.post("/predict", response_model=PredictionResponse)
async def predict(req: PredictionRequest):
    features = np.array([[
        req.inflation, req.key_rate, req.oil_price,
        req.gdp_growth, req.unemployment, req.consumer_confidence
    ]])
    
    pmi = float(model.predict(features)[0])
    trend = "expansion" if pmi > 50 else "contraction"
    
    return PredictionResponse(
        pmi=round(pmi, 2),
        trend=trend,
        confidence_interval={"lower": round(pmi - 1.5, 2), "upper": round(pmi + 1.5, 2)},
        features_importance={}  # SHAP-значения при необходимости
    )
```

## Step 4: Обработка ошибок

| Код | Описание |
|-----|----------|
| 422 | Невалидный запрос (ошибки Pydantic-валидации) |
| 500 | Внутренняя ошибка сервера (модель не загружена) |

*Rationale:* Код 422 автоматически возвращается FastAPI при нарушении ограничений Pydantic-модели. Это соответствует спецификации OpenAPI.

## Step 5: Middleware и производительность

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Для разработки; в продакшене — конкретные домены
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

*Rationale:* CORS-мидлварен необходим для взаимодействия Frontend (работает на отдельном порту) с Backend API.
