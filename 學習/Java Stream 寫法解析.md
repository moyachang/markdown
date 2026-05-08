# Java Stream 寫法解析：`stream().map().collect()` 三段式  
  
> 以實際範例拆解 `Stream.map + Collectors.toCollection` 的寫法、原理與用途。  
  
---  
  
## 一、範例程式碼  
  
```java  
Vector<MfpRulePattern> mfpRulePatternList = patterns.stream()        .map(m -> objectMapper.convertValue(m, MfpRulePattern.class))  
        .collect(Collectors.toCollection(Vector::new));  
```  
  
### 整段功能  
  
把 `patterns`（型別 `List<Map<String, Object>>`）裡每一個 Map 轉成 `MfpRulePattern` 物件，最後收集成一個 `Vector<MfpRulePattern>`。  
  
### 等同的傳統 for 迴圈  
  
```java  
Vector<MfpRulePattern> mfpRulePatternList = new Vector<>();  
for (Map<String, Object> m : patterns) {  
    MfpRulePattern pattern = objectMapper.convertValue(m, MfpRulePattern.class);  
    mfpRulePatternList.add(pattern);  
}  
```  
  
---  
  
## 二、逐行解析  
  
### Step 1：`patterns.stream()`  
  
```java  
patterns.stream()  
```  
  
- `patterns` 是 `List<Map<String, Object>>`  
- `.stream()` 把這個 List 轉成資料流（`Stream<Map<String, Object>>`）  
- Stream 是「會逐一吐出元素」的管道，不存資料、可以串接多個操作  
  
> 想像成：把 List 裡的元素一個一個放上輸送帶。  
  
---  
  
### Step 2：`.map(m -> objectMapper.convertValue(m, MfpRulePattern.class))`  
  
```java  
.map(m -> objectMapper.convertValue(m, MfpRulePattern.class))  
```  
  
- `map(...)` 是「轉換」操作：對 stream 上每個元素套用一個 function，產出新元素  
- `m -> ...` 是 **Lambda 表達式**，等同：  
  
  ```java  
  new Function<Map<String, Object>, MfpRulePattern>() {  
      @Override      public MfpRulePattern apply(Map<String, Object> m) {          return objectMapper.convertValue(m, MfpRulePattern.class);      }  }  
  ```  
- 對輸送帶上每個 `Map`，呼叫 `objectMapper.convertValue(m, MfpRulePattern.class)`，把它變成 `MfpRulePattern`  
- 經過此步後，stream 型別從 `Stream<Map<String, Object>>` 變成 `Stream<MfpRulePattern>`  
  
> 輸送帶上的 Map 一個一個被替換成 MfpRulePattern。  
  
---  
  
### Step 3：`.collect(Collectors.toCollection(Vector::new))`  
  
把輸送帶上的元素全部「收集」成一個指定的集合。拆兩層看：  
  
#### 3-1：`Collectors.toCollection(supplier)`  
  
`Collectors` 是內建工具類，`toCollection(supplier)` 會：  
  
1. 呼叫 `supplier` 產生一個空集合  
2. 把 stream 中每個元素 `add` 進去  
3. 最後回傳這個集合  
  
#### 3-2：`Vector::new`  
  
這是 **方法參考 (Method Reference)**，等同 Lambda：  
  
```java  
() -> new Vector<>()  
```  
  
也就是「每次呼叫這個 supplier 時，就 new 一個 Vector」。  
  
#### 3-3：合起來  
  
整句 `.collect(Collectors.toCollection(Vector::new))` 的意思是：  
  
> 「請你 new 一個 `Vector`，把 stream 裡每個元素都 add 進去，最後回傳這個 Vector。」  
  
回傳型別會自動推斷為 `Vector<MfpRulePattern>`（因為 stream 是 `Stream<MfpRulePattern>`）。  
  
---  
  
## 三、為什麼用 `toCollection(Vector::new)` 而不是 `toList()`？  
  
| 寫法 | 回傳型別 |  
|------|---------|  
| `.collect(Collectors.toList())` | `List<MfpRulePattern>`（實作通常是 `ArrayList`） |  
| `.collect(Collectors.toCollection(ArrayList::new))` | `ArrayList<MfpRulePattern>` |  
| `.collect(Collectors.toCollection(Vector::new))` | `Vector<MfpRulePattern>` ✅ |  
| `.collect(Collectors.toCollection(LinkedList::new))` | `LinkedList<MfpRulePattern>` |  
  
因為原本程式需要 `Vector`（舊的 WFCI API 接受 `Vector` 參數），所以用 `toCollection(Vector::new)` 指定具體實作。  
  
---  
  
## 四、為什麼這麼寫？三個好處  
  
### 1. 可讀性高  
  
整段邏輯像一句英文：  
> 「拿 patterns，對每個元素轉成 MfpRulePattern，收集成 Vector」  
  
### 2. 程式碼短  
  
從 4 行 for 迴圈 → 3 行 stream pipeline。  
  
### 3. 容易擴充  
  
例如要過濾 null：  
  
```java  
patterns.stream()  
        .filter(Objects::nonNull)                        // 多加一行過濾  
        .map(m -> objectMapper.convertValue(m, MfpRulePattern.class))        .collect(Collectors.toCollection(Vector::new));  
```  
  
或排序：  
  
