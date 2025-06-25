# BAB 5: Fitur E-Commerce dan Business Logic

## Deskripsi Bab
Bab ini akan membahas implementasi fitur-fitur inti e-commerce dan business logic yang kompleks. Anda akan mempelajari cara membangun sistem keranjang belanja, proses checkout, manajemen pesanan, sistem pembayaran, dan fitur-fitur bisnis lainnya yang essential untuk aplikasi e-commerce yang fungsional.

## Tujuan Pembelajaran
Setelah menyelesaikan bab ini, peserta diharapkan mampu:
- 🛒 Mengimplementasikan sistem keranjang belanja yang kompleks
- 💳 Membangun proses checkout dan payment gateway integration
- 📦 Membuat sistem manajemen pesanan dan status tracking
- 📊 Implementasi inventory management dan stock control
- 🎯 Membangun sistem rating dan review produk
- 🔄 Mengimplementasikan business rules dan validasi
- 📈 Membuat sistem reporting dan analytics

## Materi Pembelajaran

### 1. Shopping Cart System

**A. Cart Model (models/Cart.php)**
```php
<?php
class Cart {
    private $conn;
    private $table_name = "cart";
    
    public $id;
    public $user_id;
    public $product_id;
    public $quantity;
    public $created_at;
    public $updated_at;
    
    public function __construct($db) {
        $this->conn = $db;
    }
    
    // Tambah item ke cart
    public function addItem($user_id, $product_id, $quantity = 1) {
        // Cek apakah item sudah ada di cart
        $existing = $this->getCartItem($user_id, $product_id);
        
        if ($existing) {
            // Update quantity jika item sudah ada
            return $this->updateQuantity($user_id, $product_id, $existing['quantity'] + $quantity);
        } else {
            // Tambah item baru
            $query = "INSERT INTO " . $this->table_name . " 
                     SET user_id = :user_id, 
                         product_id = :product_id, 
                         quantity = :quantity,
                         created_at = NOW()";
            
            $stmt = $this->conn->prepare($query);
            
            $stmt->bindParam(':user_id', $user_id);
            $stmt->bindParam(':product_id', $product_id);
            $stmt->bindParam(':quantity', $quantity);
            
            return $stmt->execute();
        }
    }
    
    // Update quantity item
    public function updateQuantity($user_id, $product_id, $quantity) {
        if ($quantity <= 0) {
            return $this->removeItem($user_id, $product_id);
        }
        
        $query = "UPDATE " . $this->table_name . " 
                 SET quantity = :quantity, updated_at = NOW() 
                 WHERE user_id = :user_id AND product_id = :product_id";
        
        $stmt = $this->conn->prepare($query);
        
        $stmt->bindParam(':quantity', $quantity);
        $stmt->bindParam(':user_id', $user_id);
        $stmt->bindParam(':product_id', $product_id);
        
        return $stmt->execute();
    }
    
    // Hapus item dari cart
    public function removeItem($user_id, $product_id) {
        $query = "DELETE FROM " . $this->table_name . " 
                 WHERE user_id = :user_id AND product_id = :product_id";
        
        $stmt = $this->conn->prepare($query);
        
        $stmt->bindParam(':user_id', $user_id);
        $stmt->bindParam(':product_id', $product_id);
        
        return $stmt->execute();
    }
    
    // Ambil semua item di cart user
    public function getCartItems($user_id) {
        $query = "SELECT c.*, p.name, p.price, p.image, p.stock,
                         (c.quantity * p.price) as subtotal
                 FROM " . $this->table_name . " c
                 JOIN products p ON c.product_id = p.id
                 WHERE c.user_id = :user_id
                 ORDER BY c.created_at DESC";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':user_id', $user_id);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    // Ambil satu item cart
    public function getCartItem($user_id, $product_id) {
        $query = "SELECT * FROM " . $this->table_name . " 
                 WHERE user_id = :user_id AND product_id = :product_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':user_id', $user_id);
        $stmt->bindParam(':product_id', $product_id);
        $stmt->execute();
        
        return $stmt->fetch(PDO::FETCH_ASSOC);
    }
    
    // Hitung total cart
    public function getCartTotal($user_id) {
        $query = "SELECT SUM(c.quantity * p.price) as total
                 FROM " . $this->table_name . " c
                 JOIN products p ON c.product_id = p.id
                 WHERE c.user_id = :user_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':user_id', $user_id);
        $stmt->execute();
        
        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        return $result['total'] ?? 0;
    }
    
    // Hitung jumlah item di cart
    public function getCartCount($user_id) {
        $query = "SELECT SUM(quantity) as count FROM " . $this->table_name . " 
                 WHERE user_id = :user_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':user_id', $user_id);
        $stmt->execute();
        
        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        return $result['count'] ?? 0;
    }
    
    // Kosongkan cart
    public function clearCart($user_id) {
        $query = "DELETE FROM " . $this->table_name . " WHERE user_id = :user_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':user_id', $user_id);
        
        return $stmt->execute();
    }
    
    // Validasi stock sebelum checkout
    public function validateStock($user_id) {
        $query = "SELECT c.product_id, c.quantity, p.stock, p.name
                 FROM " . $this->table_name . " c
                 JOIN products p ON c.product_id = p.id
                 WHERE c.user_id = :user_id AND c.quantity > p.stock";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':user_id', $user_id);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
}
?>
```

