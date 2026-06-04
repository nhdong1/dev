# 2 — Migrations: Code-First, Rollback & Xung Đột Trong Team

> Migration — Di Chuyển Schema — là cơ chế để EF Core theo dõi và áp dụng thay đổi schema database theo thời gian, giống như Git quản lý code.

---

## 📌 Tổng Quan

```
Migration hoạt động như Git cho database schema:

Code thay đổi  →  Add-Migration  →  Migration files  →  Update-Database  →  DB thay đổi
(Entity class)    (tạo snapshot)    (Up/Down scripts)   (áp dụng SQL)       (schema mới)
```

EF Core lưu lịch sử migrations trong bảng `__EFMigrationsHistory` của database.

---

## 1. Workflow Cơ Bản

### Bước 1: Thay Đổi Entity Model

```csharp
// Thêm thuộc tính mới vào entity
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string? Description { get; set; }   // ← THÊM MỚI
    public bool IsActive { get; set; } = true; // ← THÊM MỚI
    public int CategoryId { get; set; }
}
```

### Bước 2: Tạo Migration

```bash
# Dùng dotnet CLI
dotnet ef migrations add AddProductDescriptionAndIsActive

# Hoặc dùng Package Manager Console trong Visual Studio
Add-Migration AddProductDescriptionAndIsActive
```

### Bước 3: Xem Migration Được Tạo Ra

```csharp
// Migrations/20260602_AddProductDescriptionAndIsActive.cs
public partial class AddProductDescriptionAndIsActive : Migration
{
    // Up() — Áp dụng thay đổi (chạy khi Update-Database)
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "Description",
            table: "Products",
            type: "nvarchar(max)",
            nullable: true);

        migrationBuilder.AddColumn<bool>(
            name: "IsActive",
            table: "Products",
            type: "bit",
            nullable: false,
            defaultValue: true);
    }

    // Down() — Hoàn tác thay đổi (chạy khi rollback)
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(name: "Description", table: "Products");
        migrationBuilder.DropColumn(name: "IsActive", table: "Products");
    }
}
```

### Bước 4: Áp Dụng Migration Vào Database

```bash
# Áp dụng tất cả migrations chưa được apply
dotnet ef database update

# Áp dụng đến một migration cụ thể
dotnet ef database update AddProductDescriptionAndIsActive
```

---

## 2. Các Lệnh Migrations Quan Trọng

```bash
# Tạo migration mới
dotnet ef migrations add <TênMigration>

# Áp dụng tất cả migrations
dotnet ef database update

# Xem danh sách migrations (đã apply và chưa apply)
dotnet ef migrations list

# Xem SQL sẽ được thực thi (không chạy thật)
dotnet ef migrations script

# Xem SQL từ migration A đến migration B
dotnet ef migrations script MigrationA MigrationB

# Xóa migration cuối cùng (chỉ khi chưa apply vào DB)
dotnet ef migrations remove

# Tạo SQL script để deploy thủ công (không cần EF trên production)
dotnet ef migrations script --output migration.sql --idempotent
```

---

## 3. Rollback — Hoàn Tác Migration

### Rollback Về Migration Trước Đó

```bash
# Rollback về một migration cụ thể (Down() sẽ được gọi)
dotnet ef database update TênMigrationMuốnQuayVề

# Rollback tất cả (về trạng thái database rỗng)
dotnet ef database update 0
```

### Ví Dụ Thực Tế

```
Lịch sử migrations:
├── 001_InitialCreate          ← Đã apply
├── 002_AddProducts            ← Đã apply
├── 003_AddCategories          ← Đã apply (hiện tại)
└── 004_AddUserRoles           ← Chưa apply (vừa tạo, có bug)

# Rollback về 003 (Down() của 004 được gọi)
dotnet ef database update 003_AddCategories
```

### Rollback Thủ Công Khi Cần

```csharp
// Trong migration Down() — luôn viết đầy đủ để rollback được
protected override void Down(MigrationBuilder migrationBuilder)
{
    // Phải viết nghịch đảo của Up() — xóa những gì Up() đã tạo
    migrationBuilder.DropTable(name: "UserRoles");
    migrationBuilder.DropColumn(name: "RoleId", table: "Users");
}
```

