# Government Pension Management System — Architecture Guide
> Designed by: Senior Software Architect Perspective (Google / Amazon Standard)
> Language: C# (.NET 8) | Platform: ASP.NET Core MVC | Database: SQL Server

---

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Database Selection](#2-database-selection)
3. [Layered Architecture](#3-layered-architecture)
4. [MVC Request Flow](#4-mvc-request-flow)
5. [OOP Design](#5-oop-design)
6. [Role-Based Access Control (RBAC)](#6-role-based-access-control-rbac)
7. [SOLID Principles Applied](#7-solid-principles-applied)
8. [Repository Pattern](#8-repository-pattern)
9. [Project Folder Structure](#9-project-folder-structure)
10. [Class Diagram Summary](#10-class-diagram-summary)
11. [Architecture Decision Record (ADR)](#11-architecture-decision-record-adr)
12. [Quick Start Checklist](#12-quick-start-checklist)

---

## 1. System Overview

```
┌─────────────────────────────────────────────────────────────┐
│              Govt. BD Pension Management System             │
│                                                             │
│   Users: System Admin | Pension Admin | Manager | Viewer    │
│   Tech:  C# .NET 8 | ASP.NET Core MVC | SQL Server         │
│   Pattern: 4-Layer Clean Architecture + Repository + RBAC   │
└─────────────────────────────────────────────────────────────┘
```

**Core Modules:**
- Authentication & Authorization (ASP.NET Core Identity + RBAC)
- Employee Management
- Pension Management
- Payment Tracking
- Department Management
- Dashboard & Analytics

---

## 2. Database Selection

### Recommendation: Microsoft SQL Server (MSSQL)

| Option         | Verdict | Reason                                                            |
|----------------|---------|-------------------------------------------------------------------|
| **SQL Server** | BEST    | Native C# integration, EF Core support, enterprise-grade          |
| SQLite         | OK      | Good for small apps, but not suited for multi-user web apps       |
| MySQL          | OK      | Open source but less native to C# ecosystem                       |
| PostgreSQL     | OK      | Powerful but adds unnecessary complexity for this scope           |
| MongoDB        | NO      | NoSQL — not suitable for relational pension data                  |

### Why SQL Server?
- **Entity Framework Core (EF Core)** integrates natively with C# and MSSQL
- Built-in support for **stored procedures**, **transactions**, and **triggers**
- **ACID compliance** — critical for financial and pension data integrity
- **Windows Authentication** — secure login without extra credential management
- Free tier available: **SQL Server Express** (up to 10GB — sufficient for this system)

### Connection String (`appsettings.json`)
```json
"ConnectionStrings": {
  "PensionDB": "Server=.;Database=GovtPensionDB;Trusted_Connection=True;TrustServerCertificate=True;"
}
```

### DbContext Registration (`Program.cs`)
```csharp
builder.Services.AddDbContext<PensionDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("PensionDB")));
```

---

## 3. Layered Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  PRESENTATION LAYER                      │
│              ASP.NET Core MVC                            │
│                                                          │
│  Controllers/              Views/                        │
│  ├── AuthController        ├── Auth/Login.cshtml         │
│  ├── DashboardController   ├── Dashboard/Index.cshtml    │
│  ├── EmployeeController    ├── Employee/Index.cshtml     │
│  ├── PensionController     ├── Pension/Index.cshtml      │
│  └── PaymentController     └── Shared/_Layout.cshtml     │
│                                                          │
│  ViewModels/               wwwroot/                      │
│  ├── LoginViewModel        ├── css/                      │
│  ├── EmployeeViewModel     ├── js/                       │
│  └── PensionViewModel      └── lib/ (Bootstrap 5)        │
└──────────────────────────┬───────────────────────────────┘
                           │  (calls via interfaces)
┌──────────────────────────▼───────────────────────────────┐
│                  APPLICATION LAYER                       │
│            Business Logic & Use Cases                    │
│                                                          │
│  Services/                 Interfaces/                   │
│  ├── AuthService           ├── IAuthService              │
│  ├── EmployeeService       ├── IEmployeeService          │
│  ├── PensionService        ├── IPensionService           │
│  ├── PaymentService        └── IPaymentService           │
│  └── DashboardService                                    │
│                                                          │
│  Security/                                               │
│  └── PermissionGuard       (RBAC enforcement)            │
└──────────────────────────┬───────────────────────────────┘
                           │  (depends on interfaces only)
┌──────────────────────────▼───────────────────────────────┐
│                  DOMAIN LAYER                            │
│           Core Entities & Contracts                      │
│                                                          │
│  Entities/                 Interfaces/                   │
│  ├── Employee              ├── IRepository<T>            │
│  ├── Pension               ├── IEmployeeRepository       │
│  ├── Payment               ├── IPensionRepository        │
│  ├── Department            ├── IPaymentRepository        │
│  ├── SystemUser            └── IUnitOfWork               │
│  └── Login                                               │
│                                                          │
│  Enums/                                                  │
│  ├── UserRole                                            │
│  └── PensionStatus                                       │
└──────────────────────────┬───────────────────────────────┘
                           │  (implements domain interfaces)
┌──────────────────────────▼───────────────────────────────┐
│                  INFRASTRUCTURE LAYER                    │
│           EF Core + Repositories + Database              │
│                                                          │
│  Data/                     Repositories/                 │
│  ├── PensionDbContext       ├── Repository.cs (generic)  │
│  └── Migrations/            ├── EmployeeRepository       │
│                             ├── PensionRepository        │
│                             ├── PaymentRepository        │
│                             └── UnitOfWork               │
└──────────────────────────────────────────────────────────┘
```

---

## 4. MVC Request Flow

```
Browser (HTTP Request)
        │
        ▼
┌───────────────────┐
│   Middleware      │  Authentication, Authorization, Routing
│   Pipeline        │  app.UseAuthentication()
│                   │  app.UseAuthorization()
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   Controller      │  Receives request, calls service
│                   │  [Authorize(Roles = "PensionAdmin")]
│  EmployeeCtrl     │  public IActionResult Index() { ... }
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   Service Layer   │  Business logic, RBAC enforcement
│                   │  PermissionGuard.Enforce(role, action)
│  EmployeeService  │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   Repository      │  Data access via EF Core
│                   │  _context.Employees.ToListAsync()
│  EmployeeRepo     │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   SQL Server DB   │  GovtPensionDB
└────────┬──────────┘
         │
         ▼ (data returned back up the chain)
┌───────────────────┐
│   ViewModel       │  Map entity → ViewModel for view
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   Razor View      │  Render HTML — Employee/Index.cshtml
│   (.cshtml)       │
└────────┬──────────┘
         │
         ▼
Browser (HTTP Response)
```

---

## 5. OOP Design

### 5.1 Encapsulation
```csharp
public class Employee
{
    private string _empName;
    private DateTime _retirementDate;

    public string EmpName
    {
        get => _empName;
        set => _empName = string.IsNullOrEmpty(value)
            ? throw new ArgumentException("Name cannot be empty")
            : value;
    }

    public bool IsRetired => DateTime.Now >= _retirementDate;
}
```

### 5.2 Inheritance
```csharp
// Abstract base — shared by all user roles
public abstract class SystemUser
{
    public int UserId { get; set; }
    public string UserName { get; set; }
    public string Email { get; set; }
    public abstract string GetRole();
    public abstract bool CanDelete();
}

// Concrete role classes
public class SystemAdmin    : SystemUser { public override string GetRole() => "SystemAdmin";    public override bool CanDelete() => true;  }
public class PensionAdmin   : SystemUser { public override string GetRole() => "PensionAdmin";   public override bool CanDelete() => true;  }
public class PensionManager : SystemUser { public override string GetRole() => "PensionManager"; public override bool CanDelete() => false; }
public class Viewer         : SystemUser { public override string GetRole() => "Viewer";         public override bool CanDelete() => false; }
```

### 5.3 Polymorphism
```csharp
// Same method call — different behavior per role type
List<SystemUser> users = new() { new SystemAdmin(), new PensionManager(), new Viewer() };

foreach (var user in users)
    Console.WriteLine($"{user.GetRole()} — Can Delete: {user.CanDelete()}");
```

### 5.4 Abstraction
```csharp
// Controller only knows the interface — not the implementation
public class PensionController : Controller
{
    private readonly IPensionService _pensionService;

    public PensionController(IPensionService pensionService)
        => _pensionService = pensionService;

    public async Task<IActionResult> Index()
    {
        var pensions = await _pensionService.GetAllActivePensionsAsync();
        return View(pensions);
    }
}
```

---

## 6. Role-Based Access Control (RBAC)

### Permission Matrix

| Feature                  | System Admin | Pension Admin | Pension Manager | Viewer |
|--------------------------|:---:|:---:|:---:|:---:|
| Login                    | YES | YES | YES | YES |
| Create / Delete Users    | YES | NO  | NO  | NO  |
| Add Employee             | NO  | YES | NO  | NO  |
| Edit Employee            | NO  | YES | NO  | NO  |
| Delete Employee          | NO  | YES | NO  | NO  |
| Manage Pension Data      | NO  | YES | NO  | NO  |
| View Employee List       | YES | YES | YES | YES |
| View Employee Details    | YES | YES | YES | NO  |
| Monitor Pension Status   | YES | YES | YES | NO  |
| View Dashboard Analytics | YES | YES | YES | NO  |
| System Configuration     | YES | NO  | NO  | NO  |

### Controller-Level Authorization
```csharp
// Only Pension Admin can access these actions
[Authorize(Roles = "PensionAdmin")]
public class EmployeeController : Controller
{
    [HttpPost]
    public async Task<IActionResult> Delete(int id) { ... }
}

// Multiple roles allowed
[Authorize(Roles = "SystemAdmin,PensionAdmin,PensionManager")]
public IActionResult Index() { ... }

// All authenticated users
[Authorize]
public IActionResult Dashboard() { ... }
```

### PermissionGuard (Service-Level Enforcement)
```csharp
public enum UserRole { SystemAdmin, PensionAdmin, PensionManager, Viewer }

public static class PermissionGuard
{
    private static readonly Dictionary<UserRole, HashSet<string>> _permissions = new()
    {
        [UserRole.SystemAdmin]    = new() { "ManageUsers", "ViewDashboard", "SystemConfig", "ViewEmployee" },
        [UserRole.PensionAdmin]   = new() { "ManageEmployee", "ManagePension", "ViewDashboard", "ViewEmployee" },
        [UserRole.PensionManager] = new() { "ViewEmployee", "MonitorPension", "ViewDashboard" },
        [UserRole.Viewer]         = new() { "ViewEmployee" }
    };

    public static bool HasPermission(UserRole role, string permission)
        => _permissions.TryGetValue(role, out var perms) && perms.Contains(permission);

    public static void Enforce(UserRole role, string permission)
    {
        if (!HasPermission(role, permission))
            throw new UnauthorizedAccessException(
                $"Role '{role}' does not have permission: '{permission}'");
    }
}
```

### Razor View — Role-Conditional UI
```html
@using Microsoft.AspNetCore.Identity

@if (User.IsInRole("PensionAdmin") || User.IsInRole("SystemAdmin"))
{
    <a asp-action="Create" class="btn btn-primary">Add Employee</a>
    <a asp-action="Delete" class="btn btn-danger">Delete</a>
}

@if (User.IsInRole("Viewer"))
{
    <p class="text-muted">Read-only access. Contact admin for changes.</p>
}
```

---

## 7. SOLID Principles Applied

### S — Single Responsibility
```csharp
// WRONG — controller doing business logic
public class EmployeeController : Controller
{
    public IActionResult Create(Employee emp)
    {
        // validation, DB save, email — all here = violation
    }
}

// CORRECT — controller delegates, service handles logic
public class EmployeeController : Controller
{
    private readonly IEmployeeService _service;
    public EmployeeController(IEmployeeService service) => _service = service;

    public async Task<IActionResult> Create(EmployeeViewModel vm)
    {
        await _service.AddEmployeeAsync(vm);  // single responsibility
        return RedirectToAction("Index");
    }
}
```

### O — Open/Closed
```csharp
// Base calculator — closed for modification
public abstract class PensionCalculator
{
    public abstract decimal Calculate(Employee emp);
}

// Open for extension — add new type without touching existing code
public class StandardPensionCalculator : PensionCalculator
{
    public override decimal Calculate(Employee emp) => emp.BasicSalary * 0.80m;
}

public class SpecialPensionCalculator : PensionCalculator
{
    public override decimal Calculate(Employee emp) => emp.BasicSalary * 0.90m;
}
```

### L — Liskov Substitution
```csharp
// Any SystemUser subclass safely replaces the base
public IActionResult Welcome(SystemUser user)
{
    ViewBag.Role = user.GetRole();       // works for ALL role types
    ViewBag.CanDelete = user.CanDelete();
    return View();
}
```

### I — Interface Segregation
```csharp
// WRONG — fat interface forces all roles to implement unused methods
public interface IUserActions
{
    void AddEmployee();
    void DeleteEmployee();
    void ViewEmployee();
    void ManageUsers();
}

// CORRECT — segregated interfaces
public interface IViewable      { Task<IEnumerable<Employee>> GetAllAsync(); }
public interface IManageable    { Task AddAsync(Employee e); Task DeleteAsync(int id); }
public interface IAdministrable { Task ManageUsersAsync(); }

// Each service implements only what it needs
public class EmployeeService  : IViewable, IManageable  { }
public class DashboardService : IViewable               { }
```

### D — Dependency Inversion
```csharp
// WRONG — controller tightly coupled to concrete class
public class EmployeeController : Controller
{
    private EmployeeService _service = new EmployeeService(); // bad
}

// CORRECT — depends on interface, injected by DI container
public class EmployeeController : Controller
{
    private readonly IEmployeeService _service;
    public EmployeeController(IEmployeeService service) => _service = service;
}

// Registered in Program.cs
builder.Services.AddScoped<IEmployeeRepository, EmployeeRepository>();
builder.Services.AddScoped<IEmployeeService, EmployeeService>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

---

## 8. Repository Pattern

### Generic Interface
```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}
```

### Specific Repository Interfaces
```csharp
public interface IEmployeeRepository : IRepository<Employee>
{
    Task<IEnumerable<Employee>> GetByStatusAsync(string status);
    Task<IEnumerable<Employee>> GetByDepartmentAsync(int deptId);
    Task<Employee?> SearchByIdAsync(string empId);
}

public interface IPensionRepository : IRepository<Pension>
{
    Task<Pension?> GetByEmployeeIdAsync(int empId);
    Task<IEnumerable<Pension>> GetActivePensionsAsync();
}

public interface IPaymentRepository : IRepository<Payment>
{
    Task<IEnumerable<Payment>> GetByPensionIdAsync(int pensionId);
}
```

### Generic Implementation
```csharp
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly PensionDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(PensionDbContext context)
    {
        _context = context;
        _dbSet   = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id)      => await _dbSet.FindAsync(id);
    public async Task<IEnumerable<T>> GetAllAsync() => await _dbSet.ToListAsync();

    public async Task AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
        await _context.SaveChangesAsync();
    }

    public async Task UpdateAsync(T entity)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync();
    }

    public async Task DeleteAsync(int id)
    {
        var entity = await GetByIdAsync(id);
        if (entity is not null)
        {
            _dbSet.Remove(entity);
            await _context.SaveChangesAsync();
        }
    }
}
```

### Unit of Work
```csharp
public interface IUnitOfWork : IDisposable
{
    IEmployeeRepository Employees { get; }
    IPensionRepository  Pensions  { get; }
    IPaymentRepository  Payments  { get; }
    Task<int> CommitAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly PensionDbContext _context;

    public IEmployeeRepository Employees { get; }
    public IPensionRepository  Pensions  { get; }
    public IPaymentRepository  Payments  { get; }

    public UnitOfWork(PensionDbContext context,
        IEmployeeRepository emp,
        IPensionRepository  pen,
        IPaymentRepository  pay)
    {
        _context  = context;
        Employees = emp;
        Pensions  = pen;
        Payments  = pay;
    }

    public async Task<int> CommitAsync() => await _context.SaveChangesAsync();
    public void Dispose() => _context.Dispose();
}
```

---

## 9. Project Folder Structure

```
GovtPensionSystem/
│
├── Domain/                               # No external dependencies
│   ├── Entities/
│   │   ├── Employee.cs
│   │   ├── Pension.cs
│   │   ├── Payment.cs
│   │   ├── Department.cs
│   │   ├── SystemUser.cs                 # Abstract base user
│   │   └── Login.cs
│   ├── Enums/
│   │   ├── UserRole.cs
│   │   └── PensionStatus.cs
│   └── Interfaces/
│       ├── IRepository.cs
│       ├── IEmployeeRepository.cs
│       ├── IPensionRepository.cs
│       ├── IPaymentRepository.cs
│       └── IUnitOfWork.cs
│
├── Application/                          # Depends only on Domain
│   ├── Services/
│   │   ├── AuthService.cs
│   │   ├── EmployeeService.cs
│   │   ├── PensionService.cs
│   │   ├── PaymentService.cs
│   │   └── DashboardService.cs
│   ├── Interfaces/
│   │   ├── IAuthService.cs
│   │   ├── IEmployeeService.cs
│   │   ├── IPensionService.cs
│   │   └── IDashboardService.cs
│   └── Security/
│       └── PermissionGuard.cs
│
├── Infrastructure/                       # Depends on Domain
│   ├── Data/
│   │   ├── PensionDbContext.cs
│   │   └── Migrations/
│   └── Repositories/
│       ├── Repository.cs
│       ├── EmployeeRepository.cs
│       ├── PensionRepository.cs
│       ├── PaymentRepository.cs
│       └── UnitOfWork.cs
│
├── Presentation/                         # ASP.NET Core MVC
│   ├── Controllers/
│   │   ├── AuthController.cs
│   │   ├── DashboardController.cs
│   │   ├── EmployeeController.cs
│   │   ├── PensionController.cs
│   │   └── PaymentController.cs
│   ├── Views/
│   │   ├── Auth/
│   │   │   ├── Login.cshtml
│   │   │   └── Register.cshtml
│   │   ├── Dashboard/
│   │   │   └── Index.cshtml
│   │   ├── Employee/
│   │   │   ├── Index.cshtml
│   │   │   ├── Create.cshtml
│   │   │   ├── Edit.cshtml
│   │   │   └── Details.cshtml
│   │   ├── Pension/
│   │   │   ├── Index.cshtml
│   │   │   ├── Create.cshtml
│   │   │   └── Details.cshtml
│   │   └── Shared/
│   │       ├── _Layout.cshtml
│   │       ├── _Navbar.cshtml
│   │       └── _ValidationScripts.cshtml
│   ├── ViewModels/
│   │   ├── LoginViewModel.cs
│   │   ├── EmployeeViewModel.cs
│   │   ├── PensionViewModel.cs
│   │   └── DashboardViewModel.cs
│   └── wwwroot/
│       ├── css/
│       ├── js/
│       └── lib/                          # Bootstrap 5, jQuery
│
├── Tests/
│   ├── EmployeeServiceTests.cs
│   ├── PensionServiceTests.cs
│   └── PermissionGuardTests.cs
│
├── appsettings.json
├── architecture.md
├── README.md
└── Program.cs                            # DI registration + middleware pipeline
```

---

## 10. Class Diagram Summary

```
SystemUser (abstract)
├── SystemAdmin
├── PensionAdmin
├── PensionManager
└── Viewer

Department (1:N) ──── Employee (1:1) ──── Pension (1:N) ──── Payment
                           │
                          User (1:1) ──── Login

IRepository<T>
├── IEmployeeRepository
├── IPensionRepository
└── IPaymentRepository

IUnitOfWork
└── UnitOfWork
    ├── IEmployeeRepository
    ├── IPensionRepository
    └── IPaymentRepository

Controller ──► IService ──► IRepository ──► DbContext ──► SQL Server
```

---

## 11. Architecture Decision Record (ADR)

| Decision | Choice | Reason |
|---|---|---|
| Language | C# .NET 8 | OOP-native, strongly typed, enterprise-ready |
| Web Framework | ASP.NET Core MVC | Industry standard, Razor views, clean MVC separation |
| Database | SQL Server Express | ACID compliant, EF Core native, free tier available |
| ORM | Entity Framework Core | Code-first migrations, LINQ queries, reduces boilerplate |
| Pattern | Repository + Unit of Work | Decouples DB logic, fully testable, maintainable |
| Access Control | RBAC (ASP.NET Core Identity + PermissionGuard) | Matches 4 defined roles, built-in middleware support |
| Architecture | 4-Layer Clean Architecture | Clear separation of concerns, scalable, independently testable |
| DI Container | Microsoft.Extensions.DI | Built-in to ASP.NET Core, no extra libraries needed |
| View Engine | Razor (.cshtml) | Server-side rendering, native to ASP.NET Core MVC |
| Frontend | Bootstrap 5 | Responsive UI, fast to set up, widely supported |
| Testing | xUnit + Moq | Industry standard for C# unit and mock testing |

---

## 12. Quick Start Checklist

- [ ] Create SQL Server Express database `GovtPensionDB`
- [ ] Configure connection string in `appsettings.json`
- [ ] Implement Domain entities and enums
- [ ] Implement `IRepository<T>` and specific repository interfaces
- [ ] Implement `Repository<T>`, specific repositories, and `UnitOfWork`
- [ ] Implement Application services with SOLID principles
- [ ] Add `PermissionGuard` for service-level RBAC
- [ ] Register all services and repositories in `Program.cs` (DI)
- [ ] Build Controllers with `[Authorize(Roles = "...")]` attributes
- [ ] Create Razor Views with role-conditional UI rendering
- [ ] Apply EF Core migrations (`dotnet ef database update`)
- [ ] Write unit tests for services and permission guard
- [ ] Final integration and browser testing

---

> "Good architecture is not about tools — it's about clarity, responsibility, and the ability to change without fear."
> — Senior Architect Principle