**B. Cart Controller (controllers/CartController.php)**
```php
<?php
require_once '../config/database.php';
require_once '../models/Cart.php';
require_once '../models/Product.php';
require_once '../config/SessionManager.php';

class CartController {
    private $db;
    private $cart;
    private $product;
    
    public function __construct() {
        $database = new Database();
        $this->db = $database->getConnection();
        $this->cart = new Cart($this->db);
        $this->product = new Product($this->db);
        
        SessionManager::start();
    }
    
    public function handleRequest() {
        $action = $_GET['action'] ?? $_POST['action'] ?? 'view';
        
        switch ($action) {
            case 'add':
                $this->addToCart();
                break;
            case 'update':
                $this->updateCart();
                break;
            case 'remove':
                $this->removeFromCart();
                break;
            case 'clear':
                $this->clearCart();
                break;
            case 'count':
                $this->getCartCount();
                break;
            case 'total':
                $this->getCartTotal();
                break;
            default:
                $this->viewCart();
        }
    }
    
    private function addToCart() {
        if (!SessionManager::isLoggedIn()) {
            $this->jsonResponse(false, 'Silakan login terlebih dahulu');
            return;
        }
        
        $input = json_decode(file_get_contents('php://input'), true);
        $product_id = $input['product_id'] ?? null;
        $quantity = $input['quantity'] ?? 1;
        $user_id = SessionManager::getUserId();
        
        if (!$product_id) {
            $this->jsonResponse(false, 'Product ID tidak valid');
            return;
        }
        
        // Validasi produk
        $product = $this->product->getById($product_id);
        if (!$product) {
            $this->jsonResponse(false, 'Produk tidak ditemukan');
            return;
        }
        
        // Cek stock
        if ($product['stock'] < $quantity) {
            $this->jsonResponse(false, 'Stock tidak mencukupi');
            return;
        }
        
        // Tambah ke cart
        if ($this->cart->addItem($user_id, $product_id, $quantity)) {
            $cart_count = $this->cart->getCartCount($user_id);
            $this->jsonResponse(true, 'Produk berhasil ditambahkan ke keranjang', [
                'cart_count' => $cart_count
            ]);
        } else {
            $this->jsonResponse(false, 'Gagal menambahkan produk ke keranjang');
        }
    }
    
    private function updateCart() {
        if (!SessionManager::isLoggedIn()) {
            $this->jsonResponse(false, 'Silakan login terlebih dahulu');
            return;
        }
        
        $input = json_decode(file_get_contents('php://input'), true);
        $product_id = $input['product_id'] ?? null;
        $quantity = $input['quantity'] ?? 1;
        $user_id = SessionManager::getUserId();
        
        if (!$product_id) {
            $this->jsonResponse(false, 'Product ID tidak valid');
            return;
        }
        
        // Validasi stock jika quantity > 0
        if ($quantity > 0) {
            $product = $this->product->getById($product_id);
            if ($product['stock'] < $quantity) {
                $this->jsonResponse(false, 'Stock tidak mencukupi');
                return;
            }
        }
        
        if ($this->cart->updateQuantity($user_id, $product_id, $quantity)) {
            $cart_count = $this->cart->getCartCount($user_id);
            $cart_total = $this->cart->getCartTotal($user_id);
            $this->jsonResponse(true, 'Keranjang berhasil diupdate', [
                'cart_count' => $cart_count,
                'cart_total' => $cart_total
            ]);
        } else {
            $this->jsonResponse(false, 'Gagal mengupdate keranjang');
        }
    }
    
    private function removeFromCart() {
        if (!SessionManager::isLoggedIn()) {
            $this->jsonResponse(false, 'Silakan login terlebih dahulu');
            return;
        }
        
        $input = json_decode(file_get_contents('php://input'), true);
        $product_id = $input['product_id'] ?? null;
        $user_id = SessionManager::getUserId();
        
        if (!$product_id) {
            $this->jsonResponse(false, 'Product ID tidak valid');
            return;
        }
        
        if ($this->cart->removeItem($user_id, $product_id)) {
            $cart_count = $this->cart->getCartCount($user_id);
            $this->jsonResponse(true, 'Produk berhasil dihapus dari keranjang', [
                'cart_count' => $cart_count
            ]);
        } else {
            $this->jsonResponse(false, 'Gagal menghapus produk dari keranjang');
        }
    }
    
    private function clearCart() {
        if (!SessionManager::isLoggedIn()) {
            $this->jsonResponse(false, 'Silakan login terlebih dahulu');
            return;
        }
        
        $user_id = SessionManager::getUserId();
        
        if ($this->cart->clearCart($user_id)) {
            $this->jsonResponse(true, 'Keranjang berhasil dikosongkan');
        } else {
            $this->jsonResponse(false, 'Gagal mengosongkan keranjang');
        }
    }
    
    private function getCartCount() {
        if (!SessionManager::isLoggedIn()) {
            $this->jsonResponse(true, '', ['count' => 0]);
            return;
        }
        
        $user_id = SessionManager::getUserId();
        $count = $this->cart->getCartCount($user_id);
        
        $this->jsonResponse(true, '', ['count' => $count]);
    }
    
    private function getCartTotal() {
        if (!SessionManager::isLoggedIn()) {
            $this->jsonResponse(true, '', ['total' => 0]);
            return;
        }
        
        $user_id = SessionManager::getUserId();
        $total = $this->cart->getCartTotal($user_id);
        
        $this->jsonResponse(true, '', ['total' => $total]);
    }
    
    private function viewCart() {
        if (!SessionManager::isLoggedIn()) {
            header('Location: /views/user/login.php');
            exit;
        }
        
        $user_id = SessionManager::getUserId();
        $cart_items = $this->cart->getCartItems($user_id);
        $cart_total = $this->cart->getCartTotal($user_id);
        
        include '../views/user/cart.php';
    }
    
    private function jsonResponse($success, $message, $data = []) {
        header('Content-Type: application/json');
        echo json_encode([
            'success' => $success,
            'message' => $message,
            'data' => $data
        ]);
        exit;
    }
}

// Handle request
$controller = new CartController();
$controller->handleRequest();
?>
```

