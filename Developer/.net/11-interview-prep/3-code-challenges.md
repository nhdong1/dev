# Code Challenges — Bài Tập Lập Trình Phỏng Vấn

> Các bài tập lập trình thường gặp trong phỏng vấn .NET Developer, tập trung vào C# idioms, LINQ, async, và design patterns. Mỗi bài có hướng dẫn tiếp cận, code mẫu và phân tích độ phức tạp.

---

## Cách Tiếp Cận Live Coding

```
1. Đọc kỹ đề → Tóm tắt lại bằng lời → Hỏi clarifying questions
2. Nêu approach trước khi code: "Tôi sẽ làm X vì Y"
3. Viết test case (input/output) trước khi code
4. Code từng bước nhỏ, giải thích từng dòng
5. Test với edge cases: null, empty, boundary values
6. Nêu time/space complexity
7. Đề xuất cách tối ưu (nếu có)
```

---

## Chủ Đề 1 — String & Char Manipulation

### Bài 1.1 — Palindrome — Kiểm Tra Chuỗi Đối Xứng

**Đề:** Viết hàm kiểm tra một string có phải palindrome không (đọc xuôi hay ngược đều như nhau). Bỏ qua khoảng trắng và không phân biệt hoa thường.

**Ví dụ:**
```
"racecar"     → true
"A man a plan" → true  (bỏ khoảng trắng: "amanaplanacanalpanama")
"hello"       → false
""            → true   (edge case!)
null          → throw ArgumentNullException
```

**Approach:** Two-pointer — hai con trỏ từ hai đầu, di chuyển vào giữa.

```csharp
// Solution 1: Two-pointer — O(n) time, O(1) space
public static bool IsPalindrome(string s)
{
    ArgumentNullException.ThrowIfNull(s);
    
    // Normalize: loại bỏ khoảng trắng, lowercase
    var cleaned = s.Where(char.IsLetterOrDigit)
                   .Select(char.ToLower)
                   .ToArray();
    
    int left = 0, right = cleaned.Length - 1;
    
    while (left < right)
    {
        if (cleaned[left] != cleaned[right])
            return false;
        left++;
        right--;
    }
    
    return true;
}

// Solution 2: LINQ one-liner — O(n) time, O(n) space (elegant, interview-friendly)
public static bool IsPalindromeLINQ(string s)
{
    var cleaned = s?.Where(char.IsLetterOrDigit)
                    .Select(char.ToLower)
                    .ToArray()
                  ?? throw new ArgumentNullException(nameof(s));
    
    return cleaned.SequenceEqual(cleaned.Reverse());
}

// Solution 3: Span<T> — O(n) time, O(1) space (zero-allocation — senior bonus!)
public static bool IsPalindromeSpan(ReadOnlySpan<char> s)
{
    int left = 0, right = s.Length - 1;
    
    while (left < right)
    {
        while (left < right && !char.IsLetterOrDigit(s[left])) left++;
        while (left < right && !char.IsLetterOrDigit(s[right])) right--;
        
        if (char.ToLower(s[left]) != char.ToLower(s[right]))
            return false;
        
        left++;
        right--;
    }
    
    return true;
}

// Unit Tests
[Theory]
[InlineData("racecar", true)]
[InlineData("hello", false)]
[InlineData("", true)]
[InlineData("A man a plan", true)]
public void IsPalindrome_Tests(string input, bool expected)
    => Assert.Equal(expected, IsPalindrome(input));
```

---

### Bài 1.2 — Anagram — Kiểm Tra Từ Đảo Chữ

**Đề:** Hai string có phải anagram của nhau không? (Cùng ký tự, khác thứ tự)

```
"listen" vs "silent" → true
"hello" vs "world"   → false
"" vs ""             → true
```

```csharp
// Approach 1: Sort and Compare — O(n log n)
public static bool IsAnagram(string s1, string s2)
{
    if (s1 == null || s2 == null)
        throw new ArgumentNullException();
    
    if (s1.Length != s2.Length) return false;
    
    return s1.Order().SequenceEqual(s2.Order());
}

// Approach 2: Frequency Count — O(n) time, O(1) space (alphabet size = 26)
public static bool IsAnagramOptimal(string s1, string s2)
{
    if (s1.Length != s2.Length) return false;
    
    var freq = new int[128]; // ASCII
    
    foreach (char c in s1) freq[c]++;
    foreach (char c in s2)
    {
        freq[c]--;
        if (freq[c] < 0) return false; // Sớm trả về false
    }
    
    return true;
}
```

