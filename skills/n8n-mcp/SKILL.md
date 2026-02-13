---
name: n8n MCP Integration
description: >-
  Manage n8n workflows via MCP Server. Create, update, delete, execute workflows,
  manage credentials, validate, backup — all through HTTP API calls to the
  n8n-custom-mcp server (31 tools).
---

# n8n MCP Integration

You have access to an n8n automation platform via an MCP (Model Context Protocol) server.
Use `curl` to interact with the MCP server at `http://172.17.0.1:3002/mcp`.

## Connection

The MCP server uses **Streamable HTTP / SSE** transport. All calls are POST requests to:

```
http://172.17.0.1:3002/mcp
```

**No session initialization needed** — just send POST requests directly.

## How to Call MCP Tools

```bash
curl -s --max-time 30 -X POST http://172.17.0.1:3002/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"TOOL_NAME","arguments":{...}}}'
```

The response is SSE format. Parse lines starting with `data:` as JSON.
The tool result is at `data.result.content[0].text` (JSON string).
If `data.result.isError` is `true`, the call failed — read the error text.

## Available Tools (31 total)

### Workflow Management (10 tools)
| Tool | Arguments | Description |
|------|-----------|-------------|
| `list_workflows` | `{}` or `{"active": true, "limit": 10}` | List all workflows |
| `get_workflow` | `{"id": "ID"}` | Get workflow details (nodes, connections) |
| `create_workflow` | `{"name": "...", "nodes": [...], "connections": {...}, "settings": {}}` | Create workflow (**settings required**) |
| `update_workflow` | `{"id": "ID", "name": "...", "nodes": [...], "connections": {...}, "settings": {}}` | Update workflow (**cần gửi nodes**) |
| `delete_workflow` | `{"id": "ID"}` | Delete workflow |
| `activate_workflow` | `{"id": "ID", "active": true}` | Activate/deactivate workflow |
| `execute_workflow` | `{"id": "ID"}` | Execute workflow (smart fallback — see below) |
| `trigger_webhook` | `{"webhook_path": "path", "method": "POST", "body": {...}}` | Trigger workflow via webhook path |
| `list_executions` | `{"workflowId": "ID", "status": "error", "limit": 10}` | List execution history |
| `get_execution` | `{"id": "ID"}` | Get execution details (debug errors) |

### Credentials Management (6 tools)
| Tool | Arguments | Description |
|------|-----------|-------------|
| `get_credential_schema` | `{"typeName": "githubApi"}` | Get required fields for a credential type |
| `list_credentials` | `{}` | List all credentials |
| `create_credential` | `{"name": "...", "type": "...", "data": {...}}` | Create credential |
| `update_credential` | `{"id": "ID", "name": "...", "data": {...}}` | Update credential |
| `delete_credential` | `{"id": "ID"}` | Delete credential (safety check) |
| `test_credential` | `{"credentialId": "ID"}` | Test credential validity |

### Node Types (2 tools)
| Tool | Arguments | Description |
|------|-----------|-------------|
| `list_node_types` | `{}` | List available n8n node types |
| `get_node_schema` | `{"nodeName": "n8n-nodes-base.httpRequest"}` | Get schema for a node type |

### Validation & Linting (5 tools)
| Tool | Arguments | Description |
|------|-----------|-------------|
| `validate_workflow_structure` | `{"workflow": {...}}` | Check JSON structure |
| `validate_workflow_credentials` | `{"workflow": {...}}` | Check credentials |
| `validate_workflow_expressions` | `{"workflow": {...}}` | Validate expressions |
| `lint_workflow` | `{"workflow": {...}}` | Detect logic issues (score 0-100) |
| `suggest_workflow_improvements` | `{"workflow": {...}}` | Optimization suggestions |

### Template System (4 tools)
| Tool | Arguments | Description |
|------|-----------|-------------|
| `search_templates` | `{"query": "slack"}` | Search n8n.io template library |
| `get_template_details` | `{"id": 123}` | Get template JSON |
| `import_template` | `{"templateId": 123}` | Import template into n8n |
| `export_workflow_as_template` | `{"workflowId": "ID"}` | Export workflow (credentials removed) |

### Backup & Versioning (4 tools)
| Tool | Arguments | Description |
|------|-----------|-------------|
| `backup_workflow` | `{"workflowId": "ID"}` | Create backup |
| `list_workflow_backups` | `{"workflowId": "ID"}` | List backups |
| `restore_workflow` | `{"workflowId": "ID", "backupId": "backup_xxx"}` | Restore from backup |
| `diff_workflow_versions` | `{"workflowId": "ID", "backupId1": "...", "backupId2": "..."}` | Compare versions |

## execute_workflow — Smart Fallback

`execute_workflow` hoạt động với smart fallback:

1. **Workflow có Webhook node** → tự activate + trigger webhook → **chạy ngay**
2. **Workflow không có Webhook** → activate workflow → chạy khi trigger fire (schedule, event...)

```bash
# Execute workflow → chạy ngay nếu có webhook
curl -s --max-time 30 -X POST http://172.17.0.1:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"execute_workflow","arguments":{"id":"<workflow-id>"}}}'
# → {"executionMethod":"webhook-trigger","result":{"status":200,"data":{"message":"Workflow was started"}}}
```

## Example: Full Workflow Lifecycle

```bash
# 1. Create workflow with webhook trigger
curl -s --max-time 30 -X POST http://172.17.0.1:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"create_workflow","arguments":{
    "name":"My Webhook Workflow",
    "nodes":[
      {"parameters":{"path":"my-hook","httpMethod":"POST","responseMode":"onReceived","responseData":"allEntries"},
       "id":"wh1","name":"Webhook","type":"n8n-nodes-base.webhook","typeVersion":2,"position":[250,300]},
      {"parameters":{},"id":"noop1","name":"NoOp","type":"n8n-nodes-base.noOp","typeVersion":1,"position":[450,300]}
    ],
    "connections":{"Webhook":{"main":[[{"node":"NoOp","type":"main","index":0}]]}},
    "settings":{}
  }}}'

# 2. Execute it (auto activate + trigger webhook)
curl -s --max-time 30 -X POST http://172.17.0.1:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"execute_workflow","arguments":{"id":"<WORKFLOW_ID>"}}}'

# 3. Check execution history
curl -s --max-time 30 -X POST http://172.17.0.1:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"list_executions","arguments":{"limit":5}}}'

# 4. Delete when done
curl -s --max-time 30 -X POST http://172.17.0.1:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"delete_workflow","arguments":{"id":"<WORKFLOW_ID>"}}}'
```

## Important Notes

- **No session initialization needed** — POST directly to `/mcp`
- **Parse SSE response**: look for lines starting with `data:` containing JSON
- **Always include `"settings": {}` when creating or updating workflows** (n8n API requirement)
- **Update workflow cần gửi `nodes`** ở top-level, không nested trong `workflow` object
- The MCP server connects to n8n internally via Docker network
- n8n instance UI: http://localhost:5678
- Webhook mới tạo qua API cần ~5-10s để n8n đăng ký (queue mode)
