# 專案職務樹 API 修正建議  
  
> 針對目前工作目錄的 `GetProjectRoleTreeUseCase` 與 `GET /up/api/project/{projectId}/org-tree`。  
> 日期：2026-10-10。此文件為設計建議，所有目標目錄與示意程式均尚未實作。只讀取專案，不修改程式、不執行建置或資料庫操作。  
  
## 1. 建議採用的方向  
  
以 **UseCase 表達一個應用操作、DDD 管理需要業務行為的模型、Clean 管理依賴方向、六角架構提供 Port／Adapter 邊界**，這四件事可以搭配。  
  
這支 API 最適合先整理為「讀取用 UseCase＋查詢 Port＋持久化 Adapter＋Web Mapper」。保留批次取得資料和集中組樹的做法，修正型別與 package 依賴即可，不必把每一次讀取都改成載入 Aggregate。  
  
`UseCase` 名稱比 `Service` 更能指出一個操作的目的。Spring 的 `@Service` 是註冊 Bean 的方式，不妨礙類別叫 UseCase；也不用為了形式立即建立每個 UseCase 的介面。  
  
**第一階段的完成標準：UseCase 不認識 Web DTO，Domain 不認識 Application Query Model，持久化轉換不放在 Web package。** 這是讓目前程式成為可解釋案例最有價值的範圍。  
  
## 2. 現況與需要修正的責任  
  
| 目前實作                                                            | 問題／判斷                                 | 建議                                                    |     |
| --------------------------------------------------------------- | ------------------------------------- | ----------------------------------------------------- | --- |
| UseCase 注入 `ProjectRoleTreeMapper`、回傳 `ProjectRoleTreeResponse` | Application 依賴 Web 表示型別               | UseCase 回 Application Result；Controller 呼叫 Web Mapper |     |
| Query Repository 介面在 Domain，回傳 `application.query` records      | Domain 反向依賴 Application               | 移至 Application 的 outbound/query Port                  |     |
| `MemberInfoMapper` 位於 `adapter.web.mapper`，輸入為 JPA Entity       | package 與工作內容不符；Infrastructure 依賴 Web | 移至 persistence mapper，明確是 Entity → Query Model        |     |
| `MemberTypeRow` 暴露在 Application，但僅供人員 Adapter 分流                | 持久化實作細節進入應用契約                         | 移入 persistence projection；同步修改 JPQL constructor 名稱    |     |
| `RoleRow` 與 `RoleTreeResult` 命名太泛                               | 容易與其他角色、Domain 模型混淆                   | 用 ProjectRole／TreeNode 明確命名                           |     |
| Assembler 建索引、檢查完整性、組樹                                          | 集中且可獨立驗證，方向合理                         | 保留在此 UseCase 的 Application 功能 package                 |     |
| `JpaMemberQueryRepositoryAdapter` 批次分流查 PTM／PTS／一般人員            | 是讀取整合 Adapter 的合理責任                   | 保留批次流程，不為每個類型新增一層轉手 UseCase                           |     |
| `ProjectRole.buildTree` 與 Query Assembler 有兩種組樹方式               | 缺失父節點的處理不同，形成兩套語意                     | 此 API 明確只採 Query Assembler；其他模型另查使用處後再決定整併            |     |
  
最後一項的具體差異：`ProjectRole.buildTree` 把找不到父節點的角色當根；`ProjectRoleTreeAssembler` 拋錯並檢查循環。本次不直接刪除前者，因為它不在選定 API 的呼叫鏈，應先盤點其他使用處。  
  
## 3. 建議目錄：保留層級、在 Application 內依功能集中  
  
根 package 仍為 `com.flowring.up`。目標如下；這是本 API 的局部整理，其他功能不需一次跟著遷移。  
  
