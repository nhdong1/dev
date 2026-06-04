# Azure Services cho .NET — App Service, AKS, Azure Functions, Azure SQL

> Microsoft Azure cung cấp hệ sinh thái dịch vụ cloud toàn diện cho ứng dụng .NET. Bài này trình bày các dịch vụ Azure quan trọng nhất: App Service — PaaS cho web app, AKS — Kubernetes được quản lý, Azure Functions — serverless computing, Azure SQL — database managed service, và các dịch vụ hỗ trợ như Azure Cache for Redis, Azure Service Bus.

---

## 1. So Sánh Các Hướng Triển Khai .NET trên Azure

```
┌──────────────────────────────────────────────────────────────────────┐
│                   MỨC ĐỘ KIỂM SOÁT vs ĐỘ PHỨC TẠP                  │
│                                                                      │
│  Kiểm soát                                                           │
│  nhiều  │                                           Azure VMs        │
│         │                              AKS ────────►  (IaaS)         │
│         │              App Service                                   │
│         │              (PaaS) ─────►  Container                      │
│         │   Azure                     Apps                           │
│         │   Functions ─────────────►                                 │
│  ít     │   (Serverless)                                             │
│         └──────────────────────────────────────────────────────────► │
│              Ít phức tạp                              Rất phức tạp   │
└──────────────────────────────────────────────────────────────────────┘
```

| Dịch Vụ | Loại | Khi Nào Dùng | Chi Phí Cơ Sở |
|---------|------|-------------|----------------|
| **Azure Functions** | Serverless | Event-driven, short tasks | Theo lần gọi |
| **Azure App Service** | PaaS | Web app, API đơn giản-trung bình | Theo plan |
| **Azure Container Apps** | PaaS + Container | Microservices không muốn quản lý K8s | Theo CPU/mem |
| **Azure Kubernetes Service** | Managed K8s | Enterprise, nhiều microservices | Theo node VMs |
| **Azure VMs** | IaaS | Full control cần thiết | Theo VM size |

---

## 2. Azure App Service — PaaS cho Web App

### Khái Niệm

- **App Service Plan** — Kế Hoạch Dịch Vụ: định nghĩa tài nguyên (CPU, RAM, scale) cho nhiều Apps
- **Web App**: thực thể ứng dụng — có thể là .NET, Node.js, Python, Java, PHP
- **Deployment Slot** — Slot Triển Khai: môi trường staging để test trước khi swap sang production

### Tạo App Service qua Azure CLI

```bash
# Tạo Resource Group — nhóm tài nguyên
az group create \
    --name myapp-rg \
    --location eastasia

# Tạo App Service Plan (tier Standard S1)
az appservice plan create \
    --name myapp-plan \
    --resource-group myapp-rg \
    --sku S1 \
    --is-linux

# Tạo Web App cho .NET 8
az webapp create \
    --resource-group myapp-rg \
    --plan myapp-plan \
    --name myapp-api \
    --runtime "DOTNETCORE:8.0"

# Deploy từ Docker image
az webapp create \
    --resource-group myapp-rg \
    --plan myapp-plan \
    --name myapp-api \
    --deployment-container-image-name myacr.azurecr.io/myapp-api:latest
```

### Cấu Hình App Settings — Biến Môi Trường

```bash
# Set connection string (được encrypt — mã hóa ở rest)
az webapp config connection-string set \
    --resource-group myapp-rg \
    --name myapp-api \
    --settings DefaultConnection="Server=myserver..." \
    --connection-string-type SQLAzure

# Set app settings (environment variables)
az webapp config appsettings set \
    --resource-group myapp-rg \
    --name myapp-api \
    --settings \
        ASPNETCORE_ENVIRONMENT=Production \
        Redis__ConnectionString="myredis.redis.cache.windows.net:6380..."
```

### Deployment Slots — Triển Khai Không Gián Đoạn

