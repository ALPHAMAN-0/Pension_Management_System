# Government Pension Management System — Architecture Guide
> Designed by: Senior Software Architect Perspective (Google / Amazon Standard)
> Language: C# | Platform: Desktop (Windows Forms) / or ASP.NET Core (Web)

---

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Database Selection](#2-database-selection)
3. [Layered Architecture](#3-layered-architecture)
4. [OOP Design](#4-oop-design)
5. [Role-Based Access Control (RBAC)](#5-role-based-access-control-rbac)
6. [SOLID Principles Applied](#6-solid-principles-applied)
7. [Repository Pattern](#7-repository-pattern)
8. [Project Folder Structure](#8-project-folder-structure)
9. [Class Diagram Summary](#9-class-diagram-summary)
10. [Architecture Decision Record (ADR)](#10-architecture-decision-record-adr)

---

## 1. System Overview

```
┌─────────────────────────────────────────────────────────────┐
│              Govt. BD Pension Management System             │
│                                                             │
│   Users: System Admin | Pension Admin | Manager | Viewer    │
│   Tech:  C# | Windows Forms or ASP.NET Core | SQL Server    │
│   Pattern: Layered Architecture + Repository + RBAC         │
└─────────────────────────────────────────────────────────────┘
```

**Core Modules:**
- Authentication & Authorization (RBAC)
- Employee Management
- Pension Management
- Payment Tracking
- Department Management
- Dashboard & Analytics

---

## 2. Database Selection

### Recommendation: Microsoft SQL Server (MSSQL)

| Option        | Verdict  | Reason                                              |
|---------------|----------|-----------------------------------------------------|
| **SQL Server**| BEST     | Native C# integration, Entity Framework support, enterprise-grade |
| SQLite        | OK       | Good for small desktop apps, no server needed       |
| MySQL         | OK       | Open source but less native to C#                  |
| PostgreSQL    | OK       | Powerful but overkill for desktop                  |
| MongoDB       | NO       | NoSQL — not suitable for relational pension data    |

### Why SQL Server?
- **Entity Framework Core (EF Core)** integrates perfectly with C# and MSSQL
- Built-in support for **stored procedures**, **transactions**, and **triggers**
- **ACID compliance** — critical for financial/pension data
- **Windows Authentication** — secure, no extra config for government systems
- Free tier available: **SQL Server Express** (up to 10GB, enough for this system)

### Connection (EF Core)
```csharp
// appsettings.json
"ConnectionStrings": {
  "PensionDB": "Server=.;Database=GovtPensionDB;Trusted_Connection=True;"
}

// DbContext
services.AddDbContext<PensionDbContext>(options =>
    options.UseSqlServer(configuration.GetConnectionString("PensionDB")));
```

---

## 3. Layered Architecture

```
┌──────────────────────────────────────────────┐
│           PRESENTATION LAYER                 │
│   Windows Forms / ASP.NET MVC Views          │
│   - Login Form, Dashboard, Employee Forms    │
└───────────────────┬──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│           APPLICATION / SERVICE LAYER        │
│   Business Logic & Use Case Handlers         │
│   - AuthService, EmployeeService             │
│   - PensionService, PaymentService           │
└───────────────────┬──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│           DOMAIN LAYER                       │
│   Core Entities & Interfaces                 │
│   - Employee, Pension, Payment, Department   │
│   - IRepository<T>, IUnitOfWork              │
└───────────────────┬──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│           INFRASTRUCTURE / DATA LAYER        │
│   EF Core DbContext + Repository Impl        │
│   - EmployeeRepository, PensionRepository    │
│   - SQL Server Database                      │
└──────────────────────────────────────────────┘
```

---

## 4. OOP Design

### 4.1 Encapsulation
```csharp
public class Employee
{
    // Private fields — data is protected
    private string _empName;
    private DateTime _retirementDate;

    // Public properties — controlled access
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

### 4.2 Inheritance
```csharp
// Base class
public abstract class SystemUser
{
    public int UserId { get; set; }
    public string UserName { get; set; }
    public string Email { get; set; }
    public abstract string GetRole();
    public abstract bool CanDelete();
}

// Derived classes
public class SystemAdmin : SystemUser
{
    public override string GetRole() => "SystemAdmin";
    public override bool CanDelete() => true;
}

public class PensionAdmin : SystemUser
{
    public override string GetRole() => "PensionAdmin";
    public override bool CanDelete() => true;
}

public class PensionManager : SystemUser
{
    public override string GetRole() => "PensionManager";
    public override bool CanDelete() => false;
}

public class Viewer : SystemUser
{
    public override string GetRole() => "Viewer";
    public override bool CanDelete() => false;
}
```

### 4.3 Polymorphism
```csharp
// Same method — different behavior per role
List<SystemUser> users = new List<SystemUser>
{
    new SystemAdmin(),
    new PensionManager(),
    new Viewer()
};

foreach (var user in users)
{
    Console.WriteLine($"{user.GetRole()} - Can Delete: {user.CanDelete()}");
}
```

### 4.4 Abstraction
```csharp
public interface IPensionService
{
    Task<Pension> GetPensionByEmployeeIdAsync(int empId);
    Task<bool> UpdatePensionStatusAsync(int pensionId, string status);
    Task<IEnumerable<Pension>> GetAllActivePensionsAsync();
}

// Complex DB logic hidden behind simple interface
public class PensionService : IPensionService { /* implementation */ }
```

---

## 5. Role-Based Access Control (RBAC)

### Permission Matrix

| Feature                  | System Admin | Pension Admin | Pension Manager | Viewer |
|--------------------------|:---:|:---:|:---:|:---:|
| Login                    | YES | YES | YES | YES |
| Create/Delete Users      | YES | NO  | NO  | NO  |
| Add Employee             | NO  | YES | NO  | NO  |
| Edit Employee            | NO  | YES | NO  | NO  |
| Delete Employee          | NO  | YES | NO  | NO  |
| Manage Pension Data      | NO  | YES | NO  | NO  |
| View Employee List       | YES | YES | YES | YES |
| View Employee Details    | YES | YES | YES | NO  |
| Monitor Pension Status   | YES | YES | YES | NO  |
| View Dashboard Analytics | YES | YES | YES | NO  |
| System Configuration     | YES | NO  | NO  | NO  |

### RBAC Implementation
```csharp
// Role enum
public enum UserRole
{
    SystemAdmin,
    PensionAdmin,
    PensionManager,
    Viewer
}

// Permission checker
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

// Usage in service
public class EmployeeService
{
    public void DeleteEmployee(SystemUser currentUser, int empId)
    {
        PermissionGuard.Enforce(currentUser.Role, "ManageEmployee");

        if (!currentUser.CanDelete())
            throw new UnauthorizedAccessException("This role cannot delete records.");

        // proceed with deletion
    }
}
```

---

## 6. SOLID Principles Applied

### S — Single Responsibility
```csharp
// WRONG — one class doing everything
public class EmployeeManager
{
    public void AddEmployee() { }
    public void SendEmail() { }       // NOT employee's job
    public void GenerateReport() { } // NOT employee's job
}

// CORRECT — each class has one job
public class EmployeeService   { public void AddEmployee() { } }
public class EmailService      { public void SendEmail() { } }
public class ReportService     { public void GenerateReport() { } }
```

### O — Open/Closed
```csharp
// Open for extension, closed for modification
public abstract class PensionCalculator
{
    public abstract decimal Calculate(Employee emp);
}

public class StandardPensionCalculator : PensionCalculator
{
    public override decimal Calculate(Employee emp)
        => emp.BasicSalary * 0.80m;
}

public class SpecialPensionCalculator : PensionCalculator
{
    public override decimal Calculate(Employee emp)
        => emp.BasicSalary * 0.90m; // extended, not modified
}
```

### L — Liskov Substitution
```csharp
// Any subclass of SystemUser can replace the base class safely
public void ShowUserDashboard(SystemUser user)
{
    Console.WriteLine($"Welcome {user.UserName} — Role: {user.GetRole()}");
}

// Works with ALL derived types
ShowUserDashboard(new SystemAdmin());
ShowUserDashboard(new Viewer());
```

### I — Interface Segregation
```csharp
// WRONG — fat interface
public interface IUserActions
{
    void AddEmployee();
    void DeleteEmployee();
    void ViewEmployee();   // Viewer only needs this
    void ManageUsers();
}

// CORRECT — split by responsibility
public interface IViewable    { Task<IEnumerable<Employee>> GetAllAsync(); }
public interface IManageable  { Task AddAsync(Employee e); Task DeleteAsync(int id); }
public interface IAdministrable { Task ManageUsersAsync(); }

// Viewer only implements what it needs
public class ViewerDashboard : IViewable { /* only view */ }

// Pension Admin implements more
public class PensionAdminDashboard : IViewable, IManageable { /* view + manage */ }
```

### D — Dependency Inversion
```csharp
// WRONG — depends on concrete class
public class EmployeeService
{
    private EmployeeRepository _repo = new EmployeeRepository(); // tightly coupled
}

// CORRECT — depends on abstraction
public class EmployeeService
{
    private readonly IEmployeeRepository _repo;

    public EmployeeService(IEmployeeRepository repo) // injected
    {
        _repo = repo;
    }
}

// Register in DI container
services.AddScoped<IEmployeeRepository, EmployeeRepository>();
services.AddScoped<IEmployeeService, EmployeeService>();
```

---

## 7. Repository Pattern

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

### Specific Repository Interface
```csharp
public interface IEmployeeRepository : IRepository<Employee>
{
    Task<IEnumerable<Employee>> GetByStatusAsync(string status);   // Active / Retired
    Task<Employee?> GetByDepartmentAsync(int deptId);
    Task<Employee?> SearchByIdAsync(string empId);
}

public interface IPensionRepository : IRepository<Pension>
{
    Task<Pension?> GetByEmployeeIdAsync(int empId);
    Task<IEnumerable<Pension>> GetActivePensionsAsync();
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

    public async Task<T?> GetByIdAsync(int id)        => await _dbSet.FindAsync(id);
    public async Task<IEnumerable<T>> GetAllAsync()   => await _dbSet.ToListAsync();
    public async Task AddAsync(T entity)               { await _dbSet.AddAsync(entity); await _context.SaveChangesAsync(); }
    public async Task UpdateAsync(T entity)            { _dbSet.Update(entity); await _context.SaveChangesAsync(); }
    public async Task DeleteAsync(int id)
    {
        var entity = await GetByIdAsync(id);
        if (entity != null) { _dbSet.Remove(entity); await _context.SaveChangesAsync(); }
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

## 8. Project Folder Structure

```
GovtPensionSystem/
│
├── Domain/                         # Core business entities & interfaces
│   ├── Entities/
│   │   ├── Employee.cs
│   │   ├── Pension.cs
│   │   ├── Payment.cs
│   │   ├── Department.cs
│   │   ├── SystemUser.cs           # Base user class
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
├── Application/                    # Business logic / services
│   ├── Services/
│   │   ├── AuthService.cs
│   │   ├── EmployeeService.cs
│   │   ├── PensionService.cs
│   │   ├── PaymentService.cs
│   │   └── DashboardService.cs
│   ├── Interfaces/
│   │   ├── IAuthService.cs
│   │   ├── IEmployeeService.cs
│   │   └── IPensionService.cs
│   └── Security/
│       └── PermissionGuard.cs      # RBAC enforcement
│
├── Infrastructure/                 # DB, EF Core, Repositories
│   ├── Data/
│   │   ├── PensionDbContext.cs
│   │   └── Migrations/
│   └── Repositories/
│       ├── Repository.cs           # Generic base
│       ├── EmployeeRepository.cs
│       ├── PensionRepository.cs
│       ├── PaymentRepository.cs
│       └── UnitOfWork.cs
│
├── Presentation/                   # UI Layer (Windows Forms)
│   ├── Forms/
│   │   ├── LoginForm.cs
│   │   ├── DashboardForm.cs
│   │   ├── EmployeeForm.cs
│   │   └── PensionForm.cs
│   └── Controllers/                # (if ASP.NET Core)
│       ├── EmployeeController.cs
│       └── PensionController.cs
│
├── Tests/                          # Unit & Integration tests
│   ├── EmployeeServiceTests.cs
│   ├── PensionServiceTests.cs
│   └── PermissionGuardTests.cs
│
└── Program.cs                      # Entry point + DI setup
```

---

## 9. Class Diagram Summary

```
SystemUser (abstract)
├── SystemAdmin
├── PensionAdmin
├── PensionManager
└── Viewer

Department ──── Employee ──── Pension ──── Payment
                   │
                  User ──── Login

IRepository<T>
├── IEmployeeRepository
├── IPensionRepository
└── IPaymentRepository

IUnitOfWork
└── UnitOfWork (aggregates all repositories)
```

---

## 10. Architecture Decision Record (ADR)

| Decision | Choice | Reason |
|---|---|---|
| Language | C# | OOP-native, strong typing, enterprise support |
| Database | SQL Server Express | ACID, EF Core native, Windows Auth, free |
| ORM | Entity Framework Core | Reduces boilerplate, migrations, LINQ queries |
| Pattern | Repository + UoW | Decouples DB logic, testable, maintainable |
| Access Control | RBAC (Role-Based) | Matches 4 defined roles in requirements |
| Architecture | Layered (4-tier) | Clean separation, scalable, testable |
| DI Container | Microsoft.Extensions.DI | Built-in, no extra library needed |
| Testing | xUnit + Moq | Industry standard for C# unit testing |

---

## Quick Start Checklist

- [ ] Create SQL Server Express database `GovtPensionDB`
- [ ] Set up EF Core and run migrations
- [ ] Implement Domain entities
- [ ] Implement Generic Repository and specific repositories
- [ ] Implement Unit of Work
- [ ] Implement Services with SOLID principles
- [ ] Add PermissionGuard for RBAC
- [ ] Build UI Forms with role-based visibility
- [ ] Write unit tests for services
- [ ] Final integration testing

---

> "Good architecture is not about tools — it's about clarity, responsibility, and the ability to change without fear."
> — Senior Architect Principle
