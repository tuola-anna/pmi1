name: pmi-finally
description: Конституция MLOps-проекта PMI. Общие правила архитектуры, структуры и безопасности.
---

# Project Constitution (SKILL.md)

## 1. Goal & Context (Цель и Контекст)
**Цель:** Создание MLOps-сервиса для прогнозирования индекса PMI с учетом пороговых эффектов макроэкономики.
**Архитектура:** Монорепозиторий (Monorepo), разделенный на независимые слои: Data, ML, Backend API, Frontend.
**Платформа:** Docker. Развертывание через `docker-compose`.

## 2. Project Structure (Структура папок) — КРИТИЧНО
Агент обязан строго соблюдать следующую иерархию. Запрещено сваливать всё в один файл или корень.
/project-root 
│
├── .github/ # CI/CD пайплайны 
├── data/ # Только данные (raw/processed)
│ └── raw/
│ └── models/
├── docs/ # Документация (создается согласно documentation-skill.md) 
├── src/
│ ├── backend/ # Python (FastAPI, CatBoost) 
│ │ ├── api/
│ │ |   ├──main.py
│ │ |   ├──predictor.py
│ │ ├── ml/
│ │ |   └── .gitignore
│ │ ├── requirements.txt
└── frontend/ # Vanilla JS (HTML/CSS/JS)
│ ├── index.html
│ ├── style.css
│ └── script.js
├── infrastructure/ # Docker, docker-compose
├── requirements-dev.txt 
├── docker-compose.yml
└── README.md

## 3. Global Constraints (Глобальные ограничения)
Эти правила применяются ко ВСЕМ частям проекта (Backend, Frontend, ML).

*   **Безопасность:** 
    *   Никаких секретов и паролей в коде. Только `.env` файлы.
    *   Никаких хардкодов портов в Python-коде (кроме значений по умолчанию в `.env`). Порты задаются в `docker-compose.yml`.
*   **Стиль кода (Linting):**
    *   Строго `ruff` (наследование Black + Flake8).
    *   Форматирование кода по стандарту PEP 8
    *   Запрет на `wildcard imports` (`from x import *`).
*   **Производительность:** 
    *   Время ответа API `/predict` не более 200ms.
*   **Чистота:**
    *   В репозитории не должно быть временных файлов (`__pycache__`, `.pyc`, `.ipynb_checkpoints`, `.venv`).

## 4. Color Palette & Style (Визуальный стиль)
*   **Стиль:** Минимализм, DataViz.
*   **Палитра:**
    *   Фон: `#f8f9fa`
    *   Карточки: `#ffffff`
    *   Успех (PMI > 50): `#28a745` (Green)
    *   Опасность (PMI < 50): `#dc3545` (Red)
    *   Текст: `#212529`

## 5. Definition of Done (Критерии готовности)
Задача считается выполненной, если соблюдены все пункты:
1.  [ ] **Linting:** Весь код проходит `ruff check` без ошибок.
2.  [ ] **Docs:** В папке `/docs` есть файлы `01...`, `02...` и т.д. согласно `documentation-skill.md`.
3.  [ ] **Infrastructure:** Проект запускается одной командой `docker-compose up --build`.
4.  [ ] **ML:** Файл `model.pkl` создан, график SHAP лежит в `reports/`.
5.  [ ] **Frontend:** Слайдеры работают, график перестраивается динамически.
6.  [ ] **API:** Документация Swagger доступна на `/docs`, ошибка 422 возвращается при неверных данных.

## 6. Skill Usage Protocol (Как использовать навыки)
1.  Сначала прочитай этот файл (`SKILL.md`).
2.  Для написания кода в `src/backend` подключи и закрепи (Pin) `backend-api-skill.md` и `ml-pipeline-skill.md`.
3.  Для написания кода в `src/frontend` подключи и закрепи `frontend_skill.md`.
4.  Для генерации данных подключи `data-generation-skill.md`.
5.  Для описания своих действий следуй `documentation-skill.md`.
