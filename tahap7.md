Sikapmu sangat keren, Junior! Itulah mental seorang Engineer sejati: tidak mau setengah jalan, dan menjadikan setiap proyek sebagai investasi ilmu. 

Saya sangat menghargai niatmu. Oleh karena itu, saya akan menuntunmu menyelesaikan **20% sisa kekurangan** tersebut. Kita akan tuliskan kodenya lengkap. Kode-kode di bawah ini adalah *boilerplate* (pola dasar) yang bisa kamu tiru untuk modul-modul lainnya kelak.

Mari kita tutup semua lubang tersebut!

---

### TAHAP 28: MENYEMPURNAKAN SHELL APLIKASI (LAYOUT & NAVIGASI)

Sebelumnya kita buat `NavMenu.razor`, tapi belum kita bungkus di `MainLayout.razor`. Layout ini akan menjadi "cangkang" utama aplikasimu. Menggunakan MudBlazor, kita akan buat tampilan seperti dashboard admin modern.

Buka `src/MES.Server/Shared/MainLayout.razor`, timpa seluruh isinya dengan ini:

```razor
@inherits LayoutComponentBase

<MudThemeProvider IsDarkMode="true" />
<MudDialogProvider>
    <MudSnackbarProvider />
</MudDialogProvider>

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

Lalu buka `src/MES.Server/Shared/NavMenu.razor`, pastikan isinya seperti ini agar navigasinya rapi:

```razor
<MudNavMenu>
    <MudNavLink Href="/live-dashboard" Icon="@Icons.Material.Filled.Dashboard">Live Dashboard</MudNavLink>
    
    <MudNavGroup Icon="@Icons.Material.Filled.Assignment" Title="Work Order">
        <MudNavLink Href="/supervisor/workorders" Icon="@Icons.Material.Filled.AddTask">Manage WO</MudNavLink>
    </MudNavGroup>

    <MudNavGroup Icon="@Icons.Material.Filled.ProductionQuantityLimits" Title="Production Floor">
        <MudNavLink Href="/operator/input" Icon="@Icons.Material.Filled.Input">Input Produksi</MudNavLink>
        <MudNavLink Href="/qc/input" Icon="@Icons.Material.Filled.BugReport">Input Defect (QC)</MudNavLink>
        <MudNavLink Href="/engineering/downtime" Icon="@Icons.Material.Filled.Build">Input Downtime (Eng)</MudNavLink>
    </MudNavGroup>

    <MudNavGroup Icon="@Icons.Material.Filled.Folder" Title="Master Data">
        <MudNavLink Href="/admin/lines" Icon="@Icons.Material.Filled.ViewList">Lines</MudNavLink>
        <MudNavLink Href="/admin/products" Icon="@Icons.Material.Filled.Category">Products</MudNavLink>
        <MudNavLink Href="/admin/defects" Icon="@Icons.Material.Filled.Warning">Defect Codes</MudNavLink>
        <MudNavLink Href="/admin/downtimes" Icon="@Icons.Material.Filled.Timer">Downtime Codes</MudNavLink>
    </MudNavGroup>

    <MudNavLink Href="/reports" Icon="@Icons.Material.Filled.Assessment">Reports</MudNavLink>
</MudNavMenu>
```

---

### TAHAP 29: MEMBUAT GATE MASUK (LOGIN PAGE)

Kita sudah pasang Identity, tapi butuh halaman Login-nya. Buat file baru `Login.razor` di `src/MES.Server/Pages/`:

```razor
@page "/login"
@using Microsoft.AspNetCore.Identity
@inject UserManager<IdentityUser> UserManager
@inject SignInManager<IdentityUser> SignInManager
@inject NavigationManager NavigationManager

<MudContainer MaxWidth="MaxWidth.ExtraSmall" Class="mt-16">
    <MudPaper Class="pa-8" Style="background-color: #1e1e1e;">
        <MudText Typo="Typo.h4" Align="Align.Center" Class="mb-6">MES Login</MudText>
        
        <MudTextField T="string" Label="Email" @bind-Value="InputEmail" Variant="Variant.Filled" />
        <MudTextField T="string" Label="Password" @bind-Value="InputPassword" Variant="Variant.Filled" InputType="InputType.Password" Class="mt-4" />

        @if (!string.IsNullOrEmpty(ErrorMessage))
        {
            <MudAlert Severity="Severity.Error" Class="mt-4">@ErrorMessage</MudAlert>
        }

        <MudButton Variant="Variant.Filled" Color="Color.Primary" FullWidth="true" Class="mt-6" OnClick="LoginUser">
            SIGN IN
        </MudButton>
    </MudPaper>
</MudContainer>

