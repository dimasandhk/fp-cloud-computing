# Final Project Cloud Computing

**Kelompok B4**
|Nama|NRP |
|--|--|
|Dimas Andhika Diputra|5027231074|
|Rafael Ega Krisaditya|5027231025|
|Rafi Afnaan Fathurrahman|5027231040|
|Kevin Anugerah Faza|5027221027|
|Aryasatya Alaauddin|5027221082|

Demo Youtube: https://youtu.be/SdVI43f2Rxo

# I) Introduksi Permasalahan

Anda adalah seorang lulusan Teknologi Informasi, sebagai ahli IT, salah satu kemampuan yang harus dimiliki adalah **Keampuan merancang, membangun, mengelola aplikasi berbasis komputer menggunakan layanan awan untuk memenuhi kebutuhan organisasi.**

Pada suatu saat anda mendapatkan project untuk mendeploy sebuah aplikasi Sentiment Analysis dengan komponen Backend menggunakan python: sentiment-analysis.py dengan spesifikasi sebagai berikut

### Endpoints:

1. **Analyze Text**

   - **Endpoint:** `POST /analyze`
   - **Description:** This endpoint accepts a text input and returns the sentiment score of the text.
   - **Request:**
     ```json
     {
       "text": "Your text here"
     }
     ```
   - **Response:**
     ```json
     {
       "sentiment": <sentiment_score>
     }
     ```

2. **Retrieve History**
   - **Endpoint:** `GET /history`
   - **Description:** This endpoint retrieves the history of previously analyzed texts along with their sentiment scores.
   - **Response:**
     ```json
     {
      {
        "text": "Your previous text here",
        "sentiment": <sentiment_score>
      },
      ...
     }
     ```

---

Kemudian juga disediakan sebuah Frontend sederhana menggunakan [index.html](/Resources/FE/index.html) dan [styles.css](/Resources/FE/styles.css) dengan tampilan antarmuka sebagai berikut

![](./img/preview-fe.png)

Kemudian anda diminta untuk mendesain arsitektur cloud yang sesuai dengan kebutuhan aplikasi tersebut. Apabila dana maksimal yang diberikan adalah **1 juta rupiah per bulan (65 US$)**
konfigurasi cloud terbaik seperti apa yang bisa dibuat?

# II) Rancangan Arsitektur dan Tabel Harga Spesifikasi VM

Di revisi ini, kami mengubah rancangan arsitektur menjadi menggunakan 3 droplet VM dengan cloud provider **digital ocean**. Kami menggunakan 3 droplet VM dengan spesifikasi sebagai berikut:

<img src="./img/vm-amd.png" />

Total biaya yang kami keluarkan sekitar **3 VM x $21 = $63**. Berikut adalah preview droplet yang kami buat:

![alt text](./img/image.png)

# III) Implementasi dan Konfigurasi Teknologi

Untuk konfigurasi dari arsitektur yang telah kami buat terbagi menjadi 2, yaitu konfigurasi untuk worker (2 vm), dan konfigurasi untuk load balancer dari worker tersebut (1 vm)

### Konfigurasi Worker

VM untuk setiap worker kami install beberapa dependencies:

- Nginx untuk frontend
- Python3-pip, Python3-venv (virtual environment) untuk setup aplikasi flasknya (backend)
  - Flask: Framework backend yang kami pakai.
  - Flask-CORS: Digunakan untuk mengatasi masalah Cross-Origin Resource Sharing (CORS) di aplikasi Flask.
  - TextBlob: Library untuk pemrosesan teks dan analisis sentimen di aplikasi Python.
  - PyMongo: Driver resmi MongoDB untuk Python yang digunakan untuk menghubungkan dan berinteraksi dengan database MongoDB.
  - Gunicorn: HTTP server untuk aplikasi Python WSGI, sering digunakan untuk menjalankan aplikasi Flask di prod.
  - Gevent: Library untuk concurrency di Python, memungkinkan aplikasi untuk menangani banyak koneksi secara efisien.
- MongoDB database untuk menyimpan hasil sentimen beserta teksnya

#### **Setup Frontend:**

```
sudo apt-get update
sudo apt-get install nginx
```

setelah menginstall nginx seharusnya kita mendapatkan file dan direktori baru oleh nginx seperti direktori `/var/www/html` dan `/etc/nginx/sites-available/`

