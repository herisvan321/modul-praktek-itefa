# BAB 4: Frontend dan User Interface

## Deskripsi Bab
Bab ini akan membahas implementasi frontend dan user interface untuk aplikasi e-commerce. Anda akan mempelajari cara membangun tampilan yang responsif, user-friendly, dan modern menggunakan HTML5, CSS3, Bootstrap, dan JavaScript. Fokus utama adalah pada pengalaman pengguna (UX) dan antarmuka pengguna (UI) yang optimal.

## Tujuan Pembelajaran
Setelah menyelesaikan bab ini, peserta diharapkan mampu:
- 🎨 Memahami struktur view dan layout system dalam MVC
- 📱 Mengimplementasikan responsive design dengan Bootstrap
- 🖥️ Membuat user interface yang konsisten dan user-friendly
- ⚡ Mengintegrasikan JavaScript untuk interaktivitas
- 🔄 Implementasi AJAX untuk dynamic content loading
- 🎯 Mengoptimalkan user experience (UX) dan user interface (UI)

## Materi Pembelajaran

### 1. Layout System

**A. Header Layout (views/layouts/header.php)**
```php
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?php echo $pageTitle ?? 'Simple E-Commerce'; ?></title>
    
    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <!-- Font Awesome -->
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <!-- Custom CSS -->
    <link href="/assets/css/style.css" rel="stylesheet">
    
    <!-- Meta tags untuk SEO -->
    <meta name="description" content="<?php echo $pageDescription ?? 'Simple E-Commerce - Belanja Online Mudah dan Aman'; ?>">
    <meta name="keywords" content="<?php echo $pageKeywords ?? 'ecommerce, belanja online, produk'; ?>">
    
    <!-- Open Graph untuk social media -->
    <meta property="og:title" content="<?php echo $pageTitle ?? 'Simple E-Commerce'; ?>">
    <meta property="og:description" content="<?php echo $pageDescription ?? 'Belanja Online Mudah dan Aman'; ?>">
    <meta property="og:type" content="website">
</head>
<body>
    <!-- Navigation Bar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary sticky-top">
        <div class="container">
            <a class="navbar-brand fw-bold" href="/">
                <i class="fas fa-store me-2"></i>Simple E-Commerce
            </a>
            
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="/views/user/home.php">
                            <i class="fas fa-home me-1"></i>Home
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="/views/user/products.php">
                            <i class="fas fa-box me-1"></i>Produk
                        </a>
                    </li>
                </ul>
                
                <ul class="navbar-nav">
                    <?php if (SessionManager::isLoggedIn()): ?>
                        <!-- User Menu -->
                        <li class="nav-item">
                            <a class="nav-link position-relative" href="/views/user/cart.php">
                                <i class="fas fa-shopping-cart me-1"></i>Keranjang
                                <span class="badge bg-danger position-absolute top-0 start-100 translate-middle" id="cart-count">
                                    <?php echo count($_SESSION['cart'] ?? []); ?>
                                </span>
                            </a>
                        </li>
                        
                        <?php if (SessionManager::getUserRole() === 'admin'): ?>
                            <!-- Admin Menu -->
                            <li class="nav-item dropdown">
                                <a class="nav-link dropdown-toggle" href="#" role="button" data-bs-toggle="dropdown">
                                    <i class="fas fa-cog me-1"></i>Admin
                                </a>
                                <ul class="dropdown-menu">
                                    <li><a class="dropdown-item" href="/views/admin/dashboard.php">
                                        <i class="fas fa-tachometer-alt me-2"></i>Dashboard
                                    </a></li>
                                    <li><a class="dropdown-item" href="/views/admin/orders.php">
                                        <i class="fas fa-shopping-bag me-2"></i>Pesanan
                                    </a></li>
                                    <li><a class="dropdown-item" href="/views/admin/reports.php">
                                        <i class="fas fa-chart-bar me-2"></i>Laporan
                                    </a></li>
                                    <li><hr class="dropdown-divider"></li>
                                    <li><a class="dropdown-item" href="/views/admin/create_product.php">
                                        <i class="fas fa-plus me-2"></i>Tambah Produk
                                    </a></li>
                                </ul>
                            </li>
                        <?php endif; ?>
                        
                        <!-- User Profile -->
                        <li class="nav-item dropdown">
                            <a class="nav-link dropdown-toggle" href="#" role="button" data-bs-toggle="dropdown">
                                <i class="fas fa-user me-1"></i><?php echo $_SESSION['user_name']; ?>
                            </a>
                            <ul class="dropdown-menu">
                                <li><a class="dropdown-item" href="/views/user/profile.php">
                                    <i class="fas fa-user-edit me-2"></i>Profil
                                </a></li>
                                <li><a class="dropdown-item" href="/views/user/orders.php">
                                    <i class="fas fa-history me-2"></i>Riwayat Pesanan
                                </a></li>
                                <li><hr class="dropdown-divider"></li>
                                <li><a class="dropdown-item" href="/controllers/AuthController.php?action=logout">
                                    <i class="fas fa-sign-out-alt me-2"></i>Logout
                                </a></li>
                            </ul>
                        </li>
                    <?php else: ?>
                        <!-- Guest Menu -->
                        <li class="nav-item">
                            <a class="nav-link" href="/views/user/login.php">
                                <i class="fas fa-sign-in-alt me-1"></i>Login
                            </a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="/views/user/register.php">
                                <i class="fas fa-user-plus me-1"></i>Register
                            </a>
                        </li>
                    <?php endif; ?>
                </ul>
            </div>
        </div>
    </nav>
    
    <!-- Alert Messages -->
    <?php if (isset($_SESSION['success_message'])): ?>
        <div class="alert alert-success alert-dismissible fade show m-0" role="alert">
            <i class="fas fa-check-circle me-2"></i><?php echo $_SESSION['success_message']; ?>
            <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
        </div>
        <?php unset($_SESSION['success_message']); ?>
    <?php endif; ?>
    
    <?php if (isset($_SESSION['error_message'])): ?>
        <div class="alert alert-danger alert-dismissible fade show m-0" role="alert">
            <i class="fas fa-exclamation-circle me-2"></i><?php echo $_SESSION['error_message']; ?>
            <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
        </div>
        <?php unset($_SESSION['error_message']); ?>
    <?php endif; ?>
    
    <!-- Main Content -->
    <main class="flex-grow-1">
```