---

### Bài 1.3 — Group Anagrams — Nhóm Các Từ Đảo Chữ

**Đề:** Cho mảng string, nhóm các anagram lại với nhau.

```
Input:  ["eat", "tea", "tan", "ate", "nat", "bat"]
Output: [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

```csharp
// LINQ GroupBy solution
public static IEnumerable<IGrouping<string, string>> GroupAnagrams(string[] words)
{
    return words.GroupBy(w => string.Concat(w.Order()));
}

// Dictionary solution — O(n * k log k) where k = avg word length
public static List<List<string>> GroupAnagramsDict(string[] words)
{
    var groups = new Dictionary<string, List<string>>();
    
    foreach (var word in words)
    {
        var key = string.Concat(word.Order()); // Sort characters
        
        if (!groups.TryGetValue(key, out var group))
        {
            group = new List<string>();
            groups[key] = group;
        }
        
        group.Add(word);
    }
    
    return groups.Values.ToList();
}
```

---

## Chủ Đề 2 — Collections & LINQ

### Bài 2.1 — Top N Elements — Lấy N Phần Tử Lớn Nhất

**Đề:** Từ danh sách số nguyên, tìm K số lớn nhất. Xử lý duplicates.

```csharp
// Approach 1: Sort — O(n log n)
public static IEnumerable<int> TopK_Sort(IEnumerable<int> nums, int k)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(k);
    return nums.OrderByDescending(x => x).Take(k);
}

// Approach 2: Min-Heap — O(n log k) — tốt hơn khi k << n
public static IEnumerable<int> TopK_Heap(IEnumerable<int> nums, int k)
{
    // .NET không có built-in min-heap, dùng PriorityQueue
    var pq = new PriorityQueue<int, int>(k + 1); // Min-heap
    
    foreach (var num in nums)
    {
        pq.Enqueue(num, num);
        if (pq.Count > k)
            pq.Dequeue(); // Loại bỏ phần tử nhỏ nhất
    }
    
    return pq.UnorderedItems.Select(x => x.Element).OrderByDescending(x => x);
}

// Thực tế interview: LINQ thường là đáp án được chấp nhận
var top5 = products
    .GroupBy(p => p.CategoryId)
    .Select(g => new
    {
        CategoryId = g.Key,
        TopProducts = g.OrderByDescending(p => p.Sales).Take(5).ToList()
    });
```

---

### Bài 2.2 — Flatten Nested List — Làm Phẳng Danh Sách Lồng Nhau

**Đề:** Flatten một `IEnumerable<IEnumerable<T>>` thành `IEnumerable<T>`.

```csharp
// LINQ SelectMany
var nested = new List<List<int>> { new() {1,2}, new() {3,4,5}, new() {6} };
var flat = nested.SelectMany(x => x).ToList(); // [1,2,3,4,5,6]

// Recursive flatten cho cây (tree) — O(n)
public static IEnumerable<T> Flatten<T>(IEnumerable<IEnumerable<T>> source)
    => source.SelectMany(x => x);

