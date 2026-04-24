# 為什麼有那麼多 JSON 庫?  
  
## 現況分析  
  
你的專案中使用了 4 個 JSON 庫：  
  
- **Jackson** - Spring Boot 預設  
- **FastJSON** - 阿里巴巴高性能庫  
- **GSON** - Google 官方庫  
- **Hutool** - 工具庫內建  
  
---  
  
## 原因分析  
  
### 1. 歷史遺留 (最主要原因)  
  
```  
舊系統 (WFCIService)        新系統 (UP4)
    ↓                          ↓
  用 FastJSON           用 Spring Boot 預設 (Jackson)
    ↓                          ↓
    └─────── 共存於同一個應用 ─────┘
                ↓
      引進 Hutool (工具庫)
      引進 GSON (某個依賴)
                ↓
         4 個 JSON 庫並存
```  
  
**結果**：兩個系統各用各的，導致共存。  
  
### 2. 依賴層級不同  
  
#### Jackson (Spring Boot 標準)  
```xml  
<!-- Spring Boot 內建，轉移 jackson-core, jackson-databind --><dependency>  
    <groupId>org.springframework.boot</groupId>    <artifactId>spring-boot-starter-web</artifactId></dependency>  
```  
  
#### FastJSON (阿里巴巴)  
```xml  
<dependency>  
    <groupId>com.alibaba</groupId>    <artifactId>fastjson</artifactId>    <version>1.2.83</version></dependency>  
```  
  
#### GSON (Google)  
```xml  
<dependency>  
    <groupId>com.google.code.gson</groupId>    <artifactId>gson</artifactId>    <version>2.8.5</version></dependency>  
```  
  
#### Hutool (工具庫)  
```xml  
<dependency>  
    <groupId>cn.hutool</groupId>    <artifactId>hutool-all</artifactId>    <version>5.3.5</version>  <!-- 內建包含 JSONUtil --></dependency>  
```  
  
### 3. 各模組獨立引入  
  
```  
common/pom.xml  
├─ 引入 Jackson (Spring Boot)├─ 引入 GSON (某個功能需要)  
└─ 引入 Hutool (工具類)  
  
WFCIService/pom.xml  
├─ 引入 FastJSON (舊系統偏好)  
└─ 引入 Jackson (來自 Spring Boot)  
up/pom.xml  
└─ 引入 Jackson (Spring Boot 標準)  
```  
  
---  
  
## 各庫的特性對比  
  
| 庫 | 性能 | 易用性 | 安全性 | 學習曲線 | 推薦度 |  
|---|------|--------|--------|---------|---------|  
| **Jackson** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |  
| **FastJSON** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⚠️ 曾有漏洞 | ⭐⭐ | ⭐⭐⭐ |  
| **GSON** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |  
| **Hutool** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |  
  
---  
  
## 各庫的具體用途  
  
### Jackson (推薦首選)  
  
**最適合 Spring Boot 應用**  
  
```java  
// Spring MVC @ResponseBody 序列化 JSON// 所有 REST API 回應都用 Jackson  
@RestController  
@RequestMapping("/api/activity")  
public class ActivityController {  
    @GetMapping("/{id}")    public Activity getActivity(@PathVariable String id) {        // 自動用 Jackson 序列化為 JSON        return activity;    }}  
  
// 配置示例  
ObjectMapper mapper = new ObjectMapper();  
String json = mapper.writeValueAsString(activity);  
Activity activity = mapper.readValue(json, Activity.class);  
```  
  
**優點**：  
- Spring 預設使用  
- 自動配置 @RequestMapping 回應  
- 支援 @JsonProperty, @JsonIgnore 等註解  
- 性能好，功能全  
  
### FastJSON (舊系統)  
  
**高性能，但逐漸被 Jackson 取代**  
  
```java  
// 舊系統常見寫法  
Activity activity = new Activity(...);  
String json = JSON.toJSONString(activity);  
Activity parsed = JSON.parseObject(json, Activity.class);  
  
// 直接轉為 JSONObjectJSONObject jsonObj = JSON.parseObject(json);  
String name = jsonObj.getString("name");  
```  
  
**優點**：  
- 性能最高（序列化/反序列化速度快）  
- 使用簡潔  
  
**缺點**：  
- ⚠️ 曾有安全漏洞（CVE）  
- 維護不如 Jackson 活躍  
- 與 Spring 集成度不如 Jackson  
  
### GSON (打包進依賴)  
  
**Google 官方，但在 common 模組中**  
  
```java  
// 某些舊依賴或特定功能使用  
Gson gson = new Gson();  
String json = gson.toJson(activity);  
Activity activity = gson.fromJson(json, Activity.class);  
```  
  
**優點**：  
- Google 官方支持  
- 穩定性好  
- 易學易用  
  
