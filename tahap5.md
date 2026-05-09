Mantap, Junior! Kalau sistem sudah aman sampai tahap ini, berarti **Core Engine MES-mu sudah hidup**. Engine OEE, SignalR, dan Alur Work Order sudah terbukti berjalan.

Namun, sebagai Senior, saya harus jujur: **Sistem yang "hidup" belum tentu "selesai"**. Kalau kita lihat kembali ke *Brief* awal, ada beberapa aktor utama yang masih "terdiam" dan belum bisa bekerja di lantai produksi:

1. **Engineering** belum bisa input Downtime (OEE Availability tidak akan pernah turun secara realistis).
2. **QC** belum bisa input Defect spesifik (OEE Quality hanya mengandalkan input Reject dari Operator, tidak teranalisis per jenis defect).
3. **Management** belum bisa unduh laporan (Reporting).
4. **Keamanan PC Lantai** belum diatur (Session Timeout).

Mari kita tutup **20% sisa kekurangan** ini agar sistemmu benar-benar *Production-Ready* dan siap di-*deploy* ke pabrik!

---

### TAHAP 23: MODULE DOWNTIME (ENGINEERING INPUT)

Downtime adalah pembunuh OEE Availability. Engineering harus bisa mencatat kapan mesin berhenti dan mengapa.

**1. Buat Halaman `EngineeringInput.razor` di `src/MES.Server/Pages/Engineering/`**

```razor
@page "/engineering/downtime"
@using MES.Core.Entities
@using MES.Application.Services
@using MES.Infrastructure.Data
@inject ProductionService ProdService
@inject MesDbContext Db

<PageTitle>Input Downtime</PageTitle>
<MudText Typo="Typo.h4" Class="mb-4">Downtime Input (Engineering)</MudText>

<MudPaper Class="pa-6" Style="background-color: #1e1e1e;">
    <MudSelect T="Guid" Label="Select Line" @bind-Value="SelectedLineId" Variant="Variant.Filled" Class="mb-4">
        @foreach(var line in Lines)
        {
            <MudSelectItem Value="@line.Id">@line.LineCode - @line.LineName</MudSelectItem>
        }
    </MudSelect>

    <MudDateTimePicker Label="Start Time" @bind-Value="StartTime" Variant="Variant.Filled" Class="mb-4" />
    <MudDateTimePicker Label="End Time" @bind-Value="EndTime" Variant="Variant.Filled" Class="mb-4" />

    <MudTextField Label="Root Cause" @bind-Value="RootCause" Variant="Variant.Filled" Class="mb-4" />
    <MudTextField Label="Action Taken" @bind-Value="ActionTaken" Variant="Variant.Filled" Class="mb-4" />

    <MudButton Variant="Variant.Filled" Color="Color.Error" Size="Size.Large" FullWidth="true" OnClick="SubmitDowntime">
        SUBMIT DOWNTIME
    </MudButton>
</MudPaper>

@code {
    private List<ProductionLine> Lines = new();
    private Guid SelectedLineId;
    private DateTime? StartTime = DateTime.Now;
    private DateTime? EndTime = DateTime.Now;
    private string RootCause = "";
    private string ActionTaken = "";

    protected override async Task OnInitializedAsync()
    {
        Lines = await Db.ProductionLines.ToListAsync();
    }

    private async Task SubmitDowntime()
    {
        if (SelectedLineId == Guid.Empty || StartTime == null || EndTime == null) return;

        // Panggil service yang sudah kita buat di Tahap 11 lalu
        await ProdService.InputDowntime(SelectedLineId, StartTime.Value, EndTime.Value, RootCause, "Eng1");
        
        RootCause = "";
        ActionTaken = "";
    }
}
```
*Sekarang, setiap Engineering input downtime, OEE Availability di dashboard akan otomatis turun dan terupdate realtime!*

---

### TAHAP 24: MODULE QC DEFECT (QUALITY CONTROL)

Operator hanya input "Reject Qty". Tapi QC perlu menganalisis **jenis defect-nya apa** (misal: Misprint, Blur, Color Mismatch). Ini vital untuk perbaikan proses.

**1. Tambahkan Entity QcResult (di MES.Core/Entities)**
```csharp
public class QcResult
{
    public Guid Id { get; set; }
    public Guid WorkOrderId { get; set; }
    public string DefectCode { get; set; } = string.Empty; // misal: DEF-01
    public int DefectQty { get; set; }
    public string Notes { get; set; } = string.Empty;
    public DateTime InputTime { get; set; } = DateTime.UtcNow;
}
```
*(Jangan lupa tambah `DbSet<QcResult>` di DbContext dan jalankan Migrasi: `dotnet ef migrations add AddQcResult` -> `update`).*

**2. Tambahkan Method di `ProductionService.cs`**
Logikanya: Saat QC input defect, reject qty di Work Order juga harus bertambah, lalu OEE Quality turun.

```csharp
public async Task InputQcDefect(Guid woId, string defectCode, int defectQty, string notes)
{
    var wo = await _db.WorkOrders.Include(x => x.Line).Include(x => x.Product).FirstOrDefaultAsync(x => x.Id == woId);
    if (wo == null) throw new Exception("WO tidak ditemukan!");

    // 1. Simpan detail QC
    _db.QcResults.Add(new QcResult
    {
        WorkOrderId = woId,
        DefectCode = defectCode,
        DefectQty = defectQty,
        Notes = notes
    });

    // 2. Update total reject di WO (ini yang bakal ngaruh ke OEE Quality)
    wo.RejectQty += defectQty;

    await _db.SaveChangesAsync();

    // 3. Hitung ulang OEE & Broadcast
    var dashboardData = CalculateOEEForLine(wo);
    await _hubContext.Clients.All.SendAsync("ReceiveProductionUpdate", dashboardData);
}
```

