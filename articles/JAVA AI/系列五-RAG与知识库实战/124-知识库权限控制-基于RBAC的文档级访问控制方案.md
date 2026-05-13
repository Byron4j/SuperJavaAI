# 知识库权限控制：基于 RBAC 的文档级访问控制方案，别让实习生搜到高管会议纪要

> 某互联网公司真实事故：实习生在公司内部 AI 助手里输入"明年战略规划"，结果返回了高管会议中讨论的"Q1 将裁员 15%"的未公开信息。一周后，整个部门被约谈。

---

## 一、开篇：RAG 系统中的权限黑洞

如果你正在把公司内部知识库接入 RAG，你必须直面一个灵魂拷问：

**"你的 AI 助手，比公司的权限系统更懂权限吗？"**

传统的企业权限体系已经非常成熟——ERP 有角色权限、OA 有审批流权限、网盘有文件夹权限。但当这些数据被一股脑丢进向量库后，**所有权限信息全部丢失了**。

向量检索的工作原理是"语义相似度匹配"，它根本不认识什么"权限"、"密级"、"部门隔离"。用户搜什么，它就返回最相似的 Chunk。

这意味着：
- 财务部的同事搜"奖金方案"，可能搜到 HR 尚未公布的调薪文档
- 实习生搜"技术方案"，可能搜到核心算法的实现细节
- 跨部门的同事搜"项目进展"，可能搜到保密项目的竞标策略

**RAG 的权限控制不是"加分项"，而是"准入条件"。** 没有权限控制的知识库 RAG，在公司法务和合规部门眼里就是一颗定时炸弹。

---

## 二、RAG 权限控制的三大挑战

### 2.1 挑战全景

```
┌─────────────────────────────────────────────────────┐
│  挑战 1：文档级权限                                   │
│  不同部门/角色的文档需要隔离，但向量库不原生支持权限      │
├─────────────────────────────────────────────────────┤
│  挑战 2：段落级权限                                   │
│  同一份文档的不同段落可能有不同的密级要求               │
├─────────────────────────────────────────────────────┤
│  挑战 3：检索时权限过滤                               │
│  检索发生在权限判断之前，如何高效过滤无权限的结果？       │
└─────────────────────────────────────────────────────┘
```

传统的做法是"预先过滤"——检索时带上权限条件，只从有权访问的文档中检索。但这在语义检索场景下几乎不可行，因为向量检索不支持复杂的布尔过滤（部分向量库支持，但性能极差）。

**我们的策略：先检索，后过滤。** 这是目前工程实践中性价比最高的方案。

---

## 三、RBAC 权限模型设计

### 3.1 核心实体

```java
@Entity
@Table(name = "rag_users")
public class User {
    
    @Id
    private String id;
    
    private String username;
    private String department;  // 部门
    private String position;    // 职位
    
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(name = "rag_user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<Role> roles;
    
    @ElementCollection
    @CollectionTable(name = "rag_user_security_levels")
    @Column(name = "security_level")
    private Set<String> securityLevels; // ["PUBLIC", "INTERNAL", "CONFIDENTIAL", "TOP_SECRET"]
}

@Entity
@Table(name = "rag_roles")
public class Role {
    
    @Id
    private String id;
    
    private String name;        // ADMIN, HR, ENGINEER, FINANCE, INTERN
    private String description;
    
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(name = "rag_role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id"))
    private Set<Permission> permissions;
}

@Entity
@Table(name = "rag_permissions")
public class Permission {
    
    @Id
    private String id;
    
    private String resource;    // "DOCUMENT", "CHUNK", "QUERY"
    private String action;      // "READ", "WRITE", "DELETE", "SEARCH"
}
```

### 3.2 文档权限元数据

