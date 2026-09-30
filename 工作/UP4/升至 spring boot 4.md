要注意，而且這次不能只把 `pom.xml` 的版本從 `3.5.6` 改成 `4.x` 就算完成。Spring Boot 4 是 **major upgrade**，核心會一起升到 Spring Framework 7、Spring Security 7、Jakarta EE 11、Jackson 3，並且重新拆分不少 Spring Boot 模組。

先講結論：**你們目前已經是 Java 21、Spring Boot 3.5，起點算很好；最大的風險不是 Java，而是相依套件、Jackson、自訂 Spring Boot 整合，以及測試程式。**

## 一、先問清楚「要升哪個 Spring Boot 4」

截至 2026 年 7 月，Spring Boot 4.0 已不是唯一的 Boot 4 版本，Spring Boot **4.1.0 已在 2026 年 6 月發布**。([Home](https://spring.io/blog/2026/06/10/spring-boot-4 "Spring Boot 4.1.0 available now"))

所以主管說「升到 Spring Boot 4」，至少要確認是：

- 升到最新的 `4.0.x`
    
- 還是直接升到 `4.1.x`
    

我會建議第一次升級先採：

```text
3.5 最新維護版
    ↓
4.0 最新維護版
    ↓
視需求再升 4.1
```

不要直接從 `3.5.6` 跳到 `4.1.x`。因為 Boot 4.1 已經移除 Boot 4.0 時期標記為 deprecated 的部分 API，直接跳過 4.0，會少掉一個過渡與排錯階段。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.1-Release-Notes?utm_source=chatgpt.com "Spring Boot 4.1 Release Notes"))

而且官方也明確要求：升 Boot 4 前，應先升到最新的 `3.5.x`，並先清除 Boot 3 時代的 deprecated API。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

---

# 二、你們專案最需要注意的項目

## 1. Spring Framework 6 → 7

Spring Boot 4 使用 Spring Framework 7，最低仍是 Java 17，所以你們的 Java 21 沒有問題。Boot 4 同時改成 Jakarta EE 11，Servlet baseline 是 Servlet 6.1。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

需要掃描：

```java
import org.springframework.lang.Nullable;
import org.springframework.lang.NonNull;
```

Spring Framework 7 推進 JSpecify null-safety，原本 Spring 自己的 nullable annotation 要重新檢查。若有使用 NullAway、SpotBugs、Checker Framework，或 Kotlin，可能直接產生編譯錯誤。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

你們主要是 Java，影響可能沒有 Kotlin 專案大，但仍建議搜尋：

```text
org.springframework.lang.Nullable
org.springframework.lang.NonNull
org.springframework.lang.NonNullApi
```

---

## 2. Jackson 2 → Jackson 3，是高風險區

這很可能是你們實際修改量最大的一項。

Boot 4 預設 JSON library 改成 Jackson 3，Jackson 的 Maven groupId 與 Java package 大量從：

```java
com.fasterxml.jackson
```

改為：

```java
tools.jackson
```

但 `jackson-annotations` 是例外，部分 annotation 仍留在：

```java
com.fasterxml.jackson.annotation
```

因此不會是簡單的全專案文字取代。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

需要特別盤點：

```java
ObjectMapper
JsonMapper
JsonSerializer
JsonDeserializer
JsonParser
JsonGenerator
Module
SimpleModule
Jackson2ObjectMapperBuilderCustomizer
@JsonComponent
@JsonMixin
```

其中 Boot API 也有改名：

```java
Jackson2ObjectMapperBuilderCustomizer
```

改為：

```java
JsonMapperBuilderCustomizer
```

以及：

```java
@JsonComponent → @JacksonComponent
@JsonMixin     → @JacksonMixin
```

另外，Boot 4 預設會自動註冊 classpath 上找到的所有 Jackson module；Boot 3 只註冊部分 known modules。這可能造成序列化結果、日期格式或自訂 module 執行順序改變。必要時可以先設：

```properties
spring.jackson.find-and-add-modules=false
```

避免升級初期出現無預期行為。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

### 對你們專案尤其要驗證

你們有：

- REST API DTO
    
- Retrofit
    
- JWT
    
- MapStruct
    
- Oracle／SQL Server
    
- 可能有舊系統資料交換
    

所以至少要做 JSON contract regression test，驗證：

```text
欄位名稱
null 欄位是否輸出
日期時間格式
enum 格式
BigDecimal 精度
未知欄位處理
大小寫
巢狀 DTO
錯誤回應格式
JWT claim 內容
```