// Flatten với depth — cho nested object graph
public static IEnumerable<Category> FlattenTree(IEnumerable<Category> categories)
{
    foreach (var cat in categories)
    {
        yield return cat;
        foreach (var child in FlattenTree(cat.Children))
            yield return child;
    }
}
```

---

### Bài 2.3 — LINQ Complex Query

**Đề:** Cho dữ liệu orders, tính tổng doanh thu theo category, chỉ lấy categories có doanh thu > 10,000, sắp xếp giảm dần.

```csharp
public class Order
{
    public int Id { get; set; }
    public string Category { get; set; } = "";
    public decimal Amount { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsCompleted { get; set; }
}

// Query phức tạp — hay được yêu cầu viết trong interview
var report = orders
    .Where(o => o.IsCompleted && o.CreatedAt.Year == 2024)
    .GroupBy(o => o.Category)
    .Select(g => new
    {
        Category = g.Key,
        TotalRevenue = g.Sum(o => o.Amount),
        OrderCount = g.Count(),
        AverageOrderValue = g.Average(o => o.Amount)
    })
    .Where(x => x.TotalRevenue > 10_000)
    .OrderByDescending(x => x.TotalRevenue)
    .ToList();

// Method syntax vs Query syntax — interview thường hỏi cả hai
// Method syntax (preferred in industry)
var result1 = orders.Where(o => o.Amount > 100).Select(o => o.Category);

// Query syntax (SQL-like, dễ đọc hơn cho complex)
var result2 = from o in orders
              where o.Amount > 100
              select o.Category;
```

---

## Chủ Đề 3 — Async & Concurrency

### Bài 3.1 — Parallel Processing — Xử Lý Song Song

**Đề:** Tải dữ liệu từ nhiều API cùng lúc, giới hạn concurrency để không overwhelm server.

```csharp
// Approach 1: Task.WhenAll — tất cả parallel, không giới hạn
public async Task<List<ProductDto>> GetAllProductsAsync(int[] productIds)
{
    var tasks = productIds.Select(id => _apiClient.GetProductAsync(id));
    var results = await Task.WhenAll(tasks);
    return results.ToList();
}

// Approach 2: SemaphoreSlim — giới hạn concurrency (phổ biến trong interview)
public async Task<List<ProductDto>> GetProductsWithThrottlingAsync(
    int[] productIds, 
    int maxConcurrency = 5)
{
    var semaphore = new SemaphoreSlim(maxConcurrency);
    var results = new ConcurrentBag<ProductDto>();
    
    var tasks = productIds.Select(async id =>
    {
        await semaphore.WaitAsync();
        try
        {
            var product = await _apiClient.GetProductAsync(id);
            results.Add(product);
        }
        finally
        {
            semaphore.Release();
        }
    });
    
    await Task.WhenAll(tasks);
    return results.ToList();
}

// Approach 3: Parallel.ForEachAsync (.NET 6+) — elegant
public async Task ProcessProductsAsync(int[] ids)
{
    await Parallel.ForEachAsync(
        ids,
        new ParallelOptions { MaxDegreeOfParallelism = 5 },
        async (id, ct) =>
        {
            var product = await _apiClient.GetProductAsync(id, ct);
            await _db.SaveProductAsync(product, ct);
        });
}
```

---

### Bài 3.2 — Retry với Exponential Backoff — Thử Lại Với Chờ Tăng Dần

**Đề:** Implement retry logic cho HTTP calls với exponential backoff và jitter.

```csharp
public async Task<T> ExecuteWithRetryAsync<T>(
    Func<CancellationToken, Task<T>> operation,
    int maxRetries = 3,
    CancellationToken cancellationToken = default)
{
    for (int attempt = 0; attempt <= maxRetries; attempt++)
    {
        try
        {
            return await operation(cancellationToken);
        }
        catch (HttpRequestException ex) when (attempt < maxRetries)
        {
            // Exponential backoff với jitter — tránh thundering herd
            var baseDelay = TimeSpan.FromSeconds(Math.Pow(2, attempt)); // 1s, 2s, 4s
            var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000));
            var delay = baseDelay + jitter;
            
            _logger.LogWarning(ex, 
                "Attempt {Attempt}/{MaxRetries} failed. Retrying in {Delay}ms",
                attempt + 1, maxRetries, delay.TotalMilliseconds);
            
            await Task.Delay(delay, cancellationToken);
        }
    }
    
    throw new InvalidOperationException($"Operation failed after {maxRetries} retries");
}

