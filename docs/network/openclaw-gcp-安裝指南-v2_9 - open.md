# OpenClaw on GCP 完整安裝操作指南

> v2.9 底下的操作指令，雖然有可能已經不適合了，因為迭代太快了，主要是留個紀錄，讓我找資料比較快

**本文件適合對象**：有 GCP 帳號、已開啟 Billing、具備基本 Linux 指令能力的網路/系統管理員。  
全程使用瀏覽器版 SSH（GCP Console），不需事先設定本機 gcloud。

> 📋 **主流程共 8 個 STEP，照順序做即可。**  
> tmux 相關操作已移至文件最末的附錄，初學者不需要理會。

---

## 一、前置確認清單

| 項目 | 說明 / 取得方式 |
|------|----------------|
| GCP Billing 已啟用 | 前往 GCP Console → 帳單，確認帳戶已啟用付費 |
| AI API Key | 至 console.anthropic.com 或 platform.openai.com 建立 Key |
| Telegram Bot Token | 搜尋 @BotFather → /newbot → 取得 Token |

> ⚠️ **若尚未取得 API Key**  
> Anthropic：https://console.anthropic.com  
> OpenAI：https://platform.openai.com/api-keys  
> 兩者擇一，建議 Anthropic（Claude 模型）。

---

## 二、建立 GCP VM（STEP 1）

前往 GCP Console → Compute Engine → VM 執行個體 → 建立執行個體：

| 設定項目 | 建議值 |
|----------|--------|
| 機型 | e2-small（個人測試）或 e2-medium（正式使用） |
| 作業系統 | Ubuntu 24.04 LTS 或 Debian 12 |
| 磁碟大小 | **30 GB**（低於此值在 apt upgrade 時容易爆滿） |
| Zone | asia-east1-b（彰化機房，台灣延遲最低） |
| 防火牆 | 預設即可；OpenClaw Gateway 使用 18789 埠 |

> 💡 **省錢小技巧**  
> e2-small 每月費用約 USD 12～15，Spot VM 可省 60%～80%，但重啟後需手動恢復服務。  
> 磁碟一開始設 30 GB，避免日後執行擴容操作。

---

## 三、確認 SSH 帳號（STEP 2）

這是安裝失敗最常見的根本原因。GCP 有兩種 SSH 入口，帳號名稱可能不同：

| SSH 方式 | 帳號範例 |
|----------|---------|
| GCP Console 瀏覽器 SSH 按鈕 | `evanslin` |
| 本機 gcloud compute ssh | `evan_lin_yourdomain_com`（格式不同！） |

兩種視窗各執行一次，確認帳號是否一致：

```bash
whoami
```

若不同，往後 gcloud SSH 需明確指定帳號：

```bash
gcloud compute ssh evanslin@openclaw-myvm
```

> 💡 整個安裝過程全程使用「瀏覽器版 SSH」，避免帳號切換造成路徑混亂。

---

## 四、基礎系統更新（STEP 3）

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl build-essential
```

---

## 五、用 NVM 安裝 Node.js（STEP 4）

### 為什麼不直接 apt install nodejs？

GCP 預設以 SSH Key 登入，帳號無密碼。直接跑官方安裝腳本會遇到：

```
sudo-rs: I'm sorry evanslin. I'm afraid I can't do that
```

解法：用 NVM 把 Node.js 裝在使用者目錄下，完全繞過 sudo 問題。

```bash
# 1. 安裝 NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# 2. 重新載入環境（必要！）
source ~/.bashrc

# 3. 安裝 Node.js
nvm install node