```text  
com.flowring.up  
├─ application  
│  └─ projectroletree  
│     ├─ GetProjectRoleTreeUseCase.java  
│     ├─ ProjectRoleTreeAssembler.java  
│     ├─ model  
│     │  ├─ ProjectRoleSnapshot.java  
│     │  ├─ ProjectRoleMemberLink.java  
│     │  ├─ ProjectRoleTreeMember.java  
│     │  └─ ProjectRoleTreeNode.java  
│     ├─ port  
│     │  └─ out  
│     │     ├─ ProjectRoleTreeQueryPort.java  
│     │     └─ ProjectRoleTreeMemberQueryPort.java  
│     └─ exception  
│        └─ InvalidProjectRoleTreeException.java  
├─ adapter  
│  └─ web  
│     ├─ controller/projectorg/ProjectOrgTreeController.java  
│     ├─ dto/response/ProjectRoleTreeResponse.java  
│     ├─ dto/response/ProjectRoleEmployeeResponse.java  
│     └─ mapper/ProjectRoleTreeResponseMapper.java  
├─ infrastructure  
│  └─ persistence/jpa  
│     ├─ repository/JpaProjectRoleTreeQueryAdapter.java  
│     ├─ repository/JpaProjectRoleTreeMemberQueryAdapter.java  
│     ├─ repository/SpringData...Repository.java  
│     ├─ mapper/ProjectRoleTreeMemberPersistenceMapper.java  
│     ├─ projection/ProjectMemberTypeProjection.java  
│     └─ entity/...JpaEntity.java  
└─ domain  
   └─ ...（這支 Query 不新增 Domain Entity 或 Domain Service）```  
  
`adapter.web` 與 `infrastructure.persistence` 都是架構外圈，目前根名稱不同不是最優先問題；先保留現有外圈命名，減少無關搬動。`port.out` 表示 UseCase 向外要求資料的契約，**不是**說原始碼應依賴外圈實作。  
  
Application 的功能 package 能把完成同一需求所需的類別集中，避免所有 UseCase 堆在 `application.service`、所有 result 放在另一個大型目錄。  
  
### 類別與方法對照  
  
| 現有 | 建議 | 原因 |  
| --- | --- | --- |  
| `application.service.GetProjectRoleTreeUseCase` | `application.projectroletree.GetProjectRoleTreeUseCase` | 保留操作名稱，集中功能 |  
| `ProjectRoleQueryRepository` | `ProjectRoleTreeQueryPort` | 明確為讀取樹所需資料，不是 Aggregate Repository |  
| `MemberQueryRepository` | `ProjectRoleTreeMemberQueryPort` | 限定本功能，避免形成全能人員 Repository |  
| `RoleRow` | `ProjectRoleSnapshot` | 已投影的角色資料；不代表 Entity |  
| `RoleMemberRow` | `ProjectRoleMemberLink` | 表達職務與人員的關聯 |  
| `application.query.MemberInfo` | `ProjectRoleTreeMember` | 與 common／domain 的同名 MemberInfo 區隔 |  
| `RoleTreeResult` | `ProjectRoleTreeNode` | 回傳的是節點清單，不是單一總結果物件 |  
| `ProjectRoleTreeMapper` | `ProjectRoleTreeResponseMapper` | 明確轉成 Web Response |  
| `MemberInfoMapper` | `ProjectRoleTreeMemberPersistenceMapper` | 明確從持久化來源轉為 Application 模型 |  
| `findRoles` | `findRolesByProjectId` | 查詢範圍清楚 |  
| `findRoleMembers` | `findRoleMemberLinksByProjectId` | 回傳關聯，不是詳細人員 |  
| `findByIds` | `findMembersByProjectIdAndIds` | 顯示 projectId 參與；註明僅部分底層來源有 project 條件 |  
  
內部型別逐步使用 `roleId`、`memberId`、`parentRoleId`、`memberType`、`avatarUrl`。資料庫欄位 `ROL_ID`／`MEM_ID` 與既有 Web 欄位 `rolId`／`memId` 保留，由 Adapter／Mapper 對映。內部改名不應順便破壞客戶端 JSON。  
  
## 4. 目標依賴方向  
  
```mermaid  
flowchart TB  
    C[Web Controller] --> U[GetProjectRoleTreeUseCase]    C --> WM[Web Response Mapper]    WM --> Model[Application Query Models]    U --> Model    U --> AS[Application Tree Assembler]    U --> P[Application Outbound Query Ports]    P --> Model    JA[JPA Query Adapters] --> P    JA --> PM[Persistence Mapper]    PM --> Model    JA --> SD[Spring Data Repositories]    SD --> E[JPA Entities / Persistence Projection]    SD --> Model```  
  