### 2. Order Management System

**A. Order Model (models/Order.php)**
```php
<?php
class Order {
    private $conn;
    private $table_name = "orders";
    private $order_items_table = "order_items";
    
    public $id;
    public $user_id;
    public $order_number;
    public $total_amount;
    public $status;
    public $payment_method;
    public $payment_status;
    public $shipping_address;
    public $notes;
    public $created_at;
    public $updated_at;
    
    // Status constants
    const STATUS_PENDING = 'pending';
    const STATUS_CONFIRMED = 'confirmed';
    const STATUS_PROCESSING = 'processing';
    const STATUS_SHIPPED = 'shipped';
    const STATUS_DELIVERED = 'delivered';
    const STATUS_CANCELLED = 'cancelled';
    
    const PAYMENT_PENDING = 'pending';
    const PAYMENT_PAID = 'paid';
    const PAYMENT_FAILED = 'failed';
    const PAYMENT_REFUNDED = 'refunded';
    
    public function __construct($db) {
        $this->conn = $db;
    }
    
    // Buat pesanan baru
    public function create($user_id, $cart_items, $shipping_address, $payment_method, $notes = '') {
        try {
            $this->conn->beginTransaction();
            
            // Generate order number
            $order_number = $this->generateOrderNumber();
            
            // Hitung total
            $total_amount = 0;
            foreach ($cart_items as $item) {
                $total_amount += $item['subtotal'];
            }
            
            // Insert order
            $query = "INSERT INTO " . $this->table_name . " 
                     SET user_id = :user_id,
                         order_number = :order_number,
                         total_amount = :total_amount,
                         status = :status,
                         payment_method = :payment_method,
                         payment_status = :payment_status,
                         shipping_address = :shipping_address,
                         notes = :notes,
                         created_at = NOW()";
            
            $stmt = $this->conn->prepare($query);
            
            $stmt->bindParam(':user_id', $user_id);
            $stmt->bindParam(':order_number', $order_number);
            $stmt->bindParam(':total_amount', $total_amount);
            $stmt->bindValue(':status', self::STATUS_PENDING);
            $stmt->bindParam(':payment_method', $payment_method);
            $stmt->bindValue(':payment_status', self::PAYMENT_PENDING);
            $stmt->bindParam(':shipping_address', $shipping_address);
            $stmt->bindParam(':notes', $notes);
            
            $stmt->execute();
            $order_id = $this->conn->lastInsertId();
            
            // Insert order items
            foreach ($cart_items as $item) {
                $this->addOrderItem($order_id, $item);
                
                // Update stock produk
                $this->updateProductStock($item['product_id'], $item['quantity']);
            }
            
            $this->conn->commit();
            return $order_id;
            
        } catch (Exception $e) {
            $this->conn->rollback();
            throw $e;
        }
    }
    
    // Tambah item pesanan
    private function addOrderItem($order_id, $item) {
        $query = "INSERT INTO " . $this->order_items_table . " 
                 SET order_id = :order_id,
                     product_id = :product_id,
                     quantity = :quantity,
                     price = :price,
                     subtotal = :subtotal";
        
        $stmt = $this->conn->prepare($query);
        
        $stmt->bindParam(':order_id', $order_id);
        $stmt->bindParam(':product_id', $item['product_id']);
        $stmt->bindParam(':quantity', $item['quantity']);
        $stmt->bindParam(':price', $item['price']);
        $stmt->bindParam(':subtotal', $item['subtotal']);
        
        return $stmt->execute();
    }
    
    // Update stock produk
    private function updateProductStock($product_id, $quantity) {
        $query = "UPDATE products SET stock = stock - :quantity WHERE id = :product_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':quantity', $quantity);
        $stmt->bindParam(':product_id', $product_id);
        
        return $stmt->execute();
    }
    
    // Generate order number
    private function generateOrderNumber() {
        $prefix = 'ORD';
        $date = date('Ymd');
        $random = str_pad(mt_rand(1, 9999), 4, '0', STR_PAD_LEFT);
        
        return $prefix . $date . $random;
    }
    
    // Ambil pesanan berdasarkan ID
    public function getById($id) {
        $query = "SELECT o.*, u.name as user_name, u.email as user_email
                 FROM " . $this->table_name . " o
                 JOIN users u ON o.user_id = u.id
                 WHERE o.id = :id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':id', $id);
        $stmt->execute();
        
        return $stmt->fetch(PDO::FETCH_ASSOC);
    }
    
    // Ambil item pesanan
    public function getOrderItems($order_id) {
        $query = "SELECT oi.*, p.name as product_name, p.image as product_image
                 FROM " . $this->order_items_table . " oi
                 JOIN products p ON oi.product_id = p.id
                 WHERE oi.order_id = :order_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':order_id', $order_id);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    // Ambil pesanan user
    public function getUserOrders($user_id, $limit = 10, $offset = 0) {
        $query = "SELECT * FROM " . $this->table_name . " 
                 WHERE user_id = :user_id 
                 ORDER BY created_at DESC 
                 LIMIT :limit OFFSET :offset";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':user_id', $user_id);
        $stmt->bindParam(':limit', $limit, PDO::PARAM_INT);
        $stmt->bindParam(':offset', $offset, PDO::PARAM_INT);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    // Update status pesanan
    public function updateStatus($order_id, $status) {
        $query = "UPDATE " . $this->table_name . " 
                 SET status = :status, updated_at = NOW() 
                 WHERE id = :order_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':status', $status);
        $stmt->bindParam(':order_id', $order_id);
        
        return $stmt->execute();
    }
    
    // Update payment status
    public function updatePaymentStatus($order_id, $payment_status) {
        $query = "UPDATE " . $this->table_name . " 
                 SET payment_status = :payment_status, updated_at = NOW() 
                 WHERE id = :order_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':payment_status', $payment_status);
        $stmt->bindParam(':order_id', $order_id);
        
        return $stmt->execute();
    }
    
    // Ambil semua pesanan (untuk admin)
    public function getAll($limit = 20, $offset = 0, $status = null) {
        $where_clause = $status ? "WHERE status = :status" : "";
        
        $query = "SELECT o.*, u.name as user_name, u.email as user_email
                 FROM " . $this->table_name . " o
                 JOIN users u ON o.user_id = u.id
                 $where_clause
                 ORDER BY o.created_at DESC 
                 LIMIT :limit OFFSET :offset";
        
        $stmt = $this->conn->prepare($query);
        
        if ($status) {
            $stmt->bindParam(':status', $status);
        }
        
        $stmt->bindParam(':limit', $limit, PDO::PARAM_INT);
        $stmt->bindParam(':offset', $offset, PDO::PARAM_INT);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    // Statistik pesanan
    public function getOrderStats() {
        $query = "SELECT 
                    COUNT(*) as total_orders,
                    SUM(CASE WHEN status = 'pending' THEN 1 ELSE 0 END) as pending_orders,
                    SUM(CASE WHEN status = 'confirmed' THEN 1 ELSE 0 END) as confirmed_orders,
                    SUM(CASE WHEN status = 'shipped' THEN 1 ELSE 0 END) as shipped_orders,
                    SUM(CASE WHEN status = 'delivered' THEN 1 ELSE 0 END) as delivered_orders,
                    SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) as cancelled_orders,
                    SUM(total_amount) as total_revenue
                 FROM " . $this->table_name;
        
        $stmt = $this->conn->prepare($query);
        $stmt->execute();
        
        return $stmt->fetch(PDO::FETCH_ASSOC);
    }
    
    // Cancel order
    public function cancelOrder($order_id, $reason = '') {
        try {
            $this->conn->beginTransaction();
            
            // Get order items untuk restore stock
            $order_items = $this->getOrderItems($order_id);
            
            // Restore stock
            foreach ($order_items as $item) {
                $this->restoreProductStock($item['product_id'], $item['quantity']);
            }
            
            // Update order status
            $this->updateStatus($order_id, self::STATUS_CANCELLED);
            
            // Log cancellation reason jika ada
            if ($reason) {
                $this->addOrderNote($order_id, "Order cancelled: " . $reason);
            }
            
            $this->conn->commit();
            return true;
            
        } catch (Exception $e) {
            $this->conn->rollback();
            throw $e;
        }
    }
    
    // Restore stock produk
    private function restoreProductStock($product_id, $quantity) {
        $query = "UPDATE products SET stock = stock + :quantity WHERE id = :product_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':quantity', $quantity);
        $stmt->bindParam(':product_id', $product_id);
        
        return $stmt->execute();
    }
    
    // Tambah catatan pesanan
    private function addOrderNote($order_id, $note) {
        $query = "UPDATE " . $this->table_name . " 
                 SET notes = CONCAT(IFNULL(notes, ''), '\n', :note) 
                 WHERE id = :order_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':note', $note);
        $stmt->bindParam(':order_id', $order_id);
        
        return $stmt->execute();
    }
}
?>
```