```java
@Entity
@Table(name = "rag_document_permissions")
public class DocumentPermission {
    
    @Id
    private String documentId;
    
    // 密级
    @Enumerated(EnumType.STRING)
    private SecurityLevel securityLevel; // PUBLIC, INTERNAL, CONFIDENTIAL, TOP_SECRET
    
    // 允许访问的部门
    @ElementCollection
    @CollectionTable(name = "rag_doc_allowed_depts")
    @Column(name = "department")
    private Set<String> allowedDepartments;
    
    // 允许访问的角色
    @ElementCollection
    @CollectionTable(name = "rag_doc_allowed_roles")
    @Column(name = "role")
    private Set<String> allowedRoles;
    
    // 允许访问的用户（白名单）
    @ElementCollection
    @CollectionTable(name = "rag_doc_allowed_users")
    @Column(name = "user_id")
    private Set<String> allowedUsers;
    
    // 禁止访问的用户（黑名单）
    @ElementCollection
    @CollectionTable(name = "rag_doc_denied_users")
    @Column(name = "user_id")
    private Set<String> deniedUsers;
    
    // 有效期
    private LocalDateTime accessStartTime;
    private LocalDateTime accessEndTime;
}
```

### 3.3 权限判定逻辑

```java
@Service
public class PermissionEvaluator {
    
    public boolean hasAccess(User user, DocumentPermission docPerm) {
        
        // 1. 黑名单优先：明确拒绝 > 其他规则
        if (docPerm.getDeniedUsers().contains(user.getId())) {
            return false;
        }
        
        // 2. 白名单放行：明确允许 > 通用规则
        if (docPerm.getAllowedUsers().contains(user.getId())) {
            return true;
        }
        
        // 3. 时效性检查
        LocalDateTime now = LocalDateTime.now();
        if (docPerm.getAccessStartTime() != null && now.isBefore(docPerm.getAccessStartTime())) {
            return false;
        }
        if (docPerm.getAccessEndTime() != null && now.isAfter(docPerm.getAccessEndTime())) {
            return false;
        }
        
        // 4. 密级检查：用户密级 >= 文档密级
        if (!hasSufficientSecurityLevel(user, docPerm.getSecurityLevel())) {
            return false;
        }
        
        // 5. 部门检查
        if (!docPerm.getAllowedDepartments().isEmpty() 
            && !docPerm.getAllowedDepartments().contains(user.getDepartment())) {
            return false;
        }
        
        // 6. 角色检查
        if (!docPerm.getAllowedRoles().isEmpty()) {
            Set<String> userRoleNames = user.getRoles().stream()
                .map(Role::getName)
                .collect(Collectors.toSet());
            if (Collections.disjoint(userRoleNames, docPerm.getAllowedRoles())) {
                return false;
            }
        }
        
        return true;
    }
    
    private boolean hasSufficientSecurityLevel(User user, SecurityLevel docLevel) {
        // 密级等级: PUBLIC(0) < INTERNAL(1) < CONFIDENTIAL(2) < TOP_SECRET(3)
        int userMaxLevel = user.getSecurityLevels().stream()
            .mapToInt(this::securityLevelToInt)
            .max()
            .orElse(0);
        
        return userMaxLevel >= securityLevelToInt(docLevel.name());
    }
    
    private int securityLevelToInt(String level) {
        return switch (level) {
            case "TOP_SECRET" -> 3;
            case "CONFIDENTIAL" -> 2;
            case "INTERNAL" -> 1;
            default -> 0;
        };
    }
}
```

---

## 四、检索时的权限过滤核心实现

### 4.1 核心架构：检索 → 权限过滤 → 结果返回

