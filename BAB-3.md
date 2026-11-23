# BAB 3 — DOCKER NETWORK, VOLUME, BIND MOUNT, TMPFS, DAN COMPOSE

## 1. Tujuan Praktikum

Praktikum ini bertujuan untuk memahami penggunaan Docker Network, Volume, Bind Mount, tmpfs, dan Docker Compose. Melalui praktikum ini, dilakukan pengujian komunikasi antar-container menggunakan user-defined bridge network, penyimpanan data menggunakan named volume, serta proses backup dan restore data.

Selain itu, praktikum ini bertujuan memahami konfigurasi aplikasi multi-container menggunakan Docker Compose yang terdiri dari Nginx, Flask, dan PostgreSQL.

## 2. Langkah Praktikum dan Hasil

### 2.1 Membuat Direktori Praktikum

Langkah pertama adalah membuat direktori khusus untuk menyimpan file dan konfigurasi praktikum Bab 3.

Perintah yang digunakan:

```bash
mkdir -p ~/docker-lab/bab-3
cd ~/docker-lab/bab-3
```

Direktori ini digunakan sebagai tempat kerja selama pelaksanaan praktikum.

### 2.2 Membuat User-Defined Bridge Network

Pada praktikum ini dibuat sebuah user-defined bridge network dengan nama `lab-net` menggunakan subnet `172.20.0.0/16`.

Perintah yang digunakan:

```bash
docker network create --driver bridge --subnet 172.20.0.0/16 lab-net
```

Setelah network berhasil dibuat, dua container berbasis image `nginx:alpine`, yaitu `server-a` dan `server-b`, dijalankan dan dihubungkan ke network tersebut.

```bash
docker run -d --name server-a --network lab-net nginx:alpine

docker run -d --name server-b --network lab-net nginx:alpine
```

Selanjutnya dilakukan pengujian komunikasi menggunakan perintah:

```bash
docker exec server-a ping -c 3 server-b
```

Pengujian ini bertujuan memastikan bahwa `server-a` dapat menemukan dan berkomunikasi dengan `server-b` menggunakan nama container sebagai hostname melalui DNS internal Docker.

Jika tiga paket berhasil diterima, komunikasi antar-container pada user-defined bridge network berjalan dengan baik.

Setelah pengujian selesai, kedua container dihapus menggunakan:

```bash
docker rm -f server-a server-b
```

**Hasil Praktikum:**

Pembuatan user-defined bridge network dan pengujian komunikasi antar-container dilakukan untuk membuktikan bahwa container dalam network yang sama dapat saling berkomunikasi menggunakan nama container.

### 2.3 Volume Backup dan Restore

Pada praktikum ini dibuat named volume dengan nama `data-vol`.

```bash
docker volume create data-vol
```

Selanjutnya dijalankan container `writer` menggunakan image `alpine:3.20`. Volume `data-vol` di-mount ke direktori `/app/data`.

```bash
docker run -d --name writer -v data-vol:/app/data alpine:3.20 sh -c "while true; do date >> /app/data/log.txt; sleep 5; done"
```

Container tersebut menjalankan proses yang secara berkala menuliskan tanggal ke file `/app/data/log.txt` setiap 5 detik.

Setelah menunggu selama 15 detik, container dihentikan dan dihapus menggunakan:

```bash
docker rm -f writer
```

Data pada volume kemudian diverifikasi dengan menjalankan container sementara:

```bash
docker run --rm -v data-vol:/data alpine:3.20 cat /data/log.txt
```

Hasil tersebut membuktikan bahwa data tetap tersimpan pada named volume meskipun container yang membuat data telah dihapus.

Setelah data terverifikasi, dilakukan proses backup dengan me-mount `data-vol` sebagai `/source` dalam mode read-only dan direktori kerja host sebagai `/backup`.

```bash
docker run --rm -v data-vol:/source:ro -v $(pwd):/backup alpine:3.20 tar czf /backup/data-vol-backup.tar.gz -C /source .
```

Perintah `tar czf` digunakan untuk mengarsipkan seluruh isi volume menjadi file `data-vol-backup.tar.gz`.

**Hasil Praktikum:**

Praktikum menunjukkan bahwa named volume dapat digunakan untuk menyimpan data secara persisten. Data tetap tersedia setelah container dihapus dan dapat dibuat menjadi file backup untuk kebutuhan pemulihan.

### 2.4 Bind Mount

Bind mount merupakan mekanisme untuk menghubungkan direktori atau file pada host secara langsung ke dalam container.

