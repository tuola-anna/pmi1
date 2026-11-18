# 01_data_generation.md

## Step 1: Источники данных

Для прогнозирования индекса PMI используются синтетические макроэкономические данные, имитирующие реальные распределения.

*Rationale:* Синтетические данные позволяют контролировать распределение, добавлять известные паттерны и тестировать модель в контролируемых условиях без зависимости от внешних API.

## Step 2: Ключевые признаки

Генерируются следующие макроэкономические индикаторы:

| Признак | Описание | Распределение |
|---------|----------|---------------|
| `inflation` | Уровень инфляции (%) | Нормальное с выбросами |
| `key_rate` | Ключевая ставка ЦБ (%) | Нормальное |
| `oil_price` | Цена нефти (USD/barrel) | Лог-нормальное |
| `gdp_growth` | Рост ВВП (%) | Нормальное |
| `unemployment` | Уровень безработицы (%) | Нормальное |
| `consumer_confidence` | Индекс потребительского доверия | Равномерное [0, 100] |

*Rationale:* Эти признаки выбраны на основе экономической теории — они напрямую влияют на деловую активность и, следовательно, на PMI.

## Step 3: Формула PMI

Индекс PMI рассчитывается как взвешенная комбинация признаков с добавлением пороговых эффектов:

```python
import numpy as np

def generate_pmi_data(n_samples=10000):
    np.random.seed(42)
    
    data = {
        'inflation': np.random.normal(4.0, 1.5, n_samples),
        'key_rate': np.random.normal(7.5, 2.0, n_samples),
        'oil_price': np.random.lognormal(4.5, 0.5, n_samples),
        'gdp_growth': np.random.normal(2.0, 1.0, n_samples),
        'unemployment': np.random.normal(5.0, 1.5, n_samples),
        'consumer_confidence': np.random.uniform(30, 100, n_samples)
    }
    
    # Базовая линейная комбинация
    pmi = (
        45 +
        0.5 * data['gdp_growth'] -
        0.8 * data['inflation'] +
        0.3 * data['key_rate'] -
        1.2 * data['unemployment'] +
        0.05 * data['consumer_confidence'] +
        0.02 * data['oil_price']
    )
    
    # Добавление нелинейных пороговых эффектов
    high_inflation_mask = data['inflation'] > 5.0
    pmi[high_inflation_mask] -= 2.0
    
    data['pmi'] = pmi
    
    return pd.DataFrame(data)
```

*Rationale:* Формула включает как линейные эффекты (рост ВВП повышает PMI, инфляция и безработица снижают), так и нелинейный пороговый эффект — при инфляции выше 5% деловая активность падает сильнее.

## Step 4: Валидация данных

После генерации проверяются:
1. Распределение каждого признака (графики гистограмм)
2. Корреляционная матрица признаков
3. Диапазон значений PMI (должен быть в пределах [20, 80])

```python
import matplotlib.pyplot as plt
import seaborn as sns

df = generate_pmi_data()

# Проверка распределения
fig, axes = plt.subplots(2, 3, figsize=(15, 10))
for idx, col in enumerate(df.columns[:-1]):
    row, col_idx = idx // 3, idx % 3
    axes[row, col_idx].hist(df[col], bins=50, edgecolor='black')
    axes[row, col_idx].set_title(f'Distribution of {col}')

plt.tight_layout()
plt.savefig('reports/distributions.png', dpi=150)
```

![Распределения признаков](../reports/distributions.png)

## Step 5: Сохранение данных

Сгенерированные данные сохраняются в двух форматах:
- `data/raw/train.csv` — полный датасет для обучения
- `data/raw/test.csv` — тестовая выборка (20% данных)

```python
train_size = int(0.8 * len(df))
train_df = df.iloc[:train_size]
test_df = df.iloc[train_size:]

train_df.to_csv('data/raw/train.csv', index=False)
test_df.to_csv('data/raw/test.csv', index=False)
```
