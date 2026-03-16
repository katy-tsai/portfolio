# 📹 Live View Player — WebRTC & HLS 雙協議影像播放系統

> 雲端影像管理平台的核心播放模組，支援 WebRTC 即時串流與 HLS 點播，具備多通道播放、數位縮放、截圖、錄影等完整功能。

---

## 🧭 系統架構總覽

整體架構分為四層：主要元件層、播放器層、影片控制層、Hook 層。

```
LiveView
  └── LayoutViewBox        # 多格影片容器（支援拖曳排序）
        └── LayoutView     # 單一影片格，依 connectType 選擇播放器
              ├── CeresPlayer   ──→ HLS Stream
              └── WebrtcPlayer  ──→ WebRTC Stream
                    └── TutkVideo          # 核心播放元件
                          ├── useVideo     # 播放控制
                          ├── useZoom      # 縮放 / 拖曳
                          ├── useCapture   # 截圖 / 縮圖上傳
                          └── useCanvasRecord  # Canvas 錄影
```

```mermaid
flowchart TD
  A[LiveView] --> B[LayoutViewBox]
  B --> C[LayoutView]
  C --> D{connectType}
  D -- ceres --> E[CeresPlayer]
  D -- webrtc --> F[WebrtcPlayer]
  E & F --> G[TutkVideo]
  G --> H[useVideo]
  G --> I[useZoom]
  G --> J[useCapture]
  G --> K[useCanvasRecord]
```

---

## 🔀 播放器選擇策略

| connectType | 播放器 | 串流協議 | 適用場景 |
|-------------|--------|----------|----------|
| `ceres` | CeresPlayer | HLS (hls.js) | 穩定錄播、低延遲需求較低 |
| `webrtc` | WebrtcPlayer | WebRTC | 即時監控、雙向語音 |

`connectType` 由設備屬性決定，在裝置初始化時由後端回傳，前端根據此值動態載入對應播放器。

---

## 🎬 核心流程

### 初始化流程

```mermaid
sequenceDiagram
  actor User
  participant LiveView
  participant LayoutViewBox
  participant LayoutView
  participant Player as CeresPlayer / WebrtcPlayer
  participant TutkVideo

  User->>LiveView: 開啟 Live View 頁面
  LiveView->>LiveView: 初始化（取得 viewList / playList）
  LiveView->>LayoutViewBox: 傳遞 viewList, playList
  LayoutViewBox->>LayoutView: 依序產生多個影片格
  LayoutView->>Player: 依 connectType 建立播放器
  Player->>TutkVideo: 掛載串流到 video DOM
  TutkVideo->>TutkVideo: 初始化 useVideo / useZoom / useCapture / useCanvasRecord
```

### 影片播放狀態機

```mermaid
stateDiagram-v2
  [*] --> 初始化
  初始化 --> 載入中
  載入中 --> 串流建立

  state 串流建立 {
    [*] --> 選擇播放器
    選擇播放器 --> CeresPlayer : connectType=ceres
    選擇播放器 --> WebrtcPlayer : connectType=webrtc
  }

  串流建立 --> 播放中 : 串流建立成功
  串流建立 --> 錯誤狀態 : 串流建立失敗

  state 播放中 {
    [*] --> 播放
    播放 --> 暫停 : togglePlay()
    暫停 --> 播放 : togglePlay()
    播放 --> 縮放模式 : onZoomIn()
    縮放模式 --> 播放 : onZoomOut()
    播放 --> 錄影中 : onRecordStart()
    錄影中 --> 播放 : onRecordEnd()
    播放 --> 擷取畫面 : onCapture()
    擷取畫面 --> 播放 : 擷取完成
  }

  錯誤狀態 --> 串流建立 : onReConnection() 重連
  播放中 --> [*] : 離開頁面
```

---

## 🧩 元件說明

### TutkVideo — 核心播放元件

包裝 HTML `<video>` 標籤，整合所有播放控制功能，同時支援 HLS 與 WebRTC 兩種模式。

**主要功能：**

