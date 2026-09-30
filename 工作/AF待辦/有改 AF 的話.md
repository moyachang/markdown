(1) 匯入智雲的部份，要加 log 及註解，不然很難看看得懂。
我應該把程式備在

以下這三個的寫法都很危險，
**
<font color="#4bacc6">• 先檢查 webAgendaUrl 是否為 null / 空字串</font>

<font color="#4bacc6">• 發 request 後先檢查 HTTP status</font>

<font color="#4bacc6">• 只有 2xx 才進一步處理</font>

<font color="#4bacc6">• 檢查 response 是否為空</font>

<font color="#4bacc6">• 檢查 response 是否以 { 開頭，避免把 HTML 500 頁誤當 JSON</font>

<font color="#4bacc6">• 失敗時改為記錄錯誤並回傳 false，不再直接噴 JSONException</font>
**
removeWebAgendaArtifactCache-->主要修改這支，智雲要用其他看狀況。

removeWebAgendaOrgCache
removeWebAgendaProcessCacheByProId


```java=
// 清 browser 表單 Cache

public boolean removeWebAgendaArtifactCache() {

SerRuntimeEnv env = new SerRuntimeEnv();

String webAgendaUrl = env.getWebAgendaURL();

if (webAgendaUrl.isEmpty() || webAgendaUrl == null) {

logger.warn("Please check webagendaurl in server properties");

return false;

}

  

HttpClient httpClient = HttpClientBuilder.create().build();

HttpPost post = new HttpPost(webAgendaUrl + "/api/saas/art/cache/remove");

post.addHeader("content-type", "application/json;charset=utf-8");

HttpResponse httpresponse;

try {

httpresponse = httpClient.execute(post);

int statusCode = httpresponse.getStatusLine() != null ? httpresponse.getStatusLine().getStatusCode() : -1;

// String result = EntityUtils.toString(httpresponse.getEntity(), "UTF-8");

String result = httpresponse.getEntity() != null ? EntityUtils.toString(httpresponse.getEntity(), "UTF-8") : "";

if (statusCode < 200 || statusCode >= 300) {

logger.warn("removeWebAgendaArtifactCache failed, statusCode: " + statusCode + ", result: " + result);

return false;

}

if (result == null || result.trim().isEmpty()) {

logger.warn("removeWebAgendaArtifactCache returned empty response");

return false;

}

String trimmedResult = result.trim();

if (!trimmedResult.startsWith("{")) {

logger.error("removeWebAgendaArtifactCache returned non-JSON response: " + trimmedResult);

return false;

}

JSONObject json = JSONObject.fromObject(trimmedResult);

return true;

} catch (ClientProtocolException e) {

logger.error("removeWebAgendaArtifactCache protocol error", e);

} catch (IOException e) {

logger.error("removeWebAgendaArtifactCache io error", e);

} catch (Exception e) {

logger.error("removeWebAgendaArtifactCache unexpected error", e);

}

return false;

}

```

```java
//原 AI 建議程式
public boolean removeWebAgendaArtifactCache() {
    SerRuntimeEnv env = new SerRuntimeEnv();
    String webAgendaUrl = env.getWebAgendaURL();
    if (webAgendaUrl == null || webAgendaUrl.isEmpty()) {
        logger.warn("Please check webagendaurl in server properties");
        return false;
    }

    HttpClient httpClient = HttpClientBuilder.create().build();
    HttpPost post = new HttpPost(webAgendaUrl + "/api/saas/art/cache/remove");
    post.addHeader("content-type", "application/json;charset=utf-8");
    try {
        HttpResponse httpresponse = httpClient.execute(post);
        int statusCode = httpresponse.getStatusLine() != null ? httpresponse.getStatusLine().getStatusCode() : -1;
        String result = httpresponse.getEntity() != null ? EntityUtils.toString(httpresponse.getEntity(), "UTF-8") : "";

        if (statusCode < 200 || statusCode >= 300) {
            logger.error("removeWebAgendaArtifactCache failed, statusCode=" + statusCode + ", response=" + result);
            return false;
        }

        if (result == null || result.trim().isEmpty()) {
            logger.warn("removeWebAgendaArtifactCache returned empty response");
            return false;
        }

        String trimmedResult = result.trim();
        if (!trimmedResult.startsWith("{")) {
            logger.error("removeWebAgendaArtifactCache returned non-JSON response: " + trimmedResult);
            return false;
        }

        JSONObject.fromObject(trimmedResult);
        return true;
    } catch (ClientProtocolException e) {
        logger.error("removeWebAgendaArtifactCache protocol error", e);
    } catch (IOException e) {
        logger.error("removeWebAgendaArtifactCache io error", e);
    } catch (Exception e) {
        logger.error("removeWebAgendaArtifactCache unexpected error", e);
    }
    return false;
}

```