### 3. Checkout Process

**A. Checkout Controller (controllers/CheckoutController.php)**
```php
<?php
require_once '../config/database.php';
require_once '../models/Cart.php';
require_once '../models/Order.php';
require_once '../models/User.php';
require_once '../config/SessionManager.php';

class CheckoutController {
    private $db;
    private $cart;
    private $order;
    private $user;
    
    public function __construct() {
        $database = new Database();
        $this->db = $database->getConnection();
        $this->cart = new Cart($this->db);
        $this->order = new Order($this->db);
        $this->user = new User($this->db);
        
        SessionManager::start();
    }
    
    public function handleRequest() {
        if (!SessionManager::isLoggedIn()) {
            header('Location: /views/user/login.php');
            exit;
        }
        
        $action = $_POST['action'] ?? 'view';
        
        switch ($action) {
            case 'process':
                $this->processCheckout();
                break;
            default:
                $this->viewCheckout();
        }
    }
    
    private function viewCheckout() {
        $user_id = SessionManager::getUserId();
        
        // Ambil item cart
        $cart_items = $this->cart->getCartItems($user_id);
        
        if (empty($cart_items)) {
            $_SESSION['error_message'] = 'Keranjang belanja kosong';
            header('Location: /views/user/cart.php');
            exit;
        }
        
        // Validasi stock
        $stock_errors = $this->cart->validateStock($user_id);
        if (!empty($stock_errors)) {
            $error_message = 'Stock tidak mencukupi untuk: ';
            foreach ($stock_errors as $error) {
                $error_message .= $error['name'] . ' (tersedia: ' . $error['stock'] . '), ';
            }
            $_SESSION['error_message'] = rtrim($error_message, ', ');
            header('Location: /views/user/cart.php');
            exit;
        }
        
        // Ambil data user
        $user_data = $this->user->getById($user_id);
        
        // Hitung total
        $cart_total = $this->cart->getCartTotal($user_id);
        $shipping_cost = $this->calculateShipping($cart_total);
        $tax = $this->calculateTax($cart_total);
        $grand_total = $cart_total + $shipping_cost + $tax;
        
        include '../views/user/checkout.php';
    }
    
    private function processCheckout() {
        $user_id = SessionManager::getUserId();
        
        // Validasi input
        $required_fields = ['shipping_address', 'payment_method'];
        foreach ($required_fields as $field) {
            if (empty($_POST[$field])) {
                $_SESSION['error_message'] = 'Semua field harus diisi';
                header('Location: /views/user/checkout.php');
                exit;
            }
        }
        
        $shipping_address = $_POST['shipping_address'];
        $payment_method = $_POST['payment_method'];
        $notes = $_POST['notes'] ?? '';
        
        // Ambil item cart
        $cart_items = $this->cart->getCartItems($user_id);
        
        if (empty($cart_items)) {
            $_SESSION['error_message'] = 'Keranjang belanja kosong';
            header('Location: /views/user/cart.php');
            exit;
        }
        
        // Validasi stock sekali lagi
        $stock_errors = $this->cart->validateStock($user_id);
        if (!empty($stock_errors)) {
            $_SESSION['error_message'] = 'Stock tidak mencukupi';
            header('Location: /views/user/cart.php');
            exit;
        }
        
        try {
            // Buat pesanan
            $order_id = $this->order->create($user_id, $cart_items, $shipping_address, $payment_method, $notes);
            
            // Kosongkan cart
            $this->cart->clearCart($user_id);
            
            // Redirect ke halaman sukses
            $_SESSION['success_message'] = 'Pesanan berhasil dibuat';
            header('Location: /views/user/order_success.php?order_id=' . $order_id);
            exit;
            
        } catch (Exception $e) {
            $_SESSION['error_message'] = 'Gagal memproses pesanan: ' . $e->getMessage();
            header('Location: /views/user/checkout.php');
            exit;
        }
    }
    
    private function calculateShipping($total) {
        // Gratis ongkir untuk pembelian di atas 100.000
        if ($total >= 100000) {
            return 0;
        }
        
        // Ongkir flat 15.000
        return 15000;
    }
    
    private function calculateTax($total) {
        // PPN 11%
        return $total * 0.11;
    }
}

// Handle request
$controller = new CheckoutController();
$controller->handleRequest();
?>
```