**B. Footer Layout (views/layouts/footer.php)**
```php
    </main>
    
    <!-- Footer -->
    <footer class="bg-dark text-light py-5 mt-5">
        <div class="container">
            <div class="row">
                <div class="col-md-4 mb-4">
                    <h5 class="fw-bold mb-3">
                        <i class="fas fa-store me-2"></i>Simple E-Commerce
                    </h5>
                    <p class="text-muted">
                        Platform belanja online terpercaya dengan berbagai pilihan produk berkualitas.
                    </p>
                    <div class="social-links">
                        <a href="#" class="text-light me-3"><i class="fab fa-facebook-f"></i></a>
                        <a href="#" class="text-light me-3"><i class="fab fa-twitter"></i></a>
                        <a href="#" class="text-light me-3"><i class="fab fa-instagram"></i></a>
                        <a href="#" class="text-light"><i class="fab fa-whatsapp"></i></a>
                    </div>
                </div>
                
                <div class="col-md-2 mb-4">
                    <h6 class="fw-bold mb-3">Navigasi</h6>
                    <ul class="list-unstyled">
                        <li><a href="/" class="text-muted text-decoration-none">Home</a></li>
                        <li><a href="/views/user/products.php" class="text-muted text-decoration-none">Produk</a></li>
                        <li><a href="/about.php" class="text-muted text-decoration-none">Tentang Kami</a></li>
                        <li><a href="/contact.php" class="text-muted text-decoration-none">Kontak</a></li>
                    </ul>
                </div>
                
                <div class="col-md-3 mb-4">
                    <h6 class="fw-bold mb-3">Layanan</h6>
                    <ul class="list-unstyled">
                        <li><a href="#" class="text-muted text-decoration-none">Bantuan</a></li>
                        <li><a href="#" class="text-muted text-decoration-none">Kebijakan Privasi</a></li>
                        <li><a href="#" class="text-muted text-decoration-none">Syarat & Ketentuan</a></li>
                        <li><a href="#" class="text-muted text-decoration-none">FAQ</a></li>
                    </ul>
                </div>
                
                <div class="col-md-3 mb-4">
                    <h6 class="fw-bold mb-3">Kontak Info</h6>
                    <ul class="list-unstyled text-muted">
                        <li><i class="fas fa-map-marker-alt me-2"></i>Jakarta, Indonesia</li>
                        <li><i class="fas fa-phone me-2"></i>+62 123 456 789</li>
                        <li><i class="fas fa-envelope me-2"></i>info@simpleecommerce.com</li>
                    </ul>
                </div>
            </div>
            
            <hr class="my-4">
            
            <div class="row align-items-center">
                <div class="col-md-6">
                    <p class="text-muted mb-0">
                        &copy; <?php echo date('Y'); ?> Simple E-Commerce. All rights reserved.
                    </p>
                </div>
                <div class="col-md-6 text-md-end">
                    <p class="text-muted mb-0">
                        Made with <i class="fas fa-heart text-danger"></i> for learning purposes
                    </p>
                </div>
            </div>
        </div>
    </footer>
    
    <!-- Bootstrap JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    <!-- Chart.js untuk reporting -->
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <!-- Custom JS -->
    <script src="/assets/js/script.js"></script>
    
    <!-- Page specific scripts -->
    <?php if (isset($pageScripts)): ?>
        <?php foreach ($pageScripts as $script): ?>
            <script src="<?php echo $script; ?>"></script>
        <?php endforeach; ?>
    <?php endif; ?>
</body>
</html>
```

