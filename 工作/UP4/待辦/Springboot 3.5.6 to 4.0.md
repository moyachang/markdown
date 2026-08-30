# Spring Boot 3.5.6 → 4.0 升級檢查清單  
  
## 📋 概述  
本檔案列出將 newWFCIService 專案從 Spring Boot 3.5.6 升級到 4.0 所需的所有檢查項目和處理步驟。  
  
---  
  
## 1. Java 版本相容性  
- ✅ 目前使用 Java 21（滿足要求）  
- Spring Boot 4.0 需要 Java 17+，已相容  
  
**狀態**：✅ 無需調整  
  
---  
  
## 2. Dependency 版本更新  
  
- [ ] Spring Boot 版本：3.5.6 → 4.0.x  
- [ ] Spring Framework 版本（目前 6.2.11）  
- [ ] Spring Security 版本  
- [ ] Spring Data 版本  
- [ ] 檢查所有 Spring 系列套件的相容性  
  
**狀態**：⏳ 待處理  
  
---  
  
## 3. Jakarta EE 相容性  
  
✅ **已準備好**（專案已正確使用 Jakarta）  
  
- ✅ 已使用 `jakarta.` 命名空間（不用改 javax）  
- ✅ Shiro 已使用 Jakarta classifier  
- ✅ Hibernate Validator 8.0.1（相容）  
- ✅ Jakarta Persistence API 3.1.0（相容）  
  
**狀態**：✅ 無需調整  
  
---  
  
## 4. 第三方依賴版本檢查  
  
需要驗證以下套件與 Spring Boot 4.0 的相容性：  
  
| 依賴套件                         | 目前版本    | 狀態  | 備註              |                 |
| ---------------------------- | ------- | --- | --------------- | --------------- |
| apache-shiro                 | 2.0.1   | [ ] | 檢查相容性           |                 |
| jjwt-jackson                 | 0.11.5  | [ ] | JWT 簽章          |                 |
| nimbus-jose-jwt              | -       | [ ] | JWT 工具          |                 |
| retrofit-spring-boot-starter | 3.2.0   | [ ] | HTTP 客戶端        |                 |
| springdoc                    | 2.5.0   | [ ] | OpenAPI/Swagger |                 |
| lombok                       | 1.18.42 | [ ] | 代碼生成            |                 |
| hibernate-validator          | 8.0.1   | [ ] | 驗證框架            |                 |
| commons-lang3                | 3.3.2   | [ ] | 工具函式庫           | 3.14.0 (建議)     |
| fastjson                     | 1.2.83  | ⚠️  | 有安全風險，考慮升級      |                 |
| POI                          | 3.17    | ⚠️  | 版本較舊，建議升級       | 5.2.x           |
| httpclient                   | 4.5.6   | ⚠️  | 已知安全漏洞          | 4.5.14 or 5.2.x |
| httpcore                     | 4.4.10  | ⚠️  | 已知安全漏洞          | 4.4.16 or 5.2.x |
| bcprov-jdk15on               | 1.54    | ⚠️  | 版本較舊            |                 |
  
**狀態**：⏳ 待測試  
  
---  
  
## 5. 本地 JAR 依賴檢查  
  
以下是專案中使用的本地 JAR，需要驗證相容性：  
  
- [ ] myjs (v1.0.0)  
- [ ] wfci (v1.0.0)  
- [ ] pe (v1.0.0) - plugin/pe  
- [ ] js (v1.0.0)  
- [ ] l2fprod (v1.0.0)  
- [ ] webform (v1.0.0)  
- [ ] afui (v1.0.0)  
- [ ] json_simple (v1.0.0)  
  
**狀態**：⏳ 待檢查  
  
---  
  
## 6. 程式碼層面調整  
  
- [ ] 檢查已廢棄的 API 用法  
- [ ] 檢查 Spring Security 配置（可能有 API 變更）  
- [ ] 檢查 Spring MVC 配置  
- [ ] 檢查自訂 Jackson 配置  
- [ ] 掃描 `@Deprecated` 標記的方法使用  
  
**狀態**：⏳ 待檢查  
  
---  
  
## 7. 構建配置  
  
- [ ] Maven 版本確認（建議 3.6.3+）  
- [ ] 檢查 maven-wagon-plugin 配置  
- [ ] 檢查本地 JAR 依賴的路徑設定  
- [ ] 驗證 install-local-jars.bat 指令  
  
**狀態**：⏳ 待檢查  
  
---  
  
## 8. 安全性檢查 ⚠️  
  
**需要重點關注的安全問題**：  
  
- ⚠️ **httpclient 4.5.6** - 已知安全漏洞，建議升級到最新版本  
- ⚠️ **httpcore 4.4.10** - 已知安全漏洞，建議升級到最新版本  
- ⚠️ **bcprov-jdk15on 1.54** - 版本較舊，考慮升級到 bcprov-jdk18on 或更新版本  
- ⚠️ **fastjson 1.2.83** - 有已知的反序列化安全漏洞，建議升級或改用 Jackson  
  
