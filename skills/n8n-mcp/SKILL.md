---
name: n8n MCP Integration
description: >-
  Chuyên gia n8n workflow automation. Quản lý workflows qua MCP Server (31 tools).
  Bao gồm kiến thức chuyên sâu: 5 workflow patterns, expression syntax, webhook data
  access, Code node format, và quy trình tự động tạo-test-debug workflows.
---

# n8n MCP Integration

Bạn là chuyên gia n8n workflow automation. Bạn có quyền truy cập n8n thông qua MCP Server với đầy đủ quyền CRUD, Testing và Debugging.

## Connection

```
POST http://172.17.0.1:3002/mcp
```

Không cần session — gọi trực tiếp. Response là SSE format, parse `data:` lines.

## Cách gọi MCP Tool

```bash
curl -s --max-time 30 -X POST http://172.17.0.1:3002/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"TOOL_NAME","arguments":{...}}}'
```

Kết quả nằm trong `data.result.content[0].text` (JSON string).

---

## 🚨 QUY TẮC QUAN TRỌNG

### 1. Webhook Data LUÔN nằm dưới `$json.body`

```javascript
// ❌ SAI — sẽ trả về undefined
{{ $json.email }}
{{ $json.name }}

// ✅ ĐÚNG — webhook data nằm dưới .body
{{ $json.body.email }}
{{ $json.body.name }}
```

Webhook node output:
```json
{
  "headers": {...},
  "params": {...},
  "query": {...},
  "body": {
    "email": "user@example.com",
    "name": "John"
  }
}
```

### 2. Code Node — Return format BẮT BUỘC

```javascript
// ❌ SAI — n8n sẽ lỗi
return { email: "test@example.com" };

// ✅ ĐÚNG — luôn trả về array of objects với key "json"
return [{ json: { email: "test@example.com" } }];

// Nhiều items
return [
  { json: { name: "Alice" } },
  { json: { name: "Bob" } }
];
```

### 3. Expression Syntax — Luôn dùng {{ }}

```javascript
// ❌ SAI — treated as literal text
$json.body.email

// ✅ ĐÚNG
{{ $json.body.email }}

// ❌ SAI trong Code node — không dùng {{ }} trong code
const email = '={{ $json.email }}';

// ✅ ĐÚNG trong Code node — dùng trực tiếp
const email = $json.email;
const allItems = $input.all();
```

### 4. Tham chiếu node khác

```javascript
// Node không có space
{{ $node["Set"].json.value }}

// Node có space (PHẢ DÙNG QUOTES + BRACKETS)
{{ $node["HTTP Request"].json.data }}

// Từ webhook qua node khác
{{ $node["Webhook"].json.body.email }}
```

### 5. create/update workflow PHẢI có `settings: {}`

```json
{
  "name": "My Workflow",
  "nodes": [...],
  "connections": {...},
  "settings": {}
}
```

---

## 5 Workflow Patterns

### Pattern 1: Webhook Processing (Phổ biến nhất - 35%)
```
Webhook → Validate/Transform → Action → Respond to Webhook
```
Khi: Nhận data từ bên ngoài (form, payment, callback)

### Pattern 2: HTTP API Integration
```
Trigger → HTTP Request → Transform → Output → Error Handler
```
Khi: Fetch data từ REST APIs, đồng bộ services

### Pattern 3: Database Operations
```
Schedule → Query → Transform → Write → Verify
```
Khi: ETL, sync databases, backup data

### Pattern 4: AI Agent Workflow
```
Trigger → AI Agent (Model + Tools + Memory) → Output
```
Khi: Chatbot, content generation, data analysis

### Pattern 5: Scheduled Tasks (28%)
```
Schedule → Fetch → Process → Deliver → Log
```
Khi: Reports hàng ngày, monitoring, maintenance

---

## Available Tools (31 total)

### Workflow Management (10 tools)
| Tool | Arguments |
|------|-----------|
| `list_workflows` | `{}` hoặc `{"active": true, "limit": 10}` |
| `get_workflow` | `{"id": "ID"}` |
| `create_workflow` | `{"name": "...", "nodes": [...], "connections": {...}, "settings": {}}` |
| `update_workflow` | `{"id": "ID", "name": "...", "nodes": [...], "settings": {}}` |
| `delete_workflow` | `{"id": "ID"}` |
| `activate_workflow` | `{"id": "ID", "active": true}` |
| `execute_workflow` | `{"id": "ID"}` — smart fallback: webhook → activate |
| `trigger_webhook` | `{"webhook_path": "path", "method": "POST", "body": {...}}` |
| `list_executions` | `{"workflowId": "ID", "status": "error", "limit": 10}` |
| `get_execution` | `{"id": "ID"}` |

### Credentials (6 tools)
| Tool | Arguments |
|------|-----------|
| `get_credential_schema` | `{"typeName": "githubApi"}` |
| `list_credentials` | `{}` |
| `create_credential` | `{"name": "...", "type": "...", "data": {...}}` |
| `update_credential` | `{"id": "ID", "name": "...", "data": {...}}` |
| `delete_credential` | `{"id": "ID"}` |
| `test_credential` | `{"credentialId": "ID"}` |

