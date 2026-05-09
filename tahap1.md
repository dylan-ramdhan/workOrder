---

### BAGIAN 1: PERSIAPAN & STRUKTUR PROYEK

Buka Terminal/Command Prompt di laptopmu. Ketik perintah ini secara berurutan untuk membuat arsitektur Modular Monolith:

```bash
# 1. Buat solution
dotnet new sln -n MES.DirectInkJet

# 2. Buat proyek Core (Entity & Interface)
dotnet new classlib -n MES.Core -o src/MES.Core

# 3. Buat proyek Infrastructure (Database & Repo)
dotnet new classlib -n MES.Infrastructure -o src/MES.Infrastructure

# 4. Buat proyek Application (Logic & OEE Engine)
dotnet new classlib -n MES.Application -o src/MES.Application

# 5. Buat proyek Server (Blazor UI & SignalR)
dotnet new blazorserver -n MES.Server -o src/MES.Server

# 6. Masukkan semua ke Solution
dotnet sln add src/MES.Core
dotnet sln add src/MES.Infrastructure
dotnet sln add src/MES.Application
dotnet sln add src/MES.Server

# 7. Atur Referensi antar proyek (SALAH SATU INI PENTING!)
dotnet add src/MES.Infrastructure reference src/MES.Core
dotnet add src/MES.Application reference src/MES.Core src/MES.Infrastructure
dotnet add src/MES.Server reference src/MES.Application
```

---

### BAGIAN 2: INSTALL NUGET PACKAGE

Jalankan perintah ini di terminal:
```bash
# Infrastructure butuh EF Core & Identity
dotnet add src/MES.Infrastructure package Microsoft.EntityFrameworkCore.SqlServer
dotnet add src/MES.Infrastructure package Microsoft.EntityFrameworkCore.Tools
dotnet add src/MES.Infrastructure package Microsoft.AspNetCore.Identity.EntityFrameworkCore

# Server butuh Charting UI untuk Dashboard
dotnet add src/MES.Server package Blazor-ApexCharts
```

---

### BAGIAN 3: CORE LAYER (Entity & Enum)

Buat file-file berikut di `src/MES.Core/Entities/`:

**1. Enums.cs**
```csharp
namespace MES.Core.Enums
{
    public enum WOStatus { Draft, Released, Running, Hold, Complete }
    public enum LineStatus { Idle, Running, Down }
}
```

**2. ProductionLine.cs**
```csharp
namespace MES.Core.Entities
{
    public class ProductionLine
    {
        public Guid Id { get; set; }
        public string LineCode { get; set; } = string.Empty; // DIJ-01
        public string LineName { get; set; } = string.Empty;
        public LineStatus Status { get; set; }
    }
}
```

**3. Product.cs**
```csharp
namespace MES.Core.Entities
{
    public class Product
    {
        public Guid Id { get; set; }
        public string ProductCode { get; set; } = string.Empty;
        public string ProductName { get; set; } = string.Empty;
        public double IdealCycleTimeSeconds { get; set; } // Waktu standar 1 pcs
    }
}
```

**4. WorkOrder.cs**
```csharp
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
```

**5. ProductionResult.cs** (Input dari Operator)
```csharp
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
```

---

### BAGIAN 4: INFRASTRUCTURE LAYER (Database Context)

Buat file `MesDbContext.cs` di `src/MES.Infrastructure/Data/`:

```csharp
using MES.Core.Entities;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace MES.Infrastructure.Data
{
    // Turunan dari IdentityDbContext agar builtin Auth & Roles
    public class MesDbContext : IdentityDbContext<IdentityUser>
    {
        public MesDbContext(DbContextOptions<MesDbContext> options) : base(options) { }

        public DbSet<ProductionLine> ProductionLines => Set<ProductionLine>();
        public DbSet<Product> Products => Set<Product>();
        public DbSet<WorkOrder> WorkOrders => Set<WorkOrder>();
        public DbSet<ProductionResult> ProductionResults => Set<ProductionResult>();

        protected override void OnModelCreating(ModelBuilder builder)
        {
            base.OnModelCreating(builder);
            // Seed Data Awal untuk testing
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

### BAGIAN 5: APPLICATION LAYER (OEE Engine & Service Logic)

Ini otaknya. Buat file `ProductionService.cs` di `src/MES.Application/Services/`:

```csharp
using MES.Core.Entities;
using MES.Core.Enums;
using MES.Infrastructure.Data;
using Microsoft.AspNetCore.SignalR;
using Microsoft.EntityFrameworkCore;