### 4. Payment Gateway Integration

**A. Payment Service (services/PaymentService.php)**
```php
<?php
class PaymentService {
    private $config;
    
    public function __construct() {
        $this->config = [
            'midtrans' => [
                'server_key' => 'your-midtrans-server-key',
                'client_key' => 'your-midtrans-client-key',
                'is_production' => false
            ],
            'xendit' => [
                'secret_key' => 'your-xendit-secret-key',
                'public_key' => 'your-xendit-public-key'
            ]
        ];
    }
    
    // Generate payment link Midtrans
    public function createMidtransPayment($order) {
        $params = [
            'transaction_details' => [
                'order_id' => $order['order_number'],
                'gross_amount' => (int) $order['total_amount']
            ],
            'customer_details' => [
                'first_name' => $order['user_name'],
                'email' => $order['user_email']
            ],
            'enabled_payments' => [
                'credit_card', 'bca_va', 'bni_va', 'bri_va', 
                'echannel', 'permata_va', 'other_va', 'gopay', 
                'shopeepay', 'qris'
            ]
        ];
        
        $curl = curl_init();
        
        curl_setopt_array($curl, [
            CURLOPT_URL => $this->getMidtransUrl() . '/snap/v1/transactions',
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($params),
            CURLOPT_HTTPHEADER => [
                'Accept: application/json',
                'Content-Type: application/json',
                'Authorization: Basic ' . base64_encode($this->config['midtrans']['server_key'] . ':')
            ]
        ]);
        
        $response = curl_exec($curl);
        $http_code = curl_getinfo($curl, CURLINFO_HTTP_CODE);
        curl_close($curl);
        
        if ($http_code === 201) {
            $result = json_decode($response, true);
            return [
                'success' => true,
                'token' => $result['token'],
                'redirect_url' => $result['redirect_url']
            ];
        }
        
        return [
            'success' => false,
            'message' => 'Failed to create payment'
        ];
    }
    
    // Verify payment Midtrans
    public function verifyMidtransPayment($order_id) {
        $curl = curl_init();
        
        curl_setopt_array($curl, [
            CURLOPT_URL => $this->getMidtransUrl() . '/v2/' . $order_id . '/status',
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Accept: application/json',
                'Authorization: Basic ' . base64_encode($this->config['midtrans']['server_key'] . ':')
            ]
        ]);
        
        $response = curl_exec($curl);
        $http_code = curl_getinfo($curl, CURLINFO_HTTP_CODE);
        curl_close($curl);
        
        if ($http_code === 200) {
            $result = json_decode($response, true);
            return [
                'success' => true,
                'status' => $result['transaction_status'],
                'payment_type' => $result['payment_type'],
                'transaction_time' => $result['transaction_time']
            ];
        }
        
        return [
            'success' => false,
            'message' => 'Failed to verify payment'
        ];
    }
    
    // Create Xendit invoice
    public function createXenditInvoice($order) {
        $params = [
            'external_id' => $order['order_number'],
            'amount' => (int) $order['total_amount'],
            'description' => 'Payment for order ' . $order['order_number'],
            'invoice_duration' => 86400, // 24 hours
            'customer' => [
                'given_names' => $order['user_name'],
                'email' => $order['user_email']
            ],
            'success_redirect_url' => 'https://yoursite.com/payment/success',
            'failure_redirect_url' => 'https://yoursite.com/payment/failed'
        ];
        
        $curl = curl_init();
        
        curl_setopt_array($curl, [
            CURLOPT_URL => 'https://api.xendit.co/v2/invoices',
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($params),
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'Authorization: Basic ' . base64_encode($this->config['xendit']['secret_key'] . ':')
            ]
        ]);
        
        $response = curl_exec($curl);
        $http_code = curl_getinfo($curl, CURLINFO_HTTP_CODE);
        curl_close($curl);
        
        if ($http_code === 200) {
            $result = json_decode($response, true);
            return [
                'success' => true,
                'invoice_id' => $result['id'],
                'invoice_url' => $result['invoice_url']
            ];
        }
        
        return [
            'success' => false,
            'message' => 'Failed to create invoice'
        ];
    }
    
    private function getMidtransUrl() {
        return $this->config['midtrans']['is_production'] 
            ? 'https://api.midtrans.com' 
            : 'https://api.sandbox.midtrans.com';
    }
    
    // Handle webhook dari payment gateway
    public function handleWebhook($payload, $signature, $provider) {
        switch ($provider) {
            case 'midtrans':
                return $this->handleMidtransWebhook($payload, $signature);
            case 'xendit':
                return $this->handleXenditWebhook($payload, $signature);
            default:
                return false;
        }
    }
    
    private function handleMidtransWebhook($payload, $signature) {
        // Verify signature
        $local_signature = hash('sha512', 
            $payload['order_id'] . 
            $payload['status_code'] . 
            $payload['gross_amount'] . 
            $this->config['midtrans']['server_key']
        );
        
        if ($signature !== $local_signature) {
            return false;
        }
        
        // Process payment status
        $order_id = $payload['order_id'];
        $status = $payload['transaction_status'];
        
        // Update order payment status berdasarkan status dari Midtrans
        switch ($status) {
            case 'capture':
            case 'settlement':
                $this->updateOrderPaymentStatus($order_id, 'paid');
                break;
            case 'pending':
                $this->updateOrderPaymentStatus($order_id, 'pending');
                break;
            case 'deny':
            case 'cancel':
            case 'expire':
                $this->updateOrderPaymentStatus($order_id, 'failed');
                break;
        }
        
        return true;
    }
    
    private function updateOrderPaymentStatus($order_number, $payment_status) {
        // Implementation untuk update payment status di database
        $database = new Database();
        $db = $database->getConnection();
        
        $query = "UPDATE orders SET payment_status = :payment_status WHERE order_number = :order_number";
        $stmt = $db->prepare($query);
        $stmt->bindParam(':payment_status', $payment_status);
        $stmt->bindParam(':order_number', $order_number);
        
        return $stmt->execute();
    }
}
?>
```

