# Government Pension Management System
### Govt. BD Pension — Centralized Pension Administration for Government Institutions

---

## Table of Contents
1. [About the Project](#about-the-project)
2. [Features](#features)
3. [User Roles](#user-roles)
4. [Tech Stack](#tech-stack)
5. [Architecture](#architecture)
6. [Database Design](#database-design)
7. [Project Structure](#project-structure)
8. [Getting Started](#getting-started)
9. [Role Permission Matrix](#role-permission-matrix)
10. [Design Principles](#design-principles)
11. [Contributing](#contributing)

---

## About the Project

**Govt. BD Pension** is a web-based Pension Management System built with **ASP.NET Core MVC** and **SQL Server**, designed to digitalize and centralize government pension record management.

Many government offices still manage pension records manually or through partially digital systems — leading to slow processing, repeated data entries, lack of transparency, and record-handling mistakes. This system solves those problems by providing a secure, structured, and role-controlled web platform.

> Built applying **OOP concepts**, **SOLID principles**, **Repository Pattern**, and **Role-Based Access Control (RBAC)**, based on a real-life government pension management scenario.

---

## Features

### Core Capabilities
- Secure role-based login and authentication
- Employee record management (Add / Edit / Delete / Search)
- Pension data management and status tracking
- Payment history tracking per pension record
- Department-based employee classification
- Dashboard analytics and reporting
- Filter employees by status (Active / Retired / Pending)

### Technical Highlights
- ASP.NET Core MVC — clean separation of Models, Views, Controllers
- 4-layer clean architecture (Presentation, Application, Domain, Infrastructure)
- Generic Repository Pattern + Unit of Work
- SOLID design principles throughout
- Entity Framework Core with SQL Server
- Role-Based Access Control (RBAC) with ASP.NET Core Identity
- Dependency Injection via `Microsoft.Extensions.DependencyInjection`
- Razor Views for dynamic server-side rendered UI

---

## User Roles

The system uses **Role-Based Access Control (RBAC)** with 4 distinct roles:

| Role | Description |
|---|---|
| **System Admin** | Highest authority — manages users, assigns roles, configures system |
| **Pension Admin** | Manages employee and pension records, views analytics |
| **Pension Manager** | Monitors pension status in read-only mode, verifies data |
| **Viewer** | Limited read-only access to basic employee information |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | C# (.NET 8) |
| Web Framework | ASP.NET Core MVC |
| View Engine | Razor (.cshtml) |
| Database | Microsoft SQL Server Express |
| ORM | Entity Framework Core |
| Authentication | ASP.NET Core Identity |
| Architecture | 4-Layer Clean Architecture |
| Pattern | Repository Pattern + Unit of Work |
| Access Control | Role-Based Access Control (RBAC) |
| Dependency Injection | Microsoft.Extensions.DependencyInjection |
| Frontend Styling | Bootstrap 5 |
| Testing | xUnit + Moq |

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│               PRESENTATION LAYER                     │
│         ASP.NET Core MVC                             │
│                                                      │
│   Controllers/          Views/                       │
│   ├── HomeController    ├── Home/                    │
│   ├── AuthController    ├── Auth/                    │
│   ├── EmployeeCtrl      ├── Employee/                │
│   ├── PensionCtrl       ├── Pension/                 │
│   └── DashboardCtrl     └── Dashboard/               │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│               APPLICATION LAYER                      │
│         Business Logic & Services                    │
│                                                      │
│   Services/             Interfaces/                  │
│   ├── AuthService       ├── IAuthService             │
│   ├── EmployeeService   ├── IEmployeeService         │
│   ├── PensionService    ├── IPensionService          │
│   ├── PaymentService    └── IPaymentService          │
│   └── DashboardService                               │
│                                                      │
│   Security/                                          │
│   └── PermissionGuard   (RBAC enforcement)           │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│               DOMAIN LAYER                           │
│         Core Entities & Interfaces                   │
│                                                      │
│   Entities/             Interfaces/                  │
│   ├── Employee          ├── IRepository<T>           │
│   ├── Pension           ├── IEmployeeRepository      │
│   ├── Payment           ├── IPensionRepository       │
│   ├── Department        ├── IPaymentRepository       │
│   ├── SystemUser        └── IUnitOfWork              │
│   └── Login                                          │
│                                                      │
│   Enums/                                             │
│   ├── UserRole                                       │
│   └── PensionStatus                                  │
└───────────────────────┬──────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────┐
│               INFRASTRUCTURE LAYER                   │
│         EF Core + Repositories + DB                  │
│                                                      │
│   Data/                 Repositories/                │
│   ├── PensionDbContext  ├── Repository.cs (generic)  │
│   └── Migrations/       ├── EmployeeRepository       │
│                         ├── PensionRepository        │
│                         ├── PaymentRepository        │
│                         └── UnitOfWork               │
└──────────────────────────────────────────────────────┘
```

For full architecture and code details, see [architecture.md](architecture.md).

---

## Database Design

The database is normalized up to **Second Normal Form (2NF)** with 6 core tables:

```
Login      → UserID, Password, UserType
User       → UserID, UserName, Email, Phone
Department → DeptID, DeptName
Employee   → EmpID, EmpName, EmpAddress, EmpPhone, JoinDate, RetirementDate, Role, DeptID (FK)
Pension    → PensionID, PensionAmount, PensionStatus, StartDate, EmpID (FK)
Payment    → PaymentID, PayDate, Method, Amount, PensionID (FK)
```

### Entity Relationships

```
User       (1:1)  ────  Login
Department (1:N)  ────  Employee
Employee   (1:1)  ────  Pension
Pension    (1:N)  ────  Payment
```

### Database Setup

```sql
CREATE DATABASE GovtPensionDB;
```

### Connection String

```json
"ConnectionStrings": {
  "PensionDB": "Server=.;Database=GovtPensionDB;Trusted_Connection=True;"
}
```

---

## Project Structure

```
GovtPensionSystem/
│
├── Domain/                               # Core business rules — no dependencies
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
├── Application/                          # Business logic — depends only on Domain
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
│       └── PermissionGuard.cs            # RBAC permission enforcement
│
├── Infrastructure/                       # DB access — depends on Domain
│   ├── Data/
│   │   ├── PensionDbContext.cs
│   │   └── Migrations/
│   └── Repositories/
│       ├── Repository.cs                 # Generic base repository
│       ├── EmployeeRepository.cs
│       ├── PensionRepository.cs
│       ├── PaymentRepository.cs
│       └── UnitOfWork.cs
│
├── Presentation/                         # ASP.NET Core MVC — depends on Application
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
│   │   └── PensionViewModel.cs
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
├── architecture.md
├── README.md
├── appsettings.json
└── Program.cs                            # DI setup + middleware pipeline
```

---

## Getting Started

### Prerequisites

- [Visual Studio 2022](https://visualstudio.microsoft.com/) or [VS Code](https://code.visualstudio.com/)
- [.NET 8 SDK](https://dotnet.microsoft.com/en-us/download)
- [SQL Server Express](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) (free)
- [SQL Server Management Studio (SSMS)](https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms) (optional)

### Installation

**1. Clone the repository**
```bash
git clone https://github.com/your-username/GovtPensionSystem.git
cd GovtPensionSystem
```

**2. Configure the connection string**

Edit `appsettings.json`:
```json
"ConnectionStrings": {
  "PensionDB": "Server=.;Database=GovtPensionDB;Trusted_Connection=True;TrustServerCertificate=True;"
}
```

**3. Apply EF Core migrations**
```bash
dotnet ef database update
```

**4. Run the application**
```bash
dotnet run
```

**5. Open in browser**
```
https://localhost:5001
```

---

## Role Permission Matrix

| Feature | System Admin | Pension Admin | Pension Manager | Viewer |
|---|:---:|:---:|:---:|:---:|
| Login | YES | YES | YES | YES |
| Create / Delete Users | YES | NO | NO | NO |
| Add Employee | NO | YES | NO | NO |
| Edit Employee | NO | YES | NO | NO |
| Delete Employee | NO | YES | NO | NO |
| Manage Pension Data | NO | YES | NO | NO |
| View Employee List | YES | YES | YES | YES |
| View Employee Details | YES | YES | YES | NO |
| Monitor Pension Status | YES | YES | YES | NO |
| View Dashboard Analytics | YES | YES | YES | NO |
| System Configuration | YES | NO | NO | NO |

---

## Design Principles

### OOP (Object-Oriented Programming)

| Principle | Applied In |
|---|---|
| **Encapsulation** | Private fields with controlled property access in all entities |
| **Inheritance** | `SystemAdmin`, `PensionAdmin`, `PensionManager`, `Viewer` extend `SystemUser` |
| **Polymorphism** | Role-based behavior via overridden `GetRole()` and `CanDelete()` methods |
| **Abstraction** | Service interfaces hide complex business and DB logic from controllers |

### SOLID Principles

| Principle | Applied In |
|---|---|
| **Single Responsibility** | Each controller and service handles one domain only |
| **Open / Closed** | `PensionCalculator` extended without modifying base logic |
| **Liskov Substitution** | All role subclasses safely replace `SystemUser` base class |
| **Interface Segregation** | `IViewable`, `IManageable`, `IAdministrable` split by responsibility |
| **Dependency Inversion** | Controllers depend on service interfaces injected via DI container |

### Repository Pattern

| Component | Purpose |
|---|---|
| `IRepository<T>` | Generic base — GetById, GetAll, Add, Update, Delete |
| `IEmployeeRepository` | Employee-specific queries (filter by status, search by ID) |
| `IPensionRepository` | Pension-specific queries (get active pensions, get by employee) |
| `UnitOfWork` | Coordinates all repositories in a single DB transaction |

### MVC Flow

```
HTTP Request
    │
    ▼
Controller  ──►  Service (Application Layer)
    │                  │
    │            Repository (Infrastructure)
    │                  │
    │            SQL Server Database
    │
    ▼
View (.cshtml) ──► HTTP Response
```

---

## Contributing

1. Fork the repository
2. Create a feature branch
   ```bash
   git checkout -b feature/your-feature-name
   ```
3. Commit your changes
   ```bash
   git commit -m "Add: your feature description"
   ```
4. Push to branch
   ```bash
   git push origin feature/your-feature-name
   ```
5. Open a Pull Request

---

## License

This project is developed for academic purposes under standard academic use guidelines.

---

> "Managing pension data is not just a technical task — it is a responsibility to the people who served."
