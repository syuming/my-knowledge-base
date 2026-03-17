# OpenClaw Mission Control 完整安裝指南

> v1.3 · 2026-03-08  
> 適用環境：GCP Ubuntu VM (e2-medium, 30GB)、已安裝 OpenClaw Gateway、已設定 Tailscale

---

## 前置確認清單

| 項目 | 說明 |
|------|------|
| GCP VM | e2-medium（2 vCPU, 4GB RAM）以上 |
| 磁碟空間 | 至少 10GB 剩餘空間（`df -h` 確認） |
| OpenClaw Gateway | 已安裝並以 systemd 管理（`sudo systemctl status openclaw`） |
| Tailscale | 已設定並取得 Tailnet URL |

> ⚠️ **磁碟空間不足**：Docker build 需要約 3～5GB，建議根目錄剩餘 10GB 以上再開始。

---

## 架構說明

Mission Control 包含四個容器：

| 容器 | 說明 | Port |
|------|------|------|
| frontend | Next.js 網頁介面 | 3000 |
| backend | FastAPI REST API | 8000 |
| db | PostgreSQL 資料庫 | 5432（僅內部） |
| redis | Redis 工作佇列 | 6379（僅內部） |

---

## STEP 1：安裝 Docker

```bash
curl -fsSL https://get.docker.com | bash
sudo usermod -aG docker $USER
newgrp docker
```

確認版本：

```bash
docker --version && docker compose version
```

> ✅ 正確結果：`Docker version 29.x.x` 和 `Docker Compose version v5.x.x`

---

## STEP 2：Clone 專案

```bash
cd ~
git clone https://github.com/abhi1693/openclaw-mission-control.git
cd openclaw-mission-control
```

---

## STEP 3：設定環境變數

### 複製設定檔

```bash
cp .env.example .env
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env.local
```

### 關閉 Clerk 認證（本地自架不需要）

```bash
sed -i 's/^NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=.*/NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=/' frontend/.env.local
```

### 產生 LOCAL_AUTH_TOKEN

```bash
TOKEN=$(openssl rand -hex 32)
echo "TOKEN: $TOKEN"  # 記下這串，登入時需要用到
```

### 修正 backend/.env

```bash
# 修正資料庫連線（localhost → db）
sed -i 's|postgresql+psycopg://postgres:postgres@localhost:5432|postgresql+psycopg://postgres:postgres@db:5432|' backend/.env

# 修正 Redis 連線（localhost → redis）
sed -i 's|redis://localhost:6379|redis://redis:6379|' backend/.env

# 填入 LOCAL_AUTH_TOKEN
sed -i "s|LOCAL_AUTH_TOKEN=|LOCAL_AUTH_TOKEN=$TOKEN|" backend/.env

# 填入 BASE_URL（換成你的 Tailscale URL）
sed -i 's|^BASE_URL=$|BASE_URL=https://你的tailscale網址|' backend/.env

# 填入 CORS_ORIGINS
sed -i 's|^CORS_ORIGINS=.*|CORS_ORIGINS=http://localhost:3000,https://你的tailscale網址,https://你的tailscale網址:8443|' backend/.env
```

### 修正根目錄 .env

```bash
# 移除空的 LOCAL_AUTH_TOKEN 行
sed -i '/^LOCAL_AUTH_TOKEN=$/d' .env

# 填入必要變數
echo "LOCAL_AUTH_TOKEN=$TOKEN" >> .env
echo "AUTH_MODE=local" >> .env
echo "BASE_URL=https://你的tailscale網址" >> .env
echo "CORS_ORIGINS=http://localhost:3000,https://你的tailscale網址,https://你的tailscale網址:8443" >> .env

# 修正 NEXT_PUBLIC_API_URL（換成你的 Tailscale URL + :8443）
sed -i 's|^NEXT_PUBLIC_API_URL=.*|NEXT_PUBLIC_API_URL=https://你的tailscale網址:8443|' .env
```

