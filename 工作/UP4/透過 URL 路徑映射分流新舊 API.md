# 現況架構：透過 URL 路徑映射分流新舊 API  
  
## 最新更正（2026 年 4 月 22 日）  
  
之前的架構圖標註為「透過 DispatcherServlet 規則區分新舊 api」，但實際現況應該是：  
  
> **透過 Spring MVC 的 URL 路徑映射與 Security/Interceptor 規則分流，而非多 DispatcherServlet 配置**  
  
---  
  
## 現況架構圖（修正版）  
  
```text
┌────────────────────────────────────────────────────────────────┐  
│                     客户端 (Clients)                            │  
│                   Web / Mobile / API                           │  
└─────────────────────────────┬────────────────────────────────┘  
                              │                              ↓ HTTP Request┌────────────────────────────────────────────────────────────────┐  
│                    Spring Boot 應用                             │  
│                                                                │  
│  ┌───────────────────────────────────────────────────────────┐│  
│  │ Tomcat Servlet Container                                  ││  
│  │  ↓                                                         ││  
│  │ ┌─────────────────────────────────────────────────────┐  ││  
│  │ │ 單一 DispatcherServlet (Spring 標準配置)            │  ││  
│  │ │                                                     │  ││  
│  │ │  ◆ 核心職責：HTTP 請求分派 → 找到對應的 handler  │  ││  
│  │ └─────────────────────────────────────────────────────┘  ││  
│  │  ↓                                                         ││  
│  │ ┌─────────────────────────────────────────────────────┐  ││  
│  │ │ HandlerMapping（根據 URL 路徑分流）                │  ││  
│  │ ├─────────────────────────────────────────────────────┤  ││  
│  │ │ ✓ /api/**        → Legacy Controllers              │  ││  
│  │ │ ✓ /up/api/**     → UP4 Controllers                 │  ││  
│  │ │ ✓ 其他路徑       → 靜態資源 / 其他 handler         │  ││  
│  │ └─────────────────────────────────────────────────────┘  ││  
│  └───────────────────────────────────────────────────────────┘│  
│                                                                │  
│  ┌───────────────────────────────────────────────────────────┐│  
│  │ 安全與攔截層（共用，但依路徑分流）                        ││  
│  ├───────────────────────────────────────────────────────────┤│  
│  │                                                            ││  
│  │ Security Filter Chain (統一配置)                          ││  
│  │  ├─ /api/**       → permitAll (由 @Login AOP 驗證)      ││  
│  │  └─ /up/api/**    → @PreAuthorize + JWT Filter          ││  
│  │                                                            ││  
│  │ Interceptor Registry (統一配置)                           ││  
│  │  ├─ /api/**       → AuthorizationInterceptor            ││  
│  │  │                  RpaAuthorizationInterceptor         ││  
│  │  └─ /up/api/**    → (暫無，由 Security 層處理)           ││  
│  │                                                            ││  
│  └───────────────────────────────────────────────────────────┘│  
│                                                                │  
└────────────────────────────────────────────────────────────────┘  
                              │                    ┌─────────┴─────────┐                    ↓                   ↓    ┌───────────────────────┐ ┌────────────────────┐  
    │  Legacy Controllers   │ │ UP4 Controllers    │  
    │  (/api/**)            │ │ (/up/api/**)       │  
    │                       │ │                    │    │ - LaleLayoutSetting.. │ │ - LaleLayoutSetting│  
    │ - Activity            │ │ - Activity         │  
    │ - ...                 │ │ - ...              │  
    └───────────┬───────────┘ └────────┬───────────┘                │                      │                ↓                      ↓    ┌───────────────────────┐ ┌────────────────────┐  
    │ Legacy Services       │ │ UP4 Services       │  
    │ (WFCIService)         │ │ (up module)        │  
    └───────────┬───────────┘ └────────┬───────────┘                │                      │                ↓                      ↓    ┌───────────────────────┐ ┌────────────────────┐  
    │ Persistence Layer     │ │ Persistence Layer  │  
    │ (Repository/Entity)   │ │ (DDD Structure)    │  
    └───────────┬───────────┘ └────────┬───────────┘                │                      │                ↓                      ↓    ┌───────────────────────┐ ┌────────────────────┐  
    │ PASE 參考系統 / RIM   │ │ UP4 Database       │  
    └───────────────────────┘ └────────────────────┘
```  
  
---  
  
## 與舊架構圖的差異  
  
### ❌ 舊圖的問題描述  
```  
Spring Boot 內的路由  
  ├─ DispatcherServlet / 舊系統路由  
  └─ DispatcherServlet / 新系統路由  
```  
  
**理解上的混淆**：  
- 暗示有多個 DispatcherServlet  
- 讀者誤以為兩套系統物理分離  
- 沒有明確標出實際分流點  
  
