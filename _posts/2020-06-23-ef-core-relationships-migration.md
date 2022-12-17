---
layout: post
title: "Entity Framework Core 關聯配置與 Migration"
date: 2020-06-23 14:30:00 +0800
categories: [框架, .NET]
tags: [EF Core, Migration, Relationships]
---

上週學習了 LINQ 查詢，本週深入研究 EF Core 的關聯配置和資料庫遷移機制。

> 本文環境：**EF Core 3.1** + **.NET Core 3.1 LTS**

## JPA 關聯配置回顧

在 JPA 中，我們使用註解定義關聯：

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;
    
    @OneToMany(mappedBy = "product")
    private List<OrderItem> orderItems;
}
```

## EF Core 關聯類型

EF Core 支援三種關聯：
1. **一對多** (One-to-Many)
2. **一對一** (One-to-One)
3. **多對多** (Many-to-Many)

## 一對多關聯

### 使用慣例配置

```csharp
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    // 導覽屬性
    public ICollection<Product> Products { get; set; }
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    // 外鍵
    public int CategoryId { get; set; }
    
    // 導覽屬性
    public Category Category { get; set; }
}
```

EF Core 會自動識別：
- `CategoryId` 是外鍵
- `Category` 是父實體的導覽屬性
- `Products` 是子實體的集合導覽屬性

### 使用 Fluent API 配置

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Product>()
        .HasOne(p => p.Category)
        .WithMany(c => c.Products)
        .HasForeignKey(p => p.CategoryId)
        .OnDelete(DeleteBehavior.Cascade);
}
```

JPA 對應：
```java
@ManyToOne
@JoinColumn(name = "category_id", nullable = false)
private Category category;
```

### 必填與選填關聯

```csharp
// 必填關聯（不可為 null）
public class Product
{
    public int CategoryId { get; set; }
    public Category Category { get; set; }
}

// 選填關聯（可為 null）
public class Product
{
    public int? CategoryId { get; set; }
    public Category Category { get; set; }
}

// Fluent API 設定
modelBuilder.Entity<Product>()
    .HasOne(p => p.Category)
    .WithMany(c => c.Products)
    .HasForeignKey(p => p.CategoryId)
    .IsRequired(false); // 選填
```

## 一對一關聯

### 使用者與個人資料

```csharp
public class User
{
    public int Id { get; set; }
    public string Email { get; set; }
    
    public UserProfile Profile { get; set; }
}

public class UserProfile
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime BirthDate { get; set; }
    
    public int UserId { get; set; }
    public User User { get; set; }
}
```

### Fluent API 配置

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>()
        .HasOne(u => u.Profile)
        .WithOne(p => p.User)
        .HasForeignKey<UserProfile>(p => p.UserId);
}
```

JPA 對應：
```java
@Entity
public class User {
    @OneToOne(mappedBy = "user")
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @OneToOne
    @JoinColumn(name = "user_id")
    private User user;
}
```

## 多對多關聯

### EF Core 5.0+ 自動配置

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    public ICollection<Course> Courses { get; set; }
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; }
    
    public ICollection<Student> Students { get; set; }
}
```

EF Core 5.0+ 會自動建立中介資料表 `CourseStudent`。

### EF Core 3.1 需要明確定義中介實體

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    public ICollection<StudentCourse> StudentCourses { get; set; }
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; }
    
    public ICollection<StudentCourse> StudentCourses { get; set; }
}

public class StudentCourse
{
    public int StudentId { get; set; }
    public Student Student { get; set; }
    
    public int CourseId { get; set; }
    public Course Course { get; set; }
    
    // 額外欄位
    public DateTime EnrolledDate { get; set; }
    public int? Grade { get; set; }
}
```

### Fluent API 配置

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<StudentCourse>()
        .HasKey(sc => new { sc.StudentId, sc.CourseId });

    modelBuilder.Entity<StudentCourse>()
        .HasOne(sc => sc.Student)
        .WithMany(s => s.StudentCourses)
        .HasForeignKey(sc => sc.StudentId);

    modelBuilder.Entity<StudentCourse>()
        .HasOne(sc => sc.Course)
        .WithMany(c => c.StudentCourses)
        .HasForeignKey(sc => sc.CourseId);
}
```

