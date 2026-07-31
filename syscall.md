# Cheatsheet Syscall Linux (x86) — File Descriptor, read, write

Referensi cepat: cara memanggil kernel Linux dari assembly untuk input & output.
Mencakup **32-bit** (`int 0x80`) dan **64-bit** (`syscall`) sekaligus, karena
nomor & register-nya berbeda dan sering tertukar.

---

## 1. File Descriptor (FD)

Angka yang menunjuk "ke/dari mana" data mengalir. Tiga yang selalu tersedia:

```
0 = stdin    → input   (keyboard)          dipakai dgn read
1 = stdout   → output  (layar, teks biasa) dipakai dgn write
2 = stderr   → output  (layar, pesan error) dipakai dgn write
```

Cara ingat: **0 masuk, 1 & 2 keluar.** stdout untuk hasil normal, stderr untuk
pesan kesalahan (supaya bisa dipisah, misal `program 2> error.txt`).

### Contoh tiap FD (32-bit)

FD 0 (stdin) — baca dari keyboard:

```nasm
    mov eax, 3          ; sys_read
    mov ebx, 0          ; ← fd 0 = stdin
    mov ecx, buffer
    mov edx, 100
    int 0x80
```

FD 1 (stdout) — hasil normal ke layar:

```nasm
    mov eax, 4          ; sys_write
    mov ebx, 1          ; ← fd 1 = stdout
    mov ecx, pesan
    mov edx, panjang
    int 0x80
```

FD 2 (stderr) — pesan error ke layar:

```nasm
    mov eax, 4          ; sys_write
    mov ebx, 2          ; ← fd 2 = stderr
    mov ecx, err
    mov edx, errlen
    int 0x80
```

Beda FD 1 vs 2 baru terasa saat aliran dipisah di terminal:

```bash
./program 1> hasil.txt 2> error.txt
```

Yang lewat stdout masuk `hasil.txt`, yang lewat stderr masuk `error.txt`.

---

## 2. Cara memanggil syscall

Kernel dipanggil dengan mengisi register lalu memicu instruksi khusus. Isi
register menentukan layanan mana + argumennya.

### 32-bit (`int 0x80`)

```
EAX = nomor syscall
EBX = argumen ke-1
ECX = argumen ke-2
EDX = argumen ke-3
ESI = argumen ke-4
EDI = argumen ke-5
      → picu dengan:  int 0x80
      → nilai balik ada di EAX
```

### 64-bit (`syscall`)

```
RAX = nomor syscall
RDI = argumen ke-1
RSI = argumen ke-2
RDX = argumen ke-3
R10 = argumen ke-4
R8  = argumen ke-5
R9  = argumen ke-6
      → picu dengan:  syscall
      → nilai balik ada di RAX
```

> Jangan campur. `int 0x80` untuk kode 32-bit, `syscall` untuk 64-bit. Mencampur
> keduanya (mis. `int 0x80` di biner elf64) sering menyebabkan **segfault**.

---

## 3. Nomor syscall yang sering dipakai

**Perhatikan: nomor 32-bit dan 64-bit BERBEDA.** Ini sumber bug paling umum.

| Syscall | Fungsi                | No. 32-bit | No. 64-bit |
|---------|-----------------------|------------|------------|
| read    | menerima input        | 3          | 0          |
| write   | menampilkan output    | 4          | 1          |
| open    | buka file             | 5          | 2          |
| close   | tutup file            | 6          | 3          |
| exit    | keluar program        | 1          | 60         |

Jebakan klasik: di 32-bit write=4/exit=1, di 64-bit write=1/exit=60. Kalau kode
"kelihatan benar" tapi crash, cek dulu nomornya cocok dengan mode yang kamu pakai.

---

## 4. sys_write — menampilkan output

Butuh 3 argumen: **fd, alamat teks, panjang teks.**

```
write(fd, buffer, jumlah_byte)
```

### 32-bit

```nasm
    mov eax, 4          ; sys_write
    mov ebx, 1          ; fd = 1 (stdout)
    mov ecx, pesan      ; alamat teks
    mov edx, panjang    ; jumlah byte
    int 0x80
```

### 64-bit

```nasm
    mov rax, 1          ; sys_write
    mov rdi, 1          ; fd = 1 (stdout)
    mov rsi, pesan      ; alamat teks
    mov rdx, panjang    ; jumlah byte
    syscall
```

---

## 5. sys_read — menerima input

Butuh 3 argumen: **fd, buffer tujuan, kapasitas maksimum.** Membaca dari keyboard
lalu menyimpan ke buffer. Nilai balik (EAX/RAX) = jumlah byte yang benar-benar dibaca.

```
read(fd, buffer, maks_byte)  →  mengembalikan jumlah byte terbaca
```

### 32-bit

```nasm
    mov eax, 3          ; sys_read
    mov ebx, 0          ; fd = 0 (stdin)
    mov ecx, buffer     ; tempat menyimpan input
    mov edx, 100        ; baca maksimal 100 byte
    int 0x80
    ; sekarang EAX = berapa byte yang dibaca (termasuk Enter/newline)
```

### 64-bit

```nasm
    mov rax, 0          ; sys_read
    mov rdi, 0          ; fd = 0 (stdin)
    mov rsi, buffer     ; tempat menyimpan input
    mov rdx, 100        ; baca maksimal 100 byte
    syscall
    ; sekarang RAX = jumlah byte yang dibaca
```

> Buffer-nya harus disediakan di `.bss` dengan `resb`, contoh: `buffer resb 100`.
> Nilai balik di RAX/EAX berguna: itu panjang input asli, bisa dipakai sebagai
> `edx` saat menampilkannya kembali dengan write.

