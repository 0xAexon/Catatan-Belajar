# Panduan Register & Section Assembly x86 (NASM, Linux)

Catatan praktis soal: register itu apa, EAX/ECX buat apa, kenapa ada `section`,
kenapa pakai `mov` dan bukan `sub`, plus contoh kode yang bisa langsung dijalankan.

Fokus di **x86 32-bit (i386) dengan sintaks NASM**. Bagian 64-bit disinggung di akhir.

---

## Daftar Isi

1. [Register itu apa?](#1-register-itu-apa)
2. [General Purpose Register](#2-general-purpose-register)
3. [Register Spesial: EIP & EFLAGS](#3-register-spesial-eip--eflags)
4. [Section: `.data`, `.bss`, `.text`](#4-section-data-bss-text)
5. [Kenapa `mov`, kenapa bukan `sub`?](#5-kenapa-mov-kenapa-bukan-sub)
6. [Contoh 1 — Hello World](#6-contoh-1--hello-world)
7. [Contoh 2 — Aritmatika](#7-contoh-2--aritmatika)
8. [Contoh 3 — Loop pakai ECX](#8-contoh-3--loop-pakai-ecx)
9. [Contoh 4 — Fungsi & Stack Frame](#9-contoh-4--fungsi--stack-frame)
10. [Cara Compile & Jalankan](#10-cara-compile--jalankan)
11. [Cheat Sheet](#11-cheat-sheet)
12. [Catatan 64-bit](#12-catatan-64-bit)

---

## 1. Register itu apa?

Register adalah **variabel bawaan CPU**. Ukurannya kecil (32-bit = 4 byte) dan
jumlahnya terbatas (cuma 8 yang umum dipakai), tapi aksesnya jauh lebih cepat
daripada RAM.

```
CPU register  : ~1 siklus clock
Cache L1      : ~4 siklus
RAM           : ~200+ siklus
```

Karena itu di assembly hampir semua kerja dilakukan di register: ambil data dari
memori ke register → olah di register → simpan balik ke memori.

**Penting:** kebanyakan instruksi x86 **tidak bisa memory-to-memory**.

```nasm
mov [hasil], [angka]   ; ❌ ERROR — dua-duanya memori
mov eax, [angka]       ; ✅ memori → register
mov [hasil], eax       ; ✅ register → memori
```

Ini alasan utama kenapa register selalu jadi "perantara".

---

## 2. General Purpose Register

Ada 8 register 32-bit. Namanya diawali `E` (Extended) karena dulunya 16-bit.

| 32-bit | 16-bit | 8-bit (high/low) | Nama panjang       | Kebiasaan dipakai untuk        |
|--------|--------|------------------|--------------------|--------------------------------|
| EAX    | AX     | AH / AL          | Accumulator        | hasil hitung, nilai balik, nomor syscall |
| EBX    | BX     | BH / BL          | Base               | pointer/base address, arg syscall |
| ECX    | CX     | CH / CL          | Counter            | penghitung loop, jumlah shift  |
| EDX    | DX     | DH / DL          | Data               | pasangan EAX di mul/div, I/O port |
| ESI    | SI     | —                | Source Index       | alamat **sumber** saat copy data |
| EDI    | DI     | —                | Destination Index  | alamat **tujuan** saat copy data |
| ESP    | SP     | —                | Stack Pointer      | puncak stack — **jangan diutak-atik sembarangan** |
| EBP    | BP     | —                | Base Pointer       | pangkal stack frame fungsi     |

### Cara baca sub-register

`EAX` bisa diakses sebagian:

```
 31                             0
┌───────────────┬───────┬───────┐
│               │  AH   │  AL   │   ← AL = 8 bit terbawah
│               └───────┴───────┘      AH = 8 bit berikutnya
│                   AX (16 bit) │      AX = 16 bit terbawah
│            EAX (32 bit)       │
└───────────────────────────────┘
```

```nasm
mov eax, 0x12345678
; sekarang: AX = 0x5678, AH = 0x56, AL = 0x78

mov al, 0xFF
; EAX jadi 0x123456FF — hanya 8 bit terbawah yang berubah
```

### "Kebiasaan" vs "kewajiban"

Sebagian besar isi tabel di atas cuma **konvensi** — secara teknis kamu boleh
menghitung pakai ECX atau looping pakai EBX. Tapi ada instruksi yang benar-benar
**memaksa** register tertentu:

| Instruksi          | Register yang dipaksa                              |
|--------------------|----------------------------------------------------|
| `mul` / `div`      | EAX (dan EDX untuk 64-bit hasil / sisa bagi)       |
| `loop`             | ECX (otomatis dikurangi 1 tiap iterasi)            |
| `shl`/`shr` variabel | CL (jumlah geser harus di CL)                    |
| `rep movsb`        | ESI (sumber), EDI (tujuan), ECX (jumlah byte)      |
| `int 0x80` (Linux) | EAX = nomor syscall, EBX/ECX/EDX/ESI/EDI = argumen |

Contoh yang memaksa:

```nasm
mov eax, 10
mov ebx, 3
xor edx, edx        ; EDX wajib dinolkan dulu (jadi bagian atas pembagi)
div ebx             ; hasil bagi → EAX (3), sisa → EDX (1)
```

`div` tidak menerima "hasilnya taruh di register X". Selalu EAX dan EDX. Titik.

### Caller-saved vs callee-saved (konvensi cdecl)

Kalau kamu memanggil fungsi C atau dipanggil oleh C:

- **Boleh dirusak** (caller-saved): `EAX`, `ECX`, `EDX`
- **Wajib dikembalikan** (callee-saved): `EBX`, `ESI`, `EDI`, `EBP`, `ESP`
- **Nilai balik fungsi**: selalu di `EAX`

Ini juga alasan lain kenapa ECX yang dipakai buat loop: dia "murah", boleh hilang.

---

## 3. Register Spesial: EIP & EFLAGS

### EIP — Instruction Pointer

Menyimpan alamat instruksi **berikutnya** yang akan dieksekusi. Tidak bisa
di-`mov` langsung; yang mengubahnya adalah `jmp`, `call`, `ret`, dan `jz`/`jne` dkk.

### EFLAGS — status hasil operasi terakhir

Kumpulan bit yang di-set otomatis oleh instruksi aritmatika/logika:

| Flag | Nama      | Di-set kalau...                    |
|------|-----------|------------------------------------|
| ZF   | Zero      | hasilnya 0                         |
| SF   | Sign      | hasilnya negatif (bit tertinggi 1) |
| CF   | Carry     | ada carry/borrow (unsigned overflow) |
| OF   | Overflow  | overflow signed                    |

Ini **inti dari percabangan**. `cmp` sebenarnya cuma `sub` yang hasilnya dibuang,
yang disimpan cuma flag-nya:

```nasm
cmp eax, 5      ; hitung EAX - 5, buang hasilnya, simpan flag
je  sama        ; lompat kalau ZF=1 (artinya EAX == 5)
jl  lebih_kecil ; lompat kalau SF≠OF
```

Catat baik-baik: **`mov` TIDAK mengubah flag sama sekali.** Fakta ini yang jadi
jawaban pertanyaan nomor 5 nanti.

---

## 4. Section: `.data`, `.bss`, `.text`

Program tidak ditaruh acak di memori. Linker mengelompokkannya jadi beberapa
wilayah, lalu OS memberi **izin akses berbeda** ke tiap wilayah.

| Section   | Isi                             | Izin memori     | Masuk file binary? |
|-----------|---------------------------------|-----------------|--------------------|
| `.text`   | kode/instruksi                  | Read + Execute  | Ya                 |
| `.data`   | variabel dengan nilai awal      | Read + Write    | Ya                 |
| `.rodata` | konstanta, string literal       | Read only       | Ya                 |
| `.bss`    | variabel kosong / belum diisi   | Read + Write    | **Tidak** (cuma ukurannya) |

### Kenapa harus dipisah?

1. **Keamanan.** `.text` tidak boleh ditulisi → penyerang tidak bisa menimpa kode
   program. `.data` tidak boleh dieksekusi (NX bit) → data yang disuntikkan
   penyerang tidak bisa dijalankan sebagai kode.
2. **Hemat ukuran file.** Buffer 1 MB kosong di `.bss` menambah **0 byte** ke file
   executable — yang disimpan cuma catatan "sediakan 1 MB, isi nol". Kalau ditaruh
   di `.data`, file kamu membengkak 1 MB berisi nol semua.
3. **Efisiensi cache.** Kode berdekatan dengan kode, data dengan data.

### Cara deklarasi

**`.data` — ada nilai awal**, pakai `db/dw/dd/dq` (define byte/word/dword/qword):

```nasm
section .data
    huruf   db  'A'                 ; 1 byte
    umur    db  25
    tahun   dw  2026                ; 2 byte
    gaji    dd  5000000             ; 4 byte
    pesan   db  "Halo dunia", 0x0A  ; string + newline
    pjg     equ $ - pesan           ; $ = alamat sekarang; ini konstanta, bukan variabel
    array   dd  10, 20, 30, 40
    kosong  times 10 db 0           ; 10 byte berisi 0
```

**`.bss` — belum ada isi**, pakai `resb/resw/resd/resq` (reserve):

```nasm
section .bss
    buffer  resb 256    ; 256 byte
    angka   resd 1      ; 1 dword
    tabel   resd 100    ; 100 dword = 400 byte
```

**`.text` — kode**:

```nasm
section .text
    global _start       ; ekspor simbol supaya linker menemukan entry point

_start:
    ; instruksi di sini
```

> `$ - pesan` artinya "alamat saat ini dikurangi alamat awal `pesan`" = panjang
> string. Cara ini bikin panjang string otomatis benar walau nanti teksnya diedit.

---

## 5. Kenapa `mov`, kenapa bukan `sub`?

Pertanyaan bagus, dan jawabannya ada tiga lapis.

### Lapis 1: Mereka melakukan hal yang berbeda

```nasm
mov eax, 5      ; EAX = 5           ← menimpa, tidak peduli isi lama
sub eax, 5      ; EAX = EAX - 5     ← butuh isi lama sebagai bahan
```

`mov` = **menyalin**. `sub` = **menghitung**. Kalau kamu mau EAX berisi 5, `sub`
cuma benar kalau EAX kebetulan berisi 10 sebelumnya. `mov` benar selalu.

```nasm
; Mau isi EAX dengan 4 (nomor syscall write):
mov eax, 4      ; ✅ pasti 4
sub eax, 4      ; ❌ hasilnya tergantung isi EAX sebelumnya — bug
```

### Lapis 2: `mov` tidak mengganggu EFLAGS

Ini yang sering tidak disadari pemula:

```nasm
cmp eax, ebx    ; set flag berdasarkan perbandingan
mov ecx, 100    ; ✅ AMAN — flag tidak berubah
je  sama        ; masih membandingkan hasil cmp tadi

cmp eax, ebx
sub ecx, 100    ; ❌ BAHAYA — flag ditimpa hasil pengurangan ini
je  sama        ; sekarang mengecek "ECX-100 == 0", bukan "EAX == EBX"
```

Jadi `mov` bisa disisipkan di mana saja tanpa merusak logika percabangan. `sub`
tidak bisa.

### Lapis 3: Kapan `sub` justru dipakai untuk "set nilai"

Ada satu kasus di mana orang sengaja **tidak** pakai `mov`: **mengosongkan register**.

```nasm
mov eax, 0      ; 5 byte  (B8 00 00 00 00)
sub eax, eax    ; 2 byte  (29 C0)
xor eax, eax    ; 2 byte  (31 C0)  ← ini yang paling umum
```

Ketiganya menghasilkan EAX = 0, tapi:

- `xor eax, eax` **lebih pendek** (hemat 3 byte per pemakaian).
- CPU modern mengenali `xor reg, reg` sebagai *zeroing idiom*: dia tahu hasilnya
  pasti nol tanpa perlu membaca nilai lama, jadi **memutus dependensi** dan bisa
  dieksekusi lebih paralel. `sub eax, eax` juga dikenali, tapi `xor` sudah jadi
  standar tak tertulis.
- Karena `mov eax, 0` tidak mengubah flag sementara `xor` mengubahnya, di
  situasi sensitif-flag kamu tetap harus pakai `mov eax, 0`.

**Kesimpulan praktis:**

| Tujuan                        | Pakai            |
|-------------------------------|------------------|
| Isi register dengan nilai     | `mov`            |
| Kurangi nilai yang sudah ada  | `sub`            |
| Nolkan register (umum)        | `xor reg, reg`   |
| Nolkan register tanpa ganggu flag | `mov reg, 0` |
| Bandingkan tanpa merusak nilai | `cmp` / `test`  |

### Instruksi dasar lain yang sering dipakai

```nasm
mov  eax, ebx       ; salin
add  eax, 10        ; EAX += 10
sub  eax, 3         ; EAX -= 3
inc  eax            ; EAX++
dec  eax            ; EAX--
neg  eax            ; EAX = -EAX
and  eax, 0xFF      ; masking, ambil 8 bit terbawah
or   eax, ebx       ; bitwise OR
xor  eax, eax       ; nolkan
shl  eax, 1         ; geser kiri = kali 2
shr  eax, 1         ; geser kanan = bagi 2
lea  eax, [ebx+4]   ; ambil ALAMAT-nya, bukan isinya
push eax            ; simpan ke stack (ESP -= 4)
pop  eax            ; ambil dari stack (ESP += 4)
```

Beda halus `mov` dan `lea`:

```nasm
mov eax, [angka]    ; EAX = ISI dari variabel angka
lea eax, [angka]    ; EAX = ALAMAT variabel angka
mov eax, angka      ; sama seperti lea di NASM — alamatnya
```

---

## 6. Contoh 1 — Hello World

`hello.asm`

```nasm
; ============================================
; Hello World — Linux x86 32-bit, NASM
; nasm -f elf32 hello.asm -o hello.o
; ld -m elf_i386 hello.o -o hello
; ============================================

section .data
    pesan       db  "Halo, dunia!", 0x0A   ; 0x0A = newline (\n)
    pesan_len   equ $ - pesan              ; panjang otomatis = 13

section .text
    global _start

_start:
    ; --- syscall write(1, pesan, pesan_len) ---
    mov eax, 4              ; nomor syscall: sys_write
    mov ebx, 1              ; arg1: file descriptor 1 = stdout
    mov ecx, pesan          ; arg2: alamat buffer yang mau ditulis
    mov edx, pesan_len      ; arg3: berapa byte
    int 0x80                ; panggil kernel

    ; --- syscall exit(0) ---
    mov eax, 1              ; nomor syscall: sys_exit
    xor ebx, ebx            ; arg1: exit code 0
    int 0x80
```

### Kenapa register-registernya begitu?

`int 0x80` adalah interupsi yang bilang ke kernel Linux: "tolong kerjakan sesuatu".
Kernel **selalu** membaca permintaan dari register yang sudah ditentukan:

```
EAX = nomor syscall (4 = write, 1 = exit, 3 = read)
EBX = argumen ke-1
ECX = argumen ke-2
EDX = argumen ke-3
ESI = argumen ke-4
EDI = argumen ke-5
```

Jadi `mov ecx, pesan` bukan karena "ECX itu counter", tapi karena posisi argumen
kedua `write()` kebetulan jatuh di ECX. Ini kontrak yang tidak bisa ditawar —
kalau kamu taruh alamat buffer di EDI, kernel tidak akan menemukannya.

---

## 7. Contoh 2 — Aritmatika

`hitung.asm`

```nasm
section .data
    a       dd  25
    b       dd  7
    hasil   dd  0

section .text
    global _start

_start:
    ; hasil = a + b
    mov eax, [a]        ; EAX = 25   (kurung siku = ambil ISI memori)
    add eax, [b]        ; EAX = 25 + 7 = 32
    mov [hasil], eax    ; simpan balik ke memori

    ; hasil = a - b
    mov eax, [a]
    sub eax, [b]        ; EAX = 18
    mov [hasil], eax

    ; hasil = a * b
    mov eax, [a]        ; mul WAJIB pakai EAX
    mov ebx, [b]
    mul ebx             ; EDX:EAX = EAX * EBX → EAX = 175
    mov [hasil], eax

    ; hasil = a / b  (dan sisanya)
    mov eax, [a]
    xor edx, edx        ; WAJIB! EDX jadi 32 bit atas dari pembagi
    mov ebx, [b]
    div ebx             ; EAX = 3 (hasil bagi), EDX = 4 (sisa)
    mov [hasil], eax

    ; keluar dengan exit code = isi EAX tadi
    mov ebx, eax
    mov eax, 1
    int 0x80
```

Cek hasilnya:

```bash
./hitung; echo $?    # → 3
```

> ⚠️ Lupa `xor edx, edx` sebelum `div` adalah bug klasik. Kalau EDX berisi sampah,
> pembagiannya jadi bilangan 64-bit raksasa dan program crash dengan
> *Floating point exception*.

---

## 8. Contoh 3 — Loop pakai ECX

`loop.asm` — cetak angka 1 sampai 5.

```nasm
section .data
    digit   db  '0', 0x0A       ; 1 karakter + newline

section .text
    global _start

_start:
    mov ecx, 5                  ; ECX = pencacah, loop 5 kali
    mov bl, '1'                 ; BL = karakter yang dicetak

.ulang:
    push ecx                    ; SIMPAN ECX! syscall akan merusaknya
    mov  [digit], bl            ; taruh karakter ke buffer

    mov  eax, 4                 ; sys_write
    mov  ebx, 1                 ; stdout
    mov  ecx, digit             ; ← ECX ketimpa di sini
    mov  edx, 2                 ; 2 byte: digit + newline
    int  0x80

    pop  ecx                    ; KEMBALIKAN ECX
    inc  byte [digit]           ; tidak dipakai, tapi contoh increment memori
    mov  bl, [digit]
    loop .ulang                 ; ECX--; kalau ECX != 0 → lompat ke .ulang

    mov eax, 1
    xor ebx, ebx
    int 0x80
```

### Poin penting

- **`loop` bekerja otomatis dengan ECX.** Dia mengurangi ECX 1, lalu melompat
  kalau hasilnya belum 0. Kamu tidak perlu (dan tidak bisa) menunjuk register lain.
- **`push`/`pop` wajib di sini** karena `int 0x80` untuk `write` memakai ECX
  sebagai alamat buffer — nilai pencacahmu akan hilang kalau tidak diselamatkan.
- Label yang diawali titik (`.ulang`) adalah **label lokal** di NASM: dia menempel
  ke label global terakhir (`_start`), jadi nama `.ulang` boleh dipakai lagi di
  fungsi lain tanpa bentrok.

Versi manual tanpa `loop` (lebih fleksibel, dan sebenarnya lebih cepat di CPU modern):

```nasm
    mov ecx, 0
.ulang:
    ; ... kerjakan sesuatu ...
    inc ecx
    cmp ecx, 5
    jl  .ulang          ; lompat selama ECX < 5
```

---

## 9. Contoh 4 — Fungsi & Stack Frame

Di sinilah EBP dan ESP berperan.

```nasm
section .text
    global _start

; int tambah(int a, int b)
tambah:
    push ebp                ; simpan frame pemanggil
    mov  ebp, esp           ; EBP jadi titik acuan frame ini

    mov  eax, [ebp + 8]     ; argumen pertama
    add  eax, [ebp + 12]    ; argumen kedua → hasil di EAX

    mov  esp, ebp           ; bersihkan frame lokal
    pop  ebp                ; pulihkan EBP pemanggil
    ret                     ; ambil alamat balik dari stack, lompat ke sana

_start:
    push 7                  ; argumen didorong TERBALIK (kanan ke kiri)
    push 25
    call tambah             ; call = push alamat balik, lalu jmp
    add  esp, 8             ; bersihkan 2 argumen dari stack (cdecl)
    ; sekarang EAX = 32

    mov  ebx, eax
    mov  eax, 1
    int  0x80
```

Susunan stack tepat setelah `mov ebp, esp`:

```
alamat tinggi
        ┌──────────────────┐
        │   7  (arg ke-2)  │  [ebp + 12]
        ├──────────────────┤
        │  25  (arg ke-1)  │  [ebp + 8]
        ├──────────────────┤
        │  alamat balik    │  [ebp + 4]   ← didorong oleh `call`
        ├──────────────────┤
EBP  →  │  EBP lama        │  [ebp + 0]
        ├──────────────────┤
        │  variabel lokal  │  [ebp - 4]   ← kalau ada `sub esp, N`
        └──────────────────┘
alamat rendah                             ← ESP
```

**Kenapa perlu EBP kalau sudah ada ESP?** Karena ESP terus bergerak setiap ada
`push`/`pop` di tengah fungsi. EBP dipatok sekali di awal dan tidak bergerak, jadi
`[ebp+8]` selalu menunjuk argumen yang sama sepanjang fungsi berjalan.

Kalau butuh variabel lokal:

```nasm
    push ebp
    mov  ebp, esp
    sub  esp, 16            ; sediakan 16 byte untuk variabel lokal
    mov  dword [ebp-4], 100 ; variabel lokal pertama
    ; ...
    mov  esp, ebp           ; kembalikan ESP (16 byte tadi dilepas)
    pop  ebp
    ret
```

---

## 10. Cara Compile & Jalankan

Install dulu:

```bash
sudo apt install nasm binutils         # Debian/Ubuntu
sudo pacman -S nasm binutils           # Arch
```

Assemble dan link (32-bit):

```bash
nasm -f elf32 program.asm -o program.o
ld -m elf_i386 program.o -o program
./program
echo $?                                # lihat exit code
```

Debug dengan GDB:

```bash
nasm -f elf32 -g -F dwarf program.asm -o program.o
ld -m elf_i386 program.o -o program
gdb ./program
```

Perintah GDB yang sering dipakai:

```
break _start        # pasang breakpoint
run                 # jalankan
stepi   (si)        # eksekusi 1 instruksi
info registers      # lihat semua register
p/x $eax            # cetak EAX dalam hex
x/4xw &angka        # periksa 4 word memori di alamat `angka`
x/s $ecx            # baca string yang ditunjuk ECX
layout asm          # tampilan kode + register
```

Kalau `ld` mengeluh soal 32-bit di mesin 64-bit:

```bash
sudo apt install gcc-multilib
```

---

## 11. Cheat Sheet

### Register

```
EAX  akumulator, nilai balik, nomor syscall, wajib untuk mul/div
EBX  base/pointer, callee-saved, arg syscall ke-1
ECX  counter loop, jumlah shift (CL), arg syscall ke-2
EDX  pasangan EAX di mul/div, arg syscall ke-3
ESI  sumber saat copy string/array
EDI  tujuan saat copy string/array
ESP  puncak stack — jangan diubah manual
EBP  acuan stack frame fungsi
```

### Section

```
.text    kode          R+X    ada di file
.data    data terisi   R+W    ada di file
.rodata  konstanta     R      ada di file
.bss     data kosong   R+W    TIDAK ada di file (cuma ukurannya)
```

### Ukuran data

```
db / resb   1 byte    (8 bit)
dw / resw   2 byte    (16 bit)
dd / resd   4 byte    (32 bit)
dq / resq   8 byte    (64 bit)
```

### Syscall Linux 32-bit yang sering dipakai

```
EAX=1  exit(EBX)
EAX=3  read(EBX=fd, ECX=buf, EDX=count)
EAX=4  write(EBX=fd, ECX=buf, EDX=count)
EAX=5  open(EBX=path, ECX=flags, EDX=mode)
EAX=6  close(EBX=fd)
```

### Percabangan setelah `cmp a, b`

```
je / jz     lompat kalau a == b
jne / jnz   lompat kalau a != b
jl  / jnge  lompat kalau a <  b   (signed)
jle         lompat kalau a <= b   (signed)
jg  / jnle  lompat kalau a >  b   (signed)
jge         lompat kalau a >= b   (signed)
jb  / jbe   versi unsigned dari jl / jle
ja  / jae   versi unsigned dari jg / jge
```

---

## 12. Catatan 64-bit

Kalau nanti pindah ke x86-64, yang berubah:

- Register jadi `RAX`, `RBX`, `RCX`, `RDX`, `RSI`, `RDI`, `RSP`, `RBP`, plus
  tambahan baru `R8`–`R15`. `EAX` tetap ada sebagai 32 bit bawah dari `RAX`.
- Syscall pakai instruksi `syscall`, bukan `int 0x80`.
- Nomor dan urutan argumen syscall berbeda:

```
RAX = nomor syscall  (1 = write, 60 = exit — beda dari 32-bit!)
RDI = arg1
RSI = arg2
RDX = arg3
R10 = arg4
R8  = arg5
R9  = arg6
```

Hello World versi 64-bit:

```nasm
section .data
    pesan     db  "Halo, dunia!", 0x0A
    pesan_len equ $ - pesan

section .text
    global _start

_start:
    mov rax, 1              ; sys_write
    mov rdi, 1              ; stdout
    mov rsi, pesan
    mov rdx, pesan_len
    syscall

    mov rax, 60             ; sys_exit
    xor rdi, rdi
    syscall
```

```bash
nasm -f elf64 hello64.asm -o hello64.o
ld hello64.o -o hello64
```

> Catatan kecil: menulis ke register 32-bit di mode 64-bit (`mov eax, 5`) otomatis
> menolkan 32 bit atas dari RAX. Menulis ke `ax` atau `al` tidak. Ini sumber bug
> yang cukup halus.

---

## Bacaan Lanjutan

- **NASM Manual** — `man nasm` atau dokumentasi resmi NASM
- **Intel Software Developer's Manual Volume 2** — referensi lengkap tiap instruksi
- `man 2 syscall` — tabel syscall Linux per arsitektur
- Bongkar biner C untuk belajar: `gcc -S -masm=intel -O0 file.c`
