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

---

## Bacaan Lanjutan

- **NASM Manual** — `man nasm` atau dokumentasi resmi NASM
- **Intel Software Developer's Manual Volume 2** — referensi lengkap tiap instruksi
- `man 2 syscall` — tabel syscall Linux per arsitektur
- Bongkar biner C untuk belajar: `gcc -S -masm=intel -O0 file.c`
