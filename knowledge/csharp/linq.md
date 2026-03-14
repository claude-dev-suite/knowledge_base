# C# LINQ (Language Integrated Query)

> Official Documentation: https://learn.microsoft.com/dotnet/csharp/linq/

## Table of Contents

1. [Method Syntax vs Query Syntax](#method-syntax-vs-query-syntax)
2. [Filtering](#filtering)
3. [Projection](#projection)
4. [Ordering](#ordering)
5. [Grouping](#grouping)
6. [Aggregation](#aggregation)
7. [Element Operators](#element-operators)
8. [Set Operations](#set-operations)
9. [Modern Set Operations (.NET 6+)](#modern-set-operations)
10. [Chunk (.NET 6+)](#chunk)
11. [Quantifiers](#quantifiers)
12. [Join Operations](#join-operations)
13. [Deferred vs Immediate Execution](#deferred-vs-immediate-execution)
14. [Zip, Range, Repeat](#zip-range-repeat)
15. [Custom Extension Methods](#custom-extension-methods)
16. [LINQ to Objects vs LINQ to Entities](#linq-to-objects-vs-linq-to-entities)
17. [Best Practices and Common Pitfalls](#best-practices-and-common-pitfalls)

---

## Method Syntax vs Query Syntax

LINQ provides two interchangeable syntax styles.

```csharp
var products = new List<Product>
{
    new("Laptop", 999.99m, "Electronics"),
    new("Phone", 699.99m, "Electronics"),
    new("Shirt", 29.99m, "Clothing"),
    new("Book", 14.99m, "Books"),
    new("Tablet", 449.99m, "Electronics"),
};

// Method syntax (lambda) - more common, more operators
var expensive = products
    .Where(p => p.Price > 100)
    .OrderByDescending(p => p.Price)
    .Select(p => new { p.Name, p.Price });

// Query syntax - familiar to SQL users
var expensive2 =
    from p in products
    where p.Price > 100
    orderby p.Price descending
    select new { p.Name, p.Price };

// Both produce identical results
```

### When to Use Each

| Syntax | Best For |
|--------|----------|
| Method syntax | Most operations, chaining, single-expression |
| Query syntax | Joins, multiple `from` clauses, `let` keyword |

### The `let` Keyword (Query Syntax Only)

```csharp
var result =
    from p in products
    let discountedPrice = p.Price * 0.9m
    where discountedPrice > 100
    select new { p.Name, OriginalPrice = p.Price, DiscountedPrice = discountedPrice };
```

---

## Filtering

### Where

```csharp
// Single condition
var electronics = products.Where(p => p.Category == "Electronics");

// Multiple conditions
var expensiveElectronics = products
    .Where(p => p.Category == "Electronics" && p.Price > 500);

// With index
var evenIndexed = products.Where((p, index) => index % 2 == 0);

// Type filtering
object[] mixed = [1, "hello", 2.0, "world", 3, true];
var strings = mixed.OfType<string>(); // ["hello", "world"]
var ints = mixed.OfType<int>();       // [1, 3]
```

---

## Projection

### Select

```csharp
// Transform elements
var names = products.Select(p => p.Name);

// Anonymous type projection
var summary = products.Select(p => new
{
    p.Name,
    FormattedPrice = $"${p.Price:F2}",
    IsExpensive = p.Price > 100
});

// Projection with index
var indexed = products.Select((p, i) => new { Index = i, p.Name, p.Price });

// Tuple projection
var tuples = products.Select(p => (p.Name, p.Price));
foreach (var (name, price) in tuples)
{
    Console.WriteLine($"{name}: {price:C}");
}
```

### SelectMany (Flatten Nested Collections)

```csharp
var customers = new List<Customer>
{
    new("Alice", [new Order(1, 100m), new Order(2, 200m)]),
    new("Bob", [new Order(3, 150m)]),
    new("Carol", [new Order(4, 300m), new Order(5, 50m), new Order(6, 75m)]),
};

record Customer(string Name, List<Order> Orders);
record Order(int Id, decimal Amount);

// Flatten all orders from all customers
var allOrders = customers.SelectMany(c => c.Orders);
// [Order(1, 100), Order(2, 200), Order(3, 150), Order(4, 300), Order(5, 50), Order(6, 75)]

// Include parent info
var orderDetails = customers.SelectMany(
    c => c.Orders,
    (customer, order) => new { customer.Name, order.Id, order.Amount });
// [{ Alice, 1, 100 }, { Alice, 2, 200 }, { Bob, 3, 150 }, ...]

// Flatten nested arrays
string[][] nested = [["a", "b"], ["c", "d"], ["e"]];
var flat = nested.SelectMany(x => x); // ["a", "b", "c", "d", "e"]

// Split and flatten strings
var words = new[] { "hello world", "foo bar baz" }
    .SelectMany(s => s.Split(' ')); // ["hello", "world", "foo", "bar", "baz"]
```

---

## Ordering

```csharp
// Ascending
var byName = products.OrderBy(p => p.Name);

// Descending
var byPriceDesc = products.OrderByDescending(p => p.Price);

// Multiple sort criteria
var sorted = products
    .OrderBy(p => p.Category)
    .ThenByDescending(p => p.Price)
    .ThenBy(p => p.Name);

// Custom comparer
var caseInsensitive = products
    .OrderBy(p => p.Name, StringComparer.OrdinalIgnoreCase);

// Reverse
var reversed = products.OrderBy(p => p.Name).Reverse();

// Order by complex key
var byNameLength = products.OrderBy(p => p.Name.Length);
```

---

## Grouping

### GroupBy

```csharp
// Basic grouping
var byCategory = products.GroupBy(p => p.Category);

foreach (var group in byCategory)
{
    Console.WriteLine($"Category: {group.Key} ({group.Count()} items)");
    foreach (var product in group)
    {
        Console.WriteLine($"  - {product.Name}: {product.Price:C}");
    }
}

// Group with element selector
var namesByCategory = products.GroupBy(
    p => p.Category,
    p => p.Name);

// Group with result selector
var categoryStats = products.GroupBy(
    p => p.Category,
    (category, items) => new
    {
        Category = category,
        Count = items.Count(),
        Total = items.Sum(p => p.Price),
        Average = items.Average(p => p.Price),
        Cheapest = items.Min(p => p.Price),
        MostExpensive = items.Max(p => p.Price)
    });

// Group by multiple keys
var byGroupedKeys = products.GroupBy(p => new { p.Category, IsExpensive = p.Price > 100 });

// Group and flatten (GroupBy + SelectMany)
var topPerCategory = products
    .GroupBy(p => p.Category)
    .SelectMany(g => g.OrderByDescending(p => p.Price).Take(2));
```

### ToLookup (Immediate Execution)

```csharp
// Like GroupBy but executes immediately (no deferred execution)
var lookup = products.ToLookup(p => p.Category);
var electronics = lookup["Electronics"]; // IEnumerable<Product>
var unknown = lookup["Unknown"];         // Empty (not null)
```

---

## Aggregation

```csharp
// Count
int total = products.Count();
int electronics = products.Count(p => p.Category == "Electronics");
long bigCount = products.LongCount();

// Sum
decimal totalPrice = products.Sum(p => p.Price);

// Average
decimal avgPrice = products.Average(p => p.Price);

// Min / Max
decimal cheapest = products.Min(p => p.Price);
decimal mostExpensive = products.Max(p => p.Price);

// MinBy / MaxBy (.NET 6+) - returns the element, not just the value
var cheapestProduct = products.MinBy(p => p.Price);
var mostExpensiveProduct = products.MaxBy(p => p.Price);
Console.WriteLine(cheapestProduct?.Name); // "Book"

// Aggregate (custom aggregation / fold)
var csv = products.Aggregate(
    seed: "",
    func: (acc, p) => acc == "" ? p.Name : $"{acc}, {p.Name}");
// "Laptop, Phone, Shirt, Book, Tablet"

// Aggregate with result selector
var summary = products.Aggregate(
    seed: new { Count = 0, Total = 0m },
    func: (acc, p) => new { Count = acc.Count + 1, Total = acc.Total + p.Price },
    resultSelector: acc => new { acc.Count, acc.Total, Average = acc.Total / acc.Count });
```

---

## Element Operators

```csharp
var products = new List<Product>
{
    new("Laptop", 999.99m, "Electronics"),
    new("Phone", 699.99m, "Electronics"),
};

// First / FirstOrDefault
var first = products.First();                                // Throws if empty
var firstElectronics = products.First(p => p.Category == "Electronics");
var firstOrNull = products.FirstOrDefault();                 // null if empty
var firstOrDefault = products.FirstOrDefault(p => p.Price > 2000); // null

// Single / SingleOrDefault (throws if more than one match)
var single = products.Single(p => p.Name == "Laptop");      // Throws if 0 or 2+
var singleOrNull = products.SingleOrDefault(p => p.Name == "Missing"); // null

// Last / LastOrDefault
var last = products.Last();
var lastOrNull = products.LastOrDefault(p => p.Price > 2000);

// ElementAt / ElementAtOrDefault
var second = products.ElementAt(1);
var outOfRange = products.ElementAtOrDefault(999); // null

// Default value override (.NET 6+)
var fallback = products.FirstOrDefault(
    p => p.Price > 5000,
    new Product("N/A", 0m, "None")); // Returns fallback instead of null

// Index from end (.NET 6+)
var lastItem = products.ElementAt(^1); // Same as products[^1] for arrays
```

---

## Set Operations

```csharp
var set1 = new[] { 1, 2, 3, 4, 5 };
var set2 = new[] { 3, 4, 5, 6, 7 };

// Distinct - remove duplicates
var unique = new[] { 1, 2, 2, 3, 3, 3 }.Distinct(); // [1, 2, 3]

// Union - all elements from both (no duplicates)
var union = set1.Union(set2); // [1, 2, 3, 4, 5, 6, 7]

// Intersect - elements in both
var intersect = set1.Intersect(set2); // [3, 4, 5]

// Except - elements in first but not second
var except = set1.Except(set2); // [1, 2]

// With custom comparer
var distinctProducts = products.Distinct(new ProductNameComparer());

// SequenceEqual - check if two sequences are identical
bool equal = set1.SequenceEqual(new[] { 1, 2, 3, 4, 5 }); // true
```

---

## Modern Set Operations

### DistinctBy, UnionBy, IntersectBy, ExceptBy (.NET 6+)

```csharp
// DistinctBy - distinct by a key selector (no custom comparer needed)
var uniqueByCategory = products.DistinctBy(p => p.Category);
// [Laptop (Electronics), Shirt (Clothing), Book (Books)] - first of each category

// UnionBy
var list1 = new[] { new Product("A", 10m, "X"), new Product("B", 20m, "Y") };
var list2 = new[] { new Product("C", 30m, "X"), new Product("D", 40m, "Z") };
var unionByCategory = list1.UnionBy(list2, p => p.Category);
// [A(X), B(Y), D(Z)] - first occurrence per category

// IntersectBy
var expensiveCategories = new[] { "Electronics", "Clothing" };
var inCategories = products.IntersectBy(expensiveCategories, p => p.Category);

// ExceptBy
var excludeCategories = new[] { "Electronics" };
var nonElectronics = products.ExceptBy(excludeCategories, p => p.Category);
```

---

## Chunk

Split a sequence into chunks of a specified size (.NET 6+).

```csharp
var numbers = Enumerable.Range(1, 10);

var chunks = numbers.Chunk(3);
// [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]

foreach (var chunk in chunks)
{
    Console.WriteLine($"Chunk: [{string.Join(", ", chunk)}]");
}

// Practical: batch database inserts
var allProducts = GetAllProducts(); // Thousands of products
foreach (var batch in allProducts.Chunk(100))
{
    await BulkInsertAsync(batch);
}

// Practical: parallel processing in batches
var batches = urls.Chunk(10);
foreach (var batch in batches)
{
    var tasks = batch.Select(url => httpClient.GetStringAsync(url));
    var results = await Task.WhenAll(tasks);
    ProcessResults(results);
}
```

---

## Quantifiers

```csharp
// Any - at least one element matches
bool hasExpensive = products.Any(p => p.Price > 500); // true
bool isEmpty = !products.Any(); // false

// All - every element matches
bool allActive = products.All(p => p.Price > 0); // true

// Contains
bool hasLaptop = products.Select(p => p.Name).Contains("Laptop"); // true

// Contains with comparer
var names = new[] { "LAPTOP", "phone" };
bool found = names.Contains("laptop", StringComparer.OrdinalIgnoreCase); // true
```

---

## Join Operations

### Inner Join

```csharp
var categories = new[]
{
    new { Id = 1, Name = "Electronics" },
    new { Id = 2, Name = "Clothing" },
    new { Id = 3, Name = "Books" }
};

var productsWithCat = new[]
{
    new { Name = "Laptop", CategoryId = 1, Price = 999.99m },
    new { Name = "Phone", CategoryId = 1, Price = 699.99m },
    new { Name = "Shirt", CategoryId = 2, Price = 29.99m },
    new { Name = "Orphan", CategoryId = 99, Price = 0m }, // No matching category
};

// Method syntax
var joined = productsWithCat.Join(
    categories,
    p => p.CategoryId,     // Outer key
    c => c.Id,             // Inner key
    (p, c) => new { p.Name, p.Price, Category = c.Name });
// [{ Laptop, 999.99, Electronics }, { Phone, 699.99, Electronics }, { Shirt, 29.99, Clothing }]

// Query syntax (often clearer for joins)
var joined2 =
    from p in productsWithCat
    join c in categories on p.CategoryId equals c.Id
    select new { p.Name, p.Price, Category = c.Name };
```

### GroupJoin (Left Outer Join)

```csharp
// Method syntax
var grouped = categories.GroupJoin(
    productsWithCat,
    c => c.Id,
    p => p.CategoryId,
    (category, prods) => new
    {
        Category = category.Name,
        Products = prods.ToList(),
        Count = prods.Count()
    });
// [{ Electronics, [Laptop, Phone], 2 }, { Clothing, [Shirt], 1 }, { Books, [], 0 }]

// Left outer join (query syntax)
var leftJoin =
    from c in categories
    join p in productsWithCat on c.Id equals p.CategoryId into grp
    from p in grp.DefaultIfEmpty()
    select new
    {
        Category = c.Name,
        ProductName = p?.Name ?? "(none)",
        Price = p?.Price ?? 0m
    };
```

### Cross Join

```csharp
var sizes = new[] { "S", "M", "L", "XL" };
var colors = new[] { "Red", "Blue", "Green" };

// All combinations
var variants = sizes.SelectMany(
    _ => colors,
    (size, color) => new { Size = size, Color = color });
// [{ S, Red }, { S, Blue }, { S, Green }, { M, Red }, ...]

// Query syntax
var variants2 =
    from size in sizes
    from color in colors
    select new { Size = size, Color = color };
```

---

## Deferred vs Immediate Execution

### Deferred Execution

The query is not executed until the results are enumerated.

```csharp
var query = products.Where(p => p.Price > 100); // No execution yet

products.Add(new Product("Camera", 599.99m, "Electronics"));

var results = query.ToList(); // Executes NOW - includes Camera
```

### Immediate Execution

These operators trigger execution immediately:

| Operator | Returns |
|----------|---------|
| `ToList()` | `List<T>` |
| `ToArray()` | `T[]` |
| `ToDictionary()` | `Dictionary<K,V>` |
| `ToHashSet()` | `HashSet<T>` |
| `ToLookup()` | `ILookup<K,V>` |
| `Count()` | `int` |
| `Sum()` / `Average()` / `Min()` / `Max()` | Scalar |
| `First()` / `Single()` / `Last()` | `T` |
| `Any()` / `All()` / `Contains()` | `bool` |
| `Aggregate()` | `T` |

### Avoiding Multiple Enumeration

```csharp
// BAD: Enumerates the query twice
IEnumerable<Product> expensive = products.Where(p => p.Price > 100);
var count = expensive.Count();     // First enumeration
var list = expensive.ToList();     // Second enumeration

// GOOD: Materialize once
var expensive = products.Where(p => p.Price > 100).ToList();
var count = expensive.Count;  // Property, not method - no re-enumeration
```

---

## Zip, Range, Repeat

### Zip

```csharp
var names = new[] { "Alice", "Bob", "Carol" };
var scores = new[] { 95, 87, 92 };

// Two sequences
var zipped = names.Zip(scores, (name, score) => new { name, score });
// [{ Alice, 95 }, { Bob, 87 }, { Carol, 92 }]

// Tuple zip (.NET 6+)
var zipped2 = names.Zip(scores);
// [(Alice, 95), (Bob, 87), (Carol, 92)]

// Three sequences (.NET 6+)
var ranks = new[] { 1, 2, 3 };
var zipped3 = names.Zip(scores, ranks);
// [(Alice, 95, 1), (Bob, 87, 2), (Carol, 92, 3)]
```

### Range and Repeat

```csharp
// Generate a range of numbers
var numbers = Enumerable.Range(1, 10); // [1, 2, 3, ..., 10]
var evens = Enumerable.Range(1, 50).Where(n => n % 2 == 0);

// Repeat a value
var fiveZeros = Enumerable.Repeat(0, 5); // [0, 0, 0, 0, 0]
var defaultProducts = Enumerable.Repeat(new Product("TBD", 0m, "None"), 10);

// Empty
IEnumerable<Product> empty = Enumerable.Empty<Product>();
```

---

## Custom Extension Methods

### Reusable LINQ Extensions

```csharp
public static class EnumerableExtensions
{
    // WhereIf - conditional filter
    public static IEnumerable<T> WhereIf<T>(
        this IEnumerable<T> source,
        bool condition,
        Func<T, bool> predicate)
    {
        return condition ? source.Where(predicate) : source;
    }

    // ForEach (side effects)
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source)
            action(item);
    }

    // Batch / Chunk (pre .NET 6)
    public static IEnumerable<IReadOnlyList<T>> Batch<T>(
        this IEnumerable<T> source, int batchSize)
    {
        var batch = new List<T>(batchSize);
        foreach (var item in source)
        {
            batch.Add(item);
            if (batch.Count >= batchSize)
            {
                yield return batch.AsReadOnly();
                batch = new List<T>(batchSize);
            }
        }
        if (batch.Count > 0)
            yield return batch.AsReadOnly();
    }

    // DistinctBy (pre .NET 6)
    public static IEnumerable<T> DistinctBy<T, TKey>(
        this IEnumerable<T> source, Func<T, TKey> keySelector)
    {
        var seen = new HashSet<TKey>();
        foreach (var item in source)
        {
            if (seen.Add(keySelector(item)))
                yield return item;
        }
    }

    // ToDelimitedString
    public static string ToDelimitedString<T>(
        this IEnumerable<T> source, string delimiter = ", ")
    {
        return string.Join(delimiter, source);
    }

    // None (opposite of Any)
    public static bool None<T>(this IEnumerable<T> source, Func<T, bool>? predicate = null)
    {
        return predicate is null ? !source.Any() : !source.Any(predicate);
    }
}

// Usage
var filtered = products
    .WhereIf(searchTerm is not null, p => p.Name.Contains(searchTerm!))
    .WhereIf(minPrice.HasValue, p => p.Price >= minPrice!.Value)
    .ToList();

var csv = products.Select(p => p.Name).ToDelimitedString("; ");
```

---

## LINQ to Objects vs LINQ to Entities

| Feature | LINQ to Objects | LINQ to Entities (EF Core) |
|---------|----------------|---------------------------|
| Execution | In-memory (C#) | Translated to SQL |
| Data source | `IEnumerable<T>` | `IQueryable<T>` |
| Custom methods | Fully supported | Must be translatable |
| String operations | All C# methods | Limited subset |
| Client evaluation | Always | Throws by default (EF Core 3+) |
| Lazy evaluation | Deferred | Deferred (until `ToList` etc.) |

### IQueryable vs IEnumerable

```csharp
// IQueryable - filter runs in the database
IQueryable<Product> queryable = context.Products;
var dbFiltered = queryable.Where(p => p.Price > 100).ToList();
// SQL: SELECT * FROM Products WHERE Price > 100

// IEnumerable - filter runs in memory (loads ALL rows first!)
IEnumerable<Product> enumerable = context.Products;
var memFiltered = enumerable.Where(p => p.Price > 100).ToList();
// SQL: SELECT * FROM Products  (loads everything, then filters in C#)
```

### Non-Translatable LINQ

```csharp
// FAILS with EF Core: custom method can't be translated to SQL
var results = context.Products
    .Where(p => MyCustomFilter(p)) // Not translatable!
    .ToList();

// WORKS: fetch first, then apply in-memory
var results = await context.Products
    .Where(p => p.IsActive)          // Translatable - runs in DB
    .ToListAsync();

var filtered = results
    .Where(p => MyCustomFilter(p))   // Runs in memory
    .ToList();
```

---

## Best Practices and Common Pitfalls

### Best Practices

| Practice | Description |
|----------|-------------|
| Use method syntax for simple chains | More concise and common in C# codebases |
| Use query syntax for complex joins | Clearer than nested lambda expressions |
| Materialize early to avoid multiple enumeration | Call `ToList()` before using result multiple times |
| Use `DistinctBy`, `MinBy`, `MaxBy` (.NET 6+) | Cleaner than manual `GroupBy` + `First` |
| Use `Chunk()` for batch processing | Built-in, efficient batching |
| Prefer `Any()` over `Count() > 0` | `Any` short-circuits on first match |
| Use `ToHashSet()` for O(1) lookups | Convert to set before repeated `Contains` calls |
| Use strongly-typed projections | Records or named tuples over anonymous types for public APIs |

### Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Multiple enumeration | Query runs multiple times (perf hit) | Materialize with `ToList()` |
| `Count() > 0` instead of `Any()` | Counts ALL elements when you only need one | Use `Any()` |
| `IEnumerable` from `IQueryable` | Database loads ALL rows, filters in memory | Keep as `IQueryable` until final materialization |
| LINQ in tight loops | Hidden allocations from closures/delegates | Pre-compute or use `for` loop |
| Null reference in `Select` | Element may be null | Use null-conditional or filter nulls first |
| `OrderBy` after `Where` on `IQueryable` | Order of operations matters for SQL | `OrderBy` after `Where` is fine; just be intentional |
| Forgetting deferred execution | Adding items after query definition changes results | Materialize before modifying source |
| Using LINQ for simple operations | Overhead for tiny collections | Use direct loop for 1-3 elements |
| `Single` without uniqueness guarantee | Throws at runtime if > 1 match | Ensure uniqueness or use `First` |
| Capturing variables in closures | Variable changes affect deferred query | Copy to local variable |
