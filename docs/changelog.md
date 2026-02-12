# OpenClaw Configuration Changelog

## 2026-02-12 — Initial Setup & Fix Configuration

### Vấn đề gốc
- OpenClaw gateway khởi động nhưng báo lỗi config: `Unrecognized key: "apiProvider"`
- Key `apiProvider` không tồn tại trong schema, khiến gateway chạy ở chế độ best-effort

### Thay đổi 1: Sửa config schema
**File:** `openclaw.json`

- ❌ Xóa key sai: `agents.defaults.apiProvider`
- ✅ Thêm `models.providers.cli-proxy` — cấu hình custom provider đúng schema
  - `baseUrl`: `https://cli-proxy.thetaphoa.store/v1`
  - `apiKey`: `khuong`
  - `api`: `openai-completions`
- ✅ Thêm `models.mode: "replace"` — chặn fallback sang built-in providers (Google OAuth, Anthropic), chỉ dùng CLI Proxy

### Thay đổi 2: Fix model trả response rỗng
**File:** `openclaw.json`

- **Vấn đề:** Model `claude-opus-4-6-thinking` trả response rỗng vì CLI Proxy chưa có auth (Google OAuth hết hạn)
- **Giải pháp tạm:** Đổi primary model sang `gemini-2.5-pro` (hoạt động OK)
- **Giải pháp cuối:** Sau khi re-authenticate OAuth trên CLI Proxy, đổi primary model về `claude-opus-4-6-thinking`

### Thay đổi 3: Cấu hình model list & aliases
**File:** `openclaw.json`

- Thêm models vào `models.providers.cli-proxy.models` với `contextWindow`, `maxTokens`, `input` capabilities
- Cấu hình `agents.defaults.models` với aliases cho phép dùng `/model <alias>` trong chat

---

## 2026-02-12 — Google quét auth, xóa CLI auth

### Vấn đề
- Google quét phát hiện auth bất thường → xóa CLI auth trên CLI Proxy
- Mất tất cả Claude models (dùng qua Google OAuth channel `antigravity`)
- Mất một số model preview (`tab_flash_lite_preview`, `gpt-oss-120b-medium`, ...)

### Thay đổi 4: Đổi primary model
**File:** `openclaw.json`

- Đổi primary model từ `claude-opus-4-6-thinking` → `gemini-3-pro-preview`
- Cập nhật fallbacks: `gemini-2.5-pro`, `gpt-5.2`, `gemini-2.5-flash`
- Xóa Claude models khỏi aliases (không còn khả dụng)

---

## 2026-02-12 — Thêm kênh Telegram

### Thay đổi 5: Kết nối Telegram Bot
**File:** `openclaw.json`

- Thêm `channels.telegram` với bot `@heyyolo_bot`
- Config: `dmPolicy: "open"`, `allowFrom: ["*"]`, `groups: requireMention: true`
- Thêm `plugins.entries.telegram.enabled: true`

---

## 2026-02-12 — Thêm kênh Zalo Bot

### Thay đổi 6: Kết nối Zalo Bot API
**File:** `openclaw.json`

- Thêm `channels.zalo` với bot token từ https://bot.zaloplatforms.com
- Config: `dmPolicy: "open"`, `allowFrom: ["*"]`, `mediaMaxMb: 5`
- Thêm `plugins.entries.zalo.enabled: true` và `plugins.allow: ["telegram", "zalo"]`
- Zalo dùng long-polling (không cần public URL)
- **Lưu ý:** Chỉ hỗ trợ DM, chưa hỗ trợ group chat

### Restart gateway
```bash
cd /home/khuong/Downloads/Source/openclaw
docker compose restart openclaw-gateway
```
