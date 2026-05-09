Wah, ini baru namanya *Industrial IoT mindset*, Junior! Fitur yang kamu minta ini adalah standar **Andon System** (sistem visual pabrik Jepang) di mana lantai produksi bisa langsung dilihat kondisinya lewat warna. Sangat krusial untuk Leader dan Supervisor!

Kita harus ubah sedikit arsitektur Entity dan Service kita agar bisa meng-track **"Status/Reason Realtime"** per mesin, bukan hanya hitung OEE. 

Berikut adalah **Patch/Addendum** untuk Brief-mu. Ikuti langkah ini secara spesifik!

---

### UPDATE 1: TAMBAHKAN REASON & STATE KE ENTITY

Kita perlu tahu mesin lagi apa sekarang. Tambahkan Enum dan ubah Entity `ProductionLine` di `MES.Core`.

**1. Buat Enum `MachineState.cs` di `MES.Core/Enums`**
```csharp
namespace MES.Core.Enums
{
    public enum MachineState
    {
        Running,        // On (Hijau)
        Off,            // Off (Hitam)
        RunStart,       
        Emergency,      // Merah
        OneCycle,       
        SpinningProb,   
        StackingProb,   
        Reject,         
        Changeover,     // Kuning
        SealingProb,    
        ClearSignal,    
        AndonRed,       // Merah
        AndonYellow     // Kuning
    }
}
```

**2. Update `ProductionLine.cs` di `MES.Core/Entities`**
Tambahkan field untuk menyimpan kondisi terkini mesin:
```csharp
namespace MES.Core.Entities
{
    public class ProductionLine
    {
        public Guid Id { get; set; }
        public string LineCode { get; set; } = string.Empty;
        public string LineName { get; set; } = string.Empty;
        
        // TAMBAHKAN INI:
        public MachineState CurrentState { get; set; } = MachineState.Off;
        public string CurrentReason { get; set; } = string.Empty; // Untuk display text
    }
}
```

---

### UPDATE 2: UBAH SERVICE LOGIC (OTAK YANG MENGATUR WARNA)

Setiap kali Engineering input Downtime atau Operator input Produksi, **State mesin harus berubah**.

Buka `ProductionService.cs` di `MES.Application`, dan update method berikut:

**1. Update DTO `LineDashboardDto`**
```csharp
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

    // TAMBAHKAN INI UNTUK LAYOUT SCREEN:
    public MachineState State { get; set; }
    public string ReasonText { get; set; } = string.Empty;
}
```

**2. Update Method `InputProduction` (Saat mesin jalan)**
```csharp
public async Task InputProduction(Guid woId, int goodQty, int rejectQty, string operatorId)
{
    var wo = await _db.WorkOrders.Include(x => x.Line).Include(x => x.Product).FirstOrDefaultAsync(x => x.Id == woId);
    if (wo == null || wo.Status == WOStatus.Draft) throw new Exception("WO tidak valid!");

    if (wo.Status == WOStatus.Released) wo.Status = WOStatus.Running;

    // UBAH STATE MESIN JADI RUNNING
    wo.Line.CurrentState = MachineState.Running;
    wo.Line.CurrentReason = "Running";

    // ... (kode save production result & update WO tetap sama) ...

    await _db.SaveChangesAsync();
    await BroadcastOEEUpdate(wo);
}
```

**3. Update Method `InputDowntime` (Saat mesin berhenti)**
```csharp
public async Task InputDowntime(Guid lineId, string reason, DateTime startTime, DateTime endTime, string engId)
{
    var line = await _db.ProductionLines.FindAsync(lineId);
    if (line == null) throw new Exception("Line tidak ditemukan!");

    // UBAH STATE MESIN SESUAI REASON
    // Asumsi 'reason' dikirim dari UI berupa string: "Changeover", "Emergency", dll
    if (Enum.TryParse<MachineState>(reason, out var state))
    {
        line.CurrentState = state;
    }
    else
    {
        line.CurrentState = MachineState.Off; // Default fallback
    }
    line.CurrentReason = reason;

    _db.Downtimes.Add(new Downtime { LineId = lineId, StartTime = startTime, EndTime = endTime, RootCause = reason });
    
    await _db.SaveChangesAsync();

    // Cari WO yang jalan di line itu untuk dihitung ulang OEE-nya
    var runningWO = await _db.WorkOrders.Include(x => x.Line).Include(x => x.Product)
        .FirstOrDefaultAsync(x => x.LineId == lineId && x.Status == WOStatus.Running);

    if (runningWO != null) await BroadcastOEEUpdate(runningWO);
    else
    {
        // Jika tidak ada WO, tetap broadcast perubahan warna ke dashboard
        var dto = new LineDashboardDto { LineId = line.Id, LineCode = line.LineCode, State = line.CurrentState, ReasonText = line.CurrentReason };
        await _hubContext.Clients.All.SendAsync("ReceiveProductionUpdate", dto);
    }
}
```