### ✅ 新圖的改進  
```  
單一 DispatcherServlet  ↓HandlerMapping（URL 路徑分流）  
  ├─ /api/**     → Legacy  └─ /up/api/**  → UP4  ↓Security + Interceptor（統一配置，路徑規則分流）  
```  
  
**優點**：  
- 清楚標出只有一個 servlet  
- 明確顯示分流點是 **HandlerMapping** 與 **Security/Interceptor**  
- 避免誤解為物理隔離  
  
---  
  
## 關鍵配置對應  
  
### 1. DispatcherServlet 配置  
**檔案**：`WFCIService/src/main/java/com/flowring/WFCIServiceApplication.java`  
  
```java  
@SpringBootApplication(scanBasePackages = {"com.flowring"})@EnableJpaRepositories(basePackages = {...})  // 自動掃描 controllerpublic class WFCIServiceApplication {    public static void main(String[] args) {  
        SpringApplication.run(...);  // 建立單一 DispatcherServlet    }}  
```  
  
**現況**：  
- ✅ 單一 Spring Boot 應用 = 單一 DispatcherServlet  
- ✅ 掃描所有 package → controller 自動註冊到 HandlerMapping  
  
### 2. HandlerMapping（URL 路徑分流）  
**自動配置**（無需手寫）：  
  
```  
Spring 會掃描：  
  ├─ WFCIService/.../controller/LaleLayoutSettingController  │   @RequestMapping("/api/v1/layout-setting")  │  
  └─ up/.../controller/LaleLayoutSettingController      @RequestMapping("/up/api/layout-setting")
```  
  
**結果**：  
- `/api/v1/layout-setting` → 舊 controller  
- `/up/api/layout-setting` → 新 controller  
  
### 3. Security Filter Chain（安全層分流）  
**檔案**：`common/src/main/java/com/flowring/common/security/WebSecurityConfig.java`  
  
```java  
@Bean  
public SecurityFilterChain SecurityFilterChain(HttpSecurity http) throws Exception {    http.authorizeHttpRequests(request -> request  
        // 新系統路由  
        .requestMatchers("/up/api/hello/*", "/api/v1/auth/**").permitAll()        // 舊系統路由（由 @Login AOP 驗證，這裡放行）  
        .requestMatchers("/api/**").permitAll()        // 其他  
        .anyRequest().authenticated()    );    return http.build();  
}  
```  
  
**現況**：  
- ✅ 單一 `SecurityFilterChain`  
- ✅ 用 `requestMatchers()` 依路徑分流  
- ✅ `/api/**` 放行給 AOP (舊系統驗證機制)  
- ✅ `/up/api/**` 由 JWT filter 驗證（新系統）  
  
### 4. Interceptor（攔截層分流）  
**檔案**：`WFCIService/src/main/java/com/flowring/modules/api/config/WebMvcConfig.java`  
  
```java  
@Configuration  
public class WebMvcConfig implements WebMvcConfigurer {    @Override  
    public void addInterceptors(InterceptorRegistry registry) {  
        // 只攔舊系統的 /api/**        registry.addInterceptor(authorizationInterceptor)  
          .addPathPatterns("/api/**");        registry.addInterceptor(rpaAuthorizationInterceptor)  
          .addPathPatterns("/api/**");        // 新系統 (/up/api/**) 由 Security 層處理，不走這個 interceptor    }}  
```  
  
**現況**：  
- ✅ 攔截規則只影響舊系統  
- ✅ 新系統自動跳過  
  
---  
  
## 三層分流對比表  
  
| 分流點 | 舊架構理解 | 現況實作 | 檔案位置 |  
|------|---------|--------|--------|  
| **Servlet** | 多 DispatcherServlet | 單一 DispatcherServlet | `WFCIServiceApplication.java` |  
| **Handler Mapping** | （舊圖未明確標出） | URL 路徑分流 | Spring 自動配置 |  
| **Security** | （舊圖未明確標出） | 單一 FilterChain + 路徑規則 | `WebSecurityConfig.java` |  
| **Interceptor** | （舊圖未明確標出） | 單一 Registry + 路徑規則 | `WebMvcConfig.java` |  
  
---  
  
## 現況的優缺點  
  
### ✅ 優點（為什麼現況「路徑分流」可行）  
  
1. **開發快速**  
   - 無需自訂多個 servlet  
   - 使用 Spring Boot 預設配置  
  
2. **維護簡單**  
   - 配置集中在少數檔案  
   - 無需維護多套 Spring Context  
  
3. **資源效率**  
   - 單一 servlet 實例  
   - 共用 Spring 容器  
   - 記憶體佔用較低  
  
4. **轉移彈性**  
   - 如果未來要分開，可改用 API Gateway  
   - 邏輯層面已經分明，不是技術層面耦合  
  
