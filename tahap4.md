**Betul sekali, Junior! Ini 100% Web Application.** 

Biar Senior jelaskan arsitekturnya agar kamu paham bedanya:
1. **Desktop App** (seperti WinForms/WPF) di-install di tiap PC. Kalau ada update, kamu harus install ulang di 10 PC di lantai produksi. Repot!
2. **Blazor Server (Web App)** yang kita bangun ini, **kode C# dan Database-nya hanya berjalan di 1 komputer Server** (misal: PC Server di ruang IT). PC Operator/Leader di lantai produksi **hanya butuh Google Chrome/Edge**. Mereka cukup mengetikkan IP Address (contoh: `http://192.168.1.50:8080`), dan sistemnya langsung terbuka. Kalau ada update codingan, kamu cuma update di Server, dan semua PC otomatis dapat versi terbaru. Ini standar pabrik modern!

---

Nah, sebelum kita anggap sistem ini "Hijau" (Go-Live), ada **Bug Kritis** yang harus kita perbaiki dari kode tahap sebelumnya, dan kita harus lengkapi standar Enterprise.

### BUG FIX: Dashboard Hanya Bisa 1 Line!
Di kode sebelumnya, dashboard menggunakan `LineDashboardDto? LineData` (hanya menampung 1 line). Padahal area DIJ punya DIJ-01, DIJ-02, dst. Jika Operator DIJ-02 input, data DIJ-01 akan tertimpa di dashboard. Kita harus ubah jadi `List` atau `Dictionary`.

---

### TAHAP 19: UPGRADE DASHBOARD MULTI-LINE (REAL-TIME)

Kita akan buat Dashboard yang menampilkan *semua line sekaligus*, dan update masing-masing line secara independen.

Buka `LiveDashboard.razor`, ubah seluruh kode `@code` dan tampilannya:

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
                
                <!-- OEE BIG NUMBER -->
                <MudText Typo="Typo.h1" Style="color: @GetOeeColor(line.OEE); font-weight: 900;">
                    @line.OEE %
                </MudText>

                <MudStack Row="true" Spacing="6" Class="mt-4 mb-6">
                    <div class="text-center">
                        <MudText Typo="Typo.h6" Color="Color.Info">A: @line.Availability %</MudText>
                    </div>
                    <div class="text-center">
                        <MudText Typo="Typo.h6" Color="Color.Warning">P: @line.Performance %</MudText>
                    </div>
                    <div class="text-center">
                        <MudText Typo="Typo.h6" Color="Color.Success">Q: @line.Quality %</MudText>
                    </div>
                </MudStack>

                <MudDivider Class="mb-4" />
                
                <MudText Typo="Typo.body1" Color="Color.Surface">Output: <b>@line.TotalGood</b> / Target: <b>@line.TotalTarget</b></MudText>
                <MudProgressLinear Color="Color.Success" Size="Size.Large" Value="@(line.TotalTarget > 0 ? Math.Round((double)line.TotalGood / line.TotalTarget * 100) : 0)" Class="mt-2" />
            </MudPaper>
        </MudItem>
    }
</MudGrid>

@code {
    // Gunakan Dictionary agar mudah lookup LineId saat SignalR menembak data
    private Dictionary<Guid, LineDashboardDto> DashboardData = new();
    private HubConnection? hubConnection;

    protected override async Task OnInitializedAsync()
    {
        // 1. Inisialisasi data awal untuk semua line (Walaupun belum ada produksi, line tetap muncul)
        var lines = await Db.ProductionLines.ToListAsync();
        foreach (var line in lines)
        {
            DashboardData[line.Id] = new LineDashboardDto
            {
                LineId = line.Id,
                LineCode = line.LineCode,
                OEE = 0, Availability = 0, Performance = 0, Quality = 0,
                TotalGood = 0, TotalTarget = 0
            };
        }

        // 2. Setup SignalR
        hubConnection = new HubConnectionBuilder()
            .WithUrl(NavigationManager.ToAbsoluteUri("/productionHub"))
            .Build();

        hubConnection.On<LineDashboardDto>("ReceiveProductionUpdate", (update) =>
        {
            // Update HANYA line yang terdampak, line lain tidak berkedip
            DashboardData[update.LineId] = update;
            StateHasChanged(); 
        });

        await hubConnection.StartAsync();
    }

    private string GetOeeColor(double oee) => oee switch
    {
        >= 85 => "#4caf50", // Hijau
        >= 60 => "#ff9800", // Orange
        _ => "#f44336"      // Merah
    };

    public async ValueTask DisposeAsync()
    {
        if (hubConnection is not null) await hubConnection.DisposeAsync();
    }
}
```

---

### TAHAP 20: MEMBUKA KUNCI OPERATOR INPUT (SEBELUMNYA MASIH DUMMY)

Di tahap sebelumnya, `OperatorInput.razor` komennya masih `"Asumsi service punya method..."`. Mari kita buat ini benar-benar berjalan.

Tambahkan method ini di `ProductionService.cs` (`src/MES.Application/Services/`):

```csharp
public async Task<List<WorkOrder>> GetActiveWorkOrders()
{
    // Ambil WO yang sudah Released atau sedang Running
    return await _db.WorkOrders
        .Include(x => x.Product)
        .Include(x => x.Line)
        .Where(x => x.Status == WOStatus.Released || x.Status == WOStatus.Running)
        .OrderByDescending(x => x.WONumber)
        .ToListAsync();
}
```

Lalu lengkapi `OperatorInput.razor` di bagian `@code`:

```csharp
@page "/operator/input"
@using MES.Core.Entities
@using MES.Application.Services
@inject ProductionService ProdService

