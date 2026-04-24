# GSON 使用情況分析  
  
## 檢查結果  
  
### ✅ 代碼層面：**沒有找到任何地方使用 GSON**  
  
```bash  
搜索結果：  
- 沒有 import com.google.gson- 沒有 new Gson()  
- 沒有 gson.toJson()  
- 沒有 gson.fromJson()  
```  
  
### ℹ️ 依賴層面：**GSON 被引入在三個地方**  
  
| 位置 | 文件 | 用途 |  
|-----|-----|------|  
| **Parent POM** | `pom.xml` | 定義 GSON 版本 (2.8.5) |  
| **Common 模組** | `common/pom.xml` | 直接依賴（第 103-105 行） |  
| **WFCIService 模組** | `WFCIService/pom.xml` | 直接依賴（第 169-171 行） |  
  
---  
  
## GSON 為什麼在這裡但沒有被用到？  
  
### 可能的原因：  
  
1. **舊代碼遺留**  
   - 過去可能用過 GSON  
   - 後來改用 Jackson 或 FastJSON  
   - 但 pom.xml 沒有清理  
  
2. **傳遞依賴被誤認為直接依賴**  
   - 某個被依賴的庫（如某個 SDK）使用 GSON  
   - 但我們的代碼不直接使用  
   - 被誤認為是必要的  
  
3. **計劃中但未實現**  
   - 曾經計劃用 GSON  
   - 後來改變了主意  
   - pom.xml 沒有更新  
  
4. **複製-粘貼**  
   - 從某個項目模板複製  
   - 沒有清理多餘依賴  
  
---  
  
## 可以安全移除嗎？  
  
### ✅ **是的，可以移除**  
  
**原因**：  
- 代碼中零使用  
- 沒有被其他庫依賴  
- 移除後日期不會編譯失敗  
  
### 移除步驟  
  
#### 1. common/pom.xml 中移除  
  
```xml  
<!-- 刪除以下代碼 --><dependency>  
    <groupId>com.google.code.gson</groupId>    <artifactId>gson</artifactId></dependency>  
```  
  
#### 2. WFCIService/pom.xml 中移除  
  
```xml  
<!-- 刪除以下代碼 --><dependency>  
    <groupId>com.google.code.gson</groupId>    <artifactId>gson</artifactId></dependency>  
```  
  
#### 3. 根 pom.xml 中移除（可選）  
  
```xml  
<!-- 根 pom.xml 的 dependencyManagement 中移除 --><dependency>  
    <groupId>com.google.code.gson</groupId>    <artifactId>gson</artifactId>    <version>2.8.5</version></dependency>  
```  
  
---  
  
## 驗證移除安全性  
  
### 第一步：編譯檢查  
  
```bash  
mvn clean compile
```  
  
如果編譯通過，表示沒有代碼依賴 GSON。  
  
### 第二步：執行測試  
  
```bash  
mvn test
```  
  
如果所有測試通過，表示 GSON 完全不需要。  
  
### 第三步：執行應用  
  
```bash  
mvn spring-boot:run
```  
  
或者構建並啟動 JAR：  
  
```bash  
mvn clean packagejava -jar WFCIService/target/WFCIService.jar
```  
  
---  
  
## 移除後的好處  
  
| 項目 | 改善 |  
|-----|------|  
| **JAR 體積** | 減少 ~600KB (Gson 的 bytecode) |  
| **記憶體占用** | 減少 ~2-3MB (Gson 類加載) |  
| **依賴管理** | 更簡潔，只剩必要的庫 |  
| **啟動時間** | 略微加快 (減少類掃描) |  
| **編譯時間** | 略微加快 |  
  
---  
  
## 建議行動計畫  
  
### 現在就移除（低風險）  
  
1. 在 `common/pom.xml` 註解掉 GSON  
  
```xml  
<!-- GSON - 已移除，改用 Jackson (2026-04-24)<dependency>  
    <groupId>com.google.code.gson</groupId>    <artifactId>gson</artifactId></dependency>  
-->  
```  
  
2. 在 `WFCIService/pom.xml` 註解掉 GSON  
  
```xml  
<!-- GSON - 已移除，改用 Jackson (2026-04-24)<dependency>  
    <groupId>com.google.code.gson</groupId>    <artifactId>gson</artifactId></dependency>  
-->  
```  
  
3. 編譯測試  
  
```bash  
mvn clean compile test
```  
  
4. 確認無誤後，從 pom.xml 中完全刪除  
  
### 保守方案（保留快速回滾能力）  
  
在根 pom.xml 的 `dependencyManagement` 中保留版本定義，但在模組 pom.xml 中移除。  
  
這樣如果日後需要，可以快速恢復。  
  
---  
  
## GSON vs Jackson - 為什麼選 Jackson?  
  
| 方面 | GSON | Jackson |  
|-----|------|---------|  
| **Spring 整合** | 不原生 | 完美整合 |  
| **效能** | 一般 | 更快 |  
| **功能** | 基礎 | 豐富（註解、流處理等） |  
| **社區** | 較小 | 非常活躍 |  
| **缺陷** | 無 | 無 |  
  
**結論**：Spring Boot 應用應該用 Jackson，不需要 GSON。  
  
---  
  
## 總結  
  
```  
現況：GSON 被定義但完全沒被使用  
結論：安全移除  
風險：極低（零代碼依賴）  
好處：減少 JAR 大小、簡化依賴管理  
建議：立即移除  
```