@code {
    private string InputEmail = "admin@mes.com";
    private string InputPassword = "Password123!";
    private string ErrorMessage = "";

    private async Task LoginUser()
    {
        var result = await SignInManager.PasswordSignInAsync(InputEmail, InputPassword, isPersistent: false, lockoutOnFailure: false);
        
        if (result.Succeeded)
        {
            NavigationManager.NavigateTo("/live-dashboard", forceLoad: true);
        }
        else
        {
            ErrorMessage = "Invalid login attempt.";
        }
    }
}
```

---

### TAHAP 30: MASTER DATA - PRODUCTS (KRUSIAL UNTUK OEE PERFORMANCE)

Tanpa Master Product, Operator tidak bisa pilih WO, dan OEE Performance tidak bisa dihitung karena tidak tahu *Ideal Cycle time*.

Buat file `MasterProducts.razor` di `src/MES.Server/Pages/Admin/`:

```razor
@page "/admin/products"
@using MES.Core.Entities
@using MES.Infrastructure.Data
@inject MesDbContext Db

<PageTitle>Master Products</PageTitle>
<MudText Typo="Typo.h4" Class="mb-4">Master Products</MudText>

<MudButton Variant="Variant.Filled" Color="Color.Primary" Class="mb-4" OnClick="() => OpenModal(new Product())">+ Add Product</MudButton>

<MudTable Items="@Products" Hover="true" Elevation="2" Style="background-color: #1e1e1e;">
    <ToolBarContent>
        <MudText Typo="Typo.h6">Product List</MudText>
    </ToolBarContent>
    <HeaderContent>
        <MudTh>Product Code</MudTh>
        <MudTh>Product Name</MudTh>
        <MudTh>Ideal Cycle Time (Sec)</MudTh>
        <MudTh>Action</MudTh>
    </HeaderContent>
    <RowTemplate>
        <MudTd>@context.ProductCode</MudTd>
        <MudTd>@context.ProductName</MudTd>
        <MudTd>@context.IdealCycleTimeSeconds</MudTd>
        <MudTd><MudButton Color="Color.Warning" OnClick="() => OpenModal(context)">Edit</MudButton></MudTd>
    </RowTemplate>
</MudTable>

<!-- Modal Add/Edit -->
@if (ShowModal)
{
    <MudDialog IsOpen="ShowModal" DialogOptions="new DialogOptions() { CloseButton = true }">
        <DialogContent>
            <MudTextField T="string" Label="Product Code" @bind-Value="EditingProduct.ProductCode" Class="mb-3" />
            <MudTextField T="string" Label="Product Name" @bind-Value="EditingProduct.ProductName" Class="mb-3" />
            <MudNumericField<double> Label="Ideal Cycle Time (Seconds)" @bind-Value="EditingProduct.IdealCycleTimeSeconds" Min="0.1" />
        </DialogContent>
        <DialogActions>
            <MudButton OnClick="CloseModal">Cancel</MudButton>
            <MudButton Color="Color.Primary" OnClick="SaveProduct">Save</MudButton>
        </DialogActions>
    </MudDialog>
}

@code {
    private List<Product> Products = new();
    private bool ShowModal = false;
    private Product EditingProduct = new();

    protected override async Task OnInitializedAsync()
    {
        Products = await Db.Products.ToListAsync();
    }

    private void OpenModal(Product product)
    {
        EditingProduct = product.Id == Guid.Empty ? new Product() : product;
        ShowModal = true;
    }

    private void CloseModal() => ShowModal = false;

    private async Task SaveProduct()
    {
        if (EditingProduct.Id == Guid.Empty)
        {
            EditingProduct.Id = Guid.NewGuid();
            Db.Products.Add(EditingProduct);
        }
        else
        {
            Db.Products.Update(EditingProduct);
        }

        await Db.SaveChangesAsync();
        Products = await Db.Products.ToListAsync();
        CloseModal();
    }
}
```

---

### TAHAP 31: MASTER DATA - DEFECT CODES & DOWNTIME CODES

Modul ini persis sama polanya dengan MasterProducts. Kamu cukup buat file `MasterDefects.razor` (di `/admin/defects`) dan `MasterDowntimes.razor` (di `/admin/downtimes`).

**1. Tambahkan Entity di `MES.Core`:**
```csharp
// DefectCode.cs
public class DefectCode { public Guid Id { get; set; } public string Code { get; set; } public string Description { get; set; } }

// DowntimeCode.cs
public class DowntimeCode { public Guid Id { get; set; } public string Code { get; set; } public string Description { get; set; } }
```

**2. Tambahkan DbSet di `MesDbContext`:**
```csharp
public DbSet<DefectCode> DefectCodes => Set<DefectCode>();
public DbSet<DowntimeCode> DowntimeCodes => Set<DowntimeCode>();
```

**3. Buat Halaman CRUD-nya:**
Gunakan *Copy-Paste* dari `MasterProducts.razor`, ganti kata `Product` jadi `DefectCode`, dan hilangkan field `IdealCycleTimeSeconds` (karena Defect/Downtime hanya butuh Code & Description).

---

### TAHAP 32: MODULE QC DEFECT INPUT

Ini modul penutup untuk melengkapi alur Quality. Kita sudah buat logic-nya di `ProductionService`, sekarang buat UI-nya.

Buat `QcInput.razor` di `src/MES.Server/Pages/QC/`:

```razor
@page "/qc/input"
@using MES.Core.Entities
@using MES.Application.Services
@using MES.Infrastructure.Data
@inject ProductionService ProdService
@inject MesDbContext Db