```bash
# Tạo staging slot
az webapp deployment slot create \
    --resource-group myapp-rg \
    --name myapp-api \
    --slot staging

# Deploy phiên bản mới vào staging
az webapp deployment source config-zip \
    --resource-group myapp-rg \
    --name myapp-api \
    --slot staging \
    --src ./publish.zip

# Swap staging → production (zero-downtime)
az webapp deployment slot swap \
    --resource-group myapp-rg \
    --name myapp-api \
    --slot staging \
    --target-slot production

# Rollback: swap lại nếu có vấn đề
az webapp deployment slot swap \
    --resource-group myapp-rg \
    --name myapp-api \
    --slot production \
    --target-slot staging
```

### Auto-Scale — Tự Động Mở Rộng

```bash
# Tạo autoscale rule dựa trên CPU
az monitor autoscale create \
    --resource-group myapp-rg \
    --resource myapp-plan \
    --resource-type Microsoft.Web/serverfarms \
    --name myapp-autoscale \
    --min-count 2 \
    --max-count 10 \
    --count 2

# Scale out khi CPU > 70%
az monitor autoscale rule create \
    --resource-group myapp-rg \
    --autoscale-name myapp-autoscale \
    --scale out 2 \
    --condition "Percentage CPU > 70 avg 5m"

# Scale in khi CPU < 30%
az monitor autoscale rule create \
    --resource-group myapp-rg \
    --autoscale-name myapp-autoscale \
    --scale in 1 \
    --condition "Percentage CPU < 30 avg 10m"
```

---

## 3. Azure Kubernetes Service — AKS

### AKS là gì?

AKS — Azure Kubernetes Service — là Kubernetes được quản lý hoàn toàn trên Azure. Microsoft quản lý control plane — mặt phẳng điều khiển (API server, etcd, scheduler), bạn chỉ trả tiền cho worker nodes.

### Tạo AKS Cluster

```bash
# Tạo AKS cluster
az aks create \
    --resource-group myapp-rg \
    --name myapp-aks \
    --node-count 3 \
    --node-vm-size Standard_D2s_v3 \
    --enable-managed-identity \
    --enable-addons monitoring \
    --generate-ssh-keys \
    --kubernetes-version 1.29.0

# Lấy credentials để dùng kubectl
az aks get-credentials \
    --resource-group myapp-rg \
    --name myapp-aks

# Kiểm tra kết nối
kubectl get nodes
```

### Tích Hợp ACR — Azure Container Registry

```bash
# Tạo Azure Container Registry — Registry Lưu Trữ Docker Image
az acr create \
    --resource-group myapp-rg \
    --name myacr \
    --sku Basic

# Build và push image lên ACR
az acr build \
    --registry myacr \
    --image myapp-api:1.0.0 \
    --file Dockerfile .

# Cấp quyền cho AKS pull image từ ACR
az aks update \
    --name myapp-aks \
    --resource-group myapp-rg \
    --attach-acr myacr
```

### Node Pools — Nhóm Node

```bash
# Thêm node pool với GPU cho workloads đặc biệt
az aks nodepool add \
    --resource-group myapp-rg \
    --cluster-name myapp-aks \
    --name gpupool \
    --node-count 1 \
    --node-vm-size Standard_NC6

# Thêm spot node pool — rẻ hơn nhưng có thể bị thu hồi
az aks nodepool add \
    --resource-group myapp-rg \
    --cluster-name myapp-aks \
    --name spotpool \
    --priority Spot \
    --eviction-policy Delete \
    --spot-max-price -1 \
    --node-count 2
```

### Workload Identity — Danh Tính Khối Lượng Công Việc

Thay vì hardcode credentials, dùng Managed Identity để Pod tự lấy token:

```bash
# Enable OIDC issuer và workload identity
az aks update \
    --resource-group myapp-rg \
    --name myapp-aks \
    --enable-oidc-issuer \
    --enable-workload-identity

# Tạo Managed Identity
az identity create \
    --resource-group myapp-rg \
    --name myapp-identity

# Gán role Key Vault Secret Reader
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee <identity-client-id> \
    --scope /subscriptions/.../resourceGroups/myapp-rg/providers/Microsoft.KeyVault/vaults/mykeyvault
```