Boot 4 仍可暫時讓 Jackson 2 與 Jackson 3 共存，但這只適合過渡，不建議長期同時存在兩套 `ObjectMapper`。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

---

## 3. Starter 與模組拆分

==Boot 4 重新模組化，過去集中在大型 jar 裡的 auto-configuration，現在拆成更細的模組==。某些功能即使第三方 dependency 還在，Spring Boot 的 auto-configuration 可能不會自動進來。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

例如：

```xml
spring-boot-starter-web
```

已被重新命名為：

```xml
spring-boot-starter-webmvc
```

舊名稱目前可能仍可用，但已 deprecated，未來會移除。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

建議你們最後改成：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webmvc</artifactId>
</dependency>
```

其他需盤點的 starter 包括：

```text
spring-boot-starter-security
spring-boot-starter-validation
spring-boot-starter-data-jpa
spring-boot-starter-jdbc
spring-boot-starter-quartz
spring-boot-starter-actuator
spring-boot-starter-log4j2
```

官方提供 `classic` starter 當過渡方案：

```xml
<artifactId>spring-boot-starter-classic</artifactId>
<artifactId>spring-boot-starter-test-classic</artifactId>
```

它可以先讓 classpath 接近 Boot 3，協助修正 import 與啟動問題；但官方也建議遷移完成後移除，不要長期使用。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

我會採兩階段：

```text
階段一：使用 classic starter，先讓系統能編譯、啟動、測試
階段二：移除 classic starter，補上實際需要的細分 starter
```

---

## 4. 測試程式會大量受影響

Boot 4 已移除：

```java
@MockBean
@SpyBean
```

改用 Spring Framework 提供的：

```java
@MockitoBean
@MockitoSpyBean
```

([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

大部分 field 用法可以直接替換：

```java
@SpringBootTest
class UserServiceTest {

    @MockitoBean
    private UserRepository userRepository;
}
```

但有一個重要差異：新的 `@MockitoBean`／`@MockitoSpyBean` 不能像以前一樣放在 `@Configuration` class 的 field 上。

如果你們有這種共用 mock 設計：

```java
@TestConfiguration
class MockConfig {

    @MockBean
    private UserService userService;
}
```

就必須重新設計成：

```java
@MockitoBean(types = {
    UserService.class,
    OrderService.class
})
@SpringBootTest
class ApplicationTest {
}
```

另外 Boot 4 將 test starter 依技術拆開，例如：

```text
spring-boot-starter-webmvc-test
spring-boot-starter-data-jpa-test
spring-boot-starter-security-test
spring-boot-starter-validation-test
```

如果你們有 `@WithMockUser`，官方特別提醒，要有對應的：

```xml
spring-boot-starter-security-test
```

否則可能遇到難以判斷的測試失敗。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

---

## 5. JPA／Hibernate 與資料庫

Boot 4 會帶動 Spring Data 與 Hibernate 的 major version 變動。

至少需要重新驗證：

```text
JPQL／HQL 語法
native query
Pageable 與 count query
Entity mapping
Enum mapping
CLOB／LOB
日期時間型別
sequence／identity
transaction propagation
lazy loading
dirty checking
自訂 Hibernate dialect
Criteria API
JpaSpecificationExecutor
```

你們同時支援 Oracle 與 SQL Server，而且有 CLOB、複雜 native SQL、五百萬筆以上資料表，所以不要只驗證「專案能啟動」。

尤其要跑：

```text
Oracle 整合測試
SQL Server 整合測試
大量資料分頁
複雜 NOT EXISTS／NOT IN 查詢
CLOB 欄位讀寫
交易 rollback
lock／timeout
批次操作
```

若有使用 JPA metamodel，Boot 4 的 Hibernate dependency 已將：

```xml
hibernate-jpamodelgen
```

改為：

```xml
hibernate-processor
```

([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

---

## 6. Spring Security 7

因為你們正在使用 Spring Security + JWT，這一區必須獨立列成升級工作。

需要檢查：

```java
SecurityFilterChain
HttpSecurity
authorizeHttpRequests
requestMatchers
AuthenticationProvider
OncePerRequestFilter
JwtDecoder
JwtEncoder
PasswordEncoder
Method Security
AuthenticationEntryPoint
AccessDeniedHandler
CORS
CSRF
```

還要重新做安全行為測試：

```text
未登入回 401 還是 403
JWT 過期
JWT 簽章錯誤
角色權限
URL matcher
OPTIONS 預檢
例外回應 JSON
Filter 順序
```

Boot 4 對應 Spring Security 7，官方也要求分別閱讀各 Spring portfolio 的 major-release migration notes，而不能只看 Boot migration guide。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

---

## 7. Spring Retry

Boot 4 已不再管理 Spring Retry 的版本。

如果你們有：

```java
@EnableRetry
@Retryable
@Recover
RetryTemplate
```

現在要嘛：

1. 自己明確指定 `spring-retry` 版本。
    
2. 遷移到 Spring Framework 新的 retry 能力。
    

官方較建議逐步移往 Spring Framework。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

你們的舊 RMI 相容層、Retrofit 呼叫或外部服務重試邏輯，值得搜尋一下有沒有直接或間接使用 Spring Retry。

---

## 8. Servlet container 與部署環境

Boot 4 要求 Servlet 6.1。

如果你們使用 embedded Tomcat，通常由 Boot 管理，問題較小；如果部署到公司外部的：

```text
Tomcat
WebLogic
WebSphere
JBoss／WildFly
其他公司共用 application server
```

就必須確認該版本支援 Servlet 6.1。

Undertow 支援已被 Boot 4 移除，官方也不建議把 Boot 4 應用部署到不符合 Servlet 6.1 的容器。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

你們若是：

```bash
java -jar app.jar
```

加 embedded Tomcat，通常比 WAR 外部部署簡單很多。

---

## 9. 設定檔屬性異動

升級時可以暫時加入：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-properties-migrator</artifactId>
    <scope>runtime</scope>
</dependency>
```

