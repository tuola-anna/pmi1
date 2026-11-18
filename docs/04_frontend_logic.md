# 04_frontend_logic.md

## Step 1: Архитектура Frontend

Frontend реализован как **Vanilla JS** приложение (HTML + CSS + JS) без фреймворков.

*Rationale:* Для данного проекта не требуется сложное управление состоянием или компонентная архитектура. Vanilla JS обеспечивает минимальный бандл, быструю загрузку и простоту деплоя через статический сервер.

## Step 2: Структура файлов

```
frontend/
├── index.html    # Разметка: слайдеры, карточки результатов, кнопка
├── style.css     # Стили: минимализм, адаптивность
└── script.js     # Логика: обработка слайдеров, запрос к API, отрисовка графика
```

## Step 3: Разметка (index.html)

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PMI Predictor</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <h1>PMI Predictor</h1>
        
        <div class="inputs">
            <div class="slider-group">
                <label>Inflation: <span id="inflation-val">4.0</span>%</label>
                <input type="range" id="inflation" min="0" max="20" step="0.1" value="4.0">
            </div>
            
            <div class="slider-group">
                <label>Key Rate: <span id="key-rate-val">7.5</span>%</label>
                <input type="range" id="key-rate" min="0" max="25" step="0.1" value="7.5">
            </div>
            
            <div class="slider-group">
                <label>Oil Price: <span id="oil-price-val">80</span> USD</label>
                <input type="range" id="oil-price" min="20" max="150" step="1" value="80">
            </div>
            
            <div class="slider-group">
                <label>GDP Growth: <span id="gdp-growth-val">2.0</span>%</label>
                <input type="range" id="gdp-growth" min="-10" max="15" step="0.1" value="2.0">
            </div>
            
            <div class="slider-group">
                <label>Unemployment: <span id="unemployment-val">5.0</span>%</label>
                <input type="range" id="unemployment" min="0" max="25" step="0.1" value="5.0">
            </div>
            
            <div class="slider-group">
                <label>Consumer Confidence: <span id="consumer-confidence-val">65</span></label>
                <input type="range" id="consumer-confidence" min="0" max="100" step="1" value="65">
            </div>
        </div>
        
        <button id="predict-btn">Predict PMI</button>
        
        <div class="result" id="result" style="display: none;">
            <div class="pmi-value" id="pmi-value">--</div>
            <div class="trend" id="trend">--</div>
            <div class="confidence" id="confidence">--</div>
        </div>
        
        <canvas id="chart"></canvas>
    </div>
    
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="script.js"></script>
</body>
</html>
```

## Step 4: Стили (style.css)

```css
:root {
    --bg-color: #f8f9fa;
    --card-bg: #ffffff;
    --success-color: #28a745;
    --danger-color: #dc3545;
    --text-color: #212529;
}

body {
    font-family: 'Segoe UI', sans-serif;
    background-color: var(--bg-color);
    color: var(--text-color);
    margin: 0;
    padding: 20px;
}

.container {
    max-width: 600px;
    margin: 0 auto;
}

.slider-group {
    margin-bottom: 15px;
    padding: 10px;
    background: var(--card-bg);
    border-radius: 8px;
}

.slider-group label {
    display: block;
    margin-bottom: 5px;
    font-weight: 500;
}

.result {
    margin-top: 20px;
    padding: 20px;
    background: var(--card-bg);
    border-radius: 8px;
    text-align: center;
}

.pmi-value {
    font-size: 48px;
    font-weight: bold;
}

.trend.expansion { color: var(--success-color); }
.trend.contraction { color: var(--danger-color); }
```

*Rationale:* Цветовая палитра соответствует разделу 4 SKILL.md: зелёный для PMI > 50 (экспансия), красный для PMI < 50 (контракция).

## Step 5: Логика (script.js)

```javascript
const API_URL = 'http://localhost:8000';

// Ссылки на элементы
const sliders = document.querySelectorAll('input[type="range"]');
const predictBtn = document.getElementById('predict-btn');
const resultDiv = document.getElementById('result');
const pmiValue = document.getElementById('pmi-value');
const trendDiv = document.getElementById('trend');
const confidenceDiv = document.getElementById('confidence');
const ctx = document.getElementById('chart').getContext('2d');

let chart = null;

// Обновление значений слайдеров в реальном времени
sliders.forEach(slider => {
    const valueSpan = document.getElementById(slider.id + '-val');
    slider.addEventListener('input', () => {
        valueSpan.textContent = slider.value;
    });
});

// Отправка запроса к API
predictBtn.addEventListener('click', async () => {
    const payload = {
        inflation: parseFloat(document.getElementById('inflation').value),
        key_rate: parseFloat(document.getElementById('key-rate').value),
        oil_price: parseFloat(document.getElementById('oil-price').value),
        gdp_growth: parseFloat(document.getElementById('gdp-growth').value),
        unemployment: parseFloat(document.getElementById('unemployment').value),
        consumer_confidence: parseFloat(document.getElementById('consumer-confidence').value)
    };
    
    try {
        const response = await fetch(`${API_URL}/predict`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });
        
        if (!response.ok) {
            throw new Error(`HTTP ${response.status}`);
        }
        
        const data = await response.json();
        
        // Отображение результатов
        resultDiv.style.display = 'block';
        pmiValue.textContent = data.pmi;
        trendDiv.textContent = data.trend === 'expansion' ? 'Expansion' : 'Contraction';
        trendDiv.className = `trend ${data.trend}`;
        confidenceDiv.textContent = `CI: ${data.confidence_interval.lower} - ${data.confidence_interval.upper}`;
        
        // Обновление графика
        updateChart(data);
        
    } catch (error) {
        alert('Ошибка запроса: ' + error.message);
    }
});

// Отрисовка графика с Chart.js
function updateChart(data) {
    if (chart) chart.destroy();
    
    chart = new Chart(ctx, {
        type: 'bar',
        data: {
            labels: ['PMI'],
            datasets: [{
                label: 'Прогноз PMI',
                data: [data.pmi],
                backgroundColor: data.trend === 'expansion' ? '#28a745' : '#dc3545',
                borderColor: data.trend === 'expansion' ? '#1e7e34' : '#bd2130',
                borderWidth: 1
            }]
        },
        options: {
            scales: {
                y: { beginAtZero: false, min: 20, max: 80 }
            },
            plugins: {
                legend: { display: false }
            }
        }
    });
}
```

*Rationale:*
- Запрос к API отправляется по кнопке, а не автоматически — это снижает нагрузку на Backend.
- Chart.js подключается через CDN для простоты (без сборщиков).
- График перестраивается динамически при каждом новом прогнозе.
- Цвет графика меняется в зависимости от тренда (зелёный/красный).

## Step 6: Взаимодействие с API

```
Frontend (порт 3000) → HTTP POST → Backend API (порт 8000/predict)
```

*Rationale:* Frontend и Backend работают на разных портах в Docker-сети. CORS-мидлварен на Backend разрешает запросы с любого origin (для разработки).
