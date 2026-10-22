# ArchUnit 設置

單元測試驗證「功能」；ArchUnit 驗證「依賴方向」。
即使 API 回傳正確，若 Controller 開始依賴 Service 實作，也應讓建置失敗。

**檔案**：`up/pom.xml`

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.3.0</version>
    <scope>test</scope>
</dependency>
```

- 使用 JUnit 5 時，選用 `archunit-junit5` 這個專用模組。
- `test` scope：不會被包進正式執行檔。
---

## `ArchitectureTest` 逐行解說

**檔案**：`up/src/test/java/com/flowring/up/architecture/ArchitectureTest.java`

```java
@AnalyzeClasses(packages = "com.flowring.up")
class ArchitectureTest {

    @ArchTest
    static final ArchRule layers = layeredArchitecture()
            .consideringOnlyDependenciesInLayers()
            .layer("Adapter").definedBy("..adapter..")
            .layer("Application").definedBy("..application..")
            .layer("Domain").definedBy("..domain..")
            .layer("Infrastructure").definedBy("..infrastructure..")
            .whereLayer("Adapter").mayNotBeAccessedByAnyLayer()
            .whereLayer("Application").mayOnlyBeAccessedByLayers("Adapter", "Infrastructure")
            .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Infrastructure", "Adapter")
            .whereLayer("Infrastructure").mayNotBeAccessedByAnyLayer();

    @ArchTest
    static final ArchRule domainHasNoFramework = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                    "org.springframework..", "jakarta.persistence..", "com.fasterxml..");

    @ArchTest
    static final ArchRule adapterUsesUseCaseNotService = noClasses()
            .that().resideInAPackage("..adapter..")
            .should().dependOnClassesThat().resideInAPackage("..application.service..");
}
```

### 逐項說明

| 語法 | 意思 |
| --- | --- |
| `@AnalyzeClasses(packages = ...)` | 載入 `com.flowring.up` 下的類別供規則檢查。 |
| `@ArchTest static final ArchRule` | 把一條規則登錄成一個 JUnit 測試。 |
| `layer("X").definedBy("..x..")` | 以套件名稱定義一層；`..` 代表任意層級的套件。 |
| `mayNotBeAccessedByAnyLayer()` | 這一層不能被任何層依賴（最外層）。 |
| `mayOnlyBeAccessedByLayers(...)` | 這一層只能被列出的層依賴。 |
| `noClasses().that()...should().dependOnClassesThat()...` | 通用的套件依賴禁令。 |

### 允許方向矩陣

橫列為「依賴者」，直欄為「被依賴者」：

| 依賴者 ＼ 被依賴者 | Adapter | Application | Domain | Infrastructure |
| --- | --- | --- | --- | --- |
| Adapter | — | ✅ | ✅ | ❌ |
| Application | ❌ | — | ✅ | ❌ |
| Domain | ❌ | ❌ | — | ❌ |
| Infrastructure | ❌ | ✅ | ✅ | — |

### 三條規則各擋下什麼

| 規則 | 擋下的問題 | 與 `[04]` 的關係 |
| --- | --- | --- |
| `layers` | 層與層之間出現不允許的依賴 | Service 若 import Web 的 Mapper 或 Response，會失敗。 |
| `domainHasNoFramework` | Domain 引入 Spring、JPA、Jackson | `Up4ProjectRol` 等領域物件保持純 Java。 |
| `adapterUsesUseCaseNotService` | Controller 繞過介面，直接依賴 Service 實作 | Controller 必須注入 `GetProjectMembersUseCase`。 |

### `consideringOnlyDependenciesInLayers()` 的意義

它表示分層規則**只檢查這四層彼此的依賴**，Spring、JDK、`common` 等不屬於任何一層的類別會被忽略，所以才需要第二、三條規則補位。（此為依專案現況與我對 ArchUnit 的理解所作的說明，分享前請對照官方 User Guide 確認。官方使用案例頁的分層範例使用的是 `consideringAllDependencies()`。）