JPA 對應：
```java
@Entity
public class Student {
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private List<Course> courses;
}
```

## 資料表與欄位配置

### 資料表名稱

```csharp
[Table("tbl_products")]
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
}

// 或使用 Fluent API
modelBuilder.Entity<Product>()
    .ToTable("tbl_products");
```

### 欄位配置

```csharp
public class Product
{
    public int Id { get; set; }
    
    [Column("product_name", TypeName = "nvarchar(200)")]
    [Required]
    [MaxLength(200)]
    public string Name { get; set; }
    
    [Column(TypeName = "decimal(18,2)")]
    public decimal Price { get; set; }
    
    [Column("created_at")]
    public DateTime CreatedAt { get; set; }
}

// 或使用 Fluent API
modelBuilder.Entity<Product>(entity =>
{
    entity.Property(p => p.Name)
        .HasColumnName("product_name")
        .HasColumnType("nvarchar(200)")
        .IsRequired()
        .HasMaxLength(200);
    
    entity.Property(p => p.Price)
        .HasColumnType("decimal(18,2)");
    
    entity.Property(p => p.CreatedAt)
        .HasColumnName("created_at")
        .HasDefaultValueSql("GETDATE()");
});
```

### 主鍵配置

```csharp
// 複合主鍵
modelBuilder.Entity<OrderItem>()
    .HasKey(oi => new { oi.OrderId, oi.ProductId });

// GUID 主鍵
public class Product
{
    [Key]
    public Guid Id { get; set; } = Guid.NewGuid();
}

// 非自動遞增主鍵
modelBuilder.Entity<Product>()
    .Property(p => p.Id)
    .ValueGeneratedNever();
```

### 索引配置

```csharp
// 單一欄位索引
[Index(nameof(Email), IsUnique = true)]
public class User
{
    public int Id { get; set; }
    public string Email { get; set; }
}

// Fluent API
modelBuilder.Entity<User>()
    .HasIndex(u => u.Email)
    .IsUnique()
    .HasDatabaseName("IX_User_Email");

// 複合索引
modelBuilder.Entity<Product>()
    .HasIndex(p => new { p.CategoryId, p.Price });
```

## 資料庫遷移 (Migration)

### 建立遷移

```bash
# 建立第一個遷移
dotnet ef migrations add InitialCreate

# 建立後續遷移
dotnet ef migrations add AddProductDescription
```

這會產生三個檔案：
- `yyyyMMddHHmmss_InitialCreate.cs` - 遷移邏輯
- `yyyyMMddHHmmss_InitialCreate.Designer.cs` - 模型快照
- `ApplicationDbContextModelSnapshot.cs` - 目前模型

### 遷移檔案內容

```csharp
public partial class InitialCreate : Migration
{
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
            constraints: table =>
            {
                table.PrimaryKey("PK_Categories", x => x.Id);
            });

        migrationBuilder.CreateTable(
            name: "Products",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(maxLength: 200, nullable: false),
                Price = table.Column<decimal>(type: "decimal(18,2)", nullable: false),
                CategoryId = table.Column<int>(nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Products", x => x.Id);
                table.ForeignKey(
                    name: "FK_Products_Categories_CategoryId",
                    column: x => x.CategoryId,
                    principalTable: "Categories",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Cascade);
            });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Products");
        migrationBuilder.DropTable(name: "Categories");
    }
}
```

### 套用遷移

```bash
# 套用所有未執行的遷移
dotnet ef database update

# 套用到特定遷移
dotnet ef database update AddProductDescription

# 回滾到初始狀態
dotnet ef database update 0

# 回滾到特定遷移
dotnet ef database update InitialCreate
```

### 程式碼中套用遷移

```csharp
// Program.cs 或 Startup.cs
public static void Main(string[] args)
{
    var host = CreateHostBuilder(args).Build();
    
    using (var scope = host.Services.CreateScope())
    {
        var context = scope.ServiceProvider
            .GetRequiredService<ApplicationDbContext>();
        
        // 套用遷移
        context.Database.Migrate();
    }
    
    host.Run();
}
```

### 查看 SQL 腳本

```bash
# 產生 SQL 腳本
dotnet ef migrations script

# 產生特定範圍的 SQL
dotnet ef migrations script InitialCreate AddProductDescription

# 產生所有遷移的 SQL
dotnet ef migrations script -o migration.sql
```

