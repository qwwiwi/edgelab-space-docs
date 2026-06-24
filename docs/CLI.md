# CLI / curl — доступ без MCP-клиента

Если у вас нет MCP-клиента, библиотеку EdgeLab Space можно дёргать прямой строкой
`curl` — для скриптов, cron-задач, ботов и быстрой проверки. Есть два пути.

- **REST API** — простой и основной путь: обычные `GET`-запросы и JSON-ответы,
  никакого JSON-RPC. Рекомендуется для нового кода. Полный справочник —
  [REST.md](REST.md).
- **JSON-RPC через MCP** — запасной путь для инструментов, которые уже умеют
  говорить с MCP-сервером по JSON-RPC.

Оба используют один персональный ключ `edgelabspace_...` из кабинета на
`platform.edgelab.space/agent-identity` и заголовок
`Authorization: Bearer edgelabspace_...`. Заголовок `Accept` должен принимать
`application/json`.

Не оставляйте ключ в истории shell и не коммитьте в репозитории — держите его
в переменной окружения и подставляйте через
`-H "Authorization: Bearer $EDGELAB_KEY"`.

## Через REST (рекомендуется)

База — `https://mcp.edgelab.space/v1`. Все запросы — обычные `GET`, ответ — конверт
`{"data": ..., "meta": ...}`. Подробно — в [REST.md](REST.md).

### Список материалов

```bash
curl -sS "https://mcp.edgelab.space/v1/library?category=skill&limit=10" \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

### Поиск по библиотеке

```bash
curl -sS "https://mcp.edgelab.space/v1/library/search?q=telegram%20бот%20на%20claude%20code&limit=10" \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

### Полный материал по slug

```bash
curl -sS https://mcp.edgelab.space/v1/library/materials/your-material-slug \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

Ответ несёт полную карточку: цель, результат, статью, копируемый промпт скилла
(`agent_prompt`), видео, транскрипт (поле `transcript`: таймкодные главы + полный вербатим после сентинела `<!-- EDGELAB:FULL_TRANSCRIPT:v1 -->`), шаги и дочерние уроки.

### Проверка

```bash
curl -sS "https://mcp.edgelab.space/v1/library?limit=5" \
  -H "Authorization: Bearer edgelabspace_YOUR_KEY" \
  -H "Accept: application/json"
```

Непустой список подтверждает, что ключ принят и подписка активна. Полный список
эндпоинтов (`/hub/questions`, `/digests`, …) — в [REST.md](REST.md).

## Через MCP (JSON-RPC) — запасной путь

MCP-сервер EdgeLab Space работает stateless и отвечает обычным JSON-RPC, поэтому
его инструменты можно вызывать тем же `curl`. Подходит инструментам, которые уже
говорят с сервером по JSON-RPC.

- Эндпоинт: `https://mcp.edgelab.space/mcp`
- Аутентификация: `Authorization: Bearer edgelabspace_...` (тот же ключ)

Каждый вызов — один `POST` с JSON-RPC телом.

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

`get_material` возвращает полную карточку: цель, результат, статью, копируемый
промпт скилла (`agent_prompt`), видео, транскрипт (поле `transcript`: таймкодные главы + полный вербатим после сентинела `<!-- EDGELAB:FULL_TRANSCRIPT:v1 -->`), шаги и дочерние
уроки.

### Доступные инструменты

| Инструмент | Аргументы |
|---|---|
| `list_library` | `category?` (lesson / skill / usecase / live / workshop / guide), `limit` |
| `search_library` | `query`, `limit` |
| `get_material` | `slug` |
| `list_hub_questions` | `query?`, `category?`, `limit`, `offset` |
| `get_hub_question` | `question_id` |
| `list_digests` | `period?`, `date?`, `from_?`, `to?`, `last_n_days?` |
| `get_digest` | `period?`, `date?`, `from_?`, `to?`, `last_n_days?` |

Полное описание инструментов — в [MCP.md](MCP.md).

## Подсказки

- Замените `edgelabspace_YOUR_KEY` на свой ключ из кабинета. Не оставляйте его
  в истории shell и не коммитьте в репозитории — лучше держать в переменной
  окружения и подставлять через `-H "Authorization: Bearer $EDGELAB_KEY"`.
- `401` (REST: `unauthorized`; MCP: `Unauthorized`) означает неверный ключ или
  неактивную подписку.
- Есть MCP-нативный клиент (Claude Code, Codex, cowork, Cursor)? Подключите тот
  же сервер заголовком — инструменты появятся как нативные. См. [MCP.md](MCP.md).