---

## 6. open & close — bekerja dengan file

`open` membuka/membuat file dan **mengembalikan file descriptor baru** di EAX/RAX.
`close` menutupnya. Keduanya paling jelas ditunjukkan bersama.

```
open(path, flags, izin)  →  mengembalikan fd baru (biasanya 3, 4, ...)
close(fd)
```

Contoh utuh (32-bit) — tulis teks ke file `hasil.txt`:

```nasm
section .data
    namafile db "hasil.txt", 0      ; WAJIB diakhiri 0 (null-terminated)
    isi      db "Halo dari assembly!", 0xA
    isilen   equ $ - isi

section .text
    global _start
_start:
    ; open("hasil.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644)
    mov eax, 5
    mov ebx, namafile
    mov ecx, 0x241          ; flag: tulis + buat kalau belum ada + kosongkan
    mov edx, 0x1A4          ; izin 0644 (rw-r--r--)
    int 0x80
    mov esi, eax            ; ← SIMPAN fd hasil open!

    ; write ke file (pakai fd dari open, BUKAN 1)
    mov eax, 4
    mov ebx, esi
    mov ecx, isi
    mov edx, isilen
    int 0x80

    ; close(fd)
    mov eax, 6
    mov ebx, esi
    int 0x80

    ; exit(0)
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

Dua jebakan yang wajib diingat:

1. **`open` memberi fd baru di EAX** — simpan (di contoh ke `esi`) untuk dipakai
   `write` dan `close`. Kalau kamu tulis `mov ebx, 1`, tulisannya malah ke layar,
   bukan ke file.
2. **Path harus diakhiri byte `0`** (`db "hasil.txt", 0`). Kernel membaca nama
   sampai ketemu null; tanpa `, 0` bisa error atau membuka file bernama sampah.

Angka flag `open` yang sering dipakai (32 & 64-bit sama):

```
0x001  O_WRONLY   hanya tulis
0x002  O_RDWR     baca + tulis
0x040  O_CREAT    buat kalau belum ada
0x200  O_TRUNC    kosongkan isi lama
0x400  O_APPEND   tambah di akhir
gabung dgn OR, mis. 0x241 = O_WRONLY|O_CREAT|O_TRUNC
```

---

## 7. Contoh utuh — tanya nama, lalu tampilkan (64-bit)

```nasm
section .bss
    nama    resb 100            ; buffer input, 100 byte

section .data
    tanya   db  "Siapa nama kamu? "
    tlen    equ $ - tanya

section .text
    global _start

_start:
    ; 1) tampilkan pertanyaan (write)
    mov rax, 1
    mov rdi, 1                  ; stdout
    mov rsi, tanya
    mov rdx, tlen
    syscall

    ; 2) baca input dari keyboard (read)
    mov rax, 0
    mov rdi, 0                  ; stdin
    mov rsi, nama
    mov rdx, 100
    syscall                     ; RAX = jumlah byte yang diketik

    mov r8, rax                 ; simpan panjang input untuk dipakai nanti

    ; 3) tampilkan kembali input tadi (write)
    mov rax, 1
    mov rdi, 1
    mov rsi, nama
    mov rdx, r8                 ; sepanjang input tadi
    syscall

    ; 4) keluar
    mov rax, 60
    xor rdi, rdi               ; exit code 0
    syscall
```

Compile & jalankan:

```bash
nasm -f elf64 nama.asm -o nama.o
ld nama.o -o nama
./nama
```

Versi 32-bit: ganti register ke `eax/ebx/ecx/edx`, nomor jadi read=3/write=4/exit=1,
`int 0x80`, lalu `nasm -f elf32` + `ld -m elf_i386`.

---

## 8. Cheat Sheet Ringkas

### File descriptor
```
0 stdin   input  (read)
1 stdout  output (write) — hasil normal
2 stderr  output (write) — pesan error
```

### Register argumen
```
32-bit:  EAX=no  EBX=arg1  ECX=arg2  EDX=arg3   → int 0x80   → balik: EAX
64-bit:  RAX=no  RDI=arg1  RSI=arg2  RDX=arg3   → syscall    → balik: RAX
```

### Nomor syscall (32 / 64)
```
read   3 / 0      write  4 / 1
open   5 / 2      close  6 / 3
exit   1 / 60
```

### Pola write
```
write(fd=1, alamat_teks, panjang)   → menampilkan
```

### Pola read
```
read(fd=0, buffer, maks)            → menerima, RAX = byte terbaca
```

### Aturan emas
```
- int 0x80  ↔  elf32  ↔  EAX/EBX...   (jangan campur)
- syscall   ↔  elf64  ↔  RAX/RDI...   (jangan campur)
- nomor syscall 32-bit ≠ 64-bit — selalu cek modenya
- buffer untuk read disiapkan di .bss pakai resb
- open memberi fd baru di EAX/RAX — simpan sebelum write/close
- path file harus null-terminated: db "nama", 0
```

### Perintah build (WAJIB cocok!)
```
32-bit:  nasm -f elf32 f.asm -o f.o  &&  ld -m elf_i386 -o f f.o
64-bit:  nasm -f elf64 f.asm -o f.o  &&  ld -o f f.o
```
Lupa `-m elf_i386` di jalur 32-bit → error "i386 incompatible with i386:x86-64".

---

## Bacaan Lanjutan

- Tabel syscall lengkap: `man 2 syscalls`, atau file
  `/usr/include/asm/unistd_32.h` dan `unistd_64.h`
- Detail tiap syscall: `man 2 read`, `man 2 write`, `man 2 exit`
- Untuk melihat syscall yang dipanggil program: `strace ./program`
