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

**Govt. BD Pension** is a desktop-based Pension Management System built with **C# Windows Forms** and **SQL Server**, designed to digitalize and centralize government pension record management.

Many government offices still manage pension records manually or through partially digital systems — leading to slow processing, repeated data entries, lack of transparency, and record-handling mistakes. This system solves those problems by providing a secure, structured, and role-controlled platform.

> Built as an academic application of **OOP concepts**, **GUI design**, **SOLID principles**, and **database integration**, based on a real-life government pension management scenario.

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
- 4-layer clean architecture (Presentation, Application, Domain, Infrastructure)
- Generic Repository Pattern + Unit of Work
- SOLID design principles throughout
- Entity Framework Core with SQL Server
- Role-Based Access Control (RBAC) with `PermissionGuard`
- Dependency Injection via `Microsoft.Extensions.DependencyInjection`

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
| Language | C# (.NET) |
| UI Framework | Windows Forms |
| Database | Microsoft SQL Server Express |
| ORM | Entity Framework Core |
| Architecture | 4-Layer (Presentation / Application / Domain / Infrastructure) |
| Pattern | Repository Pattern + Unit of Work |
| Access Control | Role-Based Access Control (RBAC) |
| Dependency Injection | Microsoft.Extensions.DependencyInjection |
| Testing | xUnit + Moq |

---

## Architecture

```
┌──────────────────────────────────────────────┐
│           PRESENTATION LAYER                 │
│   Windows Forms UI                           │
│   Login, Dashboard, Employee, Pension Forms  │
└───────────────────┬──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│           APPLICATION LAYER                  │
│   Business Logic & Services                  │
│   AuthService, EmployeeService               │
│   PensionService, PaymentService             │
│   PermissionGuard (RBAC)                     │
└───────────────────┬──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│           DOMAIN LAYER                       │
│   Core Entities & Interfaces                 │
│   Employee, Pension, Payment, Department     │
│   IRepository<T>, IUnitOfWork                │
└───────────────────┬──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│           INFRASTRUCTURE LAYER               │
│   EF Core DbContext + Repositories           │
│   EmployeeRepository, PensionRepository      │
│   SQL Server Database                        │
└──────────────────────────────────────────────┘
```

For full architecture details, see [architecture.md](architecture.md).

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
User (1:1) ──── Login
Department (1:N) ──── Employee
Employee (1:1) ──── Pension
Pension (1:N) ──── Payment
```

### Database Setup (SQL Server)

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
├── Domain/
│   ├── Entities/
│   │   ├── Employee.cs
│   │   ├── Pension.cs
│   │   ├── Payment.cs
│   │   ├── Department.cs
│   │   ├── SystemUser.cs
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
├── Application/
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
│       └── PermissionGuard.cs
│
├── Infrastructure/
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
├── Presentation/
│   └── Forms/
│       ├── LoginForm.cs
│       ├── DashboardForm.cs
│       ├── EmployeeForm.cs
│       └── PensionForm.cs
│
├── Tests/
│   ├── EmployeeServiceTests.cs
│   ├── PensionServiceTests.cs
│   └── PermissionGuardTests.cs
│
├── architecture.md
├── README.md
└── Program.cs
```

---

## Getting Started

### Prerequisites

- [Visual Studio 2022](https://visualstudio.microsoft.com/) or later
- [.NET 8 SDK](https://dotnet.microsoft.com/)
- [SQL Server Express](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) (free)
- [SQL Server Management Studio (SSMS)](https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms) (optional)

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/your-username/GovtPensionSystem.git
   cd GovtPensionSystem
   ```

2. **Setup the database**
   ```bash
   # Open SQL Server Management Studio and create database
   CREATE DATABASE GovtPensionDB;
   ```

3. **Configure connection string**

   Update `appsettings.json` or `App.config`:
   ```json
   "ConnectionStrings": {
     "PensionDB": "Server=.;Database=GovtPensionDB;Trusted_Connection=True;"
   }
   ```

4. **Run EF Core Migrations**
   ```bash
   dotnet ef database update
   ```

5. **Build and run**
   ```bash
   dotnet build
   dotnet run
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

This project is built following industry-standard software engineering principles:

### OOP (Object-Oriented Programming)
| Principle | Applied In |
|---|---|
| **Encapsulation** | Private fields with controlled property access in all entities |
| **Inheritance** | `SystemAdmin`, `PensionAdmin`, `PensionManager`, `Viewer` extend `SystemUser` |
| **Polymorphism** | Role-based behavior using overridden methods (`GetRole()`, `CanDelete()`) |
| **Abstraction** | Service interfaces hide complex business and DB logic |

### SOLID Principles
| Principle | Applied In |
|---|---|
| **Single Responsibility** | Each service class handles one domain (Employee, Pension, Payment) |
| **Open/Closed** | `PensionCalculator` extended without modifying base logic |
| **Liskov Substitution** | All role subclasses safely replace `SystemUser` base class |
| **Interface Segregation** | `IViewable`, `IManageable`, `IAdministrable` split by responsibility |
| **Dependency Inversion** | All services depend on interfaces, injected via DI container |

### Repository Pattern
- Generic `IRepository<T>` base interface
- Specific repositories (`IEmployeeRepository`, `IPensionRepository`)
- `UnitOfWork` coordinates all repositories in a single transaction

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit changes: `git commit -m "Add your feature"`
4. Push to branch: `git push origin feature/your-feature`
5. Open a Pull Request

---

## License

This project is developed for academic purposes under standard academic use guidelines.

---

> "Managing pension data is not just a technical task — it is a responsibility to the people who served."
