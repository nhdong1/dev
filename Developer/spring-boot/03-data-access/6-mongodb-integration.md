# Spring Data MongoDB — Tích Hợp Cơ Sở Dữ Liệu Tài Liệu

> MongoDB là document database (cơ sở dữ liệu tài liệu) lưu dữ liệu dưới dạng JSON-like BSON documents.
> Phù hợp cho dữ liệu phi cấu trúc, schema linh hoạt, và các use case cần scalability cao theo chiều ngang.
> Spring Data MongoDB cung cấp Repository abstraction tương tự JPA nhưng cho MongoDB.

---

## 📋 Mục Tiêu

- [ ] Cấu hình **Spring Data MongoDB** và kết nối với MongoDB
- [ ] Định nghĩa **Document** với `@Document`, `@Field`, `@Id`
- [ ] Sử dụng **`MongoRepository`** — CRUD và Derived Queries
- [ ] Thao tác linh hoạt với **`MongoTemplate`**
- [ ] Xây dựng **Aggregation Pipeline** (Đường Ống Tổng Hợp) phức tạp
- [ ] Tạo **Text Search** (Tìm Kiếm Văn Bản) và **Geospatial Queries** (Truy Vấn Địa Lý)
- [ ] Hiểu khi nào dùng MongoDB thay vì PostgreSQL

---

## 1. Cấu Hình

### Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGO_URI:mongodb://localhost:27017/mydb}
      # Hoặc cấu hình riêng từng thành phần:
      host: localhost
      port: 27017
      database: mydb
      username: ${MONGO_USER:}
      password: ${MONGO_PASS:}
      authentication-database: admin
      auto-index-creation: true  # Tự tạo index từ @Indexed — production nên tắt

  # Connection pool (Bể Kết Nối)
  # Cấu hình qua MongoClientSettings bean
```

### MongoConfig — Cấu Hình Nâng Cao

```java
@Configuration
public class MongoConfig extends AbstractMongoClientConfiguration {

    @Value("${spring.data.mongodb.uri}")
    private String mongoUri;

    @Override
    protected String getDatabaseName() {
        return "mydb";
    }

    @Override
    public MongoClient mongoClient() {
        MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(new ConnectionString(mongoUri))
            .applyToConnectionPoolSettings(builder ->
                builder.maxSize(50)                     // Max connections
                       .minSize(5)                      // Min idle connections
                       .maxWaitTime(2, TimeUnit.SECONDS)
                       .maxConnectionLifeTime(30, TimeUnit.MINUTES)
            )
            .applyToSocketSettings(builder ->
                builder.connectTimeout(5, TimeUnit.SECONDS)
                       .readTimeout(10, TimeUnit.SECONDS)
            )
            .build();
        return MongoClients.create(settings);
    }

    // Custom converters — Java ↔ MongoDB type mapping
    @Override
    protected void configureConverters(MongoCustomConversions.MongoConverterConfigurationAdapter adapter) {
        adapter.registerConverter(new MoneyWriteConverter());
        adapter.registerConverter(new MoneyReadConverter());
    }
}
```

---

## 2. Document — Định Nghĩa Tài Liệu MongoDB

```java
@Document(collection = "products")   // Tên collection (tương đương table trong SQL)
public class ProductDocument {

    @Id                               // MongoDB ObjectId (_id field)
    private String id;                // String hoặc ObjectId

    @Field("product_name")            // Tên field trong MongoDB (mặc định: tên field Java)
    private String name;

    @Indexed(unique = true)           // Tạo index unique
    private String sku;

    private BigDecimal price;

    @Field("category")
    private String categoryName;

    // Nested document — không cần annotation riêng
    private ProductDetails details;

    // Array của string
    @Field("tags")
    private List<String> tags;

    // Array của nested documents
    @Field("variants")
    private List<ProductVariant> variants;

    @Indexed                          // Index thường (không unique)
    private LocalDateTime createdAt;

    @TextIndexed                      // Full-text search index
    private String description;

    // Compound index — khai báo trên class
    @CompoundIndex(def = "{'category': 1, 'price': -1}",
                   name = "idx_category_price")
    // 1 = ascending, -1 = descending
    public static class Indexes { }
}