### 5. Product Review System

**A. Review Model (models/Review.php)**
```php
<?php
class Review {
    private $conn;
    private $table_name = "product_reviews";
    
    public $id;
    public $product_id;
    public $user_id;
    public $rating;
    public $comment;
    public $is_verified;
    public $created_at;
    
    public function __construct($db) {
        $this->conn = $db;
    }
    
    // Tambah review
    public function create($product_id, $user_id, $rating, $comment) {
        // Cek apakah user sudah pernah review produk ini
        if ($this->hasUserReviewed($product_id, $user_id)) {
            return false;
        }
        
        $query = "INSERT INTO " . $this->table_name . " 
                 SET product_id = :product_id,
                     user_id = :user_id,
                     rating = :rating,
                     comment = :comment,
                     created_at = NOW()";
        
        $stmt = $this->conn->prepare($query);
        
        $stmt->bindParam(':product_id', $product_id);
        $stmt->bindParam(':user_id', $user_id);
        $stmt->bindParam(':rating', $rating);
        $stmt->bindParam(':comment', $comment);
        
        if ($stmt->execute()) {
            // Update average rating produk
            $this->updateProductRating($product_id);
            return true;
        }
        
        return false;
    }
    
    // Cek apakah user sudah review
    public function hasUserReviewed($product_id, $user_id) {
        $query = "SELECT id FROM " . $this->table_name . " 
                 WHERE product_id = :product_id AND user_id = :user_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':product_id', $product_id);
        $stmt->bindParam(':user_id', $user_id);
        $stmt->execute();
        
        return $stmt->rowCount() > 0;
    }
    
    // Ambil review produk
    public function getProductReviews($product_id, $limit = 10, $offset = 0) {
        $query = "SELECT r.*, u.name as user_name
                 FROM " . $this->table_name . " r
                 JOIN users u ON r.user_id = u.id
                 WHERE r.product_id = :product_id
                 ORDER BY r.created_at DESC
                 LIMIT :limit OFFSET :offset";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':product_id', $product_id);
        $stmt->bindParam(':limit', $limit, PDO::PARAM_INT);
        $stmt->bindParam(':offset', $offset, PDO::PARAM_INT);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    // Update rating produk
    private function updateProductRating($product_id) {
        $query = "UPDATE products 
                 SET average_rating = (
                     SELECT AVG(rating) 
                     FROM " . $this->table_name . " 
                     WHERE product_id = :product_id
                 ),
                 review_count = (
                     SELECT COUNT(*) 
                     FROM " . $this->table_name . " 
                     WHERE product_id = :product_id
                 )
                 WHERE id = :product_id";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':product_id', $product_id);
        
        return $stmt->execute();
    }
    
    // Ambil statistik rating
    public function getRatingStats($product_id) {
        $query = "SELECT 
                    rating,
                    COUNT(*) as count,
                    (COUNT(*) * 100.0 / (
                        SELECT COUNT(*) 
                        FROM " . $this->table_name . " 
                        WHERE product_id = :product_id
                    )) as percentage
                 FROM " . $this->table_name . " 
                 WHERE product_id = :product_id
                 GROUP BY rating
                 ORDER BY rating DESC";
        
        $stmt = $this->conn->prepare($query);
        $stmt->bindParam(':product_id', $product_id);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
}
?>
```

