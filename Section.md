# Panduan Segment Register x86 (CS, DS, SS, ES, FS, GS)

Penjelasan soal register segmen: apa itu, `CS`/`DS`/`SS` masing-masing buat apa,
kenapa dulu penting banget dan sekarang hampir tidak dipakai, plus hubungannya
dengan section `.text`/`.data`/`.bss` yang sering bikin bingung.

> Catatan istilah penting di depan: **"segment register"** (CS, DS, SS, ...) itu
> BEDA dengan **"section/segmen program"** (`.text`, `.data`, `.bss`). Namanya
> mirip tapi konsepnya tidak sama. Bedanya dijelaskan di [bagian 6](#6-segment-register-vs-section-text-data-bss).

---

## Daftar Isi

1. [Kenapa ada segmen? Masalah asalnya](#1-kenapa-ada-segmen-masalah-asalnya)
2. [Enam segment register dan tugasnya](#2-enam-segment-register-dan-tugasnya)
3. [Real mode: alamat = segmen : offset](#3-real-mode-alamat--segmen--offset)
4. [Protected mode: selector, descriptor, GDT](#4-protected-mode-selector-descriptor-gdt)
5. [64-bit (long mode): segmentasi hampir mati](#5-64-bit-long-mode-segmentasi-hampir-mati)
6. [Segment register vs section (.text/.data/.bss)](#6-segment-register-vs-section-text-data-bss)
7. [FS & GS: pemakaian modern (TLS, per-CPU)](#7-fs--gs-pemakaian-modern-tls-per-cpu)
8. [Contoh kode](#8-contoh-kode)
9. [Cheat Sheet](#9-cheat-sheet)

---

## 1. Kenapa ada segmen? Masalah asalnya

Prosesor 8086 (1978) punya register 16-bit. Register 16-bit cuma bisa menghitung
angka sampai `0xFFFF` = 65.536. Artinya, kalau alamat memori langsung dari
register 16-bit, komputer hanya bisa mengakses **64 KB** memori. Terlalu kecil.

Intel mau bisa mengakses **1 MB** (butuh alamat 20-bit). Solusinya: gabungkan
**dua** register 16-bit untuk membentuk alamat 20-bit. Salah satunya disebut
**segmen** (basis), satunya **offset** (jarak dari basis):

```
alamat fisik = segmen × 16 + offset
```

Register segmen inilah yang menyimpan "basis" tadi. Jadi sejak awal, segment
register adalah **cara memperluas jangkauan alamat**, sekaligus **cara memisahkan
wilayah memori** (kode, data, stack) supaya tidak saling tabrakan.

---

## 2. Enam segment register dan tugasnya

Ada 6 register segmen. Semuanya 16-bit. Tiga yang pertama paling penting.

| Register | Nama panjang        | Menunjuk ke wilayah...            | Dipasangkan otomatis dengan |
|----------|---------------------|----------------------------------|-----------------------------|
| **CS**   | Code Segment        | tempat **kode/instruksi** berada | `EIP` / `RIP`               |
| **DS**   | Data Segment        | tempat **data** default          | akses memori biasa `[...]`  |
| **SS**   | Stack Segment       | tempat **stack** berada          | `ESP`/`EBP` (`push`/`pop`)  |
| **ES**   | Extra Segment       | segmen data tambahan             | operasi string (`movs`,`stos`) |
| **FS**   | (huruf setelah ES)  | segmen serbaguna                 | dipakai OS (lihat bagian 7) |
| **GS**   | (huruf setelah FS)  | segmen serbaguna                 | dipakai OS (lihat bagian 7) |

### CS — Code Segment
Menunjuk ke area kode. Prosesor mengambil instruksi berikutnya dari
`CS:EIP` (segmen CS, offset EIP). Kamu **tidak bisa** menulis `mov cs, ...`
langsung — CS hanya berubah lewat `jmp far`, `call far`, `ret`, atau interupsi.
Ini pengaman supaya alur eksekusi tidak bisa diacak sembarangan.

### DS — Data Segment
Segmen default untuk hampir semua akses data. Saat kamu menulis:

```nasm
mov eax, [angka]
```

secara implisit itu `mov eax, [DS:angka]` — CPU memakai DS sebagai basis.
Kamu jarang menyebut DS karena dia terpasang otomatis.

### SS — Stack Segment
Segmen tempat stack hidup. Setiap `push`, `pop`, `call`, `ret` bekerja di
`SS:ESP`. Register `ESP` dan `EBP` otomatis dipasangkan dengan SS — itulah kenapa
stack dan data dulu bisa dipisah ke wilayah memori berbeda.

### ES, FS, GS — segmen tambahan
`ES` dipakai instruksi string sebagai tujuan (contoh `rep movsb` menyalin dari
`DS:ESI` ke `ES:EDI`). `FS` dan `GS` tidak punya tugas tetap dari perangkat keras
— dibiarkan bebas untuk dipakai sistem operasi (peran modernnya di
[bagian 7](#7-fs--gs-pemakaian-modern-tls-per-cpu)).

---

## 3. Real mode: alamat = segmen : offset

Di **real mode** (mode 8086 asli, dan mode boot awal setiap PC), register segmen
berisi angka basis langsung. Rumusnya:

```
alamat fisik = (segmen << 4) + offset
             = segmen × 16 + offset
```

Contoh `CS = 0x1000`, `EIP = 0x0234`:

```
0x1000 × 16 = 0x10000
0x10000 + 0x0234 = 0x10234   ← alamat fisik instruksi
```

Penulisannya di assembly memakai titik dua:

```nasm
mov ax, [ds:0x0100]     ; ambil data di DS basis + offset 0x0100
jmp 0x1000:0x0234       ; lompat ke CS=0x1000, offset=0x0234
```

Konsekuensi menarik real mode: satu alamat fisik bisa ditulis banyak cara,
karena segmen bertumpang tindih tiap 16 byte. `0x1000:0x0000` dan `0x0FFF:0x0010`
menunjuk alamat fisik yang **sama** (`0x10000`).

Real mode tidak punya proteksi apa pun: program mana pun bisa mengakses seluruh
1 MB. Ini yang diperbaiki protected mode.

---

## 4. Protected mode: selector, descriptor, GDT

Mulai 80286/80386, hadir **protected mode**. Di sini isi register segmen **bukan
lagi angka basis**, melainkan sebuah **selector** — semacam nomor indeks yang
menunjuk ke sebuah tabel deskripsi.

```
┌─ isi register segmen (16-bit) di protected mode ─┐
│  indeks (13 bit)          │ TI │  RPL (2 bit)    │
└───────────────────────────┴────┴─────────────────┘
   ↑ nomor entri di tabel     ↑ tabel mana  ↑ level hak akses
```

- **indeks** → menunjuk baris ke berapa di tabel deskriptor.
- **TI** (Table Indicator) → 0 = GDT (Global Descriptor Table), 1 = LDT (Local).
- **RPL** (Requested Privilege Level) → 0 = kernel, 3 = user (ring).

Tabelnya sendiri, **GDT**, berisi **descriptor** — tiap descriptor menyimpan info
lengkap satu segmen: alamat basis (32-bit), batas/limit (ukuran), dan hak akses
(boleh dieksekusi? boleh ditulis? level ring berapa?).

Jadi saat CPU mengakses `DS:offset`, langkahnya:

```
1. Ambil selector di DS → cari indeksnya di GDT
2. Baca descriptor → dapat basis + limit + izin
3. Cek: offset masih di dalam limit? izin cukup?
4. Kalau lolos → alamat linear = basis + offset
5. Kalau melanggar → CPU melempar exception (General Protection Fault)
```

Inilah asal muasal **proteksi memori**: program user (ring 3) tidak bisa
menyentuh segmen kernel (ring 0), dan tidak bisa menulis ke segmen kode.

Contoh isi selector:
```nasm
mov ax, 0x10        ; indeks 2, GDT, RPL 0  (0x10 = 2<<3)
mov ds, ax          ; muat ke DS
```

### "Flat memory model"

Sistem operasi modern (Linux, Windows) memakai trik: mereka membuat semua
descriptor (CS, DS, SS) punya **basis = 0** dan **limit = maksimum**. Efeknya,
segmentasi jadi "transparan" — `alamat linear = 0 + offset = offset`. Semua
segmen menutupi seluruh ruang alamat yang sama. Proteksi antar-proses lalu
ditangani oleh **paging**, bukan segmentasi.

Itu sebabnya di assembly Linux modern kamu **tidak pernah** menyentuh CS/DS/SS —
semuanya sudah diatur kernel jadi flat, dan kamu tinggal pakai offset biasa.

---

## 5. 64-bit (long mode): segmentasi hampir mati

Di mode 64-bit, Intel/AMD hampir **membuang segmentasi sepenuhnya**:

- **CS, DS, ES, SS** dipaksa basis = 0, limit diabaikan. Isinya praktis tidak
  berpengaruh ke alamat. Kamu tidak bisa lagi bikin segmen data terpisah.
- Alamat murni datar (flat) 64-bit; offset langsung jadi alamat linear.
- **FS dan GS** adalah pengecualian: keduanya **tetap punya basis aktif** dan
  malah jadi makin penting (lihat bawah).

Jadi kalau kamu belajar assembly x86-64, praktis kamu bisa **lupakan CS/DS/SS/ES**
untuk urusan alamat sehari-hari. Yang tersisa relevan cuma FS/GS.

---

## 6. Segment register vs section (.text/.data/.bss)

Ini sumber kebingungan paling umum, jadi kita pisahkan tegas.

| Aspek        | Segment register (CS, DS, SS)           | Section / segmen program (`.text`, `.data`, `.bss`) |
|--------------|------------------------------------------|-----------------------------------------------------|
| Apa ini?     | register di **CPU**                       | pengelompokan di **file program & memori**          |
| Ditentukan   | oleh **perangkat keras + OS**             | oleh **kamu (programmer) & linker**                 |
| Fungsi       | menghitung/melindungi **alamat**          | mengelompokkan isi (kode vs data) + izin memori     |
| Kapan aktif  | saat CPU membaca instruksi/data           | saat linker menyusun & saat OS memuat program       |
| Contoh nilai | `CS=0x08`, `DS=0x10`                       | `.text`, `.data`, `.rodata`, `.bss`                 |

Cara memahami hubungannya:

- **Section** = keputusan **desain program**. Kamu tulis `section .data` untuk
  bilang "kelompokkan variabel ini bersama". Linker menaruhnya di file dan
  memberi izin (kode = R+X, data = R+W).
- **Segment register** = mekanisme **CPU saat berjalan** untuk mengubah
  offset jadi alamat nyata dan mengecek izin.

Di sistem modern (flat model), keduanya nyaris tidak berinteraksi langsung:
DS/CS/SS semua basis 0, jadi ketika instruksimu di `.text` mengakses variabel di
`.data`, CPU memakai DS (basis 0) + offset variabel — dan offset itu ditentukan
oleh di mana linker menaruh section `.data`. Section mengatur *tata letak*;
segment register mengatur *penerjemahan alamat*.

Analogi: **section** seperti kamu mengelompokkan barang ke kotak berlabel
"dapur", "kamar", "gudang" saat pindahan. **Segment register** seperti alamat
rumah + aturan siapa boleh masuk ruangan mana. Dua hal berbeda yang kebetulan
sama-sama soal "wilayah".

---

## 7. FS & GS: pemakaian modern (TLS, per-CPU)

Karena CS/DS/SS jadi tidak berguna di 64-bit, FS dan GS "dipungut" OS untuk tugas
baru yang penting. Basisnya diisi lewat MSR khusus (`wrfsbase`/`wrgsbase` atau
lewat kernel), lalu dipakai sebagai **penunjuk cepat ke struktur data spesial**:

- **FS → Thread-Local Storage (TLS).** Tiap thread punya variabel sendiri
  (misalnya `errno` di C, atau variabel `__thread`). FS menunjuk ke blok TLS
  milik thread yang sedang berjalan. Akses `mov eax, [fs:0x28]` (Windows: stack
  canary/TEB; Linux: TLS) mengambil data milik thread ini tanpa perlu tahu thread
  ID.
- **GS → data per-CPU / per-thread di kernel.** Kernel Linux memakai GS untuk
  menunjuk struktur "CPU ini" (per-CPU data). Windows memakai GS di user mode
  untuk menunjuk TEB, dan di kernel untuk KPCR.

Contoh khas (Linux x86-64, ambil stack canary dari TLS):

```nasm
mov rax, [fs:0x28]      ; ambil stack guard milik thread saat ini
```

Ini satu-satunya tempat kamu masih rutin menulis prefix segmen di kode 64-bit
modern. Sisanya (`[cs:...]`, `[ds:...]`) praktis tidak pernah muncul.

---

## 8. Contoh kode

### Real mode: pakai segmen : offset eksplisit

```nasm
; Real mode 16-bit (misalnya kode bootloader)
    mov ax, 0x07C0      ; segmen tempat BIOS memuat boot sector
    mov ds, ax          ; DS = 0x07C0  (tidak bisa langsung mov ds, imm)
    mov si, pesan       ; offset ke string, relatif ke DS
    ; sekarang [si] membaca dari alamat fisik 0x07C0*16 + si
```

> Perhatikan: `mov ds, 0x07C0` **tidak sah** — register segmen tidak bisa diisi
> nilai langsung. Harus lewat register umum dulu (`ax`), baru `mov ds, ax`.

### Protected/flat mode: kamu tidak menyentuh segmen sama sekali

```nasm
; Linux 32-bit — CS/DS/SS sudah diatur kernel (flat), jadi cukup offset biasa
section .data
    angka   dd  42

section .text
    global _start
_start:
    mov eax, [angka]    ; implisit [ds:angka], tapi DS basis 0 → langsung offset
    ; ... tidak ada urusan dengan CS/DS/SS di sini ...
```

### 64-bit: hanya FS/GS yang muncul

```nasm
; Linux x86-64
    mov rax, [fs:0]     ; menunjuk ke struct TLS milik thread ini
    mov rbx, [angka]    ; data biasa — tanpa prefix segmen, flat
```

---

## 9. Cheat Sheet

### Enam segment register

```
CS   Code Segment    → kode; dipasangkan dgn EIP/RIP; ubah via jmp/call far
DS   Data Segment    → data default; dipakai [ ... ] biasa
SS   Stack Segment   → stack; dipasangkan dgn ESP/EBP; push/pop/call/ret
ES   Extra Segment   → tujuan operasi string (movs/stos)
FS   (bebas)         → modern: Thread-Local Storage (TLS)
GS   (bebas)         → modern: data per-CPU / TEB
```

### Tiga mode, tiga arti isi register segmen

```
Real mode        isi = basis langsung;   alamat = segmen×16 + offset
Protected mode   isi = selector;         alamat = descriptor.basis + offset (+ cek izin)
Long mode (64)   CS/DS/SS/ES basis 0 (mati); FS/GS tetap punya basis aktif
```

### Rumus alamat

```
Real mode:       fisik  = (segmen << 4) + offset
Protected mode:  linear = GDT[selector].basis + offset
Flat model:      linear = 0 + offset = offset
```

### Aturan yang gampang lupa

```
- mov cs, ...            → TERLARANG (CS hanya lewat jmp/call/ret far/interupsi)
- mov ds, 0x10           → SALAH (harus lewat register umum dulu)
- di Linux modern        → jangan sentuh CS/DS/SS, sudah flat
- di 64-bit              → hanya FS/GS yang masih relevan untuk alamat
- register segmen 16-bit → walau di mode 32/64-bit tetap 16-bit
```

### Segment register vs section — bedakan!

```
Segment register (CS/DS/SS)  = mekanisme CPU untuk alamat + proteksi (runtime)
Section (.text/.data/.bss)   = pengelompokan isi program (kamu + linker)
```
