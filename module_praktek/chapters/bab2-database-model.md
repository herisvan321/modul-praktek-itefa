# BAB 2: Database dan Model

## Deskripsi Bab
Bab ini akan membahas secara mendalam tentang desain database untuk sistem e-commerce dan implementasi layer Model dalam arsitektur MVC. Anda akan mempelajari cara merancang schema database yang efisien, membuat model classes yang robust, dan mengimplementasikan operasi CRUD yang aman.

## Tujuan Pembelajaran
Setelah menyelesaikan bab ini, peserta diharapkan mampu:
- 🗄️ Memahami dan menganalisis struktur database e-commerce yang kompleks
- 🏗️ Merancang dan mengimplementasikan model classes dengan best practices
- 🔄 Mengimplementasikan operasi CRUD (Create, Read, Update, Delete) yang aman
- 🔐 Menerapkan prepared statements untuk mencegah SQL injection
- 📊 Memahami relasi antar tabel dan foreign key constraints
- 🎯 Mengoptimalkan query database untuk performa yang baik

## Materi Pembelajaran

### 1. Struktur Database E-Commerce

**A. Tabel Users**
```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role ENUM('user', 'admin') DEFAULT 'user',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```
- **Fungsi**: Menyimpan data pengguna (customer dan admin)
- **Key Fields**: `id` (PK), `email` (unique), `role` (authorization)
- **Security**: Password di-hash menggunakan `password_hash()`

**B. Tabel Products**
```sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    stock INT DEFAULT 0,
    image VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```
- **Fungsi**: Katalog produk dengan informasi lengkap
- **Key Fields**: `id` (PK), `price` (decimal untuk akurasi), `stock` (inventory)
- **Features**: Auto-timestamp untuk tracking perubahan

**C. Tabel Orders**
```sql
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    shipping_address TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```
- **Fungsi**: Header informasi pesanan
- **Key Fields**: `user_id` (FK), `status` (workflow), `total_amount`
- **Relationships**: One-to-Many dengan `users`

**D. Tabel Order Items**
```sql
CREATE TABLE order_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);
```
- **Fungsi**: Detail item dalam setiap pesanan
- **Key Fields**: `order_id` (FK), `product_id` (FK), `quantity`, `price`
- **Relationships**: Many-to-One dengan `orders` dan `products`

### 2. Database Connection dengan PDO

**A. Class Database (config/database.php)**
```php
class Database {
    private $host = 'localhost';
    private $db_name = 'simple_ecommerce';
    private $username = 'root';
    private $password = '';
    private $charset = 'utf8mb4';
    public $pdo;
    
    public function getConnection() {
        $this->pdo = null;
        
        try {
            $dsn = "mysql:host=" . $this->host . ";dbname=" . $this->db_name . ";charset=" . $this->charset;
            
            $options = [
                PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
                PDO::ATTR_EMULATE_PREPARES => false,
                PDO::MYSQL_ATTR_INIT_COMMAND => "SET NAMES utf8mb4"
            ];
            
            $this->pdo = new PDO($dsn, $this->username, $this->password, $options);
            
        } catch(PDOException $exception) {
            error_log("Connection error: " . $exception->getMessage());
            throw new Exception("Database connection failed");
        }
        
        return $this->pdo;
    }
}
```

**B. Keunggulan PDO**
- **Prepared Statements**: Mencegah SQL injection
- **Multiple Database Support**: MySQL, PostgreSQL, SQLite
- **Object-Oriented Interface**: Lebih modern dan fleksibel
- **Error Handling**: Exception-based error handling

### 3. Implementasi Model Classes

**A. Base Model Class**
```php
abstract class BaseModel {
    protected $pdo;
    protected $table;
    
    public function __construct($database) {
        $this->pdo = $database;
    }
    
    protected function executeQuery($sql, $params = []) {
        try {
            $stmt = $this->pdo->prepare($sql);
            $stmt->execute($params);
            return $stmt;
        } catch(PDOException $e) {
            error_log("Query error: " . $e->getMessage());
            throw new Exception("Database query failed");
        }
    }
}
```

