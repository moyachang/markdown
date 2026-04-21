`ArchUnit` 的核心做法其實很直接：

它**不是在執行時幫你管理架構**，而是把你的 **compiled bytecode / class 結構匯入**，再用測試規則去檢查「哪些 package / class 可以依賴哪些東西」。所以它本質上是 **架構測試（architecture test）**，不是框架。它能檢查 package 依賴、layer 規則、循環依賴等，並且可以用一般的 JUnit 測試來跑。([ArchUnit](https://www.archunit.org/userguide/html/000_Index.html?utm_source=chatgpt.com "ArchUnit User Guide"))

你可以把它理解成：

**「把你嘴巴上說的架構規則，寫成可執行的單元測試。」**

---

## 1. 它是怎麼做到「規範架構」的？

做法通常是這 4 步：

### 第一步：先定義你的架構邊界

例如你專案是：

- `domain..`
    
- `application..`
    
- `adapter..`
    
- `infrastructure..`
    

你先講清楚規則，例如：

- `domain` 不可依賴 `application / adapter / infrastructure`
    
- `application` 不可依賴 `adapter / infrastructure`
    
- `adapter` 不可依賴 `infrastructure`
    
- `infrastructure` 可以依賴 `domain` 與 `application`
    

---

### 第二步：用 ArchUnit 把規則寫成測試

它提供兩種常見方式：

一種是直接寫「package 依賴規則」；  
另一種是用它內建的 **layered architecture** library API 來描述 layer。官方文件明確提到 Library API 有提供像 layered architecture、slices/cycle checks 這類較高階的預設規則。([ArchUnit](https://www.archunit.org/userguide/html/000_Index.html?utm_source=chatgpt.com "ArchUnit User Guide"))

例如：

```java=
package com.flowring.up.arch;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit5.AnalyzeClasses;
import com.tngtech.archunit.junit5.ArchTest;
import com.tngtech.archunit.library.Architectures;

@AnalyzeClasses(
    packages = "com.flowring.up",
    importOptions = ImportOption.DoNotIncludeTests.class
)
public class ArchitectureTest {

    @ArchTest
    static final com.tngtech.archunit.lang.ArchRule layered_architecture_rule =
        Architectures.layeredArchitecture()
            .consideringAllDependencies()
            .layer("Adapter").definedBy("..adapter..")
            .layer("Application").definedBy("..application..")
            .layer("Domain").definedBy("..domain..")
            .layer("Infrastructure").definedBy("..infrastructure..")

            .whereLayer("Adapter").mayNotBeAccessedByAnyLayer()
            .whereLayer("Application").mayOnlyBeAccessedByLayers("Adapter", "Infrastructure")
            .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Infrastructure")
            .whereLayer("Infrastructure").mayNotBeAccessedByAnyLayer();
}
```

這種寫法的意思不是在「教團隊要怎麼做」，而是：

**只要有人寫壞依賴，測試就 fail。**

這就是它真正規範架構的方式。

---

## 2. 為什麼這種方式有效？

因為它把架構從「文件」變成「驗證」。

很多團隊都有架構圖，但很少人會認真執行，而 ArchUnit 的價值在於：

- 架構規則可以版本化
    
- 可放進 CI/CD
    
- 新人不用全靠口頭傳承
    
- refactor 時能立即知道有沒有破壞邊界
    

官方也明確說它的主要目標就是 **自動化測試 architecture and coding rules**，而且可直接整合一般 Java 測試框架。([GitHub](https://github.com/TNG/archUnit?utm_source=chatgpt.com "TNG/ArchUnit: A Java architecture test library, to specify ..."))

---

## 3. 最常見的幾種規範方式

### A. 規範 package 依賴

最基本也最實用。

```java
@ArchTest
static final ArchRule domain_should_not_depend_on_outer_layers =
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..application..", "..adapter..", "..infrastructure..");
```

這條就很適合 Clean Architecture 用法。

---

### B. 規範命名與位置

例如：

- Controller 一定放 `adapter.web`
    
- RepositoryImpl 一定放 `infrastructure.persistence`
    
- UseCase 一定放 `application.usecase`
    

```java
@ArchTest
static final ArchRule controllers_should_reside_in_adapter_web =
    classes()
        .that().haveSimpleNameEndingWith("Controller")
        .should().resideInAPackage("..adapter.web..");
```

這種規則對大型團隊很有用，因為可以避免 class 到處亂放。

---

### C. 規範 annotation 使用

例如：

- `domain` 不准出現 `@Service`
    
- `domain` 不准依賴 Spring
    
- `domain model` 不准標 `@RestController`
    

```java
@ArchTest
static final ArchRule domain_should_not_depend_on_spring =
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("org.springframework..");
```

如果你是走「純 domain」風格，這條很重要。

---

### D. 檢查循環依賴

ArchUnit 也能檢查 slices 間的 cycle。官方文件有明講它可以檢查 cyclic dependencies。([GitHub](https://github.com/TNG/archUnit?utm_source=chatgpt.com "TNG/ArchUnit: A Java architecture test library, to specify ..."))

例如：

```java
@ArchTest
static final ArchRule no_cycles_between_modules =
    com.tngtech.archunit.library.dependencies.SlicesRuleDefinition.slices()
        .matching("com.flowring.up.(*)..")
        .should().beFreeOfCycles();
```

這很適合你如果之後有：

- `up`
    
- `af`
    
- `dms`
    

這種模組化結構。

---

## 4. 以你現在的 Clean Architecture 觀點，最值得先下的規則

你現在最適合先做的，不是一次把所有規則塞滿，而是先抓 **最核心、最不該破的邊界**。

我會建議先從這 5 條開始：

### 規則 1：domain 不得依賴外層

```java
noClasses()
    .that().resideInAPackage("..domain..")
    .should().dependOnClassesThat()
    .resideInAnyPackage("..application..", "..adapter..", "..infrastructure..");
```

### 規則 2：application 不得依賴 adapter

```java
noClasses()
    .that().resideInAPackage("..application..")
    .should().dependOnClassesThat()
    .resideInAnyPackage("..adapter..");
```

### 規則 3：adapter 不直接碰 domain internals 以外不該碰的東西

例如你若規定只能呼叫 application service/use case，就可以限制。

### 規則 4：domain 不得依賴 Spring / JPA

```java
noClasses()
    .that().resideInAPackage("..domain..")
    .should().dependOnClassesThat()
    .resideInAnyPackage("org.springframework..", "jakarta.persistence..");
```

### 規則 5：模組之間不得循環依賴

```java
slices().matching("com.flowring.(*)..").should().beFreeOfCycles();
```

---

## 5. 它的優點是什麼？

### 好處

- **可自動驗證**：不是靠 code review 記憶
    
- **可進 CI**：違反架構直接 fail build
    
- **重構時很有感**：知道哪個規則被破壞
    
- **適合長期演進**：規則可以慢慢加
    

### 真正實務價值

對你這種正在整理：

- Clean Architecture
    
- DDD 分層
    
- package 邊界
    
- repository / adapter / infrastructure 分工
    

的專案，很適合。

因為這類問題，**最怕不是一開始沒設計，而是過幾個月後慢慢被污染**。  
ArchUnit 就是在防這件事。

---

## 6. 它做不到什麼？

這點也很重要。

ArchUnit 主要檢查的是 **靜態依賴關係**。  
所以它擅長：

- import 了誰
    
- package 依賴方向對不對
    
- 類別位置對不對
    
- annotation 用法對不對
    
- 模組有沒有 cycle
    

但它**不會真正理解你的業務語意**，例如：

- 這個 UseCase 的交易邏輯對不對
    
- 這個 Entity 設計是不是好 Aggregate
    
- 某個 mapper 雖然位置正確，但語意是否合理
    

所以它是 **架構守門員**，不是架構設計師。

---

## 7. 一句話總結

`ArchUnit` 能規範架構，是因為它把：

**「架構原則」→「可執行測試」→「違規即失敗」**

這條路打通了。  
也就是說，它不是用文件要求大家守規矩，而是用測試強制驗證規矩。([ArchUnit](https://www.archunit.org/userguide/html/000_Index.html?utm_source=chatgpt.com "ArchUnit User Guide"))

---

## 8. 你現在可以怎麼實行

如果你是 `com.flowring.up` 這種專案，我會建議你先建立一個：

```text
src/test/java/com/flowring/up/arch/ArchitectureTest.java
```

先放這三類：

1. **分層依賴規則**
    
2. **Spring/JPA 不得進 domain**
    
3. **模組不可 cycle**

