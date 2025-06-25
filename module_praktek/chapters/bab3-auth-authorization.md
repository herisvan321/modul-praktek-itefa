# BAB 3: Authentication dan Authorization

## Deskripsi Bab
Bab ini akan membahas implementasi sistem authentication dan authorization yang aman untuk aplikasi e-commerce. Anda akan mempelajari cara membuat sistem login/register, mengelola session, mengimplementasikan role-based access control, dan menerapkan best practices security.

## Tujuan Pembelajaran
Setelah menyelesaikan bab ini, peserta diharapkan mampu:
- 🔐 Mengimplementasi sistem login dan register yang aman
- 🛡️ Mengelola session dan cookies dengan best practices
- 👥 Membedakan akses berdasarkan role (user dan admin)
- 🔒 Menerapkan password hashing dan validation
- 🚫 Mencegah common security vulnerabilities
- 📝 Implementasi middleware untuk proteksi halaman

## Materi Pembelajaran

### 1. AuthController Implementation

**A. Structure AuthController (controllers/AuthController.php)**
```php
class AuthController {
    private $userModel;
    private $database;
    
    public function __construct() {
        $this->database = (new Database())->getConnection();
        $this->userModel = new User($this->database);
    }
    
    public function login($email, $password) {
        // Validasi input
        if (empty($email) || empty($password)) {
            return ['success' => false, 'message' => 'Email dan password harus diisi'];
        }
        
        // Validasi format email
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            return ['success' => false, 'message' => 'Format email tidak valid'];
        }
        
        try {
            // Cari user berdasarkan email
            $user = $this->userModel->findByEmail($email);
            
            if (!$user) {
                return ['success' => false, 'message' => 'Email atau password salah'];
            }
            
            // Verifikasi password
            if (!$this->userModel->verifyPassword($password, $user['password'])) {
                return ['success' => false, 'message' => 'Email atau password salah'];
            }
            
            // Set session
            $this->setUserSession($user);
            
            // Update last login
            $this->userModel->updateLastLogin($user['id']);
            
            return [
                'success' => true, 
                'message' => 'Login berhasil',
                'redirect' => $user['role'] === 'admin' ? '/views/admin/dashboard.php' : '/views/user/home.php'
            ];
            
        } catch (Exception $e) {
            error_log("Login error: " . $e->getMessage());
            return ['success' => false, 'message' => 'Terjadi kesalahan sistem'];
        }
    }
    
    public function register($userData) {
        // Validasi input
        $validation = $this->validateRegistrationData($userData);
        if (!$validation['valid']) {
            return ['success' => false, 'message' => $validation['message']];
        }
        
        try {
            // Cek apakah email sudah terdaftar
            $existingUser = $this->userModel->findByEmail($userData['email']);
            if ($existingUser) {
                return ['success' => false, 'message' => 'Email sudah terdaftar'];
            }
            
            // Buat user baru
            $result = $this->userModel->create($userData);
            
            if ($result) {
                return ['success' => true, 'message' => 'Registrasi berhasil, silakan login'];
            } else {
                return ['success' => false, 'message' => 'Gagal membuat akun'];
            }
            
        } catch (Exception $e) {
            error_log("Registration error: " . $e->getMessage());
            return ['success' => false, 'message' => 'Terjadi kesalahan sistem'];
        }
    }
    
    public function logout() {
        // Hapus semua session data
        session_start();
        session_unset();
        session_destroy();
        
        // Hapus session cookie
        if (isset($_COOKIE[session_name()])) {
            setcookie(session_name(), '', time() - 3600, '/');
        }
        
        return ['success' => true, 'message' => 'Logout berhasil'];
    }
    
    private function setUserSession($user) {
        session_start();
        $_SESSION['user_id'] = $user['id'];
        $_SESSION['user_name'] = $user['name'];
        $_SESSION['user_email'] = $user['email'];
        $_SESSION['user_role'] = $user['role'];
        $_SESSION['login_time'] = time();
        
        // Regenerate session ID untuk security
        session_regenerate_id(true);
    }
    
    private function validateRegistrationData($data) {
        // Validasi nama
        if (empty($data['name']) || strlen($data['name']) < 2) {
            return ['valid' => false, 'message' => 'Nama minimal 2 karakter'];
        }
        
        // Validasi email
        if (empty($data['email']) || !filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            return ['valid' => false, 'message' => 'Format email tidak valid'];
        }
        
        // Validasi password
        if (empty($data['password']) || strlen($data['password']) < 6) {
            return ['valid' => false, 'message' => 'Password minimal 6 karakter'];
        }
        
        // Validasi konfirmasi password
        if ($data['password'] !== $data['confirm_password']) {
            return ['valid' => false, 'message' => 'Konfirmasi password tidak cocok'];
        }
        
        return ['valid' => true];
    }
}
```

