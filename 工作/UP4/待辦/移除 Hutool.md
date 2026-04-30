# Hutool 移除指南  
  
## 概述  
  
Hutool 是一個中文工具庫集合。在你的專案中有 **5 個位置使用 Hutool**。  
  
本文件列出所有使用位置和移除方案，**不涉及程式碼修改**。  
  
---  
  
## 使用位置總結  
  
| 檔案                           | 類別              | 方法                         | 替代方案      |     |
| ---------------------------- | --------------- | -------------------------- | --------- | --- |
| **ActivityServiceImpl.java** | `JSONUtil`      | `parseObj()` `toJsonStr()` | Jackson   |     |
| **LayoutMapper.java**        | `BeanUtil`      | `copyProperties()`         | MapStruct |     |
| **多個 Entity**                | `Builder`       | `@Builder`                 | Lombok    |     |
| **WebSecurityConfig.java**   | `ImmutableList` | `List.of()`                | Java 原生   |     |
| **pom.xml**                  | `hutool-all`    | 依賴                         | 移除        |     |
  
---  
  
## 詳細分析  
  
### 1. ActivityServiceImpl.java - JSONUtil 使用  
  
**位置**：`WFCIService/src/main/java/com/flowring/modules/api/service/impl/ActivityServiceImpl.java`  
  
#### 使用情況  
  
**方法 1：parseObj() - JSON 字串轉物件**  
  
```java  
// 第 XXX 行  
String jsonStr = objectMapper.writeValueAsString(act);  
JSONObject jsonObject = JSONUtil.parseObj(jsonStr);  // ← Hutool  
```  
  
**用途**：將 Jackson 序列化的字串轉回 Hutool 的 JSONObject  
  
**替代方案**：直接用 Jackson ObjectMapper  
  
```java  
// 改用 JacksonObjectMapper mapper = new ObjectMapper();  
JsonNode jsonNode = mapper.readTree(jsonStr);  
// 或轉為 MapMap<String, Object> map = mapper.readValue(jsonStr, Map.class);  
```  
  
#### 使用次數  
  
- `JSONUtil.parseObj()` - 多次出現  
- 出現在迴圈中，可能有效能問題  
  
---  
  
### 2. LayoutMapper.java - BeanUtil 使用  
  
**位置**：`up/src/main/java/com/flowring/up/adapter/web/mapper/LayoutMapper.java`  
  
#### 使用情況  
  
```java  
BeanUtil.copyProperties(source, target);  // ← Hutool  
```  
  
**用途**：複製物件屬性  
  
**替代方案**：使用 MapStruct（已在專案中使用）  
  
```java  
@Mapping(source = "source.field", target = "target.field")TargetObject map(SourceObject source);  
```  
  
---  
  
### 3. 多個 Entity 的 @Builder  
  
**位置**：各 Entity 類別  
  
#### 使用情況  
  
```java  
@Entity  
@Builder  // ← Hutool 提供  
public class SomeEntity {    // ...}  
```  
  
**替代方案**：Lombok 的 @Builder（已在使用）  
  
```java  
@Entity  
@Builder  // ← Lombokpublic class SomeEntity {    // ...}  
```  
  
**說明**：導入已改為 Lombok  
  
---  
  
### 4. WebSecurityConfig.java - ImmutableList  
  
**位置**：`common/src/main/java/com/flowring/common/security/WebSecurityConfig.java`  
  
#### 使用情況  
  
```java  
ImmutableList.of("*", "GET", "POST", ...)  // ← Hutool  
```  
  
**替代方案**：Java 原生方法  
  
```java  
// 方案 A：Collections.unmodifiableList()  
Collections.unmodifiableList(  
    Arrays.asList("*", "GET", "POST", ...));  
  
// 方案 B：List.of()（Java 9+）  
List.of("*", "GET", "POST", ...)  
  
// 方案 C：Guava ImmutableList（已有 Guava）  
com.google.common.collect.ImmutableList.of(...)  
```  
  
---  
  
## 移除計劃  
  
### Phase 1：替換 JSON 工具類（高優先級）  
  
**檔案**：ActivityServiceImpl.java  
  
**現況**：  
- 混合使用 Jackson 和 Hutool  
- 效能可能受影響（多次序列化轉換）  
  
**替代方案**：統一用 Jackson ObjectMapper  
  
**估計工時**：1-2 小時  
  
---  
  
### Phase 2：替換 Bean 複製工具（中優先級）  
  
**檔案**：LayoutMapper.java  
  
**現況**：  
- 使用 Hutool BeanUtil  
- 但 MapStruct 已在專案中使用  
  
**替代方案**：全面改用 MapStruct  
  
**估計工時**：30 分鐘  
  
---  
  
### Phase 3：統一 @Builder（低優先級）  
  
**檔案**：各 Entity  
  
