# RMI 集成與遺留系統架構  

>[!note] 
>call wfci.jar 到另一個專案查詢資料並傳回的方式。
>就是一種 "RMI (Remote Method Invocation)" 技術。

## 技術定義：RMI (<font color="#ff0000">Remote Method Invocation</font>)  
  
你在 `ActivityServiceImpl.java` 中的這行程式碼：  
  
```java  
Vector parentIdList = wfci.getParentIDListOfMember(user.getID(), false, true);
```  
  
使用的是 **Java 的 RMI 技術**。  
## Stub: 
用戶端 (Client) 代理人
在分散式系統中，Stub 還有另一個意義：它是**用戶端（Client）的代理人**。 當你的電腦想呼叫遠端伺服器的功能時，你會呼叫一個本地的 Stub 物件，這個 Stub 負責把請求封裝成網路封包傳出去，讓你感覺好像在呼叫本地函式一樣。
  
---  
  
## 關鍵證據  
  
### 1. Package 名稱  
  
```java  
import si.wfinterface.WFCI;  // si.wfinterface 暗示是遠端介面  
```  
  
### 2. 取得方式  
  
```java  
WFCI wfci = WebSystem.getWFCI();  // 透過 factory 取得遠端 stub  
```  
  
### 3. 呼叫方式  
  
```java  
Vector parentIdList = wfci.getParentIDListOfMember(user.getID(), false, true);// 看起來像本地呼叫，但實際上是遠端叫用  
```  
  
### 4. 傳回型別  
  
```java  
Vector  // 老式 Java 集合，符合 RMI 時代的特徵（2000-2010年代）  
```  
  
---  
  
## RMI 的工作原理  
  
```  
你的程式 (WFCIService)
    |
    v
wfci.jar (Stub 代理物件)
    |
    | RMI 通訊協議 (二進位傳輸)
    v
遠端 PASE 系統 (WFCI 實現)
    |
    v 查詢資料庫 / 執行業務邏輯
    |
    v 結果序列化
    |
    | RMI 回傳結果
    v
wfci.jar (Skeleton 解序列化)
    |
    v
你的程式 (收到 Vector)
```  
  
---  
  
## 具體運作流程（你這段程式碼）  
  
| 步驟 | 發生樓層 | 動作 |  
|-----|--------|-----|  
| 1 | 本地（你的程式） | 呼叫 `wfci.getParentIDListOfMember(...)` |  
| 2 | wfci.jar (Stub) | 攔截方法，序列化參數 `user.getID(), false, true` |  
| 3 | 網路傳輸 | 透過 RMI Registry 與遠端 PASE 系統通訊 |  
| 4 | 遠端 PASE | 執行實際方法，查詢該使用者的父級部門 ID 清單 |  
| 5 | 遠端 PASE | 序列化結果 `Vector` 回傳 |  
| 6 | wfci.jar (Skeleton) | 反序列化結果 |  
| 7 | 本地（你的程式） | 收到 `Vector parentIdList` |  
  
---  
  
## 為什麼用 RMI？  
  
在你的架構文件中說：  
  
> 舊功能繼續透過 wfci.jar 使用 RMI 呼叫 PASE 系統之 API  
  
### 優點  
  
- 對 Java 開發者透明（看起來就像呼叫本地物件）  
- 二進位傳輸，比 HTTP 快  
- 強型別，編譯期檢查  
  
### 缺點  
  
- 只能用 Java 之間通訊  
- 防火牆穿越困難  
- 序列化/反序列化開銷（傳 `Vector` 這種老舊型別效率差）  
- 已是過時技術（現代用 REST API）  
  
---  
  
## 同類型的技術對比  
  
| 技術                                                             | 傳輸方式                                                                      | 原理                             | 現況         |     |
| -------------------------------------------------------------- | ------------------------------------------------------------------------- | ------------------------------ | ---------- | --- |
| **RMI**                                                        | <font color="#4bacc6">二進位                                                 | 方法序列化 + Socket</font>          | 過時 (你現在用的) |     |
| <mark style="background:#b1ffff">**Web Service (SOAP)**</mark> | <font color="#f79646">XML over HTTP                                       | 複雜的 XML 定義    </font>          | 漸淘汰        |     |
| <mark style="background:#b1ffff">**REST API**</mark>           | <font color="#00b0f0"><mark style="background:#fdbfff">JSON over HTTP     | 簡單的 HTTP 呼叫     </mark></font> | 現代標準       |     |
| <mark style="background:#b1ffff">**gRPC**                      | <font color="#0070c0">Protocol Buffer                             </font> | </mark> 高效能二進位                 | 新世代        |     |
| **Message Queue (RabbitMQ)**                                   | <font color="#6425d0">非同步                             </font>             | 事件驅動                           | 微服務推薦      |     |
  
---  
  