```java
@Service
@Slf4j
public class SecureSearchService {
    
    private final HybridSearchService searchService;
    private final PermissionEvaluator permissionEvaluator;
    private final DocumentPermissionRepository permRepository;
    private final AuditLogService auditLogService;
    
    /**
     * 带权限控制的检索
     */
    public SecureSearchResult secureSearch(SearchRequest request, User user) {
        long startTime = System.currentTimeMillis();
        
        // 步骤1：宽检索（不过滤权限，召回更多结果）
        int overFetch = request.getTopK() * 3; // 多取3倍，给权限过滤留余量
        List<SearchHit> rawResults = searchService.search(
            SearchRequest.builder()
                .query(request.getQuery())
                .topK(overFetch)
                .build()
        );
        
        if (rawResults.isEmpty()) {
            return SecureSearchResult.empty();
        }
        
        // 步骤2：批量加载权限信息
        Map<String, DocumentPermission> permMap = loadPermissions(rawResults);
        
        // 步骤3：权限过滤
        List<SearchHit> filtered = filterByPermission(rawResults, user, permMap);
        
        // 步骤4：截断到目标数量
        List<SearchHit> finalResults = filtered.stream()
            .limit(request.getTopK())
            .toList();
        
        // 步骤5：审计日志
        long latency = System.currentTimeMillis() - startTime;
        auditLogService.logSearch(user, request.getQuery(), 
            finalResults, filtered.size(), rawResults.size(), latency);
        
        return SecureSearchResult.builder()
            .items(finalResults)
            .totalFiltered(filtered.size() - finalResults.size())
            .searchLatency(latency)
            .build();
    }
    
    private Map<String, DocumentPermission> loadPermissions(List<SearchHit> hits) {
        // 提取所有涉及的文档 ID（去重）
        Set<String> documentIds = hits.stream()
            .map(SearchHit::getDocumentId)
            .collect(Collectors.toSet());
        
        // 批量查询 + 缓存
        List<DocumentPermission> permissions = permRepository
            .findByDocumentIdIn(new ArrayList<>(documentIds));
        
        return permissions.stream()
            .collect(Collectors.toMap(
                DocumentPermission::getDocumentId, 
                Function.identity()));
    }
    
    private List<SearchHit> filterByPermission(
            List<SearchHit> hits, 
            User user, 
            Map<String, DocumentPermission> permMap) {
        
        return hits.stream()
            .filter(hit -> {
                DocumentPermission perm = permMap.get(hit.getDocumentId());
                
                if (perm == null) {
                    // 没有权限配置的文档，默认允许访问（或默认拒绝，看策略）
                    log.warn("No permission config for document: {}", hit.getDocumentId());
                    return true; // 宽松策略
                }
                
                boolean allowed = permissionEvaluator.hasAccess(user, perm);
                
                if (!allowed) {
                    log.info("Permission denied: user={}, document={}, reason=RBAC",
                        user.getUsername(), hit.getDocumentId());
                }
                
                return allowed;
            })
            .toList();
    }
}
```

### 4.2 权限缓存优化

```java
@Component
public class PermissionCache {
    
    private final CacheManager cacheManager;
    private Cache<String, DocumentPermission> documentCache;
    private Cache<String, Set<String>> userDocAccessCache;
    
    @PostConstruct
    public void init() {
        this.documentCache = cacheManager.getCache("document-permissions");
        this.userDocAccessCache = cacheManager.getCache("user-doc-access");
    }
    
    /**
     * 获取文档权限（带缓存）
     */
    public DocumentPermission getDocumentPermission(String documentId) {
        return documentCache.get(documentId, key -> 
            permRepository.findByDocumentId(key));
    }
    
    /**
     * 获取用户能访问的文档列表（预计算+缓存）
     */
    public Set<String> getAccessibleDocuments(String userId) {
        return userDocAccessCache.get(userId, key -> {
            User user = userRepository.findById(key).orElseThrow();
            List<DocumentPermission> allPerms = permRepository.findAll();
            
            return allPerms.stream()
                .filter(perm -> permissionEvaluator.hasAccess(user, perm))
                .map(DocumentPermission::getDocumentId)
                .collect(Collectors.toSet());
        });
    }
    
    /**
     * 权限变更时清除缓存
     */
    @CacheEvict(value = {"document-permissions", "user-doc-access"}, allEntries = true)
    public void evictAll() {
        log.info("Permission cache evicted due to permission change");
    }
}
```

---

## 五、元数据 + 权限联合过滤

### 5.1 在 Elasticsearch 中实现权限预过滤