# 4. 確認版本（應顯示 v25.x.x 以上）
node --version
npm --version
```

### 確認 ~/.bashrc 有以下三行

```bash
grep -n "NVM_DIR" ~/.bashrc
```

若沒有，手動加入：

```bash
cat >> ~/.bashrc << 'EOF'
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
EOF
source ~/.bashrc
```

---

## 六、安裝 OpenClaw（STEP 5）

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

確認安裝正確：

```bash
which openclaw     # 預期：~/.nvm/versions/node/v25.x.x/bin/openclaw
openclaw --version
```

---

## 七、初始設定（STEP 6）

```bash
openclaw onboard
```

依照互動精靈逐步完成：
1. 選擇 AI 模型供應商（Anthropic 或 OpenAI）
2. 貼上 API Key
3. 選擇通訊頻道（建議選 Telegram）
4. 貼上 Telegram Bot Token

---

## 八、建立啟動腳本（STEP 7）

統一透過此腳本啟動，確保環境變數正確載入：

```bash
cat << 'EOF' > ~/start_openclaw.sh
#!/bin/bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
export PATH="$HOME/.nvm/versions/node/v25.8.0/bin:$PATH"
openclaw "$@"
EOF
chmod +x ~/start_openclaw.sh
```

> ⚠️ `v25.8.0` 請換成你的實際版本，用 `node --version` 確認。版本號錯誤會導致 `openclaw: command not found`。

---

## 九、設定開機自動啟動（STEP 8）

設定 systemd 後，Gateway 開機自動啟動、崩潰自動重啟，不需要手動操作。

### Step 1：確認你的 Node.js 路徑

```bash
node --version   # 記下版本號，例如 v25.8.0
which openclaw   # 記下完整路徑
```

### Step 2：建立 systemd 服務

> ⚠️ **必須替換的兩個地方**  
> `User=` 和 `WorkingDirectory=` 的 `jason` 換成你的帳號（用 `whoami` 確認）  
> `v25.8.0` 換成你的實際 Node 版本（用 `node --version` 確認）

```bash
sudo tee /etc/systemd/system/openclaw.service << 'EOF'
[Unit]
Description=OpenClaw AI Gateway
After=network.target tailscaled.service
Wants=network.target

[Service]
Type=simple
User=jason
WorkingDirectory=/home/jason
Environment=NVM_DIR=/home/jason/.nvm
Environment=PATH=/home/jason/.nvm/versions/node/v25.8.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/home/jason/.nvm/versions/node/v25.8.0/bin/openclaw gateway
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

### Step 3：啟用並啟動

```bash
sudo systemctl daemon-reload
sudo systemctl enable openclaw    # 開機自動啟動
sudo systemctl start openclaw     # 立即啟動
```

### Step 4：確認狀態

```bash
sudo systemctl status openclaw
# 看到 Active: active (running) 表示成功
```

### 常用管理指令

| 指令 | 用途 |
|------|------|
| `sudo systemctl status openclaw` | 查看目前狀態 |
| `sudo systemctl restart openclaw` | 重啟 Gateway |
| `sudo systemctl stop openclaw` | 停止 Gateway |
| `sudo journalctl -u openclaw -f` | 即時查看 log |

> ✅ **完成！** 之後重開機 Gateway 會自動啟動。日常管理只需要用上方指令即可。  
> 不需要、也不要手動執行 `openclaw gateway`，否則會造成衝突（詳見十五、排查流程）。

### Step 5：檢查是否有 user 層級的衝突服務（重要！）

> 💰 **這個步驟會影響你的 API 費用！**  
> OpenClaw 安裝時會自動建立一個 user 層級的 `openclaw-gateway.service`。  
> 若沒有停用，它會和剛才設定的系統層級 systemd **同時運行兩個 Gateway**，導致：  
> - Telegram Bot **每則訊息回覆兩次**  
> - API token **消耗翻倍**，一天就能燒完所有額度

```bash
# 檢查 user 層級是否有 openclaw 相關服務
systemctl --user list-units | grep openclaw
```

若出現 `openclaw-gateway.service`，立刻停用：

```bash
systemctl --user stop openclaw-gateway
systemctl --user disable openclaw-gateway
```

確認停用成功：

```bash
# 確認 user 層級已清除
systemctl --user list-units | grep openclaw
# 應該沒有任何輸出

# 確認整機只有一個 openclaw-gateway process
ps aux | grep openclaw
# 正確結果：只有一個 openclaw-gateway 和 grep 那行
```

---

## 九之二、關閉 OpenClaw 自動覆蓋 Tailscale Serve（安裝 Mission Control 後必做）

> 💡 **僅在已安裝 Mission Control 的情況下需要執行此節。**  
> 純 OpenClaw Gateway 不需要。

### 問題說明