它會在啟動時指出已改名或移除的：

```text
application.yml
application.properties
環境變數
command-line properties
```

也會暫時協助轉換部分設定。完成後必須移除，不能把它當永久 dependency。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide "Spring Boot 4.0 Migration Guide · spring-projects/spring-boot Wiki · GitHub"))

---

# 三、你們專案建議先做的掃描

在改 Boot 版本前，先做這些搜尋：

```text
@Deprecated
@MockBean
@SpyBean
Jackson2ObjectMapperBuilderCustomizer
@JsonComponent
@JsonMixin
com.fasterxml.jackson.databind
org.springframework.lang.Nullable
org.springframework.lang.NonNull
@EnableRetry
@Retryable
RetryTemplate
hibernate-jpamodelgen
spring-boot-starter-web
spring.factories
EnvironmentPostProcessor
BootstrapRegistry
loaderImplementation
```

Maven 也先執行：

```bash
mvn clean verify
mvn dependency:tree
mvn dependency:analyze
```

建議把 dependency tree 留存成檔案：

```bash
mvn dependency:tree > dependency-tree-boot-3.5.txt
```

升級後再輸出：

```bash
mvn dependency:tree > dependency-tree-boot-4.txt
```

然後比較：

```bash
git diff --no-index dependency-tree-boot-3.5.txt dependency-tree-boot-4.txt
```

---

# 四、我建議的實際升級順序

```text
1. 先把 3.5.6 升到最新 3.5.x
2. 在 3.5.x 清除所有 deprecated warning
3. 建立 Boot 4 專用分支
4. 固定 Java 21，不要同時升 Java 版本
5. 產生並比較 dependency tree
6. 確認所有第三方套件支援 Spring 7／Jakarta EE 11
7. 先處理編譯錯誤
8. 再處理 starter／auto-configuration 缺失
9. 處理 Jackson 3
10. 修改 @MockBean／@SpyBean
11. 加入 properties migrator 處理設定
12. 執行 Oracle、SQL Server 整合測試
13. 驗證 JWT、錯誤格式與所有 API contract
14. 做效能與記憶體比較
15. 移除 classic starter 與 properties migrator
```

---

# 五、依你們目前技術棧的風險排序

|風險|項目|原因|
|---|---|---|
|很高|Jackson 3|package、API、設定與序列化行為都有變化|
|很高|第三方舊套件|可能不支援 Spring Framework 7／Jakarta EE 11|
|高|Spring Security 7 + JWT|Filter、例外與授權行為要全面回歸|
|高|JPA／Hibernate|Oracle、SQL Server、CLOB、native SQL 都要測|
|高|測試程式|`@MockBean`、`@SpyBean` 已移除|
|中高|Boot 模組化|缺 starter 時可能不是編譯錯，而是 auto-config 消失|
|中|Quartz|主要確認 Boot auto-config 與 JDBC schema|
|中|Log4j2|檢查版本、設定與 bridge 衝突|
|中|MapStruct|確認 annotation processor 與 Java 21／新依賴相容|
|中高|舊 RMI 相容層|Spring 7 未必直接影響，但 classpath 與舊 jar 相容性要完整回歸|

