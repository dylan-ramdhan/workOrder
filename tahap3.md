Pertanyaan yang sangat kritis, Junior! Sebagai Senior, saya sangat senang kamu memikirkan UI/UX sejak awal. 

Jawabannya: **Untuk proyek MES kelas Enterprise di Blazor, saya sangat menyarankan menggunakan Component Library, BUKAN TailwindCSS murni.**

**Kenapa?**
1. **TailwindCSS** itu luar biasa untuk custom design, tapi di Blazor kamu harus racik sendiri semua komponennya (tabel, modal, dropdown, chart) dari nol. Ini memakan waktu development yang sangat lama.
2. Di lingkungan manufacturing, kita butuh komponen data-heavy yang stabil (Tabel yang bisa scroll ribuan data, Chart realtime, Dialog yang smooth).

**Pilihan Senjata Utama kita: MUDBLAZOR**
MudBlazor adalah library UI open-source paling populer untuk Blazor. Mengusung Material Design, sangat cepat, punya komponen Chart bawaan, dan yang paling penting: **Tampilannya sangat cocok dijadikan Industrial Dashboard** (Gelap, kontras tinggi, kotak-kotak tegas).

Mari kita level up tampilan MES kita agar layak tampil di lantai produksi!

---

### TAHAP 15: INSTALASI & SETUP MUDBLAZOR (INDUSTRIAL UI)

1. **Install NuGet Package**
Buka terminal di root solution:
```bash
dotnet add src/MES.Server package MudBlazor
```

2. **Daftarkan di `Program.cs`**
Tambahkan ini sebelum `builder.Build()`:
```csharp
builder.Services.AddMudServices();
```

3. **Tambahkan Import**
Buka file `_Imports.razor` di `src/MES.Server/`, tambahkan:
```csharp
@using MudBlazor
```

4. **Setup Layout Utama**
Buka `App.razor` di `src/MES.Server/`, bungkus dengan provider MudBlazor:
```razor
<MudThemeProvider IsDarkMode="true" /> <!-- Dark Mode khas Industrial -->
<MudDialogProvider />
<MudSnackbarProvider />
<Router AppAssembly="@typeof(App).Assembly">
    <Found Context="routeData">
        <RouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)" />
        <FocusOnNavigate RouteData="@routeData" Selector="h1" />
    </Found>
</Router>
```

5. **Ubah `MainLayout.razor`**
Kita buat layout sidebar yang rapi untuk navigasi MES:
```razor
@inherits LayoutComponentBase

<MudLayout>
    <MudDrawer Elevation="2" Color="Color.Dark">
        <MudDrawerHeader>
            <MudText Typo="Typo.h6">MES DIJ</MudText>
        </MudDrawerHeader>
        <NavMenu />
    </MudDrawer>
    <MudMainContent Class="pa-4" Style="background-color: #121212; color: #ffffff;">
        @Body
    </MudMainContent>
</MudLayout>
```

---

### TAHAP 16: MENGUBAH LIVE DASHBOARD MENJADI INDUSTRIAL GRADE

Ini adalah bagian favorit saya. Kita akan buat Dashboard yang bikin "WOW" buat Leader dan Supervisor di lantai produksi. Font besar, warna kontras, update realtime.

Hapus isi `LiveDashboard.razor` sebelumnya, ganti dengan ini:

