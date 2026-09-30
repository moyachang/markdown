以下整理從 Spring Boot 3.5.6（JDK 21）升級到 Spring Boot 4 時「一定要處理」的重點項目，依照優先順序排列。

## 1. 升級前置作業:先把 3.5.x 打到最新版

升級前務必先升到最新的 3.5.x 版本,這樣才能確保你是基於該版本線最新的相依套件在建置。同時解決 3.5.x 版本跳出的所有棄用(deprecation)警告——3.5 版本本來就是為了預先標示 4.0 會變更的內容,所以這一步能免費幫你做掉大半的排查工作。

## 2. 確認 Java 與建置工具版本

雖然Spring Boot 4.0 最低要求 Java 17,你目前用 JDK 21 沒有問題,官方也建議使用 21 或 25 以取得較好的執行期表現。但要注意建置工具需為 Gradle 8.14 以上(建議 Gradle 9)或近期版本的 Maven,CI 環境的工具鏈也要一併更新。

## 3. Jackson 2 → Jackson 3(最容易踩雷的一項)

這是本次升級**衝擊最大**的變更之一:

- 套件與 groupID 從 com.fasterxml.jackson 改為 tools.jackson(jackson-annotations 例外,維持不變)。
- 既有的 Jackson 2 相依在過渡期內仍可運作,但日期時間格式化、null 處理、預設型別處理等行為差異,可能導致編譯正常卻在執行期出錯。
- 日期序列化方式、欄位順序、自訂序列化器、舊有的 mapper customizer,以及儲存好的 JSON 文件反序列化,都可能因此受影響。
- 若暫時不想處理,可在 application.yml 設定 preferred-json-mapper 為 jackson2 作為過渡方案。
- Feign 等整合也需調整,例如Jackson 3 環境下要改用 Jackson3Decoder / Jackson3Encoder 來設定 Feign 的序列化器。

## 4. Spring Security 7 預設行為改變

- 升級後最常見的問題是 REST API 回傳 403,原因是 Spring Security 7 預設開啟 CSRF 保護,原本無狀態(stateless)REST API 若沒特別設定 CSRF,升級後很可能直接壞掉。
- WebSecurityConfigurerAdapter 等在 3.x 已標示棄用的項目,在 4.0 全數移除。

## 5. 模組化與套件搬遷(Starter / Package 重整)

- 建議逐步不再使用「classic starter」,多個 starter POM 已重新命名以對齊對應的模組;舊 starter 雖仍保留但視為棄用,未來版本會移除。
- 模組化也影響專案的套件結構,每個模組現在都以專屬的 org.springframework.boot.<module> 開頭。
- 相關的還包括:RestTemplate 自動組態改為需手動開啟(opt-in)、@ConfigurationProperties 綁定到 public 欄位等功能也被移除。

## 6. 移除已棄用的 API 與測試工具

- JUnit 4 被整個移除,Jackson 升到有破壞性變更的主要版本,Spring Security 預設值也會悄悄弄壞 REST API。
- 總計移除了 36 個已棄用類別(約佔所有棄用項目的 88%),包含 MockBean 與 SpyBean(3.4 版棄用、3.5 版已提供替代方案)、多個 actuator/security/web/data/messaging/server 相關的棄用設定 key、舊版安全設定(如 WebSecurityConfigurerAdapter)、舊版 MVC/WebFlux API 與訊息轉換路徑,以及 Spring Data repository 的舊版基底類別。

## 7. Hibernate、Jakarta EE 對齊

- 此次升級涉及模組化、Jackson 3、Spring Security 7 預設值、JSpecify null 安全性等真實的破壞性變更,底層建構在Spring Framework 7 與 Jakarta EE 11 之上。
- 升級 Spring Boot 父 POM 版本會連帶升級 Hibernate 7、Jackson 3、Tomcat、Micrometer、Spring Security 等版本(BOM 強制版本一致)。

## 8. 檢查未受 Spring Boot 管理的相依套件

若專案使用非由 Spring Boot 管理的相依套件(例如 Spring Cloud),由於專案會明確指定版本,升級前需先確認相容版本。

## 9. 善用官方工具加速排查

- 可使用 OpenRewrite 的 Migrate to Spring Boot 4.0 recipe,自動處理 Spring Cloud、Spring Framework 7、Spring Security 7、Spring Batch 6、Jackson 屬性搬遷、@MockBean/@SpyBean 替換、Web Server 類別搬遷、Hibernate 7.1、Testcontainers 2.x 等大量變更。
- 官方也維護一份持續更新的 Spring Boot 4.0 Migration Guide(GitHub Wiki),建議對照著看。

---

### 建議實務步驟

1. 先升到最新 3.5.x,修完所有 deprecation warning。
2. 建立獨立分支,跑一次 OpenRewrite 的 Spring Boot 4 recipe。
3. 逐項檢視自動變更,特別留意 Jackson 相關程式碼。
4. 更新建置工具版本(Gradle/Maven)、CI 設定。
5. 針對 批次作業(batch job)、JPA repository 啟動流程、Security filter chain 加強整合測試與 staging 測試,這些是官方特別提醒容易出問題的地方。

如果你告訴我專案有沒有用到 Spring Security、Spring Cloud、Spring Data、Feign 或 Spring Modulith 等模組,我可以針對你的實際狀況列出更精確的檢查清單。

------
看過你的 pom.xml,這是一個從舊系統(Flowring WFCIService)延續下來的多模組專案,裡面混雜不少舊版套件與手動鎖定版本。真正會讓 Spring Boot 4 升級「卡關」的,主要是下面這幾類:

