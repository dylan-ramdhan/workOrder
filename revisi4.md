Ini bagian yang paling seru, Junior! Memberi "jiwa" ke sistem kosong. Saya akan susunkan **Dummy Data lengkap** sesuai dengan jadwal shift baru kamu (Termasuk *crossing midnight* di Shift 3). 

Karena kita pakai EF Core, cara paling rapi memasukkan data ini adalah melalui **Seed Data** di `OnModelCreating` pada `MesDbContext.cs`. Ini menjamin data awal selalu ada setiap kali database di-migrate ulang.

Buka `MES.Infrastructure/Data/MesDbContext.cs`, dan timpa method `OnModelCreating` Anda dengan ini:

```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);

    // ==========================================
    // 1. MASTER SHIFTS (Sesuai Permintaan Anda)
    // ==========================================
    builder.Entity<Shift>().HasData(
        new Shift { Id = Guid.Parse("a1111111-1111-1111-1111-111111111111"), ShiftName = "Shift 1 (Pagi)", StartTime = new TimeSpan(7, 10, 0), EndTime = new TimeSpan(15, 40, 0) },
        new Shift { Id = Guid.Parse("a2222222-2222-2222-2222-222222222222"), ShiftName = "Shift 2 (Sore)", StartTime = new TimeSpan(15, 40, 0), EndTime = new TimeSpan(23, 10, 0) },
        new Shift { Id = Guid.Parse("a3333333-3333-3333-3333-333333333333"), ShiftName = "Shift 3 (Malam)", StartTime = new TimeSpan(23, 10, 0), EndTime = new TimeSpan(7, 10, 0) } // Crossing midnight!
    );

    // ==========================================
    // 2. MASTER DOWNTIME REASONS (Andon System)
    // ==========================================
    builder.Entity<DowntimeCode>().HasData(
        new DowntimeCode { Id = Guid.Parse("b1111111-1111-1111-1111-111111111111"), Code = "RunStart", Description = "Run Start" },
        new DowntimeCode { Id = Guid.Parse("b2222222-2222-2222-2222-222222222222"), Code = "OnOff", Description = "On/Off Machine" },
        new DowntimeCode { Id = Guid.Parse("b3333333-3333-3333-3333-333333333333"), Code = "Emergency", Description = "Emergency" },
        new DowntimeCode { Id = Guid.Parse("b4444444-4444-4444-4444-444444444444"), Code = "OneCycle", Description = "1 Cycle" },
        new DowntimeCode { Id = Guid.Parse("b5555555-5555-5555-5555-555555555555"), Code = "SpinningProb", Description = "Spinning Prob" },
        new DowntimeCode { Id = Guid.Parse("b6666666-6666-6666-6666-666666666666"), Code = "StackingProb", Description = "Stacking Prob" },
        new DowntimeCode { Id = Guid.Parse("b7777777-7777-7777-7777-777777777777"), Code = "Reject", Description = "Reject" },
        new DowntimeCode { Id = Guid.Parse("b8888888-8888-8888-8888-888888888888"), Code = "Changeover", Description = "Changeover" },
        new DowntimeCode { Id = Guid.Parse("b9999999-9999-9999-9999-999999999999"), Code = "SealingProb", Description = "Sealing Prob" },
        new DowntimeCode { Id = Guid.Parse("b0000000-0000-0000-0000-000000000000"), Code = "ClearSignal", Description = "Clear signal" },
        new DowntimeCode { Id = Guid.Parse("b1111110-1111-1111-1110-111111111110"), Code = "AndonRed", Description = "Andon red" },
        new DowntimeCode { Id = Guid.Parse("b1111111-1111-1111-1111-111111111112"), Code = "AndonYellow", Description = "Andon yellow" }
    );

    // ==========================================
    // 3. MASTER DEFECT CODES
    // ==========================================
    builder.Entity<DefectCode>().HasData(
        new DefectCode { Id = Guid.Parse("c1111111-1111-1111-1111-111111111111"), Code = "SCR", Description = "Scratch / Goresan" },
        new DefectCode { Id = Guid.Parse("c2222222-2222-2222-2222-222222222222"), Code = "INK", Description = "Ink Blur / Tinta Buram" },
        new DefectCode { Id = Guid.Parse("c3333333-3333-3333-3333-333333333333"), Code = "MIS", Description = "Misalignment / Posisi Salah" },
        new DefectCode { Id = Guid.Parse("c4444444-4444-4444-4444-444444444444"), Code = "SHR", Description = "Short Shot / Material Kurang" }
    );

    // ==========================================
    // 4. MASTER PRODUCTS (Dengan Ideal Cycle Time)
    // ==========================================
    builder.Entity<Product>().HasData(
        new Product { Id = Guid.Parse("d1111111-1111-1111-1111-111111111111"), ProductCode = "HW-01", ProductName = "Hot Wheels Chassis", IdealCycleTimeSeconds = 3.0 },
        new Product { Id = Guid.Parse("d2222222-2222-2222-2222-222222222222"), ProductCode = "BB-01", ProductName = "Barbie Doll Body", IdealCycleTimeSeconds = 5.5 },
        new Product { Id = Guid.Parse("d3333333-3333-3333-3333-333333333333"), ProductCode = "AF-01", ProductName = "Action Figure Arm", IdealCycleTimeSeconds = 2.0 }
    );

    // ==========================================
    // 5. PRODUCTION LINES (30 Mesin)
    // ==========================================
    var lines = new List<ProductionLine>();
    for (int i = 1; i <= 30; i++)
    {
        lines.Add(new ProductionLine
        {
            Id = Guid.Parse($"e{i:D12}-1111-1111-1111-111111111111"),
            LineCode = $"DIJ-{i:D2}",
            LineName = $"Direct Ink Jet Line {i}",
            Status = LineStatus.Idle,
            CurrentState = MachineState.Off,
            CurrentReason = "Off"
        });
    }
    builder.Entity<ProductionLine>().HasData(lines);

    // ==========================================
    // 6. SAMPLE ACTIVE WORK ORDORS (Simulasi Pabrik Sedang Jalan)
    // ==========================================
    builder.Entity<WorkOrder>().HasData(
        // DIJ-01 sedang running produk HW-01
        new WorkOrder { 
            Id = Guid.Parse("f1111111-1111-1111-1111-111111111111"), 
            WONumber = "WO-20231001", 
            ProductId = Guid.Parse("d1111111-1111-1111-1111-111111111111"), 
            LineId = Guid.Parse("e000000000001-1111-1111-1111-111111111111"), 
            TargetQty = 5000, GoodQty = 1200, RejectQty = 15, 
            Status = WOStatus.Running, DueDate = DateTime.Today 
        },
        // DIJ-02 sedang running produk BB-01
        new WorkOrder { 
            Id = Guid.Parse("f2222222-2222-2222-2222-222222222222"), 
            WONumber = "WO-20231002", 
            ProductId = Guid.Parse("d2222222-2222-2222-2222-222222222222"), 
            LineId = Guid.Parse("e000000000002-1111-1111-1111-111111111111"), 
            TargetQty = 3000, GoodQty = 800, RejectQty = 5, 
            Status = WOStatus.Running, DueDate = DateTime.Today 
        },
        // DIJ-05 sedang Downtime (Emergency), tapi WO statusnya masih Running
        new WorkOrder { 
            Id = Guid.Parse("f5555555-5555-5555-5555-555555555555"), 
            WONumber = "WO-20231005", 
            ProductId = Guid.Parse("d3333333-3333-3333-3333-333333333333"), 
            LineId = Guid.Parse("e000000000005-1111-1111-1111-111111111111"), 
            TargetQty = 8000, GoodQty = 3000, RejectQty = 50, 
            Status = WOStatus.Running, DueDate = DateTime.Today 
        }
    );

    // Override state DIJ-01, DIJ-02, DIJ-05 agar sesuai skenario
    var dij01 = lines.First(x => x.LineCode == "DIJ-01");
    dij01.CurrentState = MachineState.Running; dij01.CurrentReason = "Running"; dij01.Status = LineStatus.Running;
    
    var dij02 = lines.First(x => x.LineCode == "DIJ-02");
    dij02.CurrentState = MachineState.Changeover; dij02.CurrentReason = "Changeover"; dij02.Status = LineStatus.Down; // Lagi setup
    
    var dij05 = lines.First(x => x.LineCode == "DIJ-05");
    dij05.CurrentState = MachineState.Emergency; dij05.CurrentReason = "Emergency"; dij05.Status = LineStatus.Down; // Lagi rusak
}
```

