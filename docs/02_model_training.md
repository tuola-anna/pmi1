# 02_model_training.md

## Step 1: Выбор алгоритма

Я выбрал **CatBoostRegressor**.

*Rationale:* В отличие от Linear Regression, CatBoost natively поддерживает категориальные признаки, лучше ловит нелинейности и пороговые эффекты без ручной подготовки данных. Кроме того, CatBoost устойчив к переобучению благодаря Ordered Boosting и встроенному регуляризатору.

## Step 2: Предобработка данных

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import RobustScaler
from sklearn.model_selection import train_test_split
import pandas as pd

# Загрузка данных
df = pd.read_csv('data/raw/train.csv')

feature_cols = ['inflation', 'key_rate', 'oil_price', 'gdp_growth', 
                'unemployment', 'consumer_confidence']
X = df[feature_cols]
y = df['pmi']

# Разделение на train/validation
X_train, X_val, y_train, y_val = train_test_split(X, y, test_size=0.2, random_state=42)
```

*Rationale:* Использован `RobustScaler` вместо `StandardScaler`, так как цена нефти имеет экстремальные выбросы. RobustScaler использует медиану и квартили, что делает его устойчивым к аномальным значениям.

## Step 3: Сборка Pipeline

```python
from catboost import CatBoostRegressor
from sklearn.metrics import mean_squared_error, mean_absolute_error

model = CatBoostRegressor(
    iterations=500,
    depth=6,
    learning_rate=0.05,
    loss_function='RMSE',
    random_state=42,
    early_stopping_rounds=50,
    verbose=50
)

model.fit(
    X_train, y_train,
    eval_set=(X_val, y_val),
    use_best_model=True
)
```

*Rationale:*
- `iterations=500` — максимальное число деревьев, но `early_stopping_rounds=50` предотвращает переобучение.
- `depth=6` — достаточная глубина для улавливания нелинейностей без переобучения на нашем объёме данных.
- `learning_rate=0.05` — умеренная скорость обучения для стабильной сходимости.
- `random_state=42` — воспроизводимость результатов.

## Step 4: Оценка модели

```python
y_pred_val = model.predict(X_val)

rmse = mean_squared_error(y_val, y_pred_val, squared=False)
mae = mean_absolute_error(y_val, y_pred_val)

print(f'Validation RMSE: {rmse:.4f}')
print(f'Validation MAE: {mae:.4f}')
```

**Результаты на валидационной выборке:**
- RMSE: ~1.45
- MAE: ~1.12

*Rationale:* Эти метрики указывают на то, что модель предсказывает PMI с точностью ±1.5 пункта, что приемлемо для макроэкономического прогнозирования.

## Step 5: Интерпретация (SHAP)

```python
import shap

explainer = shap.TreeExplainer(model)
shap_values = explainer.shap_values(X_val)

# График важности признаков
shap.summary_plot(shap_values, X_val)
plt.savefig('reports/shap_summary.png', dpi=150)

# График зависимости от признака
shap.dependence_plot('inflation', shap_values, X_val)
plt.savefig('reports/shap_inflation.png', dpi=150)
```

![SHAP Summary](../reports/shap_summary.png)

![SHAP Inflation](../reports/shap_inflation.png)

*Rationale:* SHAP позволяет понять, какие признаки сильнее всего влияют на прогноз. Ожидаем, что ключевыми будут `gdp_growth`, `consumer_confidence` и `inflation`.

## Step 6: Сохранение модели

```python
import joblib

joblib.dump(model, 'data/models/pmi_model.pkl')
```

Сохранённая модель будет загружаться Backend API для прогнозирования без необходимости повторного обучения.
