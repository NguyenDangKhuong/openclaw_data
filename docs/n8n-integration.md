# OpenClaw + n8n Integration

## Kiến trúc

```
OpenClaw AI ──curl──▶ MCP Server (port 3002) ──API──▶ n8n (port 5678)
                            │                              │
                      n8n-custom-mcp                  n8n + FFmpeg
                      31 tools                        Workflows
                      SSE transport                   PostgreSQL + Redis
```

## Cách hoạt động

1. **OpenClaw** đọc skill file `skills/n8n-mcp/SKILL.md` khi khởi động
2. AI agent **tự biết** có thể gọi 31 tools qua `curl` POST requests
3. Mỗi request gửi đến `http://172.17.0.1:3002/mcp` (MCP server)
4. MCP server gọi n8n API → trả kết quả cho AI

## Yêu cầu

- n8n stack đang chạy (`docker compose up -d` trong `/home/khuong/Downloads/Source/n8n/`)
- MCP server healthy: `curl -s http://localhost:3002/` → `{"status":"running"}`
- Skill `n8n-mcp` enabled trong `openclaw.json`:

```json
{
  "skills": {
    "entries": {
      "n8n-mcp": { "enabled": true }
    }
  }
}
```

## Setup từ đầu

### 1. Start n8n stack

```bash
cd /home/khuong/Downloads/Source/n8n
docker compose up -d
```

Kiểm tra tất cả services healthy:
```bash
docker compose ps
# postgres, redis, n8n, n8n-worker, n8n-mcp đều phải "Up"
```

### 2. Tạo API Key

1. Mở http://localhost:5678 → đăng nhập
2. Settings → API → Create API Key
3. Copy API key → update `/home/khuong/Downloads/Source/n8n/.env`:
   ```
   N8N_API_KEY=<api-key>
   ```
4. Restart MCP: `docker compose up -d --force-recreate n8n-mcp`

### 3. Test MCP connection

```bash
# Server status
curl -s http://localhost:3002/
# → {"status":"running","transport":"sse/hybrid","tools_count":31}

# List workflows
curl -s -X POST http://localhost:3002/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"list_workflows","arguments":{}}}'
```

### 4. Enable skill trong OpenClaw

Thêm vào `openclaw_data/openclaw.json` → `skills.entries`:
```json
"n8n-mcp": { "enabled": true }
```

### 5. Test từ OpenClaw

Chat với AI:
- "Liệt kê tất cả workflows trong n8n"
- "Tạo workflow mới có webhook trigger"
- "Bật workflow Post Facebook"

## IP Address

OpenClaw container gọi MCP qua `172.17.0.1` (Docker bridge gateway IP).

Nếu IP khác, check:
```bash
ip route | grep docker0
# → 172.17.0.0/16 dev docker0 ...
# IP gateway là 172.17.0.1
```

## Troubleshooting

### OpenClaw không gọi được MCP
```bash
# Kiểm tra từ host
curl -s http://localhost:3002/
# → {"status":"running",...}

# Kiểm tra từ Docker bridge IP
curl -s http://172.17.0.1:3002/
# → phải giống kết quả trên
```

### MCP trả lỗi "unauthorized"
API key trong `.env` không khớp → tạo key mới trong n8n Settings → API.

### MCP trả lỗi "Cannot activate an archived workflow"
Workflow đã bị archive → unarchive trong n8n UI trước.

### Webhook trả 404
Webhook mới tạo cần ~5-10s để n8n đăng ký (queue mode). Thử lại sau vài giây.
