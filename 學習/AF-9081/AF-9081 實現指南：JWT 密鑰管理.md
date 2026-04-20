# AF-9081 實現指南：JWT 密鑰管理

  

## 概述

  

本文檔提供了解決 AF-9081（硬編碼密鑰問題）的完整實現指南，包括最小改動方案和企業級完整方案。

  

---

  

## 快速開始（最小改動方案）

  

### 步驟 1: 創建配置文件

  

在 `src/resources/` 目錄下創建 `jwt-secret.properties`：

  

```properties

# jwt-secret.properties

jwt.secret.base=SuperSecureRandomKey123!@#$%^&*()_+-=[]{}

jwt.algorithm=HS256

jwt.expiration.days=365

jwt.issuer=Flowring

```

  

### 步驟 2: 修改 Util 類

  

在 `com.flowring.servlet.api.auth.Util` 類中，修改 `getSecret()` 方法：

  

**原有代碼**:

```java

public static String getSecret(String productUserName) {

    String secret = "cj86ux/6nji3vu;4j62u6"; // ? 硬編碼

    secret += productUserName;

    return secret;

}

```

  

**改進方案**:

```java

// 使用 Spring 注入配置

@Autowired

private JwtSecretConfig jwtSecretConfig;

  

public static String getSecret(String productUserName) {

    // 從配置文件讀取密鑰

    String secret = jwtSecretConfig.getSecret(productUserName);

    return secret;

}

```

  

### 步驟 3: 實現 LaleTokenUtils.getDecodedJWTInToken()

  

**原有代碼**:

```java

public static DecodedJWT getDecodedJWTInToken(String token) {

    return null; // AF-9081 Use of Hard-coded Cryptographic Key

}

```

  

**完整實現**:

```java

public static DecodedJWT getDecodedJWTInToken(String token) {

    try {

        String companyName = WebSystem.getCorp();

        String secret = Util.getSecret(companyName);

        if (secret == null || secret.isEmpty()) {

            logger.error("Failed to obtain JWT secret for company: " + companyName);

            return null;

        }

        Algorithm algorithm = Algorithm.HMAC256(secret);

        JWTVerifier verifier = JWT.require(algorithm).build();

        return verifier.verify(token);

    } catch (JWTVerificationException e) {

        logger.debug("JWT token verification failed: " + e.getMessage());

        return null;

    } catch (IllegalArgumentException e) {

        logger.error("Invalid algorithm or secret: " + e.getMessage());

        return null;

    } catch (Exception e) {

        logger.error("Unexpected error while decoding JWT token", e);

        return null;

    }

}

```

  

---

  

## 完整實現方案（推薦）

  

### 架構設計

  

```

LaleTokenUtils

    │

    └─> Util.getSecret()

            │

            └─> SecretProviderManager

                    │

                    ├─> EnvironmentVariableSecretProvider (優先級 1)

                    ├─> ConfigFileSecretProvider (優先級 2)

                    └─> CustomSecretProvider (優先級 3，可選)

```

  

### 方案優勢

  

| 特性 | 優勢 |

|------|------|

| **配置驅動** | 無需重編譯即可更改密鑰 |

| **多源支持** | 支持多種密鑰來源 |

| **優先級鏈** | 靈活的密鑰獲取策略 |

| **易於擴展** | 可添加自定義密鑰提供者 |

| **環境隔離** | 開發/測試/生產環境密鑰分離 |

  

---

  

## 詳細實現步驟

  

### 第 1 步：創建配置類

  

**文件**: `com.flowring.servlet.api.auth.config.JwtSecretConfig`

  

**職責**:

- 從 properties 文件讀取配置

- 驗證密鑰強度

- 提供 getSecret() 方法

  

**關鍵代碼**:

- 使用 `@Value` 注入配置值

- 實現 `getSecret(String companyName)` 方法

- 添加密鑰強度驗證邏輯

  

**配置項**:

- `jwt.secret.base` - 密鑰基礎部分

- `jwt.algorithm` - 算法選擇（默認 HS256）

- `jwt.expiration.days` - Token 過期天數

