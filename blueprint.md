Berikut adalah Blueprint Lengkap (Cetak Biru) untuk proyek MES Direct Ink Jet. Kamu bisa langsung *copy* seluruh teks di bawah ini, simpan ke dalam file bernama `MES_BLUEPRINT.md` di laptopmu, dan jadikan pedoman utama saat *coding*.

***

# 🏭 MES DIRECT INK JET - ENTERPRISE BLUEPRINT
**Production-Ready Manufacturing Execution System**

## 📋 1. RINGKASAN PROYEK
- **Tujuan:** Sistem MES berbasis web untuk area Direct Ink Jet (Pengelolaan WO, Monitoring Realtime, Defect, Downtime, OEE, Reporting).
- **Pola Arsitektur:** Modular Monolith (Stabil, cepat, mudah di-maintenance untuk 1 area produksi).
- **Teknologi Stack:**
  - Backend: ASP.NET Core 8 Blazor Server
  - Frontend: MudBlazor (Material Design Industrial Dark Mode)
  - Database: Microsoft SQL Server (EF Core Code-First)
  - Realtime: SignalR
  - Reporting: ClosedXML (Excel Export)

---

## 🏗️ 2. STRUKTUR PROYEK (MODULAR MONOLITH)

Jalankan perintah ini di terminal untuk membuat struktur awal:
```bash
dotnet new sln -n MES.DirectInkJet
dotnet new classlib -n MES.Core -o src/MES.Core
dotnet new classlib -n MES.Infrastructure -o src/MES.Infrastructure
dotnet new classlib -n MES.Application -o src/MES.Application
dotnet new blazorserver -n MES.Server -o src/MES.Server

dotnet sln add src/MES.Core src/MES.Infrastructure src/MES.Application src/MES.Server

dotnet add src/MES.Infrastructure reference src/MES.Core
dotnet add src/MES.Application reference src/MES.Core src/MES.Infrastructure
dotnet add src/MES.Server reference src/MES.Application
```

### NuGet Packages:
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

---

## 🗄️ 3. DESAIN DATABASE & ENTITY (MES.CORE)

### 3.1 Enums (`Enums.cs`)
```csharp
namespace MES.Core.Enums
{
    public enum WOStatus { Draft, Released, Running, Hold, Complete }
    public enum LineStatus { Idle, Running, Down }
}
```

### 3.2 Master Entities
```csharp
// Entities/ProductionLine.cs
namespace MES.Core.Entities
{
    public class ProductionLine
    {
        public Guid Id { get; set; }
        public string LineCode { get; set; } = string.Empty;
        public string LineName { get; set; } = string.Empty;
        public LineStatus Status { get; set; }
    }
}

// Entities/Product.cs
namespace MES.Core.Entities
{
    public class Product
    {
        public Guid Id { get; set; }
        public string ProductCode { get; set; } = string.Empty;
        public string ProductName { get; set; } = string.Empty;
        public double IdealCycleTimeSeconds { get; set; } // Kunci untuk OEE Performance
    }
}

// Entities/DefectCode.cs
namespace MES.Core.Entities
{
    public class DefectCode
    {
        public Guid Id { get; set; }
        public string Code { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
    }
}

// Entities/DowntimeCode.cs
namespace MES.Core.Entities
{
    public class DowntimeCode
    {
        public Guid Id { get; set; }
        public string Code { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
    }
}
```