圖中箭頭是原始碼依賴。Spring Data 的角色查詢仍可直接用 JPQL constructor projection 回 Application Model：外圈依賴內圈是允許的。唯有 `ProjectMemberTypeProjection` 這種未出現在 Port 契約中的實作細節，留在 Infrastructure。  
  
流程仍是 Controller → UseCase → Port 的實作 → DB → Application Model → Assembler → Web Mapper → Response。這裡不加 Domain 層轉手呼叫；DDD 的寫入模型可以另有 Entity 行為與 Repository。  
  
### 為什麼暫不建立 inbound Port？  
  
目前只有 HTTP Controller 使用此 UseCase。直接依賴具體 UseCase 已足以表達應用邊界。若之後有排程、CLI、訊息 consumer 或多套執行實作，再抽 `application/.../port/in/GetProjectRoleTree` 介面，由 UseCase 實作。使用六角觀念不表示每個類別必須有同名介面。  
  
## 5. 建議的核心契約與示意程式  
  
以下皆為目標設計節錄，並非完整可編譯 patch，也尚未寫回專案。  
  
### 5.1 Application Model  
  
```java  
public record ProjectRoleSnapshot(  
        String roleId, String name, String parentRoleId) {}  
  
public record ProjectRoleMemberLink(String roleId, String memberId) {}  
  
public record ProjectRoleTreeMember(  
        String memberId, String memberType, String name,  
        String avatarUrl, String orgPathName, String orgPathId) {}  
  
public record ProjectRoleTreeNode(  
        String roleId, String name,  
        List<ProjectRoleTreeMember> members,  
        List<ProjectRoleTreeNode> children) {}  
```  
  
這些型別不是 Domain Entity；不使用 JPA 註解、不 import Web DTO。為方便分段遷移，第一階段可先保留原 record 名稱，只改 package 與依賴；名稱調整放下一個可獨立檢查的變更。  
  
目前 `avatarUrl` 由共用服務產生，先保留在讀取模型並明確命名即可。若之後同一 UseCase 用於非 Web 情境，再考慮把頭像來源資訊與 URL 呈現分開，沒有必要現在多加一個介面。  
  
### 5.2 Query Ports  
  
```java  
public interface ProjectRoleTreeQueryPort {    List<ProjectRoleSnapshot> findRolesByProjectId(String projectId);    List<ProjectRoleMemberLink> findRoleMemberLinksByProjectId(String projectId);}  
  
public interface ProjectRoleTreeMemberQueryPort {    List<ProjectRoleTreeMember> findMembersByProjectIdAndIds(  
            String projectId, Set<String> memberIds);  
}  
```  
  
約定：回非 null 清單；找不到個別人員時，由 Assembler 在檢查關聯時產生明確完整性錯誤。memberIds 空集合時回空清單。它不是授權 Port：參數有 projectId，不表示底層每個資料來源都有專案隔離條件，也不表示已檢查呼叫者權限。  
  
### 5.3 UseCase  
  
```java  
@Service  
@RequiredArgsConstructor  
public class GetProjectRoleTreeUseCase {    private final ProjectRoleTreeQueryPort roleQuery;  
    private final ProjectRoleTreeMemberQueryPort memberQuery;  
    private final ProjectRoleTreeAssembler assembler;  
  
    @Transactional(readOnly = true)    public List<ProjectRoleTreeNode> execute(String projectId) {        var roles = roleQuery.findRolesByProjectId(projectId);  
        var links = roleQuery.findRoleMemberLinksByProjectId(projectId);  
        var memberIds = links.stream()                .map(ProjectRoleMemberLink::memberId)                .collect(Collectors.toSet());        var members = memberIds.isEmpty() ? List.<ProjectRoleTreeMember>of()                : memberQuery.findMembersByProjectIdAndIds(projectId, memberIds);  
        return assembler.assemble(roles, links, members);  
    }}  
```  
  
交易留在 UseCase，不移到 Controller。允許 Application 使用 Spring `@Service`／`@Transactional` 作為目前務實取捨；若未来確實要求核心零框架依賴，再以組裝配置或 decorator 處理。現在先修掉 Web／JPA 型別外洩，比移除所有 Spring 註解更有收益。  
  
只有一個 String 輸入時，先不增加 `GetProjectRoleTreeQuery`。若加入 actor、篩選、讀取模式等條件，再引入 query record。  
  
### 5.4 Controller  
  
