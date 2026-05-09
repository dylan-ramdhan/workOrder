
# 🏭 MES Direct Ink Jet - Enterprise Manufacturing Execution System

[![.NET](https://img.shields.io/badge/.NET-8.0-512BD4)](https://dotnet.microsoft.com/)
[![Blazor](https://img.shields.io/badge/Blazor-Server-512BD4)](https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor)
[![MudBlazor](https://img.shields.io/badge/UI-MudBlazor-7E6FFF)](https://mudblazor.com/)
[![Database](https://img.shields.io/badge/DB-SQL%20Server-CC2927)](https://www.microsoft.com/en-us/sql-server)

A Production-Ready, Realtime Manufacturing Execution System (MES) specifically designed for the **Direct Ink Jet** production area in a Mattel-style manufacturing environment. This system serves as the central digital operational hub for managing Work Orders, tracking production output, monitoring OEE, and logging defects/downtimes in real-time.

---

## 🌟 Key Features

- **Work Order Management:** Full lifecycle tracking (Draft -> Released -> Running -> Complete).
- **Realtime Production Monitoring:** Live dashboard updating output and line status instantly via SignalR.
- **OEE Calculation Engine:** Realtime calculation of Availability, Performance, and Quality.
- **Quality Control (QC):** Defect tracking and analysis per Work Order.
- **Downtime Monitoring:** Root cause tracking and automatic OEE Availability adjustment.
- **Industrial Dark UI:** High-contrast, large readable KPIs designed for production floor PC displays.
- **Role-Based Access Control (RBAC):** Granular permissions for Operator, Leader, Supervisor, QC, Engineering, and Admin.
- **Export Reporting:** Generate daily production reports to Excel instantly.

---

## 🏗️ Architecture & Tech Stack

This project uses a **Modular Monolith Architecture**, chosen for its stability, faster development cycle, and suitability for a single production area, while maintaining strict separation of concerns.

| Layer | Technology | Description |
| :--- | :--- | :--- |
| **Backend** | ASP.NET Core 8 Blazor Server | Real-time web framework, C# runs entirely on the server |
| **Frontend** | MudBlazor | Material Design component library (Dark Industrial Theme) |
| **Database** | Microsoft SQL Server | Relational DB managed via Entity Framework Core |
| **Realtime** | SignalR | WebSockets for instant dashboard updates |
| **Reporting** | ClosedXML | Excel (.xlsx) generation engine |
| **Hosting** | IIS Windows Server | Production deployment target |

### Project Structure
```text
MES.DirectInkJet/
├── src/
│   ├── MES.Core/              # Entities, Enums, Interfaces (No dependencies)
│   ├── MES.Infrastructure/    # EF Core DbContext, Repositories (Depends on Core)
│   ├── MES.Application/       # Business Logic, OEE Engine, Services (Depends on Core, Infra)
│   └── MES.Server/            # Blazor UI, SignalR Hubs, DI Configuration (Depends on App)
└── MES.DirectInkJet.sln
```

---

## 🚀 Getting Started (Installation)

### Prerequisites
- Windows 10/11 or Windows Server
- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- [SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) (Express/Developer/Standard)
- Visual Studio 2022 (Recommended) or VS Code

### Step 1: Clone or Setup Project
If you are starting from the blueprint, create the project structure using the terminal:
```bash
dotnet new sln -n MES.DirectInkJet
dotnet new classlib -n MES.Core -o src/MES.Core
dotnet new classlib -n MES.Infrastructure -o src/MES.Infrastructure
dotnet new classlib -n MES.Application -o src/MES.Application
dotnet new blazorserver -n MES.Server -o src/MES.Server

# Add to solution and setup references
dotnet sln add src/MES.Core src/MES.Infrastructure src/MES.Application src/MES.Server
dotnet add src/MES.Infrastructure reference src/MES.Core
dotnet add src/MES.Application reference src/MES.Core src/MES.Infrastructure
dotnet add src/MES.Server reference src/MES.Application
```

### Step 2: Install NuGet Packages
```bash
# Infrastructure
dotnet add src/MES.Infrastructure package Microsoft.EntityFrameworkCore.SqlServer
dotnet add src/MES.Infrastructure package Microsoft.EntityFrameworkCore.Tools
dotnet add src/MES.Infrastructure package Microsoft.AspNetCore.Identity.EntityFrameworkCore

# Application
dotnet add src/MES.Application package ClosedXML

# Server
dotnet add src/MES.Server package MudBlazor
```

### Step 3: Configure Database Connection
Edit `src/MES.Server/appsettings.json` and update the connection string to point to your SQL Server instance:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MES_DirectInkJet;Trusted_Connection=True;MultipleActiveResultSets=true;TrustServerCertificate=True"
  }
}
```

### Step 4: Apply Entity Framework Migrations
Run the following commands from the root directory to generate the database schema:
```bash
cd src/MES.Infrastructure
dotnet ef migrations add InitialCreate --startup-project ../MES.Server
dotnet ef database update --startup-project ../MES.Server
cd ../../
```

### Step 5: Run the Application
```bash
cd src/MES.Server
dotnet run
```
Open your browser and navigate to `https://localhost:5001` or `http://localhost:5000`.

---

## 🔐 Default Accounts

The system automatically seeds these accounts on the first run for UAT and initial setup:

| Email | Password | Role |
| :--- | :--- | :--- |
| `admin@mes.com` | `Password123!` | Admin |
| *(Create manually via Admin panel)* | | Operator, Supervisor, QC, Engineering, Leader |

---

## 📊 OEE Calculation Logic

Overall Equipment Effectiveness (OEE) is the core metric of this system, calculated in real-time upon every production input, defect, or downtime entry.

**OEE = Availability × Performance × Quality**

1. **Availability** = `Operating Time / Planned Production Time`
   - *Operating Time* decreases automatically when Engineering logs a downtime.
2. **Performance** = `(Ideal Cycle Time × Total Output) / Operating Time`
   - *Ideal Cycle Time* is fetched from the Master Product data.
3. **Quality** = `Good Output / Total Output`
   - Decreases automatically when QC logs a defect (increasing reject ratio).

---

## 👥 User Roles & Access

| Role | Primary Function | Access Highlights |
| :--- | :--- | :--- |
| **Operator** | Input daily production | Input Produksi page |
| **Leader** | Monitor line status | Live Dashboard |
| **Supervisor** | Manage production plans | Create & Release Work Orders |
| **QC** | Ensure product quality | Input Defects page |
| **Engineering** | Handle machine issues | Input Downtime page |
| **Admin** | System configuration | Master Data, User Management |

---

## 🌐 Deployment to Production (IIS)

1. **Publish the Project:**
   Run `dotnet publish src/MES.Server -c Release -o C:\MES_Publish`
2. **Server Requirements:**
   - Install the [.NET 8 Hosting Bundle](https://dotnet.microsoft.com/download/dotnet/8.0) on the Windows Server.
   - Enable **WebSocket Protocol** in IIS Features (Crucial for SignalR).
3. **IIS Setup:**
   - Create a new Website in IIS Manager.
   - Point the Physical path to the publish folder.
   - Set Application Pool `.NET CLR version` to **No Managed Code**.
4. **Firewall:**
   - Allow inbound connections on your designated port (e.g., 8080).

---

## 📄 License

This project is proprietary and intended for internal manufacturing use only.

---
*Built with ❤️ and Industrial-Grade Engineering.*
```