### 3.3 Transactional Entities
```csharp
// Entities/WorkOrder.cs
using MES.Core.Enums;
namespace MES.Core.Entities
{
    public class WorkOrder
    {
        public Guid Id { get; set; }
        public string WONumber { get; set; } = string.Empty;
        public Guid ProductId { get; set; }
        public Product Product { get; set; } = null!;
        public Guid LineId { get; set; }
        public ProductionLine Line { get; set; } = null!;
        public int TargetQty { get; set; }
        public int GoodQty { get; set; }
        public int RejectQty { get; set; }
        public WOStatus Status { get; set; }
        public DateTime DueDate { get; set; }
    }
}

// Entities/ProductionResult.cs (Input Operator)
namespace MES.Core.Entities
{
    public class ProductionResult
    {
        public Guid Id { get; set; }
        public Guid WorkOrderId { get; set; }
        public WorkOrder WorkOrder { get; set; } = null!;
        public int GoodQty { get; set; }
        public int RejectQty { get; set; }
        public DateTime InputTime { get; set; } = DateTime.UtcNow;
        public string OperatorId { get; set; } = string.Empty;
    }
}

// Entities/QcResult.cs (Input QC)
namespace MES.Core.Entities
{
    public class QcResult
    {
        public Guid Id { get; set; }
        public Guid WorkOrderId { get; set; }
        public string DefectCode { get; set; } = string.Empty;
        public int DefectQty { get; set; }
        public string Notes { get; set; } = string.Empty;
        public DateTime InputTime { get; set; } = DateTime.UtcNow;
    }
}

// Entities/Downtime.cs (Input Engineering)
namespace MES.Core.Entities
{
    public class Downtime
    {
        public Guid Id { get; set; }
        public Guid LineId { get; set; }
        public DateTime StartTime { get; set; }
        public DateTime? EndTime { get; set; }
        public string RootCause { get; set; } = string.Empty;
        public string ActionTaken { get; set; } = string.Empty;
    }
}

// Entities/ActivityLog.cs (Audit Trail)
namespace MES.Core.Entities
{
    public class ActivityLog
    {
        public Guid Id { get; set; }
        public string UserName { get; set; } = string.Empty;
        public string Action { get; set; } = string.Empty;
        public string Detail { get; set; } = string.Empty;
        public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    }
}
```

---

## ⚙️ 4. INFRASTRUCTURE LAYER (MES.INFRASTRUCTURE)

### 4.1 DbContext (`Data/MesDbContext.cs`)
```csharp
using MES.Core.Entities;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace MES.Infrastructure.Data
{
    public class MesDbContext : IdentityDbContext<IdentityUser>
    {
        public MesDbContext(DbContextOptions<MesDbContext> options) : base(options) { }

        public DbSet<ProductionLine> ProductionLines => Set<ProductionLine>();
        public DbSet<Product> Products => Set<Product>();
        public DbSet<DefectCode> DefectCodes => Set<DefectCode>();
        public DbSet<DowntimeCode> DowntimeCodes => Set<DowntimeCode>();
        public DbSet<WorkOrder> WorkOrders => Set<WorkOrder>();
        public DbSet<ProductionResult> ProductionResults => Set<ProductionResult>();
        public DbSet<QcResult> QcResults => Set<QcResult>();
        public DbSet<Downtime> Downtimes => Set<Downtime>();
        public DbSet<ActivityLog> ActivityLogs => Set<ActivityLog>();

        protected override void OnModelCreating(ModelBuilder builder)
        {
            base.OnModelCreating(builder);
            builder.Entity<ProductionLine>().HasData(
                new ProductionLine { Id = Guid.Parse("11111111-1111-1111-1111-111111111111"), LineCode = "DIJ-01", LineName = "Direct Ink Jet Line 1", Status = Core.Enums.LineStatus.Idle }
            );
            builder.Entity<Product>().HasData(
                new Product { Id = Guid.Parse("22222222-2222-2222-2222-222222222222"), ProductCode = "PRD-001", ProductName = "Action Figure Part A", IdealCycleTimeSeconds = 5.0 }
            );
        }
    }
}
```

---

## 🧠 5. APPLICATION LAYER (MES.APPLICATION)

