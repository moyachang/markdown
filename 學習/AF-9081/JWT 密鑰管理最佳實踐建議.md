# JWT 密鑰管理最佳實踐建議

  

## 當前問題分析

  

根據代碼分析，AF-9081 這個 issue 主要關注 JWT 的硬編碼密鑰問題：

  

### 現狀

1. **LaleTokenUtils.getDecodedJWTInToken()** - 目前返回 `null`（無實現）

2. **Util.getSecret()** 方法中存在硬編碼密鑰問題：

   ```java

   String secret = "cj86ux/6nji3vu;4j62u6";

   secret += productUserName;

   ```

3. **相關調用鏈**：

   - `LaleAuthServlet.getDecodedJWTInToken()` 先調用 `AuthenticationService.authenticateToken()`

   - 失敗時才降級調用 `LaleTokenUtils.getDecodedJWTInToken()`

  

---

  

## 推薦的密鑰管理方案

  

### 方案 1: 使用外部配置文件（推薦）

  

**優點**: 安全、靈活、易於維護

  

**實現步驟**:

  

#### 1.1 創建配置文件

在 `/src/resources/` 下新增 `jwt-secret.properties` （分環境管理）：

  

```properties

# jwt-secret.properties

jwt.secret.base=your-secure-secret-key-here

jwt.algorithm=HS256

jwt.expiration.days=365

```

  

**安全建議**:

- 不同環境使用不同的密鑰（開發、測試、生產）

- 密鑰長度至少 32 字符

- 包含大小寫、數字、特殊字符

  

#### 1.2 創建配置類

  

```java

package com.flowring.servlet.api.auth.config;

  

import org.springframework.beans.factory.annotation.Value;

import org.springframework.stereotype.Component;

  

@Component

public class JwtSecretConfig {

    @Value("${jwt.secret.base}")

    private String baseSecret;

    @Value("${jwt.algorithm:HS256}")

    private String algorithm;

    @Value("${jwt.expiration.days:365}")

    private int expirationDays;

    public String getSecret(String companyName) {

        // 結合基礎密鑰和公司名稱，但不硬編碼初始值

        return baseSecret + companyName;

    }

    public String getAlgorithm() {

        return algorithm;

    }

    public int getExpirationDays() {

        return expirationDays;

    }

}

```

  

#### 1.3 修改 LaleTokenUtils

  

```java

public static DecodedJWT getDecodedJWTInToken(String token) {

    String companyName = WebSystem.getCorp();

    String secret = Util.getSecret(companyName); // 改由配置類提供

    try {

        Algorithm algorithm = Algorithm.HMAC256(secret);

        JWTVerifier verifier = JWT.require(algorithm).build();

        return verifier.verify(token);

    } catch (JWTVerificationException e) {

        logger.warn("Failed to verify JWT token", e);

        return null;

    }

}

```

  

---

  

### 方案 2: 使用環境變量

  

**優點**: 容器化友好、CI/CD 流程集成度高

  

#### 2.1 使用環境變量獲取密鑰

  

```java

public class Util {

    public static String getSecret(String productUserName) {

        // 從環境變量讀取密鑰

        String secret = System.getenv("JWT_SECRET_BASE");

        if (secret == null || secret.trim().isEmpty()) {

            // 降級到配置文件或拋出異常

            throw new IllegalStateException(

                "JWT_SECRET_BASE environment variable not set"

            );

        }

        return secret + productUserName;

    }

}

```

  

#### 2.2 Docker 部署示例

  

```dockerfile

FROM tomcat:8.5-jdk8

ENV JWT_SECRET_BASE=your-secure-secret-key-here

```

  

```bash

# 命令行運行

docker run -e JWT_SECRET_BASE="your-secure-key" -d tomcat:8.5

```

  

---

  

### 方案 3: 使用密鑰管理服務（最安全）

  

**優點**: 最高安全性、適合企業級應用

  

#### 3.1 使用 Vault 或類似服務

  

```java

public class VaultSecretProvider implements SecretProvider {

    private VaultClient vaultClient;

    public String getSecret(String companyName) {

        // 從 Vault 獲取密鑰

        Secret secret = vaultClient.read("secret/jwt/" + companyName);

        return secret.getData().get("key") + companyName;

    }

}

```

  

#### 3.2 使用 AWS Secrets Manager

  

