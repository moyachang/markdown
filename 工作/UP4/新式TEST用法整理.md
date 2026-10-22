# 原本 test 的寫法較舊

```java
package com.flowring.up.application.service;

import com.flowring.common.domain.model.ExternalUser;
import com.flowring.common.domain.port.repository.ExternalUserRepository;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj
```

這段程式碼並不是「舊」或錯誤，而是採用了**「手動建立與注入（Pure/Manual Mocking）」**的風格。

在現代 Java 專案中，這種風格與主流的 **Mockito 註解風格**相比，確實有一些寫法上的演進與偏好差異。以下為你分析兩種風格的差異與建議改進的地方：

---

### 1. 現代主流：`@ExtendWith(MockitoExtension.class)` 註解風格

現代 JUnit 5 專案中最推薦且最普遍的寫法是使用 Mockito 的註解：

```java
package com.flowring.up.application.service;

import com.flowring.common.domain.model.ExternalUser;
import com.flowring.common.domain.port.repository.ExternalUserRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.LocalDateTime;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.mockito.Mockito.when; // 建議靜態匯入 when

@ExtendWith(MockitoExtension.class) // 1. 讓 Mockito 自動處理生命週期
class GetExternalUserUseCaseTest {

    @Mock
    private ExternalUserRepository repository; // 2. 宣告 Mock 物件

    @InjectMocks
    private GetExternalUserUseCase useCase; // 3. 自動把 Mock 注入進 UseCase

    @Test
    void returnsUserWhenFound() {
        ExternalUser user = new ExternalUser(
                "U1", "部門", "工程師", "123", "zh-TW",
                "C1", "公司", "02-1234", true,
                "王小明", "avatar.png", "0912345678", "user@example.com",
                LocalDateTime.of(2024, 11, 29, 16, 27, 22, 539_000_000),
                null, true
        );
        
        when(repository.findById("U1")).thenReturn(Optional.of(user));

        assertEquals(Optional.of(user), useCase.execute("U1"));
    }

    @Test
    void returnsEmptyWhenNotFound() {
        when(repository.findById("missing")).thenReturn(Optional.empty());

        assertTrue(useCase.execute("missing").isEmpty());
    }
}
````

### 2. 兩種寫法的對比

|**比較項目**|**你目前的寫法 (Manual Mocking)**|**現代註解寫法 (@ExtendWith)**|
|---|---|---|
|**可讀性**|較為冗長，每次都要 `new` 與手動傳參|宣告簡潔，依賴關係一目瞭然|
|**維護性**|若 `UseCase` 的建構子增加新參數，所有測試檔都要手動改 `new`|`@InjectMocks` 會自動匹配新參數，減少修改成本|
|**執行效率**|**極高**（不需載入 Extension 機制）|**極高**（僅比手動慢幾毫秒，幾乎無感）|
|**靜態匯入**|撰寫 `Mockito.when()`|使用 `import static org.mockito.Mockito.when;` 簡化為 `when()`|

### 3. 補充：測試資料過長的問題（Builder / Factory）

除了 Mock 方式之外，你程式碼中的 `new ExternalUser(...)` 帶了 **16 個參數**，這在單元測試中很容易造成閱讀負擔。
**建議改善方式：**
1. **使用 Builder 模式**：如果 Domain Model 有 `@Builder`，只填寫測試關心的欄位即可。
2. **抽離 Test Data Factory**：寫一個輔助方法（例如 `createTestUser()`）來產生預設的假資料，讓測試方法本身保持乾淨。

```java
// 範例：使用測試資料工廠方法
private ExternalUser createDummyUser(String id) {
    return new ExternalUser(
            id, "部門", "工程師", "123", "zh-TW",
            "C1", "公司", "02-1234", true,
            "王小明", "avatar.png", "0912345678", "user@example.com",
            LocalDateTime.now(), null, true
    );
}
```

### 總結

你原本的寫法**邏輯完全正確**，而且執行速度最快。但改用 **`@ExtendWith(MockitoExtension.class)` + `@Mock` + `@InjectMocks`** 會更符合現代 Java 社群的慣用風格，程式碼也會更乾淨！

---

這是幫你重構後的重構版本。使用了現代主流的 **JUnit 5 + Mockito 註解風格**，並增加了 **`@DisplayName`** 提升測試報告的可讀性，同時將預設假資料抽取成輔助方法（Factory Method）：

```java
package com.flowring.up.application.service;

import com.flowring.common.domain.model.ExternalUser;
import com.flowring.common.domain.port.repository.ExternalUserRepository;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.LocalDateTime;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class) // 由 Mockito 管理生命週期
class GetExternalUserUseCaseTest {

    @Mock
    private ExternalUserRepository repository; // 宣告 Mock 物件

    @InjectMocks
    private GetExternalUserUseCase useCase; // 自動將 Mock 注入此 UseCase

    @Test
    @DisplayName("當使用者存在時，應成功回傳 ExternalUser 物件")
    void returnsUserWhenFound() {
        // Arrange
        String userId = "U1";
        ExternalUser mockUser = createDummyUser(userId);
        
        when(repository.findById(userId)).thenReturn(Optional.of(mockUser));

        // Act
        Optional<ExternalUser> result = useCase.execute(userId);

        // Assert
        assertEquals(Optional.of(mockUser), result);
        verify(repository).findById(userId); // 可選：驗證 Repository 方法確實有被呼叫
    }

    @Test
    @DisplayName("當使用者不存在時，應回傳空的 Optional")
    void returnsEmptyWhenNotFound() {
        // Arrange
        String userId = "missing";
        when(repository.findById(userId)).thenReturn(Optional.empty());

        // Act
        Optional<ExternalUser> result = useCase.execute(userId);

        // Assert
        assertTrue(result.isEmpty());
        verify(repository).findById(userId);
    }

    /**
     * 抽取假資料產生的輔助方法，讓測試主體保持乾淨
     */
    private ExternalUser createDummyUser(String id) {
        return new ExternalUser(
                id, "部門", "工程師", "123", "zh-TW",
                "C1", "公司", "02-1234", true,
                "王小明", "avatar.png", "0912345678", "user@example.com",
                LocalDateTime.of(2024, 11, 29, 16, 27, 22, 539_000_000),
                null, true
        );
    }
}
```

### 改進亮點說明

1. **`@ExtendWith(MockitoExtension.class)`**：不再需要手動呼叫 `Mockito.mock(...)` 與 `new GetExternalUserUseCase(...)`。
2. **`@Mock` + `@InjectMocks`**：未來如果 `GetExternalUserUseCase` 構造函數增加了新的依賴（例如注入 Log 或 EventPublisher），測試檔也不需要手動改寫 `new` 的地方。
3. **靜態匯入（Static Imports）**：`when(...)` 與 `verify(...)` 取代了 `Mockito.when(...)`，提升程式碼可讀性。
4. **`@DisplayName(...)`**：取代原本在註解中的文字說明，執行測試時 IDE 的測試面板會顯示漂亮的中文說明。
5. **抽取 `createDummyUser`**：將長達 16 個參數的物件初始化細節移到最下方，讓核心測試方法（`returnsUserWhenFound`）專注在「邏輯與斷言」上。