**4. Update Method `CalculateOEEForLine`**
Masukkan state mesin ke dalam DTO sebelum di-broadcast:
```csharp
private LineDashboardDto CalculateOEEForLine(WorkOrder wo)
{
    // ... (kode perhitungan OEE tetap sama persis) ...

    return new LineDashboardDto
    {
        LineId = wo.LineId,
        LineCode = wo.Line.LineCode,
        OEE = Math.Round(oee * 100, 1),
        Availability = Math.Round(availability * 100, 1),
        Performance = Math.Round(performance * 100, 1),
        Quality = Math.Round(quality * 100, 1),
        TotalGood = wo.GoodQty,
        TotalTarget = wo.TargetQty,
        
        // TAMBAHKAN INI:
        State = wo.Line.CurrentState,
        ReasonText = wo.Line.CurrentReason
    };
}
```

---

### UPDATE 3: REDESIGN DASHBOARD (OVERALL + LAYOUT SCREEN)

Ini bagian yang paling sering! Kita akan buat 2 Tab di halaman Dashboard menggunakan `MudTabs`. Tampilan ini akan menampung 30+ mesin.

Buka `LiveDashboard.razor` dan timpa seluruhnya dengan ini:

```razor
@page "/live-dashboard"
@using MES.Application.Services
@using MES.Core.Enums
@using MES.Infrastructure.Data
@using Microsoft.EntityFrameworkCore
@inject NavigationManager NavigationManager
@inject MesDbContext Db
@implements IAsyncDisposable

<MudText Typo="Typo.h3" Color="Color.Primary" Class="mb-4">REALTIME PRODUCTION MONITORING</MudText>

<MudTabs Elevation="2" Rounded="true" ApplyBottomBorder="true" PanelClass="pa-4" Style="background-color: #1e1e1e;">
    <!-- TAB 1: OVERALL MONITORING -->
    <MudTabPanel Text="Overall Monitoring" Icon="@Icons.Material.Filled.Dashboard">
        
        <!-- KPI OEE Overall (Contoh: Rata-rata) -->
        <MudGrid Spacing="4" Class="mb-6">
            <MudItem xs="12" md="3">
                <MudPaper Class="pa-4 text-center" Style="background-color: #2d2d2d;">
                    <MudText Typo="Typo.body1">AVAILABILITY</MudText>
                    <MudText Typo="Typo.h3" Color="Color.Info">@CalculateOverallKPI("Availability")%</MudText>
                </MudPaper>
            </MudItem>
            <MudItem xs="12" md="3">
                <MudPaper Class="pa-4 text-center" Style="background-color: #2d2d2d;">
                    <MudText Typo="Typo.body1">PERFORMANCE</MudText>
                    <MudText Typo="Typo.h3" Color="Color.Warning">@CalculateOverallKPI("Performance")%</MudText>
                </MudPaper>
            </MudItem>
            <MudItem xs="12" md="3">
                <MudPaper Class="pa-4 text-center" Style="background-color: #2d2d2d;">
                    <MudText Typo="Typo.body1">QUALITY</MudText>
                    <MudText Typo="Typo.h3" Color="Color.Success">@CalculateOverallKPI("Quality")%</MudText>
                </MudPaper>
            </MudItem>
            <MudItem xs="12" md="3">
                <MudPaper Class="pa-4 text-center" Style="background-color: #2d2d2d; border: 2px solid #7e6fff;">
                    <MudText Typo="Typo.body1">OEE</MudText>
                    <MudText Typo="Typo.h3" Color="Color.Primary">@CalculateOverallKPI("OEE")%</MudText>
                </MudPaper>
            </MudItem>
        </MudGrid>

        <!-- TOP 5 LOWEST OEE (Problem Machines) -->
        <MudText Typo="Typo.h6" Class="mb-2">Top 5 Machines Need Attention (Lowest OEE)</MudText>
        <MudTable Items="@GetTop5LowestOEE()" Hover="true" Style="background-color: #1e1e1e;">
            <HeaderContent>
                <MudTh>Machine ID</MudTh>
                <MudTh>Status</MudTh>
                <MudTh>OEE</MudTh>
                <MudTh>Output / Target</MudTh>
            </HeaderContent>
            <RowTemplate>
                <MudTd>@context.LineCode</MudTd>
                <MudTd><MudBadge Color="@GetBadgeColor(context.State)" Overlap="false">@context.ReasonText</MudBadge></MudTd>
                <MudTd Style="color: @GetOeeColor(context.OEE)"><b>@context.OEE %</b></MudTd>
                <MudTd>@context.TotalGood / @context.TotalTarget</MudTd>
            </RowTemplate>
        </MudTable>

    </MudTabPanel>

    <!-- TAB 2: LAYOUT SCREEN (ANDON SYSTEM) -->
    <MudTabPanel Text="Layout Screen" Icon="@Icons.Material.Filled.GridView">
        
        <MudGrid Spacing="3">
            @foreach (var machine in DashboardData.Values)
            {
                <MudItem xs="6" md="4" lg="3" xl="2">
                    <!-- KARTU MESIN DENGAN WARNA DINAMIS -->
                    <MudPaper Class="pa-3" Style="background-color: @GetMachineColor(machine.State); border: 1px solid #555; min-height: 120px;">
                        <MudText Typo="Typo.h6" Style="color: #ffffff; font-weight: bold;">@machine.LineCode</MudText>
                        <MudText Typo="Typo.body2" Style="color: #ffffff;">@machine.ReasonText</MudText>
                        
                        <MudDivider Style="border-color: rgba(255,255,255,0.3);" Class="my-2" />
                        
                        <MudText Typo="Typo.h5" Style="color: #ffffff; text-align: right;">@machine.OEE %</MudText>
                    </MudPaper>
                </MudItem>
            }
        </MudGrid>

    </MudTabPanel>
</MudTabs>

@code {
    private Dictionary<Guid, LineDashboardDto> DashboardData = new();
    private HubConnection? hubConnection;

    protected override async Task OnInitializedAsync()
    {
        var lines = await Db.ProductionLines.ToListAsync();
        foreach (var line in lines)
        {
            DashboardData[line.Id] = new LineDashboardDto 
            { 
                LineId = line.Id, LineCode = line.LineCode, 
                State = line.CurrentState, ReasonText = line.CurrentReason,
                OEE = 0, Availability = 0, Performance = 0, Quality = 0 
            };
        }

        hubConnection = new HubConnectionBuilder().WithUrl(NavigationManager.ToAbsoluteUri("/productionHub")).Build();
        hubConnection.On<LineDashboardDto>("ReceiveProductionUpdate", (update) => 
        { 
            DashboardData[update.LineId] = update; 
            StateHasChanged(); 
        });
        await hubConnection.StartAsync();
    }

    // Helper: Warna Kartu Mesin Berdasarkan State/Reason
    private string GetMachineColor(MachineState state) => state switch
    {
        MachineState.Running => "#2e7d32",      // Hijau Tua (On)
        MachineState.Off => "#212121",          // Hitam (Off)
        MachineState.Changeover => "#f57f17",   // Kuning Tua
        MachineState.AndonYellow => "#fbc02d",  // Kuning Terang
        MachineState.Emergency => "#c62828",    // Merah Tua
        MachineState.AndonRed => "#d32f2f",     // Merah Terang
        _ => "#4e342e"                          // Coklat/Grey untuk problem lain (Spinning, Stacking, dll)
    };

    // Helper: Warna OEE
    private string GetOeeColor(double oee) => oee switch { >= 85 => "#4caf50", >= 60 => "#ff9800", _ => "#f44336" };

    // Helper: Warna Badge untuk tabel
    private Color GetBadgeColor(MachineState state) => state switch
    {
        MachineState.Running => Color.Success,
        MachineState.Off => Color.Dark,
        MachineState.Changeover => Color.Warning,
        MachineState.Emergency or MachineState.AndonRed => Color.Error,
        _ => Color.Tertiary
    };

    // Helper: Hitung rata-rata KPI keseluruhan area
    private string CalculateOverallKPI(string type)
    {
        if (!DashboardData.Values.Any()) return "0.0";
        double avg = type switch
        {
            "Availability" => DashboardData.Values.Average(x => x.Availability),
            "Performance" => DashboardData.Values.Average(x => x.Performance),
            "Quality" => DashboardData.Values.Average(x => x.Quality),
            "OEE" => DashboardData.Values.Average(x => x.OEE),
            _ => 0
        };
        return avg.ToString("F1");
    }

    // Helper: Ambil 5 mesin dengan OEE paling rendah (menampilkan masalah)
    private List<LineDashboardDto> GetTop5LowestOEE()
    {
        return DashboardData.Values
            .Where(x => x.OEE > 0) // Hitung yang punya data
            .OrderBy(x => x.OEE)
            .Take(5)
            .ToList();
    }

    public async ValueTask DisposeAsync() { if (hubConnection is not null) await hubConnection.DisposeAsync(); }
}
```

