
# 智慧魚缸自動監控與控制系統

Smart-FishTank-Lab — Backend

## 1. 專案介紹

本系統使用 Arduino Mega 2560 進行感測資料擷取與設備控制，
ESP32 負責網路通訊，並搭配 Node.js 後端、MongoDB Atlas、
MQTT、WebSocket 及 Android App，實現魚缸即時監測、
遠端控制、自動控制及異常推播功能。

## 2. 系統架構

- Arduino Mega：感測器讀取與設備控制
- ESP32：UART、Wi-Fi、HTTP、MQTT、WebSocket
- Node.js：後端 API、資料處理及通訊
- MongoDB Atlas：感測資料及控制設定儲存
- Render：後端雲端部署
- HiveMQ：MQTT 通訊服務
- Firebase Cloud Messaging：異常推播通知
- Android App：監控、控制及設定介面

感測資料透過 HTTP POST 上傳至後端。
控制指令優先使用 MQTT，失敗時改用 WebSocket，
兩者皆無法連線時由 ESP32 使用 HTTP 備援取得設定。

## 3. 後端程式安裝

### 開發環境

- Node.js
- npm
- Git
- Visual Studio Code

Node.js 版本應與目前正式部署環境相容。

### 下載專案

```bash
git clone https://github.com/Smart-FishTank-Lab/fishtank-api.git
cd fishtank-api
npm install
```

### 建立環境變數

將 .env.example 複製為 .env，
並填入實際使用的環境變數。

Windows PowerShell：

```powershell
Copy-Item .env.example .env
```

啟動後端：

```bash
npm start
```

本機後端預設使用 Port 5000。

## 4. 環境變數說明

| 變數名稱 | 說明 |
|---|---|
| API_KEY | API 與 WebSocket 驗證金鑰 |
| MONGO_URI | MongoDB 連線字串 |
| MQTT_URL | MQTT Broker 連線網址 |
| MQTT_USERNAME | MQTT 使用者名稱 |
| MQTT_PASSWORD | MQTT 密碼 |
| MQTT_DEVICE_ID | 魚缸裝置識別名稱 |
| GOOGLE_APPLICATION_CREDENTIALS | Firebase 憑證路徑 |
| FIREBASE_SERVICE_ACCOUNT_JSON | 舊環境設定，是否需要使用應再確認 |
| TZ | 系統時區 |

裝置識別名稱：

fish_Tank_001

系統時區：

Asia/Taipei

請勿將真實的 .env、密碼或 Firebase 私密金鑰上傳至 GitHub。

## 5. MongoDB Atlas

目前使用的專案：

- Organization：Smart-FishTank-Lab
- Project：FishTank-Database
- Cluster：Cluster0
- Database：fish

管理網址：

https://cloud.mongodb.com/

交接時可將接手人員加入 Organization，
並授予適當的管理權限。

不需重新建立 Cluster，原有歷史資料應予保留。

若重新部署 Render，須確認 MongoDB 的
Database User 與 IP Access List 設定正確。

## 6. Render 後端部署

管理網址：

https://dashboard.render.com/

目前正式後端網址：

https://fishtank-api.onrender.com

### 手動部署設定

- Source：Smart-FishTank-Lab/fishtank-api
- Branch：main
- Root Directory：留空
- Build Command：npm install
- Start Command：npm start
- Health Check Path：/health
- Instance Type：依實際需求選擇

亦可使用 render.yaml 建立 Blueprint。

部署前須完成環境變數與 Firebase Secret Files 設定。

新後端應先使用測試資料庫及獨立 MQTT Broker
或測試裝置識別名稱進行驗證。

不得讓兩套正式後端同時控制同一魚缸。

## 7. Firebase Cloud Messaging

管理網址：

https://console.firebase.google.com/

- 專案名稱：Smart-FishTank-Lab
- Project ID：fishtank-95285

目前 Firebase 使用服務帳戶 JSON 憑證。

重新部署 Render 時，於 Environment 的
Secret Files 新增：

firebase-service-account.json

Contents 填入 Firebase 服務帳戶 JSON 憑證。

並設定環境變數：

GOOGLE_APPLICATION_CREDENTIALS=/etc/secrets/firebase-service-account.json

請勿將 Firebase 私密金鑰上傳至 GitHub。

若保留原 Firebase Project，
Android App 的 google-services.json 原則上不需修改。

## 8. MQTT 通訊

目前使用 HiveMQ Cloud Serverless。

管理網址：