```java  
@GetMapping("/{projectId}/org-tree")  
@PreAuthorize("isAuthenticated()")  
public ApiResponse<List<ProjectRoleTreeResponse>> getProjectRoleTree(  
        @PathVariable String projectId) {    var tree = getProjectRoleTreeUseCase.execute(projectId);    return ApiResponse.success(responseMapper.toResponses(tree));}  
```  
  
Controller 同時注入 UseCase 與 Response Mapper。JSON 仍用 `treeId`、`rolId`、`employees`、`orgs`；Mapper 將 roleId 對到 rolId，members 對到 employees，children 對到 orgs。人員 treeId 仍維持 `roleId_memberId`。方法名中的 `OfAF` 可移除，但路徑是否真的只限 AF 人員是另一個需求問題。  
  
## 6. 行為問題要與 package 整理分開處理  
  
整理 package 前後應先保持目前行為；以下調整牽涉需求或客戶端契約，另列變更，不混進單純搬檔。  
  
| 問題 | 現有行為 | 建議決策 |  
| --- | --- | --- |  
| API 是否只回組織內部人員 | 人員 Adapter 支援 PTM、PTS 與其他類型，未限內部人員 | 先確認此樹是所有成員或 AF 專屬；若要篩選，連關聯／完整性策略一起調整，不能只濾掉人員後造成缺失錯誤 |  
| 專案不存在 | roles／links 都空時成功回空清單 | 區分「存在但無角色」與「專案不存在」是否必要；若需要，透過有實際來源的專案查詢契約確認，不假造已存在 Repository |  
| 專案授權 | Controller 只有 isAuthenticated | 若資料按專案限制可見，新增 Application 授權檢查；caller 由 Web／安全 Adapter 取得並傳入，UseCase 不直接讀 SecurityContext |  
| 父職務缺失／人員缺失 | IllegalStateException，整次查詢失敗 | 第一階段保留 fail-fast；轉為可識別的應用例外及錯誤代碼，避免靜默丟資料 |  
| 循環與重複 ID | visited／toMap 間接拋錯 | 明確分類 missing-parent、missing-member、cycle、duplicate-id；log 記錄定位資訊，HTTP 不暴露內部 ID／堆疊細節 |  
| 根角色定義 | parentId null／blank | 確認 DB 是否有其他 sentinel；如有，在 persistence 轉換處正規化，勿隨意把找不到父職務都視為根 |  
| 排序 | rolId；人員關聯按 rolId、memId | 先保留；若 UI 要業務順序，新增明確排序契約及資料來源 |  
| 深度與大資料量 | 遞迴組樹、大集合 IN 查詢 | 有規模證據後再選 iterative traversal、深度限制或 IN 分批；不預先增加複雜度 |  
  
缺失父職務等情況來自持久化資料一致性，不宜全部回 HTTP 400，因客戶端不一定能修正。可用內部資料錯誤類別對應 HTTP 500；project not found 對應 404，forbidden 對應 403，但須配合既有 API 版本與客戶端需求。  
  
`InvalidProjectRoleTreeException` 放 Application，因目前規則用於讀取模型組裝。若後續同樣規則成為職務搬移的業務不變條件，再評估提到 Domain 並供寫入流程使用；不要讓 Domain 依賴這些 Query records。  
  
## 7. common 邊界：這次做到哪裡  
  
本 API 的 up persistence Adapter 直接使用 common 的 Spring Data Repository／Entity。此依賴發生在外圈，未必違反 Clean 的核心規則，但讓 up 綁定 common 持久化結構。  
  
建議第一階段維持現有跨模組查詢，把 Mapper 移對位置即可。文件註明「common persistence 目前共用」，不要為了架構圖先包一個沒有價值的 Facade。  
  
若下一階段要求 common 可獨立變更，或有第二個功能也需要相同人員查詢，再抽 common 對外的批次人員讀取契約，由 common Adapter 實作，回傳穩定的共用資料型別。此時一般人員／外部人員查詢可移入 common；專案 PTM 與 UP4_MEM 分流仍由 up 組合。common 的對外契約不得回傳 up 的 ProjectRoleTreeMember，以免模組依賴反轉。  
  
`ApiResponse`、`RRExceptionHandler` 目前留在 common 外圈。回應格式／HTTP status 統一要獨立處理，先盤點其他 endpoint，避免只改這條鏈便破壞既有客戶端。  
  