**建議行動**：  
- [ ] 升級 httpclient 和 httpcore 到最新安全版本  
- [ ] 升級 bcprov 到支援 Java 21 的版本  
- [ ] 考慮用 Jackson（已在 Spring Boot 中）替代 fastjson  
  
**狀態**：⚠️ 優先處理  
  
---  
  
## 9. 測試驗證  
  
升級後需要執行完整的測試：  
  
- [ ] 執行所有 unit tests (`mvn test`)  
- [ ] 執行 integration tests  
- [ ] 檢查應用啟動是否正常  
- [ ] 驗證 JWT 簽章流程  
- [ ] 驗證數據庫連接（MSSQL）  
- [ ] 驗證 Shiro 授權邏輯  
- [ ] 驗證 OAuth2 Resource Server 配置  
- [ ] 性能測試  
  
**狀態**：⏳ 升級後執行  
  
---  
  
## 10. 廢棄功能檢查  
  
- [ ] 檢查代碼中是否使用 `Log4j2` Starter（已註釋，但確認沒有舊 imports）  
- [ ] 檢查是否有 Spring Boot 4.0 移除的特性  
- [ ] 掃描所有 `import` 陳述式中的廢棄模組  
  
**狀態**：⏳ 待檢查  
  
---  
  
## 11. 文件更新  
  
- [ ] 更新 README 或文件中的版本資訊  
- [ ] 更新相關技術文檔  
- [ ] 更新開發指南  
  
**狀態**：⏳ 升級完成後  
  
---  
  
## 📌 優先級建議  
  
### 🔴 **第一優先**（必做）  
1. 升級 Spring Boot 到 4.0.x  
2. 解決安全漏洞（httpclient、httpcore、fastjson）  
3. 驗證 Jakarta EE 相容性  
  
### 🟡 **第二優先**（強烈建議）  
1. 驗證所有 Spring 系列套件相容性  
2. 升級 POI 到較新版本  
3. 升級 bcprov 到最新版本  
  
### 🟢 **第三優先**（可選）  
1. 驗證本地 JAR 依賴相容性  
2. 程式碼優化和清理  
3. 文件更新  
  
---  
  
## 🔄 升級建議策略  
  
### **分階段升級方案**  
1. **階段一**：Spring Boot 3.5.6 → 3.6.x（驗證通過）  
2. **階段二**：Spring Boot 3.6.x → 4.0.x（完整測試）  
  
### **關鍵測試流程**  
```bash  
# 升級 pom.xml 後  
mvn clean verify  
  
# 執行應用程式  
mvn spring-boot:run  
  
# 執行測試  
mvn test  
  
# 檢查依賴樹  
mvn dependency:tree  
```  
  
---  
  
## 📝 升級記錄  
  
| 日期 | 任務 | 狀態 | 備註 |  
|------|------|------|------|  
| 2026-08-25 | 建立檢查清單 | ✅ | 初版 |  
| - | 開始升級 | ⏳ | 待開始 |  
| - | 升級完成 | ⏳ | 待完成 |  
  
---  
  
## 📚 參考資源  
  
- [Spring Boot 4.0 升級指南](https://spring.io/blog/2024/11/12/spring-boot-4-0-released)  
- [Spring Framework 6.2+ 遷移指南](https://spring.io/projects/spring-framework)  
- [Jakarta EE 遷移指南](https://jakarta.ee/)  
- [Apache Shiro 2.0 相容性](https://shiro.apache.org/download.html)  
  
---  
  
**檔案建立日期**：2026-08-25    
**最後更新**：2026-08-25


--- 另一個查詢 ---
建議版本表如下：

| 項目 | 目前版本 | Spring Boot 4 升級建議 |
|---|---:|---:|
| Spring Boot | 3.5.6 | **4.0.x**，優先使用最新穩定 patch 版 |
| Spring Framework | 6.2.11 | 由 Boot 4 的 BOM 自動管理，預期為 **7.x** |
| Spring Security | Boot 管理 | 使用 Boot 4 對應版本 |
| Jackson | Boot 管理 | 使用 Boot 4 對應版本，不要自行固定 |
| Java | 21 | **維持 Java 21** |
| Jakarta EE | Jakarta | 檢查並升級至 Boot 4 對應版本 |
| jjwt | 0.11.5 | 建議 **0.12.x** |
| Springdoc | 2.5.0 | 升級至支援 Boot 4 的版本 |
| Hibernate Validator | 8.0.1 | 確認並升級至 Boot 4 相容版本 |
| Apache POI | 3.17 | 建議 **5.2.x** |
| HttpClient | 4.5.6 | 建議 **4.5.14**；若 API 可調整則使用 5.2.x |
| HttpCore | 4.4.10 | 建議 **4.4.16**；或配合 HttpClient 5.x |
| Bouncy Castle | 1.54 | 建議 **bcprov-jdk18on 1.78+** |
| Fastjson | 1.2.83 | 建議改用 Jackson，或升級至 fastjson2 |

重點是 **`4.0.x` 表示 Spring Boot 4.0 版本系列**；`4.0.5+` 不是固定要求，實際應以升級當下發布的最新穩定版本為準。
