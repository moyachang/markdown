# P6Spy SQL 日誌設定說明  
  
本文件依據本專案目前的 Spring Boot 4.0.0、P6Spy starter 2.0.0 設定整理。  
  
目標：透過既有 Logback 輸出已代入參數值的 SQL，讓日誌標頭與 SQL 分行，方便閱讀及複製。  
  
## 1. 父 POM 管理版本  
  
檔案：`pom.xml`，放在 `dependencyManagement/dependencies` 中。  
  
```xml  
<dependency>  
    <groupId>com.github.gavlyukovskiy</groupId>
    <artifactId>p6spy-spring-boot-starter</artifactId>    
    <version>2.0.0</version>
</dependency>        
```  
  
本專案由原本的 `1.9.2` 更新為 `2.0.0`，以配合 Spring Boot 4。官方 2.0.0 release 開始支援 Boot 4；升級時應確認 starter 與 Boot 的相容性。  
  
`dependencyManagement` 只管理版本，不會自動把依賴加入子模組。  
  
## 2. 啟動模組加入依賴  
  
檔案：`WFCIService/pom.xml`，放在 `dependencies` 中。  
  
```xml  
<dependency>  
    <groupId>com.github.gavlyukovskiy</groupId>    
    <artifactId>p6spy-spring-boot-starter</artifactId>
</dependency>  
```  
  
版本由父 POM 繼承，子模組不需再指定。  
  
## 3. 設定 P6Spy 輸出格式  
  
檔案：`WFCIService/src/main/resources/application.yml`。  
  
```yaml  
spring:  
  jpa:    
	  show-sql: false  
decorator:  
  datasource:    
	  p6spy:      
		  enable-logging: true      
		  logging: slf4j      
		  log-format: "%(executionTime) ms | %(category) | connection %(connectionId)\n%(sql)"  
```  
  
請合併至既有 YAML 節點，避免重複宣告 `spring` 或 `decorator`。  
  
| 設定 | 用途 |  
| --- | --- |  
| `show-sql: false` | 關閉 Hibernate 直接輸出的 SQL；Hibernate logger 需另行設定 |  
| `enable-logging: true` | 啟用 P6Spy JDBC 事件日誌 |  
| `logging: slf4j` | 交給既有 SLF4J / Logback 處理 |  
| `%(executionTime)` | JDBC 操作耗時，單位為毫秒 |  
| `%(category)` | 事件類型，例如 `statement` 或 `commit` |  
| `%(connectionId)` | P6Spy 連線識別碼 |  
| `\n` | 標頭與 SQL 之間換行；YAML 使用雙引號才會解析此跳脫字元 |  
| `%(sql)` | 顯示代入參數值後的 SQL，保留 SQL 原有換行 |  
  
輸出範例：  
  
```text  
2026-11-12 11:35:10 INFO --- [http-nio-8083-exec-2] p6spy : 2 ms | statement | connection 1  
SELECT a.ID FROM AUTH a WHERE a.MODULE = 'project' AND a.TYPE = 'manager'  
```  
  
此設定只讓標頭與 SQL 分行，不會自動替 SQL 關鍵字換行或縮排。原始 SQL 若是一整行，SQL 本身仍是一整行。  
  
### 格式選項差異  
  
| 選項 | 行為 |  
| --- | --- |  
| `%(sql)` | 已代入參數值的 SQL，保留原有換行 |  
| `%(sqlSingleLine)` | 已代入參數值的 SQL，移除換行 |  
| `%(effectiveSql)` | 傳給 JDBC driver 的 SQL，通常仍含 `?` |  
| `multiline: true` | 使用內建多行格式，不會自動排版 SQL |  
  
指定 `log-format` 後，starter 會使用 P6Spy 的 `CustomLineFormat`，不再由 `multiline` 決定格式。本專案的 `application-mssql.yml` 仍有 `multiline: false`，在目前自訂格式下不影響輸出；若該 profile 日後也設定 `log-format`，則以 profile 覆寫後的值為準。  
  
目前不需要 `PrettySqlFormat.java` 或 `spy.properties`，兩者已移除。  
  
## 4. 避免 Hibernate 重複輸出  
  
檔案：`WFCIService/src/main/resources/logback-spring.xml`。目前以下設定位於 `local` profile：  
  