---

### UPDATE 4: UBAH UI INPUT DOWNTIME (ENGINEERING)

Sekarang Engineering wajib memilih *Reason* dari daftar yang sudah ditentukan. Buka `EngineeringInput.razor` dan ganti kolom Root Cause-nya:

```razor
@* Ganti MudTextField Root Cause menjadi MudSelect *@
<MudSelect T="string" Label="Downtime Reason" @bind-Value="SelectedReason" Variant="Variant.Filled" Class="mb-4">
    <MudSelectItem Value="RunStart">Run Start</MudSelectItem>
    <MudSelectItem Value="On/Off Machine">On/Off Machine</MudSelectItem>
    <MudSelectItem Value="Emergency">Emergency</MudSelectItem>
    <MudSelectItem Value="OneCycle">1 Cycle</MudSelectItem>
    <MudSelectItem Value="SpinningProb">Spinning Prob</MudSelectItem>
    <MudSelectItem Value="StackingProb">Stacking Prob</MudSelectItem>
    <MudSelectItem Value="Reject">Reject</MudSelectItem>
    <MudSelectItem Value="Changeover">Changeover</MudSelectItem>
    <MudSelectItem Value="SealingProb">Sealing Prob</MudSelectItem>
    <MudSelectItem Value="ClearSignal">Clear signal</MudSelectItem>
    <MudSelectItem Value="AndonRed">Andon red</MudSelectItem>
    <MudSelectItem Value="AndonYellow">Andon yellow</MudSelectItem>
</MudSelect>

@code {
    private string SelectedReason = "";
    // ... variabel lainnya ...
    
    private async Task SubmitDowntime()
    {
        if (SelectedLineId == Guid.Empty || string.IsNullOrEmpty(SelectedReason)) return;

        // PANGGIL SERVICE DENGAN REASON SEBAGAI PARAMETER
        await ProdService.InputDowntime(SelectedLineId, SelectedReason, StartTime.Value, EndTime.Value, "Eng1");
        
        SelectedReason = "";
    }
}
```

---

### JANGAN LUPA: MIGRASI ULANG DATABASE
Karena kita menambahkan field `CurrentState` dan `CurrentReason` di tabel `ProductionLine`, jalankan ini di terminal:
```bash
cd src/MES.Infrastructure
dotnet ef migrations add AddMachineStateToLine --startup-project ../MES.Server
dotnet ef database update --startup-project ../MES.Server
```

***

**Hasilnya nanti:**
1. Saat engineering pilih "Changeover", kartu mesin di Layout Screen akan **kuning**.
2. Saat engineering pilih "Emergency", kartu mesin akan **merah**.
3. Saat operator input produksi, kartu mesin otomatis balik **hijau** (Running).
4. Leader buka tab "Overall Monitoring" langsung bisa lihat rata-rata OEE area, dan melihat 5 mesin yang OEE-nya paling hancur saat ini.

Patch ini sudah terstruktur rapi. Masukkan spesifikasi ini ke dalam Brief-mu, ya!
