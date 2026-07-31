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

## 6. Contoh utuh — tanya nama, lalu tampilkan (64-bit)

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

## 7. Cheat Sheet Ringkas

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
```

---

## Bacaan Lanjutan

- Tabel syscall lengkap: `man 2 syscalls`, atau file
  `/usr/include/asm/unistd_32.h` dan `unistd_64.h`
- Detail tiap syscall: `man 2 read`, `man 2 write`, `man 2 exit`
- Untuk melihat syscall yang dipanggil program: `strace ./program`
