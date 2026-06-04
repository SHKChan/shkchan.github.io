---
title: Takeaways_RPA
author: SHKChan
date: 2026-06-04
category: Full_Stack
layout: post
---
 **A living scratchpad for the RPA Network project. Personal reference for "aha!" moments so I don't have to relearn them later.**

## Backend

### 1. `class` vs. `record` in Java

| Feature           | `class`                                        | `record`                                      |
| ----------------- | ------------------------------------------------ | ----------------------------------------------- |
| **Purpose** | **Behavior, mutable state, complex logic** | **Immutable data carriers (DTOs)**        |
| **Getters** | **`getFieldName()`**                     | **`fieldName()`(e.g.`user.email()`)** |

### 2. Spring Boot Core Dependencies

* **Spring Web — REST endpoints**
* **Spring Data JPA — ORM**
* **Database Driver — MySQL / PostgreSQL**
* **Lombok — Boilerplate reduction**
* **Spring Boot DevTools — Hot reload**

### 3. JPA Query Derivation

#### Subject Prefixes

| Keyword                   | Returns                 | Example                                          |
| ------------------------- | ----------------------- | ------------------------------------------------ |
| **`find...By`**   | **Entity / List** | **`findByName(String name)`**            |
| **`exists...By`** | **`boolean`**   | **`existsByEmail(String email)`**        |
| **`count...By`**  | **`long`**      | **`countByStatus(Status s)`**            |
| **`delete...By`** | **void**          | **`deleteByActiveFalse()`**              |
| **`stream...By`** | **`Stream`**    | **`streamAllByAgeGreaterThan(int age)`** |

#### Condition Keywords

* **Logical: `And`, `Or`**
* **Equality: `Is`, `Equals`, `Not`**
* **Range: `Between`, `LessThan`, `GreaterThan`, `After`, `Before`**
* **Null: `IsNull`, `IsNotNull`**
* **String: `StartingWith`, `EndingWith`, `Containing`, `Like`, `IgnoreCase`**
  * **e.g. `findByNameContainingIgnoreCase(String part)` → `WHERE UPPER(name) LIKE UPPER('%part%')`**
* **Collection: `In`, `NotIn`**
* **Boolean: `True`, `False`**
* **Limiting: `findFirst5ByOrderByTimestampDesc()`**
* **Distinct: `findDistinctByLastName(String name)`**

### 4. Key Annotations

* **`@Valid` — Triggers Bean Validation on incoming request DTOs (`@NotNull`, `@Min`, `@Size`, etc.)**
* **`@Transactional` — Place in the  Service layer only . Avoid self-invocation (bypasses Spring AOP).**

### 5. Project Structure

```
com.example.rpa_service
├── controller/dto        → Request / Response DTOs
├── client/.../dto        → Remote service DTOs
├── domain/entity         → JPA Entities
├── domain/type           → Enums / Constants
└── service               → Business logic, DTO ↔ Entity conversion
```

### 6. Avoiding Circular Dependencies

| Scenario                                     | Strategy                                                         |
| -------------------------------------------- | ---------------------------------------------------------------- |
| **Shared logic**                       | **Extract to `CommonService`**                           |
| **Multi-domain orchestration**         | **Create a `Manager`/`Orchestrator`layer**             |
| **$A \leftrightarrow B$ dependency** | **Refactor to a third class or use Event-driven approach** |

### 7. State Machine Pattern

**Encode valid transitions directly on the entity to prevent illegal state changes:**

```java
private static final Map<JobStatus, Set<JobStatus>> ALLOWED_TRANSITIONS = Map.of(
    JobStatus.GENERATED, Set.of(JobStatus.COMPLETED, JobStatus.FAILED)
);

public void transitionTo(JobStatus next) {
    if (!ALLOWED_TRANSITIONS.getOrDefault(this.status, Set.of()).contains(next))
        throw new IllegalStateException("Illegal transition: " + this.status + " -> " + next);
    this.status = next;
    this.updated = LocalDateTime.now();
}
```

### 8. Lazy Loading Heavy Columns

**Split large data into a separate entity and lazy-load it:**

```java
@OneToOne(fetch = FetchType.LAZY, mappedBy = "job", cascade = CascadeType.ALL, orphanRemoval = true)
private JobImgs jobImgs;
```

#### MySQL Text Column Types

