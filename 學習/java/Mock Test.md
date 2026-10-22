# 程式解說

這是一個用 **JUnit 5 + Mockito** 寫的單元測試,測試的是一個 UseCase 類別(`useCase.execute`),它內部會呼叫 `repository.findById` 去查使用者。

## 測試的目的

驗證「當 Repository 找得到使用者時,UseCase 會把那個使用者包在 `Optional` 裡回傳」。

## 逐行解釋 

### 註解

```java
@Test
@DisplayName("當使用者存在時，應成功回傳 ExternalUser 物件")
```

- `@Test`:標示這是一個測試方法,JUnit 會自動執行。
- `@DisplayName`:測試報告中顯示的名稱,可以用中文描述,比方法名更易讀。

### Arrange(準備)

```java
String userId = "U1";
ExternalUser mockUser = createDummyUser(userId);
```

準備測試資料:一個假的 userId,以及一個手動建立的假使用者物件(`createDummyUser` 是測試類別裡自己寫的輔助方法)。

```java
when(repository.findById(userId)).thenReturn(Optional.of(mockUser));
```

這行是 Mockito 的 **stubbing(設定假行為)**,意思是:

> 「當有人用 `"U1"` 呼叫 `repository.findById` 時,就回傳 `Optional.of(mockUser)`。」

這裡正好用到你剛剛問的 `Optional.of`:因為 `mockUser` 是我們自己建立的,確定不是 null,所以用 `of` 是合適的。

此處的 `repository` 應該是用 `@Mock` 宣告的假物件,不會真的連資料庫。

### Act(執行)

```java
Optional<ExternalUser> result = useCase.execute(userId);
```

真正呼叫被測試的程式。UseCase 內部會去呼叫 `repository.findById("U1")`,拿到上面設定好的假結果。

### Assert(驗證)

```java
assertEquals(Optional.of(mockUser), result);
```

檢查 `result` 是否等於 `Optional.of(mockUser)`。`Optional` 的 `equals` 會比較內部的值,所以只要裡面是同一個(或 equals 相等的)`mockUser` 就會通過。

```java
verify(repository).findById(userId);
```

驗證 `repository.findById("U1")` **確實被呼叫過一次**。這能確保 UseCase 真的有去查 Repository,而不是憑空回傳結果。

## 整體結構:AAA 模式

|階段|做什麼|
|---|---|
|Arrange|準備資料、設定 mock 行為|
|Act|執行被測試的方法|
|Assert|驗證結果與互動|

## 補充小提醒

1. **`assertEquals(Optional.of(...), result)` 可以寫得更直觀**:
    
    ```java
    assertTrue(result.isPresent());
    assertEquals(mockUser, result.get());
    ```
    
    或用 AssertJ:`assertThat(result).contains(mockUser);`,失敗時的訊息也比較清楚。
    
2. **通常還會搭配「找不到」的測試**:
    
    ```java
    when(repository.findById("U2")).thenReturn(Optional.empty());
    assertTrue(useCase.execute("U2").isEmpty());
    ```
    
3. 這個 UseCase 看起來只是單純轉手 Repository 的結果,所以測試很簡單;如果 `execute` 內有額外邏輯(轉換、驗證等),測試才會更有價值。
    

如果你貼出 `useCase` 和 `repository` 的類別宣告(含 `@Mock`、`@InjectMocks`),我可以幫你把整個測試類別一起解釋。