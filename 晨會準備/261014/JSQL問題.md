**完全正確！你的直覺非常敏銳，這正是 JPA / JPQL 長久以來為人詬病的大痛點之一。**

  

像這種寫在 `@Query` 字串裡的完整類別路徑（Fully Qualified Name），只要你用 IDE（如 IntelliJ IDEA）進行 **Refactor（重新命名或搬移套件）**，有高達 80% 的機率會**漏掉不幫你自動更新**。

  

這會導致什麼後果？

  

- IDE 靜態檢查通常**抓不到**字串裡面的路徑錯誤。
    
      
    
- 直到你實際呼叫到這個 API、執行這段 SQL 時，Spring 才會噴出 `java.lang.IllegalArgumentException` 或 `ClassNotFoundException` 崩潰。
    
      
    

### 🛡️ 如何避開這個坑？（幾種常見的改良解法）

如果你想避免這種「改了路徑卻忘記改字串」的維護惡夢，可以考慮以下幾種更安全寫法：

  

#### 1. 改用介面投影（Spring Data Projections）—— 最推薦

Spring Data JPA 支援直接用 **Interface（介面）** 來接查詢結果，不需要在 JPQL 裡面寫 `new com.xxx...` 宣告實體。

  

- **作法範例**：
    
      
    
    Java
    
    ```java
    public interface RoleMemberRow {
        String getRolId();
        String getMemId();
    }
    ```
    
- **Repository 寫法**：
    
      
    
    Java
    
    ```java
    @Query("select m.id.rolId as rolId, m.id.memId as memId from Up4ProjectRolMemJpaEntity m where m.id.projectId = :projectId")
    List<RoleMemberRow> findRows(@Param("projectId") String projectId);
    ```
    
- **好處**：
    
      
    - 不需要寫 `new com...`。
        
          
        
    - 如果 `RoleMemberRow` 介面被搬移或改名，IDE 的 **Refactor 會自動連動修改**，絕對不會漏掉！
        
          
        

#### 2. 減少跨層級引用（把 Row 物件放在 Infrastructure 層）

如果你堅持要用 class 來接，不要把這個 Row 物件放在 `application` 或 `domain` 層。

把它直接放在**與 Repository 同一個套件（Infrastructure 層）**，因為它本來就是為了適應資料庫查詢而生的「基礎設施細節」。這樣就算未來業務邏輯的 DTO 怎麼搬風，這個底層查詢物件也不會跟著亂跑。

  

#### 3. 導入 Type-Safe 查詢工具（如 QueryDSL 或 jOOQ）

如果專案規模較大、有大量這種複雜的投影查詢，改用 **QueryDSL** 或 **jOOQ**。

  

- 它們的所有欄位和類別都是透過**編譯期生成的 Metamodel** 來操作（例如 `QRoleMemberRow.roleMemberRow`）。
    
      
    
- 只要你改了類別名稱或路徑，**一按 Compile 就會直接報錯**，不可能讓你帶著錯誤扣上線。


實際修改