```java  
patterns.stream()  
        .map(m -> objectMapper.convertValue(m, MfpRulePattern.class))        .sorted(Comparator.comparing(MfpRulePattern::getId))        .collect(Collectors.toCollection(Vector::new));  
```  
  
傳統 for 迴圈要做這些就得多寫很多程式碼。  
  
---  
  
## 五、Stream 三大階段口訣  
  
| 階段 | 動作 | 常見方法 |  
|------|------|---------|  
| **來源** | 取得資料流 | `.stream()`、`Stream.of(...)` |  
| **中間操作** | 加工（可串接多個） | `.map()`、`.filter()`、`.sorted()`、`.distinct()` |  
| **終端操作** | 收尾（只能呼叫一次） | `.collect()`、`.forEach()`、`.count()`、`.findFirst()` |  
  
> Stream 是 lazy 的：中間操作不會立刻執行，要等終端操作（`.collect()`）被呼叫時才會真正跑一輪。  
  
---  
  
## 六、Lambda 與 Method Reference 對照  
  
```java  
// Lambda 寫法  
.map(m -> objectMapper.convertValue(m, MfpRulePattern.class))  
.collect(Collectors.toCollection(() -> new Vector<>()))  
  
// Method Reference 寫法（更簡潔）  
.collect(Collectors.toCollection(Vector::new))  
```  
  
`Vector::new` 是「建構子參考」，當 Lambda 只是「呼叫一個現成方法 / 建構子」時，可以用 `::` 簡寫。  
  
| Lambda | Method Reference |  
|--------|------------------|  
| `() -> new Vector<>()` | `Vector::new` |  
| `s -> s.length()` | `String::length` |  
| `(a, b) -> a.compareTo(b)` | `String::compareTo` |  
| `x -> System.out.println(x)` | `System.out::println` |  
  
---  
  
## 七、常用的 Collectors 速查  
  
| Collector | 用途 | 範例 |  
|-----------|------|------|  
| `toList()` | 收集成 `List` | `.collect(Collectors.toList())` |  
| `toSet()` | 收集成 `Set`（去重） | `.collect(Collectors.toSet())` |  
| `toCollection(Supplier)` | 收集成指定型別集合 | `.collect(Collectors.toCollection(Vector::new))` |  
| `toMap(keyFn, valueFn)` | 收集成 `Map` | `.collect(Collectors.toMap(User::getId, u -> u))` |  
| `joining(separator)` | 字串串接 | `.collect(Collectors.joining(", "))` |  
| `groupingBy(keyFn)` | 分組 | `.collect(Collectors.groupingBy(User::getDept))` |  
| `counting()` | 計數 | `.collect(Collectors.counting())` |  
  
---  
  
## 八、常用的 Stream 中間操作速查  
  
| 方法 | 用途 | 範例 |  
|------|------|------|  
| `map(fn)` | 一對一轉換 | `.map(String::toUpperCase)` |  
| `flatMap(fn)` | 展平巢狀 stream | `.flatMap(List::stream)` |  
| `filter(predicate)` | 過濾 | `.filter(s -> s.length() > 3)` |  
| `sorted()` / `sorted(cmp)` | 排序 | `.sorted(Comparator.reverseOrder())` |  
| `distinct()` | 去重 | `.distinct()` |  
| `limit(n)` | 取前 n 筆 | `.limit(10)` |  
| `skip(n)` | 跳過前 n 筆 | `.skip(5)` |  
| `peek(fn)` | 偷看（debug 用） | `.peek(System.out::println)` |  
  
---  
  
## 九、實戰範例：本專案場景  
  
### 範例 1：把 Map List 轉成 POJO List  
  
```java  
List<UserDto> dtoList = mapList.stream()        .map(m -> objectMapper.convertValue(m, UserDto.class))  
        .collect(Collectors.toList());  
```  
  
### 範例 2：取出某欄位組成新的 List  
  
```java  
List<String> userIds = userList.stream()        .map(User::getId)        .collect(Collectors.toList());  
```  
  
### 範例 3：過濾再轉換  
  
```java  
List<String> activeUserNames = userList.stream()        .filter(User::isActive)        .map(User::getName)        .collect(Collectors.toList());  
```  
  
### 範例 4：分組  
  
```java  
Map<String, List<User>> usersByDept = userList.stream()        .collect(Collectors.groupingBy(User::getDeptId));  
```  
  
### 範例 5：轉成 Map（id → entity）  
  
```java  
Map<String, User> userMap = userList.stream()        .collect(Collectors.toMap(User::getId, u -> u));  
```  
  
---  
  
## 十、注意事項  
  
1. **Stream 只能消費一次**：呼叫過終端操作後就不能再用，需要重新 `.stream()`。  
2. **不要在 Lambda 裡修改外部變數**：除非是 thread-safe 的，否則破壞 stream 設計理念。  
3. **效能**：對小集合而言，stream 與 for 迴圈差距不大；大集合或需要平行處理時用 `.parallelStream()`。  
4. **不必要時不要強行用 stream**：簡單操作（單純 add 一筆）用 for 反而清楚。  
  
---  
  
**文件版本**：v1.0    
**建立日期**：2026-05-06    
**相關文件**：`jsonarray-to-list-type-decision-guide.md`、`hutool-removal-guide.md`