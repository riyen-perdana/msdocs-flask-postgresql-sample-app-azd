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

Buka aplikasi di browser: <http://localhost:8000>

## Langkah 8 — Verifikasi

Di terminal container:

```bash
psql -h db -U app_user -d restaurant_reviews -c "\dt"
```

Password: `app_password`

Jika muncul tabel `restaurant`, `review`, dan `alembic_version`, berarti penggantian nama database **berhasil** ✅

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
- [ ] Tabel muncul di database `restaurant_reviews`

## Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine` | Docker Desktop belum berjalan | Buka Docker Desktop, tunggu sampai "Engine running". Jika tetap gagal, jalankan `wsl --update` lalu `wsl --shutdown` sebagai Administrator. |
| `database "restaurant_reviews" does not exist` | Volume lama masih berisi database `app` | Lakukan Langkah 5, lalu Rebuild Container. |
| Container `app` tidak pernah start / `dependency failed to start: container ... is unhealthy` | Perintah healthcheck salah | Periksa Langkah 2b: `-U app_user -d restaurant_reviews`. |
| `could not translate host name "db"` | Flask dijalankan di Windows, bukan di dalam container | Jalankan Flask dari terminal di dalam Dev Container, atau ubah `DBHOST=localhost` jika PostgreSQL dijalankan terpisah dengan port 5432 terbuka. |
| `ModuleNotFoundError: No module named 'flask_migrate'` | Dependency belum diinstall | Jalankan `pip install -r requirements.txt`. |

## Catatan

- Folder `migrations/` tidak menyimpan nama database, jadi tidak perlu diubah.
- [.github/workflows/python-test.yaml](../.github/workflows/python-test.yaml) memakai database `postgres` khusus untuk CI, jadi tidak terpengaruh.
- Untuk deployment ke Azure (`azd up`), nama database ditentukan oleh Bicep di [infra/resources.bicep](../infra/resources.bicep) (`<nama-app>-database`), bukan oleh file-file di atas.
