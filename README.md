# 英靈召喚（FGO 風格）— 連線版

單機抽卡／PVE／練習 PVP，加上帳號、雲端存檔與即時連線對戰（Node.js 伺服器）。

## 資料夾結構
```
.
├─ README.md                 ← 本檔
├─ render.yaml               ← Render 一鍵部署設定（Blueprint）
├─ .gitignore
├─ game/                     ← 遊戲原始碼與素材（只有開發者需要）
│  ├─ FGO/                   ← 資料與素材：FGO.xlsx（角色、卡池、技能、關卡）、劇情/、各卡池圖片
│  └─ 遊戲原始碼/
│     ├─ tpl.html            ← 遊戲本體模板（畫面、戰鬥、連線模組都在這裡）
│     ├─ gen.py              ← 讀 FGO.xlsx＋素材，產生 gacha.html
│     ├─ requirements.txt    ← gen.py 需要的 Python 套件
│     └─ 說明.txt            ← 每一批更新的詳細備註
└─ server/                   ← 連線伺服器（部署的就是這個資料夾）
   ├─ server.js              ← 帳號、雲端存檔、配對、對戰轉送、積分／榮譽
   ├─ package.json
   ├─ Dockerfile
   ├─ README.md              ← 伺服器環境變數、Render／Nginx 部署、安全邊界
   └─ public/gacha.html      ← 提供給玩家的遊戲檔（gen.py 自動產生；要 commit）
```

## 日常流程
1. 改資料：編輯 `game/FGO/FGO.xlsx`（角色、技能、關卡…）或換圖；改畫面／玩法：編輯 `game/遊戲原始碼/tpl.html`。
2. 重新產生遊戲：
   ```bash
   cd game/遊戲原始碼
   pip install -r requirements.txt   # 第一次需要（openpyxl、pillow、numpy、scipy、opencv）
   python gen.py
   ```
   會產生兩份：`game/gacha.html`（本機直接雙擊試玩單機功能，不進 git）和 `server/public/gacha.html`（伺服器版，要 commit）。
3. 本機測試連線版：
   ```bash
   cd server && npm install && npm start     # 開 http://localhost:3000
   ```
   用兩個不同的瀏覽器（或一般＋無痕視窗）各註冊一個帳號，兩邊都按「快速配對」。
4. 上線：commit & push，Render 會自動重新部署（見下）。

> 每次重新 `gen.py` 都會產生新的版本碼；版本不同的玩家不能互相配對。更新上線的瞬間，對戰中的玩家會斷線，建議挑沒人的時段。

## 上傳到 GitHub
```bash
git init
git add .
git commit -m "FGO 連線版"
git branch -M main
git remote add origin https://github.com/<你的帳號>/<repo 名稱>.git
git push -u origin main
```
- 整個 repo 約 100 MB（主要是圖片）。GitHub 單檔上限 100 MB、建議整個 repo 在 1 GB 以內，目前沒有問題；之後圖片變多可考慮 Git LFS。
- 圖片、角色立繪等素材的授權請自行確認後再把 repo 設為公開；**建議先用 Private repo**。
- **上線前務必處理 `tpl.html` 內的 `DEV_GRANT` 測試發放區塊**（會替每個存檔發放全角色與大量貨幣）。目前依你的指示保留，平衡性調整完成後再刪。

## 部署到 Render
1. Render → New → **Blueprint** → 選這個 repo（會讀根目錄的 `render.yaml`）。
2. 它會建立 Web Service（Root Directory＝`server`），自動產生 `SECRET`，並掛上 `/var/data` 的 Persistent Disk 存帳號與雲端存檔。
3. 部署完成後開 `https://<名稱>.onrender.com` 即可（Render 預設 HTTPS／WSS）。
4. **Persistent Disk 需要付費方案**；免費方案資料會在重新部署／休眠重啟時清空，也會因閒置休眠而斷線，只適合試玩。
5. 只能開單一實例（配對與對戰狀態存在記憶體）。

連線對戰規則：生命倍率 ×10（`PVP_MUL`）；榮譽每場最多 24、每人每日上限 500（`HONOR_CAP`，伺服器統計）；詳見 `server/README.md`。

## 安全邊界（簡述）
連線對戰採「伺服器轉送＋雙方各自模擬」，伺服器會核對登入、角色擁有／等級、出招合法性與雙方結果是否一致；榮譽有伺服器端每日上限。但不是完全由伺服器判定戰鬥，雲端存檔內容仍由客戶端上傳。若要做有實質價值的排名獎勵，建議之後把戰鬥搬到伺服器端。詳見 `server/README.md`。