### 2. User Views

**A. Home Page (views/user/home.php)**
```php
<?php
require_once '../../config/database.php';
require_once '../../models/Product.php';
require_once '../../config/SessionManager.php';

SessionManager::start();

$pageTitle = 'Home - Simple E-Commerce';
$pageDescription = 'Belanja online mudah dan aman dengan berbagai pilihan produk berkualitas';

// Ambil produk terbaru
$database = (new Database())->getConnection();
$productModel = new Product($database);
$latestProducts = $productModel->getAll(8); // 8 produk terbaru
$featuredProducts = $productModel->getFeaturedProducts(4); // 4 produk unggulan

include '../layouts/header.php';
?>

<!-- Hero Section -->
<section class="hero-section bg-primary text-white py-5">
    <div class="container">
        <div class="row align-items-center">
            <div class="col-lg-6">
                <h1 class="display-4 fw-bold mb-4">Belanja Online Mudah & Aman</h1>
                <p class="lead mb-4">
                    Temukan berbagai produk berkualitas dengan harga terbaik. 
                    Pengalaman belanja online yang nyaman dan terpercaya.
                </p>
                <div class="d-flex gap-3">
                    <a href="/views/user/products.php" class="btn btn-light btn-lg">
                        <i class="fas fa-shopping-bag me-2"></i>Mulai Belanja
                    </a>
                    <a href="#featured" class="btn btn-outline-light btn-lg">
                        <i class="fas fa-star me-2"></i>Produk Unggulan
                    </a>
                </div>
            </div>
            <div class="col-lg-6 text-center">
                <img src="/assets/images/hero-shopping.svg" alt="Shopping" class="img-fluid" style="max-height: 400px;">
            </div>
        </div>
    </div>
</section>

<!-- Features Section -->
<section class="py-5">
    <div class="container">
        <div class="row text-center">
            <div class="col-md-3 mb-4">
                <div class="feature-box p-4">
                    <i class="fas fa-shipping-fast fa-3x text-primary mb-3"></i>
                    <h5>Pengiriman Cepat</h5>
                    <p class="text-muted">Pengiriman ke seluruh Indonesia dengan jaminan aman dan cepat</p>
                </div>
            </div>
            <div class="col-md-3 mb-4">
                <div class="feature-box p-4">
                    <i class="fas fa-shield-alt fa-3x text-success mb-3"></i>
                    <h5>Pembayaran Aman</h5>
                    <p class="text-muted">Sistem pembayaran yang aman dengan berbagai metode pilihan</p>
                </div>
            </div>
            <div class="col-md-3 mb-4">
                <div class="feature-box p-4">
                    <i class="fas fa-headset fa-3x text-info mb-3"></i>
                    <h5>Customer Service</h5>
                    <p class="text-muted">Layanan pelanggan 24/7 siap membantu Anda</p>
                </div>
            </div>
            <div class="col-md-3 mb-4">
                <div class="feature-box p-4">
                    <i class="fas fa-undo fa-3x text-warning mb-3"></i>
                    <h5>Garansi Return</h5>
                    <p class="text-muted">Garansi pengembalian barang jika tidak sesuai</p>
                </div>
            </div>
        </div>
    </div>
</section>

<!-- Featured Products -->
<section id="featured" class="py-5 bg-light">
    <div class="container">
        <div class="text-center mb-5">
            <h2 class="fw-bold">Produk Unggulan</h2>
            <p class="text-muted">Pilihan terbaik dari koleksi kami</p>
        </div>
        
        <div class="row">
            <?php foreach ($featuredProducts as $product): ?>
                <div class="col-lg-3 col-md-6 mb-4">
                    <div class="card product-card h-100 shadow-sm">
                        <div class="position-relative">
                            <img src="<?php echo $product['image'] ?: '/assets/images/default.jpg'; ?>" 
                                 class="card-img-top" alt="<?php echo htmlspecialchars($product['name']); ?>" 
                                 style="height: 200px; object-fit: cover;">
                            <div class="position-absolute top-0 end-0 m-2">
                                <span class="badge bg-danger">Unggulan</span>
                            </div>
                        </div>
                        <div class="card-body d-flex flex-column">
                            <h6 class="card-title"><?php echo htmlspecialchars($product['name']); ?></h6>
                            <p class="card-text text-muted small flex-grow-1">
                                <?php echo substr(htmlspecialchars($product['description']), 0, 80) . '...'; ?>
                            </p>
                            <div class="d-flex justify-content-between align-items-center">
                                <span class="h5 text-primary mb-0">
                                    Rp <?php echo number_format($product['price'], 0, ',', '.'); ?>
                                </span>
                                <small class="text-muted">
                                    Stock: <?php echo $product['stock']; ?>
                                </small>
                            </div>
                        </div>
                        <div class="card-footer bg-transparent">
                            <button class="btn btn-primary w-100 add-to-cart" 
                                    data-product-id="<?php echo $product['id']; ?>">
                                <i class="fas fa-cart-plus me-2"></i>Tambah ke Keranjang
                            </button>
                        </div>
                    </div>
                </div>
            <?php endforeach; ?>
        </div>
        
        <div class="text-center mt-4">
            <a href="/views/user/products.php" class="btn btn-outline-primary btn-lg">
                Lihat Semua Produk <i class="fas fa-arrow-right ms-2"></i>
            </a>
        </div>
    </div>
</section>

<!-- Latest Products -->
<section class="py-5">
    <div class="container">
        <div class="text-center mb-5">
            <h2 class="fw-bold">Produk Terbaru</h2>
            <p class="text-muted">Koleksi terbaru yang baru saja ditambahkan</p>
        </div>
        
        <div class="row">
            <?php foreach ($latestProducts as $product): ?>
                <div class="col-lg-3 col-md-6 mb-4">
                    <div class="card product-card h-100 shadow-sm">
                        <img src="<?php echo $product['image'] ?: '/assets/images/default.jpg'; ?>" 
                             class="card-img-top" alt="<?php echo htmlspecialchars($product['name']); ?>" 
                             style="height: 200px; object-fit: cover;">
                        <div class="card-body d-flex flex-column">
                            <h6 class="card-title"><?php echo htmlspecialchars($product['name']); ?></h6>
                            <p class="card-text text-muted small flex-grow-1">
                                <?php echo substr(htmlspecialchars($product['description']), 0, 80) . '...'; ?>
                            </p>
                            <div class="d-flex justify-content-between align-items-center">
                                <span class="h5 text-primary mb-0">
                                    Rp <?php echo number_format($product['price'], 0, ',', '.'); ?>
                                </span>
                                <small class="text-muted">
                                    Stock: <?php echo $product['stock']; ?>
                                </small>
                            </div>
                        </div>
                        <div class="card-footer bg-transparent">
                            <button class="btn btn-primary w-100 add-to-cart" 
                                    data-product-id="<?php echo $product['id']; ?>">
                                <i class="fas fa-cart-plus me-2"></i>Tambah ke Keranjang
                            </button>
                        </div>
                    </div>
                </div>
            <?php endforeach; ?>
        </div>
    </div>
</section>

<!-- Newsletter Section -->
<section class="py-5 bg-primary text-white">
    <div class="container">
        <div class="row align-items-center">
            <div class="col-lg-6">
                <h3 class="fw-bold mb-3">Dapatkan Update Terbaru</h3>
                <p class="mb-0">Berlangganan newsletter kami untuk mendapatkan info produk terbaru dan penawaran menarik</p>
            </div>
            <div class="col-lg-6">
                <form class="d-flex gap-2">
                    <input type="email" class="form-control" placeholder="Masukkan email Anda" required>
                    <button type="submit" class="btn btn-light">
                        <i class="fas fa-paper-plane"></i>
                    </button>
                </form>
            </div>
        </div>
    </div>
</section>

<?php include '../layouts/footer.php'; ?>
```

