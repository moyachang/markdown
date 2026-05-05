# Retrofit 介面移除 Hutool 改用 Jackson 指南  

## 問題情境  
  
`LaleApi` 為 Retrofit 介面，呼叫外部 lale API。原本參數型別使用 Hutool 的 `cn.hutool.json.JSONObject` / `JSONArray`：  
  
```java  
@POST  
String getLoggedUsers(@HeaderMap Map<String, String> headerMap,  
                      @Url String url,  
                      @Body JSONObject body);  
```  
  
由於要移除 Hutool，需要找出可以替代的型別，且不影響 Retrofit 的序列化與外部 API 的請求行為。  
  
---  
  
## 結論  
  
**Retrofit 並不在意 `@Body` 的型別是哪一個 JSON 函式庫**，它是把物件交給設定好的 `Converter`（多半是 Jackson 或 Gson）序列化成 JSON 再送出。  所以只要把 Hutool 的型別換掉就好，**不需要改 Retrofit 設定**。  
  
可以選用以下幾種替代方案：  
  
| 替代型別 | 優點 | 適用情境 |  
|---------|------|---------|  
| `Map<String, Object>` / `List<Object>` | 最單純、零學習成本 | 欄位是動態組成，不固定 |  
| `com.fasterxml.jackson.databind.JsonNode` / `ObjectNode` / `ArrayNode` | 跟專案 Jackson 一致，可程式化建構 | 需要逐欄組裝 JSON 樹 |  
| 自訂 DTO（POJO + Lombok） | 型別安全、可讀性最佳 | 欄位固定且穩定 |  
  
---  
  
## 建議做法（最小變動）  
  
直接把 `JSONObject` → `Map<String, Object>`、`JSONArray` → `List<?>`。  
  
### 修改後的 `LaleApi`  
  
```java  
package com.flowring.modules.rest.api;  
  
import java.util.List;  
import java.util.Map;  
  
import com.github.lianjiatech.retrofit.spring.boot.core.RetrofitClient;  
  
import retrofit2.http.Body;  
import retrofit2.http.GET;  
import retrofit2.http.HeaderMap;  
import retrofit2.http.POST;  
import retrofit2.http.PUT;  
import retrofit2.http.Url;  
  
@RetrofitClient(baseUrl = "http://localhost")  
public interface LaleApi {  
  
    // 因為 lale url 從 PASE 那邊取得，所以不使用 baseUrl 添加 url ，改由 method 傳入  
    @POST    String getLoggedUsers(@HeaderMap Map<String, String> headerMap,                          @Url String url,                          @Body Map<String, Object> body);  
    // 以管理員身分取得群組資料  
    @GET    String getGroupInfoByAdm(@HeaderMap Map<String, String> headerMap, @Url String url);  
    // 以管理員身分關閉 lale 用戶登入權限  
    @PUT    String disableLaleLoginAccess(@HeaderMap Map<String, String> headerMap,                                  @Url String url,                                  @Body List<?> body);  
    // 以管理員身分恢復 lale 用戶登入權限  
    @PUT    String restoreLaleLoginAccess(@HeaderMap Map<String, String> headerMap,                                  @Url String url,                                  @Body List<?> body);  
    // 取得 lale server info    @GET    String getLaleServerInfo(@HeaderMap Map<String, String> headerMap, @Url String url);  
    // 以管理員身分取得特定外部人員好友列表  
    @GET    String getExtFriendList(@HeaderMap Map<String, String> headerMap, @Url String url);  
    // 回傳改用 String，呼叫端再用 ObjectMapper 解析；或自訂 DTO    @POST    String externalLogin(@HeaderMap Map<String, String> headerMap,                         @Url String url,                         @Body Map<String, Object> body);}  
```  
  
> 提示：建議把回傳值都統一成 `String`，呼叫端再用 `ObjectMapper.readTree(...)` 或 `readValue(..., MyDto.class)` 解析，比較好控制錯誤處理與 log。  
  
---  
  
## 呼叫端的調整範例  
  
### 原本（Hutool）  
  
```java  
JSONObject body = new JSONObject();  
body.put("userIdList", userIdList);  
body.put("page", page);  
  
String resString = laleApi.getLoggedUsers(headerMap, apiUrl, body);  
JSONObject resJson = new JSONObject(resString);  
int status = resJson.getInt("status");  
```  
  
### 改寫後（Jackson + Map）  
  
```java  
private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();  
  
Map<String, Object> body = new HashMap<>();  
body.put("userIdList", userIdList);  
body.put("page", page);  
  
String resString = laleApi.getLoggedUsers(headerMap, apiUrl, body);  
  
JsonNode root = OBJECT_MAPPER.readTree(resString);  
int status = root.path("status").asInt();  
JsonNode data = root.path("data");  
int totalNum = data.path("totalNum").asInt();  
JsonNode items = data.path("items");  
  
for (JsonNode item : items) {  
    String laleId = item.path("userId").asText();    String deviceName = item.path("deviceName").asText();    boolean loginAllow = item.path("loginAllow").asBoolean();    Integer devicePlatformType = item.has("devicePlatformType")            ? item.get("devicePlatformType").asInt() : null;    // ...}  
```  
  
重點：  
- 送出去：用 `Map<String, Object>` 組裝，Retrofit 會用 Jackson 序列化成 JSON。  
- 收回來：API 回傳 `String`，再用 `ObjectMapper.readTree` 取得 `JsonNode` 樹狀結構查值。  
- 取值用 `path()` 比 `get()` 安全（缺欄位會回 `MissingNode` 而非 `null`）。  
  
