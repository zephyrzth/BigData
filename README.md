# Tugas 1 Big Data
Nama          : Anargya Widyadhana

NRP           : 05111740000047

Mata kuliah   : Big Data

## Daftar File

* KNIME workflow    : `Tugas 1 BigData.knwf`
* File dataset  : `tc20171021.csv`
* Deskripsi dataset : Data ini diperoleh dengan menggunakan TrueCar.com untuk daftar mobil bekas pada 9/24/2017. Setiap baris mewakili satu daftar mobil bekas. Data termasuk: tahun, merek, model, harga, VIN, kota, negara bagian
* Sumber dataset    : https://www.kaggle.com/jpayne/852k-used-car-listings#tc20171021.csv

---
## Business Understanding

Dalam file dataset tersebut, data adalah berupa satu file `.csv`. Maka dari data tersebut tidak bisa dilakukan join, kecuali datanya dipisah terlebih dahulu. Beberapa proses yang dapat dilakukan pada data ini adalah:
* Selection (seleksi data baik berdasarkan baris maupun kolom) untuk menyederhanakan data atau membuang beberapa baris/kolom.
* Transformation (transformasi data ke bentuk lain) untuk mentransformasi data ke bentuk lain, bisa berupa file baru (`.csv, JSON, dll.`) maupun dalam bentuk database.
* Split (membagi data) untuk membagi data sesuai yang diinginkan, baik membagi baris ataupun kolom.
* Join (menggabungkan data) untuk menggabungkan dua atau lebih data yang bisa berasal dari berbagai sumber (file ataupun database) menjadi satu tabel. Data bisa digabung menurut baris (concatenate) atau kolom (column append).

## Data Understanding

![Isi Data](/images/isi_dataset.png)

* Jumlah baris: 1.233.042
* Makna kolom:
    1. id: identifier unik pada tabel
    2. price: harga mobil tercatat
    3. year: tahun model dari kendaraan
    4. mileage: jumlah mil dari odometer kendaraan
    5. city: kota tempat kendaraan tercatat
    6. state: state tempat kendaraan tercatat
    7. vin: vehicle identification number
    8. make: produsen mobil
    9. model: model mobil

## Data Preparation

Di sini data akan dibagi (split) menjadi 2 bagian. Bagian pertama berisikan kolom city, state, vin, make, model, yang akan disimpan ke dalam file `.csv`. Dan bagian kedua berisi kolom id, price, year, mileage, yang akan disimpan ke dalam database `mysql`. Untuk pembagian data, langsung menggunakan node pada KNIME. Langkah-langkah:
1. Instalasi KNIME (Windows):
    
    Lakukan download pada https://www.knime.com/downloads/download-knime, dan instalasi seperti biasa.

2. Pada workspace baru, dibuat node `File Reader` untuk membaca file dataset yang dihubungkan ke node `Column Splitter` untuk membagi kolom. Sebelum dijalankan, dilakukan konfigurasi pada setiap node. Pada konfigurasi File Reader, centang `read column headers`, ganti delimiter menjadi `,`, dan centang `allow short lines` pada tab `Advanced`.

    ![Split Kolom](/images/1.png)

    ![Split Kolom](/images/2.png)

3. Apply dan lakukan execute. Jika dilihat secara `File Table` akan terlihat seperti berikut:

    ![Split Kolom](/images/3.png)

4. Pada konfigurasi Column Splitter, data dipisah berdasarkan tipe data, menghasilkan 2 buah tabel sesuai pembagian di atas.

    ![Split Kolom](/images/4.png)

5. Data sudah terpisah dan proses pembagian data selesai. Berikutnya, 1 tabel yang bertipe data `int` (terdapat kolom `id`) disimpan ke dalam database, sedangkan 1 tabel lainnya bertipe data `string` disimpan menjadi file `.csv`. Di sini node yang perlu ditambahkan adalah:
    * `MySQL Connector` sebagai node konektor ke MySQL
    * `DB Table Creator` sebagai node untuk membuat tabel di database
    * `DB Writer` untuk memasukkan data ke dalam tabel yang sudah dibuat
    * `CSV Writer` untuk membuat file `.csv` dari data input

    Dan didapatkan skema seperti ini.

    ![Split Kolom](/images/5.png)

6. Berikutnya dilakukan konfigurasi koneksi database pada node `MySQL Connector`, informasi tabel pada node `DB Table Creator`, informasi data diinput ke db di `DB Writer`, dan informasi data untuk ke file csv di `CSV Writer`.

    * File csv yang terbuat akan diberi nama: `car_used_split2.csv`
    * Tabel database baru yang terbuat diberi nama: `car_used_split`

    ![Simpan DB dan CSV](/images/6.png)

    ![Simpan DB dan CSV](/images/7.png)

    ![Simpan DB dan CSV](/images/8.png)

    ![Simpan DB dan CSV](/images/9.png)

    Jika diexecute sesuai urutan skema, maka salah satu data akan masuk ke file csv dan sisanya ke database. Data yang sudah terpisah ini akan digunakan di proses berikutnya.

## Modeling

Di sini akan dilakukan proses pembacaan data melalui dua sumber yang berbeda. Dari proses sebelumnya, maka akan dibaca lagi dengan node pembaca database dan csv, dan dilakukan append/join keduanya.

1. Dibuat 2 node, yaitu `DB Reader` untuk membaca tabel dari database, dan `File Reader` untuk membaca file. Pastikan node `DB Reader` terhubung ke node sebelumnya (bisa oleh node `DB Writer`). Terakhir lakukan konfigurasi pada `File Reader` sama persis seperti sebelumnya (nama file yang dibaca: `car_used_split2.csv`). Untuk `DB Reader` tidak perlu konfigurasi karena data output dari `DB Writer` sebelumnya sudah berupa data dari tabel `car_used_split`. Dan diperoleh skema berikut.

    ![Simpan DB dan CSV](/images/10.png)

    ![Simpan DB dan CSV](/images/11.png)

    ![Simpan DB dan CSV](/images/12.png)

2. Lalu, akan dilakukan append kolom kedua tabel dengan node `Column Appender`. Sebenarnya dengan node `Joiner` tetap bisa, karena saat data diload dari database maupun csv, KNIME sudah membuat kolom `Row ID` sehingga tetap ada kolom yang sama dan bisa dilakukan `JOIN`. Tetapi hasilnya akan sama dengan append kolom karena yang kita joinkan dicocokkan berdasarkan nomor baris, bukan id tabel. Lalu akan didapat skema berikut.

    ![Simpan DB dan CSV](/images/13.png)

    ![Simpan DB dan CSV](/images/16.png)

    ![Simpan DB dan CSV](/images/14.png)

3. Proses append kolom selesai.

## Evaluation

Dapat dilihat dari proses sebelumnya dan pada data awal, bahwa proses append kolom/join yang dilakukan berhasil karena data sesuai kondisi awal, dan bisa dilihat pada gambar di atas.

## Deployment

Di sini data hasil append/join tersebut akan disimpan ke dalam file .csv bernama `car_used_completed.csv` dan database MySQL dengan nama tabel `car_used_completed`. Sama seperti sebelumnya, sebelum disimpan ke database perlu untuk membuat tabel dengan `DB Table Creator`. Dan didapatkan skema berikut:

![Simpan DB dan CSV](/images/15.png)

![Simpan DB dan CSV](/images/17.png)

![Simpan DB dan CSV](/images/18.png)

Data sudah berhasil disimpan dalam database dan file csv.