| Type                     | Max Size        | Use Case                            |
| ------------------------ | --------------- | ----------------------------------- |
| **`TINYTEXT`**   | **255 B** | **Short strings**             |
| **`TEXT`**       | **64 KB** | **Articles, comments**        |
| **`MEDIUMTEXT`** | **16 MB** | **Large JSON, documents**     |
| **`LONGTEXT`**   | **4 GB**  | **Base64 images, huge blobs** |

### 9. Permission Control (RBAC + ABAC)

* **RBAC — Broad access by role**
* **ABAC — Fine-grained, data-level access**

```java
// RBAC checks the operation permission; ABAC checks resource ownership
@PreAuthorize("hasAuthority('OP_DELETE_DOCUMENT') and @docSecurity.isOwner(authentication, #id)")
public ResponseEntity<Void> deleteDocument(@PathVariable Long id) { ... }
```

**On login, embed resolved permissions (not just roles) into JWT claims and map them to `GrantedAuthority` objects.**

### 10. Backend Optimization

**Optimization priority: Database → Application Logic → JVM Tuning**

#### Database Layer (Highest Impact)

| Problem                           | Optimization                        | Tool                                                  |
| --------------------------------- | ----------------------------------- | ----------------------------------------------------- |
| **Full table scan$O(n)$** | **B-Tree index$O(\log n)$** | **`@Index(columnList = "status")`**           |
| **N+1 queries**             | **Eager fetch via JOIN**      | **`@EntityGraph(attributePaths = {"logs"})`** |
| **Connection overhead**     | **Connection pooling**        | **HikariCP:`maximum-pool-size=20`**           |

**HikariCP pool size formula:**

$$
\text{connections} = (\text{core_count} \times 2) + \text{effective_spindle_count}
$$

#### Application Layer

- **Caching**

```java
@Cacheable("configs")
public String getConfig() { ... }

@CacheEvict(value = "configs", allEntries = true)
public void updateConfig() { ... }  // Evict on write
```

- **Async Processing**

**Return `202 Accepted` immediately; process in background:**

```java
@Async("rpaTaskExecutor")
public CompletableFuture<Void> processJobAsync(Job job) { ... }
```

- **API Efficiency**

```properties
server.compression.enabled=true
server.compression.mime-types=application/json
```

**Use `@JsonView` to exclude unnecessary fields from large response objects.**

#### Infrastructure Layer (JVM & Docker)

```dockerfile
ENTRYPOINT ["java",
  "-Xms512m", "-Xmx512m",   # Equal values prevent heap resize stutters
  "-XX:+UseG1GC",            # Low-latency GC for web backends
  "-XX:+ExitOnOutOfMemoryError",
  "-jar", "app.jar"]
```

* **G1GC — Standard for most web backends (low latency)**
* **ZGC — For heaps 8 GB+ needing sub-millisecond pauses**
* **Ensure Java 8u191+ or 11+ for Docker container memory awareness (`-XX:+UseContainerSupport`)**

#### Monitoring

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

> **Profiling tip: Connect VisualVM locally and use the Sampler tab to find the exact method hogging CPU — often unexpected (e.g. JSON parsing, regex).**

### 11. Async Endpoint

**💡 Core Concept**

* **`@Async`: Tells Spring to run the method in a **separate thread (from a task executor thread pool), freeing up the main Tomcat/Web Servlet thread immediately.**
* **`CompletableFuture`: Acts as a container for the asynchronous result, allowing the asynchronous thread to pass the data back to the controller or chain further non-blocking actions.**

**⚠️ Gotchas / Checklist**

1. **Self-Invocation (Internal Calls) **: **`@Async` works via Spring AOP proxies. If you call an `@Async` method from another method  *inside the same class* **, the async behavior **will be bypassed and it will run synchronously.**
2. **Public Only **: **`@Async` methods must be `public`.**
3. **Exceptions **: Uncaught exceptions inside **`@Async` methods returning `CompletableFuture` are captured by the future itself. Handle them using `.exceptionally()` in the pipeline.**

## Frontend

### 1. REST API

#### HTTP methods and what they mean

**Each method carries semantic meaning that servers, browsers, proxies, and CDNs all understand:**

| Method               | Meaning                        | Body?         | Safe?        | Idempotent?  |
| -------------------- | ------------------------------ | ------------- | ------------ | ------------ |
| **`GET`**    | **Read/fetch**           | **No**  | **✅** | **✅** |
| **`POST`**   | **Create something new** | **Yes** | **❌** | **❌** |
| **`PUT`**    | **Replace entirely**     | **Yes** | **❌** | **✅** |
| **`PATCH`**  | **Partial update**       | **Yes** | **❌** | **❌** |
| **`DELETE`** | **Remove**               | **No**  | **❌** | **✅** |