// Sử dụng Polly library (industry standard)
// builder.Services.AddHttpClient<IApiClient, ApiClient>()
//     .AddPolicyHandler(Policy
//         .Handle<HttpRequestException>()
//         .WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))));
```

---

### Bài 3.3 — Producer-Consumer với Channel — Kênh Sản Xuất-Tiêu Thụ

**Đề:** Implement pipeline xử lý file với producer đọc file và multiple consumers xử lý.

```csharp
public class FileProcessingPipeline
{
    public async Task RunAsync(string[] filePaths, int workerCount = 4,
        CancellationToken cancellationToken = default)
    {
        // Channel<T> — bounded để back-pressure
        var channel = Channel.CreateBounded<string>(new BoundedChannelOptions(100)
        {
            SingleWriter = true,
            SingleReader = false,
            FullMode = BoundedChannelFullMode.Wait
        });
        
        // Producer — đọc và đẩy vào channel
        var producerTask = Task.Run(async () =>
        {
            try
            {
                foreach (var path in filePaths)
                {
                    await channel.Writer.WriteAsync(path, cancellationToken);
                }
            }
            finally
            {
                channel.Writer.Complete(); // Signal: không còn item nào nữa
            }
        }, cancellationToken);
        
        // Multiple consumers — xử lý song song
        var consumerTasks = Enumerable.Range(0, workerCount)
            .Select(_ => Task.Run(async () =>
            {
                await foreach (var filePath in channel.Reader.ReadAllAsync(cancellationToken))
                {
                    await ProcessFileAsync(filePath, cancellationToken);
                }
            }, cancellationToken));
        
        await Task.WhenAll(new[] { producerTask }.Concat(consumerTasks));
    }
    
    private async Task ProcessFileAsync(string path, CancellationToken ct)
    {
        var content = await File.ReadAllTextAsync(path, ct);
        // Xử lý nội dung file...
        _logger.LogInformation("Processed: {Path}", path);
    }
}
```

---

## Chủ Đề 4 — Design Patterns

### Bài 4.1 — Repository Pattern với Generics

**Đề:** Implement generic repository pattern cho EF Core.

```csharp
// Interface — abstraction
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    void Update(T entity);
    void Delete(T entity);
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

// Generic implementation
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly DbContext _db;
    protected readonly DbSet<T> _dbSet;
    
    public Repository(DbContext db)
    {
        _db = db;
        _dbSet = db.Set<T>();
    }
    
    public async Task<T?> GetByIdAsync(int id, CancellationToken ct = default)
        => await _dbSet.FindAsync(new object[] { id }, ct);
    
    public async Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default)
        => await _dbSet.AsNoTracking().ToListAsync(ct);
    
    public async Task<IReadOnlyList<T>> FindAsync(
        Expression<Func<T, bool>> predicate, CancellationToken ct = default)
        => await _dbSet.AsNoTracking().Where(predicate).ToListAsync(ct);
    
    public async Task AddAsync(T entity, CancellationToken ct = default)
        => await _dbSet.AddAsync(entity, ct);
    
    public void Update(T entity) => _db.Entry(entity).State = EntityState.Modified;
    
    public void Delete(T entity) => _dbSet.Remove(entity);
    
    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => _db.SaveChangesAsync(ct);
}

// Specific repository với logic riêng
public interface IOrderRepository : IRepository<Order>
{
    Task<List<Order>> GetOrdersByCustomerAsync(int customerId, CancellationToken ct = default);
    Task<decimal> GetTotalRevenueAsync(DateTime from, DateTime to, CancellationToken ct = default);
}

public class OrderRepository : Repository<Order>, IOrderRepository
{
    public OrderRepository(AppDbContext db) : base(db) { }
    
    public async Task<List<Order>> GetOrdersByCustomerAsync(int customerId, CancellationToken ct)
        => await _dbSet
            .AsNoTracking()
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync(ct);
    
    public async Task<decimal> GetTotalRevenueAsync(DateTime from, DateTime to, CancellationToken ct)
        => await _dbSet
            .Where(o => o.CreatedAt >= from && o.CreatedAt <= to && o.Status == OrderStatus.Completed)
            .SumAsync(o => o.TotalAmount, ct);
}
```

---

### Bài 4.2 — Builder Pattern — Mẫu Xây Dựng

**Đề:** Implement fluent builder cho email message.

```csharp
public class EmailMessage
{
    public string From { get; private set; } = "";
    public List<string> To { get; private set; } = new();
    public string Subject { get; private set; } = "";
    public string HtmlBody { get; private set; } = "";
    public List<string> Attachments { get; private set; } = new();
    public bool IsHighPriority { get; private set; }
    
    private EmailMessage() { }
    
    public static EmailBuilder Create() => new EmailBuilder();
    