setelah itu kita bisa langsung clone repository `fp-tka` untuk mendapatkan resources file frontendnya dan memindahkannya ke folder `/var/www/html` agar diserve oleh nginx

(posisi di root)

```
git clone https://github.com/fuaddary/fp-tka.git
mv fp-tka/Resources/FE/index.html /var/www/html/index.html
mv fp-tka/Resources/FE/styles.css /var/www/html/styles.css
sudo systemctl restart nginx
```

setelah file frontend diserve seharusnya kita bisa melihat tampilan frontend di ip tersebut

<img src="./img-config/fe.png" />

setelah itu kita masih perlu setup beberapa konfigurasi baik dari `file htmlnya` maupun konfigurasi file `/etc/nginx/sites-available/default`, karena kami ingin agar dalam **worker tersebut frontend dan backend bekerja dalam port yang sama, agar pada saat load balance kita bisa mengakses frontend dan backend di port yang sama ke 2 worker yang telah kami buat**. Pertama kami ubah url fetch di dalam htmlnya menjadi hanya `/history` dan `/analyze`

<img src="./img-config/setup-fe1.png" />

setelah itu perlu juga untuk mengubah konfigurasi file `default` di `/etc/nginx/sites-available/default` karena **yang diserve di port default 80 adalah nginx, kita tidak bisa langsung mengakses endpoint backend dengan menjalankan backend tersebut di port 80, kita harus mengubah konfigurasi nginx terlebih dahulu**

<img src="./img-config/setup-fe2.png" />

lalu kita restart ulang service nginxnya

```
sudo systemctl restart nginx
```

selesai, setup untuk frontend sudah selesai dan siap untuk digunakan

<hr>

#### **Setup Database MongoDB:**

Pertama kita perlu menginstall mongodb di worker vmnya, caranya adalah sebagai berikut

> Referensi: https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/

1. Import public key yang digunakan oleh package management systemnya

```
sudo apt-get install gnupg curl
```

lalu import MongoDB public GPG keynya

```
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor
```

2. Buat list file untuk MongoDB

buat list file di `/etc/apt/sources.list.d/mongodb-org-7.0.list`

```
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

3. Install MongoDB package

```
sudo apt-get update
sudo apt-get install -y mongodb-org
```

Terakhir kita bisa langsung aktifkan mongodbnya dan konfigurasi firewall untuk mengizinkan koneksi ke port default MongoDB (27017)

```
sudo systemctl enable mongod
sudo ufw allow 27017
```

setup database selesai, database siap digunakan

<hr>

#### **Setup Backend:**

Untuk backend kita pertama perlu menginstall pip dan virtual environment

```
sudo apt-get update
sudo apt-get install python3-pip python3-venv
```

karena sudah clone repo github `fp-tka` sebelumnya jadi kita tinggal pakai file yang sudah di clone sebelumnya, kita perlu masuk ke direktori `fp-tka/Resources/BE`, membuat virtual environment, dan install semua dependencies python yang sudah disebutkan sebelumnya

```
cd fp-tka/Resources/BE
python3 -m venv venv
source venv/bin/activate
```

setelah itu seharusnya tampilan dari prompt terminal sedikit berubah, dan package python telah terisolasi

<img src="./img-config/setup-be1.png" />

jika tampilan prompt terminal sudah berubah seperti gambar di atas kita bisa install package python yang diperlukan

```
pip install flask flask-cors textblob pymongo gunicorn gevent
```

setelah itu kita bisa harus ubah nama file pythonnya terlebih dahulu dari `sentiment-analysis.py` menjadi `sentiment_analysis.py` dan langsung menjalankan backend aplikasi flask dengan daemon tanpa konfigurasi lain2

```
gunicorn -b 0.0.0.0:80 -w 5 -k gevent --timeout 60 --graceful-timeout 60 sentiment_analysis:app --daemon
```

- gunicorn: Menjalankan Gunicorn, WSGI HTTP server untuk aplikasi Python
- -b 0.0.0.0:80: Bind server ke alamat IP 0.0.0.0 pada port 80. Server akan dapat diakses dari semua alamat IP yang dapat menjangkau server.
- -w 5: Menentukan jumlah worker yang akan dijalankan oleh Gunicorn. Dalam hal ini, 5 worker.
- -k gevent: Menggunakan worker tipe gevent, yang mendukung concurrency dengan menggunakan greenlet untuk menangani banyak koneksi secara efisien.
- --timeout 60: Menentukan waktu maksimum dalam detik yang diizinkan untuk setiap request sebelum worker dibunuh dan direstart.
- --graceful-timeout 60: Menentukan waktu tambahan dalam detik yang diberikan kepada worker untuk menyelesaikan request yang sedang diproses sebelum dipaksa untuk berhenti.
- sentiment_analysis:app: Menunjukkan modul dan aplikasi Flask yang akan dijalankan oleh Gunicorn. sentiment_analysis adalah nama modul, dan app adalah instance aplikasi Flask.
- --daemon: Menjalankan Gunicorn dalam mode daemon, sehingga proses akan berjalan di latar belakang.

Jika sudah berhasil seharusnya kita juga bisa akses endpoint backend di vm tersebut dengan port 80

<img src="./img-config/setup-be2.png" /><br>

### Konfigurasi Load Balancer

Untuk konfigurasi vm sebagai load-balancer kita juga perlu menginstall nginx di vm tersebut

```
sudo apt-get update
sudo apt-get install nginx
```

lalu kita perlu menghapus tautan simbolis (symlink) dari file konfigurasi default Nginx yang biasanya terletak di `/etc/nginx/sites-enabled/default` dan membuat konfigurasi baru di `/etc/nginx/sites-available/app`

```
sudo unlink /etc/nginx/sites-enabled/default
sudo nano  /etc/nginx/sites-available/app
```

berikut adalah konfigurasi yang dipakai untuk loadbalancernya

```
upstream backend_servers {
	server 152.42.251.221;
	server 128.199.103.250;
}