// Nested document class — không cần @Document
public class ProductDetails {
    private String brand;
    private String material;
    private Double weight;
    private Map<String, String> specifications;  // Dynamic key-value
}

// Variant document
public class ProductVariant {
    private String color;
    private String size;
    private int stockQuantity;
    private BigDecimal additionalPrice;
}
```

### Annotations Quan Trọng

```
@Document(collection = "...")     — Ánh xạ class với collection
@Id                               — _id field (ObjectId hoặc String)
@Field("field_name")              — Đặt tên field trong MongoDB
@Indexed                          — Tạo single-field index
@Indexed(unique = true)           — Unique index
@CompoundIndex                    — Index nhiều field
@TextIndexed                      — Full-text search index
@GeoSpatialIndexed                — Geospatial index (2dsphere)
@DBRef                            — Reference đến document khác (như FK)
@Transient                        — Không persist field này
@Version                          — Optimistic locking
```

---

## 3. MongoRepository — Repository Cơ Bản

```java
// Repository — tương tự JpaRepository
public interface ProductRepository extends MongoRepository<ProductDocument, String> {

    // Derived Query Methods — tương tự Spring Data JPA
    Optional<ProductDocument> findBySku(String sku);

    List<ProductDocument> findByCategoryName(String category);

    List<ProductDocument> findByPriceBetween(BigDecimal min, BigDecimal max);

    List<ProductDocument> findByTagsContaining(String tag);

    List<ProductDocument> findByNameContainingIgnoreCase(String keyword);

    // Phân trang
    Page<ProductDocument> findByCategoryName(String category, Pageable pageable);

    // Sắp xếp
    List<ProductDocument> findByPriceGreaterThanOrderByPriceAsc(BigDecimal minPrice);

    long countByCategoryName(String category);

    void deleteBySku(String sku);

    // Nested field query — dùng dấu chấm hoặc camelCase
    List<ProductDocument> findByDetailssBrand(String brand);
    // → Tìm documents có details.brand = ?

    // Trong @Query dùng dot notation của MongoDB
    @Query("{'details.brand': ?0}")
    List<ProductDocument> findByBrand(String brand);

    // MongoQuery với JSON
    @Query("{'price': {$gte: ?0, $lte: ?1}, 'tags': {$in: ?2}}")
    List<ProductDocument> findByPriceRangeAndTags(
        BigDecimal min,
        BigDecimal max,
        List<String> tags
    );