| 功能 | 說明 |
|------|------|
| 播放 / 暫停 | 透過 `useVideo` 管理播放狀態 |
| 數位縮放 | 透過 `useZoom` 實現放大縮小與拖曳 |
| 截圖 | 透過 `useCapture` 擷取當前畫面 |
| 錄影 | 透過 `useCanvasRecord` 錄製成 mp4 / webm |
| PTZ 雲台控制 | 即時串流時支援鏡頭方向與焦距控制 |
| 全螢幕 | 支援進入 / 退出全螢幕 |
| 多裝置支援 | 桌面版 VideoControls / 行動版 MobileVideoControls |
| 閒置偵測 | `useActiveMouse` 自動暫停 / 恢復串流 |

---

## 🪝 Custom Hooks 說明

### `useVideo` — 播放控制

管理 video DOM 的播放、暫停、靜音、倍速、進度等所有互動狀態。

```mermaid
flowchart TD
  A[初始化 useVideo] --> B[建立 playState]
  B --> C{外部操作}
  C -- togglePlay --> D[更新 isPlay]
  D --> E[video.play / pause]
  C -- handleVideoProgress --> F[設定 currentTime]
  C -- handleVideoSpeed --> G[設定 playbackRate]
  C -- toggleMuted --> H[設定 muted / volume]
  C -- onCaptureImage --> I[canvas 擷取畫面]
```

**輸出介面：**

```ts
{
  playState,            // 播放狀態（isPlay, isMuted, 進度, 音量, 倍速）
  togglePlay,           // 切換播放 / 暫停
  setPlay,              // 強制播放
  setStop,              // 強制暫停
  handleOnTimeUpdate,   // 進度更新事件
  handleVideoProgress,  // 拖曳進度條
  handleVideoSpeed,     // 改變播放速度
  toggleMuted,          // 切換靜音
  onCaptureImage,       // 擷取畫面
  currentTime,          // 目前播放秒數
  duration,             // 影片總長度
  isLoading             // 是否 loading 中
}
```

---

### `useZoom` — 數位縮放 / 光學縮放

支援數位縮放（CSS transform）與設備端光學縮放（API 控制鏡頭），並整合拖曳移動功能。

```mermaid
flowchart TD
  A[點擊放大 / 縮小] --> B{isDevice 支援光學縮放?}
  B -- 是 --> C[呼叫 API 控制攝影機鏡頭]
  B -- 否 --> D[改變 scale，設定 video transform]
  D --> E{scale > 1?}
  E -- 是 --> F[啟用拖曳，綁定 mousemove / touchmove]
  F --> G[動態改變 transformOrigin]
  E -- 否 --> H[重設 transformOrigin，釋放事件監聽]
```

**輸出介面：**

```ts
{
  scale,          // 當前縮放比例
  setScale,       // 設定縮放比例
  onZoomInClick,  // 放大
  onZoomOutClick  // 縮小
}
```

---

### `useCapture` — 截圖 / 縮圖自動上傳

擷取 video 當前畫面為 base64 圖片，並在 Live 模式下自動定時上傳縮圖到伺服器。

```mermaid
flowchart TD
  A[video playing 事件] --> B[延遲 5 秒]
  B --> C[onCaptureImage 擷取畫面]
  C --> D[canvas 轉 base64 PNG]
  D --> E{距上次上傳超過 10 分鐘?}
  E -- 是 --> F[上傳縮圖到伺服器]
  F --> G[更新 thumbnail 與 timestamp]
  E -- 否 --> H[僅更新本地 thumbnail]
```

**輸出介面：**

```ts
{
  thumbnail,          // 當前縮圖 base64
  thumbnailUploadTs   // 上次上傳時間戳
}
```

---

### `useCanvasRecord` — Canvas 錄影（HLS 模式）

將 video 畫面繪製到 canvas 後，透過 `RecordRTC` 錄製成 webm / mp4 檔案。

