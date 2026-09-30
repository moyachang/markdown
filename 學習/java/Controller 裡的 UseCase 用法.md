# 為什麼 Controller 要用 UseCase 這種設計？

這是典型的「**Clean Architecture / DDD 分層架構**」寫法，跟傳統 MVC（Controller 直接呼叫 Service，Service 塞滿商業邏輯）差異很大。原因如下：

## 1. 單一職責原則（SRP）

傳統寫法常見的問題是 `Service` 類別會越長越大，一個 `MemberService` 裡塞了 20、30 個方法，什麼邏輯都往裡面丟，最後變成「上帝物件」。

UseCase 的做法是**一個類別只做一件事**：

- `GetMemberProfileUseCase` 只負責「取得組織人員資訊」這一件事
- `GetExternalUserUseCase` 只負責「取得企業夥伴人員資訊」這一件事

類別名稱本身就是需求文件，看到類別名稱就知道這個程式在做什麼商業行為，不用進去看實作。

## 2. Controller 變成純粹的「轉接層」

Controller 在這個設計裡**完全不含商業邏輯**，它只做三件事：

1. 接收 HTTP 請求（路徑參數、驗證權限）
2. 呼叫對應的 UseCase
3. 把結果包成 API 回應格式

這樣的好處是：

- Controller 極度穩定、幾乎不太需要改動
- 之後如果要把某個 UseCase 從 REST API 改成透過訊息佇列（MQ）或排程觸發，商業邏輯完全不用改，只是換一個「呼叫者」而已

## 3. 依賴反轉（DIP）與可測試性

因為欄位是 `final`，透過建構子注入（Lombok 幫你產生），代表：

- Controller 依賴的是**抽象的業務行為**（UseCase），而不是一堆散落的 Repository、資料庫存取邏輯
- 測試 `GetMemberProfileUseCase` 時完全不需要啟動 Spring MVC、不需要模擬 HTTP request，直接 new 一個物件丟假資料進去測就好，測試速度快很多
- 測試 Controller 時，也只需要 mock 這一個 UseCase 的 `execute()` 方法，不用管裡面到底做了什麼跨表查詢

## 4. 避免「胖 Service」導致的隱性耦合

傳統 Service 寫法常見的問題是，一個 Service 方法會去呼叫同一個 Service 裡的其他方法，久而久之方法之間互相呼叫、互相依賴，改一個地方要擔心會不會影響到十個地方。

每個 UseCase 是獨立的類別，彼此之間**沒有繼承或呼叫關係**，要修改「取得組織人員資訊」的邏輯，絕對不會不小心動到「取得企業夥伴人員資訊」的邏輯。

## 5. 符合「應用層（Application Layer）」的概念

在 DDD 分層裡：

|層級|職責|
|---|---|
|Controller（介面層）|處理協議轉換（HTTP → 物件），不含商業邏輯|
|**UseCase（應用層）**|協調領域物件、跨表查詢組裝、呼叫 Domain/Repository，代表一個完整的「使用案例」|
|Domain（領域層）|真正的核心商業規則|
|Repository（基礎設施層）|資料存取|

UseCase 剛好對應到「應用層」，它的職責是**協調**（orchestration），例如你註解寫的「跨表查詢、組裝資料」——這種串接多個資料來源、組裝成一個回應物件的邏輯，本來就不屬於 Controller，也不完全是純粹的 Domain 邏輯，放在 UseCase 剛剛好。

---

## 跟傳統寫法的直接對比

**傳統 MVC：**

```java
@GetMapping("/{memId}")
public ApiResponse<MemberProfileResponse> getAFUser(@PathVariable String memId) {
    // Controller 裡面可能還會做參數檢查、格式轉換
    MemberProfileResponse data = memberService.getMemberProfile(memId); 
    // memberService 裡面可能還有 getMemberList, updateMember, deleteMember... 20 個方法全擠在一起
    return ApiResponse.success(data);
}
```

**UseCase 寫法（本例）：**

```java
@GetMapping("/{memId}")
public ApiResponse<MemberProfileResponse> getAFUser(@PathVariable String memId) {
    MemberProfileResponse data = getMemberProfileUseCase.execute(memId);
    return ApiResponse.success(data);
}
```

表面上看起來只是「把 Service 的方法拆成獨立類別」，但實質差異在於：**強迫每個商業行為都是一個獨立、可單獨測試、可單獨替換的單元**，這在專案規模變大、團隊人數變多時，能大幅降低改動風險與理解成本。

---

## 這種設計的取捨（誠實地說）

它不是沒有代價的：

- **類別數量會暴增**：一個中型系統可能會有上百個 UseCase 類別，檔案總覽會變得很長
- **簡單 CRUD 也要走一輪流程**：對於單純的「查一張表回傳」這種需求，用 UseCase 包一層感覺有點大材小用，某些團隊會為此爭論是否過度設計（over-engineering）

