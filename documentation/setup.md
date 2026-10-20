# Setup: Mengganti Nama Database menjadi `restaurant_reviews`

Panduan ini menjelaskan cara mengganti nama database PostgreSQL dari `app` menjadi `restaurant_reviews`, lalu menjalankan aplikasi Flask di Dev Container.

## Prasyarat

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) sudah terpasang dan **berjalan** (status "Engine running").
- VS Code dengan extension **Dev Containers** (`ms-vscode-remote.remote-containers`).
- Repository ini sudah di-clone dan dibuka di VS Code.

## Gambaran Arsitektur

Dev Container menjalankan 3 container (lihat [.devcontainer/docker-compose.yml](../.devcontainer/docker-compose.yml)):

| Service | Fungsi |
|---|---|
| `app` | Container Python tempat Flask dijalankan |
| `db` | PostgreSQL 15 |
| `redis` | Redis (cache) |

Nama database ditentukan di **4 tempat** dan semuanya harus sama persis.

## Langkah 1 — Ubah `.env`

File: [.env](../.env)

```diff
- DBNAME=app
+ DBNAME=restaurant_reviews
```

Flask membaca variabel ini di [azureproject/development.py](../azureproject/development.py) untuk menyusun connection string, jadi kode Python tidak perlu diubah.

## Langkah 2 — Ubah `docker-compose.yml`

File: [.devcontainer/docker-compose.yml](../.devcontainer/docker-compose.yml), pada service `db`.

**a. Nama database yang dibuat PostgreSQL:**

```diff
    environment:
-     POSTGRES_DB: app
+     POSTGRES_DB: restaurant_reviews
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: app_password
```

**b. Healthcheck, supaya yang dicek database yang benar:**

```diff
    healthcheck:
-     test: ["CMD-SHELL", "pg_isready -U app_user"]
+     test: ["CMD-SHELL", "pg_isready -U app_user -d restaurant_reviews -h localhost"]
```

> ⚠️ Setiap flag harus langsung diikuti nilainya: `-U <user>` lalu `-d <database>`.
> Kalau ditulis `pg_isready -U -d restaurant_reviews`, healthcheck selalu gagal dan container `app` tidak akan pernah start, karena `app` menunggu `db` dinyatakan sehat (`service_healthy`).

## Langkah 3 — Ubah `devcontainer.json`

File: [.devcontainer/devcontainer.json](../.devcontainer/devcontainer.json), bagian koneksi SQLTools:

```diff
-   "database": "app",
+   "database": "restaurant_reviews",
```

## Langkah 4 — Validasi file compose

Jalankan di PowerShell dari root project:

```powershell
docker compose -f .devcontainer/docker-compose.yml config --quiet
```

Kalau tidak ada error, berarti file valid. Peringatan "`version` is obsolete" boleh diabaikan.

## Langkah 5 — Hapus volume database lama (jika ada)

Image `postgres` **hanya membuat `POSTGRES_DB` saat volume datanya masih kosong**. Kalau Dev Container sudah pernah dijalankan sebelumnya, volume lama masih berisi database `app` dan database `restaurant_reviews` tidak akan dibuat.

```powershell
docker volume ls
```

- Jika **tidak ada** volume berakhiran `_postgres-data`, lewati langkah ini.
- Jika **ada**, hapus volume tersebut. ⚠️ Semua data di database lama ikut hilang.

  ```powershell
  docker volume rm <nama_volume>_postgres-data
  ```

  Jika muncul error "volume is in use", tutup dulu Dev Container atau hentikan container-nya lewat Docker Desktop, lalu ulangi.

## Langkah 6 — Buka di Dev Container

1. Tekan `Ctrl+Shift+P`.
2. Pilih **Dev Containers: Reopen in Container**. Kalau container sudah pernah dibuat, pilih **Dev Containers: Rebuild Container**.
3. Tunggu sampai build selesai. Build pertama bisa memakan waktu beberapa menit.

## Langkah 7 — Install dependency, migrasi, dan jalankan aplikasi

Di terminal VS Code (di dalam container):

```bash
pip install -r requirements.txt
flask db upgrade
flask run --host 0.0.0.0 --port 8000
```

`pip install` perlu dijalankan manual karena `postCreateCommand` di `devcontainer.json` di-comment. Perintah `flask db upgrade` membuat tabel di database `restaurant_reviews`.

### Dari mana tabel `restaurant` dan `review` berasal?

Struktur tabel didefinisikan di [`models.py`](../models.py) (class `Restaurant`, `Review`, dan `Menu`), yang ditulis manual. `models.py` sendiri tidak membuat tabel di database. Tabel dibuat oleh file migrasi di `migrations/versions/`, yang dijalankan oleh `flask db upgrade`:

```
models.py                      → definisi tabel (ditulis manual)
   │  flask db migrate         → generate file migrasi
   ▼
migrations/versions/xxxx.py    → perintah create_table (hasil generate)
   │  flask db upgrade         → jalankan migrasi
   ▼
tabel di PostgreSQL
```

Jika Anda mengubah `models.py` (misalnya menambah kolom), jalankan `flask db migrate -m "deskripsi"` lalu `flask db upgrade`. Jangan hanya mengedit file migrasi lama, karena database yang sudah pernah di-upgrade tidak akan menjalankannya lagi.

### Setting `models.py`

File: [models.py](../models.py)

**a. Import tipe kolom dari SQLAlchemy:**

```python
from sqlalchemy import Column, DateTime, ForeignKey, Integer, Numeric, String, Text
```

**b. Model `Menu`:**

```python
class Menu(db.Model):
    __tablename__ = 'menu'
    id = Column(Integer, primary_key=True)
    item_name = Column(String(100))
    price = Column(Numeric(10, 2))
    description = Column(Text)
```

| Kolom | Tipe SQLAlchemy | Tipe di PostgreSQL | Keterangan |
|---|---|---|---|
| `id` | `Integer` (primary key) | `INTEGER` | ID menu |
| `item_name` | `String(100)` | `VARCHAR(100)` | Nama menu, maksimal 100 karakter |
| `price` | `Numeric(10, 2)` | `NUMERIC(10,2)` | Harga: 10 digit total, 2 di belakang koma |
| `description` | `Text` | `TEXT` | Deskripsi tanpa batas panjang |

> ⚠️ Untuk harga, gunakan `Numeric(10, 2)`, **bukan** `Decimal(10, 2)`. `Decimal` adalah tipe nilai Python (modul `decimal`), bukan tipe kolom SQLAlchemy, sehingga akan muncul error `name 'Decimal' is not defined`. Saat dibaca dari database, nilai `Numeric` otomatis dikembalikan sebagai `decimal.Decimal`.
>
> Gunakan `Text` untuk teks panjang tanpa batas, dan `String(n)` jika ingin membatasi panjang. Jangan menulis dua atribut `description` di class yang sama, karena yang terakhir menimpa yang pertama.

**c. Buat tabel `menu` lewat migrasi:**

File migrasi awal (`migrations/versions/d0c7b8e4b57c_initial_migration.py`) belum berisi tabel `menu`. `flask db upgrade` saja tidak cukup. Generate migrasi baru terlebih dahulu:

```bash
flask db migrate -m "add menu table"
flask db upgrade
```

Periksa file baru di `migrations/versions/` sebelum `upgrade`, pastikan isinya `op.create_table('menu', ...)`. Setelah berhasil, `psql -h db -U app_user -d restaurant_reviews -c "\dt"` akan menampilkan tabel `menu`.

Buka aplikasi di browser: <http://localhost:8000>

## Langkah 8 — Verifikasi

Di terminal container:

```bash
psql -h db -U app_user -d restaurant_reviews -c "\dt"
```

Password: `app_password`

Jika muncul tabel `restaurant`, `review`, dan `alembic_version` (serta `menu` setelah migrasi di bagian "Setting `models.py`"), berarti penggantian nama database **berhasil** ✅

Jika `psql` tidak tersedia, cek lewat extension **SQLTools** di sidebar VS Code (koneksi "Container database").

## Checklist

- [ ] `.env` → `DBNAME=restaurant_reviews`
- [ ] `docker-compose.yml` → `POSTGRES_DB: restaurant_reviews`
- [ ] `docker-compose.yml` → healthcheck `pg_isready -U app_user -d restaurant_reviews -h localhost`
- [ ] `devcontainer.json` → `"database": "restaurant_reviews"`
- [ ] `docker compose config` valid
- [ ] Volume lama dihapus (jika ada)
- [ ] Container berhasil dibuka/rebuild
- [ ] `flask db upgrade` berhasil
- [ ] `models.py` → `Menu` memakai `Numeric(10, 2)` dan `Text`, lalu `flask db migrate` + `flask db upgrade` dijalankan
- [ ] Tabel muncul di database `restaurant_reviews`

## Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine` | Docker Desktop belum berjalan | Buka Docker Desktop, tunggu sampai "Engine running". Jika tetap gagal, jalankan `wsl --update` lalu `wsl --shutdown` sebagai Administrator. |
| `database "restaurant_reviews" does not exist` | Volume lama masih berisi database `app` | Lakukan Langkah 5, lalu Rebuild Container. |
| Container `app` tidak pernah start / `dependency failed to start: container ... is unhealthy` | Perintah healthcheck salah | Periksa Langkah 2b: `-U app_user -d restaurant_reviews`. |
| `could not translate host name "db"` | Flask dijalankan di Windows, bukan di dalam container | Jalankan Flask dari terminal di dalam Dev Container, atau ubah `DBHOST=localhost` jika PostgreSQL dijalankan terpisah dengan port 5432 terbuka. |
| `NameError: name 'Decimal' is not defined` | `Decimal` dipakai sebagai tipe kolom di `models.py` | Ganti dengan `Numeric(10, 2)` dan import `Numeric` dari `sqlalchemy`. |
| Tabel `menu` tidak muncul setelah `flask db upgrade` | Belum ada file migrasi untuk `Menu` | Jalankan `flask db migrate -m "add menu table"` lalu `flask db upgrade`. |
| `ModuleNotFoundError: No module named 'flask_migrate'` | Dependency belum diinstall | Jalankan `pip install -r requirements.txt`. |

## Catatan

- Folder `migrations/` tidak menyimpan nama database, jadi tidak perlu diubah.
- [.github/workflows/python-test.yaml](../.github/workflows/python-test.yaml) memakai database `postgres` khusus untuk CI, jadi tidak terpengaruh.
- Untuk deployment ke Azure (`azd up`), nama database ditentukan oleh Bicep di [infra/resources.bicep](../infra/resources.bicep) (`<nama-app>-database`), bukan oleh file-file di atas.

## Perbandingan: Menjalankan di GitHub Codespaces

> Bagian ini hanya sebagai bahan perbandingan. Langkah utama di atas tetap untuk menjalankan secara lokal dengan Docker Desktop.

Di Codespaces, container dijalankan di server GitHub, jadi beberapa langkah bisa dilewati.

| Langkah | Lokal (Docker Desktop) | Codespaces |
|---|---|---|
| Prasyarat | Docker Desktop + VS Code + extension Dev Containers | Cukup akun GitHub dan browser |
| 1–3 Ubah `.env`, compose, `devcontainer.json` | Wajib | Cukup pastikan perubahan sudah di-commit dan di-push ke GitHub |
| 4 Validasi compose | Disarankan | Boleh dilewati. Jika file salah, Codespace gagal dibuat dan log-nya langsung terlihat |
| 5 Hapus volume lama | Wajib jika pernah dijalankan | Dilewati. Codespace baru selalu mulai dengan volume kosong |
| 6 Reopen in Container | Wajib | Otomatis. Codespace langsung terbuka di dalam container |
| 7 Install, migrasi, jalankan | Wajib | **Tetap wajib** |
| 8 Verifikasi | Wajib | Tetap dilakukan |

### Langkah di Codespaces

1. Buka repository di GitHub, lalu klik **Code → Codespaces → Create codespace on main**.
2. Tunggu sampai VS Code (versi browser) terbuka. Terminalnya sudah otomatis berada di dalam container (`vscode ➜ /workspace $`).
3. Jalankan:

   ```bash
   pip install -r requirements.txt
   flask db upgrade
   flask run --host 0.0.0.0 --port 8000
   ```

4. Saat muncul notifikasi **"Your application running on port 8000 is available"**, klik **Open in Browser**.

### Perbedaan yang perlu diperhatikan

- **Alamat aplikasi** bukan `localhost:8000`, melainkan `https://<nama-codespace>-8000.app.github.dev` (lihat tab **Ports**).
- **Troubleshooting lebih sedikit.** Error Docker Desktop/WSL dan `could not translate host name "db"` praktis tidak muncul, karena terminal selalu berada di dalam container.
- **Jika file `.devcontainer/` diubah** setelah Codespace dibuat, jalankan `F1` → **Codespaces: Rebuild Container**. Volume database lama tetap ada saat rebuild. Untuk database yang benar-benar bersih, hapus Codespace lalu buat yang baru.
- **Kuota.** Akun gratis memiliki jatah jam per bulan. Lakukan **Stop** atau **Delete** pada Codespace di <https://github.com/codespaces> setelah selesai.
- **Opsional:** uncomment `"postCreateCommand": "pip install -r requirements.txt"` di `devcontainer.json` agar `pip install` berjalan otomatis saat container dibuat. Ini berlaku untuk Codespaces maupun lokal.
- **Keamanan:** file `.env` ikut ter-commit ke repository. Ini aman untuk password latihan seperti `app_password`, tetapi jangan pernah menyimpan password asli atau kredensial Azure di file tersebut.
