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
      "name": "search_library",
      "arguments": { "query": "telegram бот на claude code", "limit": 10 }
    }
  }'
```

## Полный материал по slug

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
      "name": "get_material",
      "arguments": { "slug": "your-material-slug" }
    }
  }'
```

`get_material` возвращает полную карточку: цель, результат, статью, копируемый
промпт скилла (`agent_prompt`), видео, таймкодный транскрипт, шаги и дочерние
уроки.

## Доступные инструменты

Те же, что и в MCP-клиенте:

| Инструмент | Аргументы |
|---|---|
| `list_library` | `category?` (lesson / skill / usecase / live / workshop / guide), `limit` |
| `search_library` | `query`, `limit` |
| `get_material` | `slug` |
| `list_hub_questions` | `query?`, `category?`, `limit`, `offset` |
| `get_hub_question` | `question_id` |
| `list_digests` | `period?`, `date?`, `from_?`, `to?`, `last_n_days?` |
| `get_digest` | `period?`, `date?`, `from_?`, `to?`, `last_n_days?` |

Полное описание — в [MCP.md](MCP.md).

## Подсказки

- Замените `edgelabspace_YOUR_KEY` на свой ключ из кабинета. Не оставляйте его
  в истории shell и не коммитьте в репозитории — лучше держать в переменной
  окружения и подставлять через `-H "Authorization: Bearer $EDGELAB_MCP_KEY"`.
- `Unauthorized` в ответе означает неверный ключ или неактивную подписку.
