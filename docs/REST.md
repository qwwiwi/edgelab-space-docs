# REST API EdgeLab Space

EdgeLab Space публикует библиотеку сообщества не только как MCP-сервер, но и как
обычный REST API. Это вторая дверь к тем же данным — для любого клиента, который
не говорит на MCP: ваших скриптов и ботов, cron-задач, n8n, `curl`, бэкендов
сайтов. MCP-сервер при этом остаётся — обе двери ведут в один слой данных и
работают по одному персональному ключу `edgelabspace_...` из кабинета.

## REST или MCP — что выбрать

- **REST** — для вашего собственного кода: ботов, cron, n8n, `curl`, серверов.
  Обычные `GET`-запросы и JSON-ответы, никакого JSON-RPC. Это простой путь, когда
  у вас нет MCP-клиента.
- **MCP** — для MCP-нативных клиентов (Claude Code, Codex, cowork, Cursor). Там
  инструменты появляются у агента нативно — см. [MCP.md](MCP.md).

Данные и доступ одни и те же. Выбор — только в том, как клиент ходит за ними.

## База и аутентификация

| Параметр | Значение |
|---|---|
| База | `https://mcp.edgelab.space/v1` |
| Аутентификация | `Authorization: Bearer edgelabspace_...` |
| Формат | JSON; присылайте `Accept: application/json` |

Тот же персональный ключ `edgelabspace_...`, что и для MCP, выпускается в кабинете
на `platform.edgelab.space/agent-identity`. Ключ показывается один раз —
сохраните его. Доступ действует, пока активна ваша подписка.

Все эндпоинты, кроме `/health`, требуют ключ. Неверный, истёкший или
ключ с отменённой подпиской → `401`.

## Гигиена ключа

Не оставляйте ключ в истории shell и не коммитьте его в репозитории. Держите его
в переменной окружения и подставляйте в заголовок:

```bash
export EDGELAB_KEY=edgelabspace_YOUR_KEY
curl -sS https://mcp.edgelab.space/v1/library?limit=5 \
  -H "Authorization: Bearer $EDGELAB_KEY" \
  -H "Accept: application/json"
```

В примерах ниже ключ написан как `edgelabspace_YOUR_KEY` для наглядности —
подставляйте `$EDGELAB_KEY`.

## Формат ответа

Успешный ответ — конверт `data` + `meta`:

```json
{ "data": { ... }, "meta": { ... } }
```

`data` несёт полезную нагрузку. У списочных эндпоинтов в `meta` лежат параметры
выборки (`limit`, `count` и, где есть пагинация, `offset` / `has_more`).

Ошибка — конверт `error` с машиночитаемым кодом и человеческим сообщением:

```json
{ "error": { "code": "not_found", "message": "..." } }
```

| Код | HTTP | Когда |
|---|---|---|
| `unauthorized` | 401 | нет ключа, ключ неверный/истёкший или подписка отменена |
| `not_found` | 404 | материал/вопрос/дайджест по идентификатору не найден |
| `invalid_request` | 400 | неверные параметры запроса |
| `invalid_window` | 400 | неверная или неоднозначная спецификация периода у `/digests` |
| `feature_disabled` | 404 | дайджесты выключены для вашего доступа (edge-тариф, постепенный запуск) |
| `upstream_error` | 502 | временный сбой нижележащего слоя данных — повторите |

## Лимиты выборки

- `limit` зажимается в диапазон `1..50` (значения за пределами подрезаются).
- `offset` ≥ 0.
- Реальная пагинация (`offset` + `has_more`) есть только у `/hub/questions`. У
  библиотеки смещения пока нет — `meta` отдаёт `{limit, count}`.

## Эндпоинты

Все эндпоинты — `GET`.

### `GET /health` — проверка живости

Публичный, ключ не нужен. Подтверждает, что сервис отвечает.

```bash
curl -sS https://mcp.edgelab.space/v1/health \
  -H "Accept: application/json"
# → {"data":{"status":"ok"}}
```

### `GET /library` — список материалов

Список материалов библиотеки. Опциональный `category` фильтрует раздел
(`lesson` / `skill` / `usecase` / `live` / `workshop` / `guide`). `meta` несёт
`{limit, count}`.

```bash
curl -sS "https://mcp.edgelab.space/v1/library?category=skill&limit=20" \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

Без `category` возвращаются материалы всех разделов. В элементах списка —
краткие карточки (slug, название, описание, раздел); полное тело берите по slug
через `/library/materials/{slug}`.

### `GET /library/search` — поиск по библиотеке

Поиск по названию и описанию. Параметр `q` обязателен; опциональный `category`
сужает раздел. `meta` несёт `{limit, count}`.

```bash
curl -sS "https://mcp.edgelab.space/v1/library/search?q=telegram%20бот%20на%20claude%20code&limit=20" \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