**Safe means it doesn't change server state — browsers can freely retry it, prefetch it, cache it.**

**Idempotent means calling it 10 times has the same result as calling it once — important for retry logic on network failures.**

#### REST endpoint design best practices

**Use nouns for resources, not verbs**

```
✅ GET    /api/users
✅ POST   /api/users
✅ GET    /api/users/42
✅ PUT    /api/users/42
✅ DELETE /api/users/42

❌ GET    /api/getUsers
❌ POST   /api/createUser
❌ POST   /api/deleteUser/42
```

**The HTTP method IS the verb — don't repeat it in the URL.**

**Use plural nouns consistently**

```
✅ /api/users        not  /api/user
✅ /api/orders       not  /api/order
```

**Nest for relationships, but not too deep**

```
✅ GET  /api/users/42/orders          user's orders
✅ GET  /api/users/42/orders/7        specific order
❌ GET  /api/users/42/orders/7/items/3/reviews    too deep — flatten it
✅ GET  /api/order-items/3/reviews    flattened
```

**Use query params for filtering, sorting, pagination**

```
✅ GET /api/users?role=admin&page=2&limit=20&sort=createdAt
✅ GET /api/orders?status=pending&from=2024-01-01
```

**Use HTTP status codes properly**

```
200 OK              — successful GET, PUT, PATCH
201 Created         — successful POST (include Location header)
204 No Content      — successful DELETE
400 Bad Request     — invalid input / validation failed
401 Unauthorized    — not authenticated
403 Forbidden       — authenticated but not allowed
404 Not Found       — resource doesn't exist
409 Conflict        — e.g. email already registered
422 Unprocessable   — semantic validation error
429 Too Many Requests — rate limited
500 Internal Server Error — something broke on your side
```

#### Spring Boot example following all of the above

```java
@RestController
@RequestMapping("/api/users")
publicclassUserController{

@GetMapping// GET /api/users
publicPage<UserDto>list(
@RequestParam(defaultValue ="0")int page,
@RequestParam(defaultValue ="20")int size){...}

@PostMapping// POST /api/users
@ResponseStatus(HttpStatus.CREATED)
publicResponseEntity<UserDto>create(@Valid@RequestBodyCreateUserRequest req){
UserDto created = userService.create(req);
URI location = URI.create("/api/users/"+ created.getId());
returnResponseEntity.created(location).body(created);
}

@GetMapping("/{id}")// GET /api/users/42
publicUserDtogetById(@PathVariableLong id){...}

@PatchMapping("/{id}")// PATCH /api/users/42
publicUserDtoupdate(@PathVariableLong id,
@Valid@RequestBodyUpdateUserRequest req){...}

@DeleteMapping("/{id}")// DELETE /api/users/42
@ResponseStatus(HttpStatus.NO_CONTENT)
publicvoiddelete(@PathVariableLong id){...}
}
```

### 2. Markdown Report and Print as PDF

**Renders a markdown string in a modal, with a Print PDF button that exports the full content to a multi-page A4 PDF entirely in the browser (no backend).**

- **Key moving parts**

| Piece                               | Why it matters                                                                    |
| ----------------------------------- | --------------------------------------------------------------------------------- |
| **`marked.parse(content)`** | **Converts markdown string → HTML for both display and PDF render**        |
| **`ngx-markdown`**          | **Renders markdown in the modal body with Angular-friendly pipe/component** |
| **`html2canvas`**           | **Captures the off-screen container as a single canvas image**              |
| **`jsPDF`**                 | **Slices that canvas image across A4 pages and saves the file**             |

- **The PDF pipeline (3 steps)**

1. **Create an off-screen div → inject parsed HTML → append to `document.body`**
2. **`html2canvas` captures the div at full `scrollHeight` (not viewport height)**
3. **`jsPDF` tiles the canvas image across pages using a `while (remaining > 0)` loop**