```mermaid
flowchart TD
  A[onRecordStart] --> B[建立 canvas + captureStream]
  B --> C[RecordRTC 開始錄影]
  C --> D[drawMedia 用 requestAnimationFrame 持續繪製]
  D -- 使用者結束錄影 --> E[onRecordEnd]
  E --> F[停止 MediaRecorder，取得 blob]
  F --> G[修正 webm duration]
  G --> H[觸發 onEnd callback，回傳檔案]
  H --> I[釋放 canvas 與動畫資源]
```

**輸出介面：**

```ts
{
  onRecordStart,  // 開始錄影
  onRecordEnd,    // 結束錄影
  isRecording,    // 是否錄影中
  recordStart     // 錄影開始時間（以 viewId 為 key）
}
```

---

### `useActiveMouse` — 閒置偵測

監控使用者操作行為，超過指定時間無操作時自動觸發暫停，有操作時自動恢復。

```mermaid
flowchart TD
  A[每秒累加 count] --> B{count >= outTime × 60?}
  B -- 否 --> A
  B -- 是 --> C[觸發 onPause，標記頁面非活動]
  C --> D[等待使用者操作]
  D -- mousemove / click / touchmove --> E[觸發 onStart，重設 count]
  E --> A
```

---

## 🗺️ 平面圖攝影機定位編輯器

VMS 系統的地圖管理模組，支援在 PDF 平面圖上配置攝影機位置與視角。

### 功能展示

**攝影機定位 — 一般檢視模式**

![Floor Plan View](screenshots/floor-plan-view.png)

**攝影機視角編輯模式**

![Floor Plan Edit](screenshots/floor-plan-edit.png)

### 功能說明

| 功能 | 說明 |
|------|------|
| 平面圖管理 | 支援上傳多張平面圖，右側清單切換 |
| 攝影機標記 | 在平面圖上拖曳放置攝影機圖示 |
| 視角扇形 | 以扇形區域呈現攝影機可視範圍與覆蓋角度 |
| 即時編輯 | 拖曳控制點調整扇形方向、角度與半徑 |
| 儲存 / 取消 | 編輯結果即時儲存，支援取消復原 |

### 互動流程

```mermaid
flowchart TD
  A[選擇平面圖] --> B[載入 PDF 背景圖]
  B --> C[渲染攝影機標記與視角扇形]
  C --> D{使用者操作}
  D -- 點擊編輯按鈕 --> E[進入編輯模式]
  E --> F[顯示扇形控制點]
  F --> G[拖曳控制點調整視角]
  G --> H{確認操作}
  H -- Save --> I[儲存座標與角度資料]
  H -- Cancel --> J[還原編輯前狀態]
  D -- 拖曳攝影機圖示 --> K[更新攝影機座標]
  K --> I
```

### 技術實作重點

- 使用 **Konva.js** 進行 Canvas 繪圖，管理攝影機圖示、扇形區域、控制點等圖層
- 扇形視角以**極座標計算**轉換為 Canvas Path，支援任意角度與半徑
- 拖曳控制點時即時重算扇形頂點座標，達到流暢的視覺回饋
- 攝影機座標系統以**相對比例**儲存（而非像素），確保在不同螢幕尺寸下正確還原位置

---

## 🔔 事件通知系統

VMS 系統整合三種通知管道，當設備偵測到事件時，依使用者訂閱設定即時推送通知。

### 支援通知類型

| 通知方式 | 說明 |
|----------|------|
| Web Push | 透過 Service Worker 推送瀏覽器原生通知 |
| LINE 通知 | 透過 LINE Messaging API 發送訊息 |
| Email 通知 | 發送電子郵件到使用者信箱 |

### 初始化與 Web Push 設定

```mermaid
flowchart TD
  A[使用者開啟 VMS] --> B[初始化 Service Worker]
  B --> C{瀏覽器支援 Web Push?}
  C -- 否 --> D[跳過 Web Push 設定]
  C -- 是 --> E{通知權限?}
  E -- 未授權 --> F[請求通知權限]
  F --> G{使用者回應}
  G -- 拒絕 --> D
  G -- 同意 --> H[建立推送訂閱 PushManager.subscribe]
  E -- 已授權 --> H
  H --> I[產生瀏覽器指紋 UUID]
  I --> J[呼叫 appCheckin API 註冊 Token]
  J --> K[Web Push 設定完成]
```

