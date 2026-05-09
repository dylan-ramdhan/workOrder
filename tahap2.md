
**Fase Selanjutnya (Fase 1 Lanjutan):** Kita akan membangun fitur operasional nyata agar sistem ini siap ditaruh di lantai produksi. Kita akan fokus pada:
1. Master Data CRUD (Supaya ga input manual di SQL lagi).
2. Alur Lifecyle Work Order (Supervisor Release WO).
3. Integrasi Modul Downtime & QC ke OEE Engine (Supaya OEE ga asal 100%).

Mari kita lanjutkan!

---

### TAHAP 9: MEMBANGUN MASTER DATA (MANAJEMEN DATA LINES & PRODUCTS)

Sistem MES tidak boleh bergantung pada input manual ke database. Kita butuh halaman Admin untuk mengelola master data.

Buat file `MasterLines.razor` di `src/MES.Server/Pages/Admin/`:

```razor
@page "/admin/lines"
@using MES.Core.Entities
@using MES.Infrastructure.Data
@inject MesDbContext Db

<PageTitle>Master Production Lines</PageTitle>

<h3>Master Production Lines</h3>

<button class="btn btn-primary mb-3" @onclick="() => OpenModal(new ProductionLine())">+ Add New Line</button>

<table class="table table-striped table-bordered">
    <thead class="table-dark">
        <tr>
            <th>Line Code</th>
            <th>Line Name</th>
            <th>Status</th>
            <th>Action</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var line in Lines)
        {
            <tr>
                <td>@line.LineCode</td>
                <td>@line.LineName</td>
                <td>@line.Status</td>
                <td>
                    <button class="btn btn-sm btn-warning" @onclick="() => OpenModal(line)">Edit</button>
                </td>
            </tr>
        }
    </tbody>
</table>

<!-- Modal Pop Up untuk Add/Edit -->
@if (ShowModal)
{
    <div class="modal fade show d-block" style="background-color:rgba(0,0,0,0.5)">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title">@(EditingLine.Id == Guid.Empty ? "Add Line" : "Edit Line")</h5>
                </div>
                <div class="modal-body">
                    <div class="form-group mb-2">
                        <label>Line Code</label>
                        <input @bind="EditingLine.LineCode" class="form-control" />
                    </div>
                    <div class="form-group mb-2">
                        <label>Line Name</label>
                        <input @bind="EditingLine.LineName" class="form-control" />
                    </div>
                </div>
                <div class="modal-footer">
                    <button class="btn btn-secondary" @onclick="CloseModal">Cancel</button>
                    <button class="btn btn-success" @onclick="SaveLine">Save</button>
                </div>
            </div>
        </div>
    </div>
}

@code {
    private List<ProductionLine> Lines = new();
    private bool ShowModal = false;
    private ProductionLine EditingLine = new();

    protected override async Task OnInitializedAsync()
    {
        Lines = await Db.ProductionLines.ToListAsync();
    }

    private void OpenModal(ProductionLine line)
    {
        // Kalau new, buat ID baru. Kalau edit, copy datanya
        EditingLine = line.Id == Guid.Empty ? new ProductionLine() : line;
        ShowModal = true;
    }

    private void CloseModal() => ShowModal = false;

    private async Task SaveLine()
    {
        if (EditingLine.Id == Guid.Empty)
        {
            EditingLine.Id = Guid.NewGuid();
            Db.ProductionLines.Add(EditingLine);
        }
        else
        {
            Db.ProductionLines.Update(EditingLine);
        }

        await Db.SaveChangesAsync();
        Lines = await Db.ProductionLines.ToListAsync(); // Refresh tabel
        CloseModal();
    }
}
```
*Lakukan hal yang sama untuk `MasterProducts.razor` (untuk menginput Product Code, Name, dan **Ideal Cycle Time**).*

---

### TAHAP 10: WORK ORDER LIFECYCLE (SUPERVISOR FLOW)

Work Order (WO) memiliki "State Machine". Dia tidak bisa lompat dari Draft langsung Complete.
Alurnya: **Draft -> Released -> Running -> Complete**.

Buat file `WorkOrderManagement.razor` di `src/MES.Server/Pages/Supervisor/`:

