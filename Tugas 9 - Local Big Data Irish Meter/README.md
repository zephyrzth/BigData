# Tugas Big Data - Local Big Data Irish Meter

Nama          : Anargya Widyadhana

NRP           : 05111740000047

Mata kuliah   : Big Data

Dataset       : `Irish Energy Meter`

## Section

Ada 6 tahapan CRISP-DM

- [Daftar File](#daftar-file)
- [Business Understanding](#business-understanding)
- [Data Understanding](#data-understanding)
- [Data Preparation](#data-preparation)
- [Modeling](#modeling)
- [Evaluation](#evaluation)
- [Deployment](#deployment)


## Daftar File

* KNIME workflow    : `Tugas_9_Big_Data_Irish_Meter_on_Spark_only.knwf`
* File dataset      : Digunakan `knime://knime.workflow/data/meters_01_50.csv`
* Deskripsi dataset : Merupakan data penggunaan listrik dalam timestamp, dan nilainya dalam satuan kW
* Sumber dataset    : Sudah ada dari KNIME

---

![Workflow](images/workflow.png)

Dalam workflow ini, akan dijelaskan mengenai analisis yang berasal dari whitepaper "Big Data, Smart Energy, and Predictive Analytics". Akan dibuat Local Big Data Enviroment, yang akan meload dataset ke Hive, dan ditransfer ke Spark untuk dilakukan klustering. Akan menggunakan node Spark SQL untuk operasi SQL untuk menambahkan detail waktu terkait dengan data tanggal dan waktu di dataset agar diperoleh data waktu yang bervariasi. Terdapat juga metanode untuk melakukan klustering dengan k-Means dan PCA, dan menampilkan visualisasi kluster dalam grafik. Lalu, terakhir akan disimpan hasil kluster tersebut ke dalam format Hive dan Parquets


## Business Understanding

Kali ini, akan digunakan konsep Big Data, sehingga akan ada penyimpanan data menggunakan Hive, dan proses pengolahan data klustering dengan menggunakan Apache Spark, node `Spark k-Means` dan `Spark PCA`. Oleh karena itu, pertama akan dibuat sebuah environment Big Data untuk tersambung ke konteks Spark dan Hive, lalu dilakukan proses Data Preparation untuk menyiapkan data yang ada dan disimpan ke Hive. Kemudian selanjutnya dari Hive akan diload ke Spark untuk diolah. Pertama akan dibuat pemecahan data tanggal dan waktu dari dataset menjadi timeseries yang lebih lengkap, seperti pemecahan tanggal, bulan, tahun, minggu, hari dalam minggu, dsb. Dari data ini, berikutnya akan dilakukan melalui sebuah rangkaian node untuk mencari rata-rata nilai per kategori timeseries, dijoin menjadi satu tabel, baru kemudian dilakukan modeling dari data di tabel tersebut.

Pada proses modeling, data akan dinormalisasi dan dilakukan klustering dengan node `Spark PCA` dan `Spark k-Means`. Hasilnya dijoin dan didenormalisasi lagi, baru dilakukan evaluation dan deployment.
Dalam evaluation, dilakukan plotting grafik dengan node `Scatter Plot`, dan view tabel dengan node `Table View`. Pada deployment, data akan disimpan dalam format Hive dan Parquet.


## Data Understanding

![Isi Data](images/isi_dataset.png)

* Jumlah data: 1226830
* Makna kolom:
    1. meterId: id meteran yang diukur, int
    2. enc_datetime: data tanggal bulan tahun jam menit detik dalam format int timestamp
    3. reading: jumlah kW penggunaan listrik, double


## Data Preparation

Sesuai dengan proses pada `Business Understanding`, pertama akan dibuat environment Big Data. Kita menggunakan node `Create Local Big Data Environment` seperti pada gambar di bawah.

![Create Local Big Data Environment node](images/big-data-env-var.png)

![Create Local Big Data Environment node setting](images/big-data-env-var-setting.png)

Digunakan 2 thread.

Data yang akan kita lakukan train dan predict berasal dari `knime://knime.workflow/data/meters_01_50.csv`. Maka setelah membuat Big Data Environment, file `.csv` akan diload ke KNIME Table melalui node `File Reader`.

![File Reader](images/file-reader.png)

![File Reader setting](images/file-reader-setting.png)

Selanjutnya, data diload ke Hive di dalam metanode `Load Data`.

![Load Data](images/load-data.png)

Sebelum data diload, akan dibuat tabel di Hive dengan node `DB Table Creator` yang tersambung ke konteks Hive. Setting adalah dengan dynamic, menyesuaikan kolom pada tabel yang dijadikan input port.

![DB Table Creator](images/db-table-creator.png)

![DB Table Creator setting](images/db-table-creator-setting.png)

Setelah tabel dibuat, barulah kita load data dari KNIME table ke Hive dengan node `DB Loader` (setting sudah secara otomatis terbuat mengikuti konteks tabel yang terhubung).

![DB Loader](images/db-loader.png)

![DB Loader setting](images/db-loader-setting.png)

Lalu, karena di dalam modeling digunakan node dalam konteks Spark, maka di sini kita load data dari Hive ke Spark dengan node `Hive to Spark`

![Hive to Spark](images/hive-to-spark.png)

Lalu dilakukan proses memasukkan variasi timeseries, yang berada di metanode `Extract date-time attributes`. Pada metanode tersebut, terdapat `4` node `Spark SQL Query`, yang pertama untuk mendapat tanggal (jika di kolom menggunakan int timestamp atau format belum sesuai) dengan menambah 3 digit pertama int timestamp dengan tanggal `2008-12-31` dengan query SQL. Untuk jam dan menit, dicari dengan digit ke 4-5, dengan mengkali dengan `* 30 / 60`, diikuti `% 24` untuk jam, dan `* 30 % 60` untuk menit. Nilai jam dan menit digabung dan dipisah dengan tanda `:`.

![Datetime Conversion](images/datetime-conversion.png)

Pada node kedua, data datetime akan dipecah menjadi kolom tanggal, bulan, tahun, minggu, dan hari dalam format string, dan jammasing-masing.

![Datetime Conversion 2](images/datetime-conversion-2.png)

Pada node ketiga dilakukan yang lebih spesifik yaitu mengkategorikan hari berdasarkan kategori hari kerja atau hari libur dengan kode `BD` dan `WE`.

![Datetime Conversion 3](images/datetime-conversion-3.png)

Pada node keempat dilakukan segmen hari, yaitu dengan mengkategorikan berdasarkan rentang jam, rentang `7-9`, `9-13`, `13-17`, `17-21`, `21-7`.

![Datetime Conversion 4](images/datetime-conversion-4.png)

Berikutnya adalah proses pada metanode `Aggregations and time series`. Pada metanode ini akan ada `5` pecahan proses, untuk mencari rata-rata nilai pada kolom `Daily minimum temperatures`. Sebelumnya, karena data digunakan berkali-kali, agar cepat dilakukan caching dengan node `Persist Spark DataFrame/RDD` ke dalam memory.

![Cache](images/cache.png)

Pada pecahan 1, dicari rata-rata nilai berdasarkan total, atau hanya dari id saja, dan kolom hasilnya akan direname untuk memudahkan.

![Aggregate 1](images/aggregate-1.png)

Pada pecahan 2, dicari rata-rata nilai per tahun dan per id. Pertama akan dijumlah nilai berdasarkan dengan `sum` group by `id` dan `year`, lalu untuk setiap `id`, data nilai per tahun tersebut akan dirata-rata dengan `mean`. Terakhir, dilakukan rename kolom seperti tadi.

![Aggregate 2](images/aggregate-2.png)

Pada pecahan 3, dicari rata-rata nilai per bulan, dengan group by bulan, tahun, id. Pertama akan dijumlah nilai berdasarkan dengan `sum` group by `id`, `month`, dan `year`, lalu untuk setiap `id`, data nilai per tahun tersebut akan dirata-rata dengan `mean`. Terakhir, dilakukan rename kolom seperti tadi.

![Aggregate 3](images/aggregate-3.png)

Pada pecahan 4, dicari rata-rata nilai tiap minggu, dengan group by minggu, tahun, id. Pertama akan dijumlah nilai berdasarkan dengan `sum` group by `id`, `week`, dan `year`, lalu untuk setiap `id`, data nilai per tahun tersebut akan dirata-rata dengan `mean`. Terakhir, dilakukan rename kolom seperti tadi.

![Aggregate 4](images/aggregate-4.png)

Pada pecahan 5, dicari rata-rata string hari per minggu, dengan group by year, week, dayofweek, id. Pertama akan dijumlah nilai berdasarkan dengan `sum` group by `id`, `dayofweek`, `week`, `year`. Kali ini menggunakan tabel pivot karena, nantinya ingin mendapatkan jumlah energi kW per satuan hari (Monday, Tuesday, dst.) per meterId, sehingga hasil akhirnya bisa banyak meterId yang sama lebih dari satu bersarkan harinya. Hal ini akan menimbulkan masalah saat nanti dilakukan join karena jumlah data berbeda. Oleh karena itu, dengan pivot, data hari tidak akan melebar ke bawah, tetapi ke samping dengan menambah field baru. Lalu dihitung rata-rata per harinya, dan hasil kolomnya direname seperti sebelumnya.

![Aggregate 5](images/aggregate-5.png)

Pada pecahan 6, dicari rata-rata nilai tiap hari, dengan group by eventDate (tanggal lengkap), id. Pertama akan dijumlah nilai berdasarkan dengan `sum` group by `id`, `eventDate`, lalu untuk setiap `id`, data nilai per tahun tersebut akan dirata-rata dengan `mean`. Terakhir, dilakukan rename kolom seperti tadi.

![Aggregate 6](images/aggregate-6.png)

Pada pecahan 7, dicari rata-rata nilai tiap segmen hari, dengan group by eventDate (tanggal lengkap), id, dan daySegment. Pertama akan dijumlah nilai berdasarkan dengan `sum` group by `id`, `eventDate`, `daySegment`, lalu dibuat pivot lagi, dan untuk setiap `id`, data nilai per tahun tersebut akan dirata-rata dengan `mean`. Terakhir, dilakukan rename kolom seperti tadi.

![Aggregate 7](images/aggregate-7.png)

Pada pecahan 8, dicari rata-rata nilai tiap classifier hari, dengan group by year, month, week, id, dan dayClassifier. Pertama akan dijumlah nilai berdasarkan dengan `sum` group by `year`, `month`, `week`, `id`, `dayClassifier`, lalu dibuat pivot lagi, dan untuk setiap `id`, data nilai per tahun tersebut akan dirata-rata dengan `mean`. Terakhir, dilakukan rename kolom seperti tadi.

![Aggregate 8](images/aggregate-8.png)

Pada pecahan 9, dicari rata-rata nilai tiap jam, dengan group by eventDate (tanggal lengkap), id, dan hour. Pertama akan dijumlah nilai berdasarkan dengan `sum` group by `id`, `eventDate`, `hour`, lalu dibuat pivot lagi, dan untuk setiap `id`, data nilai per tahun tersebut akan dirata-rata dengan `mean`. Terakhir, dilakukan rename kolom seperti tadi.

![Aggregate 9](images/aggregate-9.png)

Berikutnya semua data pada pecahan masing-masing akan dijoin dan jadi satu tabel.

![Joiner](images/joiner.png)

Selanjutnya, akan dicari persentase data masing-masing dayofweek terhadap minggu, masing-masing segmen hari terhadap hari, melalui `Spark SQL Query`.

![Percentage](images/percentage.png)

Semua proses di atas jika digabungkan seperti berikut.

![Data Preparation Workflow](images/data-preparation.png)

![Data Preparation Workflow 2](images/data-preparation-2.svg)

![Data Preparation Workflow 3](images/data-preparation-3.svg)

![Data Preparation Workflow 4](images/data-preparation-4.svg)


## Modeling

Proses modeling seluruhnya berada pada component `PCA, k-Means, Scatter Plot`. Pada proses ini, akan dilakukan training dengan metode clustering menggunakan algoritma k-means dan PCA di dalam konteks Spark, menggunakan node `Spark k-Means` dan `Spark PCA`. Akan digunakan 3 cluster (sesuai dengan jumlah kelas pada data) dan 300 kali iterasi.

Sebelumnya, data dinormalisasi dengan node `Spark Normalizer`.

![Spark Normalizer](images/spark-normalizer.png)

![Spark k-Means](images/spark-kmeans.png)

![Spark k-Means setting](images/spark-kmeans-setting.png)

![Spark PCA](images/spark-pca.png)

![Spark PCA setting](images/spark-pca-setting.png)

Hasil dari keduanya dijoin dan diconvert menjadi KNIME Table, dan didenormalisasi kembali.

![Join Table](images/join-table.png)

![Spark Denormalizer](images/spark-denormalizer.png)

Sampai sini, data akan dipecah menjadi 2, satu dipakai di Evaluation, satunya dilanjutkan ke Deployment. Sebelum deployment, data dalam format KNIME Table akan diconvert ke Spark lagi, dan nama kolom yang ada whitespace dari hasil PCA diubah, dan dimasukkan ke component output.

![Last Model](images/last-model.png)

Workflow Modeling keseluruhan sebagai berikut.

![Modeling Workflow](images/modeling-workflow.png)

![Modeling Workflow 2](images/modeling-workflow-2.svg)


## Evaluation

Setelah ditrain berikutnya akan dilakukan dua proses, yaitu melakukan view data dalam plot scatter dan dalam view tabel

Hasil denormalisasi pada component `PCA, k-Means, Scatter Plot` akan diconvert sebagian kolom ke string agar warna bisa lebih bersih, dan berfokus ke visual saja.

![Number to String](images/number-to-string.png)

Lalu dimasukkan ke node `Color Manager` untuk diberi warna masing-masing cluster, dan diplot dengan node `Scatter Plot`, view tabel dengan node `Table View`.

![Plot View](images/plot-view.png)

Workflow Evaluation keseluruhan dan hasilnya sebagai berikut.

![Evaluation Workflow](images/evaluation-workflow.png)

![Evaluation Workflow 2](images/evaluation-workflow-2.png)

![Evaluation Workflow 3](images/evaluation-workflow-3.png)


## Deployment

Pada deployment, dilakukan 2 proses, insert data ke Hive dan Parquet.

Jika dilihat, keseluruhan proses Deployment adalah berikut.

![Deployment Workflow](images/deployment-workflow.png)