---  
  
## 進階：用 Jackson 的 ObjectNode 程式化建構  
  
若希望像 `JSONObject` 一樣以 fluent 方式組裝，也可使用 `ObjectNode`：  
  
```java  
ObjectNode body = OBJECT_MAPPER.createObjectNode();  
body.put("deviceName", "");  
body.put("page", page);  
body.put("pageSize", pageSize);  
  
ArrayNode arr = body.putArray("userIdList");  
userIdList.forEach(arr::add);  
  
if (typeEnum.getDevicePlatformType() == null) {  
    body.putNull("devicePlatformType");} else {  
    body.put("devicePlatformType", typeEnum.getDevicePlatformType());}  
  
String resString = laleApi.getLoggedUsers(headerMap, apiUrl, body);  
```  
  
此時 `LaleApi` 的 `@Body` 型別可改為 `JsonNode`：  
  
```java  
@POST  
String getLoggedUsers(@HeaderMap Map<String, String> headerMap,  
                      @Url String url,  
                      @Body JsonNode body);  
```  
  
---  
  
## 替換對照表（Hutool → Jackson / JDK）  
  
| Hutool | 建議替代 |  
|--------|---------|  
| `new JSONObject()` | `new HashMap<String, Object>()` 或 `OBJECT_MAPPER.createObjectNode()` |  
| `new JSONArray()` | `new ArrayList<>()` 或 `OBJECT_MAPPER.createArrayNode()` |  
| `new JSONObject(jsonStr)` | `OBJECT_MAPPER.readTree(jsonStr)`（取 `JsonNode`）<br>或 `OBJECT_MAPPER.readValue(jsonStr, new TypeReference<Map<String,Object>>(){})` |  
| `obj.getStr("k")` | `node.path("k").asText()` |  
| `obj.getInt("k")` | `node.path("k").asInt()` |  
| `obj.getBool("k")` | `node.path("k").asBoolean()` |  
| `obj.getJSONArray("k")` | `node.path("k")`（仍是 `JsonNode`，可 `for (JsonNode n : arr)`） |  
| `JSONNull.NULL` | `body.putNull("key")` 或 `null` |  
  
---  
  
## 注意事項  
  
1. **Retrofit Converter**：專案使用 `retrofit-spring-boot-starter`，預設 Converter 通常是 Jackson。`Map`、POJO、`JsonNode` 都能正確序列化。如果用了 Gson Converter，`JsonNode` 可能不被支援，這時用 `Map`/POJO 最安全。  
2. **null 值處理**：原本 `JSONNull.NULL` 是要明確送 `null`。改用 `Map` 時，直接 `body.put("key", null)`；用 `ObjectNode` 時，呼叫 `putNull("key")`。  
3. **回傳型別**：原本 `externalLogin` 回傳 `JSONObject`，建議改回傳 `String` 或自訂 DTO，避免介面被 Hutool 綁住。  
4. **逐步遷移**：可以先改最容易的（`Map` 取代 `JSONObject` 當參數），呼叫端解析再慢慢轉成 `JsonNode` 或 DTO。  
  
---  
  
## 總結  
  
- Retrofit 的 `@Body` 不限定 JSON 函式庫；`Map`、`List`、`JsonNode`、POJO 都可以。  
- 移除 Hutool 後，**首選 `Map<String, Object>` + Jackson `ObjectMapper`**：改動最少、與專案既有 Jackson 一致。  
- 若要保留 fluent 寫法，改用 Jackson 的 `ObjectNode` / `ArrayNode`。  
- 若欄位穩定，最好改成 DTO，獲得型別安全與 IDE 提示。
# JSONObject resJsonObj = new JSONObject(resString)怎麼改?

```java=
// 原本：
// JSONObject resJsonObj = new JSONObject(resString);
// int status = resJsonObj.getInt("status");
// if (status != 200) {
//     throw new RRException(resJsonObj.getStr("msg"), status);
// }
// JSONObject laleData = resJsonObj.getJSONObject("data");
// JSONArray laleUserInfoList = laleData.getJSONArray("userInfoList");
// JSONArray laleIdList = laleData.getJSONArray("userList");

Map<String, Object> resJsonObj = OBJECT_MAPPER.readValue(resString, MAP_TYPE);
int status = ((Number) resJsonObj.get("status")).intValue();
if (status != 200) {
    throw new RRException((String) resJsonObj.get("msg"), status);
}

@SuppressWarnings("unchecked")
Map<String, Object> laleData = (Map<String, Object>) resJsonObj.get("data");

@SuppressWarnings("unchecked")
List<Map<String, Object>> laleUserInfoList =
        (List<Map<String, Object>>) laleData.get("userInfoList");

@SuppressWarnings("unchecked")
List<String> laleIdList = (List<String>) laleData.get("userList");
```
![[Pasted image 20260505145448.png]]
注意事項：
方法簽章要加 throws JsonProcessingException（或用 try/catch），因為 readValue 會丟。
後面 laleUserInfoList.stream().forEach(...) 內 (Map) m 已經是 Jackson 反序列化出來的 LinkedHashMap，可以直接用，不需再轉。
laleIdList.stream().forEach(s -> { String laleId = (String) s; ... }) 改成 List<String> 後，可直接 for (String laleId : laleIdList)，更乾淨。