---

## 4. Azure Functions — Serverless cho .NET

### Khi Nào Dùng Azure Functions?

- **Event-driven processing** — Xử lý theo sự kiện: process message queue, blob trigger
- **Scheduled jobs** — Công việc định kỳ: cleanup, reporting
- **HTTP APIs đơn giản** — ít logic, ít concurrency
- **Integration workflows** — Tích hợp hệ thống

### Isolated Worker Model (Khuyến Nghị cho .NET 8+)

```csharp
// Program.cs — Azure Functions .NET 8 Isolated
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices(services =>
    {
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(Environment.GetEnvironmentVariable("SqlConnection")));
    })
    .Build();

await host.RunAsync();
```

```csharp
// OrderProcessor.cs — HTTP Trigger
public class OrderFunctions
{
    private readonly IOrderService _orderService;

    public OrderFunctions(IOrderService orderService)
    {
        _orderService = orderService;
    }

    // HTTP Trigger — kích hoạt qua HTTP
    [Function("ProcessOrder")]
    public async Task<HttpResponseData> ProcessOrder(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = "orders")] HttpRequestData req)
    {
        var order = await req.ReadFromJsonAsync<CreateOrderRequest>();

        var result = await _orderService.CreateOrderAsync(order!);

        var response = req.CreateResponse(HttpStatusCode.Created);
        await response.WriteAsJsonAsync(result);
        return response;
    }

    // Timer Trigger — kích hoạt theo lịch (cron)
    [Function("DailyCleanup")]
    public async Task DailyCleanup(
        [TimerTrigger("0 0 2 * * *")] TimerInfo timerInfo,  // 2:00 AM mỗi ngày
        FunctionContext context)
    {
        var logger = context.GetLogger("DailyCleanup");
        logger.LogInformation("Starting daily cleanup at {Time}", DateTime.UtcNow);

        await _orderService.CleanupExpiredOrdersAsync();
    }

    // Service Bus Trigger — kích hoạt khi có message
    [Function("HandleOrderEvent")]
    public async Task HandleOrderEvent(
        [ServiceBusTrigger("orders-topic", "my-subscription", Connection = "ServiceBusConnection")]
        string messageBody,
        FunctionContext context)
    {
        var logger = context.GetLogger("HandleOrderEvent");
        var orderEvent = JsonSerializer.Deserialize<OrderCreatedEvent>(messageBody);

        await _orderService.SendConfirmationEmailAsync(orderEvent!.OrderId);
    }

    // Blob Trigger — kích hoạt khi file upload lên Azure Blob Storage
    [Function("ProcessUploadedFile")]
    public async Task ProcessUploadedFile(
        [BlobTrigger("uploads/{name}", Connection = "StorageConnection")] Stream blobStream,
        string name,
        FunctionContext context)
    {
        var logger = context.GetLogger("ProcessUploadedFile");
        logger.LogInformation("Processing blob: {Name}, Size: {Size}", name, blobStream.Length);

        await _orderService.ProcessImportFileAsync(blobStream, name);
    }
}
```

### Durable Functions — Functions Bền Vững

Durable Functions cho phép viết stateful workflows trong môi trường serverless:

```csharp
// Orchestrator — điều phối workflow
[Function("OrderWorkflow")]
public async Task<OrderResult> RunOrchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var input = context.GetInput<OrderRequest>()!;

    // Gọi activity functions tuần tự
    var validated = await context.CallActivityAsync<bool>("ValidateOrder", input);
    if (!validated)
        return new OrderResult { Status = "Invalid" };

    var paymentResult = await context.CallActivityAsync<PaymentResult>("ProcessPayment", input);
    if (!paymentResult.Success)
        return new OrderResult { Status = "PaymentFailed" };

    // Gọi song song
    var tasks = new[]
    {
        context.CallActivityAsync("SendConfirmationEmail", input.OrderId),
        context.CallActivityAsync("UpdateInventory", input.Items),
        context.CallActivityAsync("NotifyWarehouse", input.OrderId)
    };
    await Task.WhenAll(tasks);

    return new OrderResult { Status = "Completed", OrderId = input.OrderId };
}
```

