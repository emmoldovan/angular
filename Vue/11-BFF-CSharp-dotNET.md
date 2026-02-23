# BFF (Backend for Frontend) cu C#/.NET (Interview Prep - Senior Frontend Architect)

> BFF pattern, comunicarea Vue frontend cu BFF-ul, API Gateway vs BFF,
> exemple cod C#/.NET Web API, authentication flow, middleware.
> Nivel MODERAT - concepte + cod basic.

---

## Cuprins

1. [Ce este BFF Pattern și De Ce](#1-ce-este-bff-pattern-și-de-ce)
2. [BFF vs API Gateway - Diferențe](#2-bff-vs-api-gateway---diferențe)
3. [Arhitectura BFF + Vue Frontend](#3-arhitectura-bff--vue-frontend)
4. [C#/.NET Web API Basics](#4-cnet-web-api-basics)
5. [Structura Proiect .NET Minimal API](#5-structura-proiect-net-minimal-api)
6. [Comunicarea Vue Frontend cu BFF](#6-comunicarea-vue-frontend-cu-bff)
7. [Authentication Flow prin BFF](#7-authentication-flow-prin-bff)
8. [Middleware în .NET](#8-middleware-în-net)
9. [Migration Patterns BFF](#9-migration-patterns-bff)
10. [Paralela: Angular vs Vue consuming BFF](#10-paralela-angular-vs-vue-consuming-bff)
11. [Întrebări de interviu](#11-întrebări-de-interviu)

---

## 1. Ce este BFF Pattern și De Ce

**BFF (Backend for Frontend)** este un pattern arhitectural în care fiecare tip de client
(web, mobile, IoT) primește un **backend dedicat** care servește exact nevoile acelui frontend.

### Principii fundamentale

- **Un BFF per tip de client** - aplicația web Vue are propriul BFF, aplicația mobilă are alt BFF
- **BFF-ul agregă date** de la multiple microservices într-un singur response
- **BFF-ul adaptează răspunsurile** la nevoile specifice ale frontend-ului
- **Frontend-ul nu cunoaște** microserviciile din spate - vorbește doar cu BFF-ul
- Pattern-ul a fost popularizat de **Sam Newman** în contextul arhitecturilor microservices

### De ce BFF

**Problema fără BFF:**
```
Vue App ──→ Users Service      (call 1)
        ──→ Orders Service     (call 2)
        ──→ Products Service   (call 3)
        ──→ Reviews Service    (call 4)
        ──→ Inventory Service  (call 5)
```
Frontend-ul face 5 request-uri separate, gestionează 5 formate de răspuns diferite,
și trebuie să agregeze datele în client.

**Soluția cu BFF:**
```
Vue App ──→ BFF (1 singur call) ──→ Users Service
                                ──→ Orders Service
                                ──→ Products Service
                                ──→ Reviews Service
                                ──→ Inventory Service
```
BFF-ul face agregarea, frontend-ul primește un singur response optimizat.

### Beneficii cheie

| Beneficiu | Detalii |
|-----------|---------|
| **Reducere network calls** | 1 call la BFF în loc de N calls la microservices |
| **Optimizare per client** | Web primește date diferite față de mobile |
| **Security layer** | Frontend nu expune microserviciile direct |
| **Ownership clar** | Echipa frontend deține BFF-ul |
| **Simplificare frontend** | Logica de agregare stă în BFF, nu în UI |
| **Caching centralizat** | BFF poate face cache inteligent |
| **Versioning simplificat** | BFF poate adapta formatul fără a schimba microserviciile |

### Diagrama arhitectură generală

```
┌──────────────────────────────────────────────────────────────────┐
│                          CLIENTS                                  │
│                                                                    │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │   Vue Web    │    │   Mobile     │    │     IoT      │        │
│  │   App        │    │   App        │    │   Device     │        │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘        │
│         │                   │                    │                 │
│         ▼                   ▼                    ▼                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │   Web BFF    │    │  Mobile BFF  │    │   IoT BFF    │        │
│  │   (C#/.NET)  │    │  (C#/.NET)   │    │  (C#/.NET)   │        │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘        │
│         │                   │                    │                 │
│         └───────────────────┼────────────────────┘                │
│                             │                                      │
│              ┌──────────────┼──────────────┐                      │
│              ▼              ▼              ▼                       │
│       ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│       │  Users    │  │  Orders   │  │ Products  │                │
│       │  Service  │  │  Service  │  │  Service  │                │
│       └───────────┘  └───────────┘  └───────────┘               │
│                                                                    │
│                      MICROSERVICES                                 │
└──────────────────────────────────────────────────────────────────┘
```

### Când NU ai nevoie de BFF

- Aplicație simplă cu un singur backend monolitic
- Un singur tip de client (nu ai nevoie de adaptare per client)
- Microserviciile returnează deja datele în formatul necesar frontend-ului
- Echipa este prea mică pentru a menține un layer suplimentar

---

## 2. BFF vs API Gateway - Diferențe

### Tabel comparativ detaliat

| Aspect | API Gateway | BFF |
|--------|------------|-----|
| **Scop principal** | Routing, rate limiting, auth | Agregare, transformare date per client |
| **Cine îl deține** | Platform / DevOps team | Frontend team |
| **Număr instanțe** | Unul (sau puține) | Unul per tip de client |
| **Logică business** | Minimal (routing, policies) | Da (agregare, transformare, filtrare) |
| **Specificitate** | Generic pentru toți clienții | Specific per frontend |
| **Cunoaștere client** | Nu știe ce client consumă | Optimizat pentru clientul său |
| **Exemple** | Kong, Azure API Management, AWS API Gateway | C#/.NET custom API, Node.js Express |
| **Complexitate** | Configurare (YAML/JSON) | Cod custom (C#, TypeScript) |
| **Scalare** | Scalează independent | Scalează per tip de client |

### API Gateway - responsabilități

```
┌─────────────────────────────────┐
│         API GATEWAY             │
│                                 │
│  ✓ Rate limiting                │
│  ✓ Authentication (token valid?)│
│  ✓ Request routing              │
│  ✓ SSL termination              │
│  ✓ Load balancing               │
│  ✓ Request/Response logging     │
│  ✗ Business logic               │
│  ✗ Data aggregation             │
│  ✗ Data transformation          │
└─────────────────────────────────┘
```

### BFF - responsabilități

```
┌─────────────────────────────────┐
│              BFF                │
│                                 │
│  ✓ Data aggregation             │
│  ✓ Data transformation          │
│  ✓ Client-specific formatting   │
│  ✓ Caching per client needs     │
│  ✓ Error normalization          │
│  ✓ Response optimization        │
│  ✗ Rate limiting (API Gateway)  │
│  ✗ SSL termination              │
│  ✗ Load balancing               │
└─────────────────────────────────┘
```

### Când ambele coexistă

Într-o arhitectură matură, **API Gateway și BFF coexistă**:

```
┌──────────┐     ┌─────────────┐     ┌──────────┐     ┌──────────────┐
│  Vue App │ ──→ │ API Gateway │ ──→ │ Web BFF  │ ──→ │ Microservices│
└──────────┘     │             │     │  (C#)    │     └──────────────┘
                 │ - Rate limit│     │          │
                 │ - Auth check│     │ - Agreg. │
                 │ - Routing   │     │ - Cache  │
                 │ - SSL       │     │ - Transform│
                 └─────────────┘     └──────────┘
```

**Fluxul complet:**
1. **Vue App** trimite request la API Gateway
2. **API Gateway** verifică rate limit, validează token-ul, face routing
3. **Web BFF** primește request-ul, agregă date de la microservices
4. **BFF** transformă și returnează response optimizat
5. **Vue App** primește datele gata formatate

### Anti-pattern: Gateway ca BFF

Un **anti-pattern** frecvent este folosirea API Gateway-ului ca BFF - adăugarea de logică
de agregare și transformare în gateway. Aceasta duce la:
- Gateway bloated, greu de menținut
- Coupling între echipa de platformă și echipele frontend
- Un singur point of failure pentru toată logica de business

---

## 3. Arhitectura BFF + Vue Frontend

### Data Flow complet

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌─────────────────────────────────────────┐               │
│  │            VUE APP (SPA)                │               │
│  │                                         │               │
│  │  ┌──────────┐  ┌──────────────────┐    │               │
│  │  │Component │──│ Composable       │    │               │
│  │  │ (UI)     │  │ useProducts()    │    │               │
│  │  └──────────┘  └────────┬─────────┘    │               │
│  │                         │               │               │
│  │                ┌────────▼─────────┐    │               │
│  │                │ API Layer        │    │               │
│  │                │ useBffApi()      │    │               │
│  │                └────────┬─────────┘    │               │
│  └─────────────────────────┼───────────────┘               │
│                            │ HTTP (fetch/axios)             │
│                            ▼                                │
│  ┌─────────────────────────────────────────┐               │
│  │            WEB BFF (C#/.NET)            │               │
│  │                                         │               │
│  │  ┌──────────┐  ┌──────────────────┐    │               │
│  │  │Middleware │──│ Controller       │    │               │
│  │  │(Auth,CORS│  │ ProductsController│   │               │
│  │  │Logging)  │  └────────┬─────────┘    │               │
│  │  └──────────┘           │               │               │
│  │                ┌────────▼─────────┐    │               │
│  │                │ Service Layer    │    │               │
│  │                │ ProductService   │    │               │
│  │                └────────┬─────────┘    │               │
│  └─────────────────────────┼───────────────┘               │
│                            │ HTTP (HttpClient)              │
│                            ▼                                │
│  ┌─────────────────────────────────────────┐               │
│  │          MICROSERVICES                  │               │
│  │                                         │               │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐  │               │
│  │  │ Users   │ │ Orders  │ │Products │  │               │
│  │  │ Service │ │ Service │ │ Service │  │               │
│  │  └─────────┘ └─────────┘ └─────────┘  │               │
│  └─────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### Pașii unui request tipic

**Scenariul:** Vue App afișează pagina de detalii produs cu recenzii și stoc disponibil.

| Pas | Acțiune | Cine |
|-----|---------|------|
| 1 | User navighează la `/products/42` | Vue Router |
| 2 | Componenta apelează `useProduct(42)` | Vue Component |
| 3 | Composable face `GET /api/products/42` | useBffApi() |
| 4 | BFF primește request, validează JWT | Middleware Auth |
| 5 | BFF face paralel 3 request-uri | ProductService |
| 6 | → `GET /products/42` la Products Service | HttpClient |
| 7 | → `GET /reviews?productId=42` la Reviews Service | HttpClient |
| 8 | → `GET /inventory/42` la Inventory Service | HttpClient |
| 9 | BFF combină datele într-un ProductDetailDto | Controller |
| 10 | BFF returnează response optimizat | JSON Response |
| 11 | Composable parsează response și actualizează state | useProduct() |
| 12 | Vue reactivity actualizează UI-ul | Vue Reactivity |

### Exemplu concret de agregare

```csharp
// Ce returnează fiecare microservice:

// Products Service → { id: 42, name: "Laptop", price: 2999.99 }
// Reviews Service  → [{ rating: 5, text: "..." }, { rating: 4, text: "..." }]
// Inventory Service → { productId: 42, stock: 15, warehouse: "Bucharest" }

// Ce returnează BFF-ul (agregat și optimizat):
// {
//   id: 42,
//   name: "Laptop",
//   price: 2999.99,
//   averageRating: 4.5,
//   reviewCount: 2,
//   inStock: true,
//   stockLevel: "available"    // frontend nu vede numărul exact
// }
```

Observă cum BFF-ul:
- **Agregă** date din 3 surse într-un singur obiect
- **Calculează** `averageRating` (frontend nu trebuie să facă media)
- **Ascunde** detalii interne (`warehouse`, `stock` exact)
- **Transformă** `stock: 15` în `stockLevel: "available"` (logică de prezentare)

---

## 4. C#/.NET Web API Basics

### Controller-based API

Acesta este pattern-ul clasic ASP.NET Core pentru BFF.

```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Authorization;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly IOrderService _orderService;
    private readonly IReviewService _reviewService;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(
        IProductService productService,
        IOrderService orderService,
        IReviewService reviewService,
        ILogger<ProductsController> logger)
    {
        _productService = productService;
        _orderService = orderService;
        _reviewService = reviewService;
        _logger = logger;
    }

    // GET /api/products?category=electronics&page=1&pageSize=20
    [HttpGet]
    public async Task<ActionResult<PagedResult<ProductDto>>> GetProducts(
        [FromQuery] string? category = null,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20)
    {
        _logger.LogInformation(
            "Getting products: category={Category}, page={Page}",
            category, page);

        var products = await _productService
            .GetProductsAsync(category, page, pageSize);

        return Ok(products);
    }

    // GET /api/products/42
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDetailDto>> GetProduct(int id)
    {
        // BFF aggregation - call multiple services in parallel
        var productTask = _productService.GetProductAsync(id);
        var ordersTask = _orderService.GetRecentOrdersForProduct(id);
        var reviewsTask = _reviewService.GetReviewsForProduct(id);

        await Task.WhenAll(productTask, ordersTask, reviewsTask);

        var product = await productTask;
        if (product == null)
        {
            return NotFound(new { message = $"Product {id} not found" });
        }

        var orders = await ordersTask;
        var reviews = await reviewsTask;

        // Transform to frontend-specific DTO
        return Ok(new ProductDetailDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            Category = product.Category,
            ImageUrl = product.ImageUrl,
            AverageRating = reviews.Any()
                ? Math.Round(reviews.Average(r => r.Rating), 1)
                : 0,
            ReviewCount = reviews.Count,
            RecentOrderCount = orders.Count,
            IsPopular = orders.Count > 100,
            InStock = product.StockQuantity > 0,
            StockLevel = product.StockQuantity switch
            {
                0 => "out-of-stock",
                < 10 => "low-stock",
                _ => "available"
            }
        });
    }

    // POST /api/products
    [HttpPost]
    [Authorize]
    public async Task<ActionResult<ProductDto>> CreateProduct(
        [FromBody] CreateProductRequest request)
    {
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }

        var product = await _productService.CreateAsync(request);

        return CreatedAtAction(
            nameof(GetProduct),
            new { id = product.Id },
            product);
    }

    // PUT /api/products/42
    [HttpPut("{id}")]
    [Authorize(Roles = "Admin")]
    public async Task<ActionResult> UpdateProduct(
        int id, [FromBody] UpdateProductRequest request)
    {
        var updated = await _productService.UpdateAsync(id, request);
        if (!updated) return NotFound();

        return NoContent();
    }

    // DELETE /api/products/42
    [HttpDelete("{id}")]
    [Authorize(Roles = "Admin")]
    public async Task<ActionResult> DeleteProduct(int id)
    {
        var deleted = await _productService.DeleteAsync(id);
        if (!deleted) return NotFound();

        return NoContent();
    }
}
```

### Models / DTOs

**DTO (Data Transfer Object)** - obiectul pe care BFF-ul îl trimite la frontend.
BFF-ul nu expune modelele interne ale microserviciilor.

```csharp
// Models/ProductDto.cs
public record ProductDto(
    int Id,
    string Name,
    decimal Price,
    string Category,
    string ImageUrl
);

// Models/ProductDetailDto.cs - DTO specific pentru pagina de detalii
public class ProductDetailDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string Category { get; set; } = string.Empty;
    public string ImageUrl { get; set; } = string.Empty;
    public double AverageRating { get; set; }
    public int ReviewCount { get; set; }
    public int RecentOrderCount { get; set; }
    public bool IsPopular { get; set; }
    public bool InStock { get; set; }
    public string StockLevel { get; set; } = string.Empty;
}

// Models/CreateProductRequest.cs
public record CreateProductRequest(
    string Name,
    decimal Price,
    string Category
);

// Models/UpdateProductRequest.cs
public record UpdateProductRequest(
    string? Name,
    decimal? Price,
    string? Category
);

// Models/PagedResult.cs - pentru paginare
public class PagedResult<T>
{
    public IEnumerable<T> Items { get; set; } = Enumerable.Empty<T>();
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool HasNext => Page < TotalPages;
    public bool HasPrevious => Page > 1;
}
```

### Service Layer - comunicarea cu microserviciile

```csharp
// Services/IProductService.cs
public interface IProductService
{
    Task<PagedResult<ProductDto>> GetProductsAsync(
        string? category, int page, int pageSize);
    Task<ProductDto?> GetProductAsync(int id);
    Task<ProductDto> CreateAsync(CreateProductRequest request);
    Task<bool> UpdateAsync(int id, UpdateProductRequest request);
    Task<bool> DeleteAsync(int id);
}

// Services/ProductService.cs
public class ProductService : IProductService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        HttpClient httpClient,
        ILogger<ProductService> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<PagedResult<ProductDto>> GetProductsAsync(
        string? category, int page, int pageSize)
    {
        var url = $"/products?page={page}&pageSize={pageSize}";
        if (!string.IsNullOrEmpty(category))
        {
            url += $"&category={Uri.EscapeDataString(category)}";
        }

        _logger.LogDebug("Calling Products Service: {Url}", url);

        var response = await _httpClient.GetAsync(url);
        response.EnsureSuccessStatusCode();

        return await response.Content
            .ReadFromJsonAsync<PagedResult<ProductDto>>()
            ?? new PagedResult<ProductDto>();
    }

    public async Task<ProductDto?> GetProductAsync(int id)
    {
        try
        {
            return await _httpClient
                .GetFromJsonAsync<ProductDto>($"/products/{id}");
        }
        catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            return null;
        }
    }

    public async Task<ProductDto> CreateAsync(CreateProductRequest request)
    {
        var response = await _httpClient
            .PostAsJsonAsync("/products", request);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<ProductDto>()
            ?? throw new InvalidOperationException("Failed to deserialize product");
    }

    public async Task<bool> UpdateAsync(int id, UpdateProductRequest request)
    {
        var response = await _httpClient
            .PutAsJsonAsync($"/products/{id}", request);
        return response.IsSuccessStatusCode;
    }

    public async Task<bool> DeleteAsync(int id)
    {
        var response = await _httpClient.DeleteAsync($"/products/{id}");
        return response.IsSuccessStatusCode;
    }
}
```

### Dependency Injection - înregistrarea serviciilor

```csharp
// Registration în Program.cs
builder.Services.AddHttpClient<IProductService, ProductService>(client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["Services:Products"]!);
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

builder.Services.AddHttpClient<IOrderService, OrderService>(client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["Services:Orders"]!);
});

builder.Services.AddHttpClient<IReviewService, ReviewService>(client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["Services:Reviews"]!);
});
```

**Notă:** `AddHttpClient<TInterface, TImplementation>` creează un **typed HttpClient**
cu `IHttpClientFactory` în spate - gestionează lifecycle-ul conexiunilor HTTP corect.

---

## 5. Structura Proiect .NET Minimal API

### Structura de foldere

```
WebBFF/
├── Program.cs                          # Entry point + configuration
├── WebBFF.csproj                       # Project file (dependencies)
├── appsettings.json                    # Configuration (URLs servicii)
├── appsettings.Development.json        # Config pt development
│
├── Controllers/
│   ├── ProductsController.cs           # Endpoints produse
│   ├── UsersController.cs              # Endpoints utilizatori
│   ├── OrdersController.cs             # Endpoints comenzi
│   └── AuthController.cs               # Login, logout, refresh
│
├── Services/
│   ├── IProductService.cs              # Interface
│   ├── ProductService.cs               # Implementare - calls Products MS
│   ├── IOrderService.cs
│   ├── OrderService.cs
│   ├── IReviewService.cs
│   ├── ReviewService.cs
│   ├── IUserService.cs
│   └── UserService.cs
│
├── Models/
│   ├── ProductDto.cs                   # DTOs pentru produse
│   ├── UserDto.cs                      # DTOs pentru utilizatori
│   ├── OrderDto.cs                     # DTOs pentru comenzi
│   ├── PagedResult.cs                  # Generic paged result
│   └── ApiError.cs                     # Error response model
│
├── Middleware/
│   ├── ErrorHandlingMiddleware.cs      # Global error handling
│   ├── RequestLoggingMiddleware.cs     # Request/response logging
│   └── CorrelationIdMiddleware.cs      # Trace requests across services
│
└── Configuration/
    └── ServiceUrls.cs                  # Strongly-typed config
```

### Program.cs - configurare completă

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// ── Controllers ──
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// ── HttpClient-uri pentru microservices ──
builder.Services.AddHttpClient<IProductService, ProductService>(client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["Services:Products"]!);
});

builder.Services.AddHttpClient<IOrderService, OrderService>(client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["Services:Orders"]!);
});

builder.Services.AddHttpClient<IReviewService, ReviewService>(client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["Services:Reviews"]!);
});

// ── Authentication ──
builder.Services
    .AddAuthentication("Bearer")
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
        options.RequireHttpsMetadata =
            !builder.Environment.IsDevelopment();
    });

builder.Services.AddAuthorization();

// ── CORS - permite Vue dev server ──
builder.Services.AddCors(options =>
{
    options.AddPolicy("VueApp", policy =>
    {
        policy.WithOrigins(
                "http://localhost:5173",     // Vite dev server
                "https://myapp.example.com"  // Production
            )
            .AllowAnyHeader()
            .AllowAnyMethod()
            .AllowCredentials();
    });
});

// ── Caching ──
builder.Services.AddMemoryCache();
builder.Services.AddResponseCaching();

var app = builder.Build();

// ── Middleware pipeline (ordinea contează!) ──
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseMiddleware<CorrelationIdMiddleware>();
app.UseMiddleware<RequestLoggingMiddleware>();
app.UseMiddleware<ErrorHandlingMiddleware>();

app.UseCors("VueApp");
app.UseAuthentication();
app.UseAuthorization();
app.UseResponseCaching();

app.MapControllers();

app.Run();
```

### appsettings.json

```json
{
  "Services": {
    "Products": "http://products-service:8001",
    "Orders": "http://orders-service:8002",
    "Reviews": "http://reviews-service:8003",
    "Users": "http://users-service:8004"
  },
  "Auth": {
    "Authority": "https://auth.example.com",
    "Audience": "web-bff-api"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

### Minimal API alternativă (fără controllers)

```csharp
// Program.cs - Minimal API style (alternativă la controllers)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHttpClient<IProductService, ProductService>(client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["Services:Products"]!);
});

var app = builder.Build();

// Definire endpoints direct în Program.cs
app.MapGet("/api/products", async (
    IProductService service,
    string? category,
    int page = 1,
    int pageSize = 20) =>
{
    var products = await service.GetProductsAsync(category, page, pageSize);
    return Results.Ok(products);
});

app.MapGet("/api/products/{id}", async (
    int id,
    IProductService productService,
    IReviewService reviewService) =>
{
    var product = await productService.GetProductAsync(id);
    if (product is null) return Results.NotFound();

    var reviews = await reviewService.GetReviewsForProduct(id);

    return Results.Ok(new
    {
        product.Id,
        product.Name,
        product.Price,
        AverageRating = reviews.Any()
            ? reviews.Average(r => r.Rating) : 0,
        ReviewCount = reviews.Count
    });
});

app.MapPost("/api/products", async (
    CreateProductRequest request,
    IProductService service) =>
{
    var product = await service.CreateAsync(request);
    return Results.Created($"/api/products/{product.Id}", product);
}).RequireAuthorization();

app.Run();
```

**Notă:** Minimal API este mai concis pentru BFF-uri mici. Controller-based este preferat
pentru BFF-uri mari cu multe endpoints, deoarece oferă o organizare mai bună.

---

## 6. Comunicarea Vue Frontend cu BFF

### Composable `useBffApi` - layer principal de comunicare

```typescript
// composables/useBffApi.ts
import { ref } from 'vue'
import { useAuthStore } from '@/stores/auth'

const BFF_URL = import.meta.env.VITE_BFF_URL || 'http://localhost:5000'

class ApiError extends Error {
  constructor(
    public status: number,
    public statusText: string,
    public body: unknown
  ) {
    super(`API Error: ${status} ${statusText}`)
    this.name = 'ApiError'
  }
}

export function useBffApi() {
  const loading = ref(false)
  const error = ref<ApiError | null>(null)
  const authStore = useAuthStore()

  function getHeaders(): HeadersInit {
    const headers: HeadersInit = {
      'Content-Type': 'application/json',
    }
    if (authStore.token) {
      headers['Authorization'] = `Bearer ${authStore.token}`
    }
    return headers
  }

  async function request<T>(
    method: string,
    endpoint: string,
    data?: unknown
  ): Promise<T> {
    loading.value = true
    error.value = null

    try {
      const response = await fetch(`${BFF_URL}/api${endpoint}`, {
        method,
        headers: getHeaders(),
        body: data ? JSON.stringify(data) : undefined,
        credentials: 'include'  // trimite cookies (pt HttpOnly auth)
      })

      if (response.status === 401) {
        authStore.logout()
        throw new ApiError(401, 'Unauthorized', null)
      }

      if (!response.ok) {
        const body = await response.json().catch(() => null)
        throw new ApiError(response.status, response.statusText, body)
      }

      // 204 No Content
      if (response.status === 204) return undefined as T

      return await response.json() as T
    } catch (err) {
      if (err instanceof ApiError) {
        error.value = err
      }
      throw err
    } finally {
      loading.value = false
    }
  }

  return {
    loading,
    error,
    get: <T>(endpoint: string) => request<T>('GET', endpoint),
    post: <T>(endpoint: string, data: unknown) =>
      request<T>('POST', endpoint, data),
    put: <T>(endpoint: string, data: unknown) =>
      request<T>('PUT', endpoint, data),
    del: <T>(endpoint: string) => request<T>('DELETE', endpoint),
  }
}
```

### Composable specific: `useProducts`

```typescript
// composables/useProducts.ts
import { ref, computed } from 'vue'
import { useBffApi } from './useBffApi'

interface Product {
  id: number
  name: string
  price: number
  category: string
  imageUrl: string
}

interface ProductDetail extends Product {
  averageRating: number
  reviewCount: number
  isPopular: boolean
  inStock: boolean
  stockLevel: string
}

interface PagedResult<T> {
  items: T[]
  totalCount: number
  page: number
  pageSize: number
  totalPages: number
  hasNext: boolean
  hasPrevious: boolean
}

export function useProducts() {
  const { get, post, loading, error } = useBffApi()
  const products = ref<Product[]>([])
  const totalCount = ref(0)
  const currentPage = ref(1)

  async function fetchProducts(
    category?: string,
    page = 1,
    pageSize = 20
  ) {
    let endpoint = `/products?page=${page}&pageSize=${pageSize}`
    if (category) endpoint += `&category=${category}`

    const result = await get<PagedResult<Product>>(endpoint)
    products.value = result.items
    totalCount.value = result.totalCount
    currentPage.value = result.page
  }

  async function fetchProduct(id: number): Promise<ProductDetail> {
    return await get<ProductDetail>(`/products/${id}`)
  }

  async function createProduct(data: {
    name: string
    price: number
    category: string
  }): Promise<Product> {
    return await post<Product>('/products', data)
  }

  return {
    products,
    totalCount,
    currentPage,
    loading,
    error,
    fetchProducts,
    fetchProduct,
    createProduct,
  }
}
```

### Utilizare în componentă Vue

```vue
<!-- views/ProductListView.vue -->
<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { useProducts } from '@/composables/useProducts'

const { products, loading, error, fetchProducts } = useProducts()
const selectedCategory = ref<string>('')

onMounted(() => {
  fetchProducts()
})

function onCategoryChange(category: string) {
  selectedCategory.value = category
  fetchProducts(category)
}
</script>

<template>
  <div class="product-list">
    <div v-if="loading" class="loading">Se încarcă...</div>

    <div v-else-if="error" class="error">
      Eroare: {{ error.message }}
      <button @click="fetchProducts()">Reîncearcă</button>
    </div>

    <template v-else>
      <select @change="onCategoryChange($event.target.value)">
        <option value="">Toate categoriile</option>
        <option value="electronics">Electronice</option>
        <option value="books">Cărți</option>
      </select>

      <div
        v-for="product in products"
        :key="product.id"
        class="product-card"
      >
        <h3>{{ product.name }}</h3>
        <p>{{ product.price }} RON</p>
        <RouterLink :to="`/products/${product.id}`">
          Detalii
        </RouterLink>
      </div>
    </template>
  </div>
</template>
```

### Alternativă cu Axios și interceptors

```typescript
// api/bffClient.ts
import axios, { type AxiosInstance, type InternalAxiosRequestConfig } from 'axios'
import { useAuthStore } from '@/stores/auth'
import router from '@/router'

const bffClient: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_BFF_URL + '/api',
  timeout: 30000,
  withCredentials: true,
  headers: {
    'Content-Type': 'application/json'
  }
})

// Request interceptor - adaugă token
bffClient.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const authStore = useAuthStore()
    if (authStore.token) {
      config.headers.Authorization = `Bearer ${authStore.token}`
    }
    return config
  },
  (error) => Promise.reject(error)
)

// Response interceptor - handle 401
bffClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      const authStore = useAuthStore()
      authStore.logout()
      router.push('/login')
    }
    return Promise.reject(error)
  }
)

export default bffClient
```

```typescript
// Utilizare cu axios
import bffClient from '@/api/bffClient'

export function useProducts() {
  async function fetchProducts(category?: string) {
    const { data } = await bffClient.get('/products', {
      params: { category }
    })
    return data
  }

  return { fetchProducts }
}
```

---

## 7. Authentication Flow prin BFF

### De ce BFF pentru authentication?

Cel mai important beneficiu al BFF-ului în contextul auth: **JWT token-ul nu ajunge
niciodată în browser-ul utilizatorului**. BFF-ul păstrează token-ul server-side și
setează un **HttpOnly cookie** pe care JavaScript-ul nu îl poate citi.

### JWT Flow complet prin BFF

```
┌──────────┐         ┌──────────┐         ┌──────────────┐
│ Vue App  │         │   BFF    │         │ Auth Service │
│          │         │  (C#)    │         │ (Identity)   │
└────┬─────┘         └────┬─────┘         └──────┬───────┘
     │                     │                      │
     │ 1. POST /api/auth/login                    │
     │    { email, password }                     │
     │ ──────────────────→ │                      │
     │                     │                      │
     │                     │ 2. POST /auth/token  │
     │                     │    { email, password }│
     │                     │ ────────────────────→ │
     │                     │                      │
     │                     │ 3. JWT token          │
     │                     │ ←──────────────────── │
     │                     │                      │
     │ 4. Set-Cookie:      │                      │
     │    auth-token=JWT   │                      │
     │    (HttpOnly,Secure)│                      │
     │    + { user info }  │                      │
     │ ←────────────────── │                      │
     │                     │                      │
     │ 5. GET /api/products│                      │
     │    Cookie: auth-token=JWT                  │
     │ ──────────────────→ │                      │
     │                     │                      │
     │                     │ 6. Validate JWT      │
     │                     │    Forward request    │
     │                     │ ────→ Products Service│
     │                     │                      │
```

### AuthController în BFF

```csharp
// Controllers/AuthController.cs
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;
    private readonly IConfiguration _config;

    public AuthController(
        IAuthService authService,
        IConfiguration config)
    {
        _authService = authService;
        _config = config;
    }

    // POST /api/auth/login
    [HttpPost("login")]
    public async Task<ActionResult<UserInfo>> Login(
        [FromBody] LoginRequest request)
    {
        var authResult = await _authService.AuthenticateAsync(
            request.Email, request.Password);

        if (!authResult.Success)
        {
            return Unauthorized(new { message = "Invalid credentials" });
        }

        // Setează JWT ca HttpOnly cookie
        Response.Cookies.Append("auth-token", authResult.JwtToken,
            new CookieOptions
            {
                HttpOnly = true,     // NU e accesibil din JavaScript
                Secure = true,       // Doar HTTPS
                SameSite = SameSiteMode.Strict,  // Protecție CSRF
                Expires = DateTimeOffset.UtcNow.AddHours(1),
                Path = "/"
            });

        // Returnează doar info user, NU token-ul
        return Ok(new UserInfo
        {
            Id = authResult.UserId,
            Email = authResult.Email,
            Name = authResult.Name,
            Roles = authResult.Roles
        });
    }

    // POST /api/auth/logout
    [HttpPost("logout")]
    public ActionResult Logout()
    {
        Response.Cookies.Delete("auth-token", new CookieOptions
        {
            HttpOnly = true,
            Secure = true,
            SameSite = SameSiteMode.Strict,
            Path = "/"
        });

        return NoContent();
    }

    // GET /api/auth/me - verifică dacă user-ul e autentificat
    [HttpGet("me")]
    [Authorize]
    public ActionResult<UserInfo> GetCurrentUser()
    {
        var userId = User.FindFirst("sub")?.Value;
        var email = User.FindFirst("email")?.Value;
        var name = User.FindFirst("name")?.Value;

        return Ok(new UserInfo
        {
            Id = int.Parse(userId ?? "0"),
            Email = email ?? "",
            Name = name ?? "",
            Roles = User.FindAll("role").Select(c => c.Value).ToList()
        });
    }

    // POST /api/auth/refresh
    [HttpPost("refresh")]
    public async Task<ActionResult> RefreshToken()
    {
        var currentToken = Request.Cookies["auth-token"];
        if (string.IsNullOrEmpty(currentToken))
        {
            return Unauthorized();
        }

        var newToken = await _authService.RefreshTokenAsync(currentToken);
        if (newToken == null)
        {
            return Unauthorized();
        }

        Response.Cookies.Append("auth-token", newToken, new CookieOptions
        {
            HttpOnly = true,
            Secure = true,
            SameSite = SameSiteMode.Strict,
            Expires = DateTimeOffset.UtcNow.AddHours(1),
            Path = "/"
        });

        return NoContent();
    }
}
```

### HttpOnly Cookie vs JWT în localStorage

| Aspect | HttpOnly Cookie (BFF) | JWT în localStorage |
|--------|----------------------|---------------------|
| **Acces JavaScript** | NU (HttpOnly flag) | DA - orice script |
| **Vulnerabil XSS** | NU | DA - XSS poate fura token-ul |
| **Vulnerabil CSRF** | Parțial (SameSite ajută) | NU |
| **Transmitere** | Automatică (browser) | Manuală (Authorization header) |
| **Token vizibil** | NU (doar pe server) | DA (în DevTools) |
| **Refresh** | BFF gestionează | Frontend gestionează |
| **Logout** | Server șterge cookie | Frontend șterge din storage |

**Concluzie:** HttpOnly cookies prin BFF sunt **semnificativ mai sigure** pentru aplicații
web. JWT în localStorage este acceptabil doar pentru aplicații interne sau prototipuri.

### Vue Auth Composable

```typescript
// composables/useAuth.ts
import { ref, computed } from 'vue'
import { useBffApi } from './useBffApi'
import { useRouter } from 'vue-router'

interface UserInfo {
  id: number
  email: string
  name: string
  roles: string[]
}

const currentUser = ref<UserInfo | null>(null)

export function useAuth() {
  const { post, get } = useBffApi()
  const router = useRouter()

  const isAuthenticated = computed(() => currentUser.value !== null)
  const isAdmin = computed(() =>
    currentUser.value?.roles.includes('Admin') ?? false)

  async function login(email: string, password: string) {
    // BFF setează HttpOnly cookie automat
    currentUser.value = await post<UserInfo>(
      '/auth/login', { email, password })
    router.push('/')
  }

  async function logout() {
    await post('/auth/logout', {})
    currentUser.value = null
    router.push('/login')
  }

  async function checkAuth() {
    try {
      // Cookie-ul se trimite automat cu credentials: 'include'
      currentUser.value = await get<UserInfo>('/auth/me')
    } catch {
      currentUser.value = null
    }
  }

  return {
    currentUser,
    isAuthenticated,
    isAdmin,
    login,
    logout,
    checkAuth,
  }
}
```

---

## 8. Middleware în .NET

Middleware-ul în ASP.NET Core formează un **pipeline** - fiecare request trece prin
toate middleware-urile în ordine, iar response-ul se întoarce în ordine inversă.

### Pipeline-ul middleware

```
Request →  CORS → Logging → ErrorHandling → Auth → Controller
                                                        │
Response ← CORS ← Logging ← ErrorHandling ← Auth ← ───┘
```

### Error Handling Middleware

```csharp
// Middleware/ErrorHandlingMiddleware.cs
public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ErrorHandlingMiddleware> _logger;

    public ErrorHandlingMiddleware(
        RequestDelegate next,
        ILogger<ErrorHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (HttpRequestException ex)
            when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            _logger.LogWarning(
                "Upstream service returned 404: {Message}",
                ex.Message);
            context.Response.StatusCode = 404;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Resource not found",
                timestamp = DateTime.UtcNow
            });
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex,
                "Upstream service error: {StatusCode}",
                ex.StatusCode);
            context.Response.StatusCode = 502;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Service temporarily unavailable",
                timestamp = DateTime.UtcNow
            });
        }
        catch (TaskCanceledException)
        {
            _logger.LogWarning("Request timed out");
            context.Response.StatusCode = 504;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Request timed out",
                timestamp = DateTime.UtcNow
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception occurred");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Internal server error",
                timestamp = DateTime.UtcNow
            });
        }
    }
}
```

### Request Logging Middleware

```csharp
// Middleware/RequestLoggingMiddleware.cs
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var startTime = DateTime.UtcNow;
        var requestPath = context.Request.Path;
        var method = context.Request.Method;

        _logger.LogInformation(
            "→ {Method} {Path} started", method, requestPath);

        await _next(context);

        var duration = DateTime.UtcNow - startTime;
        var statusCode = context.Response.StatusCode;

        _logger.LogInformation(
            "← {Method} {Path} → {StatusCode} ({Duration}ms)",
            method, requestPath, statusCode,
            duration.TotalMilliseconds);
    }
}
```

### Correlation ID Middleware

Generează un ID unic per request care se propagă la toate microserviciile -
util pentru a urmări un request complet prin tot sistemul (distributed tracing).

```csharp
// Middleware/CorrelationIdMiddleware.cs
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    private const string CorrelationIdHeader = "X-Correlation-Id";

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Preia correlation ID din header sau generează unul nou
        var correlationId = context.Request.Headers[CorrelationIdHeader]
            .FirstOrDefault() ?? Guid.NewGuid().ToString();

        // Adaugă în context pentru a fi disponibil în servicii
        context.Items["CorrelationId"] = correlationId;

        // Adaugă în response header
        context.Response.OnStarting(() =>
        {
            context.Response.Headers[CorrelationIdHeader] = correlationId;
            return Task.CompletedTask;
        });

        await _next(context);
    }
}
```

### Înregistrarea middleware-urilor

**Ordinea contează!** Middleware-urile se execută în ordinea în care sunt adăugate.

```csharp
// Program.cs - ordinea recomandată
var app = builder.Build();

// 1. Correlation ID - primul, pentru a fi disponibil în toate celelalte
app.UseMiddleware<CorrelationIdMiddleware>();

// 2. Logging - al doilea, pentru a loga toate request-urile
app.UseMiddleware<RequestLoggingMiddleware>();

// 3. Error handling - al treilea, prinde excepțiile din tot ce urmează
app.UseMiddleware<ErrorHandlingMiddleware>();

// 4. CORS - înainte de auth
app.UseCors("VueApp");

// 5. Auth
app.UseAuthentication();
app.UseAuthorization();

// 6. Caching
app.UseResponseCaching();

// 7. Controllers (endpoints)
app.MapControllers();

app.Run();
```

---

## 9. Migration Patterns BFF

### Strangler Fig Pattern

Migrarea graduală de la un **monolith API** la **BFF + microservices**.
Se migrează o rută la un moment dat, fără downtime.

```
FAZA 1 - Monolith (înainte)
┌──────────┐     ┌──────────────────┐
│ Vue App  │ ──→ │  Monolith API    │
│          │     │  /api/products   │
│          │     │  /api/orders     │
│          │     │  /api/users      │
└──────────┘     └──────────────────┘

FAZA 2 - BFF proxy (primul pas)
┌──────────┐     ┌──────────┐     ┌──────────────────┐
│ Vue App  │ ──→ │   BFF    │ ──→ │  Monolith API    │
│          │     │ (proxy)  │     │  /api/products   │
│          │     │          │     │  /api/orders     │
└──────────┘     └──────────┘     └──────────────────┘

FAZA 3 - Migrare parțială (se mută produsele)
┌──────────┐     ┌──────────┐     ┌──────────────────┐
│ Vue App  │ ──→ │   BFF    │ ──→ │ Products Service │ (NOU)
│          │     │          │ ──→ │  Monolith API    │ (orders, users)
└──────────┘     └──────────┘     └──────────────────┘

FAZA 4 - Migrare completă
┌──────────┐     ┌──────────┐     ┌──────────────────┐
│ Vue App  │ ──→ │   BFF    │ ──→ │ Products Service │
│          │     │          │ ──→ │ Orders Service   │
│          │     │          │ ──→ │ Users Service    │
└──────────┘     └──────────┘     └──────────────────┘
```

### Implementare proxy pentru Strangler Fig

```csharp
// Services/MonolithProxyService.cs
// Proxy requests to the old monolith for routes not yet migrated
public class MonolithProxyService
{
    private readonly HttpClient _httpClient;

    public MonolithProxyService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<HttpResponseMessage> ForwardAsync(
        HttpRequest originalRequest)
    {
        var targetUrl = originalRequest.Path + originalRequest.QueryString;

        var proxyRequest = new HttpRequestMessage(
            new HttpMethod(originalRequest.Method), targetUrl);

        // Copiază headers relevante
        foreach (var header in originalRequest.Headers
            .Where(h => !h.Key.StartsWith("Host")))
        {
            proxyRequest.Headers.TryAddWithoutValidation(
                header.Key, header.Value.ToArray());
        }

        // Copiază body dacă există
        if (originalRequest.ContentLength > 0)
        {
            proxyRequest.Content = new StreamContent(
                originalRequest.Body);
            proxyRequest.Content.Headers.ContentType =
                new System.Net.Http.Headers.MediaTypeHeaderValue(
                    originalRequest.ContentType ?? "application/json");
        }

        return await _httpClient.SendAsync(proxyRequest);
    }
}
```

### Feature Flag Driven Migration

Folosește feature flags pentru a comuta între monolith și microservice
fără deploy nou.

```csharp
// Controllers/ProductsController.cs
[HttpGet]
public async Task<ActionResult> GetProducts(
    [FromQuery] string? category)
{
    // Feature flag controlează sursa datelor
    if (_featureFlags.IsEnabled("UseProductsMicroservice"))
    {
        // Noul microservice
        var products = await _productService.GetProductsAsync(category);
        return Ok(products);
    }
    else
    {
        // Vechiul monolith (proxy)
        var response = await _monolithProxy.ForwardAsync(Request);
        var content = await response.Content.ReadAsStringAsync();
        return Content(content, "application/json");
    }
}
```

### Canary Migration

Trimite un procent mic de trafic la noul microservice și compară rezultatele.

```csharp
// Services/CanaryProductService.cs
public class CanaryProductService : IProductService
{
    private readonly ProductService _newService;
    private readonly MonolithProxyService _oldService;
    private readonly ILogger<CanaryProductService> _logger;
    private readonly Random _random = new();

    public async Task<IEnumerable<ProductDto>> GetProductsAsync(
        string? category)
    {
        // 10% trafic la noul microservice
        if (_random.Next(100) < 10)
        {
            _logger.LogInformation(
                "Canary: routing to new Products microservice");
            return await _newService.GetProductsAsync(category);
        }

        _logger.LogInformation(
            "Canary: routing to monolith");
        return await _oldService.GetProductsAsync(category);
    }
}
```

### Checklist migrare monolith la BFF

| Pas | Acțiune | Status |
|-----|---------|--------|
| 1 | Introduce BFF ca proxy transparent | Faza 1 |
| 2 | Verifică că toate request-urile trec prin BFF | Faza 1 |
| 3 | Identifică primul domeniu de extras (ex: Products) | Faza 2 |
| 4 | Creează microservice-ul Products | Faza 2 |
| 5 | Implementează feature flag în BFF | Faza 2 |
| 6 | Canary testing (10% trafic) | Faza 3 |
| 7 | Rollout complet pentru Products | Faza 3 |
| 8 | Repetă pentru Orders, Users, etc. | Faza 4 |
| 9 | Decomisionează monolith-ul | Final |

---

## 10. Paralela: Angular vs Vue consuming BFF

### Tabel comparativ detaliat

| Aspect | Angular | Vue |
|--------|---------|-----|
| **HTTP client** | `HttpClient` (built-in, `@angular/common/http`) | `fetch` (native) sau `axios` (library) |
| **Interceptors** | `HttpInterceptor` (class-based sau functional) | Axios interceptors sau composable wrapper |
| **Dependency Injection** | Fully built-in (providers, inject) | `provide/inject` (mai simplu) |
| **Error handling** | `catchError` operator (RxJS) | `try/catch` (async/await) |
| **Auth header** | Interceptor adaugă automat | Composable / axios interceptor |
| **Response typing** | `HttpClient.get<T>()` - typed | Manual casting `as T` |
| **Observables vs Promises** | `Observable<T>` (RxJS) | `Promise<T>` (async/await) |
| **Caching** | Custom interceptor + `shareReplay()` | Custom composable + `ref` |
| **Retry logic** | `retry()`, `retryWhen()` operators | Manual sau `axios-retry` |
| **Cancel requests** | `takeUntilDestroyed()` | `AbortController` |

### Angular - HTTP call la BFF

```typescript
// Angular - services/product.service.ts
@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);
  private apiUrl = environment.bffUrl;

  getProducts(category?: string): Observable<PagedResult<Product>> {
    let params = new HttpParams();
    if (category) params = params.set('category', category);

    return this.http.get<PagedResult<Product>>(
      `${this.apiUrl}/api/products`, { params }
    ).pipe(
      retry(2),
      catchError(this.handleError)
    );
  }

  getProduct(id: number): Observable<ProductDetail> {
    return this.http.get<ProductDetail>(
      `${this.apiUrl}/api/products/${id}`
    );
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    console.error('API error:', error);
    return throwError(() => error);
  }
}
```

### Vue - echivalentul

```typescript
// Vue - composables/useProducts.ts
export function useProducts() {
  const { get } = useBffApi()
  const products = ref<Product[]>([])
  const loading = ref(false)

  async function fetchProducts(category?: string) {
    loading.value = true
    try {
      let endpoint = '/products'
      if (category) endpoint += `?category=${category}`

      const result = await get<PagedResult<Product>>(endpoint)
      products.value = result.items
    } catch (error) {
      console.error('API error:', error)
      throw error
    } finally {
      loading.value = false
    }
  }

  return { products, loading, fetchProducts }
}
```

### Angular Interceptor vs Vue Composable

```typescript
// Angular - auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  if (token) {
    req = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` }
    });
  }

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        authService.logout();
      }
      return throwError(() => error);
    })
  );
};