### 3. Responsive Design dengan Bootstrap

**A. Custom CSS (assets/css/style.css)**
```css
/* Custom Variables */
:root {
    --primary-color: #0d6efd;
    --secondary-color: #6c757d;
    --success-color: #198754;
    --danger-color: #dc3545;
    --warning-color: #ffc107;
    --info-color: #0dcaf0;
    --light-color: #f8f9fa;
    --dark-color: #212529;
    --border-radius: 0.5rem;
    --box-shadow: 0 0.125rem 0.25rem rgba(0, 0, 0, 0.075);
    --transition: all 0.3s ease;
}

/* Global Styles */
body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    line-height: 1.6;
}

/* Navigation */
.navbar-brand {
    font-size: 1.5rem;
    font-weight: 700;
}

.navbar-nav .nav-link {
    font-weight: 500;
    transition: var(--transition);
}

.navbar-nav .nav-link:hover {
    color: rgba(255, 255, 255, 0.8) !important;
}

/* Hero Section */
.hero-section {
    background: linear-gradient(135deg, var(--primary-color) 0%, #0056b3 100%);
    min-height: 500px;
    display: flex;
    align-items: center;
}

/* Product Cards */
.product-card {
    transition: var(--transition);
    border: none;
    border-radius: var(--border-radius);
}

.product-card:hover {
    transform: translateY(-5px);
    box-shadow: 0 0.5rem 1rem rgba(0, 0, 0, 0.15);
}

.product-card .card-img-top {
    transition: var(--transition);
    border-radius: var(--border-radius) var(--border-radius) 0 0;
}

.product-card:hover .card-img-top {
    transform: scale(1.05);
}

/* Feature Boxes */
.feature-box {
    transition: var(--transition);
    border-radius: var(--border-radius);
}

.feature-box:hover {
    transform: translateY(-3px);
    box-shadow: var(--box-shadow);
}

/* Buttons */
.btn {
    border-radius: var(--border-radius);
    font-weight: 500;
    transition: var(--transition);
}

.btn:hover {
    transform: translateY(-1px);
}

/* Forms */
.form-control {
    border-radius: var(--border-radius);
    border: 1px solid #dee2e6;
    transition: var(--transition);
}

.form-control:focus {
    border-color: var(--primary-color);
    box-shadow: 0 0 0 0.2rem rgba(13, 110, 253, 0.25);
}

/* Cart */
.cart-item {
    border-bottom: 1px solid #dee2e6;
    padding: 1rem 0;
}

.cart-item:last-child {
    border-bottom: none;
}

.quantity-input {
    width: 80px;
    text-align: center;
}

/* Admin Dashboard */
.dashboard-card {
    border-left: 4px solid var(--primary-color);
    transition: var(--transition);
}

.dashboard-card:hover {
    transform: translateY(-2px);
    box-shadow: var(--box-shadow);
}

.dashboard-card.success {
    border-left-color: var(--success-color);
}

.dashboard-card.warning {
    border-left-color: var(--warning-color);
}

.dashboard-card.danger {
    border-left-color: var(--danger-color);
}

.dashboard-card.info {
    border-left-color: var(--info-color);
}

/* Tables */
.table {
    border-radius: var(--border-radius);
    overflow: hidden;
}

.table thead th {
    background-color: var(--light-color);
    border-bottom: 2px solid #dee2e6;
    font-weight: 600;
}

/* Alerts */
.alert {
    border-radius: var(--border-radius);
    border: none;
}

/* Loading Spinner */
.loading-spinner {
    display: none;
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 9999;
}

/* Responsive Utilities */
@media (max-width: 768px) {
    .hero-section {
        text-align: center;
        padding: 3rem 0;
    }
    
    .hero-section .display-4 {
        font-size: 2rem;
    }
    
    .feature-box {
        margin-bottom: 2rem;
    }
    
    .product-card {
        margin-bottom: 1.5rem;
    }
}

@media (max-width: 576px) {
    .container {
        padding-left: 1rem;
        padding-right: 1rem;
    }
    
    .btn-lg {
        padding: 0.75rem 1.5rem;
        font-size: 1rem;
    }
}

/* Dark Mode Support */
@media (prefers-color-scheme: dark) {
    .card {
        background-color: #2d3748;
        border-color: #4a5568;
    }
    
    .card-body {
        color: #e2e8f0;
    }
    
    .text-muted {
        color: #a0aec0 !important;
    }
}

/* Print Styles */
@media print {
    .navbar,
    .footer,
    .btn {
        display: none !important;
    }
    
    .container {
        max-width: none !important;
    }
}
```

