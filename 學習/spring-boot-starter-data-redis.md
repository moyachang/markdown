`spring-boot-starter-data-redis` 是 Spring Boot 官方提供的「Redis 資料庫操作懶人包（Starter）」。

它的核心用途是讓您的 Spring Boot 專案能夠極為簡便、快速地連接並操作 Redis（一種記憶體內高效能 NoSQL 資料庫）。引入它之後，您不需要寫繁瑣的底層連線與關閉程式碼，Spring 會自動幫您把連線池（預設為 Lettuce 驅動）配置好。

以下是它在實際開發中的 4 大核心用途與程式碼寫法：

---

## 1. 實現快取機制（@Cacheable）

這是最常見的用途。您可以利用 Redis 當作快取層，把需要頻繁查詢、但很少變動的資料（例如：產品列表、使用者設定）暫存起來，大幅減輕 MySQL 等關聯式資料庫的負擔，提升 API 響應速度。

- 使用方式： 您甚至不需要寫 Redis 語法，只要加上註解即可：
    
    ```java
    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        // 如果 Redis 有資料，直接回傳；沒有才會去查 MySQL 
        return userRepository.findById(id).orElse(null); 
    }
    ```
    

## 2. 手動存取、操作各種 Redis 資料結構（RedisTemplate）

當您需要精細控制 Redis 內的 Key-Value，或是使用 Redis 特有的資料結構（如 `String`、`List`、`Set`、`Hash`、`ZSet` 有序集合）時，它會自動注入 `RedisTemplate` 或 `StringRedisTemplate` 給您使用。

- 使用方式：
    
    ```java
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public void saveData() {
        // 存入一個 5 分鐘後過期的驗證碼
        redisTemplate.opsForValue().set("sms:code:0912345678", "123456", 5, TimeUnit.MINUTES);
    
        // 讀取資料
        String code = redisTemplate.opsForValue().get("sms:code:0912345678");
    }
    ```
    

## 3. 分布式 Session 管理（Distributed Session）

在分散式或微服務架構中，如果使用者登入 A 伺服器，下一秒請求跑到 B 伺服器，可能就會被當作未登入。

- 使用方式： 配合 `spring-session-data-redis`，它能把使用者的登入 Token / Session 統一存在 Redis 中。這樣不管使用者連到哪一台伺服器，都能共享登入狀態（實現 Single Sign-On 概念）。 [1]

## 4. 實現分布式鎖（Distributed Lock）

在多台伺服器同時運行的環境下，為了防止「高並發搶購/超賣」或是「重複處理同一筆訂單」，需要一個跨伺服器的鎖。

- 使用方式： 它可以配合 Redis 的 `setIfAbsent` (NX) 命令，確保同一個時間點，全系統只有一台伺服器能執行特定核心邏輯。

---

## 📦 引入後的基礎配置（application.properties）

只要引入了這個 Starter，您通常只需要在設定檔中告訴它 Redis 的主機在哪裡，就可以開始用了：

```properties
# 現代 Spring Boot 3+ 的 Redis 配置路徑
spring.data.redis.host=127.0.0.1
spring.data.redis.port=6379
spring.data.redis.password=您的密碼
```

想請問您目前是打算用 Redis 來做資料快取、儲存驗證碼，還是有其他特殊的分散式架構需求呢？我可以針對您的情境提供具體的實作範例！

  

[1] [https://juejin.cn](https://juejin.cn/post/7446951664816144399)