```java
@Service
public class ElasticsearchSecureSearchService {
    
    private final RestHighLevelClient esClient;
    
    public List<SearchHit> secureVectorSearch(
            float[] queryEmbedding, User user, int topK) {
        
        // 构建用户权限过滤条件
        BoolQueryBuilder permissionFilter = buildPermissionFilter(user);
        
        // 向量相似度查询（script_score）
        ScriptScoreQueryBuilder vectorQuery = QueryBuilders.scriptScoreQuery(
            QueryBuilders.matchAllQuery(),
            new ScriptScoreQueryBuilder.Script(
                ScriptType.INLINE, "painless",
                "cosineSimilarity(params.query_vector, 'embedding') + 1.0",
                Map.of("query_vector", floatArrayToDoubleList(queryEmbedding))
            )
        );
        
        // 组合：向量相似度 + 权限过滤
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery()
            .must(vectorQuery)
            .filter(permissionFilter); // 权限作为 filter（不参与评分，但必须满足）
        
        SearchRequest searchRequest = new SearchRequest("rag_chunks");
        searchRequest.source(new SearchSourceBuilder()
            .query(boolQuery)
            .size(topK));
        
        try {
            SearchResponse response = esClient.search(searchRequest, RequestOptions.DEFAULT);
            return mapToSearchHits(response);
        } catch (IOException e) {
            throw new SearchException("Vector search failed", e);
        }
    }
    
    private BoolQueryBuilder buildPermissionFilter(User user) {
        BoolQueryBuilder filter = QueryBuilders.boolQuery();
        
        // 条件1：公开文档（所有人可见）
        filter.should(QueryBuilders.termQuery(
            "metadata.security_level", "PUBLIC"));
        
        // 条件2：用户所在部门允许的文档
        filter.should(QueryBuilders.termQuery(
            "metadata.allowed_departments", user.getDepartment()));
        
        // 条件3：用户角色允许的文档
        for (Role role : user.getRoles()) {
            filter.should(QueryBuilders.termQuery(
                "metadata.allowed_roles", role.getName()));
        }
        
        // 条件4：白名单用户
        filter.should(QueryBuilders.termQuery(
            "metadata.allowed_users", user.getId()));
        
        // 排除黑名单
        filter.mustNot(QueryBuilders.termQuery(
            "metadata.denied_users", user.getId()));
        
        // 密级过滤：排除超出用户密级的文档
        int userMaxLevel = getMaxSecurityLevel(user);
        List<String> exceedLevels = getLevelsAbove(userMaxLevel);
        if (!exceedLevels.isEmpty()) {
            filter.mustNot(QueryBuilders.termsQuery(
                "metadata.security_level", exceedLevels));
        }
        
        return filter;
    }
}
```

### 5.2 文档注入时写入权限元数据

```java
@Component
public class SecureDocumentIngestionService {
    
    private final VectorStore vectorStore;
    private final DocumentPermissionRepository permRepository;
    
    public void ingestWithPermission(RawDocument raw, DocumentPermission perm) {
        // 1. 保存权限配置
        permRepository.save(perm);
        
        // 2. 预处理
        ProcessedDocument processed = pipeline.execute(raw);
        
        // 3. 分块
        List<Chunk> chunks = chunker.split(processed.getContent());
        
        // 4. 向量化，同时注入权限元数据
        Map<String, Object> metadata = new HashMap<>(processed.getMetadata());
        metadata.put("security_level", perm.getSecurityLevel().name());
        metadata.put("allowed_departments", perm.getAllowedDepartments());
        metadata.put("allowed_roles", perm.getAllowedRoles());
        metadata.put("allowed_users", perm.getAllowedUsers());
        metadata.put("denied_users", perm.getDeniedUsers());
        
        // 5. 写入向量库（权限信息作为元数据一起存储）
        List<float[]> embeddings = embeddingService.embedBatch(chunks);
        vectorStore.insertBatch(chunks, embeddings, metadata);
    }
}
```

---

## 六、段落级敏感词过滤

### 6.1 场景

有时候同一份文档内，不同段落的密级也不同。比如：
- 项目概述 → PUBLIC
- 技术架构 → INTERNAL  
- 核心算法实现 → CONFIDENTIAL
- 性能 benchmark 数据 → TOP_SECRET

### 6.2 段落级权限标记