OpenClaw Gateway 的 `openclaw.json` 裡有一個 `tailscale.mode` 設定，預設值是 `serve`，代表每次啟動都會自動呼叫 `tailscale serve` 將 443 指向自己（18789）。如果你同時安裝了 Mission Control，443 應該要指向 Mission Control（3000），但每次 openclaw 重啟就會被覆蓋回去。

根治方法是直接把 `tailscale.mode` 改成 `off`，讓 openclaw 完全不干涉 Tailscale Serve 設定。

### 合法的 tailscale.mode 值

| 值 | 說明 |
|----|------|
| `serve` | 預設值，openclaw 自動管理 Tailscale Serve（安裝 Mission Control 後不能用） |
| `funnel` | 使用 Tailscale Funnel 對外公開 |
| `off` | 完全不碰 Tailscale Serve，由使用者自行管理 ✅ |

### 執行步驟

> ⚠️ `jason` 換成你的實際帳號（`whoami` 確認）

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

正確結果（443 沒有被覆蓋）：

```
443  → localhost:3000   ✅ Mission Control
8443 → localhost:8000   ✅ Backend API
8444 → localhost:18789  ✅ OpenClaw Gateway UI
```

> ✅ 設定完成後，之後不管 openclaw 重啟幾次，Tailscale Serve 設定都不會再被覆蓋。

---

## 十、Tailscale 遠端存取 Gateway UI

透過 Tailscale 可以在自己的電腦上安全地存取 GCP VM 的 OpenClaw 控制介面，不需要開放公開 IP。

### Step 1：設定 operator 權限（避免每次都要 sudo）

```bash
sudo tailscale set --operator=$USER
```

### Step 2：啟動 Tailscale Serve

```bash
tailscale serve localhost:18789
```

成功後會顯示：

```
Available within your tailnet:
https://instance-xxxxx.tail23ce2f.ts.net/
|-- proxy http://localhost:18789
```

記下這個 HTTPS URL，下一步會用到。

### Step 3：設定 allowedOrigins

OpenClaw 的 Control UI 有 Origin 白名單，Tailscale URL 需要加進去才能連線。

> ❌ **常見錯誤**  
> `origin not allowed (open the Control UI from the gateway host or allow it in gateway.controlUi.allowedOrigins)`  
> 出現這個錯誤就是 Tailscale URL 不在白名單，執行下方指令加入即可。

```bash
python3 -c "
import json
with open('/home/jason/.openclaw/openclaw.json', 'r') as f:
    c = json.load(f)
c['gateway'].setdefault('controlUi', {})['allowedOrigins'] = ['https://你的tailscale網址']
with open('/home/jason/.openclaw/openclaw.json', 'w') as f:
    json.dump(c, f, indent=2, ensure_ascii=False)
print('完成！')
"
```

> ⚠️ 將 `https://你的tailscale網址` 換成 Step 2 取得的實際 URL。  
> 例如：`https://instance-xxxxx.tail23ce2f.ts.net`

### Step 4：重啟 Gateway 讓設定生效

```bash
sudo systemctl restart openclaw
```

> ✅ 重新整理瀏覽器，右上角「健康狀況」顯示綠燈「連線」表示成功。

---

## 十一、首次連線 Device Pairing

第一次從瀏覽器連上 Gateway UI 時，需要完成配對才能使用。

### 找到 auth token

在瀏覽器的「網關令牌」欄位填入你的 token：

```bash
grep 'token' ~/.openclaw/openclaw.json
# 找到 "token": "xxxxxxxx" 那行，複製引號內的字串
```

填入 token 後按「連接」，若出現 `pairing required` 提示：

### Step 1：查看待配對裝置

```bash
openclaw devices list
```

輸出範例：

```
Pending (2)
┌─────────────────────────────────────┬──────────┬──────────┐
│ Request                             │ Role     │ Age      │
│ 12345678-d1c1-4b0d-a43e-1234567890ab│ operator │ just now │
│ 12345678-774a-4d38-b067-1234567890ab│ operator │ just now │
└─────────────────────────────────────┴──────────┴──────────┘
```

### Step 2：逐一核准

```bash
openclaw devices approve 12345678-d1c1-4b0d-a43e-1234567890ab
openclaw devices approve 12345678-774a-4d38-b067-1234567890ab
```

