**一開始是想把原設計換掉，排序值由後端計算。**
```json
// 原設計 
{ 
"parentId": "PRJ123", // 上層 id，若是 phase，上層是 project 
"type": "phase", 
"order": 1 // 新增的位置 
}

{
  "projectId": "PRJ_2026_01",
  "parentId": null,
  "type": "phase",
  "prevTaskId": "STAGE_01",
  "nextTaskId": "STAGE_02"
}
```

==後來 shane 告知他的考量，但是我好像看不懂，A害。==
```text
parentId 跟 projectId 沒問題， 
前端新增的行為在某筆資料上選擇在上方插入或是下方插入， 
我在想有什麼方式可以讓前後端比較好處理
```
後來問 AI，真的大夠神，還是我真的太笨?? 嗚...

他這次在想的，應該已經不是 `parentId` 和 `projectId`，而是：

 前端新增任務時，使用者是對著某一筆資料按「在上方新增／在下方新增」，那 API 到底要傳「前後兩筆 ID」，還是只傳「基準任務＋上方／下方」比較好處理？

例如畫面目前是：

```text
任務 A
任務 B
任務 C
```

使用者在「任務 B」上選擇「在下方插入」，前端最直接知道的是：

```text
基準資料：任務 B
插入方向：下方
```

所以他可能在考慮讓前端傳：

```json
{
  "projectId": "PRJ_2026_01",
  "parentId": "STAGE_01",
  "type": "mainTask",
  "referenceTaskId": "TASK_B",
  "position": "AFTER"
}
```

後端再自己查出：

```text
TASK_B 是前一筆
TASK_C 是後一筆
```

然後用：

```java
generateKeyBetween(taskB.getOrder(), taskC.getOrder());
```

產生新任務的排序值。

如果使用者選擇「在任務 B 上方插入」：

```json
{
  "referenceTaskId": "TASK_B",
  "position": "BEFORE"
}
```

後端就查出：

```text
TASK_A 是前一筆
TASK_B 是後一筆
```

## 他考慮的兩種做法

### 你的設計：前端直接傳前後兩筆

```json
{
  "projectId": "PRJ_2026_01",
  "parentId": "STAGE_01",
  "type": "mainTask",
  "prevTaskId": "TASK_A",
  "nextTaskId": "TASK_B"
}
```

優點是後端很清楚要插在哪兩筆之間，也可以直接取得兩筆排序值。

但前端要負責：

- 找出前一筆。
    
- 找出後一筆。
    
- 處理第一筆和最後一筆。
    
- 根據「上方／下方」轉換成 `prevTaskId`、`nextTaskId`。
    

### 他可能在想的設計：基準任務＋插入方向

```json
{
  "projectId": "PRJ_2026_01",
  "parentId": "STAGE_01",
  "type": "mainTask",
  "referenceTaskId": "TASK_B",
  "position": "BEFORE"
}
```

或：

```json
{
  "referenceTaskId": "TASK_B",
  "position": "AFTER"
}
```

前端只要把使用者的實際操作傳給後端，不用自己換算前後兩筆；後端根據目前資料庫的最新順序，找出相鄰任務並產生 fractional index。

## 我認為新增比較適合他的想法

如果畫面上的新增功能確實是：

> 點某筆資料 → 選擇在上方新增或下方新增

那麼新增 API 使用：

```text
referenceTaskId + position
```

會比傳：

```text
prevTaskId + nextTaskId
```

更貼近使用者操作，而且前端比較簡單。

Java DTO 可以這樣設計：

```java
public record CreateTaskRequest(
    String projectId,
    String parentId,
    TaskType type,
    String referenceTaskId,
    InsertPosition position
) {
}
```

```java
public enum InsertPosition {
    BEFORE,
    AFTER,
    FIRST,
    LAST
}
```

不過 `FIRST`、`LAST` 是否需要，取決於畫面有沒有「新增第一筆／新增到最後」的獨立按鈕。

後端處理概念：

