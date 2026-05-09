Ini adalah langkah yang sangat tepat, Junior! Seorang Engineer sejati tidak hanya bisa mengikuti tutorial, tetapi harus bisa membaca *Brief* dan mewujudkannya menjadi kode dari nol. 

Brief di bawah ini dirancang spesifik agar kamu **tidak melebar kemana-mana**. Ini adalah "pagar" ketat untuk重建 (rebuild) proyek MES-mu. Jangan tambahkan fitur lain sebelum semua checklist di sini selesai 100%.

Simpan teks di bawah ini menjadi file `BRIEF_MES_REBUILD.md`.

***

# 🏗️ PROJECT BRIEF: MES DIRECT INK JET (SELF-GUIDED REBUILD)

## 1. Identitas Proyek
- **Nama Sistem:** Manufacturing Execution System (MES) - Area Direct Ink Jet
- **Tipe Aplikasi:** Web Application (Hosted on IIS, accessed via Browser)
- **Tujuan:** Pusat kontrol digital operasional lantai produksi, tracking WO, Defect, Downtime, dan OEE Realtime.

## 2. Mandated Tech Stack (WAJIB, DILARANG GANTI)
- **Backend:** ASP.NET Core 8 (Blazor Server)
- **Frontend UI:** MudBlazor (Tema Dark Mode Industri wajib aktif)
- **Database:** Microsoft SQL Server (via Entity Framework Core Code-First)
- **Realtime:** SignalR (WebSocket)
- **Auth:** ASP.NET Core Identity
- **Reporting:** ClosedXML (Export ke Excel)
- **Architecture Pattern:** Modular Monolith

