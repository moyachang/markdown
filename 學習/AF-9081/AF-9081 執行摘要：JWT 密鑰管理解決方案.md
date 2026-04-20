# AF-9081 執行摘要：JWT 密鑰管理解決方案

  

## ? 概述

  

AF-9081 是針對 **JWT 硬編碼密鑰漏洞** 的修復。本文檔提供快速參考和實施檢查清單。

  

---

  

## ? 當前問題

  

### 代碼位置

- **文件**: `com.flowring.servlet.api.auth.LaleTokenUtils.java`

- **方法**: `getDecodedJWTInToken()` (第 24 行)

- **狀態**: 返回 `null`（未實現）

  

### 根源問題

- **文件**: `com.flowring.servlet.api.auth.Util.java`

- **方法**: `getSecret()` (第 27-28 行)

- **問題**: 硬編碼密鑰 `"cj86ux/6nji3vu;4j62u6"`

  

### 安全風險

1. 密鑰暴露在源代碼中

2. 無法輪換密鑰（需要重新編譯）

3. 多環境使用同一密鑰

4. 違反 OWASP CWE-798 標準

  

---

  

## ? 解決方案概要

  

### 三層方案對比

  

| 方案 | 難度 | 安全性 | 推薦度 | 時間 |

|------|------|--------|--------|------|

| **方案 1: 配置文件** | 低 | 中 | ???? | 1 天 |

| **方案 2: 環境變量** | 低 | 高 | ???? | 1-2 天 |

| **方案 3: 密鑰服務** | 高 | 很高 | ??? | 3-5 天 |

  

### 推薦實施順序

1. **第 1-2 週**: 實施方案 1（配置文件）

2. **第 2-3 週**: 補充方案 2（環境變量）

3. **第 3-4 週**: 生產部署和監控

  

---

  

## ? 快速實現（3 步）

  

### 步驟 1: 創建配置文件

  

**位置**: `src/resources/jwt-secret.properties`

  

```properties

jwt.secret.base=SuperSecureRandomKey123!@#$%^&*()_+-=[]{}

jwt.algorithm=HS256

jwt.expiration.days=365

jwt.issuer=Flowring

```

  

### 步驟 2: 修改 Util.getSecret()

  

**改變方式**: 從硬編碼改為讀取配置

  

```java

// 原有（?）

String secret = "cj86ux/6nji3vu;4j62u6";

  

// 改進（?）

String secret = jwtSecretConfig.getSecret(productUserName);

```

  

### 步驟 3: 實現 getDecodedJWTInToken()

  

**改變方式**: 從返回 null 改為完整實現

  

```java

// 原有（?）

return null;

  

// 改進（?）

public static DecodedJWT getDecodedJWTInToken(String token) {

    try {

        String companyName = WebSystem.getCorp();

        String secret = Util.getSecret(companyName);

        Algorithm algorithm = Algorithm.HMAC256(secret);

        JWTVerifier verifier = JWT.require(algorithm).build();

        return verifier.verify(token);

    } catch (JWTVerificationException e) {

        logger.debug("JWT verification failed", e);

        return null;

    }

}

```

  

---

  

## ? 密鑰要求

  

### 強度標準

- ? 長度: **32+ 字符**

- ? 包含: 大小寫字母、數字、特殊字符

- ? 唯一性: 每個環境不同

- ? 隨機性: 使用密碼學級隨機數

  

### 生成命令

```bash

# Linux/Mac - 推薦

openssl rand -base64 32

  

# Python

python3 -c "import secrets; print(''.join(secrets.choice('ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()_+-=[]{}') for _ in range(32)))"

```

  

### 密鑰示例

```

K7$mP2@xQ8!vL3&hR5%sT9^jF1&mN0_b  ? 好

dev-secret-key-for-testing-only  ? 缺少特殊字符

cj86ux/6nji3vu;4j62u6            ? 太短且硬編碼

```

  

---

  

## ? 部署配置

  

### 開發環境

  

**application-dev.properties**:

```properties

spring.profiles.active=dev

jwt.secret.base=dev-secret-key-for-testing-12345

```

  

### 生產環境

  

**使用環境變量** (推薦):

```bash

export JWT_SECRET_BASE="your-production-secret-key"

java -jar application.jar --spring.profiles.active=prod

```

  

**Docker**:

```bash

docker run \

  -e JWT_SECRET_BASE="your-production-key" \

  -p 8080:8080 \

  webagenda:latest

```

  

**Kubernetes**:

```yaml

apiVersion: v1

kind: Secret

metadata:

  name: jwt-secret

stringData:

  jwt-secret-base: "your-production-key"

---

env:

- name: JWT_SECRET_BASE

  valueFrom:

    secretKeyRef:

      name: jwt-secret

      key: jwt-secret-base

```

  

