<mark style="background:rgba(240, 200, 0, 0.2)"># HandlerMapping 在你程式中的位置  </mark>
  
<mark style="background:rgba(240, 200, 0, 0.2)">## 你的程式沒有顯式定義 HandlerMapping，但它隱含存在</mark>  
  
### 1. 自動註冊（最重要）  
  
在 `WFCIServiceApplication.java` 中：  
  
```java  
@SpringBootApplication(scanBasePackages = {"com.flowring"})
```  
  
這一行會讓 Spring Boot 自動：  
- 掃描所有 `com.flowring` 包下的 Controller  
- 建立 `RequestMappingHandlerMapping` 實例  
- 自動註冊所有 `@RequestMapping` 和 `@RestController`  
  
### 2. 舊 API 的 URL 映射（Legacy）  
  
程式位置：`WFCIService/src/main/java/com/flowring/modules/api/controller/*.java`  
  
每個 Controller 都有 `@RequestMapping`：  
  
```java  
@RequestMapping("/api/access")         // AccessController  
@RequestMapping("/api/activity")       // ActivityController  
@RequestMapping("/api/albums")         // AlbumController  
@RequestMapping("/api/bot")            // BotController  
// 共 20+ 個 Controller，都走 /api/** 路徑  
```  
  
### 3. 新 API 的 URL 映射（UP4）  
  
程式位置：`up/src/main/java/com/flowring/up/adapter/web/controller/*.java`  
  
例如：  
  
```java  
@RestController  
@RequestMapping("/up/api/activities")   // UP4 ActivityController  
public class ActivityController {    // ...}  
```  
  
---  
  
## 完整映射關係  
  
```  
Spring Boot 啟動  
  |  
  v
@SpringBootApplication(scanBasePackages = {"com.flowring"})  
  |  
  v
掃描 com.flowring.* 下的所有 @RequestMapping  
  |  
  v
Spring 自動建立 RequestMappingHandlerMapping  
  |  
  v
[分流點]  
  |---> /api/**        : WFCIService 的 20+ 個舊 Controller  
  |---> /up/api/**     : up 模組的新 Controller
```  
  
---  
  
## 驗證清單  
  
| 檢查項 | 現況 | 狀態 |  
|------|------|------|  
| 有 `@SpringBootApplication` | 有 | OK |  
| `scanBasePackages` 包含所有模組 | `{"com.flowring"}` | OK |  
| 舊 Controller 有 `@RequestMapping("/api/**")` | 有 20+ 個 | OK |  
| 新 Controller 有 `@RequestMapping("/up/api/**")` | 有 | OK |  
| 有 `WebMvcConfigurer` 配置 interceptor | 有 | OK |  
| HandlerMapping 自動建立 | 自動 | OK |  
  
---  
  
## 結論  
  
你的程式：  
  
- **HandlerMapping 存在且工作正常**（自動建立）  
- **路徑分流清楚**（`/api/**` 和 `/up/api/**`）  
- **無需顯式配置**（Spring 自動處理）  
  
Spring 自動建立的 `RequestMappingHandlerMapping` 根據你的 `@RequestMapping` 註解，把請求分派到不同的 Controller。  
  
---  
  
## 如果你想顯式配置 HandlerMapping（可選）  
  
在 `WebMvcConfig.java` 中加：  
  
```java  
@Configuration  
public class WebMvcConfig implements WebMvcConfigurer {    @Override  
    public void addInterceptors(InterceptorRegistry registry) {  
        registry.addInterceptor(authorizationInterceptor).addPathPatterns("/api/**");  
        registry.addInterceptor(rpaAuthorizationInterceptor).addPathPatterns("/api/**");  
    }    // 可選：顯式定義 HandlerMapping    @Bean  
    public RequestMappingHandlerMapping requestMappingHandlerMapping() {  
        RequestMappingHandlerMapping mapping = new RequestMappingHandlerMapping();        return mapping;  
    }}  
```  
  
**建議：不用改，現在的隱含配置更簡潔！**