# FGO 連線伺服器

提供：帳號（註冊／登入）、雲端存檔、連線對戰（快速配對／房間代碼）、積分（Elo）與排行榜。
只依賴 Node.js 18+ 與 `ws`。遊戲檔 `gacha.html` 由同一個伺服器提供，所以玩家直接開伺服器網址就能玩。

## 快速啟動
```bash
cd server
npm install
# public/gacha.html 已附上最新版；用 game/遊戲原始碼/gen.py 重新產生遊戲時會自動同步到這裡
npm start            # 預設 http://localhost:3000
```
開兩個不同的瀏覽器（或一個一般視窗＋一個無痕視窗）開 `http://localhost:3000` →「對戰」→「連線對戰」→ 各註冊一個帳號 → 其中一人按「快速配對」、另一人也按，就會配對成功。
（同一個瀏覽器的兩個分頁會共用登入記錄，測試請用不同瀏覽器或無痕視窗。）

## 環境變數
| 變數 | 預設 | 說明 |
|---|---|---|
| `PORT` | 3000 | 監聽埠 |
| `DATA_DIR` | `./data` | 帳號、存檔、簽章金鑰存放處（**請定期備份**） |
| `SECRET` | 自動產生並存到 `data/secret.key` | 登入憑證簽章金鑰；多台伺服器要設成同一個 |
| `GAME_FILE` | `./public/gacha.html` | 要提供給玩家的遊戲檔 |
| `PVP_MUL` | 10 | 連線對戰雙方生命倍率（由伺服器統一下發，兩端一致） |
| `HONOR_CAP` | 500 | 連線對戰每人每日榮譽上限（伺服器統計） |
| `TZ_OFFSET_HOURS` | 8 | 「每日」換日用的時區（8＝台灣，凌晨 0 點重置） |
| `DRAFT_SEC` / `TURN_SEC` | 15 / 30 | 選角每輪秒數／每次出招限時 |
| `CORS_ORIGIN` | `*` | 遊戲檔與伺服器不同網域時才需要設定 |

## 正式上線
1. **一定要用 HTTPS／WSS**（密碼與登入憑證會經過網路）。常見做法是 Nginx 反向代理：
```nginx
server {
  listen 443 ssl http2;
  server_name game.example.com;
  # ssl_certificate ... （Let's Encrypt）
  client_max_body_size 4m;
  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;      # WebSocket 必要
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_read_timeout 3600s;
  }
}
```
2. 讓程式常駐：`npm i -g pm2 && pm2 start server.js --name fgo`，或用附的 `Dockerfile`：
```bash
docker build -t fgo-server . && docker run -d -p 3000:3000 -v fgo-data:/data --restart=always fgo-server
```
3. **刪除測試發放**：`game/遊戲原始碼/tpl.html` 內 `DEV_GRANT_BEGIN … DEV_GRANT_END` 會替每個存檔發放全角色與大量貨幣，上線前請整段刪除後重新執行 `gen.py`（否則雲端存檔也會是滿的）。
4. 更新遊戲：重新 `gen.py`（會自動更新 `public/gacha.html`）→ 重啟（玩家重新整理）。版本碼（`BUILD`）不同的玩家不能互相配對，避免兩端規則不一致。


## 部署到 Render
repo 根目錄已附 `render.yaml`（Blueprint）。步驟：
1. 把整個 repo 推到 GitHub（`server/public/gacha.html` 要一起 commit；步驟見根目錄 README.md）。
2. Render → New → Blueprint → 選這個 repo → 套用。它會建立一個 Web Service（Root Directory 為 `server`、Build `npm install`、Start `node server.js`），並自動產生 `SECRET`。
3. 部署完成後直接開 Render 給的 `https://xxx.onrender.com`，Render 預設就是 HTTPS／WSS，不用另外設 Nginx。
4. **務必掛 Persistent Disk**（`render.yaml` 已設好：掛載在 `/var/data`、`DATA_DIR=/var/data`）。帳號、雲端存檔、積分都存在這顆硬碟；沒有硬碟的話，Render 每次重新部署或重啟都會清空，所有帳號與雲端存檔會消失。
   - Persistent Disk 需要付費方案（Starter 以上）。免費方案沒有硬碟，也會在閒置約 15 分鐘後休眠（休眠會斷開所有 WebSocket，喚醒要等數十秒），只適合試玩，不適合正式開放。
5. 更新遊戲：重新跑 `gen.py`（會自動更新 `server/public/gacha.html`）→ commit & push → Render 自動重新部署（對戰中的玩家會斷線，建議挑沒人的時段）。
6. 想改倍率／榮譽上限：Render → Environment 改 `PVP_MUL`、`HONOR_CAP`，重新部署即可。
7. 單一實例即可（配對與對戰狀態存在記憶體，不要開多個實例）。

## 連線對戰榮譽
- 每場：獲勝且擊倒對方角色才有，每擊倒 1 體 +8、每場最多 24；**認輸、斷線、結果不一致不發**（避免兩個帳號互刷）。
- **每人每日上限 500**，由伺服器統計（換日時區見 `TZ_OFFSET_HOURS`），客戶端無法繞過；達上限後仍可對戰與計積分，只是不再獲得榮譽。

## 運作方式
- **帳號**：密碼以 scrypt 加鹽雜湊；登入憑證是 30 天的簽章 token；登入嘗試每 IP 每分鐘最多 12 次。
- **雲端存檔**：玩家照常存在瀏覽器；登入後每次存檔約 3 秒後自動上傳。用版本號偵測衝突（另一台裝置較新時，會詢問要用雲端或本機，覆蓋前自動備份到 `gacha_save_v2_bak`）。
- **對戰同步（lockstep）**：伺服器發同一個亂數種子給雙方；兩個瀏覽器各自跑一模一樣的戰鬥，網路上只傳「選角」與「出招」。每次出招附帶戰況雜湊，兩端不一致就中止、不計分。出招限時，逾時由系統代選。撤退／斷線視為認輸。
- **積分**：兩端回報的結果一致才計分（Elo，K=32）；只有一端回報時 8 秒後採用單方回報；不一致則不計分。

## 目前的安全邊界（請知悉）
這是「伺服器轉送＋客戶端模擬」架構，**不是伺服器權威**：
- 伺服器會核對：登入、角色是否擁有（以雲端存檔為準）、等級上限、出招是否合法（對手端會檢查 NP／冷卻／卡種，不合法會回報並判對方違規）、兩端結果是否一致。
- 但改裝過的客戶端仍可能：竄改自己這端的畫面數值（會造成兩端雜湊不一致而被中止，不會讓對手吃虧）、在雲端存檔裡塞資源（存檔由客戶端上傳）、自行領取榮譽獎勵。
- 若之後要做排名賽獎勵等有實質價值的內容，建議下一步把戰鬥與資源結算搬到伺服器（需要把戰鬥引擎抽出成可在 Node 執行的模組）。