```java

public class AwsSecretProvider implements SecretProvider {

    private SecretsManagerClient client;

    public String getSecret(String companyName) {

        GetSecretValueRequest request = GetSecretValueRequest.builder()

            .secretId("jwt/secret")

            .build();

        GetSecretValueResponse response = client.getSecretValue(request);

        return response.secretString() + companyName;

    }

}

```

  

---

  

## 實現 LaleTokenUtils.getDecodedJWTInToken() 的完整建議

  

```java

package com.flowring.servlet.api.auth;

  

import com.auth0.jwt.JWT;

import com.auth0.jwt.JWTVerifier;

import com.auth0.jwt.algorithms.Algorithm;

import com.auth0.jwt.exceptions.JWTVerificationException;

import com.auth0.jwt.interfaces.Claim;

import com.auth0.jwt.interfaces.DecodedJWT;

import org.apache.log4j.Logger;

  

public class LaleTokenUtils {

    private static Logger logger = Logger.getLogger(LaleTokenUtils.class);

    public static final String CLAIM_USER_NAME = "user_name";

  

    public static String getUsernameFromToken(String token) {

        Claim claim = getClaim(token, CLAIM_USER_NAME);

        return claim != null ? claim.asString() : null;

    }

  

    public static Claim getClaim(String token, String claimName) {

        DecodedJWT jwt = getDecodedJWTInToken(token);

        if (jwt != null) {

            return jwt.getClaim(claimName);

        }

        return null;

    }

  

    /**

     * AF-9081: 使用適當的密鑰管理方式解碼 JWT Token

     *

     * 實現邏輯：

     * 1. 從配置或環境變量獲取密鑰基礎部分

     * 2. 結合公司名稱組合成完整密鑰

     * 3. 使用 HMAC256 算法創建 JWTVerifier

     * 4. 驗證 Token 並返回解碼後的 JWT

     * 5. 若驗證失敗，捕獲異常並返回 null

     *

     * @param token JWT token 字符串

     * @return 解碼後的 JWT，如果驗證失敗返回 null

     */

    public static DecodedJWT getDecodedJWTInToken(String token) {

        try {

            String companyName = WebSystem.getCorp();

            // 從配置類而不是硬編碼獲取密鑰

            // 密鑰來源（優先級）：環境變量 > 配置文件 > 密鑰管理服務

            String secret = Util.getSecret(companyName);

            // 驗證密鑰是否存在

            if (secret == null || secret.isEmpty()) {

                logger.error("Failed to obtain JWT secret for company: " + companyName);

                return null;

            }

            // 使用 HMAC256 算法驗證 Token

            Algorithm algorithm = Algorithm.HMAC256(secret);

            JWTVerifier verifier = JWT.require(algorithm).build();

            return verifier.verify(token);

        } catch (JWTVerificationException e) {

            // JWT 驗證失敗（簽名不匹配、過期、聲明無效等）

            logger.debug("JWT token verification failed: " + e.getMessage());

            return null;

        } catch (IllegalArgumentException e) {

            // 算法或密鑰配置錯誤

            logger.error("Invalid algorithm or secret: " + e.getMessage());

            return null;

        } catch (Exception e) {

            // 未預期的異常

            logger.error("Unexpected error while decoding JWT token", e);

            return null;

        }

    }

}

```

  

---

  

## 安全檢查清單

  

在實施前，請確保滿足以下條件：

  

### 密鑰強度檢查

- [ ] 密鑰長度至少 32 字符

- [ ] 密鑰包含大小寫字母

- [ ] 密鑰包含數字

- [ ] 密鑰包含特殊字符

  

### 代碼檢查

- [ ] 不在任何代碼文件中硬編碼密鑰

- [ ] 所有 JWT 驗證均使用配置的密鑰

- [ ] 密鑰獲取失敗時有適當的異常處理

- [ ] 日誌中不輸出實際的密鑰內容

  

### 配置檢查

- [ ] 密鑰文件已添加到 `.gitignore`

- [ ] 為不同環境使用不同的密鑰

- [ ] 配置文件存放在安全位置

- [ ] 環境變量正確配置

  

### 部署檢查

- [ ] 開發環境配置已測試

- [ ] 測試環境配置已測試

- [ ] 生產環境使用安全的密鑰

- [ ] 應用成功啟動並通過 JWT 驗證測試

  

---

  

## .gitignore 配置建議

  

```gitignore

# JWT 密鑰相關

**/jwt-secret.properties

**/jwt-secret-*.properties

**/.env

**/.env.local

**/secrets/

```

  

---

  

## 遷移計畫

  