- `jwt.issuer` - Token 發行者

  

### 第 2 步：創建密鑰提供者接口

  

**文件**: `com.flowring.servlet.api.auth.provider.SecretProvider`

  

**方法**:

- `getSecret(String companyName)` - 獲取密鑰

- `isAvailable()` - 檢查提供者是否可用

- `getProviderName()` - 獲取提供者名稱

  

**異常處理**:

- 定義 `SecretProviderException` 異常類

  

### 第 3 步：實現環境變量提供者

  

**文件**: `com.flowring.servlet.api.auth.provider.EnvironmentVariableSecretProvider`

  

**優先級**: 1（最高）

  

**工作方式**:

- 讀取環境變量 `JWT_SECRET_BASE`

- 若不存在則拋異常或返回備選值

- 結合公司名稱組成完整密鑰

  

**適用場景**:

- Docker 容器部署

- Kubernetes 環境

- CI/CD 流程

  

### 第 4 步：實現配置文件提供者

  

**文件**: `com.flowring.servlet.api.auth.provider.ConfigFileSecretProvider`

  

**優先級**: 2（次高）

  

**工作方式**:

- 使用 JwtSecretConfig 讀取配置

- 提供備選密鑰管理方案

- 開發環境首選方案

  

### 第 5 步：創建密鑰提供者管理器

  

**文件**: `com.flowring.servlet.api.auth.provider.SecretProviderManager`

  

**職責**:

- 管理多個密鑰提供者

- 按優先級順序嘗試獲取密鑰

- 日誌記錄和診斷

  

**優先級順序**:

1. 環境變量提供者

2. 配置文件提供者

3. 自定義提供者（如果存在）

  

### 第 6 步：更新 LaleTokenUtils

  

**改進點**:

- 移除返回 null 的實現

- 添加完整的異常處理

- 使用配置密鑰而非硬編碼

- 添加詳細的日誌記錄

  

---

  

## 密鑰強度要求

  

### 最小要求

- ? 長度: 32 字符以上

- ? 大寫字母: 至少 1 個

- ? 小寫字母: 至少 1 個

- ? 數字: 至少 1 個

- ? 特殊字符: 至少 1 個

  

### 推薦配置

```properties

# 示例密鑰（32 字符）

jwt.secret.base=K7$mP2@xQ8!vL3&hR5%sT9^jF1&mN0_b

```

  

### 密鑰檢驗清單

- [ ] 長度 ? 32

- [ ] 包含大小寫字母

- [ ] 包含數字

- [ ] 包含特殊字符

- [ ] 不包含易預測的模式

- [ ] 不包含常見密碼詞彙

  

---

  

## 環境配置管理

  

### 開發環境配置

  

**文件**: `application-dev.properties`

  

```properties

spring.profiles.active=dev

jwt.secret.base=dev-key-only-for-testing-12345678

```

  

**特點**:

- 簡單、容易記憶的密鑰

- 頻繁修改密鑰進行測試

- 日誌級別設為 DEBUG

  

### 測試環境配置

  

**文件**: `application-test.properties`

  

```properties

spring.profiles.active=test

jwt.secret.base=test-secret-key-must-be-different-12345

```

  

**特點**:

- 與開發不同的密鑰

- 定期更新（每月）

- 日誌級別設為 INFO

  

### 生產環境配置

  

**特點**:

- 密鑰通過環境變量注入

- 不存儲在配置文件中

- 由運維人員管理

- 強密鑰強度要求

  

**配置方式**:

```bash

# 方式 1：系統環境變量

export JWT_SECRET_BASE="ProductionSecureKey123!@#$%^&*()"

java -jar application.jar --spring.profiles.active=prod

  

# 方式 2：Java 命令行參數

java -DJWT_SECRET_BASE="ProductionSecureKey123!@#$%^&*()" \

     -jar application.jar --spring.profiles.active=prod

  

# 方式 3：Docker 環境變量

docker run -e JWT_SECRET_BASE="..." webagenda:latest

  

# 方式 4：Kubernetes Secret

kubectl create secret generic jwt-secret --from-literal=base="..."

```

  