// Înregistrare
provideHttpClient(withInterceptors([authInterceptor]))
```

```typescript
// Vue - echivalentul (în composable useBffApi)
// Auth header se adaugă în getHeaders()
// 401 handling se face în request()
// (vezi secțiunea 6 pentru codul complet)
```

### Observația cheie pentru interviu

**Angular** are `HttpClient` integrat cu un sistem robust de interceptors și
pattern-ul Observable/RxJS. Acest lucru face comunicarea cu BFF-ul mai standardizată
dar și mai verbose.

**Vue** nu are un HTTP client integrat. Folosești `fetch` nativ sau `axios`.
Avantajul este flexibilitatea - dezavantajul este că fiecare proiect poate avea
o abordare diferită. Composable-urile permit encapsularea logicii HTTP similar
cu serviciile Angular, dar fără overhead-ul DI.

**Pentru BFF**, diferența practică este minimă - ambele framework-uri trimit
request-uri HTTP la aceleași endpoint-uri. Pattern-urile de comunicare sunt
aproape identice, doar sintaxa diferă.

---

## 11. Întrebări de interviu

### 1. Ce este BFF Pattern și de ce l-ai folosi?

**BFF (Backend for Frontend)** este un pattern în care fiecare tip de client primește
un backend dedicat. Îl folosesc pentru a **decupla frontend-ul de microservices** -
frontend-ul face un singur request la BFF, care agregă date de la mai multe servicii.
Beneficiile principale sunt: reducerea numărului de network calls din browser,
optimizarea payload-ului per tip de client (web primește date diferite față de mobile),
și un layer suplimentar de securitate (frontend-ul nu cunoaște microserviciile direct).
BFF-ul este deținut de echipa frontend, ceea ce reduce dependențele cross-team.
Aleg acest pattern când am cel puțin 2-3 microservices și nevoi diferite per tip de client.

### 2. BFF vs API Gateway - care sunt diferențele?

**API Gateway** se ocupă de **cross-cutting concerns**: rate limiting, authentication
token validation, SSL termination, routing și load balancing. Este deținut de echipa
de platformă și este generic - servește toți clienții la fel.
**BFF** se ocupă de **logica specifică per client**: agregare date, transformare,
caching inteligent, formatare response. Este deținut de echipa frontend și este
specific per tip de client. Într-o arhitectură matură, ambele coexistă:
`Vue App → API Gateway (rate limit, auth) → Web BFF (agregare) → Microservices`.
Anti-pattern-ul este să pui logică de agregare în API Gateway - devine monolitic și
greu de menținut, și creează coupling între echipa de platformă și echipele frontend.

### 3. Cum securizezi comunicarea Vue cu BFF?

Pe mai multe nivele. **Transport**: HTTPS obligatoriu, certificate TLS valide.
**Authentication**: BFF-ul folosește **HttpOnly cookies** (nu JWT în localStorage) -
cookie-ul nu e accesibil din JavaScript, eliminând riscul XSS.
**CORS**: configurat strict - doar originile permise (ex: `http://localhost:5173`
în development, domeniul de producție). **SameSite cookies**: `Strict` pentru
protecție CSRF. **Input validation**: BFF-ul validează toate input-urile înainte
de a le forwarda la microservices. **Rate limiting**: la nivel de API Gateway sau BFF.
**Content Security Policy**: headers CSP setate de BFF.

