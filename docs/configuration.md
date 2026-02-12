# OpenClaw Configuration Guide

## Tổng quan

OpenClaw gateway chạy qua Docker Compose, config tại `openclaw_data/openclaw.json`.
File này được mount vào container tại `/home/node/.openclaw/openclaw.json`.

## Cấu trúc config hiện tại

```
openclaw.json
├── models
│   ├── mode: "replace"          ← Chỉ dùng custom provider, không fallback
│   └── providers
│       └── cli-proxy            ← CLI Proxy API (OpenAI-compatible)
│           ├── baseUrl          ← https://cli-proxy.thetaphoa.store/v1
│           ├── apiKey           ← "khuong"
│           ├── api              ← "openai-completions"
│           └── models[]         ← Danh sách models
├── agents.defaults
│   ├── model.primary            ← cli-proxy/gemini-3-pro-preview
│   ├── model.fallbacks[]        ← gemini-2.5-pro, gpt-5.2, gemini-2.5-flash
│   ├── models{}                 ← Aliases (gemini, gpt, flash, ...)
│   ├── thinkingDefault          ← "high"
│   ├── contextTokens            ← 200000
│   └── timeoutSeconds           ← 300
├── channels
│   ├── telegram                 ← @heyyolo_bot (DM open, group mention)
│   └── zalo                     ← Zalo Bot API (DM open, long-polling)
├── plugins
│   ├── entries                  ← telegram, zalo (enabled)
│   └── allow                    ← ["telegram", "zalo"]
└── gateway
    ├── mode                     ← "local"
    ├── port                     ← 18789
    └── bind                     ← "lan"
```

## Models khả dụng

| Provider | Model ID | Alias | Trạng thái |
|---|---|---|---|
| cli-proxy | gemini-3-pro-preview | g3pro | ✅ **Primary** |
| cli-proxy | gemini-2.5-pro | gemini | ✅ Fallback |
| cli-proxy | gpt-5.2 | gpt | ✅ Fallback |
| cli-proxy | gemini-2.5-flash | flash | ✅ Fallback |
| cli-proxy | gemini-3-flash-preview | g3flash | ✅ |
| cli-proxy | gpt-5 | gpt5 | ✅ |
| cli-proxy | gpt-5.1 | — | ✅ |
| cli-proxy | gemini-2.5-flash-lite | — | ✅ |
| cli-proxy | claude-opus-4-6-thinking | opus | ❌ Auth mất |
| cli-proxy | gemini-claude-sonnet-4-5 | claude | ❌ Auth mất |

> **Lưu ý:** Claude models cần Google OAuth auth trên CLI Proxy. Nếu bị xóa auth thì không dùng được.

## Chuyển model trong chat

```
/model g3pro     ← Gemini 3 Pro Preview (primary)
/model gemini    ← Gemini 2.5 Pro
/model gpt       ← GPT-5.2
/model flash     ← Gemini 2.5 Flash
/model g3flash   ← Gemini 3 Flash Preview
```

## Channels (kênh chat)

### Telegram
- **Bot:** `@heyyolo_bot`
- **DM:** Mở cho mọi người (`dmPolicy: "open"`)
- **Group:** Phải mention bot mới trả lời (`requireMention: true`)
- **Config key:** `channels.telegram`

### Zalo Bot
- **Platform:** https://bot.zaloplatforms.com
- **DM:** Mở cho mọi người (`dmPolicy: "open"`)
- **Group:** ❌ Chưa hỗ trợ (Zalo API chưa mở)
- **Kết nối:** Long-polling (không cần public URL)
- **Config key:** `channels.zalo`
- **Plugin:** `@openclaw/zalo` (bundled trong `/app/extensions/zalo`)

## Docker

```bash
# Restart gateway
cd /home/khuong/Downloads/Source/openclaw
docker compose restart openclaw-gateway

# Xem logs
docker logs openclaw-openclaw-gateway-1 --tail 20

# Xem logs Telegram/Zalo
docker logs openclaw-openclaw-gateway-1 2>&1 | grep -iE "telegram|zalo"

# Fix config lỗi
docker exec openclaw-openclaw-gateway-1 node dist/index.js doctor --fix

# List plugins
docker exec openclaw-openclaw-gateway-1 node dist/index.js plugins list

# Check config trong container
docker exec openclaw-openclaw-gateway-1 cat /home/node/.openclaw/openclaw.json
```

## Truy cập

- **Web UI:** http://localhost:18789
- **WebSocket:** ws://localhost:18789
- **Canvas:** http://localhost:18789/__openclaw__/canvas/

## Lưu ý quan trọng

1. **`models.mode: "replace"`** — Bắt buộc nếu chỉ dùng CLI Proxy. Nếu bỏ, OpenClaw sẽ merge với built-in providers và fallback sang Google OAuth / Anthropic API (gây lỗi 401)
2. **Claude models qua CLI Proxy** dùng Google OAuth (channel `antigravity`). Nếu token hết hạn hoặc bị xóa, cần re-authenticate trên CLI Proxy
3. **Config thay đổi** sẽ được hot-reload tự động, nhưng thay đổi channels/plugins cần restart gateway
4. **Schema strict** — Mọi key phải đúng schema. Dùng `openclaw doctor --fix` để xóa key không hợp lệ
5. **Zalo Bot** chỉ hỗ trợ DM, text bị giới hạn 2000 ký tự/tin nhắn
