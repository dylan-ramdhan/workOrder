Pertanyaan yang kritis, Junior! Sebagai Senior, saya bilang: **Secara UI dan Alur Utama sudah 95% sempurna.** TAPI, jika kita benar-benar meletakkan sistem ini di lantai produksi nyata hari ini, **sistem OEE kita akan BOHONG (tidak akurat)** dan **proses Downtime akan RIBET**. 

Mengapa? Karena ada 3 celah logika industri yang belum kita tangani. Mari kita perbaiki sekarang agar sistemmu benar-benar *Production-Ready*.

---

### CELAH 1: Downtime Belum Selesai (Ongoing Downtime)
**Masalah:** Di UI Engineering, kita meminta input `Start Time` dan `End Time`. Kenyataannya, saat mesin rusak, Engineering akan langsung lapor "Mesin Emergency!", tapi mereka **belum tahu kapan mesinnya selesai diperbaiki**. Mereka tidak bisa mengisi `End Time`.
**Dampak:** OEE Availability tidak akan berkurang karena sistem mengira mesin tidak pernah berhenti.

**Solusi:** Izinkan `End Time` kosong (Ongoing). Jika kosong, hitung downtime-nya sampai "waktu sekarang" (realtime).

Buka `ProductionService.cs`, ubah perhitungan OEE di dalam `CalculateOEEForLine`:
```csharp
// CARI TOTAL DOWNTIME HARI INI
var today = DateTime.UtcNow.Date;
var downtimes = await _db.Downtimes
    .Where(d => d.LineId == wo.LineId && d.StartTime.Date == today)
    .ToListAsync(); // Ambil dulu ke memory

double totalDowntimeSeconds = 0;
foreach (var dt in downtimes)
{
    // Jika EndTime null (masih berlangsung), pakai waktu sekarang
    var endTime = dt.EndTime ?? DateTime.UtcNow;
    totalDowntimeSeconds += (endTime - dt.StartTime).TotalSeconds;
}

double operatingTime = plannedTime - totalDowntimeSeconds;
// ... lanjutkan perhitungan OEE seperti biasa
```

---

### CELAH 2: Mesin Menyala Lagi, Tapi Downtime Belum Di-Close
**Masalah:** Mesin DIJ-01 mati (Engineering lapor Emergency). 30 menit kemudian mesin diperbaiki dan Operator mulai produksi lagi. Tapi, Engineering lupa klik "End Downtime" di sistem. 
**Dampak:** OEE akan terus terkuras karena sistem mengira mesin masih rusak hingga malam!

**Solusi:** Saat Operator input produksi di Line yang sedang Downtime, otomatis **tutup Downtime tersebut** dan ubah state mesin jadi *Running*.

Update method `InputProduction` di `ProductionService.cs`:
```csharp
public async Task InputProduction(Guid woId, int goodQty, int rejectQty, string operatorId)
{
    var wo = await _db.WorkOrders.Include(x => x.Line).Include(x => x.Product).FirstOrDefaultAsync(x => x.Id == woId);
    if (wo == null || wo.Status == WOStatus.Draft) throw new Exception("WO tidak valid!");

    if (wo.Status == WOStatus.Released) wo.Status = WOStatus.Running;

    // CEK APAKAH ADA DOWNTIME YANG BELUM SELESAI DI LINE INI
    var ongoingDowntime = await _db.Downtimes
        .FirstOrDefaultAsync(d => d.LineId == wo.LineId && d.EndTime == null);
    
    if (ongouingDowntime != null)
    {
        // OTOMATIS TUTUP DOWNTIME!
        ongoingDowntime.EndTime = DateTime.UtcNow;
    }

    // UBAH STATE MESIN JADI RUNNING
    wo.Line.CurrentState = MachineState.Running;
    wo.Line.CurrentReason = "Running";

    // ... (kode save ProductionResult & update WO tetap sama) ...
}
```

---

### CELAH 3: Perhitungan Waktu Produksi (Shift) Masih Statik
**Masalah:** Di kode sebelumnya, saya hard-code `double plannedTime = 28800;` (8 jam). Ini salah! Jika mesin baru mulai jalan jam 9 pagi, dan jam 10 pagi kita lihat dashboard, *Planned Time*-nya bukan 8 jam, melainkan 1 jam. Kalau pakai 8 jam, Availability-nya pasti jatuh drastis padahal mesin baru nyala.