---

  

## ? 密鑰輪換計畫

  

### 輪換周期

| 環境 | 周期 | 觸發事件 |

|------|------|---------|

| 開發 | 隨意 | 測試時更改 |

| 測試 | 每季度 | 季度審計 |

| 生產 | 每年 | 年度安全檢查 |

  

### 輪換步驟

1. 生成新密鑰 (`openssl rand -base64 32`)

2. 配置系統同時支持舊密鑰和新密鑰

3. 新 Token 使用新密鑰簽發

4. 過渡期（1-2 月）驗證接受兩個密鑰

5. 停止接受舊密鑰簽發的 Token

  

---

  

## ? 驗收標準

  

### 代碼檢查

- [ ] 無硬編碼密鑰字符串

- [ ] `getDecodedJWTInToken()` 完整實現

- [ ] 異常處理正確

- [ ] 日誌記錄適當

  

### 配置檢查

- [ ] 配置文件已創建

- [ ] `.gitignore` 包含 `jwt-secret.properties`

- [ ] 環境特定配置已分離

- [ ] 環境變量支持已實現

  

### 測試檢查

- [ ] 有效 Token 驗證成功

- [ ] 無效 Token 驗證失敗

- [ ] 過期 Token 捕獲異常

- [ ] 密鑰更改後新 Token 驗證成功

  

### 部署檢查

- [ ] 開發環境部署通過

- [ ] 測試環境部署通過

- [ ] 生產環境環境變量配置

- [ ] 監控和告警配置

  

---

  

## ? 實施時間表

  

### 第 1 週：快速修復

- **Day 1-2**: 創建配置文件和類

- **Day 3-4**: 修改 Util 和 LaleTokenUtils

- **Day 5**: 開發環境測試

- **交付物**: 能正常驗證 Token

  

### 第 2 週：環境適配

- **Day 1-2**: 為各環境創建配置

- **Day 3-4**: 配置 Spring Profiles

- **Day 5**: 集成和測試環境測試

- **交付物**: 多環境支持

  

### 第 3 週：生產部署

- **Day 1-2**: 生成生產密鑰

- **Day 3-4**: Docker/K8s 配置

- **Day 5**: 灰度部署和監控

- **交付物**: 生產環境上線

  

---

  

## ? 常見問題

  

### Q1: JWT 驗證失敗

**原因**: 密鑰不匹配

**解決**: 檢查簽發和驗證使用相同密鑰

  

### Q2: 配置文件找不到

**原因**: 路徑錯誤

**解決**: 確保在 `src/resources/` 目錄下

  

### Q3: 環境變量未讀取

**原因**: 未設置或應用未重啟

**解決**: 設置變量並重啟應用

  

### Q4: 過期 Token 異常

**原因**: 服務器時鐘不同步

**解決**: 同步服務器時鐘，使用 NTP

  

---

  

## ? 參考文檔

  

### 詳細指南（按順序閱讀）

1. **JWT_KEY_MANAGEMENT_BEST_PRACTICES.md** ?

   - 三個方案的完整說明

   - 最佳實踐和安全檢查清單

  

2. **IMPLEMENTATION_GUIDE_AF9081.md** ?

   - 詳細實現步驟

   - 故障排除指南

   - 測試建議

  

3. **JWT_CONFIGURATION_EXAMPLES.md**

   - 具體配置示例

   - Docker/K8s 部署命令

  

### 官方資源

- [Auth0 Java-JWT](https://github.com/auth0/java-jwt)

- [OWASP 密鑰管理](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)

- [CWE-798 標準](https://cwe.mitre.org/data/definitions/798.html)

  

---

  

## ? .gitignore 更新

  

```gitignore

# JWT 密鑰相關（不提交到版本控制）

**/jwt-secret.properties

**/jwt-secret-*.properties

**/.env

**/.env.local

**/secrets/

```

  

---

  

## ? 成功指標

  

? **完成後應有**:

- 無硬編碼密鑰

- 支持多環境密鑰

- Token 驗證正常運作

- 支持密鑰輪換

- 完整的日誌記錄

- 符合 OWASP 標準

  

---

  

## ? 技術支持

  

| 文檔 | 適用場景 |

|------|----------|

| JWT_KEY_MANAGEMENT_BEST_PRACTICES.md | 了解方案和最佳實踐 |

| IMPLEMENTATION_GUIDE_AF9081.md | 詳細的實現步驟和測試 |

| JWT_CONFIGURATION_EXAMPLES.md | 環境配置和部署命令 |

| 此文檔 | 快速參考和檢查清單 |

  

---

  

**版本**: 1.0  

**最後更新**: 2026-04-20  

**狀態**: ? 就緒實施