### 移除遷移

```bash
# 移除最後一個遷移（尚未套用到資料庫）
dotnet ef migrations remove

# 強制移除（即使已套用）
dotnet ef migrations remove --force
```

## 資料初始化 (Seeding)

### 使用 HasData

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Category>().HasData(
        new Category { Id = 1, Name = "電子產品" },
        new Category { Id = 2, Name = "服飾" },
        new Category { Id = 3, Name = "食品" }
    );

    modelBuilder.Entity<Product>().HasData(
        new Product { Id = 1, Name = "iPhone 13", Price = 25900, CategoryId = 1 },
        new Product { Id = 2, Name = "MacBook Pro", Price = 59900, CategoryId = 1 },
        new Product { Id = 3, Name = "T-Shirt", Price = 590, CategoryId = 2 }
    );
}
```

建立遷移後會自動產生 INSERT 語句。

### 使用自訂初始化

```csharp
public static class DbInitializer
{
    public static void Initialize(ApplicationDbContext context)
    {
        context.Database.EnsureCreated();

        if (context.Products.Any())
        {
            return; // 已有資料
        }

        var categories = new[]
        {
            new Category { Name = "電子產品" },
            new Category { Name = "服飾" },
            new Category { Name = "食品" }
        };
        context.Categories.AddRange(categories);
        context.SaveChanges();

        var products = new[]
        {
            new Product { Name = "iPhone 13", Price = 25900, CategoryId = 1 },
            new Product { Name = "MacBook Pro", Price = 59900, CategoryId = 1 },
            new Product { Name = "T-Shirt", Price = 590, CategoryId = 2 }
        };
        context.Products.AddRange(products);
        context.SaveChanges();
    }
}

// Program.cs
using (var scope = host.Services.CreateScope())
{
    var context = scope.ServiceProvider
        .GetRequiredService<ApplicationDbContext>();
    DbInitializer.Initialize(context);
}
```

## 遷移最佳實踐

### 1. 命名規範

```bash
# 好的命名
dotnet ef migrations add AddProductDescription
dotnet ef migrations add CreateOrderTables
dotnet ef migrations add UpdateProductIndexes

# 不好的命名
dotnet ef migrations add Update1
dotnet ef migrations add Fix
```

### 2. 小步遷移

每個遷移只做一件事，避免大型遷移：

```bash
# 分成多個小遷移
dotnet ef migrations add AddProductDescription
dotnet ef migrations add AddProductImage
dotnet ef migrations add AddProductTags

# 而不是
dotnet ef migrations add UpdateProductSchema
```

### 3. 測試遷移

```bash
# 測試向上遷移
dotnet ef database update

# 測試向下遷移
dotnet ef database update PreviousMigration

# 測試重新套用
dotnet ef database update CurrentMigration
```

### 4. 避免資料遺失

```csharp
// 重新命名欄位時，使用 RenameColumn
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.RenameColumn(
        name: "Description",
        table: "Products",
        newName: "ProductDescription");
}

// 而不是 DropColumn + AddColumn
```

### 5. 使用交易

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.Sql("BEGIN TRANSACTION");
    
    // 遷移操作...
    
    migrationBuilder.Sql("COMMIT TRANSACTION");
}
```

## 與 Flyway 的對比

| 功能 | EF Core Migration | Flyway |
|------|-------------------|--------|
| 產生方式 | 自動產生 | 手寫 SQL |
| 版本控制 | 時間戳 + 名稱 | V1__description.sql |
| 回滾 | Down() 方法 | 需要另寫 Undo SQL |
| 整合 | .NET 專案 | 獨立工具 |
| 學習曲線 | 中等 | 簡單 |

## 小結

EF Core 的關聯配置與遷移：
- 慣例優於配置，大部分情況不需要額外設定
- Fluent API 提供完整控制
- Migration 自動追蹤模型變更
- 支援正向與反向遷移
- 與 Code First 開發模式完美整合

相較於 JPA + Flyway：
- EF Core 更加自動化
- 不需要手寫 SQL（但也可以）
- 模型變更自動追蹤
- 開發體驗更流暢

下週將探討 ASP.NET Core 的設定管理與環境配置。
