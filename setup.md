# setup.md — установка и запуск комплекта

Профиль работает в двух вариантах браузера. Можно включить любой (или оба —
AGENTS.md не запрещает; главный принцип — работать неспешно любым инструментом).

---

## Шаг 0. Установка Hermes Agent

```bash
# Linux / macOS / WSL2
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
source ~/.bashrc
hermes doctor          # проверить, что всё живо
hermes model           # выбрать модель (или hermes setup --portal — всё сразу через Nous Portal)
```

## Шаг 1. Личность и рабочая папка

```bash
mkdir -p ~/avito-patrol
cp SOUL.md   ~/.hermes/SOUL.md          # личность «Витя» — глобально
cp AGENTS.md ~/avito-patrol/AGENTS.md   # рабочий регламент
mkdir -p ~/.hermes/skills
cp -r skills/avito-patrol ~/.hermes/skills/   # навык патруля
```

В `~/avito-patrol` Гермес будет вести `sellers.md`, `sellers.csv`, `diary.md`
(крон-задание запускается с `--workdir ~/avito-patrol`, чтобы AGENTS.md подхватился).

## Шаг 2A. Вариант «встроенные инструменты Гермеса»

У Гермеса уже есть браузерный toolset — `browser_exec` (Chromium через CDP,
со скриншотами), плюс `browser_cdp`, `browser_dialog`, `browser_vault`.

```bash
hermes tools                          # включить toolset: browser
# опционально — «невидимый» браузер из каталога плагинов:
hermes plugins install camofox        # Camofox — менее приметный браузер для автоматизации
```

Плюсы: работает headless на VPS, не требует вашего браузера.
Минусы: свой Chromium — для Авито это «чистый» профиль без истории; заходить
так безопаснее НЕ под личным аккаунтом, либо держать отдельный профиль.

## Шаг 2B. Вариант «расширение в реальном браузере»

Это browse-сервер (MCP + расширение в вашем Chrome/Яндекс.Браузере): Гермес
дейтвует в настоящем браузере с вашими куками и историей — выглядит максимально
«человечно» (живой браузер с историей и куками). Расширение уже установлено на машину; Гермесу его
нужно показать как MCP-сервер в `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  browse:
    command: "<команда запуска browse-сервера>"   # как в OpenCode (mcp.json), напр.:
    args:    ["serve"]                            # node / путь / serve — скопируйте из настроек расширения
    idle_timeout_seconds: 900                     # браузер-серверы прожорливы — перезапускать при простое
    max_lifetime_seconds: 86400
    tools:
      include: ["execute", "status", "session_*", "network", "recording"]
```

Где взять команду: тот же блок `mcpServers.browse` из конфига OpenCode
(`~/.config/opencode/...`) переносится в `mcp_servers` Гермеса один-в-один
(структура совпадает: `command`/`args`/`env`). Либо попросите Гермеса:
«добавь MCP-сервер browse из моего конфига OpenCode».

Проверка:

```bash
hermes chat
> зайди на avito.ru и скажи, сколько объявлений по первому запросу из ЗАПРОСЫ в моём регионе
```

Инструменты появятся как `mcp_browse_execute` и т.п. — Гермес сам их выберет.
После правок конфига: `/reload-mcp`.

## Шаг 3. Cron — «приход на работу»

```bash
# Утро буднего дня: начало смены (AGENTS.md сам задаёт график перекуров и обеда)
hermes cron create "0 9 * * 1-5" \
  "Рабочий день по AGENTS.md: патруль Авито по навыку avito-patrol. Работай по графику с перекурами и обедом, в ~17:00 обнови sellers.md и напиши сводку за день." \
  --skill avito-patrol \
  --workdir ~/avito-patrol \
  --name "avito-shift"

# Доставка сводки (опционально)
hermes cron edit avito-shift --deliver telegram    # или discord/local

# Вечером проверить здоровье
hermes cron status && hermes cron doctor
```

Альтернатива — сказать по-человечески в чате:
> «Каждый будний день в 9 утра начинай смену: смотри объявления по теме из
> настроек по инструкции, работай неспешно, с перекурами
> и обедом, в конце дня обнови таблицу продавцов».

Полезные настройки `~/.hermes/config.yaml`:

```yaml
cron:
  failure_nudge_threshold: 3     # после 3 неудачных смен подряд — напомнит разобраться
  retry_unreachable: true        # авто-повтор, если сеть отвалилась в момент запуска
```

## Шаг 4. Аккаунт Авито

- Войти на Авито должен **человек** (капча/СМС/2FA Гермес не решает).
- Вариант 2B: логиньтесь в своём браузере — Гермес пользуется вашими куками.
- Вариант 2A: один раз войдите в Chromium Гермеса под отдельным (одноразовым)
  аккаунтом, чтобы не рисковать основным.

## Контроль

```bash
hermes cron list            # что запланировано
hermes cron runs avito-shift --limit 10   # история смен
cat ~/avito-patrol/diary.md # что Витя записал себе в дневник
```

Если две смены подряд словили 429/439 — временно поставьте на паузу:
`hermes cron pause avito-shift` (вернуть: `hermes cron resume avito-shift`).