**B. User Model (models/User.php)**
```php
class User extends BaseModel {
    protected $table = 'users';
    
    public function create($data) {
        $sql = "INSERT INTO users (name, email, password, role) VALUES (?, ?, ?, ?)";
        $hashedPassword = password_hash($data['password'], PASSWORD_DEFAULT);
        
        return $this->executeQuery($sql, [
            $data['name'],
            $data['email'],
            $hashedPassword,
            $data['role'] ?? 'user'
        ]);
    }
    
    public function findByEmail($email) {
        $sql = "SELECT * FROM users WHERE email = ?";
        $stmt = $this->executeQuery($sql, [$email]);
        return $stmt->fetch();
    }
    
    public function verifyPassword($password, $hash) {
        return password_verify($password, $hash);
    }
    
    public function updateLastLogin($userId) {
        $sql = "UPDATE users SET last_login = NOW() WHERE id = ?";
        return $this->executeQuery($sql, [$userId]);
    }
}
```

**C. Product Model (models/Product.php)**
```php
class Product extends BaseModel {
    protected $table = 'products';
    
    public function getAll($limit = null, $offset = 0) {
        $sql = "SELECT * FROM products ORDER BY created_at DESC";
        if ($limit) {
            $sql .= " LIMIT ? OFFSET ?";
            $stmt = $this->executeQuery($sql, [$limit, $offset]);
        } else {
            $stmt = $this->executeQuery($sql);
        }
        return $stmt->fetchAll();
    }
    
    public function findById($id) {
        $sql = "SELECT * FROM products WHERE id = ?";
        $stmt = $this->executeQuery($sql, [$id]);
        return $stmt->fetch();
    }
    
    public function create($data) {
        $sql = "INSERT INTO products (name, description, price, stock, image) VALUES (?, ?, ?, ?, ?)";
        return $this->executeQuery($sql, [
            $data['name'],
            $data['description'],
            $data['price'],
            $data['stock'],
            $data['image']
        ]);
    }
    
    public function update($id, $data) {
        $sql = "UPDATE products SET name = ?, description = ?, price = ?, stock = ?, image = ? WHERE id = ?";
        return $this->executeQuery($sql, [
            $data['name'],
            $data['description'],
            $data['price'],
            $data['stock'],
            $data['image'],
            $id
        ]);
    }
    
    public function delete($id) {
        $sql = "DELETE FROM products WHERE id = ?";
        return $this->executeQuery($sql, [$id]);
    }
    
    public function searchProducts($keyword, $limit = 10) {
        $sql = "SELECT * FROM products WHERE name LIKE ? OR description LIKE ? ORDER BY name LIMIT ?";
        $searchTerm = "%" . $keyword . "%";
        $stmt = $this->executeQuery($sql, [$searchTerm, $searchTerm, $limit]);
        return $stmt->fetchAll();
    }
    
    public function updateStock($productId, $quantity) {
        $sql = "UPDATE products SET stock = stock - ? WHERE id = ? AND stock >= ?";
        $stmt = $this->executeQuery($sql, [$quantity, $productId, $quantity]);
        return $stmt->rowCount() > 0;
    }
    
    public function getLowStockProducts($threshold = 10) {
        $sql = "SELECT * FROM products WHERE stock <= ? ORDER BY stock ASC";
        $stmt = $this->executeQuery($sql, [$threshold]);
        return $stmt->fetchAll();
    }
}
```

### 4. Advanced Database Operations

**A. Transaction Management**
```php
class OrderModel extends BaseModel {
    public function createOrderWithItems($orderData, $items) {
        try {
            $this->pdo->beginTransaction();
            
            // Create order
            $sql = "INSERT INTO orders (user_id, total_amount, shipping_address) VALUES (?, ?, ?)";
            $stmt = $this->executeQuery($sql, [
                $orderData['user_id'],
                $orderData['total_amount'],
                $orderData['shipping_address']
            ]);
            
            $orderId = $this->pdo->lastInsertId();
            
            // Add order items and update stock
            foreach ($items as $item) {
                // Insert order item
                $sql = "INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (?, ?, ?, ?)";
                $this->executeQuery($sql, [$orderId, $item['product_id'], $item['quantity'], $item['price']]);
                
                // Update product stock
                $sql = "UPDATE products SET stock = stock - ? WHERE id = ?";
                $this->executeQuery($sql, [$item['quantity'], $item['product_id']]);
            }
            
            $this->pdo->commit();
            return $orderId;
            
        } catch (Exception $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
}
```