server {
    listen 80;
    server_name 157.230.246.133;

    location / {
        proxy_cache my_cache;
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;

        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;
        proxy_cache_lock_timeout 5s;

        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header X-Cached $upstream_cache_status;
    }
}
```

konfigurasi tersebut adalah load balancer dengan algoritma round robin yang menampung 2 vm worker. Setelah itu kita perlu membuat tautan simbolis (symlink) dari file konfigurasi Nginx di `/etc/nginx/sites-available/` ke direktori `/etc/nginx/sites-enabled/`

terakhir kita perlu ubah file konfigurasi `/etc/nginx/nginx.conf` untuk membuat tambahan pengaturan cache, yang tidak ada dalam konfigurasi default Nginx. Pengaturan cache ini mengkonfigurasi jalur cache, ukuran maksimal cache, dan handling penggunaan cache ketika terjadi kesalahan server.

konfigurasinya seperti berikut:

```
user www-data;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;
worker_rlimit_nofile 100000;

events {
    worker_connections 4096;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '"$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Konfigurasi cache
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m;
    proxy_temp_path /var/cache/nginx/temp;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_lock on;
    proxy_cache_lock_timeout 5s;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

setelah itu kita bisa restart dengan `sudo systemctl restart nginx` dan akses ip vm tersebut untuk mengakses load balancer

<img src="./img-config/setup-lb1.png" /><br>

# IV) Hasil Pengujian Setiap Endpoint

### Pengujian dengan Rest Client (Salah satu vm)

1. Get All History

<img src="./img/rest-client-history.png" />

2. Create a New Text

<img src="./img/rest-client-analyze.png" />

### Pengujian dari Frontend (Salah satu vm)

<img src="./img//testweb1.png" />

<img src="./img/testweb2.png" /><br>

# V) Hasil Pengujian dan Analisis Loadtesting Locust

- RPS Maksimum (load testing 60 detik)

> **RPS Maksimum yang kami dapati dari beberapa stress test locust adalah ~900-1000 RPS**

<img src="./img/locust3.jpg" />

<hr>

- Peak Concurrency Maksimum **1000** (**spawn rate 50**, load testing 60 detik)

<img src="./img/locust1.jpg" />

<hr>

- Peak Concurrency Maksimum **2000** (**spawn rate 100**, load testing 60 detik)

<img src="./img/locust2.jpg" />

<hr>

- Peak Concurrency Maksimum **3000** (**spawn rate 200**, load testing 60 detik)

<img src="./img/locust3.jpg" />

<hr>

- Peak Concurrency Maksimum **4000** (**spawn rate 500**, load testing 60 detik)

<img src="./img/locust4.jpg" /><br>

# VI) Revisi, Kesimpulan, dan Saran

### REVISI

Setelah melakukan beberapa konfigurasi ulang, berikut ini adalah poin-poin pentingnya:

- Hasil pengujian locust menunjukkan bahwa rancangan layanan awan dapat menampung beban hingga 1015 Requests per Second (RPS) tanpa adanya failures satupun. Namun, waktu respons mulai meningkat secara signifikan setelah mencapai sekitar +- 600 RPS yang menunjukkan adanya limit atau bottleneck yang terjadi dimana konfigurasi kami mulai kewalahan untuk melakukan handle requests.
- Sepanjang pengujian tidak ada satupun failures yang diberikan oleh locust yang mengindikasikan bahwa rancangan layanan awan stabil dan tidak mengalami error meskipun beban requests yang diberikan sangat tinggi.
- Waktu respons rata-rata (Average Response Time)) cukup stabil berkisar antara 200ms pada beban yang tidak tinggi hingga menengah, tetapi relatif meningkat saat mengalami beban tinggi.
- Maximum Capacity yang didapatkan berdasarkan uji locust kali ini secara efektif berkisar antara 600-700 RPS dengan tidak ada kendala apapun pada layanan awan. Namun mulai mengalami pengurangan performa sehingga menyebabkan layanan bisa berjalan tidak secara maksimal saat diatasnya