    public class EmailBuilder
    {
        private readonly EmailMessage _email = new();
        
        public EmailBuilder From(string email)
        {
            _email.From = email;
            return this;
        }
        
        public EmailBuilder To(params string[] emails)
        {
            _email.To.AddRange(emails);
            return this;
        }
        
        public EmailBuilder WithSubject(string subject)
        {
            _email.Subject = subject;
            return this;
        }
        
        public EmailBuilder WithHtmlBody(string html)
        {
            _email.HtmlBody = html;
            return this;
        }
        
        public EmailBuilder WithAttachment(string filePath)
        {
            _email.Attachments.Add(filePath);
            return this;
        }
        
        public EmailBuilder HighPriority()
        {
            _email.IsHighPriority = true;
            return this;
        }
        
        public EmailMessage Build()
        {
            if (string.IsNullOrEmpty(_email.From))
                throw new InvalidOperationException("From email is required");
            if (_email.To.Count == 0)
                throw new InvalidOperationException("At least one recipient is required");
            if (string.IsNullOrEmpty(_email.Subject))
                throw new InvalidOperationException("Subject is required");
            
            return _email;
        }
    }
}

// Sử dụng — fluent API
var email = EmailMessage.Create()
    .From("noreply@myapp.com")
    .To("user@example.com", "admin@example.com")
    .WithSubject("Order Confirmation #12345")
    .WithHtmlBody("<h1>Thank you for your order!</h1>")
    .WithAttachment("/invoices/12345.pdf")
    .HighPriority()
    .Build();
```

---

### Bài 4.3 — Observer Pattern — Mẫu Quan Sát

**Đề:** Implement event notification khi inventory thay đổi.

```csharp
// Event args
public class InventoryChangedEventArgs : EventArgs
{
    public int ProductId { get; init; }
    public int OldQuantity { get; init; }
    public int NewQuantity { get; init; }
    public bool IsLowStock => NewQuantity < 10;
}

// Subject — nguồn phát sự kiện
public class InventoryService
{
    // event delegate — sự kiện
    public event EventHandler<InventoryChangedEventArgs>? InventoryChanged;
    
    private readonly Dictionary<int, int> _inventory = new();
    
    public void UpdateStock(int productId, int quantity)
    {
        var oldQuantity = _inventory.GetValueOrDefault(productId);
        _inventory[productId] = quantity;
        
        // Phát sự kiện cho tất cả subscribers
        OnInventoryChanged(new InventoryChangedEventArgs
        {
            ProductId = productId,
            OldQuantity = oldQuantity,
            NewQuantity = quantity
        });
    }
    
    protected virtual void OnInventoryChanged(InventoryChangedEventArgs args)
        => InventoryChanged?.Invoke(this, args);
}

// Observers — người lắng nghe
public class LowStockAlertService
{
    public void Subscribe(InventoryService inventory)
        => inventory.InventoryChanged += HandleInventoryChanged;
    
    private void HandleInventoryChanged(object? sender, InventoryChangedEventArgs e)
    {
        if (e.IsLowStock)
            Console.WriteLine($"⚠️ LOW STOCK: Product {e.ProductId} — only {e.NewQuantity} left!");
    }
}

// Sử dụng
var inventoryService = new InventoryService();
var alertService = new LowStockAlertService();
alertService.Subscribe(inventoryService);

inventoryService.UpdateStock(productId: 42, quantity: 5); // Trigger LOW STOCK alert
```

---

## Chủ Đề 5 — Algorithms & Data Structures

### Bài 5.1 — Two Sum — Tổng Hai Số

**Đề:** Tìm hai số trong mảng có tổng bằng target. Trả về indices.

```csharp
// Naive — O(n²)
public static (int, int)? TwoSumBrute(int[] nums, int target)
{
    for (int i = 0; i < nums.Length; i++)
        for (int j = i + 1; j < nums.Length; j++)
            if (nums[i] + nums[j] == target)
                return (i, j);
    return null;
}

// HashSet — O(n) time, O(n) space — tốt hơn nhiều
public static (int, int)? TwoSum(int[] nums, int target)
{
    var seen = new Dictionary<int, int>(); // value → index
    
    for (int i = 0; i < nums.Length; i++)
    {
        int complement = target - nums[i];
        
        if (seen.TryGetValue(complement, out int j))
            return (j, i);
        
        seen[nums[i]] = i;
    }
    
    return null;
}