Bind mount dapat digunakan untuk kebutuhan pengembangan aplikasi, misalnya menghubungkan source code dari host ke container agar perubahan file dapat langsung digunakan tanpa harus membangun ulang image.

Contoh penggunaan bind mount:

```bash
docker run -d --name bind-test -v $(pwd):/app nginx:alpine
```

Pada contoh tersebut, direktori kerja host dihubungkan ke direktori `/app` di dalam container.

Bind mount memiliki risiko keamanan karena container dapat membaca atau mengubah file pada host apabila mount diberikan akses tulis.

Jika container hanya membutuhkan akses membaca, mount dapat dibuat read-only:

```bash
docker run -d --name bind-readonly -v $(pwd):/app:ro nginx:alpine
```

### 2.5 tmpfs Mount

tmpfs merupakan jenis mount yang menyimpan data sementara di memori host dan tidak ditujukan untuk penyimpanan permanen.

Contoh penggunaan tmpfs:

```bash
docker run -d --name tmpfs-test --tmpfs /app/temp nginx:alpine
```

Data yang disimpan pada direktori `/app/temp` bersifat sementara dan tidak digunakan untuk menyimpan data yang harus tetap tersedia setelah container dihentikan atau dibuat ulang.

tmpfs cocok digunakan untuk cache atau file sementara yang tidak membutuhkan persistensi.

### 2.6 Docker Compose Multi-Container

Docker Compose digunakan untuk mengelola aplikasi yang terdiri dari beberapa container melalui satu file konfigurasi YAML.

Pada praktikum ini dirancang aplikasi multi-container yang terdiri dari tiga service, yaitu:

1. Nginx sebagai web server.
2. Flask sebagai aplikasi backend.
3. PostgreSQL sebagai database.

Konfigurasi Compose menggunakan service, network, volume, dan healthcheck. PostgreSQL menggunakan named volume agar data database dapat disimpan secara persisten. Healthcheck digunakan untuk memeriksa kesiapan PostgreSQL sebelum service Flask bergantung kepadanya.

**Kendala yang Terjadi:**

Pada saat menjalankan stack Docker Compose, praktikum mengalami kendala karena kapasitas storage Ubuntu pada VirtualBox telah mencapai 100%.

Kendala tersebut dibuktikan melalui pemeriksaan kapasitas penyimpanan menggunakan:

```bash
df -h
```

Selain itu, muncul pesan error PostgreSQL:

```text
No space left on device
```

Pesan tersebut menunjukkan bahwa ruang penyimpanan yang tersedia tidak mencukupi untuk proses yang sedang dijalankan PostgreSQL.

Oleh karena itu, proses pengujian Compose belum dapat diselesaikan sepenuhnya. Kapasitas storage VM perlu ditambah atau file yang tidak diperlukan perlu dibersihkan sebelum pengujian dilanjutkan.

### 2.7 Perintah Pengelolaan Docker Compose

Beberapa perintah Docker Compose yang digunakan untuk mengelola lifecycle aplikasi adalah:

```bash
docker compose up -d
```

Digunakan untuk membuat dan menjalankan seluruh service di background.

```bash
docker compose ps
```

Digunakan untuk melihat status service.

```bash
docker compose logs -f
```

Digunakan untuk melihat log seluruh service secara langsung.

```bash
docker compose stop
```

Digunakan untuk menghentikan service tanpa menghapus container.

```bash
docker compose start
```

Digunakan untuk menjalankan kembali container yang telah dihentikan.

```bash
docker compose down
```

Digunakan untuk menghentikan dan menghapus container serta network yang dibuat oleh Compose.

```bash
docker compose down -v
```

Digunakan untuk menghapus container, network, dan volume yang dikelola oleh project Compose. Perintah ini perlu digunakan dengan hati-hati karena data pada volume dapat ikut terhapus.

## 3. Analisis Praktikum

Berdasarkan praktikum yang dilakukan, Docker Network berfungsi untuk mengatur komunikasi antar-container. User-defined bridge network memberikan kemudahan komunikasi melalui DNS internal Docker sehingga container dapat ditemukan menggunakan nama container.

Named volume digunakan untuk menyimpan data secara persisten. Data tetap tersedia meskipun container yang menggunakannya telah dihapus. Selain itu, data pada volume dapat dibuat menjadi file backup untuk kebutuhan pemulihan.

Bind mount memiliki fungsi menghubungkan direktori host ke container secara langsung. Mekanisme ini berguna untuk pengembangan aplikasi, tetapi memiliki risiko keamanan apabila container diberikan akses tulis ke direktori host yang sensitif.