### ⚠️ 缺點（可能的改進方向）  
  
1. **Interceptor 共存**  
   - 舊系統 interceptor 會走過所有請求  
   - (但通過 path pattern 篩選，影響不大)  
  
2. **Security 規則集中**  
   - 新舊規則混在一個 `authorizeHttpRequests` 裡  
   - 隨著時間增長會變複雜  
  
3. **未來難以分離**  
   - 如果要拆成獨立微服務，需要重構  
   - 目前邏輯隔離，技術上無法直接分離  
  
---  
  
## 如果要改進（建議）  
  
### 短期（保持路徑分流，讓代碼更清晰）  
  
```java  
// 分離 Security 規則定義  
class UpApiRules {  
    static final String[] PATHS = {"/up/api/**"};    // 新系統規則  
}  
  
class LegacyApiRules {  
    static final String[] PATHS = {"/api/**"};    // 舊系統規則  
}  
  
// 在 WebSecurityConfig 中使用  
http.authorizeHttpRequests(request -> request  
    .requestMatchers(UpApiRules.PATHS).authenticated()    .requestMatchers(LegacyApiRules.PATHS).permitAll());  
```  
  
### 中期（多 SecurityFilterChain，維持單一 Servlet）  
  
```java  
@Bean  
@Order(1)  
public SecurityFilterChain upFilterChain(HttpSecurity http) throws Exception {    http.securityMatcher("/up/api/**")  
        // 新系統獨立規則  
    return http.build();  
}  
  
@Bean  
@Order(2)  
public SecurityFilterChain legacyFilterChain(HttpSecurity http) throws Exception {    http.securityMatcher("/api/**")  
        // 舊系統獨立規則  
    return http.build();  
}  
```  
  
### 長期（完全分離，API Gateway + 多個 Spring Boot 應用）  
  
```  
API Gateway  
  ├─ /api/**     → WFCIService (獨立應用)  
  └─ /up/api/**  → UpService (獨立應用)  
```  
  
---  
  
## 總結：修改圖的要點  
  
### 什麼要改  
1. **DispatcherServlet 層**：改成「單一 DispatcherServlet」，不要畫兩個  
2. **HandlerMapping 層**：加上這一層，標示 URL 路徑分流點  
3. **安全層標籤**：改為「Security Filter Chain + 路徑規則」  
4. **Interceptor 層標籤**：改為「在 Legacy 內（不涉及 UP4）」  
  
### 視覺設計建議  
- 用**同一個方框**表示單一 DispatcherServlet  
- 用**箭頭標籤** `/api/**` 和 `/up/api/**` 表示分流  
- 用**顏色區分**新舊系統流向（例如藍色 = 新，紅色 = 舊）  
  
---  
  
## 對應的實作檔案參考  
  
| 概念 | 實作檔案 | 責任 |  
|-----|--------|-----|  
| DispatcherServlet 註冊 | `WFCIServiceApplication.java` | Spring Boot 自動化 |  
| URL 路徑 → Controller 映射 | `**/controller/*.java` 的 `@RequestMapping` | 開發者定義 |  
| Security 規則 | `common/.../WebSecurityConfig.java` | 統一安全配置 |  
| Interceptor 規則 | `WFCIService/.../WebMvcConfig.java` | 舊系統特定 |  
  
---  
  
## 原圖 vs 修正圖的關鍵差異  
  
| 項目 | 舊圖表達 | 修正圖表達 |  
|-----|--------|---------|  
| **DispatcherServlet 數量** | 暗示多個 | 明確單一 |  
| **分流層級** | 混淆 | 清楚標出（HandlerMapping + Security） |  
| **實現方式** | 不清楚 | 路徑規則 |  
| **技術細節** | 缺失 | 補充 path patterns |  
  
---  
  
## 如何修改原圖  
  
**建議工具**：  
- **Draw.io**（線上免費）：重繪  
- **Miro**：協作修改  
- **PowerPoint + 匯出**：簡單修改  
  
**修改步驟**：  
1. 把 `DispatcherServlet / 舊系統路由` 和 `DispatcherServlet / 新系統路由` 合併成一個 box  
2. 在中間加上 `HandlerMapping` 層，標註 `/api/**` 和 `/up/api/**`  
3. 修改 `微服務日志 (Micrometers / Actuators)` 標籤為「Security + Interceptor（路徑規則）」  
4. 用顏色或箭頭明確區分新舊流向  
  
---  
  
## 版本紀錄  
  
| 版本  | 日期         | 說明                                        |     |
| --- | ---------- | ----------------------------------------- | --- |
| 1.0 | 2026-04-22 | 建立修正版，說明實況為 URL 路徑分流，非多 DispatcherServlet |     |