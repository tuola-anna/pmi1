---
name: Data Generation Skill
description: Генерация синтетического макроэкономического датасета с нелинейностями и лагами
---

# Data Generation Skill

## Goal (Цель)
Создание файла `data/raw/macro.csv`, содержащего синтетические данные о PMI. 
Данные должны имитировать реальную экономику: быть векторизованными, содержать лаги (предыдущие значения) и жесткие пороговые эффекты (нелинейность), при которых экономика реагирует на изменения ставок по-разному в зависимости от уровня инфляции.

## Constraints (Ограничения)
1.  **Инструменты:** Только `numpy` и `pandas`. Циклы `for` и `while` для генерации строк **категорически запрещены**.
2. **Объем (Size):** Генерация данных минимум на **36 месяцев** от текущей даты назад с шагом один месяц. Рекомендуется 60-120 для качественного обучения.
3.  **Диапазоны (Uniform Distribution):**
    *   `inflation`: [2.0, 15.0]
    *   `key_rate`: [7.0, 21.0]
    *   `oil_price`: [40.0, 100.0]
    *   `consumer_confidence`: [80.0, 120.0]
4.  **Пороговые эффекты (Non-linearity):**
    *   **Влияние ставки:** Если `inflation <= 10`, коэффициент влияния `key_rate` равен `-0.5`. Если `inflation > 10`, коэффициент становится `-1.0` (эффект удваивается).
    *   **Шок нефти:** Если `oil_price < 50`, из итогового PMI вычитается дополнительно `3.0` пункта.
5.  **Лаги (Lags):**
    *   `inflation_lag1`: значение `inflation` из предыдущей строки.
    *   `oil_price_lag1`: значение `oil_price` из предыдущей строки.
    *   *Для первой строки лаги равны значениям текущего месяца.*
6.  **Шум:** Добавить случайный шум к PMI: `+ np.random.normal(0, 1.5)`.
7.  **Ограничение (Clipping):** Финальное значение PMI должно быть строго в пределах `[0, 100]`.
8. **Скрипт генерации данных** должен иметь свой собственный файл зависимостей.
    - В папке scripts/ создай файл requirements.txt.
    - В него должны входить: numpy, pandas.
    - Перед запуском скрипта generate_data.py агент обязан выполнить установку зависимостей именно из этого файла.

## Formula (Формула расчета)
Базовое значение PMI рассчитывается линейно, затем применяются пороги:

`base = 55`
`term_inf = -0.8 * max(0, inflation - 5)`
`term_rate = coef_rate * (key_rate - 10)`  *(где coef_rate зависит от инфляции)*
`term_oil = 0.04 * (oil_price - 70)`
`term_conf = 0.2 * (consumer_confidence - 100)`

`pmi_raw = base + term_inf + term_rate + term_oil + term_conf + noise`
`pmi_final = np.clip(pmi_raw, 0, 100)`

## Output (Ожидаемый результат)
*   **Путь:** `data/raw/macro.csv`
*   **Тип данных:** `float32` (для экономии памяти).
*   **Порядок колонок:** `date`, `inflation`, `key_rate`, `oil_price`, `consumer_confidence`, `inflation_lag1`, `oil_price_lag1`, `pmi`.
*   **Date:** Индекс или колонка с датой (месяцы), начиная с текущего.

## Implementation Example (Пример реализации)
Агент должен использовать векторизованные операции `pandas`:

```python
import pandas as pd
import numpy as np

n_periods = 120  # Количество месяцев

# 1. Генерация сырых данных (Vectorized)
data = pd.DataFrame({
    'inflation': np.random.uniform(2, 15, n_periods),
    'key_rate': np.random.uniform(7, 21, n_periods),
    'oil_price': np.random.uniform(40, 100, n_periods),
    'consumer_confidence': np.random.uniform(80, 120, n_periods)
})

# 2. Создание лагов (Vectorized)
data['inflation_lag1'] = data['inflation'].shift(1).fillna(data['inflation'])
data['oil_price_lag1'] = data['oil_price'].shift(1).fillna(data['oil_price'])

# 3. Пороговый коэффициент для ставки (Vectorized)
coef_rate = np.where(data['inflation'] > 10, -1.0, -0.5)

# 4. Расчет PMI по частям (Vectorized)
base = 55
term_inf = -0.8 * np.maximum(0, data['inflation'] - 5)
term_rate = coef_rate * (data['key_rate'] - 10)
term_oil = 0.04 * (data['oil_price'] - 70)
term_conf = 0.2 * (data['consumer_confidence'] - 100)
noise = np.random.normal(0, 1.5, n_periods)

pmi_raw = base + term_inf + term_rate + term_oil + term_conf + noise
data['pmi'] = np.clip(pmi_raw, 0, 100)

# 5. Сохранение
data.to_csv('data/raw/macro.csv', float_format='%.3f', index=False)

