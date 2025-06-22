# BAB 1: Pengenalan dan Setup Environment

## Deskripsi Bab
Bab ini merupakan fondasi pembelajaran yang akan memperkenalkan Anda pada ekosistem pengembangan web dengan PHP dan MySQL. Anda akan mempelajari cara mengatur environment development yang profesional dan memahami arsitektur aplikasi e-commerce yang akan dikembangkan.

## Tujuan Pembelajaran
Setelah menyelesaikan bab ini, peserta diharapkan mampu:
- 🎯 Memahami struktur dan arsitektur project e-commerce secara menyeluruh
- 🛠️ Mengsetup environment development yang optimal untuk PHP dan MySQL
- 🏗️ Memahami konsep MVC (Model-View-Controller) dan implementasinya
- 🔧 Mengkonfigurasi web server dan database untuk development
- 📁 Menganalisis struktur folder dan file dalam project

## Materi Pembelajaran

### 1. Struktur Project E-Commerce
```
simple_ecommerce/
├── config/              # Konfigurasi aplikasi
│   └── database.php     # Koneksi database dengan PDO
├── controllers/         # Business logic dan flow control
│   ├── AuthController.php    # Autentikasi user
│   └── ProductController.php # Manajemen produk
├── models/              # Data layer dan database operations
│   ├── Product.php      # Model untuk produk
│   └── User.php         # Model untuk user
├── views/               # Presentation layer (UI)
│   ├── admin/           # Interface untuk admin
│   │   ├── dashboard.php
│   │   ├── reports.php
│   │   └── orders.php
│   ├── user/            # Interface untuk customer
│   │   ├── home.php
│   │   ├── cart.php
│   │   └── checkout.php
│   └── layouts/         # Template dan layout
│       ├── header.php
│       └── footer.php
├── assets/              # Static resources
│   ├── css/             # Stylesheet files
│   ├── js/              # JavaScript files
│   └── images/          # Image assets
├── database.sql         # Database schema dan sample data
├── index.php           # Entry point aplikasi
└── .htaccess           # Apache configuration
```

### 2. Setup Environment Development

**A. Instalasi Web Server Stack**
- **XAMPP (Windows/Linux)** atau **MAMP (macOS)**
  - Apache Web Server 2.4+
  - PHP 7.4+ dengan ekstensi PDO, mysqli
  - MySQL 8.0+ atau MariaDB 10.4+
  - phpMyAdmin untuk database management

**B. Konfigurasi PHP**
```ini
; php.ini settings yang direkomendasikan
display_errors = On
error_reporting = E_ALL
max_execution_time = 300
memory_limit = 256M
upload_max_filesize = 10M
post_max_size = 10M
```

**C. Setup Database**
- Buat database baru: `simple_ecommerce`
- Import schema dari `database.sql`
- Konfigurasi user dan privileges
- Test koneksi database

**D. Virtual Host Configuration (Opsional)**
```apache
<VirtualHost *:80>
    DocumentRoot "/path/to/simple_ecommerce"
    ServerName ecommerce.local
    <Directory "/path/to/simple_ecommerce">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

### 3. Konsep MVC Architecture

**Model (Data Layer)**
- Bertanggung jawab untuk interaksi dengan database
- Mengimplementasikan business rules dan validasi data
- Contoh: `User.php`, `Product.php`
```php
class Product {
    private $pdo;
    
    public function __construct($database) {
        $this->pdo = $database;
    }
    
    public function getAllProducts() {
        // Database query logic
    }
}
```

**View (Presentation Layer)**
- Menampilkan data kepada user dalam format HTML
- Menggunakan template dan layout untuk konsistensi
- Responsive design dengan Bootstrap

**Controller (Business Logic)**
- Menghubungkan Model dan View
- Memproses input dari user
- Mengatur flow aplikasi
```php
class AuthController {
    public function login($email, $password) {
        // Validasi input
        // Cek ke database via Model
        // Redirect ke View yang sesuai
    }
}
```

### 4. File Konfigurasi Penting

**database.php**
```php
class Database {
    private $host = 'localhost';
    private $db_name = 'simple_ecommerce';
    private $username = 'root';
    private $password = '';
    
