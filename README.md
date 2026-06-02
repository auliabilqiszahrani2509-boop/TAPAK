## 👥 Anggota Kelompok

1. Aulia Bilqis Zahrani 06
2. Muhammad Hubbi El Fairuz 23
3. Rafa Zulfahmi 28

---

# 📘 README – Database Trigger Sistem Absensi (TAPAK)

## 🏫 Tentang Aplikasi TAPAK
**TAPAK** adalah aplikasi sistem informasi absensi sekolah berbasis *desktop* yang dibangun menggunakan bahasa pemrograman C# dan *database* SQL Server. Aplikasi ini dirancang untuk memodernisasi pengelolaan kedisiplinan siswa melalui dua lapisan pencatatan: **Absensi Harian** (diinput di pagi hari oleh Wali Kelas) dan **Absensi Mata Pelajaran** (diinput di setiap sesi jam pelajaran oleh Guru Mapel). 

Salah satu keunggulan utama TAPAK adalah fitur sinkronisasi dan pelaporan otomatisnya. Jika seorang siswa hadir di pagi hari namun terdeteksi bolos atau tidak ada di kelas pada jam mata pelajaran tertentu, sistem akan langsung mengirimkan notifikasi pelanggaran ke dasbor Wali Kelas sebagai bahan evaluasi dan tindak lanjut.

Dokumen ini secara khusus menjelaskan implementasi **trigger pada SQL Server** yang bekerja di belakang layar (*background automation*) untuk menghidupkan fitur-fitur pada sistem TAPAK.

---

## 📊 Struktur Database (ERD)
Berikut adalah Entity Relationship Diagram (ERD) yang mendasari sistem absensi TAPAK:

<img width="1547" height="1244" alt="ERD ABSENSI drawio" src="https://github.com/user-attachments/assets/cb004f03-dd53-4181-ad7a-8ef9c05ef11c" />


---

## 🎯 Tujuan

Trigger digunakan untuk:

-   Mencatat **log aktivitas perubahan** data absensi  
-   Mendeteksi **pelanggaran kehadiran** (seperti bolos jam mapel)  
-   Meneruskan **notifikasi otomatis** ke Wali Kelas  

---

## 🧩 Trigger yang Digunakan

## 🔹 1. `trg_AbsensiMapel_Log`

**Tipe:** `AFTER INSERT, UPDATE, DELETE`

### ✅ Detail Trigger:

- **Tabel Terkait:** `Absensi_Mapel` (Absensi per Mata Pelajaran)
- **Status:** Aktif (Enabled)
- **Fungsi Utama:** **Pencatatan Audit Trail / Riwayat Perubahan (Logging)**
- **Deskripsi:**
  Setiap kali ada perubahan data (Insert, Update, atau Delete) pada presensi mata pelajaran yang diinput oleh Guru Mapel, trigger ini secara otomatis merekam histori perubahan tersebut ke dalam tabel log (seperti `Log_Absensi_Mapel`). Hal ini menjamin transparansi serta mencegah adanya manipulasi data absensi jam pelajaran yang tidak bisa dilacak.

### 💻 Kode SQL:

```sql
CREATE TRIGGER trg_AbsensiMapel_Log
ON Absensi_Mapel
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- Menyimpan riwayat perubahan ke tabel log historis
    INSERT INTO Log_Absensi_Mapel (Id_absensi_mapel, Waktu_Ubah, Aksi, Diubah_Oleh)
    SELECT Id_absensi_mapel, GETDATE(), 'MODIFIED', SYSTEM_USER 
    FROM inserted;
END;
GO

```

## 🔹 2. `trg_AbsensiHarian_Log`

**Tipe:** `AFTER INSERT, UPDATE, DELETE`

### ✅ Detail Trigger:

* **Tabel Terkait:** `Absensi_Harian` (Absensi Utama / Harian)
* **Status:** Aktif (Enabled)
* **Fungsi Utama:** **Pengawasan Ketat Data Presensi Utama**
* **Deskripsi:**
Absensi harian (pagi) adalah data acuan (*baseline*) yang sangat krusial. Trigger ini merekam setiap kejadian pembuatan, modifikasi, atau penghapusan data absensi harian siswa. Ini memungkinkan sistem untuk melacak rekam jejak secara presisi, sehingga pihak sekolah bisa mengetahui siapa yang melakukan perubahan status kehadiran siswa di pagi hari.

### 💻 Kode SQL:

```sql
CREATE TRIGGER trg_AbsensiHarian_Log
ON Absensi_Harian
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- Merekam perubahan absensi harian siswa
    INSERT INTO Log_Absensi_Harian (Id_absen_harian, Waktu_Ubah, Diubah_Oleh)
    SELECT Id_absen_harian, GETDATE(), SYSTEM_USER 
    FROM inserted;
END;
GO

```

## 🔹 3. `trg_AbsensiMapel_AutoNotifikasiWalas`

**Tipe:** `AFTER INSERT, UPDATE`

### ✅ Detail Trigger:

* **Tabel Terkait:** `Absensi_Mapel` (Absensi per Mata Pelajaran)
* **Status:** Aktif (Enabled)
* **Fungsi Utama:** **Sinkronisasi Pelanggaran & Notifikasi Otomatis ke Wali Kelas**
* **Deskripsi:**
Ini adalah inti dari logika sinkronisasi aplikasi Tapak. Saat Guru Mapel menginput absensi dan menemukan ketidaksesuaian (siswa berstatus 'Bolos' di jam mapel), trigger ini akan langsung mendeteksinya. Sistem kemudian secara otomatis membuatkan catatan laporan dan menyimpannya ke dalam tabel `Notifikasi_Walas`, yang nantinya akan muncul di Dasbor Wali Kelas yang bersangkutan sebagai bahan tindak lanjut pendisiplinan.

### 💻 Kode SQL:

```sql
CREATE TRIGGER trg_AbsensiMapel_AutoNotifikasiWalas
ON Absensi_Mapel
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Memasukkan data ke antrean notifikasi Wali Kelas jika siswa terdeteksi Bolos
    INSERT INTO Notifikasi_Walas (NIS, Pesan, Waktu_Kejadian, Status_Notifikasi)
    SELECT 
        inserted.NIS, 
        'Peringatan: Siswa terdeteksi bolos/tidak ada di kelas pada jam pelajaran.', 
        GETDATE(), 
        'Belum Dibaca'
    FROM inserted
    WHERE inserted.Status = 'Bolos';
END;
GO

```

---

## 🗂️ Tabel yang Terlibat

* `Absensi_Harian`
* `Absensi_Mapel`
* `Log_Absensi_Harian`
* `Log_Absensi_Mapel`
* `Notifikasi_Walas`

---

## ⚙️ Fitur Otomatisasi

* ✅ Audit log absensi harian & mapel lengkap
* ✅ Auto-deteksi ketidaksesuaian presensi
* ✅ Auto-sinkronisasi laporan ke Dasbor Walas

```

```