其中你們那套 `spring-remoting-2.0.5` 舊 jar 相容層是我最擔心的第三方／歷史依賴之一。它本來就不是 Boot 4 支援範圍，升級後即使編譯成功，也必須實際連舊 PASE Server 測試反序列化、stub lookup、dynamic proxy、RemoteInvocation 與 classloader 行為。

整體而言，這次升級**未必像 Boot 2 → 3 的 `javax` → `jakarta` 那麼全面**，但 Jackson 3、Boot 模組拆分、Spring 7 與測試 API 的變動，仍足以構成一個正式升版專案，不應被當成單純版本調整。


---

這個錯誤是 Spring Boot 4 **模組化重整**造成的典型狀況——之前提醒過的「套件搬遷」這次真的踩到了。

## 原因

`org.springframework.boot.autoconfigure.domain` 這個 package 在 Spring Boot 3.x 裡放的是 `@EntityScan` 這個註解(完整路徑是 `org.springframework.boot.autoconfigure.domain.EntityScan`)。

Spring Boot 4 把 auto-configure 相關類別依模組打散重整,`EntityScan` 這類 JPA 相關的自動組態,很可能已經搬到新的模組路徑底下(例如 `org.springframework.boot.jpa.autoconfigure.domain` 或類似的模組化命名空間)。

## 請先確認

1. `common` 模組裡是哪一個檔案 import 了 `org.springframework.boot.autoconfigure.domain.*`?通常會是:

```java
import org.springframework.boot.autoconfigure.domain.EntityScan;
```

用在 `@EntityScan(basePackages = "...")` 這種標註 JPA Entity 掃描路徑的地方。

2. 確認一下 `common` 模組的 `spring-boot-starter-data-jpa` 依賴有沒有正確被拉進來——因為 `common/pom.xml` 裡這筆依賴的版本目前是靠父 POM 決定的,如果 Boot 4 底下 JPA 相關的 starter/模組結構整個變了,可能連帶影響。找到確切答案了。

## 原因確認

在 Spring Boot 4,`@EntityScan` 這個註解已經**搬家**了:

- **舊路徑(3.x)**:`org.springframework.boot.autoconfigure.domain.EntityScan`
- **新路徑(4.x)**:`org.springframework.boot.persistence.autoconfigure.EntityScan`

這正是我先前提醒過的「模組化重整」——JPA 相關的 auto-configuration 被拆到獨立的 `spring-boot-persistence-autoconfigure` 模組底下,不再放在通用的 `autoconfigure` package 裡。

## 修正方式

找到 `common` 模組裡有 import 這一行的檔案(通常在你的 `@SpringBootApplication` 主類別,或某個設定類別):

```java
// 舊寫法,要刪除
import org.springframework.boot.autoconfigure.domain.EntityScan;
```

改成:

```java
// 新寫法
import org.springframework.boot.persistence.autoconfigure.EntityScan;
```

**`@EntityScan(...)` 註解本身的用法完全不變**,只需要改 import 那一行,底下的 `basePackages = {...}` 等寫法都一樣。

## 提醒:這類「純搬家」問題會陸續冒出來

這是 Boot 4 模組化重整的典型模式——類別本身沒變,只是 package 位置換了。目前你已經遇到:

- `EntityScan`:`autoconfigure.domain` → `persistence.autoconfigure`

接下來你可能還會遇到類似的:

- `@WebMvcTest` / `@AutoConfigureMockMvc`:從 `org.springframework.boot.test.autoconfigure.web.servlet` 搬到 `org.springframework.boot.webmvc.test.autoconfigure`(而且需要額外加 `spring-boot-starter-webmvc-test` 這個新的測試 starter)
- `@MockBean` 已被移除,要改用 Spring Framework 7 的 `@MockitoBean`

建議你在 IDE 裡全域搜尋 `org.springframework.boot.autoconfigure.` 這個字首,一次性檢查專案裡還有沒有其他用到舊路徑、之後會陸續中鏢的 import,會比一個個等編譯報錯再修更有效率。