// ... (Tampilan MudBlazor dari Tahap 18 tetap dipakai) ...

@code {
    private List<WorkOrder> WorkOrders = new();
    private Guid SelectedWOId;
    private int InputGoodQty;
    private int InputRejectQty;
    private string Message = "";

    protected override async Task OnInitializedAsync()
    {
        // PANGGIL METHOD YANG BARU KITA BUAT
        WorkOrders = await ProdService.GetActiveWorkOrders();
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
            
            // Refresh list WO (Jika WO Complete, dia hilang dari list aktif)
            WorkOrders = await ProdService.GetActiveWorkOrders(); 
        }
        catch(Exception ex) 
        { 
            Message = $"Error: {ex.Message}"; 
        }
    }
}
```

---

### TAHAP 21: STANDAR ENTERPRISE - ROLE-BASED NAVIGATION MENU

Sistem yang aman tidak hanya melarang akses halaman, tapi juga **menyembunyikan menu yang bukan haknya**. Operator tidak perlu melihat menu "Master Data".

Buat/Ubah file `NavMenu.razor` di `src/MES.Server/Shared/`:

```razor
<MudNavMenu>
    <MudNavLink Href="" Icon="@Icons.Material.Filled.Dashboard">Live Dashboard</MudNavLink>
    
    <AuthorizeView Roles="Supervisor,Admin">
        <Authorized>
            <MudNavLink Href="supervisor/workorders" Icon="@Icons.Material.Filled.Assignment">Work Order Mgmt</MudNavLink>
        </Authorized>
    </AuthorizeView>

    <AuthorizeView Roles="Operator">
        <Authorized>
            <MudNavLink Href="operator/input" Icon="@Icons.Material.Filled.ProductionQuantityLimits">Input Produksi</MudNavLink>
        </Authorized>
    </AuthorizeView>

    <AuthorizeView Roles="Engineering">
        <Authorized>
            <MudNavLink Href="engineering/downtime" Icon="@Icons.Material.Filled.Build">Input Downtime</MudNavLink>
        </Authorized>
    </AuthorizeView>

    <AuthorizeView Roles="Admin">
        <Authorized>
            <MudNavLink Href="admin/lines" Icon="@Icons.Material.Filled.Settings">Master Data</MudNavLink>
        </Authorized>
    </AuthorizeView>
</MudNavMenu>
```

---

### TAHAP 22: STANDAR ENTERPRISE - AUDIT TRAIL LOGGING

Di pabrik, jika ada data yang manipulatif, manajemen harus tahu siapa pelakunya. Setiap aksi kritis harus dicatat.

1. Buat Entity `ActivityLog.cs` di `MES.Core/Entities`:
```csharp
public class ActivityLog
{
    public Guid Id { get; set; }
    public string UserName { get; set; } = string.Empty;
    public string Action { get; set; } = string.Empty; // "INPUT_PRODUCTION", "RELEASE_WO"
    public string Detail { get; set; } = string.Empty; // "WO-123 Good:100 Reject:5"
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}
```

2. Tambahkan `DbSet<ActivityLog> ActivityLogs` di `MesDbContext`, lalu jalankan Migrasi (`dotnet ef migrations add AddActivityLog` -> `dotnet ef database update`).

3. Suntikkan logging di `ProductionService.cs`:
```csharp
public async Task InputProduction(Guid woId, int goodQty, int rejectQty, string operatorId)
{
    // ... (kode logika produksi dan hitung OEE) ...

    // Tambahkan Logging sebelum SaveChanges
    _db.ActivityLogs.Add(new ActivityLog
    {
        UserName = operatorId,
        Action = "INPUT_PRODUCTION",
        Detail = $"Input to WO:{woId}. Good:{goodQty}, Reject:{rejectQty}"
    });

    await _db.SaveChangesAsync();
    // ... (Broadcast SignalR) ...
}
```

---

### FINAL CHECKLIST GO-LIVE

Coba jalankan sistemmu (Klik Play di Visual Studio). Lakukan skenario uji berikut:

1. [ ] Buka `http://localhost:5001/live-dashboard`. Apakah muncul kotak-kotak Line DIJ dengan OEE 0%?
2. [ ] Buka tab baru ke `http://localhost:5001/admin/lines`. Apakah kamu bisa tambah line "DIJ-02"?
3. [ ] Buka `http://localhost:5001/supervisor/workorders`. Buat WO untuk DIJ-01, lalu klik **Release**.
4. [ ] Buka `http://localhost:5001/operator/input`. Pilih WO yang baru di-release. Input Good: 50. Submit.
5. [ ] Kembali ke tab **Live Dashboard**. Apakah di kotak DIJ-01, Good Qty berubah jadi 50, dan OEE/Performance/Quality berubah angkanya secara otomatis (tanpa F5)?
6. [ ] Coba akses halaman Admin menggunakan akun Operator. Apakah menu dan aksesnya diblokir?

**Jika semua checklist di atas berjalan mulus, selamat! Kamu baru saja membangun sebuah Core Manufacturing Execution System yang berstandar Industri 4.0.**

Apakah ada error yang muncul saat kamu menjalankan step-step di atas? Atau ada fitur spesifik yang ingin kamu poles lebih dalam?