```java
OrderRange orderRange = switch (request.position()) {
    case BEFORE -> findOrderRangeBefore(
        request.referenceTaskId()
    );
    case AFTER -> findOrderRangeAfter(
        request.referenceTaskId()
    );
    case FIRST -> findFirstOrderRange(
        request.projectId(),
        request.parentId()
    );
    case LAST -> findLastOrderRange(
        request.projectId(),
        request.parentId()
    );
};

String newOrder = fractionalIndexGenerator.between(
    orderRange.previousOrder(),
    orderRange.nextOrder()
);
```

## 但拖拉不一定要跟新增使用同一套格式

新增和拖拉的操作本質不完全相同：

- 新增：通常是「相對某筆資料，在上方或下方插入」。
    
- 拖拉：前端完成拖曳後，通常已經知道新的父節點和前後節點。
    

所以可以分開設計：

新增：

```json
{
  "projectId": "PRJ_2026_01",
  "parentId": "STAGE_01",
  "type": "mainTask",
  "referenceTaskId": "TASK_101",
  "position": "AFTER"
}
```

拖拉：

```json
{
  "projectId": "PRJ_2026_01",
  "movingTaskId": "TASK_201",
  "targetParentId": "STAGE_01",
  "prevTaskId": "TASK_101",
  "nextTaskId": "TASK_102"
}
```

所以他那句話翻成白話就是：

> `parentId` 和 `projectId` 的拆分我沒意見。現在我在考慮的是，新增動作本來就是使用者選某一筆，然後決定插在它上面或下面；API 要怎麼傳，才能讓前端和後端都好處理？

你可以直接回他確認：

> 你的意思是，新增時前端只傳基準任務 ID 和 BEFORE／AFTER，由後端依目前排序找出前後節點並計算新的 order，對嗎？

我猜這就是他現在真正想討論的事情。
可能是介面顯示異常，我這邊上一則確實有回答。簡單重貼重點：

你同事現在考慮的不是 `parentId`／`projectId`，而是「新增位置要怎麼傳，前後端比較好處理」。

他的意思應該是：使用者會在某筆資料上選「在上方插入」或「在下方插入」，所以他可能想傳：

```json
{
  "projectId": "PRJ_2026_01",
  "parentId": "STAGE_01",
  "type": "mainTask",
  "referenceTaskId": "TASK_101",
  "position": "AFTER"
}
```

而不是讓前端換算成：

```json
{
  "prevTaskId": "TASK_101",
  "nextTaskId": "TASK_102"
}
```

後端收到 `referenceTaskId + BEFORE/AFTER` 後，再從資料庫查出相鄰節點，計算新的 fractional index。

所以他那句話翻成白話是：

> `projectId` 和 `parentId` 的設計沒問題。現在想討論的是，新增時要讓前端傳「前後兩筆 ID」，還是傳「以哪一筆為基準、插在它上面或下面」，哪種比較好處理？

你可以直接問他確認：

> 你的意思是，新增時前端只傳基準任務 ID 和 BEFORE／AFTER，由後端查出相鄰節點並計算新的 order，對嗎？

我的建議是：

- 新增：用 `referenceTaskId + position（BEFORE/AFTER）`，比較符合畫面操作。
    
- 拖拉：用 `targetParentId + prevTaskId + nextTaskId`，因為拖完後前端本來就知道落點。
    
- 排序字串仍統一由後端產生。

> [!question] 新增子任務怎麼辦?
> 可是，還有一種新增子任務的行為，該如何處理呢?

對，所以不能只設計成「在某一筆的上方／下方新增」，因為「新增子任務」有可能是直接點主任務的「新增子任務」，此時下面可能連一筆子任務都沒有，根本沒有可當基準的 `referenceTaskId`。

比較完整的設計應該是：

- `parentId`：決定新增在哪個父節點底下。
    
- `referenceTaskId`：決定相對哪一筆插入；沒有基準任務時可為 `null`。
    
- `position`：決定插入位置，可用 `BEFORE／AFTER／FIRST／LAST`。
    