### Step 3：確認完成

回到瀏覽器重新整理，右上角顯示綠燈「連線」即完成。

> ✅ 之後從同一台電腦連線就不需要再配對了。  
> 若更換裝置（電腦/瀏覽器），需要重新執行一次 pairing 流程。

---

## 十二、磁碟擴容

先在 GCP Console 調整磁碟大小，再 SSH 進入 VM 執行：

```bash
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
df -h
```

---

## 十三、日常健康確認指令

```bash
which openclaw                    # 確認指令可找到
df -h                             # 確認磁碟空間
sudo systemctl status openclaw    # 確認服務狀態
sudo journalctl -u openclaw -f    # 即時 log
```

openclaw 突然找不到時的**快速復活指令**：

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
export PATH="$HOME/.nvm/versions/node/v25.8.0/bin:$PATH"
```

---

## 十四、進階設定：tools.profile

OpenClaw 的 AI 能力範圍由 `tools.profile` 控制，**只能填以下四個值**：

| Profile | 說明 | 適合情境 |
|---------|------|---------|
| `minimal` | 僅基本對話 | 純聊天，不需要任何操作 |
| `messaging` | 訊息收發（預設） | 只用 Telegram/Discord 溝通 |
| `coding` | 含 read/write/exec | 讓 AI 可讀寫檔案、執行指令（建議） |
| `full` | 全開含瀏覽器控制 | 最大權限，風險最高 |

**修改 profile（以改成 `coding` 為例）：**

```bash
# 1. 停止服務
sudo systemctl stop openclaw

# 2. 修改設定
python3 -c "
import json
with open('/home/jason/.openclaw/openclaw.json', 'r') as f:
    c = json.load(f)
c['tools']['profile'] = 'coding'
with open('/home/jason/.openclaw/openclaw.json', 'w') as f:
    json.dump(c, f, indent=2, ensure_ascii=False)
print('完成！')
"

# 3. 重新啟動
sudo systemctl start openclaw
sudo systemctl status openclaw
```

> ⚠️ **填錯 profile 值會導致 Gateway 無法啟動並一直重啟！**  
> 只能填 `minimal`、`messaging`、`coding`、`full`，其他值（如 `standard`）無效。  
> 若已填錯，先 `sudo systemctl stop openclaw`，修正後再啟動。

> 💡 開了 `coding` 後，建議在 Telegram 跟 AI 說：「執行任何指令前請先告訴我你要做什麼，等我確認後再執行。」讓它養成先問再動的習慣。

---

## 十五、Token 費用優化設定

若發現每次對話消耗 token 異常偏高（例如一次問答就燒掉 100K+ tokens），可從以下三個方向排查與調整。

### 診斷：確認目前設定

```bash
python3 -c "
import json
with open('/home/$(whoami)/.openclaw/openclaw.json') as f:
    c = json.load(f)
print('profile:', c.get('tools', {}).get('profile'))
print('compaction:', c['agents']['defaults']['compaction'])
"
```

### 原因一：compaction 模式設為 safeguard（最燒錢）

`safeguard` 模式會保留完整對話歷史，直到快撞上 context limit 才壓縮，越聊 token 越多。

**合法值只有兩個：**

| 模式 | 說明 | 建議 |
|------|------|------|
| `default` | 自動定期壓縮舊對話 | ✅ **建議改用** |
| `safeguard` | 保留完整歷史直到快爆 | ❌ 最貴，不建議日常使用 |

> ⚠️ **注意**：這裡填入其他值（如 `auto`）會導致 Config invalid，Gateway 無法啟動。只能填 `default` 或 `safeguard`。

**修改方式：**

```bash
sudo systemctl stop openclaw

python3 -c "
import json
with open('/home/$(whoami)/.openclaw/openclaw.json', 'r') as f:
    c = json.load(f)
c['agents']['defaults']['compaction']['mode'] = 'default'
with open('/home/$(whoami)/.openclaw/openclaw.json', 'w') as f:
    json.dump(c, f, indent=2, ensure_ascii=False)
print('完成！')
"

