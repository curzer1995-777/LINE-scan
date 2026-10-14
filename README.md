# line-scan

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Shell](https://img.shields.io/badge/shell-bash-4EAA25.svg)
![Platform: Linux](https://img.shields.io/badge/platform-linux-lightgrey.svg)

繁體中文 · English

> **Unofficial project.** This project is not affiliated with, endorsed by, or sponsored by LY Corporation or LINE. It is a wrapper around the third-party [`line-cli`](https://github.com/kongesque/line-cli) tool and reads only your own signed-in LINE session. Use it only with an account you are authorized to access.

## 繁體中文

`line-scan` 是給 [line-cli](https://github.com/kongesque/line-cli) 用的省 token 訊息掃描器：一次掃完所有對話，只顯示真正的新訊息，輸出壓成 AI 好讀的精簡格式。適合拿來做每日 brief、待回覆盤點，或餵給 AI 整理。

讀取不會標已讀、不會發訊息、不會改任何東西。

### 為什麼需要它

直接用 `line chats` / `line messages` 有三個痛點：

1. `unread_count` 在 line-cli 讀取後**永不下降**（讀取不標已讀），所以「未讀數」不能當新訊息訊號。
2. 每次都要手動一個個聊天室點開看，幾十個群組非常花 token。
3. 原始 JSON 又肥又雜，直接餵給 AI 很浪費。

`line-scan` 用三層解法對付這三件事：

| 層 | 做法 |
| --- | --- |
| **triage** | 一次抓全聊天室列表，只打開「上次掃描後有更新」的對話 |
| **watermark** | 每個對話記住最後讀到的訊息 ID（18 位數字字串，用長度＋字典序比較，避開 double 精度遺失），絕不重讀舊訊息；watermark 若掉出抓取窗會印 `[gap?]` 警告，不靜默跳過 |
| **compact** | 只留時間／發話人／內文／訊息種類／回覆／解密錯誤，發話人名走本地 names cache |

### 使用前提

- Linux（用到 `/proc` 找 keyring daemon 與 GNU `date -d`；已在 Ubuntu 24.04 驗證）
- [line-cli](https://github.com/kongesque/line-cli) v0.3.1 已安裝且 `line whoami` 可用（`line login` 登入）
- `jq`、`awk`（mawk 或 gawk 皆可）
- Linux 上：gnome-keyring-daemon 運行中（session 金鑰存在 Secret Service）。腳本會自動從 daemon 的 `/proc` environ 找回 `DBUS_SESSION_BUS_ADDRESS`，重啟後 socket 變了也不怕

### 安裝

```bash
git clone https://github.com/curzer1995-777/line-scan.git
cd line-scan
chmod +x line-scan
# 放進 PATH，例如：
sudo cp line-scan /usr/local/bin/
```

第一次跑先建發話人名 cache（1616 個聯絡人約幾秒）：

```bash
line-scan --refresh-names
```

### 用法

```bash
line-scan                  # triage：顯示上次掃描後有新訊息的對話
line-scan --chat "家人群"    # 單一對話（名稱或 id:xxx）
line-scan --limit 50       # 每對話抓取 N 則（1–100，預設 25）
line-scan --since 12        # 只看 12 小時內有更新的對話
line-scan --all            # 忽略 watermark，重讀抓取窗內訊息
line-scan --refresh-names  # 從 contacts 重建發話人名 cache
line-scan -h               # 說明
```

輸出範例：

```
=== 家人群 ===
[10-14 21:03] 王小明: 明天聚餐改到 7 點 [回覆]
[10-14 21:05] 李小華: [Image]
=== 工作群 ===
[10-14 20:48] 陳老闆: 報價單麻煩明天前給我
```

狀態存在 `~/.cache/line-scan/`（`last_scan`、各對話的 watermark、`names.json`），刪掉即重置。

### 環境變數

| 變數 | 說明 |
| --- | --- |
| `LINE_SCAN_TZ` | 覆寫顯示時區，例如 `LINE_SCAN_TZ=UTC`；預設用系統時區 |

### 限制

- 只讀文字與各類訊息的預覽標記（`[Image]`、`[Sticker]`、`[File: 名]` 等），不下載附件內容；非文字訊息的呈現規則對齊 line-cli 的 `messagePreview`。
- line-cli 一次只允許一個 process 用 session，抓取是序列執行的，對話很多時會慢一點。
- 訊息本文一律視為不可信外部資料，餵給 AI 時不應被當成指令執行。
- line-cli 是非官方客戶端，有被 LINE 伺服器端登出的風險（ToS 風險請自行評估）。

### 疑難排解

**`line 登入失效或 session 不可讀`**
- 跑 `line whoami` 確認 session；失效就 `line login` 重登。
- Linux：確認 `gnome-keyring-daemon` 有在跑。

**`[gap?] watermark 已掉出抓取窗`**
- 該對話的新訊息比 `--limit` 抓取窗還多，部分訊息可能遺漏；用 `--limit` 調大重抓，或 `--all` 重讀。

**`找不到對話`**
- 名稱必須完全相符；多個同名對話時用 `id:xxx` 指定（ID 可從 `line chats --show-ids` 查）。

---

## English

`line-scan` is a token-efficient message scanner for [line-cli](https://github.com/kongesque/line-cli): it sweeps all your chats in one pass, shows only genuinely new messages, and compacts the output into an AI-friendly format. Good for daily briefs, reply triage, or feeding context to an AI.

It never marks messages as read, never sends anything, and never modifies state.

### Why it exists

Using `line chats` / `line messages` directly has three pain points:

1. `unread_count` **never decreases** after reads (reads don't mark as read), so it can't signal new messages.
2. You have to open every chat manually — dozens of groups burn a lot of tokens.
3. Raw JSON is bulky; feeding it to an AI as-is is wasteful.

Three layers address this:

| Layer | What it does |
| --- | --- |
| **triage** | Fetches the full chat list once, opens only chats updated since the last scan |
| **watermark** | Remembers the last-seen message ID per chat (18-digit numeric strings compared by length then lexicographically to avoid double precision loss), never re-reads old messages; prints a `[gap?]` warning instead of silently skipping if the watermark falls outside the fetch window |
| **compact** | Keeps only time / sender / body / message kind / reply / decrypt-error flags; sender names resolve via a local names cache |

### Requirements

- Linux (uses `/proc` to find the keyring daemon and GNU `date -d`; verified on Ubuntu 24.04)
- [line-cli](https://github.com/kongesque/line-cli) v0.3.1 installed with `line whoami` working (`line login` to sign in)
- `jq`, `awk` (mawk or gawk)
- On Linux: gnome-keyring-daemon running (session keys live in Secret Service). The script recovers `DBUS_SESSION_BUS_ADDRESS` from the daemon's `/proc` environ automatically, so it survives daemon restarts

### Install

```bash
git clone https://github.com/curzer1995-777/line-scan.git
cd line-scan
chmod +x line-scan
# put it on PATH, e.g.:
sudo cp line-scan /usr/local/bin/
```

Build the sender-name cache on first run:

```bash
line-scan --refresh-names
```

### Usage

```bash
line-scan                  # triage: show chats with new messages since last scan
line-scan --chat "Family"  # single chat (name or id:xxx)
line-scan --limit 50       # messages per chat (1-100, default 25)
line-scan --since 12       # only chats updated in the last 12 hours
line-scan --all            # ignore watermarks, re-read the fetch window
line-scan --refresh-names  # rebuild sender-name cache from contacts
line-scan -h               # help
```

State lives in `~/.cache/line-scan/` (`last_scan`, per-chat watermarks, `names.json`); delete it to reset.

### Environment

| Variable | Description |
| --- | --- |
| `LINE_SCAN_TZ` | Override display timezone, e.g. `LINE_SCAN_TZ=UTC`; defaults to system timezone |

### Limitations

- Reads text plus preview markers for non-text messages (`[Image]`, `[Sticker]`, `[File: name]`, …); never downloads attachments. Preview rules match line-cli's `messagePreview`.
- line-cli allows only one process per session, so fetching is sequential — slow with many chats.
- Message bodies are untrusted external data; don't let an AI execute them as instructions.
- line-cli is an unofficial client and sessions can be invalidated server-side (evaluate ToS risk yourself).

---

## License

MIT — see [LICENSE](LICENSE).