**缺點**：  
- 性能不如 Jackson 和 FastJSON  
- 在 Spring Boot 可能重複引用  
  
### Hutool (工具庫內建)  
  
**所有工具方法的集合**  
  
```java  
// Hutool 是工具類庫，內建 JSON 工具  
JSONObject jsonObj = JSONUtil.parseObj(jsonString);  
String json = JSONUtil.toJsonStr(activity);  
List<Activity> activities = JSONUtil.toList(  
    JSONUtil.parseArray(jsonString),    Activity.class  
);  
```  
  
**優點**：  
- 集合各種工具方法  
- 中文文檔  
- 開發快速  
  
**缺點**：  
- 性能一般  
- 功能雜糅，不夠專精  
- 增加依賴複雜度  
  
---  
  
## 現況問題  
  
### ⚠️ 問題 1: 依賴重複  
  
你同時引入了多個 JSON 庫，導致：  
- **JAR 包體積增加** (多個庫的 bytecode)  
- **維護困難** (程式員不知道用哪個)  
- **潛在衝突** (不同版本的 JSON 格式輸出可能有微妙差異)  
  
### ⚠️ 問題 2: 效能浪費  
  
四個庫都在記憶體中，但可能只用到一兩個。  
  
### ⚠️ 問題 3: 安全隱患  
  
FastJSON 曾有多個安全漏洞，使用舊版本有風險。  
  
```xml  
<!-- 版本 1.2.83 已很舊 --><dependency>  
    <groupId>com.alibaba</groupId>    <artifactId>fastjson</artifactId>    <version>1.2.83</version>  <!-- ⚠️ 應升級到 2.x --></dependency>  
```  
  
---  
  
## 推薦的統一方案  
  
### 短期 (保持現狀，但明確使用規則)  
  
```markdown  
# JSON 使用規約  
  
## 新系統 (UP4)：必須用 Jackson- 所有 @RestController 回應序列化用 Jackson- 配置文件採用 Spring Boot 預設  
  
## 舊系統 (WFCIService)：保持 FastJSON- 升級到最新版本 (fastjson 2.x)- 新功能不要再用 FastJSON  
## 工具類：統一用 Hutool JSONUtil- 工具方法、輔助轉換  
  
## 禁止使用 GSON (除非特定依賴需要)  
```  
  
### 中期 (逐步遷移)  
  
1. **新程式碼只用 Jackson**  
2. **舊系統 JSON 轉換層包裝**  
3. **移除 GSON (除非必須)**  
  
```java  
// 舊系統適配層  
public class JsonAdapter {  
    // 統一入口，隔離 FastJSON    public static String toJson(Object obj) {        return JSON.toJSONString(obj);    }        public static <T> T fromJson(String json, Class<T> clazz) {  
        return JSON.parseObject(json, clazz);    }}  
```  
  
### 長期 (完全遷移到 Jackson)  
  
```xml  
<!-- 最終狀態：只保留必要的 --><dependency>  
    <groupId>com.fasterxml.jackson.core</groupId>    <artifactId>jackson-databind</artifactId>    <!-- 來自 Spring Boot Starter Web --></dependency>  
  
<!-- Hutool 只用於工具方法 --><dependency>  
    <groupId>cn.hutool</groupId>    <artifactId>hutool-all</artifactId></dependency>  
  
<!-- 移除 FastJSON 和 GSON -->  
```  
  
---  
  
## 檢查清單  
  
- [ ] 確認現在程式碼中哪些地方用 FastJSON、GSON、Jackson  
- [ ] 標記哪些是必須的（舊系統）、哪些是可移除的  
- [ ] FastJSON 是否需要升級到 2.x 以修補安全漏洞？  
- [ ] Hutool JSONUtil 和 Jackson ObjectMapper 是否搞混？  
- [ ] 新程式碼是否都用 Jackson?  
- [ ] Spring @RestController 回應是否都由 Jackson 序列化？  
  
---  
  
## 建議立即行動  
  
### 1. 審查 pom.xml 依賴  
  
```bash  
mvn dependency:tree | grep -i json```  
  
看哪些是直接引入，哪些是傳遞依賴。  
  
### 2. 統計程式碼使用情況  
  
```bash  
# 統計 FastJSON 使用次數  
grep -r "JSON\." src/ | wc -l  
  
# 統計 Jackson 使用次數  
grep -r "ObjectMapper\|@JsonProperty" src/ | wc -l  
  
# 統計 GSON 使用次數  
grep -r "new Gson\|Gson" src/ | wc -l  
  
# 統計 Hutool 使用次數  
grep -r "JSONUtil" src/ | wc -l  
```  
  
### 3. 制定遷移計劃  
  
- 新開發一律用 Jackson  
- 舊代碼暫時保留 FastJSON（但升級版本）  
- 逐步重構舊代碼