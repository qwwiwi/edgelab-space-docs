---
name: edgelab-knowledge-search
description: Use when an EdgeLab Space member queries the community library — skills, lessons, usecases, lives, workshops, guides — or reads hub questions or the community digest from the CLI/terminal with curl, no MCP client. Triggers — найти навык в EdgeLab Space, поискать урок или разбор, открыть полный материал по slug, забрать копируемый промпт скилла, посмотреть дайджест сообщества за период через curl. For MCP-native clients (Claude Code, Codex, cowork, Cursor) prefer adding the server with a header — see docs/MCP.md.
---

# EdgeLab Space — поиск по базе знаний через CLI / curl

## Что это

EdgeLab Space публикует библиотеку сообщества (навыки, уроки, разборы, кейсы,
эфиры, воркшопы, гайды), ленту вопросов и дайджесты. К ним есть **REST API** —
обычные `GET`-запросы и JSON-ответы, без MCP-клиента и без JSON-RPC. Удобно для
скриптов, ботов, cron, n8n и быстрой проверки из терминала.

- **База:** `https://mcp.edgelab.space/v1`
- **Аутентификация:** `Authorization: Bearer edgelabspace_...`
- **Формат:** JSON; присылайте `Accept: application/json`.

Ключ `edgelabspace_...` выпускается в кабинете на
`platform.edgelab.space/agent-identity`, показывается один раз, работает пока
активна подписка (иначе `401`). Тот же ключ — и для MCP. Не коммитьте ключ;
держите в переменной: `export EDGELAB_KEY=edgelabspace_YOUR_KEY` и подставляйте
`$EDGELAB_KEY`.

**Ответ:** успех — `{ "data": ..., "meta": ... }`; ошибка —
`{ "error": { "code", "message" } }` (`unauthorized` 401, `not_found` 404,
`invalid_request` 400, `feature_disabled` 404, `upstream_error` 502).

## Эндпоинты (REST, все `GET`)

| Путь | Назначение |
|---|---|
| `/v1/library?category&limit` | Список материалов (category: lesson / skill / usecase / live / workshop / guide) |
| `/v1/library/search?q&category&limit` | Поиск по названию и описанию (`q` обязателен) |
| `/v1/library/materials/{slug}` | Полный материал: цель, результат, статья, `agent_prompt`, видео, транскрипт (поле `transcript`: главы + полный вербатим после `<!-- EDGELAB:FULL_TRANSCRIPT:v1 -->`), шаги, дочерние уроки |
| `/v1/hub/questions?query&category&limit&offset` | Лента вопросов (AgentTinder), пагинация `meta.{limit,offset,has_more}` |
| `/v1/hub/questions/{id}` | Полный вопрос + источники, файлы, комментарии |
| `/v1/digests?period\|date\|from+to\|last_n_days` | Дайджесты за период (только финальные); ровно одна спека |
| `/v1/digests/latest` | Последний финальный дайджест за 30 дней |

## Примеры

Поиск по библиотеке (возьмите `slug` совпадения → откройте полную карточку):

```bash
curl -sS "https://mcp.edgelab.space/v1/library/search?q=telegram%20бот%20на%20claude%20code&limit=10" \
  -H "Authorization: Bearer $EDGELAB_KEY" -H "Accept: application/json"
```

Полный материал по slug (отдаёт **копируемый промпт скилла** `agent_prompt`):

```bash
curl -sS "https://mcp.edgelab.space/v1/library/materials/your-material-slug" \
  -H "Authorization: Bearer $EDGELAB_KEY" -H "Accept: application/json"
```

Список по категории (без `category` — все разделы):

```bash
curl -sS "https://mcp.edgelab.space/v1/library?category=skill&limit=10" \
  -H "Authorization: Bearer $EDGELAB_KEY" -H "Accept: application/json"
```

Дайджест за период (`period=today|yesterday|this_week|last_week` | `date=YYYY-MM-DD`
| `from=...&to=...` ≤120 дней | `last_n_days=1..120`; тариф **edge**, в
постепенном запуске → может ответить `feature_disabled`):

```bash
curl -sS "https://mcp.edgelab.space/v1/digests?period=last_week" \
  -H "Authorization: Bearer $EDGELAB_KEY" -H "Accept: application/json"
curl -sS "https://mcp.edgelab.space/v1/digests/latest" \
  -H "Authorization: Bearer $EDGELAB_KEY" -H "Accept: application/json"
```

## Проверка

`/v1/library?limit=5` с ключом → непустой список подтверждает, что ключ принят и
подписка активна. `401 unauthorized` → проверьте ключ и статус подписки. Полный
справочник REST — [docs/REST.md](../../docs/REST.md).

## Запасной путь — JSON-RPC через MCP

Тот же сервер отвечает и JSON-RPC по `https://mcp.edgelab.space/mcp` (тот же
ключ) — нужно, только если инструмент уже умеет MCP/JSON-RPC; для простого CLI
берите REST выше. Инструменты: `search_library`, `list_library`, `get_material`,
`list_hub_questions`, `get_hub_question`, `list_digests`, `get_digest`.

```bash
curl -sS https://mcp.edgelab.space/mcp \
  -H "Authorization: Bearer $EDGELAB_KEY" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call",
       "params":{"name":"search_library","arguments":{"query":"claude code","limit":10}}}'
```

Полный JSON-RPC справочник — [docs/CLI.md](../../docs/CLI.md).

## MCP-клиент вместо curl

Есть MCP-нативный клиент (Claude Code, Codex, cowork, Cursor)? Подключите тот же
сервер заголовком — инструменты появятся как нативные. Конфиг и пример
`.mcp.json` — [docs/MCP.md](../../docs/MCP.md).

## Полный транскрипт материала

Поле `transcript` в ответе `get_material` / `/v1/library/materials/{slug}` устроено так:

```
<таймкодные главы>
<!-- EDGELAB:FULL_TRANSCRIPT:v1 -->
<полный вербатим транскрипт>
```

Главы — до сентинела, полный дословный текст — после. Достать только полный текст:

```bash
curl -sS "https://mcp.edgelab.space/v1/library/materials/SLUG" \
  -H "Authorization: Bearer $EDGELAB_KEY" \
  | jq -r '.data.transcript // ""' \
  | awk '/EDGELAB:FULL_TRANSCRIPT:v1/{f=1;next} f'
```

Если поле `null` — транскрипт ещё не добавлен для этого материала. Вкладка «Полный
транскрипт» на платформе появляется у любого типа материала (урок/эфир/воркшоп) при
наличии сентинела.
