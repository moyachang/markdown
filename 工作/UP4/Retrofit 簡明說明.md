# Retrofit 簡明說明  
  
## Retrofit 是什麼？  
  
**Retrofit** 是一個 **HTTP 客戶端庫**，用於在 Java 中調用遠端 HTTP API。  
  
它可以將 REST API 調用簡化成像呼叫本地方法一樣簡單。  
  
---  
  
## 你為什麼用 Retrofit？  
  
在你的專案中，Retrofit 用來：  
  
**呼叫外部的 HTTP 服務**（如第三方 API、其他微服務）  
  
代替使用原始的 `HttpClient` 或 `RestTemplate`。  
  
---  
  
## Retrofit vs 其他方案  
  
| 方案 | 複雜度 | 適用場景 |  
|-----|-------|---------|  
| **HttpClient** | 高（繁瑣） | 低級 HTTP 操作 |  
| **RestTemplate** | 中（Spring 內建） | Spring 應用內 REST 呼叫 |  
| **WebClient** | 低（反應式） | 異步非阻塞調用 |  
| **Retrofit** | 低（優雅） | 類型安全的 REST 服務呼叫 ⭐ 你用的 |  
  
---  
  
## Retrofit 的核心概念  
  
### 1. 定義接口（Interface）  
  
```java  
// 定義一個遠端服務的接口  
public interface PaymentService {  
        @GET("/api/payment/status/{id}")  
    Call<PaymentResponse> getPaymentStatus(@Path("id") String id);        @POST("/api/payment/pay")  
    Call<PaymentResult> pay(@Body PaymentRequest request);        @GET("/api/payment/query")  
    Call<List<Payment>> queryPayments(        @Query("userId") String userId,        @Query("status") String status    );}  
```  
  
### 2. 建立 Retrofit 例實  
  
```java  
Retrofit retrofit = new Retrofit.Builder()  
    .baseUrl("https://api.payment.com")  // 遠端服務基礎 URL    .addConverterFactory(GsonConverterFactory.create())  // JSON 轉換器  
    .build();  
// 建立服務實例  
PaymentService paymentService = retrofit.create(PaymentService.class);  
```  
  
### 3. 呼叫遠端服務  
  
```java  
// 同步呼叫  
Response<PaymentResponse> response = paymentService  
    .getPaymentStatus("PAY-001")    .execute();  
PaymentResponse result = response.body();  
  
// 異步呼叫  
paymentService.pay(request).enqueue(new Callback<PaymentResult>() {  
    @Override    public void onResponse(Call<PaymentResult> call, Response<PaymentResult> response) {        PaymentResult result = response.body();    }  
    @Override    public void onFailure(Call<PaymentResult> call, Throwable t) {        System.err.println("呼叫失敗: " + t.getMessage());  
    }});  
```  
  
---  
  
## 你的專案中的配置  
  
### pom.xml 中的依賴  
  
```xml  
<!-- Retrofit Spring Boot Starter (簡化配置) -->  
<dependency>  
    <groupId>com.github.lianjiatech</groupId>    <artifactId>retrofit-spring-boot-starter</artifactId>    <version>3.2.0</version></dependency>  
  
<!-- Retrofit 核心 --><dependency>  
    <groupId>com.squareup.retrofit2</groupId>    <artifactId>retrofit-core</artifactId>    <version>2.9.0</version></dependency>  
  
<!-- Scalar 轉換器（用於 String 回應） --><dependency>  
    <groupId>com.squareup.retrofit2</groupId>    <artifactId>converter-scalars</artifactId>    <version>2.9.0</version></dependency>  
```  
  
### 實際使用例  
  
假設你的新系統 UP4 需要呼叫外部的「帳戶服務 API」：  
  
```java  
// 1. 定義接口  
@RetrofitClient(baseUrl = "${account.service.url}")  
public interface AccountServiceClient {  
        @GET("/api/account/{userId}")  
    Call<AccountDTO> getAccount(@Path("userId") String userId);        @POST("/api/account")  
    Call<AccountDTO> createAccount(@Body CreateAccountRequest request);}  
  
// 2. 在 Service 中使用  
@Service  
public class UserService {  
        @Autowired  
    private AccountServiceClient accountClient;        public AccountDTO getAccountInfo(String userId) {  
        try {            Response<AccountDTO> response = accountClient.getAccount(userId).execute();            return response.body();        } catch (IOException e) {            throw new RuntimeException("無法獲取帳戶資訊", e);  
        }    }}  
```  
  
---  
  
## Retrofit 的優點  
  
| 優點 | 說明 |  
|-----|-----|  
| **類型安全** | 編譯期檢查，避免字符串錯誤 |  
| **簡潔** | 無需手工組裝 HTTP 請求 |  
| **易測試** | 可輕鬆 Mock 接口 |  
| **靈活** | 支持攔截器、轉換器、適配器 |  
| **異步支持** | 原生支持同步和異步調用 |  
  
---  
  
## Retrofit vs RMI（你現在用的）  
  
| 特性 | Retrofit | RMI |  
|-----|---------|-----|  
| **協議** | HTTP(S) | 二進位 |  
| **防火牆穿越** | ✅ 容易 | ❌ 難 |  
| **跨語言** | ✅ 任何語言 | ❌ 僅 Java |  
| **性能** | 中等 | 高（RMI） |  
| **學習曲線** | 平緩 | 陡峭 |  
| **你現在用到嗎？** | ❓ 沒找到具體使用 | ✅ 呼叫 PASE |  
  
---  
  
## 何時使用 Retrofit？  
  
✅ **使用場景**：  
- 調用第三方 HTTP API（如支付、短信、天氣服務）  
- 微服務之間的通訊  
- 呼叫 REST 風格的遠端服務  
  
❌ **不適合**：  
- 內部進程通訊（改用 RMI 或 gRPC）  
- 高性能、低延遲要求（改用 gRPC）  
  
---  
  
## 你的項目中的潛在用途  
  
在 UP4 新系統中，如果需要呼叫外部 API：  
  
```java  
// 例：查詢外部天氣服務  
@RetrofitClient(baseUrl = "https://api.weather.com")  
public interface WeatherClient {  
    @GET("/forecast")    
    Call<WeatherData> getForecast(@Query("location") String location);
}  
  
// 例：呼叫支付閘道  
@RetrofitClient(baseUrl = "https://api.payment.com")  
public interface PaymentGatwayClient {  
    @POST("/charge")    
    Call<ChargeResult> charge(@Body ChargeRequest request);
}  
```  
  
---  
  
## 推薦閱讀  
  
- Retrofit 官方文檔：https://square.github.io/retrofit/  
- Spring Boot Retrofit Starter：https://github.com/LianjiaTech/retrofit-spring-boot-starter