// Test cases
[Theory]
[InlineData(new[] {2,7,11,15}, 9, 0, 1)]   // 2 + 7 = 9
[InlineData(new[] {3,2,4}, 6, 1, 2)]        // 2 + 4 = 6
[InlineData(new[] {3,3}, 6, 0, 1)]          // duplicates
public void TwoSum_Tests(int[] nums, int target, int expectedI, int expectedJ)
{
    var result = TwoSum(nums, target);
    Assert.Equal((expectedI, expectedJ), result);
}
```

---

### Bài 5.2 — LRU Cache — Cache Ít Dùng Gần Đây Nhất

**Đề:** Implement LRU Cache — Least Recently Used Cache — với O(1) get và put.

```csharp
public class LRUCache<TKey, TValue> where TKey : notnull
{
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<(TKey Key, TValue Value)>> _map;
    private readonly LinkedList<(TKey Key, TValue Value)> _list;
    
    public LRUCache(int capacity)
    {
        _capacity = capacity;
        _map = new Dictionary<TKey, LinkedListNode<(TKey, TValue)>>(capacity);
        _list = new LinkedList<(TKey, TValue)>();
    }
    
    public bool TryGet(TKey key, out TValue? value)
    {
        if (!_map.TryGetValue(key, out var node))
        {
            value = default;
            return false;
        }
        
        // Move to front (most recently used)
        _list.Remove(node);
        _list.AddFirst(node);
        
        value = node.Value.Value;
        return true;
    }
    
    public void Put(TKey key, TValue value)
    {
        if (_map.TryGetValue(key, out var existingNode))
        {
            _list.Remove(existingNode);
            _map.Remove(key);
        }
        
        if (_map.Count >= _capacity)
        {
            // Evict least recently used (tail)
            var lru = _list.Last!;
            _list.RemoveLast();
            _map.Remove(lru.Value.Key);
        }
        
        var newNode = _list.AddFirst((key, value));
        _map[key] = newNode;
    }
}

// Sử dụng
var cache = new LRUCache<int, string>(capacity: 3);
cache.Put(1, "one");
cache.Put(2, "two");
cache.Put(3, "three");
cache.TryGet(1, out _); // Access 1 → now MRU
cache.Put(4, "four");   // Evicts 2 (LRU)
```

---

### Bài 5.3 — Fibonacci với Memoization — Ghi Nhớ Kết Quả

**Đề:** Tính Fibonacci — dãy số Fibonacci — hiệu quả.

```csharp
// Naive recursive — O(2^n) — KHÔNG dùng trong production
public static long FibNaive(int n) =>
    n <= 1 ? n : FibNaive(n - 1) + FibNaive(n - 2);

// Memoization — O(n) time, O(n) space
public static long FibMemo(int n, Dictionary<int, long>? memo = null)
{
    memo ??= new Dictionary<int, long>();
    if (n <= 1) return n;
    if (memo.TryGetValue(n, out var cached)) return cached;
    
    memo[n] = FibMemo(n - 1, memo) + FibMemo(n - 2, memo);
    return memo[n];
}

// Bottom-up DP — O(n) time, O(1) space — optimal
public static long FibIterative(int n)
{
    if (n <= 1) return n;
    
    long prev = 0, curr = 1;
    for (int i = 2; i <= n; i++)
        (prev, curr) = (curr, prev + curr);
    
    return curr;
}

// C# một dòng với Enumerable.Aggregate (elegant, interview bonus)
public static long FibLinq(int n) => n <= 1 ? n :
    Enumerable.Range(2, n - 1)
        .Aggregate((prev: 0L, curr: 1L), (acc, _) => (acc.curr, acc.prev + acc.curr))
        .curr;
