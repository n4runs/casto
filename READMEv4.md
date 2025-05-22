## 📌 Deskripsi Soal
Implementasi sistem file virtual berbasis FUSE bernama `antink`, yang dimount di `/mnt/antink` dan menampilkan isi dari direktori host (`/it24_host`) dengan beberapa perlakuan khusus pada file tertentu.

---

## ✅ Poin A - Penamaan File (Reverse)
### ✳️ Ketentuan:
- Semua file yang ditampilkan oleh sistem file harus dibalik **nama dan ekstensinya**, **masing-masing** secara terpisah.
- Contoh: `kimcun.txt` → `txt.nucmik`

### ✔️ Implementasi:
- Fungsi `reverse_parts()` memisahkan nama dan ekstensi, membalik masing-masing, lalu menggabungkan ulang.
- Hasil `readdir` ditampilkan menggunakan hasil dari `reverse_parts()`.

### 🔍 Contoh Output:
```bash
$ docker exec -it antink-server ls /mnt/antink
txt.tset  vsc.nucmik
```

---

## ✅ Poin B - File Berbahaya Tetap Muncul
### ✳️ Ketentuan:
- File yang mengandung kata `kimcun` atau `nafis` **tetap ditampilkan** dalam direktori (dengan nama dibalik), tetapi tidak dapat diakses.

### ✔️ Implementasi:
- `readdir`: tetap menampilkan semua file.
- `getattr` dan `read`: melakukan pengecekan dengan `is_dangerous()`, jika berbahaya maka dikembalikan error `-ENOENT` / `-EACCES`.

### 🔍 Contoh Output:
```bash
$ docker exec -it antink-server ls /mnt/antink
vsc.nucmik  txt.tset

$ docker exec -it antink-server cat /mnt/antink/nucmik.csv
cat: /mnt/antink/nucmik.csv: Permission denied
```

---

## ✅ Poin C - ROT13 untuk Isi File Normal
### ✳️ Ketentuan:
- Isi file selain `kimcun` dan `nafis` harus dienkripsi menggunakan ROT13 saat dibaca.

### ✔️ Implementasi:
- Fungsi `rot13()` diterapkan pada buffer hasil pembacaan file di `read`.

### 🔍 Contoh Output:
```bash
$ docker exec -it antink-server cat /mnt/antink/txt.tset
vav svyr abezny
```
Isi file asli `test.txt`: `ini file normal`

---

## ✅ Poin D - Logging Aktivitas
### ✳️ Ketentuan:
- Semua akses `read` dan deteksi file berbahaya harus dicatat ke log `/var/log/it24.log` dalam format `[HH:MM:SS] TYPE: path`.

### ✔️ Implementasi:
- Fungsi `log_activity(type, path)` menulis log ke `/var/log/it24.log`.
- Logging dilakukan di:
  - `getattr` → untuk file berbahaya: `WARNING`
  - `read` → untuk file normal: `READ`

### 🔍 Contoh Output Log:
```bash
$ docker exec -it antink-logger tail /var/log/it24.log
[14:00:01] WARNING: /vsc.nucmik
[14:00:14] READ: /txt.tset
```

---

## 🧱 Struktur Proyek
```text
.
├── Dockerfile
├── docker-compose.yml
├── antink.c
├── it24_host/
│   ├── kimcun.csv
│   └── test.txt
├── antink_mount/      # mountpoint
└── antink-logs/
    └── it24.log       # log aktivitas
```

---

## ⚙️ Dockerfile
```dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y gcc fuse libfuse-dev pkg-config

COPY antink.c /antink.c

RUN gcc -Wall -D_FILE_OFFSET_BITS=64 /antink.c $(pkg-config fuse --cflags --libs) -o /antink_fs

CMD ["/antink_fs", "-f", "/mnt/antink"]
```

---

## ⚙️ docker-compose.yml
```yaml
version: '3.8'

services:
  antink-server:
    build: .
    container_name: antink-server
    privileged: true
    cap_add:
      - SYS_ADMIN
    devices:
      - /dev/fuse
    volumes:
      - ./it24_host:/it24_host
      - ./antink_mount:/mnt/antink
      - ./antink-logs:/var/log

  antink-logger:
    image: ubuntu:22.04
    volumes:
      - ./antink-logs/it24.log:/var/log/it24.log
    command: tail -f /var/log/it24.log
```

---

## 🧪 Cara Uji Coba
```bash
# Bangun dan jalankan
docker compose down
docker compose build
docker compose up

# Cek hasil mount
docker exec -it antink-server ls /mnt/antink

# Baca file terenkripsi
docker exec -it antink-server cat /mnt/antink/txt.tset

# Cek log
docker exec -it antink-logger tail /var/log/it24.log
```

---

## 🏁 Status Akhir
✅ Seluruh poin soal berhasil dipenuhi:
- Penamaan terbalik ✅
- File berbahaya muncul tapi ditolak aksesnya ✅
- ROT13 berhasil diterapkan ✅
- Logging ke container logger berhasil ✅

---