```razor
@page "/supervisor/workorders"
@using MES.Core.Entities
@using MES.Core.Enums
@using MES.Infrastructure.Data
@inject MesDbContext Db

<h3>Work Order Management</h3>

<!-- Form Create WO (Sederhana) -->
<div class="card mb-4 p-3 bg-light">
    <h5>Create New Work Order</h5>
    <div class="row">
        <div class="col-md-3">
            <label>Product</label>
            <select @bind="NewWO.ProductId" class="form-control">
                @foreach(var p in Products)
                {
                    <option value="@p.Id">@p.ProductName</option>
                }
            </select>
        </div>
        <div class="col-md-3">
            <label>Line</label>
            <select @bind="NewWO.LineId" class="form-control">
                @foreach(var l in Lines)
                {
                    <option value="@l.Id">@l.LineCode</option>
                }
            </select>
        </div>
        <div class="col-md-2">
            <label>Target Qty</label>
            <input @bind="NewWO.TargetQty" type="number" class="form-control" />
        </div>
        <div class="col-md-4 d-flex align-items-end">
            <button class="btn btn-primary" @onclick="CreateWO">Create Draft</button>
        </div>
    </div>
</div>

<!-- Tabel Daftar WO -->
<table class="table table-bordered">
    <thead class="table-dark">
        <tr>
            <th>WO Number</th>
            <th>Product</th>
            <th>Line</th>
            <th>Target</th>
            <th>Status</th>
            <th>Action</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var wo in WorkOrders.OrderByDescending(x => x.DueDate))
        {
            <tr>
                <td>@wo.WONumber</td>
                <td>@wo.Product?.ProductName</td>
                <td>@wo.Line?.LineCode</td>
                <td>@wo.TargetQty</td>
                <td><span class="badge bg-@(wo.Status == WOStatus.Running ? "success" : wo.Status == WOStatus.Released ? "warning" : "secondary")">@wo.Status</span></td>
                <td>
                    @if (wo.Status == WOStatus.Draft)
                    {
                        <button class="btn btn-sm btn-warning" @onclick="() => ReleaseWO(wo.Id)">Release</button>
                    }
                </td>
            </tr>
        }
    </tbody>
</table>

@code {
    private List<WorkOrder> WorkOrders = new();
    private List<Product> Products = new();
    private List<ProductionLine> Lines = new();
    private WorkOrder NewWO = new();

    protected override async Task OnInitializedAsync()
    {
        Products = await Db.Products.ToListAsync();
        Lines = await Db.ProductionLines.ToListAsync();
        WorkOrders = await Db.WorkOrders.Include(x => x.Product).Include(x => x.Line).ToListAsync();
        
        NewWO.DueDate = DateTime.Today;
    }

    private async Task CreateWO()
    {
        NewWO.Id = Guid.NewGuid();
        NewWO.WONumber = $"WO-{DateTime.Now:yyyyMMddHHmmss}";
        NewWO.Status = WOStatus.Draft;
        NewWO.GoodQty = 0;
        NewWO.RejectQty = 0;
        
        Db.WorkOrders.Add(NewWO);
        await Db.SaveChangesAsync();
        
        NewWO = new(); // Reset form
        WorkOrders = await Db.WorkOrders.Include(x => x.Product).Include(x => x.Line).ToListAsync();
    }

    private async Task ReleaseWO(Guid woId)
    {
        var wo = await Db.WorkOrders.FindAsync(woId);
        if (wo != null)
        {
            wo.Status = WOStatus.Released;
            await Db.SaveChangesAsync();
            WorkOrders = await Db.WorkOrders.Include(x => x.Product).Include(x => x.Line).ToListAsync();
        }
    }
}
```

---

### TAHAP 11: INTEGRASI DOWNTIME & QC KE OEE ENGINE (OTAK YANG SESUNGGUHNYA)

Di Fase sebelumnya, saya hard-code `downtime = 0`. Sekarang kita buat sistemnya membaca *real downtime* dari database. 

Kita perlu mengupdate `ProductionService.cs` di `src/MES.Application/Services/`.

**1. Tambahkan Entity Downtime (di MES.Core)**
```csharp
public class Downtime
{
    public Guid Id { get; set; }
    public Guid LineId { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime? EndTime { get; set; } // Nullable karena bisa masih berjalan
    public string RootCause { get; set; }
}
```
*(Jangan lupa tambahkan `public DbSet<Downtime> Downtimes => Set<Downtime>();` di MesDbContext, dan jalankan `dotnet ef migrations add AddDowntimeTable` lalu `update`).*