所以這種設計比較適合**商業邏輯複雜、需要長期維護、多人協作**的系統；如果專案很小、生命週期短，傳統 Service 寫法反而更精簡高效。

---
# 以下是 gemini 的回答
在傳統的 MVC（Model-View-Controller）架構中，Controller 通常會直接注入一個龐大的 `OrgMemberService`，而 Service 裡可能塞滿了數十個與「組織人員」相關的方法（例如 `getMemberProfile`、`getPartnerUser`、`updateMember`、`deleteMember` 等）。

  
 
而在 DDD（領域驅動設計）或 Clean Architecture / 六角形架構中，Controller 改為注入粒度極細的 **UseCase（用例）**。這樣的設計有以下幾個核心理由與優勢：

  

### 1. 遵循「單一職責原則」（Single Responsibility Principle, SRP）

- **傳統作法（大 Service）**：一個 `OrgMemberService` 隨著業務擴展，程式碼動輒幾千行，包含查詢、異動、發送通知、匯出等所有功能。這導致任何微小的修改都需要編譯與測試整個巨大的 Service。
    
      
    
- **DDD / UseCase 作法**：一個 UseCase 類別**只專注做一件事**（例如 `GetMemberProfileUseCase` 只負責「取得會員個人檔案」）。
    
      
    - 職責極度清晰，類別體積小且專一。
        
          
        
    - 修改特定業務邏輯時，影響範圍嚴格被限制在該 UseCase 內，不會意外改壞其他功能。
        
          
        

### 2. 提高代碼的「可讀性」與「意圖導向」（Expressive / Intent-driven）

在 DDD 中，講求的是**通用語言（Ubiquitous Language）** 與**直覺性**：

  

- 傳統的 `orgMemberService.getAFUser(id)` 看不出這個操作在業務層面扮演什麼角色的完整情境。
    
      
    
- 使用 `getMemberProfileUseCase.execute(memId)`，物件名稱本身就代表了**系統的一個具體業務功能（Use Case）**。任何人只要看 class 名稱，就能立刻明白這段程式碼在處理什麼業務情境，不需要進入程式碼深處猜測邏輯。
    
      
    

### 3. 符合「介面隔離原則」（Interface Segregation Principle, ISP）與精準依賴

- **傳統作法**：Controller 注入了整個 `OrgMemberService`，但該 API 端點明明只需要用其中的一個方法，卻被逼著依賴整個包含數十個方法的 Service。
    
      
    
- **DDD / UseCase 作法**：Controller 的 API 端點需要什麼業務，就只注入該業務對應的 UseCase（如 `private final GetMemberProfileUseCase ...`）。
    
      
    - **降耦極致化**：Controller 不會接觸到它不需要的業務邏輯。
        
          
        
    - **元件明確**：從 Controller 的建構子/欄位，一眼就能看出這個 Controller 到底使用了系統哪些功能。
        
          
        

### 4. 極大幅度提升「單元測試（Unit Test）」的可維護性

- **傳統作法**：測試 Controller 或撰寫 Integration Test 時，必須 Mock 一個極其龐大的 Service，或是給予複雜的初始化條件。
    
      
    
- **DDD / UseCase 作法**：
    
      
    - 在測試 Controller 時，只需要針對單一 `GetMemberProfileUseCase` 進行 Mock（例如設定 `when(getMemberProfileUseCase.execute("123")).thenReturn(...)`），測試撰寫變得非常簡潔迅速。
        
          
        
    - 針對業務邏輯測試時，也只需要單獨測試 `GetGetMemberProfileUseCaseTest`，不需要準備一整包複雜的上下文。
        
          
        

### 5. 多人協作與 Git 合併（Merge Conflict）風險極低

在大型團隊開發中：

  

- 如果大家都修改同一支龐大的 `OrgMemberService.java`，每天在解 Git Merge Conflict（程式碼衝突）就會耗費大量時間，且極易引發 Bug。
    
      
    
- 改用 UseCase 後，每個工程師開發新功能都是**新增各自獨立的 UseCase 檔案**，不同功能之間的檔案高度隔離，幾乎不會發生程式碼衝撞。
    

### 💡 總結對比

|**比較維度**|**傳統 MVC 架構 (Big Service)**|**DDD / Clean Architecture (UseCase)**|
|---|---|---|
|**注入物件**|巨大且包羅萬象的 `OrgMemberService`|專一粒度的 `GetMemberProfileUseCase`|
|**職責劃分**|一個 Service 處理幾十種業務操作|一個 UseCase 類別只負責一種業務操作|
|**代碼影響**|修改小功能可能波及其他無關功能|影響範圍嚴格鎖定在該 UseCase 內|
|**可讀性**|偏向技術導向，需要追尋方法呼叫|業務意圖極度明確（表達一個具體用例）|
|**Git 衝突**|頻繁發生（大家都改同一支大檔）|機率極低（新增獨立的 UseCase 類別）|