### 第一階段：快速修復（1-2 週）

1. 使用方案 1（配置文件）快速修復

2. 實現 `getDecodedJWTInToken()` 方法

3. 移除硬編碼密鑰

4. 開發環境驗證

  

### 第二階段：環境適配（2-3 週）

1. 為各環境創建不同配置文件

2. 測試環境驗證

3. 根據部署環境評估方案 2（環境變量）

4. 配置 Docker / Kubernetes 環境變量

  

### 第三階段：生產部署（3-4 週）

1. 生產環境密鑰生成和配置

2. 灰度部署測試

3. 監控告警配置

4. 文檔和運維手冊更新

  

---

  

## 密鑰生成方式

  

### 使用 OpenSSL（推薦）

```bash

openssl rand -base64 32

```

  

### 使用 Python

```python

import secrets

import string

  

alphabet = string.ascii_letters + string.digits + string.punctuation

secret = ''.join(secrets.choice(alphabet) for i in range(32))

print(f"Generated Secret: {secret}")

```

  

### 使用 Linux /dev/urandom

```bash

head -c 32 /dev/urandom | base64

```

  

---

  

## 環境特定配置示例

  

### 開發環境 (jwt-secret-dev.properties)

```properties

jwt.secret.base=dev-secret-key-for-testing-only-change-in-production

jwt.algorithm=HS256

jwt.expiration.days=30

jwt.issuer=Flowring-Dev

```

  

### 測試環境 (jwt-secret-test.properties)

```properties

jwt.secret.base=test-secret-key-for-testing-only-change-in-production

jwt.algorithm=HS256

jwt.expiration.days=90

jwt.issuer=Flowring-Test

```

  

### 生產環境配置

生產環境密鑰應通過以下方式之一獲取：

1. **環境變量**：`JWT_SECRET_BASE`

2. **Docker Secrets**

3. **Kubernetes ConfigMap/Secret**

4. **密鑰管理服務**（Vault、AWS Secrets Manager 等）

  

```bash

# 設置環境變量方式

export JWT_SECRET_BASE="your-production-secret-key-here"

java -jar application.jar --spring.profiles.active=prod

```

  

---

  

## Docker 部署示例

  

### 基礎 Dockerfile

```dockerfile

FROM tomcat:8.5-jdk8

  

# 設置默認值（應在運行時覆蓋）

ENV JWT_SECRET_BASE=changeme

ENV SPRING_PROFILES_ACTIVE=prod

  

COPY target/webagenda.war /usr/local/tomcat/webapps/

  

EXPOSE 8080

CMD ["catalina.sh", "run"]

```

  

### Docker Compose

```yaml

version: '3.8'

services:

  webagenda:

    image: webagenda:latest

    environment:

      JWT_SECRET_BASE: ${JWT_SECRET_BASE}

      SPRING_PROFILES_ACTIVE: prod

    ports:

      - "8080:8080"

    volumes:

      - ./logs:/usr/local/tomcat/logs

```

  

### Docker 命令行運行

```bash

docker run \

  -e JWT_SECRET_BASE="your-secure-key-here" \

  -e SPRING_PROFILES_ACTIVE=prod \

  -p 8080:8080 \

  webagenda:latest

```

  

---

  

## Kubernetes 部署示例

  

### 創建 Secret

```yaml

apiVersion: v1

kind: Secret

metadata:

  name: jwt-secret

  namespace: default

type: Opaque

stringData:

  jwt-secret-base: "your-production-secret-key"

```

  

### 在 Deployment 中使用

```yaml

apiVersion: apps/v1

kind: Deployment

metadata:

  name: webagenda

spec:

  replicas: 3

  selector:

    matchLabels:

      app: webagenda

  template:

    metadata:

      labels:

        app: webagenda

    spec:

      containers:

      - name: webagenda

        image: webagenda:latest

        env:

        - name: JWT_SECRET_BASE

          valueFrom:

            secretKeyRef:

              name: jwt-secret

              key: jwt-secret-base

        - name: SPRING_PROFILES_ACTIVE

          value: prod

        ports:

        - containerPort: 8080

```

  

---

  

## 參考資源

  

- [Auth0 Java-JWT 文檔](https://github.com/auth0/java-jwt)

- [OWASP: Cryptographic Key Management](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)

- [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)

- [CWE-798: Use of Hard-coded Credentials](https://cwe.mitre.org/data/definitions/798.html)

- [JWT 介紹](https://jwt.io/)