sudo systemctl start openclaw
sudo systemctl status openclaw
```

### 原因二：tools.profile 開太大

`coding` 或 `full` 模式每次請求都會帶入大量 tool definitions，光 system prompt 就可能 20K～50K tokens。

日常聊天或簡單查詢建議改用 `messaging`，需要寫程式或分析時再切換回 `coding`（修改方式參考十四節）。

### 原因三：session-memory + bootstrap-extra-files hooks

這兩個 hook 預設開啟，每次對話都會重新注入記憶內容和額外檔案。若 token 消耗仍然偏高，可考慮關閉：

```bash
sudo systemctl stop openclaw

python3 -c "
import json
with open('/home/$(whoami)/.openclaw/openclaw.json', 'r') as f:
    c = json.load(f)
c['hooks']['internal']['entries']['bootstrap-extra-files']['enabled'] = False
with open('/home/$(whoami)/.openclaw/openclaw.json', 'w') as f:
    json.dump(c, f, indent=2, ensure_ascii=False)
print('完成！')
"

sudo systemctl start openclaw
```

> 💡 **實際效果**：`compaction` 從 `safeguard` 改為 `default` 是最有效的調整，一般可將單次對話的 input tokens 從 100K+ 降低到 10K～30K 範圍。

---

## 十六、Gateway 一直重啟的排查流程

遇到 Gateway 不停重啟，先看 log 找原因：

```bash
sudo journalctl -u openclaw -n 30 --no-pager
```

### 常見原因一：設定檔無效

log 出現：
```
Config invalid
tools.profile: Invalid input (allowed: "minimal", "coding", "messaging", "full")
```

解法：參考十四、修正 `tools.profile` 值。

### 常見原因二：兩個 Gateway 搶同一個 port

log 出現：
```
gateway already running (pid XXXXX); lock timeout after 5000ms
Port 18789 is already in use.
```

發生原因：除了 systemd 之外，還有手動啟動的 Gateway process 在跑（例如曾用附錄的 tmux 方式啟動過）。

**解法：**

```bash
# 1. 先停 systemd，不讓它繼續重啟
sudo systemctl stop openclaw

# 2. 殺掉所有殘留 process
pkill -f openclaw-gateway
pkill -f "openclaw$"

# 3. 確認清乾淨（只剩 grep 那行）
sleep 3
ps aux | grep openclaw

# 4. 重新啟動 systemd
sudo systemctl start openclaw
sleep 3
sudo systemctl status openclaw
```

> ⚠️ 設定好 systemd 後，就不要再手動跑 `openclaw gateway`，否則會衝突。

### 常見原因三：openclaw 自動建立 user 層級服務（最燒錢！）

這是最容易被忽略、也最花錢的問題。OpenClaw 安裝時會**自動**在 user 層級建立 `openclaw-gateway.service`，加上 STEP 8 設定的系統層級服務，變成兩個 Gateway 同時運行。

症狀：Telegram Bot 每則訊息回覆兩次，API token 一天燒完。

**確認方式：**

```bash
# 查看 user 層級有沒有 openclaw 服務
systemctl --user list-units | grep openclaw
```

若有出現 `openclaw-gateway.service`，執行以下清理流程：

```bash
# 1. 停掉系統層級 systemd，避免干擾
sudo systemctl stop openclaw

# 2. 停用 user 層級服務
systemctl --user stop openclaw-gateway
systemctl --user disable openclaw-gateway

# 3. 殺掉所有殘留 process
sudo pkill -9 -f openclaw-gateway
sudo pkill -9 -f "openclaw$"

# 4. 確認清乾淨（只剩 grep 那行）
sleep 5
ps aux | grep openclaw