namespace MES.Application.Services
{
    // DTO untuk kirim ke Dashboard via SignalR
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

    public class ProductionService
    {
        private readonly MesDbContext _db;
        private readonly IHubContext<ProductionHub> _hubContext;

        public ProductionService(MesDbContext db, IHubContext<ProductionHub> hubContext)
        {
            _db = db;
            _hubContext = hubContext;
        }

        // 1. Supervisor Release WO
        public async Task ReleaseWorkOrder(Guid woId)
        {
            var wo = await _db.WorkOrders.FindAsync(woId);
            if (wo != null && wo.Status == WOStatus.Draft)
            {
                wo.Status = WOStatus.Released;
                await _db.SaveChangesAsync();
            }
        }

        // 2. Operator Input Produksi (Memicu kalkulasi & Realtime Update)
        public async Task InputProduction(Guid woId, int goodQty, int rejectQty, string operatorId)
        {
            var wo = await _db.WorkOrders.Include(x => x.Line).Include(x => x.Product).FirstOrDefaultAsync(x => x.Id == woId);
            if (wo == null || wo.Status != WOStatus.Released) throw new Exception("WO tidak valid!");

            // Update status jadi Running jika awalnya Released
            if(wo.Status == WOStatus.Released) wo.Status = WOStatus.Running;

            // Simpan result detail
            var result = new ProductionResult
            {
                WorkOrderId = woId,
                GoodQty = goodQty,
                RejectQty = rejectQty,
                OperatorId = operatorId,
                InputTime = DateTime.UtcNow
            };
            _db.ProductionResults.Add(result);

            // Update total di WO
            wo.GoodQty += goodQty;
            wo.RejectQty += rejectQty;

            if (wo.GoodQty >= wo.TargetQty) wo.Status = WOStatus.Complete;

            await _db.SaveChangesAsync();

            // HITUNG OEE REALTIME & BROADCAST
            var dashboardData = CalculateOEEForLine(wo);
            await _hubContext.Clients.All.SendAsync("ReceiveProductionUpdate", dashboardData);
        }

        // 3. OEE Calculation Engine
        private LineDashboardDto CalculateOEEForLine(WorkOrder wo)
        {
            // --- AVAILABILITY (Asumsi shift 8 jam = 28800 detik, Downtime 0 untuk simplifikasi fase awal) ---
            double plannedTime = 28800; 
            double downtime = 0; // Nanti diambil dari tabel Downtimes
            double operatingTime = plannedTime - downtime;
            double availability = plannedTime > 0 ? operatingTime / plannedTime : 0;

            // --- PERFORMANCE ---
            double idealCycleTime = wo.Product.IdealCycleTimeSeconds;
            double totalOutput = wo.GoodQty + wo.RejectQty;
            double performance = operatingTime > 0 ? (idealCycleTime * totalOutput) / operatingTime : 0;

            // --- QUALITY ---
            double quality = totalOutput > 0 ? (double)wo.GoodQty / totalOutput : 0;

            // --- OEE ---
            double oee = availability * performance * quality;

            return new LineDashboardDto
            {
                LineId = wo.LineId,
                LineCode = wo.Line.LineCode,
                OEE = Math.Round(oee * 100, 1),
                Availability = Math.Round(availability * 100, 1),
                Performance = Math.Round(performance * 100, 1),
                Quality = Math.Round(quality * 100, 1),
                TotalGood = wo.GoodQty,
                TotalTarget = wo.TargetQty
            };
        }
    }

    // SignalR Hub Definition (ditaruh di Application agar Service bisa refer)
    public class ProductionHub : Hub { }
}
```

---

### BAGIAN 6: SERVER LAYER (Blazor UI & Wiring)

#### 1. Konfigurasi `Program.cs` di `src/MES.Server/`
Tambahkan kode ini sebelum `var app = builder.Build();`:

```csharp
// Daftarkan Database & Identity
builder.Services.AddDbContext<MesDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddDefaultIdentity<IdentityUser>(options => options.SignIn.RequireConfirmedAccount = false)
    .AddRoles<IdentityRole>()
    .AddEntityFrameworkStores<MesDbContext>();