### 4. De ce HttpOnly cookies sunt mai sigure decât JWT în localStorage?

**localStorage** este accesibil din orice JavaScript care rulează pe pagină - inclusiv
scripturi injectate prin **XSS (Cross-Site Scripting)**. Un atacator care reușește
XSS poate citi token-ul și îl poate trimite la serverul său.
**HttpOnly cookies** au flag-ul `HttpOnly` care interzice accesul din JavaScript
(`document.cookie` nu le vede). Chiar dacă un atacator reușește XSS, nu poate
exfiltra token-ul. Cookie-urile sunt trimise automat de browser la fiecare request,
deci frontend-ul nici nu trebuie să gestioneze token-ul manual.
Riscul cu cookies este **CSRF**, dar se mitigă cu `SameSite=Strict` și anti-CSRF
tokens. Pe balanță, HttpOnly cookies prin BFF sunt semnificativ mai sigure.

### 5. Cum agregă BFF-ul date de la multiple microservices?

Folosind **apeluri paralele** cu `Task.WhenAll` în C#. Când un endpoint BFF are
nevoie de date de la 3 servicii (Products, Reviews, Inventory), lansează toate
3 request-urile în paralel și așteaptă finalizarea tuturor. Apoi combină rezultatele
într-un singur DTO optimizat pentru frontend. Exemplu:
`var productTask = _productService.GetAsync(id);`
`var reviewsTask = _reviewService.GetForProductAsync(id);`
`await Task.WhenAll(productTask, reviewsTask);`
Dacă un serviciu eșuează, BFF-ul poate returna date parțiale (graceful degradation)
în loc să eșueze complet - de exemplu, returnează produsul fără recenzii.