    public function getConnection() {
        try {
            $pdo = new PDO("mysql:host=" . $this->host . ";dbname=" . $this->db_name, 
                          $this->username, $this->password);
            $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
            return $pdo;
        } catch(PDOException $e) {
            echo "Connection error: " . $e->getMessage();
        }
    }
}
```

**.htaccess**
```apache
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ index.php [QSA,L]

# Security headers
Header always set X-Content-Type-Options nosniff
Header always set X-Frame-Options DENY
Header always set X-XSS-Protection "1; mode=block"
```

## Praktikum Hands-On

### Langkah 1: Instalasi Environment
1. Download dan install XAMPP/MAMP
2. Start Apache dan MySQL services
3. Akses phpMyAdmin di `http://localhost/phpmyadmin`
4. Test PHP dengan membuat file `info.php`:
```php
<?php phpinfo(); ?>
```

### Langkah 2: Setup Database
1. Buat database `simple_ecommerce`
2. Import file `database.sql`
3. Verifikasi tabel yang terbuat:
   - `users` (data pengguna)
   - `products` (data produk)
   - `orders` (data pesanan)
   - `order_items` (detail pesanan)

### Langkah 3: Test Aplikasi
1. Copy project ke folder `htdocs` (XAMPP) atau `htdocs` (MAMP)
2. Akses `http://localhost/simple_ecommerce`
3. Test fitur login dengan akun admin:
   - Email: `admin@example.com`
   - Password: `admin123`

### Langkah 4: Eksplorasi Kode
1. Buka project di code editor (VS Code, Sublime, dll)
2. Analisis struktur folder
3. Trace flow dari `index.php` ke controller dan view
4. Identifikasi pattern MVC yang digunakan

## Tugas dan Evaluasi

### Tugas Individu
1. **Dokumentasi Setup** (25 poin)
   - Buat step-by-step guide instalasi
   - Screenshot setiap tahap instalasi
   - Troubleshooting common issues

2. **Analisis Arsitektur** (35 poin)
   - Diagram struktur folder project
   - Penjelasan fungsi setiap folder/file
   - Identifikasi komponen MVC

3. **Environment Customization** (25 poin)
   - Setup virtual host dengan domain custom
   - Konfigurasi PHP settings untuk development
   - Setup Git repository untuk version control

4. **Testing dan Debugging** (15 poin)
   - Test semua fitur dasar aplikasi
   - Identifikasi dan dokumentasikan bug (jika ada)
   - Propose improvement untuk setup

### Kriteria Penilaian
- **Kelengkapan**: Semua langkah setup berhasil dilakukan
- **Dokumentasi**: Kualitas dan detail dokumentasi
- **Pemahaman**: Kemampuan menjelaskan konsep MVC
- **Problem Solving**: Kemampuan troubleshooting masalah

## Resources Tambahan
- [PHP Official Documentation](https://www.php.net/manual/en/)
- [MySQL Documentation](https://dev.mysql.com/doc/)
- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/)
- [Bootstrap Documentation](https://getbootstrap.com/docs/)

## Tips dan Best Practices
1. **Selalu backup database** sebelum melakukan perubahan
2. **Gunakan version control** (Git) dari awal development
3. **Enable error reporting** selama development
4. **Gunakan consistent coding style** (PSR standards)
5. **Test di multiple browsers** untuk compatibility

---

## Navigasi
- [← Kembali ke Index](../README.md)
- [Lanjut ke Bab 2: Database dan Model →](./bab2-database-model.md)

---

*Pastikan Anda telah menyelesaikan semua praktikum dan tugas di bab ini sebelum melanjutkan ke bab berikutnya.*