<PageTitle>QC Defect Input</PageTitle>
<MudText Typo="Typo.h4" Class="mb-4">Defect Input (Quality Control)</MudText>

<MudPaper Class="pa-6" Style="background-color: #1e1e1e;">
    <MudSelect T="Guid" Label="Select Work Order" @bind-Value="SelectedWOId" Variant="Variant.Filled" Class="mb-4">
        @foreach(var wo in WorkOrders)
        {
            <MudSelectItem Value="@wo.Id">@wo.WONumber - @wo.Product?.ProductName</MudSelectItem>
        }
    </MudSelect>

    <MudSelect T="string" Label="Defect Code" @bind-Value="SelectedDefectCode" Variant="Variant.Filled" Class="mb-4">
        @foreach(var defect in DefectCodes)
        {
            <MudSelectItem Value="@defect.Code">@defect.Code - @defect.Description</MudSelectItem>
        }
    </MudSelect>

    <MudNumericField<int> Label="Defect Quantity" @bind-Value="DefectQty" Min="1" Variant="Variant.Filled" Class="mb-4" />
    <MudTextField T="string" Label="Notes" @bind-Value="Notes" Variant="Variant.Filled" Class="mb-4" />

    <MudButton Variant="Variant.Filled" Color="Color.Error" Size="Size.Large" FullWidth="true" OnClick="SubmitDefect">
        SUBMIT DEFECT
    </MudButton>
</MudPaper>

@code {
    private List<WorkOrder> WorkOrders = new();
    private List<DefectCode> DefectCodes = new();
    private Guid SelectedWOId;
    private string SelectedDefectCode = "";
    private int DefectQty = 1;
    private string Notes = "";

    protected override async Task OnInitializedAsync()
    {
        WorkOrders = await ProdService.GetActiveWorkOrders();
        DefectCodes = await Db.DefectCodes.ToListAsync();
    }

    private async Task SubmitDefect()
    {
        if(SelectedWOId == Guid.Empty || string.IsNullOrEmpty(SelectedDefectCode)) return;

        await ProdService.InputQcDefect(SelectedWOId, SelectedDefectCode, DefectQty, Notes);
        
        Notes = "";
        DefectQty = 1;
    }
}
```

---

### TAHAP 33: MIGRASI DATABASE TERAKHIR (WAJIB!)

Karena kita menambahkan Entity baru (`DefectCode`, `DowntimeCode`, `QcResult`, `ActivityLog`), database lokalmu belum punya tabel-tabel ini. Kamu **WAJIB** melakukan migrasi sekali lagi.

Buka Terminal di root solution:
```bash
cd src/MES.Infrastructure
dotnet ef migrations add AddMasterAndQCModules --startup-project ../MES.Server
dotnet ef database update --startup-project ../MES.Server
```

---

### TAHAP 34: FINAL RUN & UJI SKENARIO END-TO-END

Sekarang, jalankan aplikasimu di Visual Studio. Lakukan skenario produksi lengkap ini (Skenario Happy Path):

1. **Admin Login:** Buka halaman Master Data. Buat Line "DIJ-01", Product "Action Figure" (Ideal Cycle Time: 5 detik), Defect Code "SCR-01" (Scratch), Downtime Code "BRK-01" (Mesin Rusak).
2. **Supervisor Login:** Buka halaman WO, buat WO baru target 1000 pcs untuk Product "Action Figure" di Line DIJ-01. Klik **Release**.
3. **Operator Login:** Buka Input Produksi. Pilih WO tadi. Input Good: 100, Reject: 0. Klik Submit.
4. **QC Login:** Buka Input Defect. Pilih WO tadi. Pilih Defect SCR-01, Qty 5. Submit. *(Ini akan menambah reject, OEE Quality harusnya turun).*
5. **Engineering Login:** Buka Input Downtime. Pilih Line DIJ-01. Isi start dan end time (misal downtime 30 menit). Submit. *(Ini akan mengurangi operating time, OEE Availability harusnya turun).*
6. **Leader View (Dashboard):** Buka Live Dashboard. Kamu harusnya melihat Card DIJ-01 dengan:
   - OEE turun dari 100% (karena ada downtime & defect).
   - Availability turun.
   - Quality turun.
   - Good Output: 100, Target: 1000.

---

### 💡 Kata Penutup dari Senior

Dengan selesainya tahap 34 di atas, **Kamu telah berhasil membangun sistem MES Level Enterprise dari nol.** 

Kamu tidak hanya belajar coding, tapi kamu belajar:
- **Arsitektur Software** (Memisahkan Core, Infra, dan App).
- **Domain-Driven Design** (Memahami istilah WO, OEE, Shift, Downtime).
- **Real-time Programming** (SignalR).
- **Entity Framework & Database Migrations**.

Mulai dari sini, apapun bug atau fitur tambahan yang kamu inginkan (seperti notifikasi, chart historical OEE, dll), kamu punya pondasi yang sangat kuat untuk mengembangkannya sendiri.

Selamat berkoding, Junior. Kamu sudah lulus dari pelatihan ini! 🚀
