#### Drop Table Restaurant and Review
```sql
DROP TABLE restaurant,review
```


#### Create Table Restaurant
```sql
CREATE TABLE restaurant (
	id SERIAL PRIMARY KEY,
	name VARCHAR(100) NOT NULL,
	street_address VARCHAR(250) NOT NULL,
	description VARCHAR(250) NOT NULL
);
```


#### Create Table review
```sql
CREATE TABLE review
(
	id SERIAL PRIMARY KEY,
	restaurant_id INT REFERENCES restaurant(id) ON DELETE CASCADE ON UPDATE CASCADE,
	user_name VARCHAR(100),
	rating INT CHECK(rating >=1 AND rating <=5),
	review_text TEXT,
	review_date DATE
);
```

#### Insert Data Restaurant
```sql
INSERT INTO restaurant (name, street_address, description) VALUES
('Warung Makan Sederhana', 'Jl. Sudirman No. 12, Pekanbaru', 'Masakan Padang dengan menu rendang dan gulai ikan.'),
('Kopi Senja', 'Jl. Soebrantas No. 45, Pekanbaru', 'Kedai kopi dengan suasana santai, cocok untuk nongkrong dan bekerja.'),
('Sate Pak Haji', 'Jl. Arifin Ahmad No. 8, Pekanbaru', 'Sate ayam dan kambing dengan bumbu kacang khas.');
```

#### Insert Data Review
Asumsi: `id` restoran adalah 1 (Warung Makan Sederhana), 2 (Kopi Senja), dan 3 (Sate Pak Haji), sesuai urutan insert pada tabel yang masih kosong. Cek dulu dengan `SELECT id, name FROM restaurant;`.

```sql
INSERT INTO review (restaurant_id, user_name, rating, review_text, review_date) VALUES
-- Warung Makan Sederhana
(1, 'Andi Saputra', 5, 'Rendangnya empuk dan bumbunya meresap. Porsi besar, harga bersahabat.', '2026-09-02'),
(1, 'Dewi Lestari', 3, 'Rasa lumayan, tetapi nasinya agak dingin ketika disajikan.', '2026-09-14'),

-- Kopi Senja
(2, 'Fajar Nugraha', 5, 'Suasananya nyaman, wifi kencang, cocok untuk kerja seharian.', '2026-09-03'),
(2, 'Maya Anggraini', 4, 'Kopi susu gula aren-nya pas manisnya. Colokan listrik agak kurang.', '2026-09-07'),

-- Sate Pak Haji
(3, 'Agus Salim', 5, 'Sate kambingnya empuk tanpa bau prengus. Bumbu kacangnya gurih.', '2026-09-01'),
(3, 'Citra Dewi', 3, 'Rasanya enak, tetapi tempat parkir sempit dan agak sulit dicari.', '2026-09-16');
```

Cek hasilnya:

```sql
SELECT r.name, COUNT(*) AS jumlah_review, ROUND(AVG(v.rating), 2) AS rata_rata
FROM review v
JOIN restaurant r ON r.id = v.restaurant_id
GROUP BY r.name
ORDER BY r.name;
```


## C. Performing CRUD Operations

Urutan di bawah dibuat berkelanjutan, jadi jalankan dari atas ke bawah. Asumsi data awal: restoran `id` 1–3 dan review `id` 1–6 (dari bagian Insert di atas).

### 1. Create (Insert)

**Insert a new restaurant into the restaurant table**

```sql
INSERT INTO restaurant (name, street_address, description)
VALUES ('Bakso Mas Joko', 'Jl. Tuanku Tambusai No. 3, Pekanbaru', 'Bakso urat dan bakso telur dengan kuah kaldu sapi.');
```

**Insert a new review for an existing restaurant**

```sql
INSERT INTO review (restaurant_id, user_name, rating, review_text, review_date)
VALUES (1, 'Rudi Hermawan', 4, 'Gulai ayamnya enak dan porsinya pas. Pelayanan cepat.', '2026-10-05');
```

### 2. Read (Select)

**Retrieve all reviews for a specific restaurant using the `restaurant_id`**

```sql
SELECT * FROM review WHERE restaurant_id = 1;
```

**Retrieve all reviews with a rating of 4 or higher**

```sql
SELECT * FROM review WHERE rating >= 4;
```

**Use a JOIN to display a list of restaurants along with their reviews**