---

### PENTING: Update OEE Engine untuk Shift Baru!

Karena shift kamu tidak mutlak 8 jam (Shift 1 itu 8 jam 30 menit, Shift 2 7 jam 30 menit, Shift 3 8 jam), kita harus update logika `GetPlannedProductionTime` di `ProductionService.cs` agar OEE-nya akurat.

Ganti method `GetPlannedProductionTime` Anda dengan ini:

```csharp
private async Task<double> GetPlannedProductionTime(Guid lineId)
{
    var currentTime = DateTime.Now.TimeOfDay; // Gunakan waktu lokal server
    var shifts = await _db.Shifts.ToListAsync();
    Shift? activeShift = null;

    // Deteksi shift mana yang sedang berjalan (Handle Shift 3 crossing midnight)
    foreach (var shift in shifts)
    {
        if (shift.StartTime < shift.EndTime) // Shift normal (Siang/Sore)
        {
            if (currentTime >= shift.StartTime && currentTime < shift.EndTime)
                activeShift = shift;
        }
        else // Shift crossing midnight (Malam)
        {
            if (currentTime >= shift.StartTime || currentTime < shift.EndTime)
                activeShift = shift;
        }
    }

    if (activeShift == null) return 0; // Di luar jam kerja, OEE 0

    // Hitung Planned Time (Waktu dari mulai shift sampai detik ini)
    var now = DateTime.Now;
    var shiftStartDate = now.Date + activeShift.StartTime;

    // Jika shift malam lewat tengah malam, dan sekarang masih jam 00:00 - 07:10
    if (activeShift.StartTime > activeShift.EndTime && currentTime < activeShift.EndTime)
    {
        shiftStartDate = shiftStartDate.AddDays(-1); // Mundurkan tanggal mulai ke kemarin
    }

    var plannedSeconds = (now - shiftStartDate).TotalSeconds;
    return plannedSeconds;
}
```