### 6. Cum gestionezi erorile în BFF?

Pe trei nivele. **Middleware global** (`ErrorHandlingMiddleware`) prinde toate excepțiile
negestionate și returnează un response JSON consistent (nu stack trace în producție).
Diferențiez între: `HttpRequestException` (microservice indisponibil - returnez 502),
`TaskCanceledException` (timeout - returnez 504), și excepții generice (500).
**Service layer** face retry cu exponential backoff pentru erori tranzitorii.
**Controller level** gestionează business logic errors (NotFound, Validation).
Important: BFF-ul **nu propagă erori interne** ale microserviciilor direct la frontend.
Normalizează erorile într-un format consistent pe care Vue app-ul îl poate gestiona.

### 7. Cum faci migration de la monolith API la BFF?

Folosind **Strangler Fig pattern**. Pas 1: introduc BFF-ul ca proxy transparent -
toate request-urile trec prin BFF dar sunt forwardate la monolith fără modificări.
Pas 2: identific primul domeniu de extras (cel mai independent, ex: Products) și
creez microservice-ul dedicat. Pas 3: în BFF, folosesc **feature flags** pentru a
comuta între monolith și noul microservice. Pas 4: activez feature flag-ul pentru
un procent mic de trafic (**canary**), monitorizez, apoi cresc gradual. Pas 5: repet
pentru fiecare domeniu. Frontend-ul nu știe nimic despre migrare - BFF-ul face
switch-ul transparent. Monolith-ul se decomisionează când nu mai are responsabilități.