## 3. Aturan Arsitektur & Struktur Folder
Buat solution bernama `MES.DirectInkJet`. Terdiri dari 4 project **wajib**:
1. `MES.Core`: Hanya berisi Entity (C# class), Enums, Interface. **Dilarang ada dependency ke project lain.**
2. `MES.Infrastructure`: Berisi `MesDbContext` (EF Core), Repository. **Hanya boleh referensi ke MES.Core.**
3. `MES.Application`: Berisi Service (Logic), OEE Engine, DTO. **Boleh referensi ke Core & Infrastructure.**
4. `MES.Server`: Blazor UI, SignalR Hub, Program.cs. **Boleh referensi ke Application.**

## 4. Spesifikasi Database (Entity & Field)
Gunakan Guid sebagai Primary Key. Buat class-class berikut di `MES.Core`:

### A. Master Data
- **ProductionLine:** Id, LineCode (string), LineName (string), Status (Enum: Idle/Running/Down)
- **Product:** Id, ProductCode (string), ProductName (string), IdealCycleTimeSeconds (double) *Wajib ada untuk OEE*
- **DefectCode:** Id, Code (string), Description (string)
- **DowntimeCode:** Id, Code (string), Description (string)

### B. Transaksional
- **WorkOrder:** Id, WONumber (string), ProductId (FK), LineId (FK), TargetQty (int), GoodQty (int), RejectQty (int), Status (Enum: Draft/Released/Running/Hold/Complete), DueDate (DateTime)
- **ProductionResult:** Id, WorkOrderId (FK), GoodQty (int), RejectQty (int), InputTime (DateTime), OperatorId (string)
- **QcResult:** Id, WorkOrderId (FK), DefectCode (string), DefectQty (int), Notes (string), InputTime (DateTime)
- **Downtime:** Id, LineId (FK), StartTime (DateTime), EndTime (DateTime? nullable), RootCause (string), ActionTaken (string)

### C. System
- **ActivityLog:** Id, UserName (string), Action (string), Detail (string), Timestamp (DateTime)
- *(Identity tables otomatis di-handle oleh EF Core Identity)*

## 5. Rumus OEE Engine (Jantung Sistem)
Buat class `ProductionService` di Application layer. Logika OEE **wajib** mengikuti ini:

- **Planned Production Time** = 28800 detik (Asumsi standar 8 jam/shift).
- **Operating Time** = Planned Time - Total Downtime (dalam detik).
- **Total Output** = GoodQty + RejectQty.

**Kalkulasi:**
1. `Availability = Operating Time / Planned Production Time`
2. `Performance = (IdealCycleTimeSeconds * Total Output) / Operating Time`
3. `Quality = GoodQty / Total Output`
4. `OEE = Availability * Performance * Quality`

*Catatan: Jika pembagi = 0, return 0 agar tidak DividedByZero exception.*

## 6. Arsitektur Realtime (SignalR Flow)
- **Hub:** Buat class `ProductionHub` turunan dari `Hub`.
- **DTO:** Buat `LineDashboardDto` (LineId, LineCode, OEE, Availability, Performance, Quality, TotalGood, TotalTarget).
- **Trigger:** Setiap kali Service `InputProduction`, `InputQcDefect`, atau `InputDowntime` dipanggil, hitung ulang OEE lalu panggil `_hubContext.Clients.All.SendAsync("ReceiveProductionUpdate", dto)`.
- **Listener:** Dashboard wajib inject `HubConnection`, listen event `ReceiveProductionUpdate`, dan update UI state.

## 7. Spesifikasi UI/UX (MudBlazor)
- **Layout:** Gunakan `<MudThemeProvider IsDarkMode="true" />`. Sidebar kiri menggunakan `<MudDrawer>` untuk navigasi.
- **Live Dashboard:** Menggunakan `<MudGrid>` berisi card `<MudPaper>` per Line. Angka OEE wajib pakai `<MudText Typo="Typo.h1">` agar terbaca dari jarak jauh. Warna OEE: >=85% Hijau, >=60% Kuning, <60% Merah.
- **Input Forms:** Wajib gunakan `<MudSelect>`, `<MudNumericField>`, `<MudDatePicker>`. Validasi wajib diisi tidak boleh pakai alert JS native, gunakan `<MudAlert>`.
- **Tabel Master:** Gunakan `<MudTable>`. Tambah/Edit menggunakan `<MudDialog>`.

## 8. Role-Based Access Control (RBAC)
Seed 6 Role di Program.cs: Admin, Supervisor, Leader, Operator, QC, Engineering.

Aturan Akses Halaman:
- `/login` : Public
- `/live-dashboard` : Semua Role
- `/supervisor/workorders` : Hanya Supervisor, Admin (Attribute `[Authorize(Roles="Supervisor,Admin")]`)
- `/operator/input` : Hanya Operator
- `/qc/input` : Hanya QC
- `/engineering/downtime` : Hanya Engineering
- `/admin/*` : Hanya Admin
- Sidebar NavMenu: Hanya tampilkan link sesuai Role user yang login.

## 9. Fase Eksekusi (STRICT ORDER)
Lakukan rebuild-mu mengikuti urutan ini. **Dilarang lanjut Phase sebelum Phase sebelumnya 100% jalan.**

### Phase 1: Scaffolding & Database
- [ ] Buat Solution & 4 Project. Setup reference antar project.
- [ ] Install NuGet Packages.
- [ ] Buat semua Entity di MES.Core.
- [ ] Setup `MesDbContext` di Infrastructure dengan `IdentityDbContext`.
- [ ] Jalankan EF Migrations. Pastikan database terbentuk di SQL Server.

### Phase 2: Auth & Shell
- [ ] Setup Blazor (MudBlazor, Auth, SignalR) di `Program.cs`.
- [ ] Buat `MainLayout.razor` (Dark Mode, Drawer, MainContent).
- [ ] Buat `NavMenu.razor` (Hardcode dulu semua link).
- [ ] Buat `Login.razor` menggunakan `SignInManager`.
- [ ] Seed Roles & Admin default di Program.cs.
- [ ] Uji coba: Run app, login sebagai admin, pastikan masuk.

### Phase 3: Master Data (CRUD Standard)
- [ ] Buat halaman Admin untuk CRUD `ProductionLine`.
- [ ] Buat halaman Admin untuk CRUD `Product` (Jangan lupa field IdealCycleTimeSeconds).
- [ ] Buat halaman Admin untuk CRUD `DefectCode` & `DowntimeCode`.
- [ ] Uji coba: Tambah data DIJ-01, Product A, Defect SCR-01 via UI.

### Phase 4: Work Order Lifecycle
- [ ] Buat halaman Supervisor untuk Create WO (Status: Draft).
- [ ] Tambahkan tombol "Release WO" (Ubah status ke Released).
- [ ] Uji coba: Buat WO, Release WO. Cek di Database status berubah.

### Phase 5: Transaksi Inti & OEE Engine
- [ ] Buat `ProductionService` (Application). Implementasi hitung OEE & Method `InputProduction`.
- [ ] Buat halaman Operator untuk pilih WO (yang Released/Running) dan input Good/Reject.
- [ ] Saat Operator submit, pastikan Service menyimpan data, update WO, dan menghitung OEE.
- [ ] Uji coba: Input 100 Good. Debug Service pastikan nilai OEE terhitung.

### Phase 6: Realtime Dashboard
- [ ] Buat `ProductionHub`.
- [ ] Inject `IHubContext` ke `ProductionService`. Panggil SendAsync saat InputProduction.
- [ ] Buat `LiveDashboard.razor`. Setup `HubConnection` di `OnInitializedAsync`.
- [ ] Mapping data DTO ke UI Card (OEE Besar, A/P/Q, Progress Bar).
- [ ] Uji coba: Buka 2 tab. Tab 1 Dashboard, Tab 2 Operator Input. Submit produksi, Dashboard harus auto update tanpa F5.

### Phase 7: Completing The Loop (QC & Downtime)
- [ ] Implementasi `InputQcDefect` di Service (Tambah reject, hitung ulang OEE Quality, Broadcast).
- [ ] Implementasi `InputDowntime` di Service (Kurangi Operating Time, hitung ulang OEE Availability, Broadcast).
- [ ] Buat halaman UI untuk QC Input dan Engineering Input.
- [ ] Uji coba: Input Downtime, lihat Dashboard Availability turun. Input Defect, lihat Dashboard Quality turun.

### Phase 8: Reporting & Security
- [ ] Buat `ReportService` menggunakan ClosedXML.
- [ ] Buat halaman Reports. Pilih tanggal -> Export Excel.
- [ ] Implementasi Javascript interop untuk auto-download file byte[] dari Blazor Server.
- [ ] Pasang `[Authorize]` attribute di setiap halaman sesuai rulenya.
- [ ] Pasang Session Timeout (15 menit idle = auto logout) di Program.cs.
- [ ] Tambahkan ActivityLog penyimpanan di setiap Service Transaction.

## 10. Definition of Done (DoD)
Proyek dianggap selesai 100% jika:
1. Bisa multi-line monitoring (Dashboard menampilkan lebih dari 1 line secara bersamaan dan update independen).
2. OEE berubah secara realtime (Automatic, no refresh) saat ada input dari mesin manapun.
3. Role A tidak bisa mengakses halaman Role B.
4. Laporan Excel bisa diunduh dan isinya akurat.
5. Aplikasi bisa di-deploy ke IIS dan diakses via IP Address dari PC lain di jaringan.

***SELAMAT BERKODING. PATUHI BRIEF INI DAN JANGAN TAMBAHKAN APAPUN SEBELUM SEMUA CHECKLIST TERCEKLIS.***