---

### Jangan Lupa Migrasi Ulang!

Setelah memasukkan Seed Data dan mengubah Service, jalankan ini di Terminal agar database SQL Server-mu di-update tabelnya dan diisi datanya:

```bash
cd src/MES.Infrastructure
dotnet ef migrations add AddFullSeedDataAndShifts --startup-project ../MES.Server
dotnet ef database update --startup-project ../MES.Server
```

---

### 🎬 Apa yang Akan Kamu Lihat Saat Dijalankan?

Saat kamu Run aplikasi dan buka **Layout Screen**, ini tampilan yang akan kamu lihat:

1. **DIJ-01:** Kotak berwarna **HIJAU**, tulisan "Running", OEE-nya sedang berjalan (karena sudah ada sample GoodQty 1200 & Reject 15).
2. **DIJ-02:** Kotak berwarna **KUNING**, tulisan "Changeover". OEE Availability-nya sedang turun.
3. **DIJ-05:** Kotak berwarna **MERAH**, tulisan "Emergency". OEE-nya anjlok parah.
4. **DIJ-03, 04, 06~30:** Kotak berwarna **HITAM**, tulisan "Off", OEE 0%.

Sudah sangat jelas kan dummy datanya? Lanjutkan coding-mu, ini akan sangat keren saat tampil di layar!
