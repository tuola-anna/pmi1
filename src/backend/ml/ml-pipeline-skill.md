---
name: ML Pipeline Skill
description: Правила построения ML-модели и производственного пайплайна
---
# ML Pipeline Skill
## Goal (Цель)
Обучение модели `CatBoost` с использованием `sklearn Pipeline` для обеспечения воспроизводимости (Reproducibility) и защиты от утечки данных (Data Leakage). 
Обеспечение интерпретируемости модели для аналитиков через визуализацию важности признаков.
## Dependency Management (Управление зависимостями)
1.  **Файл:** Использовать `requirements.txt`.
2.  **Актуальность:** Запрещено писать версии библиотек "из головы". 
    *   *Как правильно:* Активация `venv` -> `pip install package` -> `pip freeze > requirements.txt`.
3.  **Разделение:** 
    *   Для разработки (линтеры, тесты) используется `requirements-dev.txt`.
    *   В `requirements.txt` для продакшена должны быть только библиотеки для запуска: `fastapi`, `uvicorn`, `catboost`, `joblib`, `pydantic`, `numpy`, `pandas`.
4.  **Запрет:** Не использовать `poetry` или `pipenv`. Стандарт — чистый `requirements.txt`.

## Modeling Strategy
1.  **Алгоритм:** `CatBoostRegressor`.
2.  **Метрики:** `RMSE`, `MAE`, `MAPE`.
3.  **Интерпретируемость:** Обязательное построение графика `SHAP values` (beeswarm plot) и сохранение его в папку `reports/`.
## Sklearn Pipeline Structure
Весь процесс должен быть обернут в `sklearn.pipeline.Pipeline`. 
**Важно:** После обучения пайплайна он должен быть сохранен целиком в файл `model.pkl` с помощью `joblib`. Это необходимо для того, чтобы бэкенд мог загрузить и скейлер, и модель одним файлом.
### Feature Engineering (Обязательно)
1.  **Scaling:** Использовать только `RobustScaler` (защита от выбросов в макроданных).
2.  **Interaction:** Обязательно создать новый признак (Feature Interaction), умножающий инфляцию на ключевую ставку (`inflation * key_rate`). Это математический способ объяснить модели пороговый эффект.
### Implementation (Пример реализации)
Агент должен использовать следующую структуру в файле `src/backend/ml/train.py`. 
*Примечание: FeatureInteraction — это кастомный класс, который агент должен написать сам, если его нет.*
```python
import joblib
import pandas as pd
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import RobustScaler
from sklearn.base import BaseEstimator, TransformerMixin
import numpy as np

# 1. Кастомный трансформер для создания взаимодействия (Interaction)
class FeatureInteraction(BaseEstimator, TransformerMixin):
    def __init__(self, col1: str, col2: str):
        self.col1 = col1
        self.col2 = col2
        def fit(self, X, y=None):
        return self
        def transform(self, X):
        # Создаем новую колонку как произведение
        interaction = X[self.col1] * X[self.col2]
        return pd.concat([X, interaction.rename(f"{self.col1}_x_{self.col2}")], axis=1)
# 2. Сборка пайплайна
pmis_pipeline = Pipeline([
    ('scaler', RobustScaler()),                     # Сначала масштабируем
    ('interaction', FeatureInteraction('inflation', 'key_rate')), # Затем создаем признак
    ('regressor', CatBoostRegressor(
        verbose=0, 
        random_state=42,
        allow_writing_files=False
    ))
])
```
# 3. Обучение (X_train — это DataFrame)
```python
pmis_pipeline.fit(X_train, y_train)
```
# 4. СОХРАНЕНИЕ (Обязательный шаг)
## Сохраняем весь объект Pipeline целиком

**Место сохранения модели** data/models/


```python
joblib.dump(pmis_pipeline, "model.pkl")
print("Model pipeline successfully saved to model.pkl")
```
# 5. Интерпретация (Пример)
```python
import shap
explainer = shap.Explainer(pmis_pipeline.named_steps['regressor'])
shap_values = explainer(X_train)
shap.plots.beeswarm(shap_values, show=False)
plt.savefig('reports/shap_plot.png')