```

---

## Chủ Đề 6 — Real-world .NET Challenges

### Bài 6.1 — Middleware Phân Tích Request

**Đề:** Viết middleware log request/response với timing, không ảnh hưởng performance.

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;
    
    public RequestLoggingMiddleware(RequestDelegate next, 
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        var requestId = Activity.Current?.Id ?? context.TraceIdentifier;
        
        // Log request
        _logger.LogInformation(
            "Request {RequestId}: {Method} {Path}",
            requestId,
            context.Request.Method,
            context.Request.Path);
        
        // Capture response status
        var originalBodyStream = context.Response.Body;
        try
        {
            await _next(context);
        }
        finally
        {
            sw.Stop();
            
            _logger.LogInformation(
                "Response {RequestId}: {StatusCode} in {ElapsedMs}ms",
                requestId,
                context.Response.StatusCode,
                sw.ElapsedMilliseconds);
            
            // Log slow requests
            if (sw.ElapsedMilliseconds > 1000)
                _logger.LogWarning(
                    "SLOW REQUEST {RequestId}: {Method} {Path} took {ElapsedMs}ms",
                    requestId,
                    context.Request.Method,
                    context.Request.Path,
                    sw.ElapsedMilliseconds);
        }
    }
}
```

---

### Bài 6.2 — Generic Result Pattern — Mẫu Kết Quả Tổng Quát

**Đề:** Implement Result type để tránh exceptions cho business logic.

```csharp
// Result<T> — tránh throw exception cho business errors
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }
    public string? ErrorCode { get; }
    
    private Result(T value) { IsSuccess = true; Value = value; }
    private Result(string error, string? errorCode = null)
    {
        IsSuccess = false;
        Error = error;
        ErrorCode = errorCode;
    }
    
    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error, string? code = null) => new(error, code);
    
    // Monadic operations — chuỗi kết quả
    public Result<TNext> Map<TNext>(Func<T, TNext> mapper)
        => IsSuccess ? Result<TNext>.Success(mapper(Value!)) 
                     : Result<TNext>.Failure(Error!, ErrorCode);
    
    public async Task<Result<TNext>> MapAsync<TNext>(Func<T, Task<TNext>> mapper)
        => IsSuccess ? Result<TNext>.Success(await mapper(Value!)) 
                     : Result<TNext>.Failure(Error!, ErrorCode);
}

// Sử dụng trong service
public async Task<Result<UserDto>> CreateUserAsync(CreateUserRequest request)
{
    if (await _db.Users.AnyAsync(u => u.Email == request.Email))
        return Result<UserDto>.Failure("Email already registered", "EMAIL_EXISTS");
    
    var user = new User { Email = request.Email, Name = request.Name };
    _db.Users.Add(user);
    await _db.SaveChangesAsync();
    
    return Result<UserDto>.Success(new UserDto(user.Id, user.Email, user.Name));
}

// Trong controller
[HttpPost]
public async Task<IActionResult> CreateUser(CreateUserRequest request)
{
    var result = await _userService.CreateUserAsync(request);
    
    return result.IsSuccess
        ? CreatedAtAction(nameof(GetUser), new { id = result.Value!.Id }, result.Value)
        : Conflict(new { error = result.Error, code = result.ErrorCode });
}
```

---

## 📋 Checklist Code Challenge

### Trước Khi Code

- [ ] Đọc lại đề 2 lần
- [ ] Viết ra input/output examples bằng tay
- [ ] Hỏi về constraints (null? empty? negative numbers? very large input?)
- [ ] Nêu approach trước khi gõ

### Khi Code

- [ ] Đặt tên biến rõ ràng (không dùng a, b, x, y)
- [ ] Comment khi logic không tự giải thích được
- [ ] Handle edge cases: null, empty, single element
- [ ] Giải thích từng bước quan trọng

### Sau Khi Code

- [ ] Test với ví dụ đề đã cho
- [ ] Test edge cases: null, empty, min/max values
- [ ] Nêu time complexity: O(?)
- [ ] Nêu space complexity: O(?)
- [ ] Đề xuất optimization nếu có

### Complexity Cheat Sheet

```
O(1)       — Constant: HashMap lookup, array index
O(log n)   — Logarithmic: Binary search
O(n)       — Linear: Single loop
O(n log n) — Linearithmic: Sorting (MergeSort, QuickSort avg)
O(n²)      — Quadratic: Nested loops
O(2^n)     — Exponential: Naive recursion (Fibonacci)
```

---

**Cập Nhật Lần Cuối:** 2026-06-02
**Phiên Bản:** 1.0