確認所有關鍵變數：

```bash
grep -E "LOCAL_AUTH_TOKEN|AUTH_MODE|BASE_URL|CORS_ORIGINS|NEXT_PUBLIC_API_URL" .env
grep -E "DATABASE_URL|RQ_REDIS_URL|LOCAL_AUTH_TOKEN|BASE_URL|CORS_ORIGINS" backend/.env
```

---

## STEP 4：設定 Tailscale 服務對應

```bash
# 預設 443 → Mission Control
tailscale serve --bg --https=443 localhost:3000

# 8443 → Backend API
tailscale serve --bg --https=8443 localhost:8000

# 8444 → OpenClaw Gateway UI（原本的介面）
tailscale serve --bg --https=8444 localhost:18789

# 確認
tailscale serve status
```

> ✅ 完成後服務對應：
> - `https://你的tailscale網址` → Mission Control
> - `https://你的tailscale網址:8443` → Backend API
> - `https://你的tailscale網址:8444` → OpenClaw Gateway UI

---

## STEP 4.5：關閉 OpenClaw 自動覆蓋 Tailscale Serve（必做！）

> ⚠️ **這個步驟非常重要，跳過的話每次 openclaw 重啟都會把 443 設定蓋回 18789。**

OpenClaw Gateway 預設啟動時會自動呼叫 `tailscale serve`，將 443 指向自己（18789），覆蓋掉 STEP 4 的設定。透過修改 `openclaw.json` 把這個行為關掉，才能讓 Mission Control 的 443 設定永久生效。

```bash
sudo systemctl stop openclaw

python3 -c "
import json
with open('/home/jason/.openclaw/openclaw.json', 'r') as f:
    c = json.load(f)
c['gateway']['tailscale']['mode'] = 'off'
with open('/home/jason/.openclaw/openclaw.json', 'w') as f:
    json.dump(c, f, indent=2, ensure_ascii=False)
print('完成！tailscale mode:', c['gateway']['tailscale']['mode'])
"

sudo systemctl start openclaw
sleep 10
tailscale serve status
```

> ✅ 443 仍然指向 `localhost:3000` 表示設定成功，之後不管 openclaw 重啟幾次都不會再覆蓋。

> ⚠️ `jason` 換成你的實際帳號（`whoami` 確認）

---

## STEP 5：啟動所有容器

```bash
docker compose -f compose.yml --env-file .env up -d --build
```

> ⏱️ 第一次 build 需要 5～15 分鐘，耐心等候。

確認狀態：

```bash
docker compose -f compose.yml --env-file .env ps
curl http://localhost:8000/healthz
```

> ✅ 正確結果：五個容器全部 `Up`，healthz 回傳 `{"ok":true}`

---

## STEP 6：設定 Gateway 連線（socat 轉發）

Mission Control 跑在 Docker 容器內，無法直接連到 VM 的 `127.0.0.1:18789`，需要用 socat 做 Port 轉發。

### 安裝 socat

```bash
sudo apt install -y socat
```

### 建立 systemd 服務

```bash
sudo tee /etc/systemd/system/socat-gateway.service << 'EOF'
[Unit]
Description=Socat Gateway Port Forward 18790->18789
After=network.target openclaw.service
Requires=openclaw.service

[Service]
Type=simple
ExecStart=/usr/bin/socat TCP-LISTEN:18790,fork,reuseaddr TCP:127.0.0.1:18789
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable socat-gateway
sudo systemctl start socat-gateway
sudo systemctl status socat-gateway | head -5
```

> ✅ 正確結果：`Active: active (running)`

---

## STEP 7：設定開機自動啟動（Mission Control）

```bash
sudo tee /etc/systemd/system/openclaw-mission-control.service << 'EOF'
[Unit]
Description=OpenClaw Mission Control
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/jason/openclaw-mission-control
ExecStart=/usr/bin/docker compose -f compose.yml --env-file .env up -d
ExecStop=/usr/bin/docker compose -f compose.yml --env-file .env down
User=jason

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable openclaw-mission-control
```