---

## 5. Azure SQL — Database Managed Service

### Tạo Azure SQL Database

```bash
# Tạo SQL Server
az sql server create \
    --name myapp-sqlserver \
    --resource-group myapp-rg \
    --location eastasia \
    --admin-user sqladmin \
    --admin-password "P@ssword123!"

# Tạo database
az sql db create \
    --resource-group myapp-rg \
    --server myapp-sqlserver \
    --name myappdb \
    --service-objective S3 \
    --backup-storage-redundancy Zone

# Cho phép Azure services kết nối
az sql server firewall-rule create \
    --resource-group myapp-rg \
    --server myapp-sqlserver \
    --name AllowAzureServices \
    --start-ip-address 0.0.0.0 \
    --end-ip-address 0.0.0.0
```

### Connection String với Managed Identity (không cần password)

```csharp
// Program.cs — kết nối SQL với Managed Identity
builder.Services.AddDbContext<AppDbContext>(options =>
{
    var connectionString = builder.Configuration.GetConnectionString("AzureSQL");
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: null);
    });
});

// appsettings.Production.json
// {
//   "ConnectionStrings": {
//     "AzureSQL": "Server=myapp-sqlserver.database.windows.net;
//                  Database=myappdb;
//                  Authentication=Active Directory Managed Identity;"
//   }
// }
```

### Elastic Pool — Chia Sẻ Tài Nguyên

```bash
# Tạo Elastic Pool — nhiều databases chia sẻ tài nguyên, tối ưu chi phí
az sql elastic-pool create \
    --resource-group myapp-rg \
    --server myapp-sqlserver \
    --name myapp-pool \
    --edition Standard \
    --dtu 100 \
    --db-dtu-min 10 \
    --db-dtu-max 50

# Đưa database vào pool
az sql db update \
    --resource-group myapp-rg \
    --server myapp-sqlserver \
    --name myappdb \
    --elastic-pool myapp-pool
```

---

## 6. Azure Cache for Redis — Cache Phân Tán

```bash
# Tạo Redis Cache
az redis create \
    --resource-group myapp-rg \
    --name myapp-redis \
    --location eastasia \
    --sku Standard \
    --vm-size C1

# Lấy connection string
az redis list-keys \
    --resource-group myapp-rg \
    --name myapp-redis
```

```csharp
// Program.cs — cấu hình Redis
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
    options.InstanceName = "myapp:";
});

// Sử dụng IDistributedCache
public class ProductService
{
    private readonly IDistributedCache _cache;
    private readonly AppDbContext _db;

    public ProductService(IDistributedCache cache, AppDbContext db)
    {
        _cache = cache;
        _db = db;
    }

    public async Task<Product?> GetProductAsync(int id)
    {
        var cacheKey = $"product:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached is not null)
            return JsonSerializer.Deserialize<Product>(cached);

        var product = await _db.Products.FindAsync(id);

        if (product is not null)
        {
            await _cache.SetStringAsync(cacheKey,
                JsonSerializer.Serialize(product),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
                });
        }

        return product;
    }
}
```

---

## 7. Azure Service Bus — Nhắn Tin Bất Đồng Bộ

```bash
# Tạo Service Bus Namespace
az servicebus namespace create \
    --resource-group myapp-rg \
    --name myapp-servicebus \
    --location eastasia \
    --sku Standard

# Tạo Queue
az servicebus queue create \
    --resource-group myapp-rg \
    --namespace-name myapp-servicebus \
    --name orders-queue \
    --max-size 1024 \
    --default-message-time-to-live P14D

# Tạo Topic và Subscription
az servicebus topic create \
    --resource-group myapp-rg \
    --namespace-name myapp-servicebus \
    --name order-events

az servicebus topic subscription create \
    --resource-group myapp-rg \
    --namespace-name myapp-servicebus \
    --topic-name order-events \
    --name email-subscription
```