# 5. 交還給系統層級 systemd 統一管理
sudo systemctl start openclaw
sleep 3
ps aux | grep openclaw
# 正確結果：只有一個 openclaw-gateway
```

---

## 十七、常見問題排查

| 症狀 | 原因 | 解法 |
|------|------|------|
| `command not found: openclaw` | NVM 路徑未載入 | 執行快速復活指令；確認 `whoami` 帳號 |
| NVM 重登後消失 | 帳號不一致 | `gcloud compute ssh 帳號@VM名稱` 明確指定 |
| 磁碟已滿 | 更新時空間不足 | Console 擴容後執行 `growpart` + `resize2fs` |
| `origin not allowed` | Tailscale URL 不在白名單 | 執行十、Step 3 加入 allowedOrigins |
| `pairing required` | 首次連線需配對 | `openclaw devices list` → approve 所有 requestId |
| `AI service overloaded` | API 供應商過載 | 等 30 秒重試；或設定備用模型 |
| Gateway 不啟動（systemd） | 路徑或版本號錯誤 | 確認 `node --version`，修正 service 檔版本號 |
| VM 重開機後 Gateway 消失 | 未設定 systemd | 執行 STEP 8 設定 systemd 自動啟動 |
| Gateway 一直重啟 | 設定檔錯誤或兩個 Gateway 衝突 | 執行十五、排查流程 |
| `Config invalid` + 重啟迴圈 | `tools.profile` 填了無效值 | 停服務 → 修正為合法值 → 重啟 |
| `Config invalid` + 重啟迴圈 | `compaction.mode` 填了無效值（如 `auto`） | 停服務 → 修正為 `default` 或 `safeguard` → 重啟（參考十五節）|
| **每次問答燒掉 100K+ tokens** | **`compaction` 為 `safeguard` 或 `coding` profile** | **改 compaction 為 `default`，日常改用 `messaging` profile（參考十五節）** |
| **Telegram Bot 每則訊息回覆兩次** | **同時有兩個 Gateway 運行** | **停掉手動啟動的 process，只留 systemd（參考十五、原因二）** |
| **回覆兩次 + pkill 後馬上復活** | **user 層級 openclaw-gateway.service 在自動重啟** | **`systemctl --user stop openclaw-gateway` → `disable`（參考十五、原因三）** |

---

## 十八、安裝完成後的下一步

1. 打開 Telegram，傳訊息給 Bot 完成配對
2. 說「你好，請介紹你自己」測試回應是否正常
3. 調整 `~/.openclaw/openclaw.json` 設定 Tools 與 Skills
4. 設定備用模型，避免 AI 供應商偶發過載

> 💡 **進階設定提示**  
> - 定期執行 `df -h` 監控磁碟，建議空間保持在 50% 以下  
> - `sudo journalctl -u openclaw --since '1 hour ago'` 可查看最近一小時 log  
> - 備份 `~/.openclaw/openclaw.json`，避免設定遺失

---

---

## 附錄：tmux 除錯工具（進階用，一般使用者略過）

> ⚠️ **重要：tmux 僅供除錯觀察用，不可用來啟動 Gateway 作為正式運行方式。**  
> 已設定 systemd（STEP 8）的情況下，若再用 tmux 啟動 Gateway，會導致兩個 Gateway 同時運行，造成 **Telegram Bot 每則訊息回覆兩次**。

### tmux 是什麼？

tmux 是一個終端機多工工具，讓你在 SSH 斷線後程序仍繼續在背景跑。本文件的正式方案已改用 systemd（STEP 8），穩定性更好、開機自動啟動、不需要手動操作。

tmux 的用途僅限於：**臨時觀察 Gateway 的即時輸出**，或在 systemd 設定出問題時當作備案排查工具。

### 安裝 tmux

```bash
sudo apt install -y tmux
```

### 基本操作

```bash
# 建立 session
tmux new -s debug

# 在 tmux 內觀察 Gateway log（不要用來啟動 Gateway！）
sudo journalctl -u openclaw -f

# 分離（SSH 斷線後仍持續）
# 按 Ctrl+B，放開後再按 D

# 重新連回
tmux a -t debug

# 查看所有 session
tmux ls

# 關閉 session
tmux kill-session -t debug
```

### 如果你之前用 tmux 啟動過 Gateway

先確認有沒有殘留 process，再重新讓 systemd 接管：

```bash
# 停掉 systemd 避免衝突
sudo systemctl stop openclaw

# 殺掉所有手動啟動的 Gateway
pkill -f openclaw-gateway
pkill -f "openclaw$"

# 確認清乾淨
sleep 3
ps aux | grep openclaw

# 交還給 systemd 管理
sudo systemctl start openclaw
sudo systemctl status openclaw
```