## 你現在的情況  
  
根據你的漸進式遷移策略：  
  
```  
舊系統 (WFCIService)
    ├─ 繼續用 RMI 呼叫 PASE
    └─ 保持穩定，不改動

新系統 (UP4)
    └─ 改用 REST API 或直接存取資料庫
```  
  
### 所以 RMI 還要用多久？  
  只要「舊系統」還存在，就還要用。當舊功能全部遷移到 UP4，就可以刪掉。  
  
---  
  
## RMI 在 ActivityServiceImpl 中的使用示例  
  
  ### 例子 1：查詢父級部門  
  
```java  
Vector parentIdList = wfci.getParentIDListOfMember(user.getID(), false, true);
filterParamsMap.put("parentIdList", parentIdList);  
List acts = (Vector) wfci.getActivityListByFilterMap(user.getID(), filterParamsMap);  
```  
  
**說明**：  
- 先透過 RMI 取得使用者的父級部門清單  
- 再透過 RMI 查詢該部門下的活動  
  
### 例子 2：取得活動詳細資訊  
  
```java  
Activity act = wfci.getActivity(actId);  
if (act == null) {  
    logger.debug("查無此 '{}' 代號活動!", actId);  
    return getActInfo(act, user);}  
```  
  
**說明**：  
- 透過 RMI 遠端呼叫取得 Activity 物件  
- 如果活動存在，進一步處理  
  
### 例子 3：報名功能  
  
```java  
boolean bRes = wfci.registerActivity(act, user.getID());  
```  
  
**說明**：  
- 透過 RMI 在遠端 PASE 執行報名邏輯  
- 傳回布林值表示成功或失敗  
  
### 例子 4：取得報名清單  
  
```java  
Vector mbrIdList = act.getMemberIdList();for (int i = 0; i < mbrIdList.size(); i++) {  
    String mbrId = (String) mbrIdList.get(i);  
    MemberRecord mbr = wfci.getMember(mbrId);  // RMI 呼叫  
    String depName = wfci.getDepNameByRoleID(roleId);  // RMI 呼叫  
}  
```  
  
**說明**：  
- 逐筆透過 RMI 取得報名者資訊  
- 逐筆透過 RMI 取得部門名稱  
- 這會產生 N+1 查詢問題（效能隱憂）  
  
---  
  
## 性能隱憂  
  
在你的程式碼中，有多個地方會產生 RMI 網路開銷：  
  
### 1. 迴圈內的 RMI 呼叫（N+1 問題）  
  
```java  
for (int i = 0; i < mbrIdList.size(); i++) {  
    String mbrId = (String) mbrIdList.get(i);  
    MemberRecord mbr = wfci.getMember(mbrId);  // 每次迴圈都是一次遠端呼叫  
    String depName = wfci.getDepNameByRoleID(roleId);  // 又是一次遠端呼叫  
}  
```  
  
**改善建議**：  
- 使用批量查詢 API（如果 PASE 提供）  
- 或在 UP4 新系統中改用本地資料庫查詢  
  
### 2. 序列化開銷  
  
每次 RMI 呼叫都要序列化／反序列化物件，使用 `Vector` 這種老舊集合型別會增加額外開銷。  
  
---  
  
## 遷移建議  
  
### 短期（保持 RMI，優化現況）  
  
- 批量查詢，減少 N+1 問題  
- 使用快取層（Redis）  
- 加入連線池配置  
  
### 中期（部分遷移）  
  
- UP4 新功能改用 REST API 或直接存取資料庫  
- 舊功能繼續用 RMI，但標記為「待遷移」  
  
### 長期（完全遷移）  
  
```  
PASE 系統  
  |  
  v 
REST API Gateway  
  |
UP4 應用 (直接 HTTP 呼叫，無 RMI 開銷)  
```  
  
---  
  
## 總結  
  
| 項目   | 現況              | 未來               |     |
| ---- | --------------- | ---------------- | --- |
| 通訊技術 | RMI             | REST API 或直接資料庫  |     |
| 網路開銷 | 高（每個方法呼叫都是遠端通訊） | 低（批量查詢或本地存取）     |     |
| 維護成本 | 中（依賴 wfci.jar）  | 低（標準技術棧）         |     |
| 擴展性  | 低（只能 Java）      | 高（任何語言都可呼叫 REST） |     |
| 快取策略 | 困難（RMI 物件直接返回）  | 容易（JSON 易緩存）     |     |
  
---  
  
## 參考檔案  
  
- `WFCIService/src/main/java/com/flowring/modules/api/service/impl/ActivityServiceImpl.java`  
- `WFCIService/src/main/java/com/flowring/WFCI/WebSystem.java`（取得 WFCI instance）  
- `common/src/main/java/com/flowring/common/security/WebSecurityConfig.java`（對應 /api/** 的舊系統規則）