```razor
@page "/live-dashboard"
@using MES.Application.Services
@inject NavigationManager NavigationManager
@implements IAsyncDisposable

<MudText Typo="Typo.h4" Color="Color.Primary" Class="mb-4">LIVE MONITORING - AREA DIRECT INK JET</MudText>

<MudGrid Spacing="4">
    @if (LineData != null)
    {
        <MudItem xs="12" md="4">
            <MudPaper Class="pa-4" Style="background-color: #1e1e1e; border-left: 6px solid @GetOeeColor(LineData.OEE);">
                <MudText Typo="Typo.h5" Color="Color.Surface">@LineData.LineCode</MudText>
                
                <!-- OEE BIG NUMBER -->
                <MudText Typo="Typo.h2" Style="color: @GetOeeColor(LineData.OEE); font-weight: bold;" Class="mt-2">
                    @LineData.OEE %
                </MudText>
                <MudText Typo="Typo.body2" Color="Color.Surface">Overall Equipment Effectiveness</MudText>

                <MudStack Row="true" Spacing="4" Class="mt-4">
                    <!-- Availability Gauge -->
                    <div class="text-center" style="width: 33%">
                        <MudText Typo="Typo.h5" Color="Color.Info">@LineData.Availability %</MudText>
                        <MudText Typo="Typo.body2">Availability</MudText>
                    </div>
                    <!-- Performance Gauge -->
                    <div class="text-center" style="width: 33%">
                        <MudText Typo="Typo.h5" Color="Color.Warning">@LineData.Performance %</MudText>
                        <MudText Typo="Typo.body2">Performance</MudText>
                    </div>
                    <!-- Quality Gauge -->
                    <div class="text-center" style="width: 33%">
                        <MudText Typo="Typo.h5" Color="Color.Success">@LineData.Quality %</MudText>
                        <MudText Typo="Typo.body2">Quality</MudText>
                    </div>
                </MudStack>

                <MudDivider Class="my-4" />
                
                <!-- Target vs Actual Progress Bar -->
                <MudText Typo="Typo.body2" Color="Color.Surface">Output Good: <b>@LineData.TotalGood</b> / Target: <b>@LineData.TotalTarget</b></MudText>
                <MudProgressLinear Color="Color.Success" Size="Size.Large" Value="@(LineData.TotalTarget > 0 ? (LineData.TotalGood / LineData.TotalTarget) * 100 : 0)" Class="mt-2" />
            </MudPaper>
        </MudItem>
    }
    else
    {
        <MudItem xs="12">
            <MudAlert Severity="Severity.Info">Menunggu data produksi masuk. Pastikan Operator sudah melakukan input.</MudAlert>
        </MudItem>
    }
</MudGrid>

@code {
    private LineDashboardDto? LineData;
    private HubConnection? hubConnection;

    protected override async Task OnInitializedAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(NavigationManager.ToAbsoluteUri("/productionHub"))
            .Build();

        hubConnection.On<LineDashboardDto>("ReceiveProductionUpdate", (update) =>
        {
            LineData = update;
            StateHasChanged(); 
        });

        await hubConnection.StartAsync();
    }

    private string GetOeeColor(double oee) => oee switch
    {
        >= 85 => "#4caf50", // Hijau
        >= 60 => "#ff9800", // Kuning/Orange
        _ => "#f44336"      // Merah
    };

    public async ValueTask DisposeAsync()
    {
        if (hubConnection is not null) await hubConnection.DisposeAsync();
    }
}
```
*Coba jalankan sekarang. Kamu akan melihat dashboard bergaya Dark Mode pabrik modern, dengan angka OEE raksasa dan Progress Bar output yang sangat mudah dibaca dari jarak jauh oleh Leader.*

---

### TAHAP 17: MERAPIKAN INPUT OPERATOR DENGAN MUDBLAZOR

Form untuk operator harus super simpel dan anti-error (misal: tidak bisa input huruf di kolom quantity).

Buka `OperatorInput.razor`, ubah elemen HTML standar menjadi komponen MudBlazor:

```razor
@page "/operator/input"
@using MES.Core.Entities
@using MES.Application.Services
@inject ProductionService ProdService

<MudText Typo="Typo.h4" Class="mb-4">Input Produksi (Operator)</MudText>

<MudPaper Class="pa-6" Style="background-color: #1e1e1e;">
    <MudSelect T="Guid" Label="Pilih Work Order (Running)" @bind-Value="SelectedWOId" Variant="Variant.Filled" Class="mb-4">
        @foreach(var wo in WorkOrders)
        {
            <MudSelectItem Value="@wo.Id">@wo.WONumber - @wo.Product?.ProductName</MudSelectItem>
        }
    </MudSelect>

    <MudNumericField<int> Label="Good Quantity" @bind-Value="InputGoodQty" Min="0" Variant="Variant.Filled" Class="mb-4" />
    <MudNumericField<int> Label="Reject Quantity" @bind-Value="InputRejectQty" Min="0" Variant="Variant.Filled" Class="mb-4" />

    <MudButton Variant="Variant.Filled" Color="Color.Primary" Size="Size.Large" FullWidth="true" OnClick="SubmitProduction">
        SUBMIT PRODUKSI
    </MudButton>

    @if (!string.IsNullOrEmpty(Message))
    {
        <MudAlert Severity="Severity.Success" Class="mt-4">@Message</MudAlert>
    }
</MudPaper>

@code {
    // ... (Code logic tetap sama seperti tahap sebelumnya, pastikan ambil data WorkOrders yang statusnya Released/Running dari service)
    private List<WorkOrder> WorkOrders = new();
    private Guid SelectedWOId;
    private int InputGoodQty;
    private int InputRejectQty;
    private string Message = "";

    private async Task SubmitProduction()
    {
        if(SelectedWOId == Guid.Empty) return;
        try
        {
            await ProdService.InputProduction(SelectedWOId, InputGoodQty, InputRejectQty, "Operator1");
            Message = "Data berhasil disimpan & Dashboard terupdate!";
            InputGoodQty = 0; InputRejectQty = 0;
        }
        catch(Exception ex) { Message = $"Error: {ex.Message}"; }
    }
}
```
*Perhatikan `MudNumericField`. Ini jauh lebih aman daripada `<input type="number">` biasa karena otomatis menangani validasi angka dan tombol increment/decrement untuk operator yang memakai layar sentuh/mouse.*