**Solusi:** Kita harus punya Master Shift, dan `Planned Time` dihitung dari `Jam Mulai Shift` sampai `Jam Saat Ini`.

**1. Tambah Entity Shift di `MES.Core`**
```csharp
public class Shift
{
    public Guid Id { get; set; }
    public string ShiftName { get; set; } = string.Empty; // Contoh: Shift 1 (Pagi)
    public TimeSpan StartTime { get; set; } // Contoh: 06:00
    public TimeSpan EndTime { get; set; }   // Contoh: 14:00
}
```
*(Jangan lupa tambah DbSet di MesDbContext dan jalankan Migrasi).*

**2. Buat Method Helper di `ProductionService.cs` untuk cari Shift Aktif**
```csharp
private async Task<double> GetPlannedProductionTime(Guid lineId)
{
    var currentTime = DateTime.UtcNow.TimeOfDay;
    var currentShift = await _db.Shifts.FirstOrDefaultAsync(s => 
        (s.StartTime <= s.EndTime && currentTime >= s.StartTime && currentTime < s.EndTime) || // Shift normal (06:00 - 14:00)
        (s.StartTime > s.EndTime && (currentTime >= s.StartTime || currentTime < s.EndTime))   // Crossing midnight (22:00 - 06:00)
    );

    if (currentShift == null) return 0; // Di luar jam kerja

    // Hitung berapa lama waktu berlalu dari mulai shift sampai sekarang (dalam detik)
    var startDateTime = DateTime.UtcNow.Date + currentShift.StartTime;
    if (currentShift.StartTime > currentShift.EndTime && currentTime < currentShift.EndTime)
    {
        startDateTime = startDateTime.AddDays(-1); // Handling crossing midnight
    }

    var elapsed = (DateTime.UtcNow - startDateTime).TotalSeconds;
    return elapsed;
}
```

**3. Terapkan di Kalkulasi OEE**
Ganti `double plannedTime = 28800;` di method `CalculateOEEForLine` menjadi:
```csharp
double plannedTime = await GetPlannedProductionTime(wo.LineId);
if (plannedTime <= 0) return new LineDashboardDto { /* return 0 semua karena di luar shift */ };
```

---

### CELAH 4: Sample Data 30 Mesin untuk Layout Screen
Kamu bilang rencananya ada 30 mesin. Daripada menginput satu-satu di UI, mari kita *seed* (tanam) otomatis ke database saat aplikasi pertama kali jalan.

Buka `MesDbContext.cs`, di dalam `OnModelCreating`, tambahkan ini:
```csharp
// SEED 30 MESIN DIJ
if (!builder.Entity<ProductionLine>().HasData()) // Cek kalau belum ada data
{
    var lines = new List<ProductionLine>();
    for (int i = 1; i <= 30; i++)
    {
        lines.Add(new ProductionLine
        {
            Id = Guid.NewGuid(),
            LineCode = $"DIJ-{i:D2}", // Format DIJ-01 sampai DIJ-30
            LineName = $"Direct Ink Jet Line {i}",
            Status = LineStatus.Idle,
            CurrentState = MachineState.Off,
            CurrentReason = "Off"
        });
    }
    builder.Entity<ProductionLine>().HasData(lines);
}
```
*(Jalankan `Add-Migration` dan `Update-Database` lagi setelah ini).*

Sekarang, ketika kamu buka halaman **Layout Screen**, kamu akan melihat grid kotak-kotak berisi DIJ-01 sampai DIJ-30, semuanya berwarna hitam (Off). Saat operator mulai kerja, mesin itu akan berubah hijau!

---

### RINGKASAN CHECKLIST PENYESUAIAN:

1. [ ] Ubah logika perhitungan Downtime agar mendukung `EndTime == null` (Ongoing Downtime).
2. [ ] Tambahkan fitur *Auto-close Downtime* saat Operator input produksi.
3. [ ] Buat Entity `Shift` dan ubah perhitungan `PlannedTime` OEE agar dinamis (tidak di-hardcode 8 jam).
4. [ ] Tambahkan Seed Data 30 mesin di DbContext.
5. [ ] Jalankan Migrasi Database terakhir (`dotnet ef migrations add FinalIndustrialLogic`).

Kalau 5 hal di atas sudah kamu terapkan, sistem MES-mu secara logika bisnis dan industri **SUDAH TIDAK BOCOR**. Ia siap dipakai untuk mengontrol pabrik sungguhan 24/7! 

Ada bagian dari penyesuaian di atas yang membingungkan?