**現況**：  
- @Builder 註解已是 Lombok 提供  
- 無需改動（自動轉換）  
  
**替代方案**：維持不變  
  
**估計工時**：0（自動）  
  
---  
  
### Phase 4：移除 ImmutableList（低優先級）  
  
**檔案**：WebSecurityConfig.java  
  
**現況**：  
- 使用 Hutool ImmutableList  
  
**替代方案**：使用 Guava ImmutableList（已依賴）或 Java List.of()  
  
**估計工時**：15 分鐘  
  
---  
  
## 移除步驟  
  
### Step 1：替換所有使用位置  
  
#### <mark style="background:#affad1">ActivityServiceImpl.java  </mark>
- [x] 改用 Jackson ObjectMapper ✅ 2026-04-29
- [x] 移除 `JSONUtil.parseObj()` ✅ 2026-04-29
- [x] 移除 `JSONUtil` 導入 ✅ 2026-04-29
  
#### LayoutMapper.java  
- [ ] 改用 MapStruct @Mapping  
- [ ] 移除 `BeanUtil.copyProperties()`  
- [ ] 移除 `BeanUtil` 導入  
  
#### <mark style="background:#affad1">WebSecurityConfig.java </mark> 
- [ ] 改用 `List.of()` 或 Guava ImmutableList  
- [ ] 移除 Hutool `ImmutableList` 導入  
  
#### 各 Entity  
- [ ] 驗證 @Builder 來自 Lombok（應自動轉換）  
  
### Step 2：從 pom.xml 移除 Hutool  
  
```xml  
<!-- 移除此依賴 --><!--  
<dependency>  
    <groupId>cn.hutool</groupId>    <artifactId>hutool-all</artifactId>    <version>5.3.5</version></dependency>  
-->  
```  
  
### Step 3：驗證  
  
#### 編譯檢查  
```bash  
mvn clean compile
```  
  
#### 執行測試  
```bash  
mvn test
```  
  
#### 執行應用  
```bash  
mvn spring-boot:run
```  
  
---  
  
## 風險評估  
  
| 風險 | 影響 | 緩解措施 |  
|-----|------|--------|  
| JSON 轉換邏輯改變 | 中 | 充分測試活動服務 |  
| Bean 複製遺漏屬性 | 低 | MapStruct 自動檢查 |  
| 應用啟動失敗 | 中 | 逐步替換 + 每步測試 |  
| 效能下降 | 低 | 監控序列化效能 |  
  
---  
  
## 遷移時間表  
  
```  
第 1 週  
├─ 替換 ActivityServiceImpl.java JSON 工具  
└─ 執行單元測試  
  
第 2 週  
├─ 替換 LayoutMapper.java Bean 複製  
└─ 執行集成測試  
  
第 3 週  
├─ 替換 WebSecurityConfig.java ImmutableList└─ 執行端點測試  
  
第 4 週  
├─ 從 pom.xml 移除 Hutool├─ 全面測試  
└─ 驗收並發布  
```  
  
---  
  
## 預期效果  
  
### 優點  
  
✅ **減少依賴** - 一個中文工具庫  
✅ **簡化堆疊** - 使用更多業界標準工具  
✅ **减少 JAR 体积** - 約 600KB  
✅ **提升效能** - 直接使用 Jackson，減少轉換層  
  
### 缺點  
  
❌ **遷移工時** - 約 2-3 小時開發時間  
  
---  
  
## 替代工具對比  
  
| 工具 | 用途 | 優點 | 缺點 |  
|-----|------|------|------|  
| **Jackson** | JSON 序列化 | 業界標準、效能好、Spring 整合 | 需手動轉換 |  
| **Guava** | 集合工具 | 穩定、功能豐富 | 已依賴，不用 Hutool |  
| **MapStruct** | Bean 複製 | 編譯期檢查、效能最好 | 學習曲線 |  
| **Lombok** | 程式碼生成 | 減少代碼量 | 依賴編譯期處理 |  
  
---  
  
## 檢查清單  
  
### 移除前  
  
- [ ] 所有使用位置已識別  
- [ ] 替代方案已確認  
- [ ] 單元測試已編寫  
- [ ] 團隊已知曉計劃  
  
### 移除中  
  
- [ ] ActivityServiceImpl.java 已改寫  
- [ ] LayoutMapper.java 已改寫  
- [ ] WebSecurityConfig.java 已改寫  
- [ ] 所有測試通過  
- [ ] 應用啟動正常  
  
### 移除後  
  
- [ ] pom.xml 已清理  
- [ ] 無編譯警告  
- [ ] 無運行時錯誤  
- [ ] 文檔已更新  
  
---  
  
## 版本紀錄  
  
| 版本 | 日期 | 說明 |  
|-----|-----|-----|  
| 1.0 | 2026-04-29 | 初版建立，分析 Hutool 使用並提出移除方案 |