例如新增子任務到主任務的最後面：

```json
{
  "projectId": "PRJ_2026_01",
  "parentId": "MAIN_TASK_01",
  "type": "subTask",
  "referenceTaskId": null,
  "position": "LAST"
}
```

這裡的意思是：

> 在 `MAIN_TASK_01` 底下新增一筆子任務，放在現有子任務的最後面。

如果主任務底下沒有任何子任務，後端就產生初始順序：

```java
generateKeyBetween(null, null);
```

如果已經有子任務，後端查詢目前最後一筆：

```sql
SELECT TOP 1 *
FROM UP4_PROJECT_TASK
WHERE PROJECT_ID = @projectId
  AND PARENT_ID = @parentId
  AND TYPE = @subTaskType
ORDER BY SORT_ORDER COLLATE Latin1_General_100_BIN2 DESC;
```

然後產生：

```java
generateKeyBetween(lastTask.getSortOrder(), null);
```

## 在某個子任務上方或下方新增

假設目前是：

```text
主任務 A
├─ 子任務 A1
├─ 子任務 A2
└─ 子任務 A3
```

要在 A2 下方新增：

```json
{
  "projectId": "PRJ_2026_01",
  "parentId": "MAIN_TASK_01",
  "type": "subTask",
  "referenceTaskId": "SUB_TASK_A2",
  "position": "AFTER"
}
```

後端查出 A2 後面的 A3，再計算：

```java
generateKeyBetween(
    subTaskA2.getSortOrder(),
    subTaskA3.getSortOrder()
);
```

要在 A2 上方新增則傳：

```json
{
  "projectId": "PRJ_2026_01",
  "parentId": "MAIN_TASK_01",
  "type": "subTask",
  "referenceTaskId": "SUB_TASK_A2",
  "position": "BEFORE"
}
```

## 建議統一成這組規則

|操作|parentId|referenceTaskId|position|
|---|---|---|---|
|新增第一個階段|`null`|`null`|`LAST`|
|新增階段到最後|`null`|`null`|`LAST`|
|在階段 A 下方新增階段|`null`|階段 A|`AFTER`|
|在階段底下新增主任務|階段 ID|`null`|`LAST`|
|在主任務 A 上方新增主任務|階段 ID|主任務 A|`BEFORE`|
|在主任務底下新增子任務|主任務 ID|`null`|`LAST`|
|在子任務 A 下方新增子任務|主任務 ID|子任務 A|`AFTER`|

這樣既可以支援「點某筆資料，在上方／下方新增」，也可以支援「點主任務的新增子任務按鈕」。

## 後端要驗證的規則

如果有傳 `referenceTaskId`，後端必須檢查：

```text
referenceTask.parentId == request.parentId
referenceTask.type == request.type
referenceTask.projectId == request.projectId
```

也就是基準任務必須是新任務未來的兄弟節點。

例如新增子任務時，不可以這樣：

```json
{
  "parentId": "MAIN_TASK_01",
  "type": "subTask",
  "referenceTaskId": "MAIN_TASK_02",
  "position": "AFTER"
}
```

因為 `MAIN_TASK_02` 是主任務，不是新子任務的兄弟節點。

此外：

- `BEFORE／AFTER`：`referenceTaskId` 必填。
    
- `FIRST／LAST`：`referenceTaskId` 應為 `null`。
    
- `phase`：`parentId` 必須是 `null`。
    
- `mainTask`：`parentId` 必須指向 `phase`。
    
- `subTask`：`parentId` 必須指向 `mainTask`。
    

所以你可以回同事：

> 因為還有「直接在主任務底下新增子任務」的行為，這時不一定有可參考的前後任務。是否改成由 `parentId` 決定新增層級，再以 `referenceTaskId + position` 決定位置？`position` 可包含 `BEFORE、AFTER、FIRST、LAST`；若直接新增子任務則傳主任務 ID，並使用 `LAST`。

這會比單純只有「指定某筆上方／下方」完整。