### 2. Session Management

**A. Session Configuration**
```php
// config/session.php
class SessionManager {
    public static function start() {
        if (session_status() === PHP_SESSION_NONE) {
            // Konfigurasi session yang aman
            ini_set('session.cookie_httponly', 1);
            ini_set('session.cookie_secure', 1); // Untuk HTTPS
            ini_set('session.use_strict_mode', 1);
            ini_set('session.cookie_samesite', 'Strict');
            
            session_start();
            
            // Regenerate session ID secara berkala
            if (!isset($_SESSION['created'])) {
                $_SESSION['created'] = time();
            } else if (time() - $_SESSION['created'] > 1800) { // 30 menit
                session_regenerate_id(true);
                $_SESSION['created'] = time();
            }
        }
    }
    
    public static function isLoggedIn() {
        self::start();
        return isset($_SESSION['user_id']) && !empty($_SESSION['user_id']);
    }
    
    public static function requireLogin() {
        if (!self::isLoggedIn()) {
            header('Location: /views/user/login.php');
            exit();
        }
    }
    
    public static function requireAdmin() {
        self::requireLogin();
        if ($_SESSION['user_role'] !== 'admin') {
            header('Location: /views/user/home.php');
            exit();
        }
    }
    
    public static function getUserId() {
        self::start();
        return $_SESSION['user_id'] ?? null;
    }
    
    public static function getUserRole() {
        self::start();
        return $_SESSION['user_role'] ?? null;
    }
}
```

**B. Middleware Implementation**
```php
// middleware/AuthMiddleware.php
class AuthMiddleware {
    public static function checkAuth($requiredRole = null) {
        SessionManager::start();
        
        // Cek apakah user sudah login
        if (!SessionManager::isLoggedIn()) {
            self::redirectToLogin();
        }
        
        // Cek role jika diperlukan
        if ($requiredRole && $_SESSION['user_role'] !== $requiredRole) {
            self::redirectUnauthorized();
        }
        
        // Cek session timeout
        if (isset($_SESSION['login_time']) && (time() - $_SESSION['login_time']) > 3600) {
            self::logout();
            self::redirectToLogin('Session expired');
        }
    }
    
    private static function redirectToLogin($message = null) {
        if ($message) {
            $_SESSION['error_message'] = $message;
        }
        header('Location: /views/user/login.php');
        exit();
    }
    
    private static function redirectUnauthorized() {
        $_SESSION['error_message'] = 'Anda tidak memiliki akses ke halaman ini';
        $redirectUrl = $_SESSION['user_role'] === 'admin' ? '/views/admin/dashboard.php' : '/views/user/home.php';
        header('Location: ' . $redirectUrl);
        exit();
    }
    
    private static function logout() {
        session_unset();
        session_destroy();
    }
}
```

### 3. Security Best Practices

**A. Password Security**
```php
class PasswordSecurity {
    public static function hash($password) {
        return password_hash($password, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536, // 64 MB
            'time_cost' => 4,       // 4 iterations
            'threads' => 3,         // 3 threads
        ]);
    }
    
    public static function verify($password, $hash) {
        return password_verify($password, $hash);
    }
    
    public static function needsRehash($hash) {
        return password_needs_rehash($hash, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost' => 4,
            'threads' => 3,
        ]);
    }
    
    public static function validateStrength($password) {
        $errors = [];
        
        if (strlen($password) < 8) {
            $errors[] = 'Password minimal 8 karakter';
        }
        
        if (!preg_match('/[A-Z]/', $password)) {
            $errors[] = 'Password harus mengandung huruf besar';
        }
        
        if (!preg_match('/[a-z]/', $password)) {
            $errors[] = 'Password harus mengandung huruf kecil';
        }
        
        if (!preg_match('/[0-9]/', $password)) {
            $errors[] = 'Password harus mengandung angka';
        }
        
        if (!preg_match('/[^A-Za-z0-9]/', $password)) {
            $errors[] = 'Password harus mengandung karakter khusus';
        }
        
        return [
            'valid' => empty($errors),
            'errors' => $errors
        ];
    }
}
```