```java
@Component
public class ParagraphLevelSecurityProcessor implements DocumentProcessor {
    
    private final SensitiveWordDetector detector;
    
    @Override
    public DocumentContext process(DocumentContext context) {
        String content = context.getContent();
        String[] paragraphs = content.split("\n\n");
        
        List<SecureChunk> secureChunks = new ArrayList<>();
        
        for (int i = 0; i < paragraphs.length; i++) {
            String paragraph = paragraphs[i].trim();
            if (paragraph.isEmpty()) continue;
            
            // 检测段落中是否包含敏感词
            List<SensitiveWordHit> hits = detector.detect(paragraph);
            
            int paragraphLevel = 0; // 默认 PUBLIC
            for (SensitiveWordHit hit : hits) {
                // 敏感词映射到密级
                int wordLevel = mapSensitiveWordToLevel(hit);
                paragraphLevel = Math.max(paragraphLevel, wordLevel);
            }
            
            secureChunks.add(new SecureChunk(
                paragraph, 
                SecurityLevel.fromInt(paragraphLevel),
                hits
            ));
        }
        
        context.setSecureChunks(secureChunks);
        return context;
    }
}

@Component
public class SensitiveWordDetector {
    
    private final AhoCorasickMatcher matcher;
    
    @PostConstruct
    public void init() {
        // 从配置或数据库加载敏感词库
        List<SensitiveWordConfig> words = sensitiveWordRepository.findAll();
        
        List<String> patterns = new ArrayList<>();
        Map<String, Integer> levelMap = new HashMap<>();
        
        for (SensitiveWordConfig word : words) {
            patterns.add(word.getPattern());
            levelMap.put(word.getPattern(), word.getSecurityLevel());
        }
        
        this.matcher = new AhoCorasickMatcher(patterns);
    }
    
    public List<SensitiveWordHit> detect(String text) {
        return matcher.search(text).stream()
            .map(match -> SensitiveWordHit.builder()
                .word(match.getPattern())
                .position(match.getStart())
                .build())
            .toList();
    }
}
```

---

## 七、审计日志

### 7.1 审计日志实体

```java
@Entity
@Table(name = "rag_audit_logs")
public class AuditLog {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;
    
    @Column(nullable = false)
    private String userId;
    
    @Column(nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String department;
    
    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private AuditAction action; // SEARCH, VIEW_DOCUMENT, PERMISSION_DENIED, DOWNLOAD
    
    @Column(length = 2000)
    private String query;          // 搜索关键词
    
    @Column(columnDefinition = "TEXT")
    private String resultDocIds;   // JSON: 返回的文档 ID 列表
    
    @Column(columnDefinition = "TEXT")
    private String filteredDocIds; // JSON: 被权限过滤掉的文档 ID
    
    private Integer totalResults;
    private Integer filteredCount;
    private Long latencyMs;
    
    private String ipAddress;
    private String userAgent;
    
    @Column(nullable = false)
    private LocalDateTime timestamp;
    
    @Column(columnDefinition = "JSONB")
    private String extra;          // 扩展字段
}
```

### 7.2 审计日志服务

```java
@Service
@Slf4j
public class AuditLogService {
    
    private final AuditLogRepository auditLogRepository;
    
    @Async
    public void logSearch(User user, String query, 
                           List<SearchHit> results,
                           int totalFiltered, 
                           int totalRaw, 
                           long latency) {
        
        List<String> resultDocIds = results.stream()
            .map(SearchHit::getDocumentId)
            .toList();
        
        AuditLog auditLog = AuditLog.builder()
            .userId(user.getId())
            .username(user.getUsername())
            .department(user.getDepartment())
            .action(AuditAction.SEARCH)
            .query(query)
            .resultDocIds(toJson(resultDocIds))
            .totalResults(results.size())
            .filteredCount(totalFiltered)
            .totalRaw(totalRaw)
            .latencyMs(latency)
            .timestamp(LocalDateTime.now())
            .build();
        
        auditLogRepository.save(auditLog);
    }
    
    @Async
    public void logPermissionDenied(User user, String query, String documentId, String reason) {
        AuditLog auditLog = AuditLog.builder()
            .userId(user.getId())
            .username(user.getUsername())
            .action(AuditAction.PERMISSION_DENIED)
            .query(query)
            .extra(toJson(Map.of(
                "document_id", documentId,
                "reason", reason
            )))
            .timestamp(LocalDateTime.now())
            .build();
        
        auditLogRepository.save(auditLog);
        
        // 频繁权限拒绝告警
        checkPermissionDenialThreshold(user);
    }
    
    private void checkPermissionDenialThreshold(User user) {
        long recentDenials = auditLogRepository.countByUserIdAndActionAndTimestampAfter(
            user.getId(), 
            AuditAction.PERMISSION_DENIED,
            LocalDateTime.now().minusHours(1));
        
        if (recentDenials > 50) {
            log.warn("User {} has {} permission denials in the last hour, "
                + "possible unauthorized access attempt",
                user.getUsername(), recentDenials);
            // 触发告警
        }
    }
}
```