### 5.1 OEE Engine & Service (`Services/ProductionService.cs`)
```csharp
using MES.Core.Entities;
using MES.Core.Enums;
using MES.Infrastructure.Data;
using Microsoft.AspNetCore.SignalR;
using Microsoft.EntityFrameworkCore;

namespace MES.Application.Services
{
    public class LineDashboardDto
    {
        public Guid LineId { get; set; }
        public string LineCode { get; set; } = string.Empty;
        public double OEE { get; set; }
        public double Availability { get; set; }
        public double Performance { get; set; }
        public double Quality { get; set; }
        public int TotalGood { get; set; }
        public int TotalTarget { get; set; }
    }

    public class ProductionHub : Hub { } // SignalR Hub placeholder

    public class ProductionService
    {
        private readonly MesDbContext _db;
        private readonly IHubContext<ProductionHub> _hubContext;

        public ProductionService(MesDbContext db, IHubContext<ProductionHub> hubContext)
        {
            _db = db;
            _hubContext = hubContext;
        }

        public async Task<List<WorkOrder>> GetActiveWorkOrders()
        {
            return await _db.WorkOrders
                .Include(x => x.Product)
                .Include(x => x.Line)
                .Where(x => x.Status == WOStatus.Released || x.Status == WOStatus.Running)
                .OrderByDescending(x => x.WONumber)
                .ToListAsync();
        }

        public async Task InputProduction(Guid woId, int goodQty, int rejectQty, string operatorId)
        {
            var wo = await _db.WorkOrders.Include(x => x.Line).Include(x => x.Product).FirstOrDefaultAsync(x => x.Id == woId);
            if (wo == null || wo.Status == WOStatus.Draft) throw new Exception("WO tidak valid!");

            if (wo.Status == WOStatus.Released) wo.Status = WOStatus.Running;

            _db.ProductionResults.Add(new ProductionResult { WorkOrderId = woId, GoodQty = goodQty, RejectQty = rejectQty, OperatorId = operatorId });
            
            wo.GoodQty += goodQty;
            wo.RejectQty += rejectQty;
            if (wo.GoodQty >= wo.TargetQty) wo.Status = WOStatus.Complete;

            _db.ActivityLogs.Add(new ActivityLog { UserName = operatorId, Action = "INPUT_PRODUCTION", Detail = $"Input to WO:{woId}. Good:{goodQty}, Reject:{rejectQty}" });

            await _db.SaveChangesAsync();
            await BroadcastOEEUpdate(wo);
        }

        public async Task InputQcDefect(Guid woId, string defectCode, int defectQty, string notes)
        {
            var wo = await _db.WorkOrders.Include(x => x.Line).Include(x => x.Product).FirstOrDefaultAsync(x => x.Id == woId);
            if (wo == null) throw new Exception("WO tidak ditemukan!");

            _db.QcResults.Add(new QcResult { WorkOrderId = woId, DefectCode = defectCode, DefectQty = defectQty, Notes = notes });
            wo.RejectQty += defectQty;

            await _db.SaveChangesAsync();
            await BroadcastOEEUpdate(wo);
        }

        public async Task InputDowntime(Guid lineId, DateTime startTime, DateTime endTime, string rootCause, string engId)
        {
            _db.Downtimes.Add(new Downtime { LineId = lineId, StartTime = startTime, EndTime = endTime, RootCause = rootCause });
            
            var runningWO = await _db.WorkOrders.Include(x => x.Line).Include(x => x.Product)
                .FirstOrDefaultAsync(x => x.LineId == lineId && x.Status == WOStatus.Running);

            await _db.SaveChangesAsync();
            
            if (runningWO != null) await BroadcastOEEUpdate(runningWO);
        }

        private async Task BroadcastOEEUpdate(WorkOrder wo)
        {
            var dashboardData = CalculateOEEForLine(wo);
            await _hubContext.Clients.All.SendAsync("ReceiveProductionUpdate", dashboardData);
        }

        private LineDashboardDto CalculateOEEForLine(WorkOrder wo)
        {
            double plannedTime = 28800; // 8 jam shift
            var today = DateTime.UtcNow.Date;
            var totalDowntimeSeconds = _db.Downtimes
                .Where(d => d.LineId == wo.LineId && d.StartTime.Date == today && d.EndTime != null)
                .Sum(d => (d.EndTime.Value - d.StartTime).TotalSeconds);

            double operatingTime = plannedTime - totalDowntimeSeconds;
            double availability = plannedTime > 0 ? operatingTime / plannedTime : 0;

            double idealCycleTime = wo.Product.IdealCycleTimeSeconds;
            double totalOutput = wo.GoodQty + wo.RejectQty;
            double performance = operatingTime > 0 ? (idealCycleTime * totalOutput) / operatingTime : 0;

            double quality = totalOutput > 0 ? (double)wo.GoodQty / totalOutput : 0;
            double oee = availability * performance * quality;

            return new LineDashboardDto
            {
                LineId = wo.LineId, LineCode = wo.Line.LineCode,
                OEE = Math.Round(oee * 100, 1), Availability = Math.Round(availability * 100, 1),
                Performance = Math.Round(performance * 100, 1), Quality = Math.Round(quality * 100, 1),
                TotalGood = wo.GoodQty, TotalTarget = wo.TargetQty
            };
        }
    }
}
```