**B. Input Validation dan Sanitization**
```php
class InputValidator {
    public static function sanitizeString($input) {
        return htmlspecialchars(trim($input), ENT_QUOTES, 'UTF-8');
    }
    
    public static function validateEmail($email) {
        $email = filter_var($email, FILTER_SANITIZE_EMAIL);
        return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
    }
    
    public static function validateCSRF($token) {
        if (!isset($_SESSION['csrf_token']) || !hash_equals($_SESSION['csrf_token'], $token)) {
            throw new Exception('CSRF token mismatch');
        }
    }
    
    public static function generateCSRF() {
        if (!isset($_SESSION['csrf_token'])) {
            $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
        }
        return $_SESSION['csrf_token'];
    }
}
```

### 4. Role-Based Access Control

**A. Permission System**
```php
class PermissionManager {
    private static $permissions = [
        'admin' => [
            'view_dashboard',
            'manage_products',
            'manage_orders',
            'view_reports',
            'manage_users'
        ],
        'user' => [
            'view_products',
            'manage_cart',
            'place_order',
            'view_orders'
        ]
    ];
    
    public static function hasPermission($permission) {
        $userRole = SessionManager::getUserRole();
        return in_array($permission, self::$permissions[$userRole] ?? []);
    }
    
    public static function requirePermission($permission) {
        if (!self::hasPermission($permission)) {
            throw new Exception('Insufficient permissions');
        }
    }
}
```

## Praktikum Hands-On

### Langkah 1: Analisis AuthController
1. Buka file `controllers/AuthController.php`
2. Analisis method `login()`, `register()`, dan `logout()`
3. Identifikasi security measures yang sudah diterapkan
4. Test fitur login dengan akun yang sudah ada

### Langkah 2: Implementasi Session Management
1. Buat file `config/SessionManager.php`
2. Implementasi session security configuration
3. Test session timeout dan regeneration
4. Verifikasi session data di browser developer tools

### Langkah 3: Test Authentication Flow
1. Test registrasi user baru
2. Test login dengan kredensial yang benar dan salah
3. Test akses halaman admin tanpa login
4. Test logout dan session cleanup

### Langkah 4: Implementasi Middleware
1. Buat file `middleware/AuthMiddleware.php`
2. Tambahkan middleware ke halaman admin
3. Test proteksi halaman berdasarkan role
4. Implementasi redirect yang sesuai

## Tugas dan Evaluasi

### Tugas Individu
1. **Remember Me Functionality** (25 poin)
   - Implementasi "Remember Me" checkbox pada login
   - Gunakan secure cookies dengan expiration
   - Test persistence login setelah browser ditutup

2. **Password Reset System** (30 poin)
   - Buat halaman "Forgot Password"
   - Implementasi token-based password reset
   - Kirim email reset password (simulasi)
   - Validasi token dan update password

3. **Enhanced Security** (25 poin)
   - Implementasi rate limiting untuk login attempts
   - Tambah CAPTCHA pada form register
   - Implementasi account lockout setelah failed attempts
   - Log security events

4. **Two-Factor Authentication** (20 poin)
   - Implementasi 2FA dengan TOTP (Time-based OTP)
   - Buat QR code untuk setup authenticator app
   - Validasi OTP code pada login
   - Backup codes untuk recovery

### Kriteria Penilaian
- **Security Implementation**: Penerapan best practices security
- **Code Quality**: Clean code dan error handling
- **User Experience**: Kemudahan penggunaan fitur auth
- **Testing**: Comprehensive testing semua scenario

## Resources Tambahan
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [PHP Session Security](https://www.php.net/manual/en/session.security.php)
- [Password Hashing in PHP](https://www.php.net/manual/en/book.password.php)

## Tips dan Best Practices
1. **Selalu hash password** menggunakan algoritma yang kuat
2. **Implementasi session timeout** untuk security
3. **Gunakan HTTPS** untuk transmisi data sensitif
4. **Validasi input** di sisi server, bukan hanya client
5. **Log security events** untuk monitoring dan audit
6. **Implementasi rate limiting** untuk mencegah brute force
7. **Gunakan CSRF protection** pada form yang sensitif

---

## Navigasi
- [← Bab 2: Database dan Model](./bab2-database-model.md)
- [Lanjut ke Bab 4: Frontend dan User Interface →](./bab4-frontend-ui.md)

---

*Pastikan Anda telah menyelesaikan semua praktikum dan tugas di bab ini sebelum melanjutkan ke bab berikutnya.*