https://console.hivemq.cloud/

MQTT TLS Port：8883

裝置 ID：fish_Tank_001

主要 Topic：

| Topic | 功能 |
|---|---|
| fish/control/fish_Tank_001 | 設備控制 |
| fish/settings/fish_Tank_001 | 自動控制設定 |
| fish/calibration/fish_Tank_001 | 感測器校正 |
| fish/status/fish_Tank_001 | 裝置及校正結果回報 |

### 重要維護事項

HiveMQ Cloud Serverless 預計於
2026 年 12 月 31 日停止服務。

後續維護人員需建立新的 MQTT Broker，
例如 EMQX Cloud，並修改：

1. Render 的 MQTT_URL
2. Render 的 MQTT_USERNAME
3. Render 的 MQTT_PASSWORD
4. ESP32 的 MQTT Host、帳號及密碼
5. TLS 憑證設定（如有需要）

MQTT 失效時，系統可使用 WebSocket 及 HTTP 備援。

但 DO 校正結果回報依賴 MQTT，
且 ESP32 持續重連可能影響資料更新及餵食排程。

接手後應優先完成 MQTT 更換及相關功能驗證。

## 9. WebSocket 通訊

Android App：

wss://fishtank-api.onrender.com/ws?role=app

ESP32：

wss://fishtank-api.onrender.com/ws?role=esp32

連線時須使用正確的 x-api-key。

WebSocket 由 Node.js 後端提供，
不需要另外申請 WebSocket 服務。

## 10. 主要 API

除 /health 外，以下 API 須使用
x-api-key 進行驗證。

| Method | API | 功能 |
|---|---|---|
| GET | /health | 後端服務狀態 |
| POST | /api/sensor | 感測資料上傳 |
| GET | /api/sensor/latest | 最新感測資料 |
| GET | /api/sensor/recent | 近期感測資料 |
| GET | /api/sensor/history | 歷史資料 |
| GET | /api/control | 取得控制狀態 |
| POST | /api/control | 修改控制狀態 |
| GET | /api/settings | 取得自動設定 |
| POST | /api/settings | 更新自動設定 |
| GET | /api/calibration | 取得校正資料 |
| POST | /api/calibration | 更新校正設定 |
| POST | /api/fcm-token | 註冊推播 Token |
| GET | /api/notification-settings | 取得通知設定 |
| POST | /api/notification-settings | 更新通知設定 |
| GET | /api/alarms | 查詢異常警報 |
| GET | /api/alarms/active | 查詢目前異常 |
| GET | /api/alarm-events | 查詢警報事件 |

完整參數及 JSON 格式請參考 routes 資料夾。

## 11. 專案資料夾

- models：MongoDB 資料模型
- routes：API 路由
- utils：通訊、驗證、感測評估及推播
- test：測試程式
- db.js：MongoDB 連線
- server.js：HTTP 與 WebSocket 伺服器
- package.json：套件及啟動設定
- .env.example：環境變數範本
- render.yaml：Render 部署設定

## 12. 部署後功能測試

- [ ] 後端服務正常啟動
- [ ] MongoDB 正常連線
- [ ] ESP32 感測資料正常上傳
- [ ] App 即時資料正常顯示
- [ ] App 歷史資料正常讀取
- [ ] MQTT 通訊正常
- [ ] WebSocket 備援正常
- [ ] HTTP 備援正常
- [ ] App 手動控制正常
- [ ] 自動控制設定正常
- [ ] 感測器校正正常
- [ ] DO 校正結果正常回報
- [ ] 自動餵食正常
- [ ] FCM 異常推播正常
- [ ] ESP32 斷網後本機控制正常
- [ ] 設備重新啟動後狀態正常

涉及加藥、換水及加熱的測試，
應在有人監看且設備處於安全狀態下進行。

## 13. 專題交接注意事項

1. GitHub、MongoDB、Firebase 保留原有專案。
2. 新增接手人員的專案管理權限。
3. Render 視帳號方案重新部署。
4. MQTT Broker 由接手人員進行更換。
5. 若後端網址改變，須修改 App 與 ESP32。
6. Arduino 控制邏輯原則上不需修改。
7. 備份歷史資料及感測器校正參數。
8. 確認新系統正常後，才停用舊服務及憑證。
9. 不得將私人帳號密碼直接交給接手人員。

## 14. 專案維護

GitHub Organization：

https://github.com/Smart-FishTank-Lab

本專案供智慧魚缸專題後續研究、維護及功能擴充使用。