---

## 4. Apply Migration Trong Production

### Cách 1: Chạy Khi App Khởi Động (Auto-Migration)

```csharp
// Program.cs — Tự động apply migrations khi start
public static async Task Main(string[] args)
{
    var app = builder.Build();

    // Apply migrations tự động
    using (var scope = app.Services.CreateScope())
    {
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await context.Database.MigrateAsync();
    }

    await app.RunAsync();
}
```

**Ưu điểm:** Tiện lợi, không cần bước deploy riêng cho DB.
**Nhược điểm:** Nhiều instances khởi động cùng lúc có thể gây race condition.

### Cách 2: Generate SQL Script (Khuyến Nghị Cho Production)

```bash
# Tạo idempotent SQL script — có thể chạy nhiều lần mà không lỗi
dotnet ef migrations script --idempotent --output deploy/migration.sql
```

```sql
-- Ví dụ idempotent script được tạo ra
IF NOT EXISTS (SELECT * FROM [__EFMigrationsHistory] WHERE [MigrationId] = N'20260602_AddProducts')
BEGIN
    CREATE TABLE [Products] (
        [Id] int NOT NULL IDENTITY,
        [Name] nvarchar(200) NOT NULL,
        CONSTRAINT [PK_Products] PRIMARY KEY ([Id])
    );
    INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
    VALUES (N'20260602_AddProducts', N'8.0.0');
END;
```

**Ưu điểm:** DBA review được SQL trước khi deploy, có thể rollback thủ công.
**Nhược điểm:** Cần thêm bước trong CI/CD pipeline.

### Cách 3: EF Bundles (CLI Tool)

```bash
# Tạo executable bundle chứa tất cả migrations
dotnet ef migrations bundle --output efbundle

# Chạy bundle trên server
./efbundle --connection "Server=prod;..."
```

---

## 5. Xung Đột Migration Trong Team

### Vấn Đề

```
Developer A (nhánh feature/products):
  └── Tạo migration: 20260602_120000_AddProducts

Developer B (nhánh feature/categories):
  └── Tạo migration: 20260602_120001_AddCategories

Khi merge cả 2 nhánh vào main:
  → Conflict trong file ModelSnapshot!
  → EF Core không biết migration nào được apply trước
```

### Giải Pháp

**Bước 1:** Nhận diện xung đột

```bash
# Sau khi merge, kiểm tra migrations list
dotnet ef migrations list
# Nếu thấy 2 migrations có timestamp gần nhau — có thể bị conflict
```

**Bước 2:** Resolve ModelSnapshot conflict

File `Migrations/AppDbContextModelSnapshot.cs` thường bị conflict. Đây là cách xử lý:

```bash
# Xóa cả 2 migrations bị conflict
dotnet ef migrations remove  # Xóa migration cuối
dotnet ef migrations remove  # Xóa migration kế tiếp (nếu cần)

# Tạo lại 1 migration duy nhất chứa cả 2 thay đổi
dotnet ef migrations add AddProductsAndCategories
```

**Bước 3 (Team Process):** Quy trình để tránh conflict

```
Quy tắc cho team:
1. Pull và merge main vào nhánh của bạn TRƯỚC khi tạo migration
2. Mỗi PR chỉ chứa 1 migration
3. Merge PR liên quan đến DB sequentially (không song song)
4. Đặt tên migration có ý nghĩa, không dùng số thứ tự thủ công
```

---

## 6. Customizing Migration — Tùy Chỉnh Migration

### Thêm Seed Data Vào Migration

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.CreateTable(
        name: "Categories",
        columns: table => new
        {
            Id = table.Column<int>(nullable: false)
                      .Annotation("SqlServer:Identity", "1, 1"),
            Name = table.Column<string>(maxLength: 100, nullable: false)
        },
        constraints: table => table.PrimaryKey("PK_Categories", x => x.Id));

    // Thêm dữ liệu mặc định (seed data)
    migrationBuilder.InsertData(
        table: "Categories",
        columns: new[] { "Id", "Name" },
        values: new object[,]
        {
            { 1, "Electronics" },
            { 2, "Clothing" },
            { 3, "Books" }
        });
}
```

### Chạy SQL Thủ Công Trong Migration

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // Tạo stored procedure hoặc view
    migrationBuilder.Sql(@"
        CREATE VIEW vw_ActiveProducts AS
        SELECT Id, Name, Price
        FROM Products
        WHERE IsActive = 1
    ");
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.Sql("DROP VIEW IF EXISTS vw_ActiveProducts");
}
```