### 4. JavaScript Integration

**A. Main JavaScript (assets/js/script.js)**
```javascript
// Global App Object
const App = {
    // Configuration
    config: {
        baseUrl: window.location.origin,
        apiUrl: '/api',
        csrfToken: document.querySelector('meta[name="csrf-token"]')?.getAttribute('content')
    },
    
    // Initialize app
    init() {
        this.bindEvents();
        this.initComponents();
        this.loadCartCount();
    },
    
    // Bind global events
    bindEvents() {
        // Add to cart buttons
        document.addEventListener('click', (e) => {
            if (e.target.classList.contains('add-to-cart') || e.target.closest('.add-to-cart')) {
                e.preventDefault();
                const button = e.target.classList.contains('add-to-cart') ? e.target : e.target.closest('.add-to-cart');
                const productId = button.dataset.productId;
                this.addToCart(productId);
            }
        });
        
        // Remove from cart
        document.addEventListener('click', (e) => {
            if (e.target.classList.contains('remove-from-cart') || e.target.closest('.remove-from-cart')) {
                e.preventDefault();
                const button = e.target.classList.contains('remove-from-cart') ? e.target : e.target.closest('.remove-from-cart');
                const productId = button.dataset.productId;
                this.removeFromCart(productId);
            }
        });
        
        // Update quantity
        document.addEventListener('change', (e) => {
            if (e.target.classList.contains('quantity-input')) {
                const productId = e.target.dataset.productId;
                const quantity = parseInt(e.target.value);
                this.updateCartQuantity(productId, quantity);
            }
        });
        
        // Form submissions
        document.addEventListener('submit', (e) => {
            if (e.target.classList.contains('ajax-form')) {
                e.preventDefault();
                this.submitForm(e.target);
            }
        });
    },
    
    // Initialize components
    initComponents() {
        // Initialize tooltips
        const tooltipTriggerList = [].slice.call(document.querySelectorAll('[data-bs-toggle="tooltip"]'));
        tooltipTriggerList.map(function (tooltipTriggerEl) {
            return new bootstrap.Tooltip(tooltipTriggerEl);
        });
        
        // Initialize popovers
        const popoverTriggerList = [].slice.call(document.querySelectorAll('[data-bs-toggle="popover"]'));
        popoverTriggerList.map(function (popoverTriggerEl) {
            return new bootstrap.Popover(popoverTriggerEl);
        });
        
        // Auto-hide alerts
        setTimeout(() => {
            const alerts = document.querySelectorAll('.alert');
            alerts.forEach(alert => {
                if (alert.classList.contains('alert-success')) {
                    const bsAlert = new bootstrap.Alert(alert);
                    bsAlert.close();
                }
            });
        }, 5000);
    },
    
    // Cart functions
    addToCart(productId, quantity = 1) {
        this.showLoading();
        
        fetch('/controllers/CartController.php', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-TOKEN': this.config.csrfToken
            },
            body: JSON.stringify({
                action: 'add',
                product_id: productId,
                quantity: quantity
            })
        })
        .then(response => response.json())
        .then(data => {
            this.hideLoading();
            if (data.success) {
                this.showAlert('Produk berhasil ditambahkan ke keranjang', 'success');
                this.updateCartCount(data.cart_count);
            } else {
                this.showAlert(data.message || 'Gagal menambahkan produk', 'danger');
            }
        })
        .catch(error => {
            this.hideLoading();
            console.error('Error:', error);
            this.showAlert('Terjadi kesalahan sistem', 'danger');
        });
    },
    
    removeFromCart(productId) {
        if (!confirm('Apakah Anda yakin ingin menghapus produk ini dari keranjang?')) {
            return;
        }
        
        this.showLoading();
        
        fetch('/controllers/CartController.php', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-TOKEN': this.config.csrfToken
            },
            body: JSON.stringify({
                action: 'remove',
                product_id: productId
            })
        })
        .then(response => response.json())
        .then(data => {
            this.hideLoading();
            if (data.success) {
                this.showAlert('Produk berhasil dihapus dari keranjang', 'success');
                this.updateCartCount(data.cart_count);
                // Reload page to update cart display
                if (window.location.pathname.includes('cart.php')) {
                    window.location.reload();
                }
            } else {
                this.showAlert(data.message || 'Gagal menghapus produk', 'danger');
            }
        })
        .catch(error => {
            this.hideLoading();
            console.error('Error:', error);
            this.showAlert('Terjadi kesalahan sistem', 'danger');
        });
    },
    
    updateCartQuantity(productId, quantity) {
        if (quantity < 1) {
            this.removeFromCart(productId);
            return;
        }
        
        fetch('/controllers/CartController.php', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-TOKEN': this.config.csrfToken
            },
            body: JSON.stringify({
                action: 'update',
                product_id: productId,
                quantity: quantity
            })
        })
        .then(response => response.json())
        .then(data => {
            if (data.success) {
                this.updateCartCount(data.cart_count);
                // Update total if on cart page
                if (window.location.pathname.includes('cart.php')) {
                    this.updateCartTotal();
                }
            } else {
                this.showAlert(data.message || 'Gagal mengupdate quantity', 'danger');
            }
        })
        .catch(error => {
            console.error('Error:', error);
            this.showAlert('Terjadi kesalahan sistem', 'danger');
        });
    },
    
    loadCartCount() {
        fetch('/controllers/CartController.php?action=count')
        .then(response => response.json())
        .then(data => {
            if (data.success) {
                this.updateCartCount(data.count);
            }
        })
        .catch(error => {
            console.error('Error loading cart count:', error);
        });
    },
    
    updateCartCount(count) {
        const cartCountElement = document.getElementById('cart-count');
        if (cartCountElement) {
            cartCountElement.textContent = count;
            cartCountElement.style.display = count > 0 ? 'inline' : 'none';
        }
    },
    
    updateCartTotal() {
        // Implementation for updating cart total on cart page
        fetch('/controllers/CartController.php?action=total')
        .then(response => response.json())
        .then(data => {
            if (data.success) {
                const totalElement = document.getElementById('cart-total');
                if (totalElement) {
                    totalElement.textContent = 'Rp ' + this.formatNumber(data.total);
                }
            }
        })
        .catch(error => {
            console.error('Error updating cart total:', error);
        });
    },
    
    // Utility functions
    showLoading() {
        const spinner = document.querySelector('.loading-spinner');
        if (spinner) {
            spinner.style.display = 'block';
        }
    },
    
    hideLoading() {
        const spinner = document.querySelector('.loading-spinner');
        if (spinner) {
            spinner.style.display = 'none';
        }
    },
    
    showAlert(message, type = 'info') {
        const alertContainer = document.createElement('div');
        alertContainer.className = `alert alert-${type} alert-dismissible fade show position-fixed`;
        alertContainer.style.cssText = 'top: 20px; right: 20px; z-index: 9999; min-width: 300px;';
        alertContainer.innerHTML = `
            ${message}
            <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
        `;
        
        document.body.appendChild(alertContainer);
        
        // Auto remove after 5 seconds
        setTimeout(() => {
            if (alertContainer.parentNode) {
                alertContainer.parentNode.removeChild(alertContainer);
            }
        }, 5000);
    },
    
    submitForm(form) {
        const formData = new FormData(form);
        const url = form.action || window.location.href;
        
        this.showLoading();
        
        fetch(url, {
            method: 'POST',
            body: formData
        })
        .then(response => response.json())
        .then(data => {
            this.hideLoading();
            if (data.success) {
                this.showAlert(data.message || 'Operasi berhasil', 'success');
                if (data.redirect) {
                    setTimeout(() => {
                        window.location.href = data.redirect;
                    }, 1000);
                }
            } else {
                this.showAlert(data.message || 'Operasi gagal', 'danger');
            }
        })
        .catch(error => {
            this.hideLoading();
            console.error('Error:', error);
            this.showAlert('Terjadi kesalahan sistem', 'danger');
        });
    },
    
    formatNumber(number) {
        return new Intl.NumberFormat('id-ID').format(number);
    }
};

// Initialize app when DOM is loaded
document.addEventListener('DOMContentLoaded', () => {
    App.init();
});

// Loading spinner HTML (add to footer)
const loadingSpinner = `
    <div class="loading-spinner">
        <div class="spinner-border text-primary" role="status">
            <span class="visually-hidden">Loading...</span>
        </div>
    </div>