**B. Query Optimization**
```php
// Join queries untuk performa yang lebih baik
public function getOrdersWithDetails($userId = null) {
    $sql = "SELECT o.*, u.name as user_name, u.email,
                   COUNT(oi.id) as item_count,
                   SUM(oi.quantity) as total_items
            FROM orders o
            JOIN users u ON o.user_id = u.id
            LEFT JOIN order_items oi ON o.id = oi.order_id";
    
    $params = [];
    if ($userId) {
        $sql .= " WHERE o.user_id = ?";
        $params[] = $userId;
    }
    
    $sql .= " GROUP BY o.id ORDER BY o.created_at DESC";
    
    $stmt = $this->executeQuery($sql, $params);
    return $stmt->fetchAll();
}
```

## Praktikum Hands-On

### Langkah 1: Analisis Database Schema
1. Buka phpMyAdmin dan eksplorasi struktur tabel
2. Identifikasi primary keys, foreign keys, dan indexes
3. Analisis relasi antar tabel menggunakan diagram
4. Test sample queries untuk memahami data flow

### Langkah 2: Implementasi Base Model
1. Buat file `models/BaseModel.php`
2. Implementasi method-method dasar untuk CRUD
3. Tambahkan error handling dan logging
4. Test koneksi database

### Langkah 3: Extend Product Model
1. Modifikasi `models/Product.php`
2. Tambahkan method `searchProducts()`
3. Implementasi `getLowStockProducts()`
4. Test semua method dengan data sample

### Langkah 4: Create Order Model
1. Buat file `models/Order.php`
2. Implementasi transaction untuk create order
3. Tambahkan method untuk update status
4. Test complete order flow

## Tugas dan Evaluasi

### Tugas Individu
1. **ERD Design** (30 poin)
   - Buat Entity Relationship Diagram lengkap
   - Dokumentasikan semua relasi dan constraints
   - Propose additional tables (reviews, categories, etc.)

2. **Model Implementation** (40 poin)
   - Implementasi `Order.php` model lengkap
   - Tambahkan method untuk reporting queries
   - Implementasi soft delete untuk products

3. **Query Optimization** (20 poin)
   - Analisis slow queries menggunakan EXPLAIN
   - Tambahkan indexes yang diperlukan
   - Optimize existing queries

4. **Security Enhancement** (10 poin)
   - Implementasi input validation
   - Add SQL injection prevention tests
   - Implement rate limiting untuk queries

### Kriteria Penilaian
- **Database Design**: Normalisasi dan efisiensi schema
- **Code Quality**: Clean code dan best practices
- **Security**: Prevention terhadap common vulnerabilities
- **Performance**: Query optimization dan indexing

## Resources Tambahan
- [MySQL Performance Tuning](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [PHP PDO Documentation](https://www.php.net/manual/en/book.pdo.php)
- [Database Design Best Practices](https://www.sqlshack.com/database-design-best-practices/)

## Tips dan Best Practices
1. **Selalu gunakan prepared statements** untuk mencegah SQL injection
2. **Implementasi proper indexing** untuk query yang sering digunakan
3. **Gunakan transactions** untuk operasi yang melibatkan multiple tables
4. **Validate input** sebelum melakukan database operations
5. **Log database errors** untuk debugging dan monitoring

---

## Navigasi
- [← Bab 1: Pengenalan dan Setup](./bab1-pengenalan-setup.md)
- [Lanjut ke Bab 3: Authentication dan Authorization →](./bab3-auth-authorization.md)

---

*Pastikan Anda telah menyelesaikan semua praktikum dan tugas di bab ini sebelum melanjutkan ke bab berikutnya.*