如果不用 **Hutool**，以下是常見功能的替代方案整理：

---

**1. JSONObject 操作（最常用）** Hutool 的 `JSONObject` 可用以下替代：

| 替代方案          | 說明                 |
| ------------- | ------------------ |
| **Jackson**   | Spring Boot 內建，最主流 |
| **Gson**      | Google 出品，輕量簡單     |
| **Fastjson2** | 阿里出品，效能優秀          |

Jackson 範例：

```java
ObjectMapper mapper = new ObjectMapper();

// 物件轉 JSON 字串
String json = mapper.writeValueAsString(user);

// JSON 字串轉物件
User user = mapper.readValue(json, User.class);

// JSON 字串轉 Map（類似 JSONObject 操作）
Map<String, Object> map = mapper.readValue(json, Map.class);
```

---

**2. 其他 Hutool 常用功能替代**

|Hutool 功能|替代方式|
|---|---|
|`DateUtil` 日期處理|Java 8 `LocalDateTime`、`DateTimeFormatter`|
|`StrUtil` 字串工具|Apache Commons Lang `StringUtils`|
|`CollUtil` 集合工具|Guava `Collections2`、JDK Stream API|
|`BeanUtil` 物件複製|MapStruct、BeanUtils|
|`HttpUtil` HTTP 請求|`RestTemplate`、`WebClient`、OkHttp|
|`FileUtil` 檔案操作|Apache Commons IO、NIO `Files`|
|`IdUtil` UUID/雪花ID|`UUID.randomUUID()`、自製 Snowflake|
|`DigestUtil` 加密雜湊|Java 內建 `MessageDigest`|

---

**建議：** 若專案已有 Spring Boot + Jackson，處理 JSON 完全夠用，不一定需要 Hutool。但如果習慣 Hutool 的簡潔 API，繼續用也完全合理。

BeanUtil.beanToMap(ffw)
OBJECT_MAPPER.convertValue(ffw, MAP_TYPE)

```java
private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();
private static final TypeReference<Map<String, Object>> MAP_TYPE =
        new TypeReference<Map<String, Object>>() {};
```
 
 ```java
 // 將 form 物件轉成 Map（取代 new JSONObject(form)）
        Map<String, Object> body = OBJECT_MAPPER.convertValue(form, MAP_TYPE);
 ```


對於參數的修改判斷, 註解掉的地方，是比較不安全的寫法。
這裡用的是 "import java.util.Objects;"
```java
// String id = String.valueOf(params.get("id"));  
String id = Objects.toString(params.get("id"), "");  
// boolean status = (Boolean) params.get("status");  
Object statusObj = params.get("status");  
boolean status = Boolean.parseBoolean(Objects.toString(statusObj, "false"));
```

## 判斷 null 及空值問題
> [!info]
> 判斷 null 及空值問題。

因為：

```java
resMsg.isEmpty()
```

在 `resMsg == null` 時，會直接炸：

```text
NullPointerException
```

---

# 原因

`isEmpty()` 是：

```java
String
```

物件的方法。

等於：

```java
resMsg.isEmpty()
```

其實是：

```java
呼叫 resMsg 這個物件的方法
```

但如果：

```java
resMsg == null
```

代表：

```text
根本沒有 String 物件
```

所以：

```java
null.isEmpty()
```

一定炸。

---

# 範例

## 正常

```java
String str = "";

str.isEmpty();
```

OK。

---

## 但這個

```java
String str = null;

str.isEmpty();
```

直接：

```text
NullPointerException
```

---

# 所以才會寫：

```java
if (resMsg == null || resMsg.isEmpty())
```

---

# 這裡有個 Java 很重要的特性

## `||` 是短路運算（Short Circuit）

意思：

```java
A || B
```

如果：

```java
A == true
```

那：

```java
B 不會執行
```

---

# 所以：

```java
resMsg == null || resMsg.isEmpty()
```

當：

```java
resMsg == null
```

時：

左邊：

```java
true
```

Java 就不會再執行：

```java
resMsg.isEmpty()
```

因此安全。

---

# 但如果順序反了

這樣：

```java
if (resMsg.isEmpty() || resMsg == null)
```

就錯了。

因為：

```java
resMsg.isEmpty()
```

已經先炸。

---

# 你可能會問

## 那 service 不應該保證不回 null 嗎？

其實：

```text
是，理想上應該如此
```

例如：

```java
return "";
```

而不是：

```java
return null;
```

---

# 但舊系統常常：

- DAO 回 null
    
- service 回 null
    
- 外部 API 回 null
    
- legacy code 沒保證
    

所以：

```java
null-safe
```

防禦性寫法還是很常見。

---

# Java 11 之後更常見寫法

你也可以：

```java
if (resMsg == null || resMsg.isBlank())
```

---

# isEmpty vs isBlank

## isEmpty

只判斷：

```java
""
```

---

## isBlank

連：

```java
" "
"\t"
"\n"
```

都算空。

---

# 範例

|字串|isEmpty|isBlank|
|---|---|---|
|`""`|true|true|
|`" "`|false|true|
|`"\t"`|false|true|

---

# 舊系統我通常建議

如果是：

```text
錯誤訊息
response message
前端輸入
```

用：

```java
if (resMsg == null || resMsg.isBlank())
```

會更穩。

---

# 更乾淨的方式（Apache Commons）

如果你有：

Apache Commons Lang

可以：

```java
StringUtils.isBlank(resMsg)
```

它內部已經處理：

- null
    
- ""
    
- 空白字元
    

---

# 結論

## 單寫：

```java
resMsg.isEmpty()
```

不安全。

因為：

```java
resMsg 可能為 null
```

---

# 安全寫法

## 基本版

```java
resMsg == null || resMsg.isEmpty()
```

---

## 更推薦

```java
resMsg == null || resMsg.isBlank()
```

---

## <font color="#ff0000">最方便（有 commons-lang）</font>

```java
StringUtils.isBlank(resMsg)
```

判斷參數，不要用 String.valueOf
```
	String meetId = String.valueOf(params.get("meetId"));
		String memId = String.valueOf(params.get("memId"));
		String name = String.valueOf(params.get("name"));
		String content = String.valueOf(params.get("content"));
		String note = String.valueOf(params.get("note"));
```

改用 Objects.toString()
```java
String meetId = Objects.toString(params.get("meetId"),"");  
String memId = Objects.toString(params.get("memId"),"");  
String name = Objects.toString(params.get("name"),"");  
String content = Objects.toString(params.get("content"),"");  
String note = Objects.toString(params.get("note"),"");
Boolean isSuccess = Boolean 
isSuccess =Boolean.parseBoolean(Objects.toString(resMap.get("success"),"false"));
```

integer的安全轉換
```java
Integer.parseInt(Objects.toString(resMap.get("number"), "0")
);
```

boolean  的安全轉換
```java
Boolean.parseBoolean(Objects.toString(resMap.get("status"), "false"));
```