### 事件觸發與通知推送

```mermaid
flowchart TD
  A[設備產生事件] --> B[後端查詢訂閱設定]
  B --> C{有訂閱?}
  C -- 否 --> D[不發送通知]
  C -- 是 --> E{通知類型}
  E -- Web Push --> F[後端推送訊息到瀏覽器]
  F --> G[Service Worker 接收 push 事件]
  G --> H[解析推送資料與多語系設定]
  H --> I{事件類型}
  I -- 感測器事件 --> J[導向感測器頁面]
  I -- 其他事件 --> K[導向事件列表頁面]
  J & K --> L[showNotification 顯示通知]
  L --> M[使用者點擊，開啟對應頁面]
  E -- LINE --> N[後端呼叫 LINE Bot API]
  N --> O[使用者 LINE 收到通知]
  E -- Email --> P[後端發送電子郵件]
```

### LINE 通知授權流程

```mermaid
flowchart TD
  A[使用者點擊 LINE 設定] --> B[開啟授權對話框]
  B --> C[產生 LINE State 參數]
  C --> D[建立 LINE OAuth 授權 URL]
  D --> E[顯示 QR Code / 授權按鈕]
  E --> F[使用者掃碼或點擊授權]
  F --> G[跳轉 LINE 授權頁面並同意]
  G --> H[後端取得 LINE Token]
  H --> I[向 Push Server 完成 Checkin]
  I --> J[導回加好友頁面，設定完成]
```

### 事件訂閱管理

使用者可針對每台設備選擇訂閱的事件類型與通知方式，支援批量快速設定。

```mermaid
flowchart TD
  A[進入訂閱設定頁] --> B[選擇設備 / 事件類型 / 通知方式]
  B --> C{操作}
  C -- 勾選 --> D[subscribeSettings 新增訂閱]
  C -- 取消 --> E[unsubscribeSettings 移除訂閱]
  D & E --> F[呼叫 setEventSubscribe API]
  F --> G[更新本地訂閱狀態]
```

### 通知核心服務 `NotifyService`

| 方法 | 說明 |
|------|------|
| `subscribeSettings()` | 新增事件訂閱 |
| `unsubscribeSettings()` | 移除事件訂閱 |
| `getEventTypeSettings()` | 查詢訂閱狀態 |
| `quickSubscribeSettings()` | 批量設定處理 |

---

## 🛠 技術棧

| 類別 | 技術 |
|------|------|
| 前端框架 | React.js、Redux |
| 串流協議 | WebRTC、HLS (hls.js) |
| Canvas 繪圖 | Konva.js、Canvas API |
| 錄影 | RecordRTC、Canvas API |
| 動畫 | requestAnimationFrame |
| 縮放 / 拖曳 | CSS Transform、Mouse / Touch Event |
| 通知推播 | Web Push、Service Worker、LINE Messaging API |
| 樣式 | SCSS |

---

## ✨ 主要技術亮點

- **雙協議架構**：同一套元件支援 WebRTC 與 HLS 切換，透過 `connectType` 動態選擇播放器，擴充性強
- **Custom Hook 設計**：將播放、縮放、截圖、錄影各自封裝為獨立 Hook，職責分離，易於測試與維護
- **智慧閒置管理**：`useActiveMouse` 自動暫停長時間無操作的串流，有效節省網路資源
- **縮圖自動化**：`useCapture` 在 Live 模式下自動擷取並上傳縮圖，10 分鐘防重複上傳機制
- **Canvas 錄影**：支援將 video 畫面透過 Canvas 錄製輸出，解決部分瀏覽器無法直接錄製 HLS 的限制
- **平面圖編輯器**：使用 Konva.js 實作攝影機定位與扇形視角編輯，座標以相對比例儲存確保跨裝置一致性
- **多管道通知系統**：整合 Web Push（Service Worker）、LINE Messaging API、Email 三種通知管道，支援細粒度的設備 × 事件類型 × 通知方式訂閱管理