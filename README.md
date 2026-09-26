# KW-7 AI — Корпоративный AI-ассистент

Платформа для взаимодействия с корпоративными системами через чат с AI-агентами. Поддерживает Jira, Confluence, Tempo, анализ данных, визуализацию и управление бизнес-целями.

---

## Содержание

- [Технологический стек](#технологический-стек)
- [Архитектура](#архитектура)
- [Быстрый старт](#быстрый-старт)
- [Конфигурация](#конфигурация)
- [Агенты](#агенты)
- [Интеграции](#интеграции)
- [Визуализация данных](#визуализация-данных)
- [RBAC — управление доступом](#rbac--управление-доступом)
- [Лицензирование и издания](#лицензирование-и-издания)
- [MCP Tool Server](#mcp-tool-server)
- [API](#api)

---

## Технологический стек

| Слой | Технология |
|---|---|
| Backend | Django 5.2, Django REST Framework, SQLite, Python 3.12+ |
| Frontend | Next.js 14, React 18, без TypeScript |
| LLM | GigaChat (global) или OpenAI-compatible local (Ollama / LM Studio) via LangChain |
| Агенты | Кастомный оркестратор + Model Context Protocol (MCP) over SSE |
| Интеграции | Jira, Confluence, Tempo Timesheets |
| Графики | ECharts (echarts-for-react), echarts-gl (3D) |
| Auth | Django Session + RBAC |

---

## Архитектура

```
kw_ai_01/
├── backend/
│   ├── accounts/          # Аутентификация, RBAC, настройки LLM
│   ├── chat/              # Чат-сессии, сообщения, датасеты, отчёты
│   ├── goals/             # Бизнес-цели, задачи, предложения (gap-анализ)
│   ├── orchestration/     # Агентный цикл, реестр агентов, LLM-клиент
│   │   ├── agent_loop.py       # Основной цикл (JSON-in-prompt / native tools)
│   │   ├── agent_registry.py   # Реестр агентов: код → handler + метаданные
│   │   ├── llm_client.py       # LangChain-клиент (GigaChat / OpenAI-compat)
│   │   └── mcp_client.py       # SSE-клиент для MCP Tool Server
│   ├── mcp_plugins/       # Плагины MCP Tool Server (auto-discovery)
│   ├── report_scripts/    # Python-скрипты пользовательских отчётов
│   ├── skills/            # Расширяемые навыки агентов
│   └── mcp_server.py      # Standalone MCP-сервер (порт 8010)
└── frontend/
    └── src/
        ├── app/           # Next.js App Router (page.jsx, layout.jsx)
        └── components/
            ├── ChatMvpScreen.jsx  # Главный компонент: чат + боковая панель
            ├── EChart.jsx         # Интерактивные графики (12 типов)
            ├── DataTable.jsx      # Таблицы для датасетов
            └── MiniChart.jsx      # Компактные графики
```

### Агентный цикл

`run_agent_loop(messages, tools, system, max_iterations)` — генератор SSE-событий.

Два режима:
- **JSON-in-prompt** (GigaChat) — LLM возвращает `{"type": "tool_call"|"final", ...}` в тексте
- **Native tool calling** (OpenAI-compatible) — стандартный function calling

### SSE-события (`POST /api/chat/message/stream/`)

| Событие | Описание |
|---|---|
| `status` | Промежуточный статус обработки |
| `agent_call` | Вызов агента с результатом (`result.chart_data`, `result.preview_rows`, ...) |
| `text_chunk` | Фрагмент ответа LLM |
| `done` | Обработка завершена |
| `error` | Ошибка |

---

## Быстрый старт

### Backend

```bash
cd backend/
python -m venv .venv
# Windows:
.venv\Scripts\activate
# Linux/Mac:
source .venv/bin/activate

pip install -r requirements.txt
cp .env.example .env          # Заполни GIGACHAT_TOKEN или LOCAL_LLM_BASE_URL
python manage.py migrate
python manage.py createsuperuser
python manage.py init_data    # Создаёт группы, политики доступа, профили агентов
python manage.py runserver 0.0.0.0:8005
```

### MCP Tool Server (отдельный терминал)

```bash
cd backend/
python mcp_server.py          # Запускается на порту 8010, авто-находит mcp_plugins/
```

### Frontend

```bash
cd frontend/
npm install
npm run dev                   # http://localhost:3005
```

---

## Конфигурация

### Основные переменные окружения (`.env`)

> ⚠️ **Перед запуском в проде** обязательно задайте `DJANGO_SECRET_KEY` — без него используется небезопасный дефолт `django-insecure-...`, пригодный только для локальной разработки. Сгенерировать: `python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"`.
> ```env
> DJANGO_SECRET_KEY=<случайная строка, храните в секрете>
> ```

> 🌐 **За обратным прокси на публичном домене (HTTPS).** Django сверяет CSRF по заголовку `Origin` ТОЧНО — схема + хост + **порт**. Чтобы работала админка и любые формы по такому домену:
> - `DJANGO_ALLOWED_HOSTS` — **хост без схемы** (`ai.kw-7.ru`);
> - `CORS_EXTRA_ORIGINS` — **точный origin со схемой** (`https://ai.kw-7.ru`); эта переменная кормит и CORS, и `CSRF_TRUSTED_ORIGINS`. Порт обязан совпадать с тем, что шлёт браузер: `https://ai.kw-7.ru` (443) и `https://ai.kw-7.ru:8443` — это РАЗНЫЕ origin. Несовпадение → «Ошибка проверки CSRF» (403) при сохранении.
> ```env
> DJANGO_ALLOWED_HOSTS=ai.kw-7.ru,localhost,127.0.0.1
> CORS_EXTRA_ORIGINS=https://ai.kw-7.ru
> ```
> Правку `.env` применяйте пересозданием контейнера (`docker compose up -d backend`), а НЕ `docker restart`: `env_file` читается при создании контейнера, а `load_dotenv` не переопределяет уже заданные переменные.

**GigaChat (глобальный режим):**
```env
GIGACHAT_TOKEN=your_credentials_here
GIGACHAT_MODEL=GigaChat
```

**Локальная модель (Ollama / LM Studio):**
```env
LOCAL_LLM_BASE_URL=http://localhost:11434
LOCAL_LLM_MODEL=llama3.1
LOCAL_LLM_IS_OLLAMA=true
LOCAL_LLM_USE_NATIVE_TOOLS=true
LOCAL_LLM_TIMEOUT=120
LOCAL_LLM_NUM_CTX=8192
```

**Jira:**
```env
JIRA_BASE_URL=https://your-company.atlassian.net
JIRA_API_TOKEN=your_token
JIRA_USERNAME=service@company.com
JIRA_DEFAULT_PROJECT_KEY=PROJ
JIRA_DATASET_FETCH_LIMIT=200
```

**Confluence:**
```env
CONFLUENCE_BASE_URL=https://your-company.atlassian.net/wiki
CONFLUENCE_API_TOKEN=your_token
CONFLUENCE_USERNAME=service@company.com
CONFLUENCE_SPACE_KEYS=SPACE1,SPACE2
```

**MCP Server:**
```env
MCP_SERVER_URL=http://localhost:8010/sse
MCP_PORT=8010
```

Режим LLM (global / local) переключается через интерфейс администратора или вкладку **Агенты**.

---

## Агенты

Реестр агентов определён в `orchestration/agent_registry.py`. Каждый агент — это `{description, input_schema, handler}`.

### Встроенные агенты

| Агент | Назначение |
|---|---|
| `agent_calculator` | Арифметика, функции (sqrt, sin, log), финансовое округление |
| `agent_time_now` | Текущая дата/время, относительные смещения, временные зоны |
| `agent_jira` | Поиск задач Jira, JQL-запросы, фильтрация по статусу/исполнителю/периоду |
| `agent_tempo` | Отчёты Tempo Timesheets: по дням, по сотрудникам, по типам работ; выгрузка Excel |
| `agent_chart` | Интерактивные графики 12 типов (включая 3D) из датасетов или inline-данных |
| `agent_load_dataset` | Загрузка CSV/Excel файлов в сессию (до 5 слотов) |
| `agent_analyze_dataset` | Статистика, корреляция, фильтрация, группировка датасетов |
| `agent_save_analysis` | Сохранение результатов в CSV, JSON, Excel, TXT |
| `agent_dataset_manager` | Управление сохранёнными датасетами: list, load, delete, rename |
| `agent_confluence` | Поиск и чтение страниц Confluence |
| `agent_deadline_check` | Просроченные и срочные цели/задачи |
| `agent_create_task` | Создание операционных задач, привязка к бизнес-целям |
| `agent_create_proposal` | Gap-анализ: фиксация недостающих инструментов, генерация ТЗ + MCP-схемы |
| `agent_psychology` | Коуч: мотивация, уверенность, работа с установками (КПТ + позитивная психология) |
| `agent_developer` | Генерация кода: report-скрипты, MCP-плагины, обработчики агентов |
| `agent_create_report` | Сохранение методологии анализа как переиспользуемой кнопки отчёта |
| `agent_exec_brief` | Executive dashboard: цели, задачи Jira, загрузка команды |
| `agent_team_workload` | Продуктивность команды: часы Tempo + закрытые задачи Jira + утилизация % |
| `agent_project_health` | Индекс здоровья проекта (0–100), блокеры, риски |

### Добавление нового агента

```python
# orchestration/agent_registry.py
def _my_handler(arguments: dict) -> dict:
    return {"answer": "Результат", "my_data": ...}

AGENT_REGISTRY["agent_my_tool"] = {
    "description": "Описание для LLM",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "Запрос"}
        },
        "required": ["query"],
    },
    "handler": _my_handler,
}
```

---

## Интеграции

### Jira
- Поиск по JQL, фильтрация по статусу (open / closed / all), исполнителю, дате
- Авто-разбор русских названий месяцев для фильтрации по периоду
- HTTP 400 fallback с безопасным запросом

### Confluence
- Поиск по ключевым словам, получение страниц по ID или заголовку
- Дерево страниц, просмотр пространств

### Tempo Timesheets
- Отчёты: `daily_by_user` (по дням), `summary` (итоги), `user_detail` (журнал)
- Авто-сохранение в сессию → `agent_chart` читает без передачи данных вручную
- Выгрузка в Excel

---

## Визуализация данных

Компонент `EChart.jsx` поддерживает 12 типов графиков:

| Тип | Описание |
|---|---|
| `bar` | Столбчатая диаграмма |
| `line` | Линейный график |
| `pie` | Круговая диаграмма |
| `histogram` | Гистограмма распределения |
| `scatter` | Точечный график (X/Y), мультисерийный |
| `heatmap` | Тепловая карта (2D-сетка) |
| `radar` | Радарная диаграмма (KPI / паутина) |
| `bar_stacked` | Составная столбчатая |
| `bar_grouped` | Сгруппированная столбчатая |
| `line_multi` | Несколько линий |
| `scatter3D` | 3D точечный (WebGL) |
| `bar3D` | 3D столбчатый — две категории × числовое значение (WebGL) |

### Источники данных для графиков

1. **Сессионный датасет** — загруженный файл или отчёт Tempo (приоритет)
2. **Слот датасета** — `data_slot=N` для загрузки сохранённого датасета
3. **Inline-данные** — список объектов прямо в вызове агента (только если нет сессии)

### Особенности

- `auto` — автовыбор типа по структуре данных и числу точек
- Широкоформатные DataFrame (Tempo `daily_by_user`) автоматически преобразуются для heatmap через melt
- Fuzzy-matching имён колонок (регистр, частичное совпадение, русский/английский)
- Нормализация алиасов: `"объёмный"`, `"three_axes"`, `"3d"` → `bar3D`

---

## RBAC — управление доступом

### Роли (создаются командой `python manage.py init_data`)

| Группа | Доступные агенты | Управление целями |
|---|---|---|
| Администратор | Все агенты | Да |
| Инженер | Calculator, Jira, Time, Tasks, Proposal, Psychology, DeadlineCheck | Нет |
| Инженер-Разработчик KW | Как Инженер + Developer (право разработки кода, `can_develop=True`) | Нет |
| Бизнес-аналитик | Calculator, Time, Tasks, Proposal, Psychology, DeadlineCheck | Да |

### Реализация

- `GroupAccessPolicy` — Django Group → список агентов + флаги `can_manage_goals`, `can_develop` (генерация/тест/деплой кода плагинов — проверяется и внутри `agent_developer`)
- `allowed_agents(user)` в `accounts/access.py` — QuerySet AgentProfile по группам пользователя
- Настраивается через Django Admin без изменения кода

---

## Лицензирование и издания

Система поставляется в трёх изданиях, привязанных к оборудованию клиента через офлайн-активацию (интернет у клиента не требуется).

| Издание | Функциональность |
|---|---|
| **kw-7 lite** | Локальные/глобальные модели + ПДН + 1–2 демо-агента |
| **kw-7 corp** | Все агенты, минимальная разработка плагинов (тест/доводка готовых, без деплоя новых) |
| **kw-7 global** | Максимум возможностей, включая генерацию и деплой новых плагинов |

### Запуск из открытого репозитория (без лицензии)

Клон этого репозитория запускается **без ключа и без ограничений**: публичный ключ не задан (`KW_LICENSE_PUBKEY` пуст) → сборка считается dev → активное издание **`global`**, все функции открыты. Файл `license.key` в этом режиме не требуется и даже не читается.

Проверка лицензии (валидация `license.key`, fail-closed до `lite`) включается **только в лицензированной сборке**, где публичный ключ вшит в Cython-ядро `licensing/_core*.so`. Открытый Python-исходник намеренно **не содержит enforcement** — в нём издание всегда сводимо к `global` правкой окружения. Это осознанное решение: опенсорс = свободный запуск, лицензирование = отдельная компилируемая поставка.

### Как получить ключ (только для лицензированной сборки)

1. На машине с лицензированной сборкой выполните:
   ```bash
   python manage.py license_status
   ```
2. Скопируйте блок **«КОД ЗАПРОСА»** (это только хеши компонент железа, не серийники) и отправьте поставщику: **info@kw-7.ru** (GitHub Issues — будет добавлено позже).
3. В ответ придёт подписанный `license.key` под ваше железо и издание — положите его по пути `KW_LICENSE_FILE` (по умолчанию рядом с backend) и перезапустите backend.
4. Проверка: `python manage.py license_status` покажет «Активное издание: …».

### Как это работает

1. **Издание — параметр, а не отдельная сборка.** Активное издание определяет `licensing.get_edition()`; проверка фич — в одной точке `licensing.feature_allowed(feature)`. Издание только **сужает**: эффективное право = настройка администратора **И** разрешение издания (лицензия никогда не расширяет).
2. **Привязка к железу.** `manage.py license_status` на машине клиента печатает **код запроса** — композитный отпечаток оборудования (board serial, product UUID, machine-id, MAC; наружу идут только хеши). Сверка **k-из-n**: замена одной детали не аннулирует лицензию.
3. **Подписанный `license.key`.** Поставщик по коду запроса выпускает лицензию (издание + срок + привязка к железу), подписанную Ed25519. Клиент кладёт её в `license.key`; при старте система проверяет подпись → срок → отпечаток. Подделка / чужое железо / истёкший срок → **fail-closed** до `lite`.
4. **Cython-ядро.** Критичная проверка (подпись + вшитый публичный ключ) компилируется в бинарь `licensing/_core*.so` — правкой Python её не обойти.
5. **Сервер выдачи.** Приложение `license_server` (админка выдач) работает только на стороне поставщика — включается `KW_LICENSE_ISSUER=True`; приватный ключ подписи из `KW_LICENSE_PRIVATE_KEY_FILE`, никогда не в клиентском образе.

### Переменные окружения

| Переменная | Назначение |
|---|---|
| `KW_LICENSE_PUBKEY` | Публичный ключ (base64). Задан = лицензированная сборка (требует валидный `license.key`); в лицензированном образе вшивается в Cython-ядро |
| `KW_LICENSE_FILE` | Путь к `license.key` (по умолчанию `license.key` рядом с backend) |
| `KW_EDITION` | Dev-override издания (только когда публичный ключ не задан) |
| `KW_LICENSE_ISSUER` | `true` только на инстансе выдачи поставщика — включает админку выдач |
| `KW_LICENSE_PRIVATE_KEY_FILE` | Приватный ключ подписи (только issuer-инстанс) |

> Модель угроз — B2B: цель в том, чтобы честный клиент не мог случайно развернуть вторую копию, а намеренный обход был очевиден и документируем (Cython = обфускация, не крипто-крепость).

---

## MCP Tool Server

Standalone-сервер на порту 8010, реализующий Model Context Protocol поверх SSE.

### Запуск

```bash
cd backend/
python mcp_server.py
```

### Создание плагина

Файл `mcp_plugins/my_plugin.py`:

```python
TOOL_MANIFEST = {
    "name": "my_tool",
    "description": "Описание инструмента",
    "inputSchema": {
        "type": "object",
        "properties": {
            "query": {"type": "string"}
        },
        "required": ["query"]
    }
}

def handler(arguments: dict) -> dict:
    return {"result": f"Ответ на: {arguments['query']}"}
```

- Плагины авто-обнаруживаются при старте сервера
- Новые плагины (не зарегистрированные в БД) доступны всем пользователям
- Зарегистрированные в `AgentProfile` → подчиняются RBAC

### Gap-анализ → автогенерация ТЗ

Агент `agent_create_proposal` фиксирует отсутствующий инструмент и автоматически генерирует:
- Описание проблемы и бизнес-ценности
- Полное ТЗ (tech_spec) с готовым шаблоном MCP-плагина
- Отображается во вкладке **Аналитика** → кнопка «📄 ТЗ + MCP-схема»

---

## API

### Аутентификация

Session-based. Логин:

```http
POST /api/accounts/login/
Content-Type: application/json

{"username": "user", "password": "pass"}
```

### Чат (SSE-стриминг)

```http
POST /api/chat/message/stream/
Content-Type: application/json

{"message": "Покажи задачи Jira за апрель"}
```

Ответ — поток SSE:
```
data: {"type": "status", "text": "Вызываю agent_jira..."}
data: {"type": "agent_call", "agentCode": "agent_jira", "result": {...}}
data: {"type": "text_chunk", "text": "Вот задачи за апрель:"}
data: {"type": "done"}
```

### Ключевые endpoints

```
GET  /api/health/                     — статус бэкенда и LLM
GET  /api/accounts/me/               — текущий пользователь + права
GET  /api/accounts/agents/           — доступные агенты
POST /api/chat/session/reset/        — сброс истории чата
POST /api/chat/attachments/          — загрузка файла (CSV/Excel)
GET  /api/goals/                     — список бизнес-целей
GET  /api/goals/proposals/           — предложения gap-анализа
POST /api/goals/proposals/<id>/generate/ — генерация кода плагина (право can_develop)
POST /api/goals/proposals/<id>/refine/   — автодоводка кода (SSE, право can_develop)
POST /api/goals/proposals/<id>/test/     — тест плагина + диагностика (право can_develop)
POST /api/goals/proposals/<id>/deploy/   — деплой в mcp_plugins/ (право can_develop)
```

---

## Разработка

### Конвейер разработки MCP-плагина из предложения

Весь цикл — из вкладки **Агенты** → «Предложения» (нужно право «Разработка кода»: группа «Инженер-Разработчик KW» или администратор):

1. **⚙ Сгенерировать плагин** — LLM пишет код по gap-анализу и требованиям (единый промпт с правилами среды: конфигурация через env, без вебхуков, таймауты)
2. **🤖 Автодоводка** — автономный цикл «проверить → исправить → перепроверить» (до 3 итераций): статические проверки, песочница с реалистичными подставными ответами API (незнакомые API выучиваются один раз LLM-синтезом → админка «База знаний интеграций»), LLM-ревью; итоговый отчёт
3. **🧪 Тестировать** — запуск в песочнице до деплоя (после деплоя — живой вызов через MCP); система сама комментирует результат: вердикт (корректно / баг кода / проблема окружения) + предложения
4. **🚀 Подключить** — запись в `backend/mcp_plugins/` + горячая перезагрузка MCP-сервера

Тот же цикл доступен из чата: `agent_developer` (`action=save_plugin`, флаги `regenerate`/`deploy`).

Ручной путь по-прежнему работает: «📄 ТЗ + MCP-схема» → сохранить как `backend/mcp_plugins/<name>.py` → перезапустить `mcp_server.py`.

### Структура датасетов

- До 5 слотов на пользователя (FIFO по `updated_at`)
- Повторная загрузка того же файла → обновляет существующий слот
- Сессионный датасет (in-memory) → обновляется при каждой загрузке файла или запуске Tempo

### Переменные окружения — полный список

Смотри `backend/.env.example`.
