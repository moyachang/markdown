這就為您依照 **DDD + Clean Architecture** 架構規範，並結合 **Lombok** 與 **MapStruct**，針對 `UP4_PROJECT_EVENT`（專案事件/大事記）產製完整的 JPA 相關程式碼：

  

包含：

  

1. **Domain Model** (`UP4ProjectEvent`)：支援新增與修改的商業模型。

2. **Domain Repository Port** (`UP4ProjectRepository` 概念延伸的 `UP4ProjectEventRepository`)：領域層介面。

3. **JPA Entity** (`UP4ProjectEventJpaEntity`)：對應資料庫表格與 `@Lob`（處理 `NVARCHAR(MAX)`）。

4. **Spring Data Repository** (`SpringDataUP4ProjectEventRepository`)：底層資料庫操作介面。

5. **MapStruct Mapper** (`UP4ProjectEventMapper`)：編譯期自動轉譯。

6. **Repository Adapter** (`JpaUP4ProjectEventRepositoryAdapter`)：基礎設施層的轉接器實作。

  

---

  

### 1. Domain Model (`UP4ProjectEvent`)

  

放在 Domain 層（例如 `com.flowring.up.domain.model`），使用普通 Class 支援業務行為。

  

```java

package com.flowring.up.domain.model;

  

import lombok.AllArgsConstructor;

import lombok.Builder;

import lombok.Data;

import lombok.NoArgsConstructor;

  

@Data

@Builder

@NoArgsConstructor

@AllArgsConstructor

public class UP4ProjectEvent {

    private String projectId;

    private String eventId;

    private Integer tag;          // null 表一般大事記，或對應 suspend/restart/termination/finished

    private String content;

    private String memId;

    private String memType;

    private Long updatedAt;

  

    // 業務行為範例：更新大事記內容或標籤

    public void updateEvent(Integer tag, String content) {

        this.tag = tag;

        this.content = content;

        this.updatedAt = System.currentTimeMillis();

    }

}

  

```

  

---

  

### 2. Domain Repository Port (`UP4ProjectEventRepository`)

  

放在 Domain 層（例如 `com.flowring.up.domain.repository`），定義領域層所需的操作介面。

  

```java

package com.flowring.up.domain.repository;

  

import com.flowring.up.domain.model.UP4ProjectEvent;

  

import java.util.List;

import java.util.Optional;

  

public interface UP4ProjectEventRepository {

  

    Optional<UP4ProjectEvent> findById(String eventId);

  

    List<UP4ProjectEvent> findByProjectId(String projectId);

  

    void save(UP4ProjectEvent projectEvent);

  

    void deleteById(String eventId);

}

  

```

  

---

  

### 3. JPA Entity (`UP4ProjectEventJpaEntity`)

  

放在 Infrastructure 持久化層（例如 `com.flowring.up.infrastructure.persistence.entity`），對應資料庫表格 `UP4_PROJECT_EVENT`。

  

```java

package com.flowring.up.infrastructure.persistence.entity;

  

import jakarta.persistence.*;

import lombok.AllArgsConstructor;

import lombok.Builder;

import lombok.Data;

import lombok.NoArgsConstructor;

  

@Data

@Builder

@NoArgsConstructor

@AllArgsConstructor

@Entity

@Table(name = "UP4_PROJECT_EVENT")

public class UP4ProjectEventJpaEntity {

  

    @Id

    @Column(name = "EVENT_ID", length = 30, nullable = false)

    private String eventId;

  

    @Column(name = "PROJECT_ID", length = 30, nullable = false)

    private String projectId;

  

    @Column(name = "TAG")

    private Integer tag; // tinyint

  

    @Lob

    @Column(name = "CONTENT", nullable = false)

    private String content; // nvarchar(max)

  

    @Column(name = "MEM_ID", length = 30, nullable = false)

    private String memId;

  

    @Column(name = "MEM_TYPE", length = 10)

    private String memType;

  

    @Column(name = "UPDATED_AT", nullable = false)

    private Long updatedAt; // bigint

}

  

```

  

---

  

### 4. MapStruct Mapper (`UP4ProjectEventMapper`)

  

放在 Infrastructure 層（例如 `com.flowring.up.infrastructure.persistence.mapper`），利用 MapStruct 進行編譯期轉換。

  

```java

package com.flowring.up.infrastructure.persistence.mapper;

  

import com.flowring.up.domain.model.UP4ProjectEvent;

import com.flowring.up.infrastructure.persistence.entity.UP4ProjectEventJpaEntity;

import org.mapstruct.Mapper;

  

@Mapper(componentModel = "spring")

public interface UP4ProjectEventMapper {

  

    // Entity -> Domain Model

    UP4ProjectEvent toDomain(UP4ProjectEventJpaEntity entity);

  

    // Domain Model -> Entity

    UP4ProjectEventJpaEntity toEntity(UP4ProjectEvent domain);

}

  

```

  

---

  

### 5. Spring Data JPA Repository (`SpringDataUP4ProjectEventRepository`)

  

放在 Infrastructure 層（例如 `com.flowring.up.infrastructure.persistence.repository`）。

  

```java

package com.flowring.up.infrastructure.persistence.repository;

  

import com.flowring.up.infrastructure.persistence.entity.UP4ProjectEventJpaEntity;

import org.springframework.data.jpa.repository.JpaRepository;

import org.springframework.stereotype.Repository;

  

import java.util.List;

  

@Repository

public interface SpringDataUP4ProjectEventRepository extends JpaRepository<UP4ProjectEventJpaEntity, String> {

  

    List<UP4ProjectEventJpaEntity> findByProjectId(String projectId);

}

  

```

  

---

  

### 6. Repository Adapter (`JpaUP4ProjectEventRepositoryAdapter`)

  

放在 Infrastructure 層（例如 `com.flowring.up.infrastructure.persistence.adapter`），實作 Domain 介面，並透過 Lombok 的 `@RequiredArgsConstructor` 注入相依元件。

  

```java

package com.flowring.up.infrastructure.persistence.adapter;

  

import com.flowring.up.domain.model.UP4ProjectEvent;

import com.flowring.up.domain.repository.UP4ProjectEventRepository;

import com.flowring.up.infrastructure.persistence.mapper.UP4ProjectEventMapper;

import com.flowring.up.infrastructure.persistence.repository.SpringDataUP4ProjectEventRepository;

import lombok.RequiredArgsConstructor;

import org.springframework.stereotype.Component;

  

import java.util.List;

import java.util.Optional;

  

@Component

@RequiredArgsConstructor

public class JpaUP4ProjectEventRepositoryAdapter implements UP4ProjectEventRepository {

  

    private final SpringDataUP4ProjectEventRepository springDataRepository;

    private final UP4ProjectEventMapper mapper;

  

    @Override

    public Optional<UP4ProjectEvent> findById(String eventId) {

        return springDataRepository.findById(eventId)

                .map(mapper::toDomain);

    }

  

    @Override

    public List<UP4ProjectEvent> findByProjectId(String projectId) {

        return springDataRepository.findByProjectId(projectId)

                .stream()

                .map(mapper::toDomain)

                .toList();

    }

  

    @Override

    public void save(UP4ProjectEvent projectEvent) {

        var entity = mapper.toEntity(projectEvent);

        springDataRepository.save(entity);

    }

  

    @Override

    public void deleteById(String eventId) {

        springDataRepository.deleteById(eventId);

    }

}

  

```