### Nodes (2 tools)
| Tool | Arguments |
|------|-----------|
| `list_node_types` | `{}` |
| `get_node_schema` | `{"nodeName": "n8n-nodes-base.httpRequest"}` |

### Validation & Linting (5 tools)
| Tool | Arguments |
|------|-----------|
| `validate_workflow_structure` | `{"workflow": {...}}` |
| `validate_workflow_credentials` | `{"workflow": {...}}` |
| `validate_workflow_expressions` | `{"workflow": {...}}` |
| `lint_workflow` | `{"workflow": {...}}` |
| `suggest_workflow_improvements` | `{"workflow": {...}}` |

### Templates (4 tools)
| Tool | Arguments |
|------|-----------|
| `search_templates` | `{"query": "slack"}` |
| `get_template_details` | `{"id": 123}` |
| `import_template` | `{"templateId": 123}` |
| `export_workflow_as_template` | `{"workflowId": "ID"}` |

### Backup (4 tools)
| Tool | Arguments |
|------|-----------|
| `backup_workflow` | `{"workflowId": "ID"}` |
| `list_workflow_backups` | `{"workflowId": "ID"}` |
| `restore_workflow` | `{"workflowId": "ID", "backupId": "..."}` |
| `diff_workflow_versions` | `{"workflowId": "ID", "backupId1": "...", "backupId2": "..."}` |

---

## Quy Trình Làm Việc Chuẩn

```
1. Nhận yêu cầu → Chọn 1 trong 5 patterns
2. create_workflow → tạo workflow
3. activate_workflow → bật workflow
4. trigger_webhook (test_mode: true) → test thử
5. list_executions → kiểm tra kết quả
6. get_execution → nếu lỗi, đọc chi tiết
7. update_workflow → sửa lỗi
8. Lặp lại 4-7 cho đến khi OK
```

### Ví dụ: Tạo webhook handler

```bash
# 1. Create workflow
curl -s --max-time 30 -X POST http://172.17.0.1:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"create_workflow","arguments":{
    "name":"Outlook Webhook Handler",
    "nodes":[
      {"parameters":{"path":"outlook-callback","httpMethod":"POST","responseMode":"onReceived","responseData":"allEntries"},
       "id":"wh1","name":"Webhook","type":"n8n-nodes-base.webhook","typeVersion":2,"position":[250,300]},
      {"parameters":{"values":{"string":[
        {"name":"subject","value":"={{ $json.body.subject }}"},
        {"name":"sender","value":"={{ $json.body.sender }}"}
      ]}},"id":"set1","name":"Extract Data","type":"n8n-nodes-base.set","typeVersion":3,"position":[450,300]}
    ],
    "connections":{"Webhook":{"main":[[{"node":"Extract Data","type":"main","index":0}]]}},
    "settings":{}
  }}}'

# 2. Activate
# ... activate_workflow với ID vừa tạo

# 3. Test
# ... trigger_webhook với test_mode: true, body: {"subject": "Test", "sender": "boss@company.com"}

# 4. Check
# ... list_executions → get_execution → kiểm tra output
```

---

## Expression Cheat Sheet

```javascript
// Current node data
{{ $json.body.email }}              // Webhook POST body
{{ $json.body.items[0].name }}      // Array access
{{ $json['field with spaces'] }}    // Bracket notation

// Other node data
{{ $node["HTTP Request"].json.data }}
{{ $node["Webhook"].json.body.email }}

// Date/Time (Luxon)
{{ $now.toFormat('yyyy-MM-dd') }}
{{ $now.plus({days: 7}).toISO() }}

// Conditional
{{ $json.status === 'active' ? 'Bật' : 'Tắt' }}
{{ $json.email || 'no-email@example.com' }}

// String methods
{{ $json.name.toLowerCase() }}
{{ $json.message.replace('old', 'new') }}
```

---

## Common Node Types

| Node | Type | Dùng khi |
|------|------|----------|
| Webhook | `n8n-nodes-base.webhook` | Nhận HTTP requests |
| HTTP Request | `n8n-nodes-base.httpRequest` | Gọi REST APIs |
| Set | `n8n-nodes-base.set` | Map/transform fields |
| Code | `n8n-nodes-base.code` | Custom JS/Python logic |
| IF | `n8n-nodes-base.if` | Conditional routing |
| Switch | `n8n-nodes-base.switch` | Multi-condition routing |
| Schedule | `n8n-nodes-base.scheduleTrigger` | Chạy theo lịch |
| Manual | `n8n-nodes-base.manualTrigger` | Test thủ công |
| NoOp | `n8n-nodes-base.noOp` | Placeholder/debug |
| Respond to Webhook | `n8n-nodes-base.respondToWebhook` | Trả response cho webhook |

---

## Lưu Ý Quan Trọng

- Parse SSE response: tìm lines bắt đầu bằng `data:` chứa JSON
- **Luôn** include `"settings": {}` khi create/update workflows
- update_workflow cần gửi `nodes` ở top-level
- Webhook mới tạo cần ~5-10s để n8n đăng ký (queue mode)
- Code node: JavaScript cho 95% use case, Python không có thư viện ngoài
- n8n UI: http://localhost:5678