```xml  
<springProfile name="local">  
    <!-- 其他既有設定省略 -->    <logger name="org.hibernate.SQL" level="OFF"/>    <logger name="org.hibernate.orm.jdbc.bind" level="OFF"/>    <logger name="p6spy" level="INFO"/>    <!-- 保留既有 root 與 appender 設定 --></springProfile>  
```  
  
Hibernate 的 SQL logger 與參數綁定 logger 分別輸出 SQL 和參數，不會合併成完整 SQL。關閉它們可避免與 P6Spy 重複，讓日誌更容易閱讀。  
  
P6Spy 透過現有 appender 寫入 console 與 `log/WFCIService.log`。請確認使用中的 profile 有啟用對應 root / appender，且日誌層級與 filter 允許 `INFO`。目前 P6Spy 的啟用設定放在共用 `application.yml`，並非只在 local 啟用；其他 profile 也可能輸出 SQL。  
  
## 5. DataSource 注意事項  
  
使用此 starter 時，會自動包裝符合條件的 Spring DataSource。既有 JDBC driver 與連線 URL 維持原設定即可，不需要改成 `P6SpyDriver` 或加上 `jdbc:p6spy:` 前綴。  
  
透過 Spring 管理且被包裝的 DataSource 執行的 Hibernate / JDBC 查詢，才會被攔截。自行透過 `DriverManager` 建立的連線不一定會經過此 starter。  
  
P6Spy 顯示的是為了除錯而重建的 SQL；實際 JDBC 仍使用參數綁定，不會因為日誌顯示完整值而改成字串拼接執行。  
  
## 6. 套用與確認  
  
1. 修改 POM 後，在 IntelliJ 執行 **Reload All Maven Projects**。  
2. 重新啟動服務；目前 Logback 的 `scan` 為 `false`，不要依賴設定檔即時重載。  
3. 呼叫包含參數查詢的 API。  
4. 檢查新產生的 `p6spy` 日誌，確認 SQL 位於標頭下一行，條件中包含實際值。  
  
如果曾編譯或打包過自訂 formatter，建議重新 clean build，避免舊的 `target/classes/spy.properties` 殘留並覆寫日誌格式。部署時使用重新建置的產物。  
  
### 常見問題  
  
| 現象 | 檢查方向 |  
| --- | --- |  
| SQL 還有 `?` | 先看 logger 是否為 `org.hibernate.SQL`；確認格式使用 `%(sql)`，而不是 `%(effectiveSql)` |  
| 同一個 SQL 印出兩次 | 檢查 `show-sql`、Hibernate SQL logger，以及 P6Spy 預設格式是否同時輸出原始與代入值後的 SQL |  
| 看不到 P6Spy 日誌 | 確認依賴已載入、服務已重啟、使用的 DataSource 被包裝，以及 logger / appender 允許 INFO |  
| 標頭與 SQL 沒分行 | 確認 YAML 使用雙引號與 `\n`，並檢查 profile、環境變數或舊 `spy.properties` 是否覆寫設定 |  
| 出現沒有 SQL 的 commit 訊息 | `commit` 屬於 JDBC 事件，沒有 SQL 內容；需要時可使用 `exclude-categories` 排除 |  
  
## 7. 使用注意事項  
  
- SQL 日誌會包含實際參數，可能含個資、token 或其他機密資料；依環境決定是否啟用，並限制日誌存取與保存時間。  
- 完整記錄 SQL 會增加格式處理、日誌 I/O 與磁碟用量，高流量環境應評估後啟用。  
- 暫停 SQL 日誌可設定 `decorator.datasource.p6spy.enable-logging: false`；這與完全停用 DataSource 裝飾不同。  
- 複製 SQL 至資料庫工具前，仍需確認資料庫語法、schema、session 設定及參數型別；日誌文字不是所有 driver 型別的精確執行重現。  
  
## 參考資料  
  
- [Starter 官方說明與設定選項](https://github.com/gavlyukovskiy/spring-boot-data-source-decorator)  
- [Starter 2.0.0：Spring Boot 4 支援](https://github.com/gavlyukovskiy/spring-boot-data-source-decorator/releases/tag/v2.0.0)  
- [P6Spy 自訂日誌格式](https://p6spy.readthedocs.io/en/latest/configandusage.html#customlogmessageformat)