    // Pagination với @Query
    @Query(value = "{'categoryName': ?0}",
           fields = "{'name': 1, 'price': 1, 'sku': 1}")  // Projection — chỉ lấy 3 fields
    Page<ProductDocument> findProjectedByCategory(String category, Pageable pageable);
}
```

---

## 4. MongoTemplate — Thao Tác Linh Hoạt

`MongoTemplate` cho phép xây dựng query động phức tạp hơn Repository.

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final MongoTemplate mongoTemplate;

    // Query động với nhiều điều kiện tùy chọn
    public List<ProductDocument> search(ProductSearchRequest request) {
        Query query = new Query();
        List<Criteria> criteriaList = new ArrayList<>();

        if (request.getKeyword() != null && !request.getKeyword().isBlank()) {
            criteriaList.add(Criteria.where("name")
                .regex(request.getKeyword(), "i")); // "i" = case insensitive
        }

        if (request.getCategory() != null) {
            criteriaList.add(Criteria.where("categoryName").is(request.getCategory()));
        }

        if (request.getMinPrice() != null || request.getMaxPrice() != null) {
            Criteria priceCriteria = Criteria.where("price");
            if (request.getMinPrice() != null) priceCriteria.gte(request.getMinPrice());
            if (request.getMaxPrice() != null) priceCriteria.lte(request.getMaxPrice());
            criteriaList.add(priceCriteria);
        }

        if (request.getTags() != null && !request.getTags().isEmpty()) {
            criteriaList.add(Criteria.where("tags").in(request.getTags()));
        }

        if (!criteriaList.isEmpty()) {
            query.addCriteria(new Criteria().andOperator(criteriaList.toArray(new Criteria[0])));
        }

        // Sắp xếp và phân trang
        query.with(Sort.by(Sort.Direction.DESC, "createdAt"));
        query.skip((long) request.getPage() * request.getSize());
        query.limit(request.getSize());

        // Projection — chỉ lấy fields cần thiết
        query.fields().include("name", "price", "sku", "categoryName", "tags");

        return mongoTemplate.find(query, ProductDocument.class);
    }

    // Update một số field (partial update — không thay toàn bộ document)
    public void updatePrice(String productId, BigDecimal newPrice) {
        Query query = new Query(Criteria.where("id").is(productId));
        Update update = new Update()
            .set("price", newPrice)
            .set("updatedAt", LocalDateTime.now())
            .inc("updateCount", 1);  // Tăng counter

        UpdateResult result = mongoTemplate.updateFirst(query, update, ProductDocument.class);
        if (result.getMatchedCount() == 0) {
            throw new ProductNotFoundException(productId);
        }
    }

    // Bulk update — cập nhật nhiều documents
    public long updateCategoryProducts(String oldCategory, String newCategory) {
        Query query = new Query(Criteria.where("categoryName").is(oldCategory));
        Update update = new Update().set("categoryName", newCategory);
        UpdateResult result = mongoTemplate.updateMulti(query, update, ProductDocument.class);
        return result.getModifiedCount();
    }

    // Upsert — insert nếu chưa có, update nếu đã có
    public void upsertProduct(ProductDocument product) {
        Query query = new Query(Criteria.where("sku").is(product.getSku()));
        Update update = new Update()
            .set("name", product.getName())
            .set("price", product.getPrice())
            .setOnInsert("createdAt", LocalDateTime.now())  // Chỉ set khi insert
            .set("updatedAt", LocalDateTime.now());

        mongoTemplate.upsert(query, update, ProductDocument.class);
    }

    // findAndModify — atomic find + update, trả về document
    public ProductDocument decreaseStock(String productId, int quantity) {
        Query query = new Query(Criteria.where("id").is(productId)
            .and("stockQuantity").gte(quantity));
        Update update = new Update().inc("stockQuantity", -quantity);

        FindAndModifyOptions options = FindAndModifyOptions.options()
            .returnNew(true)   // Trả về document SAU khi update
            .upsert(false);

        ProductDocument updated = mongoTemplate.findAndModify(
            query, update, options, ProductDocument.class);

        if (updated == null) {
            throw new InsufficientStockException(productId, quantity);
        }
        return updated;
    }
}
```

---

## 5. Aggregation Pipeline — Đường Ống Tổng Hợp

Aggregation Pipeline xử lý documents qua nhiều bước (stages), mạnh hơn SQL GROUP BY.