## Praktikum Hands-On

### Langkah 1: Implementasi Shopping Cart
1. Buat tabel `cart` di database
2. Implementasi Cart model dan controller
3. Buat halaman cart dengan AJAX functionality
4. Test add, update, remove cart items

### Langkah 2: Order Management
1. Buat tabel `orders` dan `order_items`
2. Implementasi Order model dengan transaction
3. Buat checkout process
4. Test order creation dan status update

### Langkah 3: Payment Integration
1. Setup payment gateway (Midtrans/Xendit)
2. Implementasi payment service
3. Handle payment webhook
4. Test payment flow

### Langkah 4: Review System
1. Buat tabel `product_reviews`
2. Implementasi Review model
3. Buat form review dan display
4. Test review functionality

## Tugas dan Evaluasi

### Tugas Individu
1. **Wishlist Feature** (25 poin)
   - Implementasi wishlist functionality
   - AJAX add/remove wishlist
   - Wishlist page dengan sharing

2. **Advanced Search & Filter** (30 poin)
   - Search dengan multiple criteria
   - Filter by price, category, rating
   - Sort functionality
   - Search suggestions

3. **Coupon System** (25 poin)
   - Coupon model dan validation
   - Discount calculation
   - Usage tracking
   - Admin coupon management

4. **Notification System** (20 poin)
   - Email notifications
   - In-app notifications
   - SMS integration
   - Notification preferences