## 8. 建議遷移順序與每步檢查  
  
1. **先固定此 API 的行為。** 記錄成功 JSON、排序、空資料與異常規則，避免重構時同時改需求。  
2. **拆開 Web 邊界。** UseCase 回原 `RoleTreeResult`；Mapper 移至 Controller 使用。先不改內部名稱。檢查 Application 已無這支 API 的 Web import。  
3. **移動 Query Ports。** 由 Domain 移至 Application；同步修改 UseCase、Adapter import。檢查 Domain 不再引用該功能的 Application Query records。  
4. **搬持久化 Mapper／Projection。** MemberInfoMapper 移 Infrastructure；MemberTypeRow 移 persistence projection。更新 JPQL 的完整類名及所有來源 import。  
5. **整理功能 package 與名稱。** 套用對照表，一次完成所有使用處、MapStruct source 名稱與 JPQL constructor 名稱。保留 DB 與 Web 名稱。  
6. **另做行為修正。** 明確應用例外、HTTP 契約；確認 AF 篩選、專案存在與授權需求後再做對應實作。  
  
不要以「先建立新版本並保留兩套平行流程」長期遷移，否則同一需求會持續有不同規則。若短期需要相容包裝，指定移除時點與唯一真實流程。  
  
### 後續實作時的驗證（本次未執行）  
  
- Controller／Mapper：確認路徑、JSON 欄位與 treeId 規則保持相容。  
- UseCase：memberIds 去重；無關聯不呼叫人員 Port；交易註解保留。  
- Assembler：多根、多層、同一人多角色、缺父／缺人、沒有根的循環、重複 ID、空資料。  
- 人員 Adapter：PTM／PTS／一般人員分流；一般人員使用 mainRoleId 對應路徑；空集合不形成 IN 查詢。  
- 啟動／持久化：JPQL 字串內 record 完整類名不是 Java import，搬 package 後需以 Spring Data／JPA 啟動或整合驗證；編譯通過不保證 JPQL 正確。  
- MapStruct：變更 getter／record component 名稱後跑 annotation processing，確保遞迴 mapper 與未對映欄位檢查通過。  
- 架構約束：可用 ArchUnit 或既有規則禁止 Application import `adapter.web`／`infrastructure`，禁止 Domain import Application；有共識後再加入工具。  
  
## 9. 何時適合拿來作技術分享案例？  
  
第一階段完成後，就能清楚展示「Web 表示、應用協調、查詢契約、持久化轉換」四種責任，不必等待整個專案重構完。若分享標題包含 DDD，另外展示一個真正呼叫 Entity 行為的寫入流程，並誠實說明此職務樹 API 是讀取模型，不會經過 Aggregate。  
  
適合本次的修正規模是三個邊界調整加功能命名整理，不是重新設計整個 common／up。後續每新增一個 UseCase 都沿用相同約定，比一次搬動所有 package 更容易維持一致。  
  
## 10. 現有程式碼參照  
  
- [GetProjectRoleTreeUseCase.java](D:/1211/newWFCIService-main/up/src/main/java/com/flowring/up/application/service/GetProjectRoleTreeUseCase.java)  
- [ProjectOrgTreeController.java](D:/1211/newWFCIService-main/up/src/main/java/com/flowring/up/adapter/web/controller/projectorg/ProjectOrgTreeController.java)  
- [ProjectRoleTreeAssembler.java](D:/1211/newWFCIService-main/up/src/main/java/com/flowring/up/application/result/ProjectRoleTreeAssembler.java)  
- [ProjectRoleTreeMapper.java](D:/1211/newWFCIService-main/up/src/main/java/com/flowring/up/adapter/web/mapper/ProjectRoleTreeMapper.java)  
- [MemberInfoMapper.java](D:/1211/newWFCIService-main/up/src/main/java/com/flowring/up/adapter/web/mapper/MemberInfoMapper.java)  
- [JpaMemberQueryRepositoryAdapter.java](D:/1211/newWFCIService-main/up/src/main/java/com/flowring/up/infrastructure/persistence/jpa/repository/JpaMemberQueryRepositoryAdapter.java)  
- [ProjectRole.java](D:/1211/newWFCIService-main/up/src/main/java/com/flowring/up/domain/model/entity/ProjectRole.java)