> ⚠️ `User=jason` 換成你的實際帳號（`whoami` 確認）

---

## STEP 8：更新 OpenClaw allowedOrigins

把新的 Tailscale URL（含 8444 port）加入 OpenClaw 白名單：

```bash
sudo systemctl stop openclaw

python3 -c "
import json
with open('/home/jason/.openclaw/openclaw.json', 'r') as f:
    c = json.load(f)
origins = c['gateway'].get('controlUi', {}).get('allowedOrigins', [])
for url in ['https://你的tailscale網址', 'https://你的tailscale網址:8444']:
    if url not in origins:
        origins.append(url)
c['gateway'].setdefault('controlUi', {})['allowedOrigins'] = origins
with open('/home/jason/.openclaw/openclaw.json', 'w') as f:
    json.dump(c, f, indent=2, ensure_ascii=False)
print('allowedOrigins:', origins)
"

sudo systemctl start openclaw
```

---

## STEP 9：登入 Mission Control 並連接 Gateway

### 登入

1. 開瀏覽器前往 `https://你的tailscale網址`
2. 填入 Profile（Name、Timezone: Asia/Taipei）
3. 填入 Access Token（STEP 3 產生的 `$TOKEN`）

### 連接 Gateway

1. 點左側選單 **Gateways** → **Create gateway**

2. 填入：

| 欄位 | 填入值 |
|------|--------|
| Gateway name | `GCP Primary` |
| Gateway URL | `ws://172.17.0.1:18790` |
| Gateway token | OpenClaw 的 token（見下方指令） |
| Workspace root | `~/.openclaw` |

取得 OpenClaw Token：

```bash
grep '"token"' ~/.openclaw/openclaw.json
```

3. 按 **Create gateway**，出現 `pairing required` 表示連線成功

### 裝置配對

```bash
openclaw devices list
openclaw devices approve <requestId>
```

---

## 完整服務清單

| 服務 | 管理方式 | 說明 |
|------|---------|------|
| `openclaw` | systemd | OpenClaw Gateway |
| `socat-gateway` | systemd | Port 轉發 18790→18789 |
| `openclaw-mission-control` | systemd | Docker Compose 自動啟動 |

---

## 日常維護指令

```bash
# 確認所有服務狀態
sudo systemctl status openclaw socat-gateway openclaw-mission-control

# Mission Control 容器狀態
cd ~/openclaw-mission-control
docker compose -f compose.yml --env-file .env ps

# 即時查看 log
sudo journalctl -u openclaw -f
docker compose -f compose.yml --env-file .env logs -f backend
docker compose -f compose.yml --env-file .env logs -f frontend

# 重啟 Mission Control
docker compose -f compose.yml --env-file .env restart
```

---

## 常見問題排查

| 症狀 | 原因 | 解法 |
|------|------|------|
| `Unable to reach backend to validate token` | Frontend 打到錯誤的 API URL | 確認 `NEXT_PUBLIC_API_URL` 設定，重新 build frontend |
| `CORS error` | Backend CORS 白名單未包含 Tailscale URL | 更新 `CORS_ORIGINS`，重啟 backend |
| `[Errno 111] Connection refused` | Gateway 只綁定 127.0.0.1，Docker 無法連到 | 確認 socat-gateway 服務有在跑 |
| `pairing required` | 新裝置（瀏覽器）需要配對 | `openclaw devices list` → `approve` |
| 容器一直重啟 | `.env` 變數缺少或錯誤 | 查看 log：`docker compose logs <服務名>` |
| 重開機後容器消失 | 未設定 systemd 自動啟動 | 執行 STEP 7 |
| **換電腦連線後，443 port 又跑回 OpenClaw Gateway，而不是 Mission Control** | 安裝 OpenClaw 時留下的舊 Tailscale Serve 設定（443 → 18789）沒有被正確覆蓋 | 參考下方「Tailscale Port 對應跑回舊設定」排查流程 |
| **openclaw 重啟後 443 又跑回 18789** | openclaw 預設啟動時會自動覆蓋 Tailscale Serve 設定 | 執行 STEP 4.5，將 `tailscale.mode` 設為 `off` |