### Migration Với Dữ Liệu Có Sẵn (Data Migration)

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // Thêm cột mới (nullable trước)
    migrationBuilder.AddColumn<string>(
        name: "Slug",
        table: "Products",
        nullable: true);

    // Cập nhật dữ liệu có sẵn
    migrationBuilder.Sql(@"
        UPDATE Products
        SET Slug = LOWER(REPLACE(Name, ' ', '-'))
        WHERE Slug IS NULL
    ");

    // Sau khi có dữ liệu, đổi sang NOT NULL
    migrationBuilder.AlterColumn<string>(
        name: "Slug",
        table: "Products",
        nullable: false,
        defaultValue: "");
}
```

---

## 7. HasData — Seed Data Qua Fluent API

```csharp
// Trong IEntityTypeConfiguration<Category>
public void Configure(EntityTypeBuilder<Category> builder)
{
    builder.HasKey(c => c.Id);
    builder.Property(c => c.Name).HasMaxLength(100).IsRequired();

    // Seed data — EF Core tạo migration để insert dữ liệu này
    builder.HasData(
        new Category { Id = 1, Name = "Electronics" },
        new Category { Id = 2, Name = "Clothing" },
        new Category { Id = 3, Name = "Books" }
    );
}
```

**Lưu ý:** `HasData` yêu cầu Id cố định (không dùng identity). Tốt cho reference data (dữ liệu tham chiếu), không phù hợp cho dữ liệu thay đổi thường xuyên.

---

## 8. Multi-Database Support (Schema Theo Môi Trường)

```csharp
// Sử dụng migration khác nhau cho từng DB provider
public class AppDbContextFactory : IDesignTimeDbContextFactory<AppDbContext>
{
    public AppDbContext CreateDbContext(string[] args)
    {
        var config = new ConfigurationBuilder()
            .AddJsonFile("appsettings.Development.json")
            .Build();

        var optionsBuilder = new DbContextOptionsBuilder<AppDbContext>();
        optionsBuilder.UseSqlServer(config.GetConnectionString("Default"));

        return new AppDbContext(optionsBuilder.Options);
    }
}
```

`IDesignTimeDbContextFactory` — được dùng khi chạy `dotnet ef` lệnh từ CLI mà không có app đang chạy.

---

## 📋 Checklist Migrations

- [ ] Đặt tên migration mô tả rõ thay đổi (VD: `AddProductDescription`, không phải `Update1`)
- [ ] Luôn viết đầy đủ `Down()` để có thể rollback
- [ ] Test migration trong môi trường staging trước production
- [ ] Dùng `--idempotent` script cho production deployment
- [ ] Review SQL script được tạo ra trước khi apply
- [ ] Không xóa migration đã apply vào database shared (team/staging/prod)
- [ ] Backup database trước khi apply migrations phức tạp

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Migration conflict trong team xảy ra thế nào và xử lý ra sao?**
> Xảy ra khi 2 developer tạo migration cùng lúc trên 2 nhánh khác nhau — file `ModelSnapshot.cs` bị conflict khi merge. Giải pháp: xóa cả 2 migrations bị conflict, merge code của cả 2, tạo lại 1 migration duy nhất. Phòng tránh bằng quy trình: mỗi PR 1 migration, merge sequentially.

**Q: Làm sao apply migrations trong production một cách an toàn?**
> Dùng `dotnet ef migrations script --idempotent` để tạo SQL script, cho DBA review, rồi chạy trong maintenance window. Tránh tự động chạy `MigrateAsync()` khi production app start-up nếu có nhiều instances.

**Q: `HasData` vs InsertData trong migration khác nhau thế nào?**
> `HasData` định nghĩa seed data trong entity configuration, EF tự tạo migration khi model thay đổi, yêu cầu ID cố định. `InsertData` trong migration là SQL insert thủ công, linh hoạt hơn nhưng không được EF quản lý như model state.