---

### TAHAP 18: DEPLOYMENT KE LANTAI PRODUKSI (IIS WINDOWS SERVER)

Ini adalah tahap akhir. Sistem yang hebat di laptopmu tidak berguna jika tidak bisa diakses oleh PC di lantai produksi.

Kita akan deploy ke IIS, sehingga sistem bisa diakses via browser di jaringan lokal (misal: `http://192.168.1.50:8080`).

**Langkah 1: Publish Project**
1. Klik kanan pada proyek `MES.Server` di Visual Studio -> **Publish**.
2. Pilih Target: **Folder**.
3. Klik **Show all settings**:
   - Configuration: `Release`
   - Target Framework: `net8.0`
   - Deployment Mode: `Self-contained` (Sangat penting agar di Windows Server tidak perlu install .NET SDK secara terpisah).
   - File Publish Options: Centang `Precompile during publishing`.
4. Klik **Publish**. Tunggu prosesnya.

**Langkah 2: Setup Windows Server / PC Host**
1. Buka PC/Laptop yang akan jadi Server (harus Windows 10/11 Pro atau Windows Server).
2. Buka **Turn Windows features on or off**.
3. Centang: **Internet Information Services (IIS)**. Pastikan di dalamnya mencentang **WebSocket Protocol** (Ini wajib agar SignalR kita jalan).
4. Download dan install **.NET 8.0 Hosting Bundle** dari website Microsoft (ini yang menjembatani Kestrel dan IIS).

**Langkah 3: Konfigurasi IIS Manager**
1. Buka **IIS Manager** (inetmgr).
2. Klik kanan **Sites** -> **Add Website**.
   - Site name: `MES_DirectInkJet`
   - Physical path: Arahkan ke folder hasil Publish tadi.
   - Port: `8080` (Jangan pakai 80 kalau sudah dipakai).
3. Klik kanan **Application Pools** -> cari `MES_DirectInkJet` -> **Basic Settings**.
   - Set **.NET CLR version** ke **No Managed Code** (Penting! IIS cuma jadi proxy, .NET yang jalanin aplikasinya).

**Langkah 4: Firewall**
Di PC Server, buka Windows Defender Firewall, izinkan port 8080 untuk Inbound Connections.

**Langkah 5: Akses!**
Sekarang, buka PC Operator/Leader di lantai produksi, buka Chrome/Edge, ketik:
`http://[IP_SERVER]:8080/live-dashboard`

*Boom! Dashboard IndustrialMU telah hidup di layar besar di dinding pabrik!*

---

### KESIMPULAN DARI SENIOR

Kita sudah menyelesaikan **Phase 1 Core MES + OEE**:
1. ✅ **Arsitektur Modular Monolith** yang rapi dan scalable.
2. ✅ **Realtime Dashboard** dengan Dark Mode Industrial menggunakan MudBlazor + SignalR.
3. ✅ **OEE Calculation Engine** yang merespons input secara langsung (Availability, Performance, Quality).
4. ✅ **Authentication & Role Authorization** (Supervisor release WO, Operator input, dll).
5. ✅ **Deployable ke IIS** siap jalan 24/7 di jaringan lokal pabrik.

Apakah ada step yang ingin kamu dalami lebih lanjut? Misalnya bagaimana cara membuat **Laporan Excel (Reporting)** yang rapi, atau bagaimana menangani **Multi-Line** di dashboard secara bersamaan? Tanyakan saja!