```java
@Service
@RequiredArgsConstructor
public class ProductAnalyticsService {

    private final MongoTemplate mongoTemplate;

    // Thống kê doanh thu theo danh mục
    public List<CategoryRevenue> getRevenueByCategory(LocalDateTime from, LocalDateTime to) {

        MatchOperation matchStage = Aggregation.match(
            Criteria.where("status").is("COMPLETED")
                    .and("createdAt").gte(from).lte(to)
        );

        // Unwind — phân tách array thành nhiều documents riêng
        UnwindOperation unwindStage = Aggregation.unwind("items");

        // Lookup — tương tự LEFT JOIN với collection products
        LookupOperation lookupStage = LookupOperation.newLookup()
            .from("products")                    // Collection cần join
            .localField("items.productId")       // Field trong collection hiện tại
            .foreignField("_id")                 // Field trong collection products
            .as("productInfo");                  // Tên field kết quả

        UnwindOperation unwindProduct = Aggregation.unwind("productInfo", true);
        // true = preserveNullAndEmptyArrays — giữ document dù không match

        // Group — gom nhóm và tính toán
        GroupOperation groupStage = Aggregation.group("productInfo.categoryName")
            .count().as("orderCount")
            .sum("items.total").as("totalRevenue")
            .avg("items.total").as("avgOrderValue")
            .addToSet("items.productId").as("uniqueProducts");

        // Project — định hình output
        ProjectionOperation projectStage = Aggregation.project()
            .andExpression("_id").as("categoryName")
            .and("orderCount").as("orderCount")
            .and("totalRevenue").as("totalRevenue")
            .and("avgOrderValue").as("avgOrderValue")
            .and(ArrayOperators.Size.lengthOfArray("uniqueProducts")).as("uniqueProductCount")
            .andExclude("_id");

        // Sort — sắp xếp kết quả
        SortOperation sortStage = Aggregation.sort(Sort.by(Sort.Direction.DESC, "totalRevenue"));

        // Limit — giới hạn kết quả
        LimitOperation limitStage = Aggregation.limit(10);

        Aggregation aggregation = Aggregation.newAggregation(
            matchStage,
            unwindStage,
            lookupStage,
            unwindProduct,
            groupStage,
            projectStage,
            sortStage,
            limitStage
        );

        AggregationResults<CategoryRevenue> results = mongoTemplate.aggregate(
            aggregation, "orders", CategoryRevenue.class);

        return results.getMappedResults();
    }

    // Bucket — phân loại theo khoảng giá
    public List<PriceBucket> getPriceDistribution() {
        BucketOperation bucketStage = Aggregation.bucket("price")
            .withBoundaries(0, 100, 500, 1000, 5000, Integer.MAX_VALUE)
            .withDefaultBucket("Luxury")
            .andOutputCount().as("count")
            .andOutput("name").push().as("productNames");

        Aggregation aggregation = Aggregation.newAggregation(bucketStage);
        return mongoTemplate.aggregate(aggregation, "products", PriceBucket.class)
                            .getMappedResults();
    }

    // FacetedSearch (Tìm Kiếm Đa Chiều) — kết quả + thống kê cùng lúc
    public FacetedSearchResult facetedSearch(String keyword, String category) {
        MatchOperation matchStage = Aggregation.match(
            Criteria.where("name").regex(keyword, "i")
        );

        FacetOperation facetStage = Aggregation.facet()
            .and(
                // Facet 1: Phân loại theo category
                Aggregation.group("categoryName").count().as("count"),
                Aggregation.sort(Sort.Direction.DESC, "count"),
                Aggregation.limit(10)
            ).as("categoryCounts")
            .and(
                // Facet 2: Phân loại theo khoảng giá
                Aggregation.bucket("price")
                    .withBoundaries(0, 100, 500, 1000)
                    .andOutputCount().as("count")
            ).as("priceRanges")
            .and(
                // Facet 3: Danh sách sản phẩm (phân trang)
                Aggregation.sort(Sort.Direction.ASC, "price"),
                Aggregation.skip(0L),
                Aggregation.limit(20)
            ).as("products");

        Aggregation aggregation = Aggregation.newAggregation(matchStage, facetStage);
        return mongoTemplate.aggregate(aggregation, "products", FacetedSearchResult.class)
                            .getUniqueMappedResult();
    }
}

// Result DTOs
public record CategoryRevenue(
    String categoryName,
    long orderCount,
    BigDecimal totalRevenue,
    BigDecimal avgOrderValue,
    int uniqueProductCount
) {}
```

---

## 6. Text Search — Tìm Kiếm Toàn Văn Bản

```java
// Tạo text index trên Document
@Document(collection = "articles")
@TextIndexed(weight = 2)           // Đặt weight cho toàn document (ít dùng)
public class ArticleDocument {

    @Id private String id;

    @TextIndexed(weight = 3)        // Weight cao → ưu tiên khi rank kết quả
    private String title;

    @TextIndexed(weight = 2)
    private String summary;

    @TextIndexed(weight = 1)
    private String content;

    private List<String> tags;
    private LocalDateTime publishedAt;
}

// Repository với text search
public interface ArticleRepository extends MongoRepository<ArticleDocument, String> {

    // Full-text search
    @Query("{ $text: { $search: ?0 } }")
    List<ArticleDocument> searchByText(String searchText);

    // Text search với score và pagination
    @Query(value = "{ $text: { $search: ?0 } }",
           sort = "{ score: { $meta: 'textScore' } }")
    Page<ArticleDocument> searchByTextWithScore(String searchText, Pageable pageable);
}

// Sử dụng với TextCriteria qua MongoTemplate
@Service
public class ArticleSearchService {

    public Page<ArticleDocument> search(String keyword, Pageable pageable) {
        TextCriteria textCriteria = TextCriteria.forDefaultLanguage()
            .matchingAny(keyword)         // Tìm bất kỳ từ nào
            .caseSensitive(false);

        Query query = TextQuery.queryText(textCriteria)
            .sortByScore()               // Sắp xếp theo relevance score
            .with(pageable);

        List<ArticleDocument> results = mongoTemplate.find(query, ArticleDocument.class);
        long count = mongoTemplate.count(query, ArticleDocument.class);

        return new PageImpl<>(results, pageable, count);
    }
}
```

