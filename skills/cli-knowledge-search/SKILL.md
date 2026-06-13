---
name: edgelab-knowledge-search
description: Use when an EdgeLab Space member wants to search the community library — skills, lessons, usecases, lives, workshops, guides — from the CLI/terminal without an MCP client. Triggers — найти навык в EdgeLab Space, поискать урок или разбор, открыть полный материал по slug, забрать копируемый промпт скилла через curl/JSON-RPC. The MCP server is called directly with curl. If an MCP-native client is available (Claude Code, Codex, cowork, Cursor), prefer adding the server with a header instead — see docs/MCP.md.
---

# EdgeLab Space — поиск по базе знаний через CLI / curl

## Что это

MCP-сервер EdgeLab Space публикует библиотеку сообщества (навыки, уроки,
разборы, кейсы, эфиры, воркшопы, гайды). Сервер работает stateless и отвечает
обычным JSON-RPC, поэтому его инструменты можно вызывать прямой строкой `curl` —
без MCP-клиента. Удобно для скриптов, cron-задач и быстрой проверки.

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
| `list_library` | `category?` (lesson / skill / usecase / live / workshop / guide), `limit` | Список материалов библиотеки |
| `search_library` | `query`, `limit` | Поиск по библиотеке по названию и описанию |
| `get_material` | `slug` | Полный материал: цель, результат, статья, копируемый промпт скилла (`agent_prompt`), видео, таймкодный транскрипт, шаги, дочерние уроки |
| `list_hub_questions` | `query?`, `category?`, `limit`, `offset` | Лента вопросов сообщества (AgentTinder) |
| `get_hub_question` | `question_id` | Полный вопрос + источники, файлы и комментарии/ответы |

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
      "name": "search_library",
      "arguments": { "query": "telegram бот на claude code", "limit": 10 }
    }
  }'
```

`search_library` возвращает совпадения по названию и описанию. Возьмите `slug`
нужного материала и откройте его полную карточку через `get_material`.

### Полный материал по slug

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

`get_material` отдаёт полный материал: цель, результат, статью, **копируемый
промпт скилла** (`agent_prompt` — готов к вставке агенту), видео, таймкодный
транскрипт, шаги и дочерние уроки.

### Список материалов по категории

```bash
curl -sS https://mcp.edgelab.space/mcp \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call",
       "params":{"name":"list_library","arguments":{"category":"skill","limit":10}}}'
```

`category` опционален (lesson / skill / usecase / live / workshop / guide); без
него возвращаются материалы всех разделов.

## Проверка

Вызовите `list_library` с небольшим `limit` — непустой список материалов
подтвердит, что ключ принят и подписка активна:

```bash
curl -sS https://mcp.edgelab.space/mcp \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/call",
       "params":{"name":"list_library","arguments":{"limit":5}}}'
```

Если сервер отвечает `Unauthorized` — проверьте ключ и статус подписки.

## MCP-клиент вместо curl

Если у вас есть MCP-нативный клиент (Claude Code, Codex, cowork, Cursor),
подключите тот же сервер заголовком вместо ручного `curl` — инструменты появятся
как нативные. Конфигурация и пример `.mcp.json` — в [docs/MCP.md](../../docs/MCP.md).