### 8. Cum faci caching în BFF?

Mai multe strategii. **In-memory cache** (`IMemoryCache` în .NET) pentru date care se
schimbă rar (categorii, configurări) - TTL scurt (5-15 minute).
**Response caching** cu headers `Cache-Control` pentru responses identice la același
request - BFF-ul setează `[ResponseCache]` pe endpoints.
**Distributed cache** (Redis) când BFF-ul rulează pe multiple instanțe - important
pentru a evita cache inconsistency.
**Cache invalidation**: cel mai greu. Folosesc TTL-based (simplest), event-based
(microservice-ul publică un event când datele se schimbă), sau cache-aside pattern.
Frontend-ul Vue poate face și caching local cu composable-uri, dar cache-ul principal
stă în BFF pentru a reduce load-ul pe microservices.

### 9. CORS configuration - de ce și cum?

**CORS (Cross-Origin Resource Sharing)** este necesar deoarece Vue dev server
(ex: `localhost:5173`) și BFF (ex: `localhost:5000`) sunt pe **origini diferite**.
Browser-ul blochează request-urile cross-origin implicit.
Configurarea în .NET: `AddCors` cu `WithOrigins` (doar originile permise, nu `*`),
`AllowAnyHeader`, `AllowAnyMethod`, `AllowCredentials` (necesar pentru cookies).
**Important:** `AllowCredentials` și `AllowAnyOrigin("*")` nu pot fi folosite
împreună - trebuie specificate originile explicit.
În producție, CORS poate fi gestionat la nivel de API Gateway sau reverse proxy
(nginx), nu neapărat în BFF. Dar în development, configurarea în BFF e cea mai simplă.