**2. Update OEE Engine di ProductionService.cs**

Ganti method `CalculateOEEForLine` yang lama dengan ini:

```csharp
private LineDashboardDto CalculateOEEForLine(WorkOrder wo)
{
    // --- AVAILABILITY (Hitung Downtime Aktual dari DB) ---
    // Asumsi shift 8 jam (28800 detik)
    double plannedTime = 28800; 
    
    // Ambil total downtime untuk Line ini hari ini (Yang sudah selesai / EndTime tidak null)
    var today = DateTime.UtcNow.Date;
    var totalDowntimeSeconds = _db.Downtimes
        .Where(d => d.LineId == wo.LineId && d.StartTime.Date == today && d.EndTime != null)
        .Sum(d => (d.EndTime.Value - d.StartTime).TotalSeconds);

    double operatingTime = plannedTime - totalDowntimeSeconds;
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
```

**3. Buat Service untuk Input Downtime**
Tambahkan method ini di `ProductionService.cs`:

```csharp
public async Task InputDowntime(Guid lineId, DateTime startTime, DateTime endTime, string rootCause, string engId)
{
    var downtime = new Downtime
    {
        Id = Guid.NewGuid(),
        LineId = lineId,
        StartTime = startTime,
        EndTime = endTime,
        RootCause = rootCause
    };

    _db.Downtimes.Add(downtime);
    await _db.SaveChangesAsync();

    // Cari WO yang sedang running di line tersebut untuk di-update dashboard
    var runningWO = await _db.WorkOrders
        .Include(x => x.Line)
        .Include(x => x.Product)
        .FirstOrDefaultAsync(x => x.LineId == lineId && x.Status == WOStatus.Running);

    if (runningWO != null)
    {
        // Hitung ulang OEE dan Broadcast karena Availability pasti berubah (turun)
        var dashboardData = CalculateOEEForLine(runningWO);
        await _hubContext.Clients.All.SendAsync("ReceiveProductionUpdate", dashboardData);
    }
}
```

*Penjelasan Konsep Junior:* Perhatikan betapa pintarnya arsitektur ini. Saat Engineering mencatat downtime, ia **tidak peduli tentang OEE**. Ia hanya menyimpan data ke tabel Downtime. Tapi di belakang layar, Service kita otomatis menghitung ulang OEE dan menembak SignalR ke Dashboard Leader. Leader akan melihat grafik Availability **turun secara realtime** saat Engineering klik Submit. Inilah yang disebut *Loose Coupling*.

---

### TAHAP 12: AUTHENTICATION & ROLE-BASED ACCESS (KEAMANAN)

Sistem MES tanpa keamanan adalah bencana. Operator tidak boleh bisa Release WO.

1. **Seed Roles & User Admin**
Di `src/MES.Server/Program.cs`, tambahkan logic seeding persis sebelum `app.Run();`:

```csharp
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

    // Seed Default Admin
    if (await userManager.FindByEmailAsync("admin@mes.com") == null)
    {
        var user = new IdentityUser { UserName = "admin@mes.com", Email = "admin@mes.com" };
        await userManager.CreateAsync(user, "Password123!");
        await userManager.AddToRoleAsync(user, "Admin");
    }
}
```

2. **Proteksi Halaman (Authorization)**
Buka file `WorkOrderManagement.razor` tadi, tambahkan baris ini paling atas:
```razor
@attribute [Authorize(Roles = "Supervisor,Admin")]
```
Sekarang, jika Operator mencoba akses halaman Supervisor, mereka akan ditendang ke halaman Login/Access Denied.

3. **Proteksi di Level UI (Hide Button)**
Buka `OperatorInput.razor`. Tombol submit hanya muncul untuk role Operator:
```razor
<AuthorizeView Roles="Operator">
    <Authorized>
        <button @onclick="SubmitProduction" class="btn btn-primary btn-lg">SUBMIT PRODUKSI</button>
    </Authorized>
</AuthorizeView>
```

---

### TAHAP 13: SHIFT MANAGEMENT (MENANGANI MIDNIGHT CROSSING)