### KESIMPULAN

Hasil pengujian Locust menunjukkan bahwa rancangan layanan awan dapat menampung beban hingga 1015 Requests per Second (RPS) tanpa adanya kegagalan, mengindikasikan bahwa arsitektur cloud yang digunakan stabil dan tidak mengalami error meskipun di bawah beban yang sangat tinggi. Namun, waktu respons mulai meningkat secara signifikan setelah mencapai sekitar 600 RPS, menunjukkan adanya limit atau bottleneck di mana konfigurasi mulai kewalahan menangani permintaan. Waktu respons rata-rata stabil di sekitar 200 ms pada beban rendah hingga menengah, tetapi meningkat pada beban tinggi. Kapasitas maksimum yang didapatkan dapat mencapai 1015 RPS dengan efektivitas berkisar antara 600-700 RPS, setelah itu performa menurun sehingga layanan tidak berjalan secara optimal di atas beban tersebut.

### SARAN

#### Jaringan dan Koneksi:

Pastikan **koneksi internet stabil dan memiliki bandwidth yang mencukupi untuk menghindari fluktuasi yang dapat memengaruhi hasil pengujian**. Pengaturan jaringan VM juga penting untuk memastikan bahwa koneksi ke aplikasi atau sistem yang diuji cukup lancar.

#### Load Balancer Alternatif:

Menggunakan **VM sebagai pengganti load balancer** merupakan pilihan yang bisa dipertimbangkan, terutama jika kita ingin mengontrol secara langsung distribusi lalu lintas atau memiliki kebutuhan khusus terkait konfigurasi jaringan.

#### Skalabilitas:

Pertimbangkan kemampuan untuk **menskalakan VM** sesuai dengan kebutuhan pengujian. Hal ini memungkinkan kita untuk menyesuaikan kapasitas infrastruktur secara dinamis jika diperlukan.
Dengan mempertimbangkan faktor-faktor ini dalam konfigurasi VM untuk pengujian dengan Locust, kita dapat meningkatkan kualitas pengujian dan meminimalkan dampak dari variabel-variabel eksternal yang dapat mempengaruhi hasil pengujian.

#### Optimasi Performa Backend:

Untuk mengatasi masalah lag pada VM backend saat menggunakan spawn rate tinggi, pertimbangkan untuk melakukan optimisasi pada aplikasi backend atau infrastruktur database. **Bisa dengan cara hapus data di database, atau hapus cache nginx pada load balancer setiap kali ingin test locust**

> Kami sempat mendapatkan bahwa data di database kami sempat ada 24k (kami mendapatkan ini sebelum menyadari bahwa data yang diinput oleh locust dalam pengujian sangat banyak)