---

  

## 密鑰輪換計畫

  

### 輪換周期

  

| 環境 | 周期 | 說明 |

|------|------|------|

| 開發 | 隨意 | 用於測試，可頻繁更改 |

| 測試 | 每季度 | 定期驗證輪換流程 |

| 生產 | 每年 | 按安全政策執行 |

  

### 輪換步驟

  

**步驟 1**: 生成新密鑰

```bash

openssl rand -base64 32

```

  

**步驟 2**: 配置系統同時支持舊密鑰和新密鑰（過渡期）

  

**步驟 3**: 新簽發的 Token 使用新密鑰

  

**步驟 4**: 過渡期內（如 1-2 月）系統仍驗證舊密鑰 Token

  

**步驟 5**: 過渡期結束後停止接受舊密鑰簽發的 Token

  

### 實現示例

  

```java

// 支持多個密鑰驗證（過渡期）

public List<String> getSecrets(String companyName) {

    List<String> secrets = new ArrayList<>();

    // 當前密鑰（簽發新 Token 使用）

    secrets.add(getCurrentSecret(companyName));

    // 舊密鑰（過渡期驗證使用）

    if (isInTransitionPeriod()) {

        secrets.add(getLegacySecret(companyName));

    }

    return secrets;

}

  

// 使用多個密鑰進行驗證

public DecodedJWT validateWithMultipleSecrets(String token, List<String> secrets) {

    for (String secret : secrets) {

        try {

            Algorithm algorithm = Algorithm.HMAC256(secret);

            JWTVerifier verifier = JWT.require(algorithm).build();

            return verifier.verify(token);

        } catch (JWTVerificationException e) {

            // 繼續嘗試下一個密鑰

        }

    }

    return null; // 所有密鑰都驗證失敗

}

```

  

---

  

## 故障排除指南

  

### 問題 1: JWT 驗證失敗 - Invalid signature

  

**症狀**: 拋出 `SignatureVerificationException`

  

**可能原因**:

1. 生成 Token 和驗證 Token 使用的密鑰不一致

2. 密鑰在傳輸過程中被篡改

3. 算法版本不匹配

  

**排除步驟**:

1. 確認密鑰值在生成和驗證時完全相同

2. 檢查是否有多餘空格或編碼問題

3. 驗證 java-jwt 庫版本

4. 檢查是否使用了正確的算法（HS256）

  

### 問題 2: Token 過期異常

  

**症狀**: 拋出 `TokenExpiredException`

  

**可能原因**:

1. Token 已超出有效期

2. 服務器時鐘不同步

3. Token 創建時間設置錯誤

  

**排除步驟**:

1. 檢查服務器系統時間是否準確

2. 檢查 Token 過期設置（expiration.days）

3. 考慮使用 NTP 同步時鐘

  

### 問題 3: 無法讀取配置文件

  

**症狀**: `NullPointerException` 或 `FileNotFoundException`

  

**可能原因**:

1. 配置文件未在正確路徑

2. 未添加 @PropertySource 注解

3. Maven 構建未包含資源文件

  

**排除步驟**:

1. 確認 `jwt-secret.properties` 在 `src/resources/` 下

2. 檢查 `@PropertySource` 路徑是否正確

3. 清理並重新構建項目 `mvn clean install`

4. 檢查 pom.xml 的資源配置

  

### 問題 4: 環境變量未被讀取

  

**症狀**: 拋出 `SecretProviderException: JWT_SECRET_BASE not set`

  

**可能原因**:

1. 環境變量未設置

2. 應用未重啟加載新變量

3. 容器環境變量配置錯誤

  

**排除步驟**:

```bash

# 驗證環境變量

echo $JWT_SECRET_BASE

  

# 設置環境變量

export JWT_SECRET_BASE="your-secret-key"

  

# 驗證設置成功

echo $JWT_SECRET_BASE

  

# 重啟應用

```

  

---

  

## 測試建議

  

### 單元測試

  

**測試場景 1**: 有效 Token 驗證

