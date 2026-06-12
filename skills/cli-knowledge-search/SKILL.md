---
name: edgelab-knowledge-search
description: Use when an EdgeLab Space member wants to search the community knowledge base — skills, lessons, drops, digests, categories — from the CLI/terminal without an MCP client. Triggers — найти навык в EdgeLab Space, поискать урок или разбор, посмотреть свежие дропы, получить дайджест трендов, открыть карточку по slug, список категорий через curl/JSON-RPC. The MCP server is called directly with curl. If an MCP-native client is available (Claude Code, Codex, cowork, Cursor), prefer adding the server with a header instead — see docs/MCP.md.
---

# EdgeLab Space — поиск по базе знаний через CLI / curl

## Что это

MCP-сервер EdgeLab Space публикует библиотеку сообщества (навыки, разборы,
дропы, дайджесты, категории). Сервер работает stateless и отвечает обычным
JSON-RPC, поэтому его инструменты можно вызывать прямой строкой `curl` — без
MCP-клиента. Удобно для скриптов, cron-задач и быстрой проверки.

- **Эндпоинт:** `https://mcp.edgelab.space/mcp`
- **Аутентификация:** `Authorization: Bearer edgelabspace_...`
- Транспорт: Streamable HTTP, stateless — каждый вызов это самостоятельный POST
  с JSON-RPC телом, без отдельного хендшейка сессии.

Персональный ключ `edgelabspace_...` выпускается в кабинете на
`platform.edgelab.space/agent-identity`, показывается один раз. Доступ работает,
пока активна подписка; при отмене сервер начинает отвечать `Unauthorized`.

Не оставляйте ключ в истории shell и не коммитьте в репозитории — держите в
переменной окружения и подставляйте через
`-H "Authorization: Bearer $EDGELAB_MCP_KEY"`.

Заголовок `Accept` должен принимать `application/json`.

## Инструменты

Все инструменты только для чтения и возвращают материалы, доступные вашей
когорте.

| Инструмент | Аргументы | Назначение |
|---|---|---|
| `search_knowledge` | `query` | Поиск по библиотеке (навыки, тренды, уроки, дайджесты) |
| `get_entry` | `slug` | Одна запись по slug — со скиллом и инструкцией установки |
| `list_latest_drops` | `limit` (по умолчанию 10) | Свежие опубликованные дропы |
| `get_digest` | `limit` (по умолчанию 5) | Последние дайджесты трендов |
| `list_categories` | — | Список доступных разделов |

## Примеры

### Список инструментов

```bash
curl -sS https://mcp.edgelab.space/mcp \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }'
```

### Поиск по библиотеке

```bash
curl -sS https://mcp.edgelab.space/mcp \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "search_knowledge",
      "arguments": { "query": "telegram бот на claude code" }
    }
  }'
```

### Карточка по slug

```bash
curl -sS https://mcp.edgelab.space/mcp \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "get_entry",
      "arguments": { "slug": "your-entry-slug" }
    }
  }'
```

### Свежие дропы и дайджест

```bash
# свежие дропы (limit по умолчанию 10)
curl -sS https://mcp.edgelab.space/mcp \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call",
       "params":{"name":"list_latest_drops","arguments":{"limit":5}}}'

# последние дайджесты трендов (limit по умолчанию 5)
curl -sS https://mcp.edgelab.space/mcp \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/call",
       "params":{"name":"get_digest","arguments":{"limit":5}}}'
```

## Проверка

Вызовите `list_categories` — пустой или непустой список категорий подтвердит,
что ключ принят и подписка активна:

```bash
curl -sS https://mcp.edgelab.space/mcp \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":6,"method":"tools/call",
       "params":{"name":"list_categories","arguments":{}}}'
```

Если сервер отвечает `Unauthorized` — проверьте ключ и статус подписки.

## MCP-клиент вместо curl

Если у вас есть MCP-нативный клиент (Claude Code, Codex, cowork, Cursor),
подключите тот же сервер заголовком вместо ручного `curl` — инструменты появятся
как нативные. Конфигурация и пример `.mcp.json` — в [docs/MCP.md](../../docs/MCP.md).