```csharp
// Gửi message lên Service Bus
public class OrderPublisher
{
    private readonly ServiceBusSender _sender;

    public OrderPublisher(ServiceBusClient client)
    {
        _sender = client.CreateSender("orders-queue");
    }

    public async Task PublishOrderCreatedAsync(OrderCreatedEvent orderEvent)
    {
        var message = new ServiceBusMessage(
            JsonSerializer.SerializeToUtf8Bytes(orderEvent))
        {
            MessageId = orderEvent.OrderId.ToString(),
            Subject = "OrderCreated",
            ContentType = "application/json",
            TimeToLive = TimeSpan.FromDays(7)
        };

        await _sender.SendMessageAsync(message);
    }
}
```

---

## 8. Azure Key Vault — Quản Lý Secrets

```bash
# Tạo Key Vault
az keyvault create \
    --resource-group myapp-rg \
    --name myapp-keyvault \
    --location eastasia

# Thêm secret
az keyvault secret set \
    --vault-name myapp-keyvault \
    --name "ConnectionStrings--Default" \
    --value "Server=prod-server;Database=myappdb;..."

# Cấp quyền cho App Service / AKS Managed Identity
az keyvault set-policy \
    --name myapp-keyvault \
    --object-id <managed-identity-object-id> \
    --secret-permissions get list
```

```csharp
// Program.cs — đọc secrets từ Key Vault khi khởi động
var keyVaultUri = new Uri($"https://myapp-keyvault.vault.azure.net/");
builder.Configuration.AddAzureKeyVault(
    keyVaultUri,
    new DefaultAzureCredential()); // dùng Managed Identity tự động

// Sau đó dùng bình thường qua IConfiguration
var connectionString = builder.Configuration["ConnectionStrings:Default"];
```

---

## Checklist Azure .NET

- [ ] Dùng Managed Identity thay vì connection string với username/password
- [ ] App Settings được cấu hình đúng cho từng Deployment Slot
- [ ] Auto-scale rules được thiết lập với min/max replicas hợp lý
- [ ] Azure SQL có Geo-Redundant Backup — sao lưu dự phòng địa lý
- [ ] Redis Cache có SSL enabled
- [ ] Key Vault được dùng cho mọi secrets trong production
- [ ] Diagnostic Settings được enable để gửi logs vào Log Analytics
- [ ] Application Insights được tích hợp để monitoring
- [ ] Network Security Group — NSG kiểm soát inbound/outbound traffic
- [ ] Private Endpoint cho Database và Redis (không expose ra internet)

---

## Câu Hỏi Phỏng Vấn

**Q: Khi nào dùng App Service thay vì AKS?**
> App Service: team nhỏ, ít microservices, ít DevOps expertise, cần deploy nhanh. AKS: nhiều microservices phức tạp, cần control chi tiết về scaling/networking/resource, team có K8s experience. App Service đơn giản hơn nhưng ít flexible hơn và có thể tốn kém hơn ở quy mô lớn.

**Q: Azure Functions có thể thay thế hoàn toàn Web API không?**
> Không lý tưởng. Functions phù hợp cho event-driven, short-lived, bursty workloads. Web API tốt hơn cho: long-running requests, complex stateful logic, low-latency requirements, WebSocket connections. Cold start — khởi động nguội của Functions (khi không có traffic) cũng gây latency cho request đầu tiên.

**Q: Managed Identity giải quyết vấn đề gì?**
> Managed Identity cho phép Azure service (App Service, AKS pod) tự xác thực với các Azure services khác (Key Vault, SQL, Storage) mà không cần lưu credentials — thông tin xác thực trong code hay config. Azure tự quản lý và rotate token. Loại bỏ hoàn toàn "secret zero problem" — vấn đề bí mật đầu tiên.

**Cập Nhật Lần Cuối:** 2026-06-02