**3. Buat Halaman `QcInput.razor`** (Pola UI-nya sama persis seperti Operator Input, tapi memanggil `InputQcDefect`).

---

### TAHAP 25: REPORTING MODULE (EXPORT KE EXCEL)

Pabrik butuh laporan rekapitulasi. Kita akan gunakan **ClosedXML** yang sudah kita install sebelumnya untuk generate file Excel dan langsung di-download oleh browser.

**1. Buat `ReportService.cs` di `src/MES.Application/Services/`**

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
        worksheet.Cell(1, 7).Value = "Status";

        // Styling Header
        var range = worksheet.Range(1, 1, 1, 7);
        range.Style.Font.Bold = true;
        range.Style.Fill.BackgroundColor = XLColor.SteelBlue;

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
            worksheet.Cell(row, 7).Value = item.Status.ToString();
            row++;
        }

        using var stream = new MemoryStream();
        workbook.SaveAs(stream);
        return stream.ToArray();
    }
}
```

*Daftarkan `builder.Services.AddScoped<ReportService>();` di `Program.cs`.*

**2. Buat Tombol Download di Halaman Report (`Reports.razor`)**

```razor
@page "/reports"
@using MES.Application.Services
@inject ReportService ReportService

<MudText Typo="Typo.h4" Class="mb-4">Production Reports</MudText>

<MudPaper Class="pa-6" Style="background-color: #1e1e1e;">
    <MudDatePicker Label="Select Date" @bind-Date="SelectedDate" Variant="Variant.Filled" Class="mb-4" />
    
    <MudButton Variant="Variant.Filled" Color="Color.Success" OnClick="DownloadReport">
        Download Excel
    </MudButton>
</MudPaper>

@code {
    private DateTime? SelectedDate = DateTime.Today;

    private async Task DownloadReport()
    {
        if (SelectedDate == null) return;

        var fileBytes = await ReportService.GenerateDailyProductionReport(SelectedDate.Value);
        
        // Trigger download file di browser
        using var stream = new MemoryStream(fileBytes);
        await JSRuntime.InvokeVoidAsync("downloadFile", 
            $"Report_Produksi_{SelectedDate.Value:yyyyMMdd}.xlsx", 
            Convert.ToBase64String(stream.ToArray()));
    }
}
```

**3. Tambahkan Javascript Helper**
Karena Blazor Server berjalan di server, untuk push file ke browser kita butuh sedikit bantuan JS. Buka `wwwroot/index.html` (atau `_Host.cshtml`), tambahkan di tag `<head>`:
```html
<script>
    window.downloadFile = function(filename, base64Data) {
        const link = document.createElement('a');
        link.download = filename;
        link.href = 'data:application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;base64,' + base64Data;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    }
</script>
```

---

### TAHAP 26: SESSION TIMEOUT & SECURITY HARDENING

Di lantai produksi, Operator sering lupa logout. PC dibiarkan terbuka. Ini celah keamanan besar! Kita harus buat sistem auto-logout.

Buka `Program.cs` di `MES.Server`, atur Cookie Policy Identity:

```csharp
builder.Services.ConfigureApplicationCookie(options =>
{
    // Jika idle 15 menit, auto logout
    options.ExpireTimeSpan = TimeSpan.FromMinutes(15); 
    options.SlidingExpiration = true; // Reset timer jika user klik sesuatu
    options.LoginPath = "/Login"; // Arahkan ke halaman login bawaan
});
```

---

### TAHAP 27: FINAL REVIEW CHECKLIST

Kalau kamu sudah implementasi tahap 23-26, coba lakukan UAT (User Acceptance Test) akhir ini:

1. [ ] **Uji Downtime:** Buka Dashboard, catat OEE Availability. Lalu buka halaman Engineering, input downtime 60 menit. Kembali ke Dashboard. Apakah Availability turun otomatis?
2. [ ] **Uji QC Defect:** Buka Dashboard, catat OEE Quality. Lalu buka halaman QC, input defect 10 pcs. Kembali ke Dashboard. Apakah Quality turun dan angka Reject bertambah?
3. [ ] **Uji Export Excel:** Buka halaman Reports, pilih tanggal hari ini, klik Download. Apakah file Excel terdownload dan isinya sesuai data WO?
4. [ ] **Uji Session Timeout:** Login sebagai Operator, biarkan tab browser terbuka selama 15-20 menit tanpa klik apapun. Coba klik menu lain. Apakah sistem otomatis menendang ke halaman Login?
5. [ ] **Uji Multi-Line:** Buat WO untuk DIJ-01 dan DIJ-02. Input produksi DIJ-01. Pastikan di Dashboard, kartu DIJ-01 berubah, tapi kartu DIJ-02 tetap tidak berkedip/berubah.

**Jika 5 checklist di atas semua HIJAU, maka sistem MES Direct Ink Jet kamu sudah 100% memenuhi standar Enterprise Manufacturing!**

Tidak ada lagi tahapan tersembunyi. Ini adalah standar industri. Apakah ada error saat mengimplementasikan Downtime / QC / Excel di atas? Atau kamu siap deploy ke IIS?