## 🔴 一定要處理(會導致編譯失敗或執行期直接壞掉)

**1. 手動鎖定 Spring Framework 版本**

```xml
<groupId>org.springframework</groupId>
<artifactId>spring-context</artifactId>
<version>6.2.11</version>
...
<artifactId>spring-webmvc</artifactId>
<version>6.2.11</version>
```

Spring Boot 4 底層是 Spring Framework 7,這兩個手動覆寫版本會跟 Boot 4 的 BOM 打架,造成 classpath 上版本不一致。**建議直接刪除這兩個 version 覆寫**,讓 parent BOM 統一管理。

**2. `spring-security-jwt`**

```xml
<groupId>org.springframework.security</groupId>
<artifactId>spring-security-jwt</artifactId>
```

這是 Spring Security OAuth 舊專案裡的遺留套件,已停止維護多年,跟 Spring Security 7 不相容。你專案裡同時又引入了 `spring-boot-starter-oauth2-resource-server`,兩者邏輯上有重疊,這支必須拿掉,JWT 簽發/驗證改用你已經有的 `jjwt-*` 或 `nimbus-jose-jwt` 就好。

**3. Jakarta EE API 版本被鎖定過舊**

```xml
jakarta.persistence-api  3.1.0
jakarta.validation-api   3.0.2
jakarta.annotation-api   2.1.1
```

Spring Boot 4 建構在 Jakarta EE 11 之上,這三個規範版本都需要往上跳(Persistence 3.2、Validation 3.1、Annotation 3.0)。建議移除這些手動 version,交給 Boot 4 BOM 決定。

**4. `hibernate-validator 8.0.1.Final`** 同上,這是配合 Jakarta Validation 3.0.x 的版本,Boot 4 起碼要新一版才能對齊,建議一併拿掉覆寫。

**5. `springdoc-openapi 2.5.0`** Boot 4 環境下,SpringDoc 必須升級到 3.0 以上的版本才有支援,2.x 系列不相容 Spring Framework 7 / Jakarta EE 11。

**6. JJWT + Jackson 版本衝突風險**

```xml
jjwt-api / jjwt-impl / jjwt-jackson  0.11.5
```

`jjwt-jackson` 內部依賴 Jackson **2**(`com.fasterxml.jackson`),而 Boot 4 預設會把 `spring-boot-starter-json` 換成 Jackson **3**(`tools.jackson`)。這兩套 Jackson 同時存在classpath 上很可能出問題。要嘛升級 JJWT 到支援 Jackson 3 的版本,要嘛暫時在 `application.yml` 設定:

```yaml
spring:
  http:
    converters:
      preferred-json-mapper: jackson2
```

先過渡,之後再排入 Jackson 3 遷移。

## 🟠 需要盤點確認(高機率有隱藏地雷)

**7. 內部自訂套件(myjs / wfci / plugin / pe / js / l2fprod / webform / afui / json_simple)** 這些是專案內部的舊 jar,加上你 pom 裡還留著被註解掉的 `struts-core` 依賴,合理懷疑這批程式碼源自更早的 javax.* 時代。**這是本次升級最大的未知風險**——如果這些內部 jar 裡還在用 `javax.servlet.*`、`javax.persistence.*` 等舊命名空間,Boot 4(Jakarta EE 11)環境下會直接執行期出錯,而且編譯時期不一定會報錯,要實際反編譯或找原始碼確認。

**8. Apache Shiro 2.0.1(jakarta classifier)** 需要確認這個版本的 Shiro 是否已驗證支援 Boot 4 用的 Servlet 6.1 / Jakarta EE 11,建議查一下 Shiro 官方相容矩陣。

**9. Apache POI 用的是非官方 groupId**

```xml
<groupId>poi</groupId>          <!-- 不是 org.apache.poi -->
<artifactId>poi</artifactId>
<version>3.17</version>
```

這是 2017 年的版本,又不是官方座標,懷疑是內部手動改過的 jar。建議趁這次機會改回官方 `org.apache.poi` 最新版。

**10. `commons-lang3` 鎖定在 3.3.2** 這是非常舊的版本號,Boot 4 BOM 通常會給遠新於此的版本。手動覆寫可能造成方法找不到(NoSuchMethodError)之類的問題,建議拿掉覆寫或至少手動升級。

**11. `mssql-jdbc 13.3.1.jre11-preview`** 用的是 preview 分類器,建議換成穩定的 `jreX` GA 版本。

## 🟡 建議一併處理但不阻塞升級

- `bcprov-jdk15on 1.54`:artifact 座標已改名為 `jdk18on`,且這個版本相當舊,有已知漏洞,建議一起換。
- `fastjson 1.2.83`:歷史上有多個反序列化漏洞,建議評估是否還有必要保留。

---

### 建議下一步

1. 先移除上述「手動鎖定 Spring/Jakarta 版本」的覆寫,讓 Boot 4 的 BOM 接手。
2. 針對 `spring-security-jwt` 和 `springdoc` 直接找替代/升級版本。
3. 花時間確認那批內部自訂 jar(myjs/wfci/plugin 等)有沒有殘留 javax.* API——這是隱藏最深、最容易在正式環境才爆炸的問題。

如果你能提供 `up`、`af`、`dms`、`common` 這幾個子模組的 pom.xml,我可以幫你看看有沒有更多子模組層級的版本衝突。