---

## 7. Geospatial Queries — Truy Vấn Địa Lý

```java
@Document(collection = "stores")
public class StoreDocument {

    @Id private String id;
    private String name;
    private String address;

    // GeoJSON Point — [longitude, latitude]
    @GeoSpatialIndexed(type = GeoSpatialIndexType.GEO_2DSPHERE)
    private GeoJsonPoint location;

    private List<String> categories;
    private boolean isOpen;
}

@Service
public class StoreLocationService {

    private final MongoTemplate mongoTemplate;

    // Tìm stores trong bán kính (Near query)
    public List<StoreDocument> findNearby(double longitude, double latitude, double radiusKm) {
        Point center = new Point(longitude, latitude);
        Distance radius = new Distance(radiusKm, Metrics.KILOMETERS);

        Query query = new Query(
            Criteria.where("location")
                    .nearSphere(center)           // nearSphere — dùng với 2dsphere index
                    .maxDistance(radius.getNormalizedValue())
                    .and("isOpen").is(true)
        );

        query.with(Sort.by("location")); // Sắp xếp theo khoảng cách (gần nhất trước)
        query.limit(20);

        return mongoTemplate.find(query, StoreDocument.class);
    }

    // Tìm stores trong bounding box (Geo Within Box)
    public List<StoreDocument> findInArea(
        double swLon, double swLat,  // Góc tây nam (southwest)
        double neLon, double neLat   // Góc đông bắc (northeast)
    ) {
        Box box = new Box(new Point(swLon, swLat), new Point(neLon, neLat));
        Query query = new Query(Criteria.where("location").within(box));
        return mongoTemplate.find(query, StoreDocument.class);
    }
}
```

---

## 8. Transactions — Giao Dịch Trong MongoDB

MongoDB hỗ trợ multi-document transactions từ phiên bản 4.0 (chỉ với Replica Set hoặc Sharded Cluster).

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MongoTemplate mongoTemplate;
    private final MongoTransactionManager transactionManager;

    // Sử dụng @Transactional — cần MongoTransactionManager bean
    @Transactional
    public void createOrder(OrderDocument order, List<OrderItemDocument> items) {
        // Insert order
        mongoTemplate.insert(order, "orders");

        // Insert items
        mongoTemplate.insertAll(items);

        // Update inventory
        for (OrderItemDocument item : items) {
            Query query = new Query(Criteria.where("_id").is(item.getProductId())
                                            .and("stock").gte(item.getQuantity()));
            Update update = new Update().inc("stock", -item.getQuantity());
            UpdateResult result = mongoTemplate.updateFirst(query, update, "products");

            if (result.getMatchedCount() == 0) {
                throw new InsufficientStockException(item.getProductId().toString());
                // Exception → rollback tất cả operations trên
            }
        }
    }
}

// Config MongoTransactionManager
@Bean
public MongoTransactionManager transactionManager(MongoDatabaseFactory dbFactory) {
    return new MongoTransactionManager(dbFactory);
}
```

---

## 9. Khi Nào Dùng MongoDB vs PostgreSQL

### Dùng MongoDB Khi

```
✅ Schema thường xuyên thay đổi (startup phase, agile development)
✅ Dữ liệu hierarchical/nested sâu (product catalog với nhiều variants)
✅ Content management — articles, blog posts, comments
✅ Event logs, activity feeds — insert-heavy, ít update
✅ User-generated content với cấu trúc khác nhau
✅ Horizontal scaling quan trọng (hàng trăm triệu documents)
✅ Geospatial queries (tìm stores gần đây)
✅ Real-time analytics với aggregation pipeline
```

### Dùng PostgreSQL Khi

```
✅ Data có quan hệ phức tạp, cần JOINS nhiều
✅ Strong ACID transactions bắt buộc (financial data)
✅ Data integrity quan trọng (foreign keys, constraints)
✅ Complex reporting với nhiều điều kiện
✅ Team quen với SQL
✅ Data kích thước vừa phải (< 100 triệu rows)
```

### Dùng Cả Hai (Polyglot Persistence)

```
PostgreSQL: Users, Orders, Payments, Inventory (transactional data)
MongoDB: Product Catalog, User Profiles, Activity Logs, Content
Redis: Sessions, Cache, Rate Limiting, Leaderboards
```

---

## 10. Indexing Best Practices — Thực Tiễn Tạo Index

```java
// Index trên Document
@Document(collection = "events")
@CompoundIndex(def = "{'userId': 1, 'createdAt': -1}", name = "idx_user_created")
@CompoundIndex(def = "{'type': 1, 'status': 1, 'createdAt': -1}", name = "idx_type_status")
public class EventDocument {