`;

// Add loading spinner to body
document.addEventListener('DOMContentLoaded', () => {
    document.body.insertAdjacentHTML('beforeend', loadingSpinner);
});
```

## Praktikum Hands-On

### Langkah 1: Modifikasi Home Page
1. Buka file `views/user/home.php`
2. Tambahkan hero section yang menarik
3. Implementasi grid system Bootstrap untuk produk
4. Test responsive design di berbagai ukuran layar

### Langkah 2: Implementasi Search Feature
1. Tambahkan search bar di navigation
2. Buat halaman `search.php` untuk hasil pencarian
3. Implementasi AJAX search dengan autocomplete
4. Test search functionality

### Langkah 3: Product Detail Page
1. Buat halaman `product_detail.php`
2. Implementasi image gallery dengan carousel
3. Tambahkan product reviews section
4. Implementasi related products

### Langkah 4: Mobile Optimization
1. Test semua halaman di mobile device
2. Optimasi touch interactions
3. Implementasi mobile-specific features
4. Test performance di mobile

## Tugas dan Evaluasi

### Tugas Individu
1. **Redesign Cart Page** (30 poin)
   - Implementasi AJAX untuk update quantity
   - Tambahkan product recommendations
   - Implementasi coupon/discount system
   - Real-time total calculation

