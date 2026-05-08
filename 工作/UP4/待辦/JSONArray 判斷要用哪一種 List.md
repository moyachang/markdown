# JSONArray 改寫成 List 的型別判斷指南  
  
> 移除 Hutool 時，把 `cn.hutool.json.JSONArray` 改寫成 JDK `List` 的型別選擇準則。  
  
---  
  
## 一、判斷準則  
  
看 `JSONArray` 裡面**實際塞了什麼東西**，分三種情境。  
  
### 1. `List<Map<String, Object>>` — 只塞 JSON 物件  
  
當每個元素都是 key-value 結構（即原本的 `JSONObject`）時用這個。  
  
**特徵**：  
  
- `jsonArray.add(new JSONObject())`  
- `jsonArray.add(someMap)`  
- 之後讀取時用 `jsonArray.getJSONObject(i)` 或 `((JSONObject) obj).getStr(...)`  
  
**範例**：  
  
```java  
JSONArray jsonArray = new JSONArray();  
JSONObject obj = new JSONObject();  
obj.put("roleId", "R001");  
jsonArray.add(obj);  
JSONObject mainRoleBean = jsonArray.getJSONObject(0);  
```  
  
→ 改 `List<Map<String, Object>>`  
  
---  
  
### 2. `List<Object>` — 混合型別  
  
當元素**型別不一致**（混合 entity、Map、字串…等）時用這個。  
  
**特徵**：  
  
- 同一個 list 裡 `.add(entityA)` 又 `.add(entityB)`，且 A、B 不是同一個父類別  
- 或同時有 `.add(map)` 和 `.add(entity)`  
  
**範例**：  
  
```java  
jsonArray.add(quickStartProcessEntity);   // type 1  
jsonArray.add(quickStartFreqApEntity);    // type 2  
```  
  
兩個 entity 沒共同父類 → `List<Object>`  
  
> 💡 如果 `QuickStartProcessEntity` 與 `QuickStartFreqApEntity` 有共同 interface（例如 `QuickStartEntity`），就改 `List<QuickStartEntity>` 更好。  
  
---  
  
### 3. `List<具體型別>` — 同一型別  
  
只塞同一種 entity / DTO。  
  
**範例**：  
  
```java  
jsonArray.add(new UserDto(...));  
jsonArray.add(new UserDto(...));  
```  
  
→ `List<UserDto>`  
  
---  
  
## 二、快速判斷流程圖  
  
```text  
看 jsonArray.add(...) 加入的東西  
        │        ├─ 全是 JSONObject / Map     ──▶ List<Map<String, Object>>        │  
        ├─ 全是同一個類別 X            ──▶ List<X>        │  
        ├─ 不同類別但有共同父類 / 介面  ──▶ List<父類別>  
        │  
        └─ 完全異質、沒共同父類       ──▶ List<Object>  
```  
  
---  
  
## 三、API 對照表（Hutool → JDK List）  
  
| 動作 | Hutool `JSONArray` | JDK `List<Object>` |  
|------|--------------------|--------------------|  
| 加入元素 | `jsonArray.add(obj)` | `jsonArray.add(obj)` |  
| 指定位置插入 | `jsonArray.add(0, obj)` | `jsonArray.add(0, obj)` |  
| 排序 | `jsonArray.sort(...)` | `jsonArray.sort(...)` |  
| 移除 | `jsonArray.remove(0)` | `jsonArray.remove(0)` |  
| 取出 JSONObject | `jsonArray.getJSONObject(i)` | `(Map<String, Object>) jsonArray.get(i)` |  
| 大小 | `jsonArray.size()` | `jsonArray.size()` |  
  
> 大多數 API 都直接相容，只有「取出後當成 JSONObject 操作」需要強轉成 `Map`。  
  
---  
  
## 四、套用到 ProcessServiceImpl 的判斷範例  
  
| 行號區段 | 加入的內容 | 建議型別 |  
|---------|----------|---------|  
| 122~184 | `QuickStartProcessEntity` + `QuickStartFreqApEntity` | `List<Object>`（或共同 interface） |  
| 191~232 | 同樣兩種 entity | `List<Object>` |  
| 262~292 | 都是 `JSONObject`（`obj.put("roleId", ...)`） | `List<Map<String, Object>>` |  
| 1350~1406 | 都是 `new JSONObject()` 加 `put` | `List<Map<String, Object>>` |  
  