```sql
SELECT r.id AS restaurant_id,
       r.name AS restaurant_name,
       v.id AS review_id,
       v.user_name,
       v.rating,
       v.review_text,
       v.review_date
FROM restaurant r
LEFT JOIN review v ON v.restaurant_id = r.id
ORDER BY r.id, v.id;
```

`LEFT JOIN` dipakai supaya restoran yang belum punya review (misalnya "Bakso Mas Joko") tetap tampil. Ganti dengan `INNER JOIN` jika hanya ingin restoran yang sudah punya review.

### 3. Update

**Update the description of one restaurant**

```sql
UPDATE restaurant
SET description = 'Masakan Padang dengan rendang, gulai ikan, dan ayam pop.'
WHERE id = 1;
```

**Update the rating of a specific review**

```sql
UPDATE review
SET rating = 5
WHERE id = 2;
```

### 4. Delete

**Delete one review based on id**

```sql
DELETE FROM review WHERE id = 7;
```

**Delete a restaurant and ensure its associated reviews are also deleted (using cascade)**

Tabel `review` dibuat dengan `restaurant_id INT REFERENCES restaurant(id) ON DELETE CASCADE`, jadi menghapus restoran otomatis menghapus semua review miliknya.

Tambahkan dulu satu review untuk restoran baru agar efek cascade terlihat:

```sql
INSERT INTO review (restaurant_id, user_name, rating, review_text, review_date)
SELECT id, 'Sari Wulandari', 5, 'Baksonya kenyal dan kuahnya gurih.', '2026-10-06'
FROM restaurant WHERE name = 'Bakso Mas Joko';
```

Hapus restoran tersebut:

```sql
DELETE FROM restaurant WHERE name = 'Bakso Mas Joko';
```

Verifikasi bahwa review-nya ikut terhapus (hasil harus 0 baris):

```sql
SELECT * FROM review WHERE user_name = 'Sari Wulandari';
```

> ⚠️ Selalu pakai `WHERE` pada `UPDATE` dan `DELETE`. Tanpa `WHERE`, semua baris ikut berubah atau terhapus.


## D. Additional Queries

### 1. Find the highest-rated restaurant based on the average rating of all its reviews

```sql
SELECT r.id,
       r.name,
       ROUND(AVG(v.rating), 2) AS rata_rata_rating
FROM restaurant r
JOIN review v ON v.restaurant_id = r.id
GROUP BY r.id, r.name
ORDER BY AVG(v.rating) DESC
LIMIT 1;
```

Jika ada dua restoran atau lebih dengan rata-rata yang sama dan semuanya ingin ditampilkan, gunakan versi ini:

```sql
SELECT id, name, rata_rata_rating
FROM (
    SELECT r.id,
           r.name,
           ROUND(AVG(v.rating), 2) AS rata_rata_rating,
           RANK() OVER (ORDER BY AVG(v.rating) DESC) AS peringkat
    FROM restaurant r
    JOIN review v ON v.restaurant_id = r.id
    GROUP BY r.id, r.name
) t
WHERE peringkat = 1;
```

### 2. Find the number of reviews each restaurant has received

```sql
SELECT r.id,
       r.name,
       COUNT(v.id) AS jumlah_review
FROM restaurant r
LEFT JOIN review v ON v.restaurant_id = r.id
GROUP BY r.id, r.name
ORDER BY jumlah_review DESC, r.id;
```

Memakai `LEFT JOIN` dan `COUNT(v.id)` (bukan `COUNT(*)`) supaya restoran tanpa review tetap tampil dengan jumlah 0.

### 3. Display the most recent review for each restaurant

```sql
SELECT DISTINCT ON (r.id)
       r.id AS restaurant_id,
       r.name AS restaurant_name,
       v.id AS review_id,
       v.user_name,
       v.rating,
       v.review_text,
       v.review_date
FROM restaurant r
JOIN review v ON v.restaurant_id = r.id
ORDER BY r.id, v.review_date DESC, v.id DESC;
```

`DISTINCT ON (r.id)` adalah fitur khusus PostgreSQL: dari tiap grup `r.id` hanya baris pertama yang diambil, yaitu review dengan `review_date` terbaru (jika tanggalnya sama, dipilih `id` terbesar). Alternatif standar SQL:

```sql
SELECT r.id AS restaurant_id,
       r.name AS restaurant_name,
       v.id AS review_id,
       v.user_name,
       v.rating,
       v.review_text,
       v.review_date
FROM restaurant r
JOIN review v ON v.restaurant_id = r.id
WHERE v.id = (
    SELECT v2.id
    FROM review v2
    WHERE v2.restaurant_id = r.id
    ORDER BY v2.review_date DESC, v2.id DESC
    LIMIT 1
);
```


## Extra Credit (Optional)

### 1. Create a menu table and insert at least 3 menu items for each restaurant

Struktur tabel mengikuti model `Menu` di [models.py](../models.py): harga memakai `NUMERIC(10,2)`, deskripsi memakai `TEXT`, dan nama menu tidak boleh sama dalam satu restoran.

```sql
CREATE TABLE menu (
	id SERIAL PRIMARY KEY,
	restaurant_id INT NOT NULL REFERENCES restaurant(id) ON DELETE CASCADE ON UPDATE CASCADE,
	item_name VARCHAR(100) NOT NULL,
	price NUMERIC(10, 2) NOT NULL,
	description TEXT,
	CONSTRAINT uq_restaurant_item UNIQUE (restaurant_id, item_name)
);
```

Insert 3 menu untuk masing-masing restoran. Restoran dicari berdasarkan nama, sehingga tidak bergantung pada nilai `id`:

```sql
INSERT INTO menu (restaurant_id, item_name, price, description)
SELECT r.id, m.item_name, m.price, m.description
FROM (VALUES
	-- Warung Makan Sederhana
	('Warung Makan Sederhana', 'Nasi Rendang',   28000.00, 'Nasi putih dengan rendang daging sapi, sayur daun singkong, dan sambal hijau.'),
	('Warung Makan Sederhana', 'Gulai Ikan',     26000.00, 'Gulai ikan kakap dengan kuah santan kental dan rempah khas Minang.'),
	('Warung Makan Sederhana', 'Ayam Pop',       25000.00, 'Ayam kampung rebus goreng sebentar, disajikan dengan sambal tomat.'),

	-- Kopi Senja
	('Kopi Senja', 'Kopi Susu Gula Aren',        22000.00, 'Espresso, susu segar, dan gula aren dengan es batu.'),
	('Kopi Senja', 'Caffe Latte',                24000.00, 'Espresso dengan susu steamed, tersedia panas atau dingin.'),
	('Kopi Senja', 'Croissant Butter',           18000.00, 'Croissant panggang dengan mentega, renyah di luar dan lembut di dalam.'),

	-- Sate Pak Haji
	('Sate Pak Haji', 'Sate Ayam (10 tusuk)',    25000.00, 'Sate ayam dengan bumbu kacang dan lontong.'),
	('Sate Pak Haji', 'Sate Kambing (10 tusuk)', 40000.00, 'Sate kambing muda dengan bumbu kecap dan acar bawang.'),
	('Sate Pak Haji', 'Lontong',                  5000.00, 'Lontong pendamping sate.')
) AS m(restaurant_name, item_name, price, description)
JOIN restaurant r ON r.name = m.restaurant_name;
```

Cek hasilnya (tiap restoran harus punya 3 menu):

```sql
SELECT r.name AS restaurant_name, COUNT(m.id) AS jumlah_menu
FROM restaurant r
LEFT JOIN menu m ON m.restaurant_id = r.id
GROUP BY r.name
ORDER BY r.name;
```

### 2. Display each restaurant with its menu and the average rating from its reviews

```sql
SELECT r.id AS restaurant_id,
       r.name AS restaurant_name,
       m.item_name,
       m.price,
       rt.rata_rata_rating
FROM restaurant r
LEFT JOIN menu m ON m.restaurant_id = r.id
LEFT JOIN (
	SELECT restaurant_id,
	       ROUND(AVG(rating), 2) AS rata_rata_rating
	FROM review
	GROUP BY restaurant_id
) rt ON rt.restaurant_id = r.id
ORDER BY r.id, m.id;
```

Rata-rata rating dihitung lebih dulu per restoran di dalam subquery, baru digabungkan dengan menu. Kalau `review` dan `menu` di-`JOIN` langsung dalam satu query, setiap review akan terduplikasi sebanyak jumlah menu, sehingga hitungan seperti `COUNT` menjadi salah. Memakai subquery menghindari masalah itu.