### 5.2 Reporting Service (`Services/ReportService.cs`)
```csharp
using ClosedXML.Excel;
using MES.Infrastructure.Data;
using Microsoft.EntityFrameworkCore;

namespace MES.Application.Services
{
    public class ReportService
    {
        private readonly MesDbContext _db;
        public ReportService(MesDbContext db) { _db = db; }

        public async Task<byte[]> GenerateDailyProductionReport(DateTime date)
        {
            using var workbook = new XLWorkbook();
            var worksheet = workbook.Worksheets.Add("Daily Production");
            worksheet.Cell(1, 1).Value = "WO Number";
            worksheet.Cell(1, 2).Value = "Product";
            worksheet.Cell(1, 3).Value = "Line";
            worksheet.Cell(1, 4).Value = "Target";
            worksheet.Cell(1, 5).Value = "Good";
            worksheet.Cell(1, 6).Value = "Reject";
            
            var data = await _db.WorkOrders.Include(x => x.Product).Include(x => x.Line)
                .Where(x => x.DueDate.Date == date.Date).ToListAsync();

            int row = 2;
            foreach (var item in data)
            {
                worksheet.Cell(row, 1).Value = item.WONumber;
                worksheet.Cell(row, 2).Value = item.Product.ProductName;
                worksheet.Cell(row, 3).Value = item.Line.LineCode;
                worksheet.Cell(row, 4).Value = item.TargetQty;
                worksheet.Cell(row, 5).Value = item.GoodQty;
                worksheet.Cell(row, 6).Value = item.RejectQty;
                row++;
            }

            using var stream = new MemoryStream();
            workbook.SaveAs(stream);
            return stream.ToArray();
        }
    }
}
```

---

## 🖥️ 6. SERVER LAYER (UI & WIRING - MES.SERVER)

### 6.1 Konfigurasi `Program.cs`
```csharp
using MES.Application.Services;
using MES.Infrastructure.Data;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Database & Identity
builder.Services.AddDbContext<MesDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddDefaultIdentity<IdentityUser>(options => options.SignIn.RequireConfirmedAccount = false)
    .AddRoles<IdentityRole>()
    .AddEntityFrameworkStores<MesDbContext>();

builder.Services.ConfigureApplicationCookie(options =>
{
    options.ExpireTimeSpan = TimeSpan.FromMinutes(15);
    options.SlidingExpiration = true;
});

// SignalR & Application Services
builder.Services.AddSignalR();
builder.Services.AddMudServices();
builder.Services.AddScoped<ProductionService>();
builder.Services.AddScoped<ReportService>();

builder.Services.AddRazorPages();
builder.Services.AddServerSideBlazor();

var app = builder.Build();

// Seed Data
using (var scope = app.Services.CreateScope())
{
    var roleManager = scope.ServiceProvider.GetRequiredService<RoleManager<IdentityRole>>();
    var userManager = scope.ServiceProvider.GetRequiredService<UserManager<IdentityUser>>();
    string[] roles = { "Admin", "Supervisor", "Leader", "Operator", "QC", "Engineering" };
    foreach (var role in roles)
    {
        if (!await roleManager.RoleExistsAsync(role))
            await roleManager.CreateAsync(new IdentityRole(role));
    }
    if (await userManager.FindByEmailAsync("admin@mes.com") == null)
    {
        var user = new IdentityUser { UserName = "admin@mes.com", Email = "admin@mes.com" };
        await userManager.CreateAsync(user, "Password123!");
        await userManager.AddToRoleAsync(user, "Admin");
    }
}

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

app.MapBlazorHub();
app.MapHub<ProductionHub>("/productionHub");
app.MapFallbackToPage("/_Host");

app.Run();
```

