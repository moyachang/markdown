# Java `Optional.of` 的用法

## 基本概念

`Optional.of(value)` 會把一個**一定不是 null** 的值包進 `Optional` 容器。如果你傳入 `null`,會直接丟出 `NullPointerException`。

```java
Optional<String> name = Optional.of("Alice");   // 正常
Optional<String> bad  = Optional.of(null);      // 拋出 NullPointerException
```

## 三種建立方式的差別

|方法|傳入 null 時|使用時機|
|---|---|---|
|`Optional.of(x)`|拋 NPE|你**確定**值不是 null|
|`Optional.ofNullable(x)`|回傳 `Optional.empty()`|值**可能是** null|
|`Optional.empty()`|(不需要傳值)|明確表示「沒有值」|

```java
String a = "hello";
String b = null;

Optional.of(a);            // Optional[hello]
Optional.ofNullable(b);    // Optional.empty
Optional.empty();          // Optional.empty
```

## 什麼時候用 `of`?

`of` 的好處是 **fail-fast**:如果你認為值不該是 null,卻真的是 null,馬上就能發現 bug,而不是讓 null 默默往下傳。

```java
// 適合:回傳值是你自己建構的,一定有值
public Optional<User> findAdmin() {
    User admin = new User("admin");
    return Optional.of(admin);
}

// 適合:有值就回傳,沒有就回傳 empty
public Optional<User> findById(int id) {
    User user = map.get(id);          // 可能是 null
    return Optional.ofNullable(user); // 這裡要用 ofNullable
}
```

**經驗法則**:來源可能為 null(例如 Map.get、資料庫查詢、外部輸入)用 `ofNullable`;你自己保證有值才用 `of`。

## 建立之後怎麼取值

```java
Optional<String> opt = Optional.of("Alice");

// 1. 判斷是否有值
opt.isPresent();      // true
opt.isEmpty();        // false (Java 11+)

// 2. 有值才執行
opt.ifPresent(n -> System.out.println(n));

// 3. 取值並提供預設值(最常用)
String n1 = opt.orElse("Unknown");
String n2 = opt.orElseGet(() -> computeDefault());   // 預設值運算較昂貴時用
String n3 = opt.orElseThrow(() -> new IllegalStateException("沒有值"));

// 4. 轉換
Optional<Integer> len = opt.map(String::length);

// 5. 過濾
Optional<String> longName = opt.filter(s -> s.length() > 3);
```

## 常見錯誤用法

```java
// ❌ 直接 get(),沒值時會拋 NoSuchElementException
String s = opt.get();

// ❌ 把 Optional 當成一般的 null 檢查,失去意義
if (opt.isPresent()) { return opt.get(); } else { return "x"; }
// ✅ 改成
return opt.orElse("x");

// ❌ 拿 Optional 當作方法參數或類別欄位(不建議)
public void foo(Optional<String> name) { ... }
```

Optional 設計上主要是用來當**方法的回傳型別**,表達「這個方法可能沒有結果」。

## 完整範例

```java
public class Demo {
    static Map<Integer, String> users = Map.of(1, "Alice", 2, "Bob");

    static Optional<String> findName(int id) {
        return Optional.ofNullable(users.get(id));
    }

    public static void main(String[] args) {
        String result = findName(3)
                .map(String::toUpperCase)
                .orElse("查無此人");
        System.out.println(result);   // 查無此人

        findName(1).ifPresent(System.out::println);  // Alice
    }
}
```


20261012 usecase 何時該回傳 Optional??
## 核心原則

`Optional` 的設計目的很窄：Java 設計者 Brian Goetz 說，它是給方法回傳型別用的有限機制，用來清楚表達「沒有結果」的情況。所以 UseCase 的 `execute` 該不該回傳 `Optional`，只看一件事：**「查不到」是不是這個業務情境下的正常結果。**

## 判斷表

|情況|回傳型別|例子|
|---|---|---|
|查不到是**正常**結果|`Optional<T>`|依 ID 查單一資料|
|一定要有，查不到代表**錯誤**|`T`，找不到就丟例外|修改一筆必須存在的資料|
|回傳多筆|`List<T>`，沒有就回傳空 List|取得專案成員清單|
|永遠會產生結果|`T`|組裝出來的畫面資料|

## 幾個要避開的用法

- **集合不要包 Optional**：Goetz 提到，回傳陣列或 List 時不該用它，應回傳空陣列或空 List。`Optional<List<T>>` 或 `List<Optional<T>>` 這類複雜型別，也被視為程式碼壞味道。
- **不要當參數或欄位**：一般建議不要把 `Optional` 用在建構子或方法參數，也不要宣告成實例變數。
- **不要為了鏈式呼叫而硬包**：只為了串接方法來取得值而建立 `Optional`，通常不是好主意。
- **不要回傳 `null` 的 Optional**：回傳了 `Optional` 就要保證它本身不是 `null`。

## 套用到你的專案

|UseCase|現在回傳|判斷|
|---|---|---|
|`GetExternalUserUseCase`|`Optional<ExternalUser>`|✅ 查無是正常情況|
|`GetManualPartnerUserUseCase`|`Optional<Up4ProjectManualMem>`|✅ 同上|
|`GetProjectMembersUseCase`|`List<ProjectMember>`|✅ 查無時回傳空 List|
|`GetProjectRoleTreeUseCase`|`List<RoleTreeResult>`|✅ 同上|
|`GetMemberProfileUseCase`|`MemberProfileResult`|⚠️ 見下方說明|

`GetMemberProfileUseCase` 值得再想一下：它查不到人員時，並不會告訴呼叫端「查無此人」，而是回傳一包空字串的結果（`name`、`email` 都是 `""`）。這等於把「查無資料」偽裝成「有資料」，前端無法分辨。比較合理的選擇有兩種：

1. 查無就回 `Optional.empty()`，Controller 決定回傳 `null` data 或 404。
2. 查無就丟例外，由全域例外處理器轉成 404。

選哪一種取決於前端目前如何處理這支 API，所以我沒有擅自改動。

## 進階：誰來決定「查不到」要回什麼？

這一段是我依一般實務整理的，沒有對應的檢索來源。

`Optional.empty()` 只表示「沒有」，**怎麼回應是 Controller 的事**。你現在的做法是 `.orElse(null)`，查不到就回 `200 + data: null`。如果前端需要區分「成功但沒資料」與「資源不存在」，可以改成 `.orElseThrow(() -> new NotFoundException(...))`，讓 HTTP 狀態碼回 404。兩種都可以，重點是整個專案要一致，並在 API 文件中寫明。

## 參考來源

- Optional 設計意圖與使用準則（整理 Brian Goetz 的說明）：https://gist.github.com/MikeChristianson/f5c12d9caa682b2ba7db4659997f6768
- Optional 使用規則（參數、欄位、回傳型別、複雜型別）：https://gist.github.com/vple/bce11c44059efd4efd5455c7e5bc62d2

如果你想處理 `GetMemberProfileUseCase` 查不到人員的情況，告訴我你偏好哪一種做法，我再幫你改。