---  
  
## 五、改寫範例  
  
### 5.1 混合型別 → `List<Object>`  
  
**原本**：  
  
```java  
JSONArray jsonArray = new JSONArray();  
if (link.getId().startsWith(Constants.PROCESS_ID_PREFIX)) {  
    DBProcess pro = wfci.getDBProcess(link.getId());    QuickStartProcessEntity entity = ProcessUtil.createQuickStartProcessEntity(pro, index, locale, proList);    jsonArray.add(entity);} else {  
    PASEFreqAp freq = wfci.getPASEFreqAp(link.getId());    QuickStartFreqApEntity entity = FreqApUtil.createQuickStartFreqApEntity(freq, index, locale, freqList, user);    jsonArray.add(entity);}  
return jsonArray;  
```  
  
**改寫**（`add` 一行都不用動，只換宣告與回傳型別）：  
  
```java  
List<Object> jsonArray = new ArrayList<>();  
if (link.getId().startsWith(Constants.PROCESS_ID_PREFIX)) {  
    DBProcess pro = wfci.getDBProcess(link.getId());    QuickStartProcessEntity entity = ProcessUtil.createQuickStartProcessEntity(pro, index, locale, proList);  
    jsonArray.add(entity);  
} else {    PASEFreqAp freq = wfci.getPASEFreqAp(link.getId());    QuickStartFreqApEntity entity = FreqApUtil.createQuickStartFreqApEntity(freq, index, locale, freqList, user);  
    jsonArray.add(entity);  
}  
return jsonArray;  
```  
  
### 5.2 純 JSONObject → `List<Map<String, Object>>`  
  
**原本**：  
  
```java  
JSONArray jsonArray = new JSONArray();  
JSONObject obj = new JSONObject();  
obj.put("roleId", roleId);  
jsonArray.add(obj);  
  
JSONObject mainRoleBean = jsonArray.getJSONObject(0);  
jsonArray.remove(0);  
jsonArray.sort(Comparator.comparing(o -> ((JSONObject) o).getStr("roleId")).reversed());  
jsonArray.add(0, mainRoleBean);  
```  
  
**改寫**：  
  
```java  
List<Map<String, Object>> jsonArray = new ArrayList<>();  
Map<String, Object> obj = new HashMap<>();  
obj.put("roleId", roleId);  
jsonArray.add(obj);  
  
Map<String, Object> mainRoleBean = jsonArray.get(0);  
jsonArray.remove(0);  
jsonArray.sort(Comparator.comparing(  
        (Map<String, Object> o) -> String.valueOf(o.get("roleId"))  
    ).reversed());  
jsonArray.add(0, mainRoleBean);  
```  
  
---  
  
## 六、補充建議  
  
當不確定時，**優先選 `List<Map<String, Object>>`**，因為：  
  
1. 大部分 Hutool 的 `JSONArray` 都是用來組裝 JSON 回傳給前端，內容本來就是 Map  
2. 比 `List<Object>` 更有型別資訊，後續操作（`.get("xxx")`）不用再強轉  
3. Jackson 序列化成 JSON 時行為與 `JSONObject` 完全一致  
  
只有確定**真的塞了不是 Map 的物件**（例如 entity / POJO）才用 `List<Object>` 或 `List<具體型別>`。  
  
---  
  
## 七、檢查步驟（Checklist）  
  
改寫一個 `JSONArray` 變數時，依序確認：  
  
- [ ] 找出該變數所有 `.add(...)` 呼叫，看加入的是什麼型別  
- [ ] 是否全為 Map / JSONObject？→ `List<Map<String, Object>>`  
- [ ] 是否全為同一個 POJO？→ `List<該 POJO>`  
- [ ] 是否混合多種無共同父類的型別？→ `List<Object>`  
- [ ] 同步修改 method 回傳型別、interface、呼叫端  
- [ ] 取值處 `getJSONObject(i)`、`getStr(...)` 改成 `get(i)` + 強轉 Map / `map.get(...)`  
- [ ] import 移除 `cn.hutool.json.*`，加入 `java.util.*`  
  
---  
  
**文件版本**：v1.0    
**建立日期**：2026-05-06    
**相關文件**：`hutool-removal-guide.md`、`hutool-class-usage-analysis.md`