Ini masalah klasik di pabrik: Shift Malam mulai jam 22:00, selesai jam 06:00 besoknya. Jika query pakai `DateTime.Date`, data jam 01:00 akan masuk ke hari besok, padahal itu produksi hari sebelumnya.

**Solusi Application Layer (di ProductionService):**

Buat method helper untuk menentukan "Tanggal Produksi" yang sebenarnya:

```csharp
private DateTime GetProductionDate(DateTime inputTime)
{
    // Asumsi: Shift 1 (06:00-14:00), Shift 2 (14:00-22:00), Shift 3 (22:00-06:00)
    // Jika jam input antara 00:00 - 06:00, maka tanggal produksinya adalah KEMARIN
    if (inputTime.Hour >= 0 && inputTime.Hour < 6)
    {
        return inputTime.Date.AddDays(-1);
    }
    return inputTime.Date;
}
```

Saat menyimpan `ProductionResult`, simpan juga `ProductionDate = GetProductionDate(DateTime.UtcNow)`. Saat query Shift Report, gunakan `ProductionDate` ini, bukan `Timestamp.Date`.

---

### TAHAP 14: REPORTING (MENGHIDUPKAN DATA HISTORIS)

Pabrik butuh laporan untuk evaluasi performa. Kita butuh export ke Excel. Install package di `MES.Server`:
```bash
dotnet add package ClosedXML
```

Buat `ReportService.cs` di `MES.Application`:
```csharp
using ClosedXML.Excel;
using MES.Infrastructure.Data;
using Microsoft.EntityFrameworkCore;

public class ReportService
{
    private readonly MesDbContext _db;

    public ReportService(MesDbContext db)
    {
        _db = db;
    }

    public async Task<byte[]> GenerateDailyProductionReport(DateTime date)
    {
        using var workbook = new XLWorkbook();
        var worksheet = workbook.Worksheets.Add("Daily Production");

        // Header
        worksheet.Cell(1, 1).Value = "WO Number";
        worksheet.Cell(1, 2).Value = "Product";
        worksheet.Cell(1, 3).Value = "Line";
        worksheet.Cell(1, 4).Value = "Target";
        worksheet.Cell(1, 5).Value = "Good";
        worksheet.Cell(1, 6).Value = "Reject";
        worksheet.Cell(1, 7).Value = "OEE %";

        // Data
        var data = await _db.WorkOrders
            .Include(x => x.Product)
            .Include(x => x.Line)
            .Where(x => x.DueDate.Date == date.Date)
            .ToListAsync();

        int row = 2;
        foreach (var item in data)
        {
            worksheet.Cell(row, 1).Value = item.WONumber;
            worksheet.Cell(row, 2).Value = item.Product.ProductName;
            worksheet.Cell(row, 3).Value = item.Line.LineCode;
            worksheet.Cell(row, 4).Value = item.TargetQty;
            worksheet.Cell(row, 5).Value = item.GoodQty;
            worksheet.Cell(row, 6).Value = item.RejectQty;
            // OEE hitung ulang atau ambil dari snapshot
            row++;
        }

        using var stream = new MemoryStream();
        workbook.SaveAs(stream);
        return stream.ToArray();
    }
}
```
Di halaman Blazor, kamu tinggal sediakan tombol "Download Excel", panggil service ini, dan return sebagai file download.

---

### CECKPOINT EVALUASI UNTUK JUNIOR

Sebelum kita lanjut ke tahap deployment dan advanced UI, pastikan sistemmu sekarang memiliki:
1. ✅ Halaman Login (bawaan Identity) & pembatasan akses Role.
2. ✅ Halaman Master Lines & Products (CRUD).
3. ✅ Supervisor bisa Create WO -> Release WO.
4. ✅ Operator bisa memilih WO yang Released -> Input Good/Reject -> WO jadi Running.
5. ✅ Engineering bisa Input Downtime -> Dashboard Availability otomatis turun.
6. ✅ Semua perubahan di atas langsung ter-reflesh ke Live Dashboard via SignalR.

Jika 6 hal di atas sudah berjalan mulus di laptopmu, sistem MES kamu sudah memasuki level **Production-Ready Prototype**.

Adakah step di atas yang membuatmu bingung atau mengalami error? Jika sudah siap, kita bisa masuk ke tahap Finishing: UI/UX Polish (Industrial Dashboard Styling) dan Deployment ke IIS Windows Server.