// Daftarkan SignalR
builder.Services.AddSignalR();

// Daftarkan Application Services
builder.Services.AddScoped<ProductionService>();
```
*Tambahkan ini setelah `app.Build();`:*
```csharp
app.MapHub<ProductionHub>("/productionHub");
```

#### 2. Connection String
Buka `src/MES.Server/appsettings.json`, tambah:
```json
"ConnectionStrings": {
  "DefaultConnection": "Server=localhost;Database=MES_DirectInkJet;Trusted_Connection=True;MultipleActiveResultSets=true;TrustServerCertificate=True"
}
```

#### 3. Buat Halaman Live Dashboard (`LiveDashboard.razor` di `src/MES.Server/Pages/`)
```razor
@page "/live-dashboard"
@using MES.Application.Services
@inject NavigationManager NavigationManager
@implements IAsyncDisposable

<MES.Server.Pages.Shared.DashboardLayout>
    <h3>LIVE MONITORING - AREA DIRECT INK JET</h3>
    
    <div class="d-flex flex-wrap gap-4 mt-4">
        @if (LineData != null)
        {
            <div class="card bg-dark text-white" style="width: 22rem; border: 2px solid @GetOeeColor(LineData.OEE)">
                <div class="card-header bg-primary"><h4>@LineData.LineCode</h4></div>
                <div class="card-body">
                    <h1 class="text-center display-4 @GetOeeColor(LineData.OEE)">@LineData.OEE %</h1>
                    <div class="row text-center mt-4">
                        <div class="col"><strong>A</strong><br />@LineData.Availability %</div>
                        <div class="col"><strong>P</strong><br />@LineData.Performance %</div>
                        <div class="col"><strong>Q</strong><br />@LineData.Quality %</div>
                    </div>
                    <hr />
                    <p>Output Good: <b>@LineData.TotalGood</b> / Target: <b>@LineData.TotalTarget</b></p>
                </div>
            </div>
        }
        else
        {
            <p>Menunggu data produksi masuk...</p>
        }
    </div>
</MES.Server.Pages.Shared.DashboardLayout>

@code {
    private LineDashboardDto? LineData;
    private HubConnection? hubConnection;

    protected override async Task OnInitializedAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(NavigationManager.ToAbsoluteUri("/productionHub"))
            .Build();

        // LISTEN SIGNALR
        hubConnection.On<LineDashboardDto>("ReceiveProductionUpdate", (update) =>
        {
            LineData = update;
            StateHasChanged(); // Force refresh UI
        });

        await hubConnection.StartAsync();
    }

    private string GetOeeColor(double oee) => oee switch
    {
        >= 85 => "text-success",
        >= 60 => "text-warning",
        _ => "text-danger"
    };

    public async ValueTask DisposeAsync()
    {
        if (hubConnection is not null) await hubConnection.DisposeAsync();
    }
}
```

#### 4. Buat Halaman Input Operator (`OperatorInput.razor` di `src/MES.Server/Pages/`)
```razor
@page "/operator/input"
@using MES.Core.Entities
@using MES.Application.Services
@inject ProductionService ProdService

<h3>Input Produksi (Operator)</h3>

<div class="row">
    <div class="col-md-6">
        <div class="form-group mb-3">
            <label>Pilih Work Order (Running)</label>
            <select @bind="SelectedWOId" class="form-control">
                <option value="">-- Pilih WO --</option>
                @foreach(var wo in WorkOrders)
                {
                    <option value="@wo.Id">@wo.WONumber - @wo.Product?.ProductName</option>
                }
            </select>
        </div>
        <div class="form-group mb-3">
            <label>Good Quantity</label>
            <input @bind="InputGoodQty" type="number" class="form-control" />
        </div>
        <div class="form-group mb-3">
            <label>Reject Quantity</label>
            <input @bind="InputRejectQty" type="number" class="form-control" />
        </div>
        <button @onclick="SubmitProduction" class="btn btn-primary btn-lg">SUBMIT PRODUKSI</button>
        
        @if (!string.IsNullOrEmpty(Message))
        {
            <div class="alert alert-success mt-3">@Message</div>
        }
    </div>
</div>