### 6.2 `App.razor`
```razor
<MudThemeProvider IsDarkMode="true" />
<MudDialogProvider />
<MudSnackbarProvider />
<Router AppAssembly="@typeof(App).Assembly">
    <Found Context="routeData">
        <AuthorizeRouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)" />
        <FocusOnNavigate RouteData="@routeData" Selector="h1" />
    </Found>
</Router>
```

### 6.3 Layout Utama (`Shared/MainLayout.razor`)
```razor
@inherits LayoutComponentBase
@using Microsoft.AspNetCore.Components.Authorization

<MudLayout>
    <MudDrawer Elevation="2" Color="Color.Dark" Open="true" Fixed="true">
        <MudDrawerHeader>
            <MudText Typo="Typo.h6" Color="Color.Primary">MES - DIJ</MudText>
        </MudDrawerHeader>
        <NavMenu />
    </MudDrawer>
    
    <MudMainContent Class="pa-6" Style="background-color: #121212; color: #ffffff; min-height: 100vh;">
        @Body
    </MudMainContent>
</MudLayout>
```

### 6.4 Navigasi (`Shared/NavMenu.razor`)
```razor
<MudNavMenu>
    <MudNavLink Href="/live-dashboard" Icon="@Icons.Material.Filled.Dashboard">Live Dashboard</MudNavLink>
    
    <AuthorizeView Roles="Supervisor,Admin">
        <Authorized>
            <MudNavLink Href="/supervisor/workorders" Icon="@Icons.Material.Filled.Assignment">Manage WO</MudNavLink>
        </Authorized>
    </AuthorizeView>

    <AuthorizeView Roles="Operator">
        <Authorized>
            <MudNavLink Href="/operator/input" Icon="@Icons.Material.Filled.Input">Input Produksi</MudNavLink>
        </Authorized>
    </AuthorizeView>

    <AuthorizeView Roles="QC">
        <Authorized>
            <MudNavLink Href="/qc/input" Icon="@Icons.Material.Filled.BugReport">Input Defect</MudNavLink>
        </Authorized>
    </AuthorizeView>

    <AuthorizeView Roles="Engineering">
        <Authorized>
            <MudNavLink Href="/engineering/downtime" Icon="@Icons.Material.Filled.Build">Input Downtime</MudNavLink>
        </Authorized>
    </AuthorizeView>

    <AuthorizeView Roles="Admin">
        <Authorized>
            <MudNavGroup Icon="@Icons.Material.Filled.Folder" Title="Master Data">
                <MudNavLink Href="/admin/lines" Icon="@Icons.Material.Filled.ViewList">Lines</MudNavLink>
                <MudNavLink Href="/admin/products" Icon="@Icons.Material.Filled.Category">Products</MudNavLink>
                <MudNavLink Href="/admin/defects" Icon="@Icons.Material.Filled.Warning">Defect Codes</MudNavLink>
                <MudNavLink Href="/admin/downtimes" Icon="@Icons.Material.Filled.Timer">Downtime Codes</MudNavLink>
            </MudNavGroup>
        </Authorized>
    </AuthorizeView>

    <MudNavLink Href="/reports" Icon="@Icons.Material.Filled.Assessment">Reports</MudNavLink>
</MudNavMenu>
```