2. **Image Gallery Implementation** (25 poin)
   - Multiple product images
   - Zoom functionality
   - Lightbox modal
   - Thumbnail navigation

3. **Dark Mode Toggle** (20 poin)
   - Implementasi dark/light mode switch
   - Save preference di localStorage
   - Smooth transition animations
   - Consistent styling across pages

4. **Progressive Web App** (25 poin)
   - Service worker implementation
   - Offline functionality
   - App manifest
   - Push notifications

### Kriteria Penilaian
- **Design Quality**: Estetika dan konsistensi visual
- **Responsiveness**: Compatibility di berbagai device
- **User Experience**: Kemudahan penggunaan
- **Performance**: Loading speed dan optimization

## Resources Tambahan
- [Bootstrap Documentation](https://getbootstrap.com/docs/)
- [CSS Grid Guide](https://css-tricks.com/snippets/css/complete-guide-grid/)
- [JavaScript ES6+ Features](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
- [Web Accessibility Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)

## Tips dan Best Practices
1. **Mobile-first approach** dalam responsive design
2. **Optimize images** untuk web performance
3. **Use semantic HTML** untuk accessibility
4. **Implement lazy loading** untuk images
5. **Minimize HTTP requests** dengan bundling
6. **Use CSS Grid dan Flexbox** untuk layout
7. **Test cross-browser compatibility**

---

## Navigasi
- [← Bab 3: Authentication dan Authorization](./bab3-auth-authorization.md)
- [Lanjut ke Bab 5: Fitur E-Commerce dan Business Logic →](./bab5-ecommerce-logic.md)

---

*Pastikan Anda telah menyelesaikan semua praktikum dan tugas di bab ini sebelum melanjutkan ke bab berikutnya.*