### Kriteria Penilaian
- **Functionality**: Kelengkapan fitur dan business logic
- **Data Integrity**: Transaction handling dan validation
- **User Experience**: Flow dan usability
- **Code Quality**: Structure dan best practices

## Resources Tambahan
- [Payment Gateway Documentation](https://docs.midtrans.com/)
- [E-commerce Best Practices](https://www.shopify.com/blog/ecommerce-best-practices)
- [Database Transaction Guide](https://dev.mysql.com/doc/refman/8.0/en/commit.html)
- [Security in E-commerce](https://owasp.org/www-project-top-ten/)

## Tips dan Best Practices
1. **Always use transactions** untuk operasi yang melibatkan multiple tables
2. **Validate stock** sebelum dan sesudah checkout
3. **Implement proper error handling** untuk payment failures
4. **Log all critical operations** untuk audit trail
5. **Use proper indexing** untuk performance
6. **Implement rate limiting** untuk API calls
7. **Secure payment data** dengan encryption

---

## Navigasi
- [← Bab 4: Frontend dan User Interface](./bab4-frontend-ui.md)
- [Lanjut ke Bab 6: Admin Panel dan Reporting →](./bab6-admin-reporting.md)

---

*Pastikan Anda telah menyelesaikan semua praktikum dan tugas di bab ini sebelum melanjutkan ke bab berikutnya.*