    @Id private String id;

    @Indexed                           // Single index
    private String userId;

    @Indexed
    private String type;

    private String status;

    @Indexed(expireAfterSeconds = 86400)  // TTL index — xóa sau 1 ngày
    private LocalDateTime expiresAt;

    private LocalDateTime createdAt;
}

// Tạo index programmatically (khi cần logic phức tạp hơn)
@Component
public class MongoIndexInitializer implements ApplicationRunner {

    private final MongoTemplate mongoTemplate;

    @Override
    public void run(ApplicationArguments args) {
        IndexOperations indexOps = mongoTemplate.indexOps("products");

        // Compound index
        indexOps.ensureIndex(new Index()
            .on("categoryName", Sort.Direction.ASC)
            .on("price", Sort.Direction.DESC)
            .named("idx_category_price")
            .background());  // Background = không block reads khi tạo index

        // Sparse index — chỉ index documents có field tồn tại
        indexOps.ensureIndex(new Index()
            .on("promotionCode", Sort.Direction.ASC)
            .sparse()
            .unique()
            .named("idx_promo_code_sparse"));

        // Partial index — chỉ index documents thỏa điều kiện
        indexOps.ensureIndex(new Index()
            .on("sku", Sort.Direction.ASC)
            .unique()
            .named("idx_active_sku")
            .partial(new PartialIndexFilter(Criteria.where("active").is(true))));
    }
}
```

---

## ✅ Checklist MongoDB Integration

- [ ] Tạo index cho mọi field thường dùng trong query
- [ ] Dùng Projection (`fields()`) để chỉ lấy fields cần thiết
- [ ] Không lưu document lớn hơn 16MB (giới hạn MongoDB)
- [ ] Dùng Aggregation Pipeline thay vì xử lý ở application layer
- [ ] Cấu hình Write Concern (Mức Độ Đảm Bảo Ghi) và Read Concern hợp lý
- [ ] Monitor slow queries qua MongoDB Atlas hay profiler
- [ ] Cân nhắc embedding vs referencing cho nested documents

---

## 💡 Embedding vs Referencing — Khi Nào Nhúng, Khi Nào Tham Chiếu

```
Embedding (Nhúng — Lưu Trong Cùng Document):
    ✅ Dữ liệu luôn cần cùng nhau (order + order items)
    ✅ Quan hệ 1-few (1 user có 3-5 addresses)
    ✅ Không cần query độc lập child data
    ❌ Không embedding khi child data lớn/nhiều → document > 16MB

Referencing (Tham Chiếu — Lưu Riêng, Dùng ID):
    ✅ Quan hệ 1-many lớn (1 user có hàng nghìn orders)
    ✅ Child data cần query độc lập
    ✅ Nhiều documents cùng tham chiếu đến 1 entity (product được nhiều orders dùng)
    ❌ Cần $lookup (JOIN) → chậm hơn embedding
```

---

## 🔗 Liên Kết

- [1-spring-data-jpa.md](1-spring-data-jpa.md) — So sánh JPA vs MongoDB repository
- [5-spring-data-redis.md](5-spring-data-redis.md) — Polyglot persistence với Redis
- [Spring Data MongoDB Reference](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/)
- [MongoDB University](https://learn.mongodb.com/) — Khóa học miễn phí chính thức

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
