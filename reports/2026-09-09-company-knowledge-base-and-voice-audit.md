# 2026-09-09 — Компания знает всё о себе: база знаний, голос, модели

**Цель сессии:** сделать так, чтобы все LLM-системы компании (ЕваБот, Консилиум, ролевые
агенты) полностью знали о компании: серверы, сервисы, домены, производство, материалы,
бизнес, LLM-агентов и проекты на Google Cloud. Дополнительно — диагностика голосовой
озвучки и проблема зависания чата при `model: auto`.

---

## 1. Диагностика (исходное состояние)

| Проблема | Причина |
|---|---|
| Бот «не знает» свои домены/серверы/модели | RAG вливал в промпт всего **3 документа** (`kbConnector.search(... limit: 3)`), а статическая `companyDatabase` в `src/core/CorporateRoles.ts` содержала **выдуманные факты** |
| Модель повторяла «n8n», «eva-line.com», «Kubernetes», «Vault», «Братислава Obchodna 37», «Qdrant/PostgreSQL» | Зашитая строка `SystemContext.build()` («…n8n (docker)») + 7 фейковых статических корпоративных документов |
| Чат «висал» при `model: auto` на пустом тексте | `requestedModel = model` передавался как `'auto'` → провайдер игнорировал/зависал |
| Озвучка резала длинные ответы | Чтение всего текста одним блоком без сегментации и без детекта языка |
| `business.evaline.online` — HTTPS не работал | Управляемый сертификат `business-evaline-online-cert` в статусе `FAILED_NOT_VISIBLE` |

## 2. Изменения

### 2.1 Голос — `public/index.html` (EvaBot / “voice”)
- `detectLang(text)` — детект языка по тексту: кириллица + `і/ї/є/ґ` → `uk-UA`; кириллица → `ru-RU`; латиница → `en-US`.
- `splitSpeech(cleanText, max = 340)` — нарезка на фрагменты по границам предложений.
- `cloudSpeak(chunk, persona, lang)` — озвучка фрагмента.
- `speak(text)` — очередь фрагментов через счётчик `speakSeq` (гарантирует последовательное озвучивание без наложений).
- `speakBrowser(chunk, lang)` — фолбэк на `speechSynthesis` при недоступности облачного TTS.
- `setTts(b)` при выключении — сброс `speakSeq` и остановка текущего `activeAudio`.

### 2.2 Модели — `src/server/routes/ChatRouter.ts`
- Оба эндпоинта `/api/chat` и `/api/chat/stream`:
  `requestedModel = model && model !== 'auto' && model !== 'default' ? model : undefined`.
- RAG-контекст: лимит найденных документов **3 → 6**.

### 2.3 База знаний компании
Создано **6 сводных документов** в `/home/evabot/evaline-online/docs/company/`
(репозиторий `evaline-online/evaline-online`), которые автоматически подхватываются
`KnowledgeBase` при старте `evabot-brain` (память + SQLite FTS5, идемпотентно):

| Файл | Содержание |
|---|---|
| `company-business.ru.md` | ТОВ «ЕВА-ЛАЙН»: ЄДРПОУ 40484497, Чорноморск, EVA-пена, 23 SKU, свойства материала |
| `company-operations.en.md` | Производство, сервисы, LLM-агенты, база знаний (EN-сводка) |
| `infrastructure.ru.md` / `infrastructure.en.md` | GCP: проекты, регионы, ВМ, systemd-сервисы, OmniRoute, MCP/LSP |
| `llm-ai-agents.ru.md` | Флот моделей, провайдеры, политика `ModelRatings`, устройство KB |
| `services-domains.ru.md` | Все домены и API-эндпоинты |

Результат после рестарта: FTS **1482 → 1530 чанков** (+48 company), поиск `/api/kb/search`
ранжирует новые документы первыми.

### 2.4 Очистка «фейковых» фактов
- `src/core/CorporateRoles.ts`: статическая `companyDatabase` переписана на проверенные
  факты (убраны `eva-line.com`, Братислава, Kubernetes, Vault, Qdrant, PostgreSQL, UNIC, «тіг-motors» и т.п.).
- `src/core/SystemContext.ts`: строка сервисов исправлена — убран `n8n (docker)`, добавлены
  `evabot-face :8093`, реальные слова и **строка со всеми доменами компании**.

### 2.5 Инфраструктура — `business.evaline.online`
- HTTPS-прокси `business-https-proxy` переведён на сертификат `business-evaline-online-cert-2`
  (был `FAILED_NOT_VISIBLE`, старый удалён из прокси).
- После дозревания сертификата (PROVISIONING → активен) HTTPS отвечает **200**.
- Подтверждён биллинг: проект `evabot-agent-server` → аккаунт `016725-23E254-FD499D`, включён.
- Топология: `business.evaline.online` → GCP HTTPS-LB `34.49.122.75` → микро-ВМ
  `evaline-micro-vm` (instance-group), НЕ Cloud Run (бэкенд LB — группа экземпляров);
  B2B API отдельно доступен на Cloud Run (`business-tier-api`).

## 3. Проверка (после фиксов)

- `model: auto` и `model: default` — отвечают (~24 с авто, ~13 с default), не виснут.
- Диалог-тест ЕваБота (4 unary + 1 stream через `openrouter/free`): знания о компании и
  EVA-материалах — верные; логика — верная; язык ответа = языку вопроса; stream — 152 SSE-чанка.
- «Какие домены?» → все 6 (evabot.online, evaline.online, evaline.network, evaline.com.ua,
  business.evaline.online, Cloud Run B2B) ✔
- «Серверы/сервисы?» → evabot-agent-vm (Frankfurt, c3-standard-8), evaline-micro-vm
  (us-central1, 136.114.26.252), порты, OmniRoute, без «n8n» ✔
- «Модели/OmniRoute?» → openrouter/free, роутер :20128 ✔
- Голос: фикс на публичном сайте (6 совпадений `splitSpeech/cloudSpeak/detectLang`).

## 4. Лимиты и политика моделей
- TTS: ~834 симв. из 900 000/мес — лимит далеко не исчерпан; голоса готовы.
- `/api/cost` и `/api/stt/usage` — **404** (эндпоинты не существуют) → потенциальный план работ.
- Фрифлот OpenRouter сбрасывается по UTC; глюки 08.09 были массовыми 429 у всех провайдеров.
- `ModelRatings`: Gemini (3.x/2.x) зарезервирован **только для разработки**. Рекомендация —
  добавить Gemini как **аварийный фолбэк** рантайма (политическое решение, не выполнено).

## 5. Как добавлять знания в будущем
1. Положить `.md` в `/home/evabot/evaline-online/docs/` → `sudo systemctl restart evabot-brain`
   (память + FTS обновятся автоматически).
2. Через чат: `/add db <название> | <текст>`.
3. Веб-URL: `POST /api/kb/link {"url": "..."}`.

## 6. Открытые вопросы
- **Telegram**: канал выключен, нужен токен бота.
- **«Промо-коды»**: в коде отсутствуют; интерпретация — free-модели/ключи (уточнить у владельца).
- **Расходы GCP**: точная сумма через gcloud недоступна (нужен BigQuery-экспорт/консоль).
- **Еva Online бренд (Eva Bot)**: бот честно отвечает «нет данных» на вопросы о собственном
  продукте — стоит добавить документ об экосистеме Eva Bot/Evaline в KB.