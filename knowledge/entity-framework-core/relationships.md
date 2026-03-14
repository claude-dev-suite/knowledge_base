# Entity Framework Core Relationships

> Official Documentation: https://learn.microsoft.com/ef/core/modeling/relationships/

## Table of Contents

1. [One-to-Many Relationships](#one-to-many-relationships)
2. [Many-to-Many Relationships](#many-to-many-relationships)
3. [One-to-One Relationships](#one-to-one-relationships)
4. [Navigation Properties](#navigation-properties)
5. [Foreign Key Configuration](#foreign-key-configuration)
6. [Cascade Delete Behavior](#cascade-delete-behavior)
7. [Required vs Optional Relationships](#required-vs-optional-relationships)
8. [Self-Referencing Relationships](#self-referencing-relationships)
9. [Fluent API Configuration](#fluent-api-configuration)
10. [Shadow Foreign Keys](#shadow-foreign-keys)
11. [Alternate Keys](#alternate-keys)
12. [Owned Entities](#owned-entities)
13. [Complex Types](#complex-types)
14. [Table Splitting](#table-splitting)
15. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## One-to-Many Relationships

The most common relationship type. One parent entity has many child entities.

### Convention-Based

```csharp
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Navigation property (collection)
    public ICollection<Product> Products { get; set; } = new List<Product>();
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    // Foreign key (convention: {NavigationProperty}Id)
    public int CategoryId { get; set; }

    // Navigation property (reference)
    public Category Category { get; set; } = null!;
}
```

### Fluent API Configuration

```csharp
public class CategoryConfiguration : IEntityTypeConfiguration<Category>
{
    public void Configure(EntityTypeBuilder<Category> builder)
    {
        builder.HasMany(c => c.Products)
            .WithOne(p => p.Category)
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

### Usage

```csharp
// Create with relationship
var category = new Category { Name = "Electronics" };
category.Products.Add(new Product { Name = "Laptop", Price = 999.99m });
category.Products.Add(new Product { Name = "Phone", Price = 699.99m });
context.Categories.Add(category);
await context.SaveChangesAsync();

// Query with related data
var categoryWithProducts = await context.Categories
    .Include(c => c.Products)
    .FirstAsync(c => c.Id == 1);
```

---

## Many-to-Many Relationships

### Skip Navigation (No Join Entity - EF Core 5+)

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public ICollection<Course> Courses { get; set; } = new List<Course>();
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;

    public ICollection<Student> Students { get; set; } = new List<Student>();
}

// EF Core auto-creates the join table "CourseStudent"
```

### Fluent API for Skip Navigation

```csharp
public class StudentConfiguration : IEntityTypeConfiguration<Student>
{
    public void Configure(EntityTypeBuilder<Student> builder)
    {
        builder.HasMany(s => s.Courses)
            .WithMany(c => c.Students)
            .UsingEntity(j => j.ToTable("Enrollments"));
    }
}
```

### With Explicit Join Entity (Payload Properties)

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public ICollection<Enrollment> Enrollments { get; set; } = new List<Enrollment>();
    public ICollection<Course> Courses { get; set; } = new List<Course>();
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;

    public ICollection<Enrollment> Enrollments { get; set; } = new List<Enrollment>();
    public ICollection<Student> Students { get; set; } = new List<Student>();
}

public class Enrollment
{
    public int StudentId { get; set; }
    public Student Student { get; set; } = null!;

    public int CourseId { get; set; }
    public Course Course { get; set; } = null!;

    // Payload properties
    public DateTime EnrolledAt { get; set; }
    public Grade? Grade { get; set; }
}

public enum Grade { A, B, C, D, F }
```

### Configuration with Join Entity

```csharp
public class EnrollmentConfiguration : IEntityTypeConfiguration<Enrollment>
{
    public void Configure(EntityTypeBuilder<Enrollment> builder)
    {
        builder.HasKey(e => new { e.StudentId, e.CourseId });

        builder.HasOne(e => e.Student)
            .WithMany(s => s.Enrollments)
            .HasForeignKey(e => e.StudentId);

        builder.HasOne(e => e.Course)
            .WithMany(c => c.Enrollments)
            .HasForeignKey(e => e.CourseId);

        builder.Property(e => e.EnrolledAt)
            .HasDefaultValueSql("GETUTCDATE()");
    }
}

// Configure skip navigations
modelBuilder.Entity<Student>()
    .HasMany(s => s.Courses)
    .WithMany(c => c.Students)
    .UsingEntity<Enrollment>();
```

### Usage

```csharp
// Add enrollment with payload
var enrollment = new Enrollment
{
    StudentId = 1,
    CourseId = 1,
    EnrolledAt = DateTime.UtcNow,
    Grade = Grade.A
};
context.Set<Enrollment>().Add(enrollment);
await context.SaveChangesAsync();

// Query through skip navigation
var studentWithCourses = await context.Students
    .Include(s => s.Courses)
    .FirstAsync(s => s.Id == 1);

// Query through join entity (access payload)
var enrollments = await context.Set<Enrollment>()
    .Include(e => e.Student)
    .Include(e => e.Course)
    .Where(e => e.Grade == Grade.A)
    .ToListAsync();
```

---

## One-to-One Relationships

```csharp
public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;

    public UserProfile? Profile { get; set; }
}

public class UserProfile
{
    public int Id { get; set; }
    public string Bio { get; set; } = string.Empty;
    public string? AvatarUrl { get; set; }
    public DateTime DateOfBirth { get; set; }

    // Foreign key to User (dependent side)
    public int UserId { get; set; }
    public User User { get; set; } = null!;
}
```

### Configuration

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasOne(u => u.Profile)
            .WithOne(p => p.User)
            .HasForeignKey<UserProfile>(p => p.UserId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### Usage

```csharp
// Create user with profile
var user = new User
{
    Email = "john@example.com",
    Profile = new UserProfile
    {
        Bio = "Software developer",
        DateOfBirth = new DateOnly(1990, 5, 15).ToDateTime(TimeOnly.MinValue)
    }
};
context.Users.Add(user);
await context.SaveChangesAsync();
```

---

## Navigation Properties

Navigation properties define the relationship between entities in the object model.

| Type | Definition | Example |
|------|-----------|---------|
| Reference navigation | Single related entity | `public Category Category { get; set; }` |
| Collection navigation | Multiple related entities | `public ICollection<Product> Products { get; set; }` |
| Skip navigation | Many-to-many without join entity | `public ICollection<Tag> Tags { get; set; }` |

### Collection Types

```csharp
public class Category
{
    // List<T> - ordered, allows duplicates
    public List<Product> Products { get; set; } = new();

    // ICollection<T> - preferred, minimal interface
    public ICollection<Product> Products { get; set; } = new List<Product>();

    // HashSet<T> - optimized for lookups, no duplicates
    public ICollection<Product> Products { get; set; } = new HashSet<Product>();
}
```

### Required vs Optional References

```csharp
public class Product
{
    // Required reference (non-nullable FK)
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;

    // Optional reference (nullable FK)
    public int? DiscountId { get; set; }
    public Discount? Discount { get; set; }
}
```

---

## Foreign Key Configuration

### Convention-Based Names

EF Core discovers foreign keys by convention:

| Convention | Example Property |
|-----------|-----------------|
| `{NavigationProperty}Id` | `CategoryId` for `Category` navigation |
| `{PrincipalType}Id` | `CategoryId` when `Category` type exists |
| `{NavigationProperty}{PrimaryKey}` | `CategoryCategoryId` (less common) |

### Explicit Foreign Key

```csharp
// Data annotation
public class Product
{
    public int Id { get; set; }

    [ForeignKey(nameof(Category))]
    public int CatId { get; set; }
    public Category Category { get; set; } = null!;
}

// Fluent API
builder.HasOne(p => p.Category)
    .WithMany(c => c.Products)
    .HasForeignKey(p => p.CatId);
```

### Composite Foreign Keys

```csharp
public class OrderItem
{
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }

    public Order Order { get; set; } = null!;
    public Product Product { get; set; } = null!;
}

// Configuration
builder.HasKey(oi => new { oi.OrderId, oi.ProductId });

builder.HasOne(oi => oi.Order)
    .WithMany(o => o.Items)
    .HasForeignKey(oi => oi.OrderId);

builder.HasOne(oi => oi.Product)
    .WithMany()
    .HasForeignKey(oi => oi.ProductId);
```

---

## Cascade Delete Behavior

| Behavior | On Delete Principal | Dependent FK Nullable | Result |
|----------|-------------------|-----------------------|--------|
| `Cascade` | Delete | N/A | Dependents deleted |
| `Restrict` | Throws exception | N/A | Prevents deletion |
| `SetNull` | Set FK to null | Yes (required) | FK set to null |
| `NoAction` | No database action | N/A | DB may throw constraint error |
| `ClientSetNull` | Default for optional | Yes | FK set to null (client-side) |

```csharp
// Cascade: deleting Category deletes all Products
builder.HasMany(c => c.Products)
    .WithOne(p => p.Category)
    .HasForeignKey(p => p.CategoryId)
    .OnDelete(DeleteBehavior.Cascade);

// Restrict: cannot delete Category if Products exist
builder.HasMany(c => c.Products)
    .WithOne(p => p.Category)
    .HasForeignKey(p => p.CategoryId)
    .OnDelete(DeleteBehavior.Restrict);

// SetNull: deleting Category sets Products.CategoryId to null
builder.HasMany(c => c.Products)
    .WithOne(p => p.Category)
    .HasForeignKey(p => p.CategoryId)  // Must be nullable int?
    .OnDelete(DeleteBehavior.SetNull);
```

---

## Required vs Optional Relationships

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Required relationship (non-nullable FK)
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;

    // Optional relationship (nullable FK)
    public int? BrandId { get; set; }
    public Brand? Brand { get; set; }
}

// Fluent API - required
builder.HasOne(p => p.Category)
    .WithMany(c => c.Products)
    .HasForeignKey(p => p.CategoryId)
    .IsRequired();

// Fluent API - optional
builder.HasOne(p => p.Brand)
    .WithMany(b => b.Products)
    .HasForeignKey(p => p.BrandId)
    .IsRequired(false);
```

---

## Self-Referencing Relationships

### Hierarchical (Parent-Child)

```csharp
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Self-referencing FK
    public int? ParentCategoryId { get; set; }
    public Category? ParentCategory { get; set; }

    public ICollection<Category> SubCategories { get; set; } = new List<Category>();
}

// Configuration
builder.HasOne(c => c.ParentCategory)
    .WithMany(c => c.SubCategories)
    .HasForeignKey(c => c.ParentCategoryId)
    .OnDelete(DeleteBehavior.Restrict);
```

### Employee-Manager

```csharp
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public int? ManagerId { get; set; }
    public Employee? Manager { get; set; }

    public ICollection<Employee> DirectReports { get; set; } = new List<Employee>();
}

// Query hierarchy
var manager = await context.Employees
    .Include(e => e.DirectReports)
        .ThenInclude(e => e.DirectReports) // Two levels deep
    .FirstAsync(e => e.Id == managerId);
```

---

## Fluent API Configuration

### Complete Fluent API Reference

```csharp
// HasOne / HasMany / WithOne / WithMany
builder.HasOne(p => p.Category)        // This entity has ONE Category
    .WithMany(c => c.Products)         // Category has MANY Products
    .HasForeignKey(p => p.CategoryId)  // FK on Product
    .HasPrincipalKey(c => c.Id)        // References Category.Id
    .OnDelete(DeleteBehavior.Restrict)
    .IsRequired();

// Inverse navigation
builder.HasMany(c => c.Products)       // This entity has MANY Products
    .WithOne(p => p.Category)          // Product has ONE Category
    .HasForeignKey(p => p.CategoryId);

// One-to-one
builder.HasOne(u => u.Profile)
    .WithOne(p => p.User)
    .HasForeignKey<UserProfile>(p => p.UserId);

// Many-to-many
builder.HasMany(s => s.Courses)
    .WithMany(c => c.Students)
    .UsingEntity<Enrollment>(
        l => l.HasOne(e => e.Course).WithMany(c => c.Enrollments).HasForeignKey(e => e.CourseId),
        r => r.HasOne(e => e.Student).WithMany(s => s.Enrollments).HasForeignKey(e => e.StudentId),
        j =>
        {
            j.HasKey(e => new { e.StudentId, e.CourseId });
            j.ToTable("Enrollments");
        });
```

---

## Shadow Foreign Keys

Shadow properties exist in the EF model but not in the .NET entity class.

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    // No CategoryId property
    public Category Category { get; set; } = null!;
}

// EF Core creates a shadow FK property "CategoryId"
builder.HasOne(p => p.Category)
    .WithMany(c => c.Products)
    .HasForeignKey("CategoryId"); // Shadow property name as string

// Access shadow properties
var categoryId = context.Entry(product).Property("CategoryId").CurrentValue;

// Query using shadow property
var products = await context.Products
    .Where(p => EF.Property<int>(p, "CategoryId") == 5)
    .ToListAsync();
```

---

## Alternate Keys

Alternate keys define unique constraints that can be used as FK targets instead of the primary key.

```csharp
public class Country
{
    public int Id { get; set; }
    public string Code { get; set; } = string.Empty;  // e.g., "US", "GB"
    public string Name { get; set; } = string.Empty;
}

public class Address
{
    public int Id { get; set; }
    public string Street { get; set; } = string.Empty;
    public string CountryCode { get; set; } = string.Empty;
    public Country Country { get; set; } = null!;
}

// Configuration
modelBuilder.Entity<Country>()
    .HasAlternateKey(c => c.Code);

modelBuilder.Entity<Address>()
    .HasOne(a => a.Country)
    .WithMany()
    .HasForeignKey(a => a.CountryCode)
    .HasPrincipalKey(c => c.Code);
```

---

## Owned Entities

Owned entities are types that don't have their own identity and are always navigated to from an owner.

### OwnsOne

```csharp
public class Order
{
    public int Id { get; set; }
    public decimal TotalAmount { get; set; }

    public Address ShippingAddress { get; set; } = null!;
    public Address BillingAddress { get; set; } = null!;
}

public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string State { get; set; } = string.Empty;
    public string ZipCode { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
}

// Configuration - stored in the same table as Order
builder.OwnsOne(o => o.ShippingAddress, sa =>
{
    sa.Property(a => a.Street).HasColumnName("ShippingStreet").HasMaxLength(200);
    sa.Property(a => a.City).HasColumnName("ShippingCity").HasMaxLength(100);
    sa.Property(a => a.State).HasColumnName("ShippingState").HasMaxLength(50);
    sa.Property(a => a.ZipCode).HasColumnName("ShippingZipCode").HasMaxLength(20);
    sa.Property(a => a.Country).HasColumnName("ShippingCountry").HasMaxLength(100);
});

builder.OwnsOne(o => o.BillingAddress, ba =>
{
    ba.Property(a => a.Street).HasColumnName("BillingStreet").HasMaxLength(200);
    ba.Property(a => a.City).HasColumnName("BillingCity").HasMaxLength(100);
    ba.Property(a => a.State).HasColumnName("BillingState").HasMaxLength(50);
    ba.Property(a => a.ZipCode).HasColumnName("BillingZipCode").HasMaxLength(20);
    ba.Property(a => a.Country).HasColumnName("BillingCountry").HasMaxLength(100);
});
```

### OwnsMany (Stored in Separate Table)

```csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public ICollection<PhoneNumber> PhoneNumbers { get; set; } = new List<PhoneNumber>();
}

public class PhoneNumber
{
    public string Number { get; set; } = string.Empty;
    public string Type { get; set; } = string.Empty; // "Mobile", "Home", "Work"
}

// Configuration
builder.OwnsMany(c => c.PhoneNumbers, pn =>
{
    pn.ToTable("CustomerPhoneNumbers");
    pn.WithOwner().HasForeignKey("CustomerId");
    pn.Property(p => p.Number).HasMaxLength(20);
    pn.Property(p => p.Type).HasMaxLength(20);
});
```

### OwnsOne with JSON Column (EF Core 7+)

```csharp
builder.OwnsOne(o => o.ShippingAddress, sa =>
{
    sa.ToJson();
});

// The Address is stored as a JSON column in the Orders table
// { "Street": "123 Main St", "City": "Seattle", ... }
```

---

## Complex Types

Complex types (.NET 8+) are value objects without identity, similar to owned types but simpler.

```csharp
[ComplexType]
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; } = "USD";
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public Money Price { get; set; } = new();
    public Money? CompareAtPrice { get; set; }
}

// Configuration
modelBuilder.Entity<Product>()
    .ComplexProperty(p => p.Price, cp =>
    {
        cp.Property(m => m.Amount).HasPrecision(18, 2);
        cp.Property(m => m.Currency).HasMaxLength(3);
    });
```

| Feature | Owned Entity | Complex Type |
|---------|-------------|--------------|
| Identity (key) | No | No |
| Can be null | Yes | No (EF Core 8) |
| Can be in collection | Yes (OwnsMany) | No |
| Table mapping | Same or separate | Same table only |
| JSON column | Yes | No |
| Simpler configuration | No | Yes |

---

## Table Splitting

Multiple entity types mapped to the same table.

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    public ProductDetails Details { get; set; } = null!;
}

public class ProductDetails
{
    public int Id { get; set; }
    public string FullDescription { get; set; } = string.Empty;
    public string Specifications { get; set; } = string.Empty;
    public byte[]? Image { get; set; }

    public Product Product { get; set; } = null!;
}

// Both map to the same "Products" table
modelBuilder.Entity<Product>()
    .HasOne(p => p.Details)
    .WithOne(d => d.Product)
    .HasForeignKey<ProductDetails>(d => d.Id);

modelBuilder.Entity<Product>().ToTable("Products");
modelBuilder.Entity<ProductDetails>().ToTable("Products");
```

### Usage - Load Light Entity First

```csharp
// Load product without heavy details
var product = await context.Products.FirstAsync(p => p.Id == 1);

// Load details when needed
await context.Entry(product).Reference(p => p.Details).LoadAsync();
```

---

## Best Practices and Common Pitfalls

### Best Practices

| Practice | Description |
|----------|-------------|
| Use `ICollection<T>` for navigation collections | Minimal interface, works with all EF features |
| Configure delete behavior explicitly | Don't rely on defaults for important relationships |
| Use owned entities for value objects | Address, Money, DateRange |
| Prefer explicit FK properties | Clearer code, easier to assign relationships |
| Use `IEntityTypeConfiguration<T>` | Separate and testable configuration per entity |
| Initialize collections in constructors | Avoid null reference exceptions |
| Use composite keys for join entities | Natural key for many-to-many with payload |
| Avoid circular eager loading | Can cause infinite loops and stack overflow |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Missing `HasForeignKey` on one-to-one | EF picks wrong side as dependent | Always specify FK side explicitly |
| Cascade delete cycles | SQL Server rejects multiple cascade paths | Use `Restrict` on one path |
| Forgetting inverse navigation | Relationship not fully configured | Always set both sides of the relationship |
| Lazy loading without proxies | Null navigation properties | Enable proxies or use explicit/eager loading |
| N+1 with lazy loading | One query per related entity | Use `Include` for known access patterns |
| Detached entity graph issues | Related entities not tracked | Use `Attach` or explicit state setting |
| Owned entity null checks | Owned entities are never null by default | Model optional ownership carefully |
| Exposing `List<T>` instead of `ICollection<T>` | Tight coupling to implementation | Use `ICollection<T>` in entity definitions |