Sementara itu, tmpfs digunakan untuk menyimpan data sementara di memori host. Data pada tmpfs tidak ditujukan untuk penyimpanan jangka panjang.

Docker Compose mempermudah pengelolaan beberapa container dalam satu aplikasi. Konfigurasi service, network, volume, dan healthcheck dapat didefinisikan dalam satu file YAML.

Pada praktikum ini, pengujian network dan volume dapat dilakukan. Namun, pengujian Docker Compose mengalami kendala akibat kapasitas storage Ubuntu pada VirtualBox yang mencapai 100%. Kondisi tersebut menyebabkan PostgreSQL tidak dapat melanjutkan proses karena ruang penyimpanan tidak mencukupi.

## 4. Jawaban Pertanyaan

**1. Apa perbedaan user-defined bridge dan default bridge?**

User-defined bridge dibuat sendiri oleh pengguna dan menyediakan fitur komunikasi serta resolusi nama container yang lebih baik. Default bridge merupakan network bawaan Docker.

**2. Apa risiko keamanan bind mount?**

Bind mount memberikan container akses langsung terhadap direktori host. Jika direktori sensitif diberikan kepada container, data host dapat berisiko dibaca atau dimodifikasi.

**3. Apa perbedaan docker compose down dan docker compose down -v?**

`docker compose down` menghapus container dan network tetapi mempertahankan named volume. Sedangkan `docker compose down -v` juga menghapus volume sehingga data yang tersimpan pada volume dapat hilang.

**4. Mengapa menggunakan healthcheck?**

Healthcheck digunakan untuk memastikan PostgreSQL benar-benar siap menerima koneksi sebelum service Flask bergantung kepadanya.

**5. Bagaimana backup database production?**

Backup production sebaiknya dilakukan secara berkala, disimpan pada lokasi berbeda dari server utama, memiliki beberapa versi backup, serta dilakukan pengujian restore secara rutin.

## 5. Troubleshooting

Kendala utama yang ditemukan selama praktikum adalah kapasitas storage Ubuntu pada VirtualBox mencapai 100%.

Pemeriksaan dilakukan menggunakan:

```bash
df -h
```

Hasil pemeriksaan menunjukkan bahwa ruang penyimpanan VM telah penuh. Hal tersebut menyebabkan PostgreSQL menampilkan pesan:

```text
No space left on device
```

Solusi yang dapat dilakukan adalah menambah kapasitas penyimpanan virtual machine atau membersihkan file dan image Docker yang tidak lagi digunakan.

Sebelum melakukan penghapusan, data dan volume yang masih dibutuhkan harus diamankan agar tidak terjadi kehilangan data praktikum.

## 6. Risiko Keamanan dan Rekomendasi

Beberapa risiko keamanan dan operasional yang perlu diperhatikan dalam penggunaan Docker Network dan Storage adalah:

1. Bind mount dapat memberikan akses langsung terhadap file host sehingga direktori sensitif harus dilindungi.
2. Port container yang dipublikasikan ke host dapat membuka layanan ke jaringan yang tidak diinginkan.
3. Named volume dapat kehilangan data jika dihapus menggunakan perintah yang tidak tepat.
4. Kapasitas storage yang penuh dapat menyebabkan aplikasi dan database gagal berjalan.
5. Credential database tidak boleh disimpan langsung dalam repository publik.

Rekomendasi untuk lingkungan production-like adalah membatasi port yang dipublikasikan, menggunakan bind mount read-only jika memungkinkan, melakukan backup volume secara berkala, mengamankan credential menggunakan secret, serta memantau kapasitas penyimpanan secara rutin.

## 7. Kesimpulan

Praktikum Docker Bab 3 berhasil memberikan pemahaman mengenai network, volume, bind mount, tmpfs, dan Docker Compose.

Pengujian user-defined bridge network dan named volume dapat dilakukan untuk memahami komunikasi antar-container dan persistensi data. Pada pengujian volume, data terbukti tetap tersimpan setelah container dihapus dan dapat dibuat menjadi file backup.

Pada Docker Compose, konfigurasi aplikasi Nginx, Flask, dan PostgreSQL telah dibuat. Namun, proses menjalankan stack mengalami kendala karena kapasitas storage Ubuntu pada VirtualBox telah mencapai 100%. Kendala tersebut dibuktikan melalui hasil `df -h` dan pesan error PostgreSQL `No space left on device`.

Oleh karena itu, kapasitas storage VM perlu ditambah atau ruang penyimpanan yang tidak diperlukan perlu dibersihkan sebelum pengujian Compose dapat dilanjutkan.
