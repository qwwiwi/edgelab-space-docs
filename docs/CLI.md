# CLI / curl — доступ без MCP-клиента

MCP-сервер EdgeLab Space работает stateless и отвечает обычным JSON-RPC, поэтому
его инструменты можно вызывать прямой строкой `curl` — без MCP-клиента. Удобно
для скриптов, cron-задач и быстрой проверки.

- Эндпоинт: `https://mcp.edgelab.space/mcp`
- Аутентификация: `Authorization: Bearer edgelabspace_...`
- Тот же ключ `edgelabspace_...` из кабинета, что и для MCP-клиента.

Каждый вызов — один POST с JSON-RPC телом. Заголовок `Accept` должен принимать
`application/json`.

## Список инструментов

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

## Поиск по библиотеке

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

## Карточка по slug

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

## Доступные инструменты

Те же, что и в MCP-клиенте:

| Инструмент | Аргументы |
|---|---|
| `search_knowledge` | `query` |
| `get_entry` | `slug` |
| `list_latest_drops` | `limit` (по умолчанию 10) |
| `get_digest` | `limit` (по умолчанию 5) |
| `list_categories` | — |

Полное описание — в [MCP.md](MCP.md).

## Подсказки

- Замените `edgelabspace_YOUR_KEY` на свой ключ из кабинета. Не оставляйте его
  в истории shell и не коммитьте в репозитории — лучше держать в переменной
  окружения и подставлять через `-H "Authorization: Bearer $EDGELAB_MCP_KEY"`.
- `Unauthorized` в ответе означает неверный ключ или неактивную подписку.