---

## 八、Spring Security 集成

### 8.1 安全配置

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class RAGSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers("/api/v1/rag/**").authenticated()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(customJwtConverter()))
            )
            .addFilterBefore(rateLimitFilter(), 
                BearerTokenAuthenticationFilter.class)
            .build();
    }
    
    @Bean
    public JwtAuthenticationConverter customJwtConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        
        JwtGrantedAuthoritiesConverter authoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        authoritiesConverter.setAuthoritiesClaimName("roles");
        
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            // 从 JWT 中提取用户完整信息（部门、密级等）
            Collection<GrantedAuthority> authorities = 
                authoritiesConverter.convert(jwt);
            
            // 将用户信息存入 SecurityContext
            RagUserDetails userDetails = RagUserDetails.builder()
                .userId(jwt.getSubject())
                .username(jwt.getClaimAsString("preferred_username"))
                .department(jwt.getClaimAsString("department"))
                .securityLevels(jwt.getClaimAsStringList("security_levels"))
                .authorities(authorities)
                .build();
            
            return new RagAuthenticationToken(userDetails, authorities);
        });
        
        return converter;
    }
}

public class RagAuthenticationToken extends JwtAuthenticationToken {
    
    private final RagUserDetails userDetails;
    
    public RagUserDetails getRagUserDetails() {
        return userDetails;
    }
}
```

### 8.2 在 Controller 中获取用户信息

```java
@RestController
@RequestMapping("/api/v1/rag")
public class SecureRAGController {
    
    @Autowired
    private SecureSearchService searchService;
    
    @GetMapping("/search")
    public SecureSearchResult search(
            @RequestParam String query,
            @RequestParam(defaultValue = "10") int topK,
            Authentication authentication) {
        
        RagUserDetails userDetails = ((RagAuthenticationToken) authentication)
            .getRagUserDetails();
        
        User user = User.builder()
            .id(userDetails.getUserId())
            .username(userDetails.getUsername())
            .department(userDetails.getDepartment())
            .securityLevels(userDetails.getSecurityLevels())
            .build();
        
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(topK)
            .build();
        
        return searchService.secureSearch(request, user);
    }
}
```

---

## 九、性能优化

### 9.1 权限过滤的性能影响

```
无权限过滤: 10ms (纯向量检索)
+ 权限预过滤(ES filter): 15ms (ES 层面过滤，增加50%)
+ 应用层过滤: 25ms (查询权限表+逐条判断，增加150%)
+ 缓存命中: 12ms (权限缓存预热后，接近无过滤性能)
```

**优化策略：**

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .maximumSize(10000)
            .recordStats());
        cacheManager.setCacheNames(Arrays.asList(
            "document-permissions",
            "user-doc-access",
            "user-roles"
        ));
        return cacheManager;
    }
}

@Component
public class PermissionPreloader {
    
    /**
     * 系统启动时预热高频用户的权限数据
     */
    @EventListener(ApplicationReadyEvent.class)
    public void preload() {
        List<User> activeUsers = userRepository.findActiveUsers(1000);
        
        for (User user : activeUsers) {
            CompletableFuture.runAsync(() -> 
                permissionCache.getAccessibleDocuments(user.getId()));
        }
        
        log.info("Preloaded permissions for {} active users", activeUsers.size());
    }
}
```

### 9.2 权限判定批量优化