- **Gotchas to remember**
  * **`scrollHeight` not `offsetHeight` — `offsetHeight` only gives the visible portion; you need `scrollHeight` to capture the whole document.**
  * **Images must finish loading before `html2canvas` runs, or they render blank. Always `await waitForImages(container)` first.**
  * **PDF container styles must be GLOBAL (`styles.scss`), not component-scoped. Angular's view encapsulation strips component styles from elements appended to `document.body`.**
  * **`width: auto` on `position: fixed` collapses to near-zero. Always set a fixed pixel width on the PDF container, then read `scrollWidth` after DOM insertion to lock it in.**
  * **JPEG not PNG for file size. PNG is lossless → huge. JPEG at 0.82 quality cuts size by ~10×.**
  * | Setting                       | Approx output            |
    | ----------------------------- | ------------------------ |
    | **PNG + scale 2**       | **100–250 MB ❌** |
    | **JPEG 0.9 + scale 1**  | **20–50 MB ⚠️** |
    | **JPEG 0.82 + scale 1** | **3–10 MB ✅**    |

### 3.Lightweight Table Sorting

**💡 Core Concept**

**A complete, lightweight, reactive data table architecture. It combines stateful column sorting, safe memory-copy sorting via a TypeScript getter, dynamic status-based HTML row styling, and event-propagation guards to handle nested actions smoothly.**

---

**🎨 Template**

**Defines clickable header anchors, renders contextual indicators, applies dynamic conditional classes, and uses control-flow blocks to build a rich data-viewing experience.**

```html
<table>
  <thead>
    <tr>
      <!-- 1. Clickable Sort Header -->
      <th class="clickable-header" (click)="toggleSort('targetProperty')">
        Column Label
        <span class="sort-icon">{{ getSortIcon('targetProperty') }}</span>
      </th>
    </tr>
  </thead>

  <tbody>
    <!-- 2. Iterate Over The Sorted Getter List -->
    @for (item of computedSortedList; track item.uniqueId) {
      <tr class="data-row">
          <!-- Text Cell -->
          <td>{{ item.identifier }}</td>
      </tr>
    }
  </tbody>
</table>
```

**---

**⚙️ Component Logic**

**Handles the internal state, toggle mutations, icon lookups, and shallow-copy array processing.**

```typescript
// ── 1. State Tracking Variables ───────────────────────────────────────────
activeField: PropertyType | null = null;
direction: 'asc' | 'desc' = 'asc';

// ── 2. Click Header Toggle Handler ────────────────────────────────────────
toggleSort(field: PropertyType): void {
  if (this.activeField === field) {
    // If clicking the same column header, invert the direction state
    this.direction = this.direction === 'asc' ? 'desc' : 'asc';
  } else {
    // If switching columns, update the targeted field and default to 'asc'
    this.activeField = field;
    this.direction = 'asc';
  }
}

// ── 3. Visual Icon Evaluator ──────────────────────────────────────────────
getSortIcon(field: PropertyType): string {
  // Return neutral symbol if the column is not currently selected for sorting
  if (this.activeField !== field) return'↕';

  // Return active indicator representing the sort direction
  returnthis.direction === 'asc' ? '↑' : '↓';
}

// ── 4. Reactive Sort Getter ───────────────────────────────────────────────
get computedSortedList(): TargetType[] {
  // Guard Clause: Return unmutated original data stream if sorting is inactive
  if (!this.activeField) returnthis.sourceList;

  const field: PropertyType = this.activeField;
  // Direction multiplier: 1 maps to ascending, -1 maps to descending
  const multiplier: number = this.direction === 'asc' ? 1 : -1;

  // Destructure array [...source] to enforce immutable operations and avoid render loops
  return [...this.sourceList].sort((a, b) => {
    // Nullish coalescing avoids string comparison crashes against null or undefined cells
    const valA = a[field] ?? '';
    const valB = b[field] ?? '';

    // Evaluate text/numeric weight and scale it using the direction multiplier
    return valA < valB ? -multiplier : valA > valB ? multiplier : 0;
  });
}
```

---

**⚠️ Key Architectural Takeaways**

* **Immutability via Shallow Copy (`[...]`)** : Calling `.sort()` directly on a data source inside a template getter mutates the underlying memory array. This triggers Angular's change detection engine indefinitely, forcing an infinite execution loop. Spreading into a shallow copy (`[...this.items]`) isolates modifications completely.
* **Bubble Interception (`$event.stopPropagation()`)** : Indispensable for internal cell buttons. It breaks the HTML click bubble propagation chain, blocking embedded button clicks from bubbling upward and firing row selection handlers (`(click)="handleRowClick(item)"`).
* **Nullish Safeguards (`?? ''`)** : Enforcing fallbacks within the sorting loop ensures uniform sorting weights and eliminates rendering runtime errors when interacting with inconsistent backend datasets.
* **Stateless Templates** : The HTML view engine passes purely hardcoded string representations of properties. The template has no direct knowledge of state changes, maintaining a unidirectional logic flow.