@code {
    private List<WorkOrder> WorkOrders = new();
    private Guid SelectedWOId;
    private int InputGoodQty;
    private int InputRejectQty;
    private string Message = "";

    protected override async Task OnInitializedAsync()
    {
        // Ambil WO yang statusnya Released atau Running
        // (Asumsi service punya method GetActiveWorkOrders, buat simple di service)
    }

    private async Task SubmitProduction()
    {
        if(SelectedWOId == Guid.Empty) return;

        try
        {
            await ProdService.InputProduction(SelectedWOId, InputGoodQty, InputRejectQty, "Operator1");
            Message = "Data berhasil disimpan & Dashboard terupdate!";
            InputGoodQty = 0;
            InputRejectQty = 0;
        }
        catch(Exception ex)
        {
            Message = $"Error: {ex.Message}";
        }
    }
}
```

---

### BAGIAN 7: EKSEKUSI DATABASE MIGRATION

Sekarang struktur kode sudah lengkap. Kita harus generate tabel ke SQL Server. Buka Terminal di root solution:

```bash
# Pindah ke folder Infrastructure
cd src/MES.Infrastructure

# Buat Migration pertama
dotnet ef migrations add InitialCreate --startup-project ../MES.Server

# Push ke Database SQL Server
dotnet ef database update --startup-project ../MES.Server
```
*Cek SQL Server Management Studio (SSMS). Database `MES_DirectInkJet` dan tabel-tabel harusnya sudah muncul beserta tabel Identity bawaan (AspNetUsers, dll).*

---

### BAGIAN 8: TESTING ALUR REALTIME (UJI COBA)

1. Jalankan proyek dari Visual Studio atau terminal (`dotnet run` di `src/MES.Server`).
2. Buka Browser (Chrome): `https://localhost:5001/live-dashboard`. (Biarkan terbuka).
3. Buka Tab Browser baru: `https://localhost:5001/operator/input`.
4. **PENTING:** Karena kita belum buat halaman CRUD WO, masukkan data WO awal manual via SSMS ke tabel `WorkOrders`:
   - Id: `NEWID()`
   - WONumber: `WO-20231001`
   - ProductId: `22222222-2222-2222-2222-222222222222` (Sesuai seed data)
   - LineId: `11111111-1111-1111-1111-111111111111` (Sesuai seed data)
   - TargetQty: `1000`
   - GoodQty: `0`
   - RejectQty: `0`
   - Status: `1` (Released)
5. Di halaman Operator Input, pilih WO-20231001, masukkan Good Qty: `100`, Reject: `5`, klik **SUBMIT**.
6. **PERHATIKAN TAB LIVE DASHBOARD:** Angka OEE, Quality, dan Output akan otomatis berubah dalam hitungan milidetik **TANPA ME-REFRESH HALAMAN**. Ini adalah keajaiban SignalR yang sudah kita bangun.

---

### KESIMPULAN & NEXT STEP UNTUK JUNIOR

Sekarang kamu sudah memegang *core engine* yang hidup. Sistem ini membuktikan arsitektur Blazor + SQL Server + SignalR berjalan sempurna.

**Apa yang harus kamu kerjakan selanjutnya (mengikuti pola yang persis sama)?**
1. **Membuat CRUD Master Data:** Buat halaman Razor untuk input Line, Product, dan Shift. (Gunakan Entity Framework standar `db.Add`, `db.SaveChanges`).
2. **Modul Work Order Release:** Buat halaman untuk Supervisor mendaftarkan WO baru (Status Draft), lalu tombol "Release" yang mengubah status jadi 1 (Released).
3. **Modul Downtime (Engineering):** Mirip seperti `ProductionResult`, buat tabel/inputan Downtime. Saat disubmit, kurangi `OperatingTime` di OEE Engine, lalu broadcast SignalR, maka `Availability` di dashboard akan otomatis turun.
4. **Modul QC Defect:** Mirip seperti Downtime, tapi mempengaruhi rasio `Quality`.

Ingat, Senior Engineer tidak menghafal kode, tapi menguasai **pola alur data**. Pola alur data di sistem MES ini adalah:
**UI Input -> Service (Logic + OEE Calc) -> Save DB -> Trigger SignalR -> Dashboard Auto Render.**

Jika kamu pahami pola di atas, kamu bisa menambahkan 100 modul dan sistem tidak akan collapse. Coba jalankan langkah 1-8 di atas, jika ada error spesifik, beritahu saya!