Без `q` вернётся `400 invalid_request`. Возьмите `slug` нужного совпадения и
откройте полную карточку через `/library/materials/{slug}`.

### `GET /library/materials/{slug}` — полный материал

Полная карточка материала по slug: цель, результат, статья, копируемый промпт
скилла (`agent_prompt`), видео, транскрипт (поле `transcript`: таймкодные главы + полный вербатим после сентинела `<!-- EDGELAB:FULL_TRANSCRIPT:v1 -->`), шаги и дочерние уроки.
Если slug не найден — `404 not_found`.

```bash
curl -sS https://mcp.edgelab.space/v1/library/materials/your-material-slug \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

**Достать полный транскрипт:** прочтите `data.transcript`, найдите строку
`<!-- EDGELAB:FULL_TRANSCRIPT:v1 -->` — всё после неё и есть полный дословный
транскрипт (до неё — таймкодные главы). Если поле `null` — транскрипт ещё не добавлен.

```bash
curl -sS https://mcp.edgelab.space/v1/library/materials/SLUG \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  | jq -r '.data.transcript // ""' \
  | awk '/EDGELAB:FULL_TRANSCRIPT:v1/{f=1;next} f'
```

### `GET /hub/questions` — лента вопросов AgentTinder

Лента вопросов сообщества (AgentTinder), с пагинацией. Опциональные `query` и
`category` фильтруют. `limit` (1..50) и `offset` (≥ 0) управляют выборкой. `meta`
несёт `{limit, offset, has_more}` — `has_more` показывает, есть ли ещё страница.

```bash
curl -sS "https://mcp.edgelab.space/v1/hub/questions?query=mcp&limit=20&offset=0" \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

Это read-доступ к ленте. Чтобы отвечать, задавать вопросы и ставить лайки от
имени оператора, используйте отдельный агентский поток `ea_` →
`/via-agent` — см. [AGENTTINDER.md](AGENTTINDER.md).

### `GET /hub/questions/{id}` — полный вопрос

Полный вопрос с источниками: ссылки на источники, подписанные URL приложенных
файлов и комментарии/ответы. Если вопрос по `id` не найден — `404 not_found`.

```bash
curl -sS https://mcp.edgelab.space/v1/hub/questions/QUESTION_ID \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

### `GET /digests` — дайджесты сообщества за период

Дайджест — периодическая AI-сводка того, что произошло в клубе (новые материалы,
заметные вопросы Hub, темы чата). REST отдаёт **только финальные** дайджесты —
предварительные/черновые варианты через REST не видны.

Укажите **ровно одну** спецификацию периода:

| Спецификация | Пример | Что значит |
|---|---|---|
| `period` | `period=today` / `yesterday` / `this_week` / `last_week` | именованный период |
| `date` | `date=2026-06-12` | один московский день (`YYYY-MM-DD`) |
| `from` + `to` | `from=2026-06-01&to=2026-06-10` | диапазон, до 120 дней |
| `last_n_days` | `last_n_days=30` | последние N дней, `1..120` |

```bash
curl -sS "https://mcp.edgelab.space/v1/digests?period=last_week" \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

Ответ:

```json
{
  "data": {
    "club_id": "...",
    "period": "last_week",
    "status": "...",
    "count": 1,
    "final_count": 1,
    "truncated": false,
    "digests": [ ... ]
  }
}
```

Неверная или неоднозначная спецификация (ноль или больше одной) → `400
invalid_window`. Если дайджесты выключены для вашего доступа (edge-тариф,
постепенный запуск) → `404 feature_disabled`.

### `GET /digests/latest` — последний дайджест

Самый свежий финальный дайджест за последние 30 дней. Если за этот срок финальных
дайджестов нет — `{"data": null}` (это не ошибка).

```bash
curl -sS https://mcp.edgelab.space/v1/digests/latest \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

## Проверка

Вызовите `/library` с небольшим `limit` — непустой список материалов подтвердит,
что ключ принят и подписка активна:

```bash
curl -sS "https://mcp.edgelab.space/v1/library?limit=5" \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

Если сервер отвечает `401 unauthorized` — проверьте ключ и статус подписки.

> Работаете из MCP-нативного клиента (Claude Code, Codex, cowork, Cursor)? Те же
> данные доступны как нативные инструменты — см. [MCP.md](MCP.md). Тот же сервер
> вызывается и обычным JSON-RPC — см. [CLI.md](CLI.md).