### 6.5 Halaman Kunci: Live Dashboard (`Pages/LiveDashboard.razor`)
```razor
@page "/live-dashboard"
@using MES.Application.Services
@using MES.Infrastructure.Data
@using Microsoft.EntityFrameworkCore
@inject NavigationManager NavigationManager
@inject MesDbContext Db
@implements IAsyncDisposable

<MudText Typo="Typo.h3" Color="Color.Primary" Class="mb-6">LIVE MONITORING - AREA DIRECT INK JET</MudText>

<MudGrid Spacing="4">
    @foreach (var line in DashboardData.Values)
    {
        <MudItem xs="12" md="6" lg="4">
            <MudPaper Class="pa-6" Style="background-color: #1e1e1e; border-top: 8px solid @GetOeeColor(line.OEE);">
                <MudText Typo="Typo.h5" Color="Color.Surface">@line.LineCode</MudText>
                <MudText Typo="Typo.h1" Style="color: @GetOeeColor(line.OEE); font-weight: 900;">@line.OEE %</MudText>
                
                <MudStack Row="true" Spacing="6" Class="mt-4 mb-6">
                    <div class="text-center"><MudText Typo="Typo.h6" Color="Color.Info">A: @line.Availability %</MudText></div>
                    <div class="text-center"><MudText Typo="Typo.h6" Color="Color.Warning">P: @line.Performance %</MudText></div>
                    <div class="text-center"><MudText Typo="Typo.h6" Color="Color.Success">Q: @line.Quality %</MudText></div>
                </MudStack>

                <MudDivider Class="mb-4" />
                <MudText Typo="Typo.body1" Color="Color.Surface">Output: <b>@line.TotalGood</b> / Target: <b>@line.TotalTarget</b></MudText>
                <MudProgressLinear Color="Color.Success" Size="Size.Large" Value="@(line.TotalTarget > 0 ? Math.Round((double)line.TotalGood / line.TotalTarget * 100) : 0)" Class="mt-2" />
            </MudPaper>
        </MudItem>
    }
</MudGrid>

@code {
    private Dictionary<Guid, LineDashboardDto> DashboardData = new();
    private HubConnection? hubConnection;

    protected override async Task OnInitializedAsync()
    {
        var lines = await Db.ProductionLines.ToListAsync();
        foreach (var line in lines)
        {
            DashboardData[line.Id] = new LineDashboardDto { LineId = line.Id, LineCode = line.LineCode, OEE = 0, Availability = 0, Performance = 0, Quality = 0, TotalGood = 0, TotalTarget = 0 };
        }

        hubConnection = new HubConnectionBuilder().WithUrl(NavigationManager.ToAbsoluteUri("/productionHub")).Build();
        hubConnection.On<LineDashboardDto>("ReceiveProductionUpdate", (update) => { DashboardData[update.LineId] = update; StateHasChanged(); });
        await hubConnection.StartAsync();
    }

    private string GetOeeColor(double oee) => oee switch { >= 85 => "#4caf50", >= 60 => "#ff9800", _ => "#f44336" };

    public async ValueTask DisposeAsync() { if (hubConnection is not null) await hubConnection.DisposeAsync(); }
}
```

*(Halaman Input Operator, Input Downtime, Input QC, dan Master Data lainnya mengikuti pola komponen MudBlazor yang telah diajarkan di Tahap 18-32 sebelumnya).*

### 6.6 JS Helper Excel Download (`wwwroot/js/interop.js`)
```javascript
window.downloadFile = function(filename, base64Data) {
    const link = document.createElement('a');
    link.download = filename;
    link.href = 'data:application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;base64,' + base64Data;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
}
```

---

## 🚀 7. DEPLOYMENT CHECKLIST

1. **Migrasi Database:**
   ```bash
   cd src/MES.Infrastructure
   dotnet ef migrations add InitialCreate --startup-project ../MES.Server
   dotnet ef database update --startup-project ../MES.Server
   ```
2. **Publish ke Folder:** Klik kanan `MES.Server` -> Publish -> Folder.
3. **Setup IIS Windows Server:**
   - Install *Hosting Bundle .NET 8*.
   - Aktifkan fitur *WebSocket Protocol* di IIS.
   - Buat Website, arahkan ke folder publish.
   - Set Application Pool ke **No Managed Code**.
4. **Akses Lantai Produksi:** Buka browser di PC Lantai -> `http://[IP_SERVER]:8080/live-dashboard`.

--- 
*Blueprint ini adalah panduan komprehensif Anda. Selamat membangun!*