```

1. 使用已知密鑰生成 Token

2. 使用相同密鑰驗證 Token

3. 驗證返回非 null 的 DecodedJWT

```

  

**測試場景 2**: 無效簽名

```

1. 使用密鑰 A 生成 Token

2. 使用密鑰 B 驗證 Token

3. 驗證返回 null（或拋異常）

```

  

**測試場景 3**: 過期 Token

```

1. 生成過期時間為當前時間的 Token

2. 驗證 Token

3. 驗證捕獲 TokenExpiredException

```

  

**測試場景 4**: 異常處理

```

1. 傳入 null Token

2. 傳入空字符串 Token

3. 傳入無效格式 Token

4. 驗證都返回 null 而不是拋異常

```

  

### 集成測試

  

**測試場景 1**: 配置文件加載

```

1. 啟動應用（開發環境）

2. 驗證配置已正確加載

3. 測試 Token 生成和驗證

```

  

**測試場景 2**: 環境變量優先級

```

1. 設置環境變量 JWT_SECRET_BASE

2. 同時存在配置文件

3. 驗證環境變量優先級更高

```

  

**測試場景 3**: 多環境配置

```

1. 測試開發環境配置

2. 測試測試環境配置

3. 測試生產環境配置

```

  

---

  

## 性能考慮

  

### 密鑰緩存

  

**是否應該緩存密鑰？**

  

**建議**: 是的，但需謹慎

  

**緩存方案**:

```java

private static final Map<String, String> secretCache = new ConcurrentHashMap<>();

  

public String getSecret(String companyName) {

    return secretCache.computeIfAbsent(companyName, key -> {

        // 從提供者獲取密鑰

        return secretProviderManager.getSecret(key);

    });

}

```

  

**清理緩存**:

- 密鑰輪換時主動清理

- 定期自動刷新（如每小時）

- 異常情況下清理並重新加載

  

### 並發安全

  

**建議使用**: `ConcurrentHashMap`

  

**原因**:

- JWTVerifier 是線程安全的

- 密鑰獲取可能存在I/O操作

- 避免鎖競爭

  

---

  

## 監控和告警

  

### 關鍵指標

  

| 指標 | 告警閾值 | 說明 |

|------|----------|------|

| JWT 驗證失敗率 | > 1% | 異常高的失敗率 |

| 密鑰獲取失敗 | > 0 | 任何失敗都應告警 |

| Token 過期異常 | 基線 +200% | 異常的過期情況 |

  

### 日誌監控

  

```java

// 成功驗證 - INFO 級別

logger.info("JWT token verified successfully for company: {}", companyName);

  

// 驗證失敗 - WARN 級別

logger.warn("JWT token verification failed: {}", exception.getMessage());

  

// 配置錯誤 - ERROR 級別

logger.error("Failed to obtain JWT secret: {}", exception.getMessage());

```

  

### 推薦工具

  

- **ELK Stack** (Elasticsearch, Logstash, Kibana)

- **Splunk** - 日誌分析

- **Prometheus** + **Grafana** - 指標監控

- **Datadog** - APM 和監控

  

---

  

## 合規性檢查

  

### OWASP 標準

  

- ? **CWE-798**: 不使用硬編碼的密鑰

- ? **CWE-321**: 使用適當的密鑰長度

- ? **CWE-760**: 使用標準加密算法（HMAC256）

  

### 安全最佳實踐

  

- ? 密鑰從外部配置獲取

- ? 不在日誌中輸出密鑰

- ? 支持密鑰輪換

- ? 異常安全處理

- ? 適當的訪問控制

  

### 審計要求

  

- 記錄所有密鑰訪問

- 追蹤 Token 的生成和驗證

- 定期審計日誌

- 保存審計日誌最少 1 年

  

---

  

## 參考資源

  

- [Auth0 Java-JWT GitHub](https://github.com/auth0/java-jwt)

- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)

- [CWE-798: Use of Hard-coded Credentials](https://cwe.mitre.org/data/definitions/798.html)

- [Spring Boot Properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html)

- [JWT.io](https://jwt.io/)