---

## 排查：Tailscale Port 對應跑回舊設定

### 症狀

換到另一台電腦連線後，發現 `https://你的tailscale網址` 顯示的還是舊的 OpenClaw Gateway UI，而不是 Mission Control。

### 原因

安裝 OpenClaw Gateway 時，教學指南第十節會執行：

```bash
tailscale serve localhost:18789
```

這個指令會把 **443 → 18789** 寫入 Tailscale 的永久設定檔（存在 VM 端）。  
如果 Mission Control 的 STEP 4 沒有成功覆蓋這筆設定，舊的對應就會繼續生效。

> ⚠️ Tailscale Serve 設定存在 **VM 端**，不是你的電腦上。在家裡覆蓋成功不代表設定一直是對的，換電腦時可以用 `tailscale serve status` 立刻確認現況。

### 修復步驟

SSH 進入 GCP VM 執行：

```bash
# 1. 確認目前設定（應該會看到 443 → 18789）
tailscale serve status

# 2. 移除舊的 443 設定
tailscale serve --https=443 off

# 3. 重新正確配置
tailscale serve --bg --https=443 localhost:3000     # Mission Control
tailscale serve --bg --https=8443 localhost:8000    # Backend API
tailscale serve --bg --https=8444 localhost:18789   # OpenClaw Gateway UI

# 4. 確認設定正確
tailscale serve status
```

正確結果：

```
443  → localhost:3000   ✅ Mission Control
8443 → localhost:8000   ✅ Backend API
8444 → localhost:18789  ✅ OpenClaw Gateway UI
```

### 修復完成後，若新電腦出現 pairing required

公司或新裝置是第一次連線，需要在 VM 上核准配對：

```bash
# 查看待配對裝置
openclaw devices list

# 核准（填入 Pending 欄位的 requestId）
openclaw devices approve <requestId>
```

核准後重新整理瀏覽器即可。

---

## 排查：openclaw 重啟後 Tailscale Serve 設定跑掉

### 症狀

每次 `sudo systemctl restart openclaw` 之後，`tailscale serve status` 顯示 443 又跑回指向 `127.0.0.1:18789`（OpenClaw），而不是 `localhost:3000`（Mission Control）。

### 原因

OpenClaw Gateway 的 `openclaw.json` 裡有一個 `tailscale.mode` 設定，預設值是 `serve`，代表每次啟動都會自動呼叫 `tailscale serve` 將 443 指向自己（18789），把 Mission Control 的設定覆蓋掉。

### 根治解法：將 tailscale.mode 設為 off

> ⚠️ `jason` 換成你的實際帳號

```bash
sudo systemctl stop openclaw

python3 -c "
import json
with open('/home/jason/.openclaw/openclaw.json', 'r') as f:
    c = json.load(f)
c['gateway']['tailscale']['mode'] = 'off'
with open('/home/jason/.openclaw/openclaw.json', 'w') as f:
    json.dump(c, f, indent=2, ensure_ascii=False)
print('完成！tailscale mode:', c['gateway']['tailscale']['mode'])
"

sudo systemctl start openclaw
sleep 10
tailscale serve status
```

> ✅ 443 仍然指向 `localhost:3000` 即代表根治成功。之後不管 openclaw 重啟幾次都不會再覆蓋。

---

## 網址總覽

| 功能 | 網址 |
|------|------|
| **Mission Control** | `https://你的tailscale網址` |
| **OpenClaw Gateway UI** | `https://你的tailscale網址:8444` |
| **Backend API healthz** | `https://你的tailscale網址:8443/healthz` |