### 10. Cum testezi BFF-ul?

**Unit tests** pentru logica de agregare și transformare din controllere și servicii -
mock-uiesc `HttpClient` cu `MockHttpMessageHandler` pentru a simula răspunsurile
microserviciilor. **Integration tests** cu `WebApplicationFactory` din ASP.NET Core -
pornesc BFF-ul în-memory și fac request-uri reale, verificând response-urile.
Mock-uiesc microserviciile cu `WireMock.Net` pentru a simula diverse scenarii
(succes, timeout, erori 500). **Contract tests** (Pact) pentru a verifica că
contractul între BFF și microservices nu se rupe. **E2E tests** cu Playwright sau
Cypress care testează fluxul complet Vue → BFF → mocked microservices.
BFF-ul se testează independent de frontend - asta e un mare avantaj al separării.

### 11. Cum gestionezi versionarea API-ului BFF?

BFF-ul este deținut de echipa frontend, deci **versionarea este mai simplă** decât
la un API public. Folosesc URL-based versioning (`/api/v1/products`, `/api/v2/products`)
sau header-based (`Api-Version: 2`). Deoarece echipa frontend controlează atât Vue
app-ul cât și BFF-ul, pot face deploy-uri coordonate. Dacă schimb contractul BFF,
actualizez și frontend-ul în același sprint. Nu am nevoie de backward compatibility
pe termen lung ca la un API public. Totuși, dacă există mai multe echipe frontend,
versionarea devine importantă - în acest caz, fiecare echipă are propriul BFF.

### 12. Ce se întâmplă dacă un microservice este indisponibil?

BFF-ul implementează **resilience patterns**: **Circuit breaker** (Polly în .NET) -
dacă un serviciu eșuează de N ori consecutiv, BFF-ul oprește request-urile către
el pentru un interval (ex: 30 secunde) și returnează date din cache sau un fallback.
**Timeout** configurat per serviciu (ex: 5 secunde). **Retry** cu exponential backoff
pentru erori tranzitorii (503, network errors). **Graceful degradation** - dacă
Reviews Service e indisponibil, BFF-ul returnează produsul fără recenzii, cu un
flag `reviewsAvailable: false`. Frontend-ul Vue verifică acest flag și afișează
un mesaj corespunzător în loc să crape complet.


---

**Următor :** [**13 - Pitch Personal Vue** →](Vue/13-Pitch-Personal-Vue.md)