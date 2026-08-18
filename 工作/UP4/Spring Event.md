這個是 0722 孝欽分享的題目，這裡問了 ＡＩ，原因是　ＡＩ 講得比較聽得懂。
簡而言之，**Spring Event 是 Spring 框架提供的一套「內部事件廣播與監聽」機制**。

它採用 **發布/訂閱（Publish-Subscribe）模式**：主要邏輯只負責「發布事件（Publish Event）」，而其他附帶業務（如寄 Email、發 LINE、寫 Log）則由各自的「監聽器（Listener）」來接收並執行。

### 一、Spring Event 可以解決什麼問題？

在傳統開發中（如建立訂單），我們常把所有附帶功能都寫在主邏輯內（`OrderService` 包辦了存檔、寄信、傳簡訊、更新快取、同步 ERP 等）。這會衍生以下痛點，而 Spring Event 能一一解決：

#### 1. 解決「強耦合與違反設計原則」問題

- **傳統痛點**：主服務負擔過重，違反**單一職責原則（SRP）**；每新增一個通知或同步功能，就要修改主要邏輯，違反**開閉原則（OCP）**。
    
- **Spring Event 解決方案**：`OrderService` 只需要專心做「建立訂單」與「發布事件」，它完全不需要知道有哪些服務會處理後續動作。
    

#### 2. 解決「維護與擴充困難」問題

- **傳統痛點**：維護時不敢隨意動舊程式碼，深怕影響主流程。
    
- **Spring Event 解決方案**：半年後如果要新增「ERP 同步」或「Kafka 發送」，**完全不需要修改 `OrderService`**，只要新增一個專門的 `@EventListener` 元件即可。
    

#### 3. 解決「同步執行導致請求卡住（效能/回應速度）」問題

- **傳統痛點**：若寄 Email 需要花 5 秒，使用者呼叫 API 就得被卡住 5 秒。
    
- **Spring Event 解決方案**：預設雖然是同步執行，但只要搭配 `@EnableAsync` 與 `@Async`，就能讓監聽器改為**非同步（背景）執行**，主流程發布完事件後可立即返回回應給使用者。
    

#### 4. 解決「資料庫事務（Transaction）未提交就發送通知」的問題

- **傳統痛點**：主流程存檔後寄出 Email，但隨後資料庫發生 Exception 導致 Rollback，結果「信已經寄了，資料卻沒存成功」。
    
- **Spring Event 解決方案**：提供 `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`，確保**資料庫 Commit 成功後**才真正觸發監聽器執行。
    

### 二、簡報重點摘要與核心概念

#### 1. 三大核心角色

- **Event（事件）**：傳遞的資料載體（Spring 4.2 起，任何 Plain Object/Record 都可以當作 Event）。
    
- **Publisher（發布者）**：使用 `ApplicationEventPublisher.publishEvent()` 發送事件。
    
- **Listener（監聽器）**：使用 `@EventListener` 標註方法，自動監聽並處理對應的 Event。
    

#### 2. 執行模式比較

|**模式**|**觸發方式**|**特性與適用場景**|
|---|---|---|
|**同步 (Synchronous)**|預設機制|依序執行，前一個 Listener 處理完才換下一個。|
|**非同步 (Asynchronous)**|加上 `@Async`|主流程發布後立刻返回，Listener 在背景非同步執行。適合寄 Email、發通知、Log 等不影響主結果的操作。|
|**事務連動 (Transactional)**|使用 `@TransactionalEventListener`|可設定在 Transaction `AFTER_COMMIT` 後才觸發，避免 Rollback 導致資料不一致的問題。|

### 三、優缺點總結與使用建議

- **優點**：低耦合、高擴充性、獨立可測試、模化切割清晰。
    
- **缺點**：因為流程被解耦，若 Listener 過多會導致**程式呼叫鏈不易追蹤**、除錯較困難。
    
- **建議**：適合用於「主業務完成後的附帶擴充動作（通知、日誌、同步等）」，**不適合**拿來處理核心的商業流程邏輯。
    

> **核心總結**：Spring Event 不是為了讓程式跑得更快，而是為了讓程式**更容易維護、擴充與解耦**。