```java
public class BatchPermissionEvaluator {
    
    /**
     * 批量权限判定（一次数据库查询，多次内存判定）
     */
    public Map<String, Boolean> batchEvaluate(User user, List<String> documentIds) {
        // 一次查询所有相关权限
        List<DocumentPermission> permissions = permRepository
            .findByDocumentIdIn(documentIds);
        
        Map<String, DocumentPermission> permMap = permissions.stream()
            .collect(Collectors.toMap(DocumentPermission::getDocumentId, Function.identity()));
        
        // 内存中逐条判定
        Map<String, Boolean> result = new HashMap<>();
        for (String docId : documentIds) {
            DocumentPermission perm = permMap.get(docId);
            result.put(docId, perm == null || permissionEvaluator.hasAccess(user, perm));
        }
        
        return result;
    }
}
```

---

## 十、完整方案总结

### 10.1 权限控制全景

```
用户请求                       检索层                     存储层
   │                            │                         │
   ▼                            ▼                         ▼
┌──────┐  JWT认证   ┌────────────┐  宽检索   ┌──────────────┐
│ 用户  │─────────▶│ Spring      │─────────▶│ Elasticsearch │
│      │          │ Security    │  topK*3   │ 向量检索      │
└──────┘          └────────────┘           └──────┬───────┘
                      │                           │
                      ▼                           ▼
               ┌────────────┐           ┌──────────────────┐
               │ 权限过滤     │◀──────────│ 检索结果(含权限    │
               │ (内存判定)    │          │ 元数据)           │
               └──────┬─────┘           └──────────────────┘
                      │
                      ▼
               ┌────────────┐
               │ 审计日志     │
               │ (异步写入)   │
               └──────┬─────┘
                      │
                      ▼
               ┌────────────┐
               │ 返回结果给   │
               │ 用户         │
               └────────────┘
```

### 10.2 安全矩阵

| 防护层级 | 机制 | 说明 |
|---------|------|------|
| 传输层 | HTTPS/TLS | 加密传输 |
| 认证层 | JWT + OAuth2 | 用户身份认证 |
| 授权层 | RBAC + ABAC | 角色+属性双维度权限 |
| 数据层 | 密级标签 | 文档级/段落级密级控制 |
| 检索层 | 权限过滤 | 先检索后过滤或ES预过滤 |
| 审计层 | 全量日志 | 所有操作可追溯 |

### 10.3 最后的话

RAG 系统的权限控制，本质上是在"检索效率"和"安全合规"之间找平衡。我给出的这套方案的核心思想是：

1. **宽松检索，严格过滤：** 检索阶段不带权限条件，召回更多候选，然后在应用层严格过滤
2. **权限元数据化：** 把权限信息作为 Chunk 的元数据一起存入向量库，检索结果自包含权限信息
3. **缓存是性能的关键：** 权限数据变化不频繁，合理使用缓存可以将权限判定的成本降到几乎为零
4. **审计是合规的底线：** 谁在什么时候搜了什么、看到了什么、被拒绝了什么——全部记录

最后，我想说一句可能有点扎心但非常现实的话：

**如果你的 RAG 系统还没上线，请在 Day 1 就把权限控制做进去。权限这件事，从零到有的成本是线性增长的，但从有到无的代价是指数级的。**

---

**往期回顾：**
- [企业级 RAG 系统架构设计：数据注入→检索→生成的全链路]()
- [知识库数据预处理管道：PDF/Word/HTML 到 Markdown 的工程实践]()
- [知识库增量更新方案：实时同步企业文档变更的流水线设计]()

---

**下篇预告：** RAG 系列的基础设施篇到此告一段落。但从下一篇文章开始，我们将进入 RAG 的**效果优化深水区**——当基础架构搭建完成后，如何通过 Query Rewriting（查询重写）、HyDE（假设性文档嵌入）、Multi-Hop Retrieval（多跳检索）、Self-RAG 等前沿技术，让 RAG 的回答准确率从 70% 提升到 95%+？效果优化才是 RAG 最难的那部分，关注我，我们下期见！

---

> 权限控制是 RAG 系统上生产前的最后一道安全阀门。如果你觉得这篇文章有用，请点赞、收藏、转发，让更多团队避免"实习生搜到高管机密"的惨剧。
