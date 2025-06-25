# BAB 6: Admin Panel dan Reporting

## Deskripsi Bab
Bab ini akan membahas implementasi admin panel yang komprehensif dan sistem reporting untuk mengelola aplikasi e-commerce. Anda akan mempelajari cara membangun dashboard admin, manajemen produk, manajemen pesanan, sistem reporting dengan visualisasi data, dan fitur-fitur administratif lainnya yang essential untuk operasional e-commerce.

## Tujuan Pembelajaran
Setelah menyelesaikan bab ini, peserta diharapkan mampu:
- 🎛️ Membangun dashboard admin yang informatif dan user-friendly
- 📊 Mengimplementasikan sistem reporting dengan chart dan analytics
- 🛠️ Membuat CRUD operations untuk manajemen produk dan kategori
- 📋 Mengimplementasikan manajemen pesanan dan status tracking
- 👥 Membangun sistem manajemen user dan role-based access
- 📈 Membuat real-time analytics dan monitoring
- 🔧 Implementasi system settings dan configuration

## Materi Pembelajaran

### 1. Admin Dashboard

**A. Dashboard Controller (controllers/admin/DashboardController.php)**
```php
<?php
require_once '../../config/database.php';
require_once '../../models/Order.php';
require_once '../../models/Product.php';
require_once '../../models/User.php';
require_once '../../config/SessionManager.php';
require_once '../../middleware/AuthMiddleware.php';

class DashboardController {
    private $db;
    private $order;
    private $product;
    private $user;
    
    public function __construct() {
        $database = new Database();
        $this->db = $database->getConnection();
        $this->order = new Order($this->db);
        $this->product = new Product($this->db);
        $this->user = new User($this->db);
        
        SessionManager::start();
        
        // Check admin access
        AuthMiddleware::requireAdmin();
    }
    
    public function index() {
        // Get dashboard statistics
        $stats = $this->getDashboardStats();
        $recent_orders = $this->getRecentOrders();
        $top_products = $this->getTopProducts();
        $sales_chart_data = $this->getSalesChartData();
        $order_status_data = $this->getOrderStatusData();
        
        $pageTitle = 'Admin Dashboard';
        include '../../views/admin/dashboard.php';
    }
    
    private function getDashboardStats() {
        // Total orders today
        $query = "SELECT COUNT(*) as count, SUM(total_amount) as revenue 
                 FROM orders 
                 WHERE DATE(created_at) = CURDATE()";
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        $today_stats = $stmt->fetch(PDO::FETCH_ASSOC);
        
        // Total orders this month
        $query = "SELECT COUNT(*) as count, SUM(total_amount) as revenue 
                 FROM orders 
                 WHERE MONTH(created_at) = MONTH(CURDATE()) 
                 AND YEAR(created_at) = YEAR(CURDATE())";
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        $month_stats = $stmt->fetch(PDO::FETCH_ASSOC);
        
        // Total products
        $query = "SELECT COUNT(*) as count FROM products";
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        $product_count = $stmt->fetch(PDO::FETCH_ASSOC)['count'];
        
        // Total users
        $query = "SELECT COUNT(*) as count FROM users WHERE role = 'user'";
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        $user_count = $stmt->fetch(PDO::FETCH_ASSOC)['count'];
        
        // Low stock products
        $query = "SELECT COUNT(*) as count FROM products WHERE stock <= 10";
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        $low_stock_count = $stmt->fetch(PDO::FETCH_ASSOC)['count'];
        
        // Pending orders
        $query = "SELECT COUNT(*) as count FROM orders WHERE status = 'pending'";
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        $pending_orders = $stmt->fetch(PDO::FETCH_ASSOC)['count'];
        
        return [
            'today_orders' => $today_stats['count'] ?? 0,
            'today_revenue' => $today_stats['revenue'] ?? 0,
            'month_orders' => $month_stats['count'] ?? 0,
            'month_revenue' => $month_stats['revenue'] ?? 0,
            'total_products' => $product_count,
            'total_users' => $user_count,
            'low_stock_products' => $low_stock_count,
            'pending_orders' => $pending_orders
        ];
    }
    
    private function getRecentOrders($limit = 10) {
        $query = "SELECT o.*, u.name as user_name 
                 FROM orders o 
                 JOIN users u ON o.user_id = u.id 
                 ORDER BY o.created_at DESC 
                 LIMIT :limit";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindParam(':limit', $limit, PDO::PARAM_INT);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    private function getTopProducts($limit = 5) {
        $query = "SELECT p.name, p.price, SUM(oi.quantity) as total_sold,
                         SUM(oi.subtotal) as total_revenue
                 FROM products p
                 JOIN order_items oi ON p.id = oi.product_id
                 JOIN orders o ON oi.order_id = o.id
                 WHERE o.status != 'cancelled'
                 GROUP BY p.id
                 ORDER BY total_sold DESC
                 LIMIT :limit";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindParam(':limit', $limit, PDO::PARAM_INT);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    private function getSalesChartData($days = 30) {
        $query = "SELECT DATE(created_at) as date, 
                         COUNT(*) as orders,
                         SUM(total_amount) as revenue
                 FROM orders 
                 WHERE created_at >= DATE_SUB(CURDATE(), INTERVAL :days DAY)
                 AND status != 'cancelled'
                 GROUP BY DATE(created_at)
                 ORDER BY date ASC";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindParam(':days', $days, PDO::PARAM_INT);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    private function getOrderStatusData() {
        $query = "SELECT status, COUNT(*) as count 
                 FROM orders 
                 GROUP BY status";
        
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    public function getApiData() {
        $type = $_GET['type'] ?? '';
        
        switch ($type) {
            case 'sales_chart':
                $days = $_GET['days'] ?? 30;
                $data = $this->getSalesChartData($days);
                break;
            case 'order_status':
                $data = $this->getOrderStatusData();
                break;
            case 'stats':
                $data = $this->getDashboardStats();
                break;
            default:
                $data = [];
        }
        
        header('Content-Type: application/json');
        echo json_encode($data);
        exit;
    }
}

// Handle request
if (isset($_GET['api'])) {
    $controller = new DashboardController();
    $controller->getApiData();
} else {
    $controller = new DashboardController();
    $controller->index();
}
?>
```

**B. Dashboard View (views/admin/dashboard.php)**
```php
<?php include '../layouts/admin_header.php'; ?>

<div class="container-fluid py-4">
    <!-- Page Header -->
    <div class="row mb-4">
        <div class="col-12">
            <h1 class="h3 mb-0 text-gray-800">Dashboard</h1>
            <p class="text-muted">Selamat datang di panel admin Simple E-Commerce</p>
        </div>
    </div>
    
    <!-- Statistics Cards -->
    <div class="row mb-4">
        <!-- Today's Orders -->
        <div class="col-xl-3 col-md-6 mb-4">
            <div class="card dashboard-card border-left-primary shadow h-100 py-2">
                <div class="card-body">
                    <div class="row no-gutters align-items-center">
                        <div class="col mr-2">
                            <div class="text-xs font-weight-bold text-primary text-uppercase mb-1">
                                Pesanan Hari Ini
                            </div>
                            <div class="h5 mb-0 font-weight-bold text-gray-800">
                                <?php echo number_format($stats['today_orders']); ?>
                            </div>
                        </div>
                        <div class="col-auto">
                            <i class="fas fa-shopping-cart fa-2x text-gray-300"></i>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Today's Revenue -->
        <div class="col-xl-3 col-md-6 mb-4">
            <div class="card dashboard-card border-left-success shadow h-100 py-2">
                <div class="card-body">
                    <div class="row no-gutters align-items-center">
                        <div class="col mr-2">
                            <div class="text-xs font-weight-bold text-success text-uppercase mb-1">
                                Revenue Hari Ini
                            </div>
                            <div class="h5 mb-0 font-weight-bold text-gray-800">
                                Rp <?php echo number_format($stats['today_revenue'], 0, ',', '.'); ?>
                            </div>
                        </div>
                        <div class="col-auto">
                            <i class="fas fa-dollar-sign fa-2x text-gray-300"></i>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Monthly Orders -->
        <div class="col-xl-3 col-md-6 mb-4">
            <div class="card dashboard-card border-left-info shadow h-100 py-2">
                <div class="card-body">
                    <div class="row no-gutters align-items-center">
                        <div class="col mr-2">
                            <div class="text-xs font-weight-bold text-info text-uppercase mb-1">
                                Pesanan Bulan Ini
                            </div>
                            <div class="h5 mb-0 font-weight-bold text-gray-800">
                                <?php echo number_format($stats['month_orders']); ?>
                            </div>
                        </div>
                        <div class="col-auto">
                            <i class="fas fa-calendar fa-2x text-gray-300"></i>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Pending Orders -->
        <div class="col-xl-3 col-md-6 mb-4">
            <div class="card dashboard-card border-left-warning shadow h-100 py-2">
                <div class="card-body">
                    <div class="row no-gutters align-items-center">
                        <div class="col mr-2">
                            <div class="text-xs font-weight-bold text-warning text-uppercase mb-1">
                                Pesanan Pending
                            </div>
                            <div class="h5 mb-0 font-weight-bold text-gray-800">
                                <?php echo number_format($stats['pending_orders']); ?>
                            </div>
                        </div>
                        <div class="col-auto">
                            <i class="fas fa-clock fa-2x text-gray-300"></i>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <!-- Charts Row -->
    <div class="row mb-4">
        <!-- Sales Chart -->
        <div class="col-xl-8 col-lg-7">
            <div class="card shadow mb-4">
                <div class="card-header py-3 d-flex flex-row align-items-center justify-content-between">
                    <h6 class="m-0 font-weight-bold text-primary">Grafik Penjualan (30 Hari Terakhir)</h6>
                    <div class="dropdown no-arrow">
                        <select id="salesPeriod" class="form-select form-select-sm">
                            <option value="7">7 Hari</option>
                            <option value="30" selected>30 Hari</option>
                            <option value="90">90 Hari</option>
                        </select>
                    </div>
                </div>
                <div class="card-body">
                    <div class="chart-area">
                        <canvas id="salesChart" width="400" height="200"></canvas>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Order Status Pie Chart -->
        <div class="col-xl-4 col-lg-5">
            <div class="card shadow mb-4">
                <div class="card-header py-3">
                    <h6 class="m-0 font-weight-bold text-primary">Status Pesanan</h6>
                </div>
                <div class="card-body">
                    <div class="chart-pie pt-4 pb-2">
                        <canvas id="orderStatusChart" width="400" height="400"></canvas>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <!-- Content Row -->
    <div class="row">
        <!-- Recent Orders -->
        <div class="col-lg-8 mb-4">
            <div class="card shadow mb-4">
                <div class="card-header py-3">
                    <h6 class="m-0 font-weight-bold text-primary">Pesanan Terbaru</h6>
                </div>
                <div class="card-body">
                    <div class="table-responsive">
                        <table class="table table-bordered" width="100%" cellspacing="0">
                            <thead>
                                <tr>
                                    <th>Order #</th>
                                    <th>Customer</th>
                                    <th>Total</th>
                                    <th>Status</th>
                                    <th>Tanggal</th>
                                    <th>Aksi</th>
                                </tr>
                            </thead>
                            <tbody>
                                <?php foreach ($recent_orders as $order): ?>
                                <tr>
                                    <td><?php echo htmlspecialchars($order['order_number']); ?></td>
                                    <td><?php echo htmlspecialchars($order['user_name']); ?></td>
                                    <td>Rp <?php echo number_format($order['total_amount'], 0, ',', '.'); ?></td>
                                    <td>
                                        <span class="badge badge-<?php echo $this->getStatusBadgeClass($order['status']); ?>">
                                            <?php echo ucfirst($order['status']); ?>
                                        </span>
                                    </td>
                                    <td><?php echo date('d/m/Y H:i', strtotime($order['created_at'])); ?></td>
                                    <td>
                                        <a href="/views/admin/order_detail.php?id=<?php echo $order['id']; ?>" 
                                           class="btn btn-sm btn-primary">
                                            <i class="fas fa-eye"></i>
                                        </a>
                                    </td>
                                </tr>
                                <?php endforeach; ?>
                            </tbody>
                        </table>
                    </div>
                    <div class="text-center">
                        <a href="/views/admin/orders.php" class="btn btn-primary">Lihat Semua Pesanan</a>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Top Products -->
        <div class="col-lg-4 mb-4">
            <div class="card shadow mb-4">
                <div class="card-header py-3">
                    <h6 class="m-0 font-weight-bold text-primary">Produk Terlaris</h6>
                </div>
                <div class="card-body">
                    <?php foreach ($top_products as $index => $product): ?>
                    <div class="d-flex align-items-center mb-3">
                        <div class="mr-3">
                            <div class="icon-circle bg-primary">
                                <span class="text-white"><?php echo $index + 1; ?></span>
                            </div>
                        </div>
                        <div class="flex-grow-1">
                            <div class="small text-gray-500"><?php echo htmlspecialchars($product['name']); ?></div>
                            <div class="font-weight-bold">
                                <?php echo number_format($product['total_sold']); ?> terjual
                            </div>
                        </div>
                        <div class="text-right">
                            <div class="text-xs text-gray-500">Revenue</div>
                            <div class="font-weight-bold text-success">
                                Rp <?php echo number_format($product['total_revenue'], 0, ',', '.'); ?>
                            </div>
                        </div>
                    </div>
                    <?php endforeach; ?>
                </div>
            </div>
        </div>
    </div>
    
    <!-- Quick Actions -->
    <div class="row">
        <div class="col-12">
            <div class="card shadow mb-4">
                <div class="card-header py-3">
                    <h6 class="m-0 font-weight-bold text-primary">Quick Actions</h6>
                </div>
                <div class="card-body">
                    <div class="row">
                        <div class="col-md-3 mb-3">
                            <a href="/views/admin/create_product.php" class="btn btn-success btn-block">
                                <i class="fas fa-plus mr-2"></i>Tambah Produk
                            </a>
                        </div>
                        <div class="col-md-3 mb-3">
                            <a href="/views/admin/orders.php?status=pending" class="btn btn-warning btn-block">
                                <i class="fas fa-clock mr-2"></i>Pesanan Pending
                            </a>
                        </div>
                        <div class="col-md-3 mb-3">
                            <a href="/views/admin/products.php?filter=low_stock" class="btn btn-danger btn-block">
                                <i class="fas fa-exclamation-triangle mr-2"></i>Stock Rendah
                            </a>
                        </div>
                        <div class="col-md-3 mb-3">
                            <a href="/views/admin/reports.php" class="btn btn-info btn-block">
                                <i class="fas fa-chart-bar mr-2"></i>Laporan
                            </a>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>

<!-- Chart.js Scripts -->
<script>
// Sales Chart
const salesCtx = document.getElementById('salesChart').getContext('2d');
let salesChart;

function loadSalesChart(days = 30) {
    fetch(`/controllers/admin/DashboardController.php?api=1&type=sales_chart&days=${days}`)
        .then(response => response.json())
        .then(data => {
            const labels = data.map(item => {
                const date = new Date(item.date);
                return date.toLocaleDateString('id-ID', { month: 'short', day: 'numeric' });
            });
            const orders = data.map(item => item.orders);
            const revenue = data.map(item => item.revenue);
            
            if (salesChart) {
                salesChart.destroy();
            }
            
            salesChart = new Chart(salesCtx, {
                type: 'line',
                data: {
                    labels: labels,
                    datasets: [{
                        label: 'Pesanan',
                        data: orders,
                        borderColor: 'rgb(54, 162, 235)',
                        backgroundColor: 'rgba(54, 162, 235, 0.1)',
                        yAxisID: 'y'
                    }, {
                        label: 'Revenue (Rp)',
                        data: revenue,
                        borderColor: 'rgb(255, 99, 132)',
                        backgroundColor: 'rgba(255, 99, 132, 0.1)',
                        yAxisID: 'y1'
                    }]
                },
                options: {
                    responsive: true,
                    interaction: {
                        mode: 'index',
                        intersect: false,
                    },
                    scales: {
                        x: {
                            display: true,
                            title: {
                                display: true,
                                text: 'Tanggal'
                            }
                        },
                        y: {
                            type: 'linear',
                            display: true,
                            position: 'left',
                            title: {
                                display: true,
                                text: 'Jumlah Pesanan'
                            }
                        },
                        y1: {
                            type: 'linear',
                            display: true,
                            position: 'right',
                            title: {
                                display: true,
                                text: 'Revenue (Rp)'
                            },
                            grid: {
                                drawOnChartArea: false,
                            },
                        }
                    }
                }
            });
        })
        .catch(error => {
            console.error('Error loading sales chart:', error);
        });
}

// Order Status Chart
const statusCtx = document.getElementById('orderStatusChart').getContext('2d');

fetch('/controllers/admin/DashboardController.php?api=1&type=order_status')
    .then(response => response.json())
    .then(data => {
        const labels = data.map(item => item.status.charAt(0).toUpperCase() + item.status.slice(1));
        const counts = data.map(item => item.count);
        const colors = {
            'Pending': '#ffc107',
            'Confirmed': '#17a2b8',
            'Processing': '#6f42c1',
            'Shipped': '#fd7e14',
            'Delivered': '#28a745',
            'Cancelled': '#dc3545'
        };
        const backgroundColors = labels.map(label => colors[label] || '#6c757d');
        
        new Chart(statusCtx, {
            type: 'doughnut',
            data: {
                labels: labels,
                datasets: [{
                    data: counts,
                    backgroundColor: backgroundColors,
                    borderWidth: 2,
                    borderColor: '#fff'
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: {
                        position: 'bottom'
                    }
                }
            }
        });
    })
    .catch(error => {
        console.error('Error loading order status chart:', error);
    });

// Event listeners
document.getElementById('salesPeriod').addEventListener('change', function() {
    loadSalesChart(this.value);
});

// Load initial chart
loadSalesChart();

// Auto refresh every 5 minutes
setInterval(() => {
    loadSalesChart(document.getElementById('salesPeriod').value);
}, 300000);
</script>

<?php 
function getStatusBadgeClass($status) {
    switch ($status) {
        case 'pending': return 'warning';
        case 'confirmed': return 'info';
        case 'processing': return 'primary';
        case 'shipped': return 'secondary';
        case 'delivered': return 'success';
        case 'cancelled': return 'danger';
        default: return 'secondary';
    }
}

include '../layouts/admin_footer.php'; 
?>
```

### 2. Product Management

**A. Product Management Controller (controllers/admin/ProductController.php)**
```php
<?php
require_once '../../config/database.php';
require_once '../../models/Product.php';
require_once '../../models/Category.php';
require_once '../../config/SessionManager.php';
require_once '../../middleware/AuthMiddleware.php';

class ProductController {
    private $db;
    private $product;
    private $category;
    
    public function __construct() {
        $database = new Database();
        $this->db = $database->getConnection();
        $this->product = new Product($this->db);
        $this->category = new Category($this->db);
        
        SessionManager::start();
        AuthMiddleware::requireAdmin();
    }
    
    public function handleRequest() {
        $action = $_GET['action'] ?? $_POST['action'] ?? 'list';
        
        switch ($action) {
            case 'create':
                $this->create();
                break;
            case 'store':
                $this->store();
                break;
            case 'edit':
                $this->edit();
                break;
            case 'update':
                $this->update();
                break;
            case 'delete':
                $this->delete();
                break;
            case 'bulk_action':
                $this->bulkAction();
                break;
            case 'export':
                $this->export();
                break;
            case 'import':
                $this->import();
                break;
            default:
                $this->index();
        }
    }
    
    public function index() {
        $page = $_GET['page'] ?? 1;
        $limit = 20;
        $offset = ($page - 1) * $limit;
        $search = $_GET['search'] ?? '';
        $category_filter = $_GET['category'] ?? '';
        $stock_filter = $_GET['stock_filter'] ?? '';
        
        // Build where conditions
        $where_conditions = [];
        $params = [];
        
        if ($search) {
            $where_conditions[] = "(p.name LIKE :search OR p.description LIKE :search)";
            $params[':search'] = "%$search%";
        }
        
        if ($category_filter) {
            $where_conditions[] = "p.category_id = :category_id";
            $params[':category_id'] = $category_filter;
        }
        
        if ($stock_filter === 'low') {
            $where_conditions[] = "p.stock <= 10";
        } elseif ($stock_filter === 'out') {
            $where_conditions[] = "p.stock = 0";
        }
        
        $where_clause = !empty($where_conditions) ? 'WHERE ' . implode(' AND ', $where_conditions) : '';
        
        // Get products
        $query = "SELECT p.*, c.name as category_name 
                 FROM products p 
                 LEFT JOIN categories c ON p.category_id = c.id 
                 $where_clause 
                 ORDER BY p.created_at DESC 
                 LIMIT :limit OFFSET :offset";
        
        $stmt = $this->db->prepare($query);
        
        foreach ($params as $key => $value) {
            $stmt->bindValue($key, $value);
        }
        
        $stmt->bindValue(':limit', $limit, PDO::PARAM_INT);
        $stmt->bindValue(':offset', $offset, PDO::PARAM_INT);
        $stmt->execute();
        
        $products = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        // Get total count for pagination
        $count_query = "SELECT COUNT(*) as total 
                       FROM products p 
                       LEFT JOIN categories c ON p.category_id = c.id 
                       $where_clause";
        
        $count_stmt = $this->db->prepare($count_query);
        
        foreach ($params as $key => $value) {
            $count_stmt->bindValue($key, $value);
        }
        
        $count_stmt->execute();
        $total_products = $count_stmt->fetch(PDO::FETCH_ASSOC)['total'];
        $total_pages = ceil($total_products / $limit);
        
        // Get categories for filter
        $categories = $this->category->getAll();
        
        $pageTitle = 'Manajemen Produk';
        include '../../views/admin/products.php';
    }
    
    public function create() {
        $categories = $this->category->getAll();
        $pageTitle = 'Tambah Produk';
        include '../../views/admin/create_product.php';
    }
    
    public function store() {
        // Validasi input
        $required_fields = ['name', 'description', 'price', 'stock', 'category_id'];
        foreach ($required_fields as $field) {
            if (empty($_POST[$field])) {
                $_SESSION['error_message'] = 'Semua field harus diisi';
                header('Location: /views/admin/create_product.php');
                exit;
            }
        }
        
        $name = $_POST['name'];
        $description = $_POST['description'];
        $price = $_POST['price'];
        $stock = $_POST['stock'];
        $category_id = $_POST['category_id'];
        $is_featured = isset($_POST['is_featured']) ? 1 : 0;
        
        // Handle image upload
        $image_path = null;
        if (isset($_FILES['image']) && $_FILES['image']['error'] === UPLOAD_ERR_OK) {
            $image_path = $this->handleImageUpload($_FILES['image']);
            if (!$image_path) {
                $_SESSION['error_message'] = 'Gagal mengupload gambar';
                header('Location: /views/admin/create_product.php');
                exit;
            }
        }
        
        // Create product
        if ($this->product->create($name, $description, $price, $stock, $category_id, $image_path, $is_featured)) {
            $_SESSION['success_message'] = 'Produk berhasil ditambahkan';
            header('Location: /views/admin/products.php');
        } else {
            $_SESSION['error_message'] = 'Gagal menambahkan produk';
            header('Location: /views/admin/create_product.php');
        }
        exit;
    }
    
    public function edit() {
        $id = $_GET['id'] ?? null;
        if (!$id) {
            $_SESSION['error_message'] = 'ID produk tidak valid';
            header('Location: /views/admin/products.php');
            exit;
        }
        
        $product = $this->product->getById($id);
        if (!$product) {
            $_SESSION['error_message'] = 'Produk tidak ditemukan';
            header('Location: /views/admin/products.php');
            exit;
        }
        
        $categories = $this->category->getAll();
        $pageTitle = 'Edit Produk';
        include '../../views/admin/edit_product.php';
    }
    
    public function update() {
        $id = $_POST['id'] ?? null;
        if (!$id) {
            $_SESSION['error_message'] = 'ID produk tidak valid';
            header('Location: /views/admin/products.php');
            exit;
        }
        
        // Validasi input
        $required_fields = ['name', 'description', 'price', 'stock', 'category_id'];
        foreach ($required_fields as $field) {
            if (empty($_POST[$field])) {
                $_SESSION['error_message'] = 'Semua field harus diisi';
                header('Location: /views/admin/edit_product.php?id=' . $id);
                exit;
            }
        }
        
        $name = $_POST['name'];
        $description = $_POST['description'];
        $price = $_POST['price'];
        $stock = $_POST['stock'];
        $category_id = $_POST['category_id'];
        $is_featured = isset($_POST['is_featured']) ? 1 : 0;
        
        // Handle image upload
        $image_path = $_POST['current_image'] ?? null;
        if (isset($_FILES['image']) && $_FILES['image']['error'] === UPLOAD_ERR_OK) {
            $new_image = $this->handleImageUpload($_FILES['image']);
            if ($new_image) {
                // Delete old image
                if ($image_path && file_exists('../../' . $image_path)) {
                    unlink('../../' . $image_path);
                }
                $image_path = $new_image;
            }
        }
        
        // Update product
        if ($this->product->update($id, $name, $description, $price, $stock, $category_id, $image_path, $is_featured)) {
            $_SESSION['success_message'] = 'Produk berhasil diupdate';
            header('Location: /views/admin/products.php');
        } else {
            $_SESSION['error_message'] = 'Gagal mengupdate produk';
            header('Location: /views/admin/edit_product.php?id=' . $id);
        }
        exit;
    }
    
    public function delete() {
        $id = $_POST['id'] ?? null;
        if (!$id) {
            $this->jsonResponse(false, 'ID produk tidak valid');
            return;
        }
        
        // Get product to delete image
        $product = $this->product->getById($id);
        
        if ($this->product->delete($id)) {
            // Delete image file
            if ($product && $product['image'] && file_exists('../../' . $product['image'])) {
                unlink('../../' . $product['image']);
            }
            
            $this->jsonResponse(true, 'Produk berhasil dihapus');
        } else {
            $this->jsonResponse(false, 'Gagal menghapus produk');
        }
    }
    
    public function bulkAction() {
        $action = $_POST['bulk_action'] ?? null;
        $product_ids = $_POST['product_ids'] ?? [];
        
        if (!$action || empty($product_ids)) {
            $this->jsonResponse(false, 'Aksi atau produk tidak valid');
            return;
        }
        
        $success_count = 0;
        
        switch ($action) {
            case 'delete':
                foreach ($product_ids as $id) {
                    $product = $this->product->getById($id);
                    if ($this->product->delete($id)) {
                        // Delete image file
                        if ($product && $product['image'] && file_exists('../../' . $product['image'])) {
                            unlink('../../' . $product['image']);
                        }
                        $success_count++;
                    }
                }
                $message = "$success_count produk berhasil dihapus";
                break;
                
            case 'feature':
                foreach ($product_ids as $id) {
                    if ($this->product->updateFeatured($id, 1)) {
                        $success_count++;
                    }
                }
                $message = "$success_count produk berhasil dijadikan featured";
                break;
                
            case 'unfeature':
                foreach ($product_ids as $id) {
                    if ($this->product->updateFeatured($id, 0)) {
                        $success_count++;
                    }
                }
                $message = "$success_count produk berhasil dihapus dari featured";
                break;
                
            default:
                $this->jsonResponse(false, 'Aksi tidak valid');
                return;
        }
        
        $this->jsonResponse(true, $message);
    }
    
    private function handleImageUpload($file) {
        $allowed_types = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
        $max_size = 5 * 1024 * 1024; // 5MB
        
        if (!in_array($file['type'], $allowed_types)) {
            return false;
        }
        
        if ($file['size'] > $max_size) {
            return false;
        }
        
        $upload_dir = '../../assets/images/products/';
        if (!is_dir($upload_dir)) {
            mkdir($upload_dir, 0755, true);
        }
        
        $file_extension = pathinfo($file['name'], PATHINFO_EXTENSION);
        $file_name = uniqid() . '.' . $file_extension;
        $file_path = $upload_dir . $file_name;
        
        if (move_uploaded_file($file['tmp_name'], $file_path)) {
            return 'assets/images/products/' . $file_name;
        }
        
        return false;
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
$controller = new ProductController();
$controller->handleRequest();
?>
```

### 3. Order Management

**A. Order Management View (views/admin/orders.php)**
```php
<?php include '../layouts/admin_header.php'; ?>

<div class="container-fluid py-4">
    <!-- Page Header -->
    <div class="row mb-4">
        <div class="col-12">
            <div class="d-flex justify-content-between align-items-center">
                <div>
                    <h1 class="h3 mb-0 text-gray-800">Manajemen Pesanan</h1>
                    <p class="text-muted">Kelola semua pesanan pelanggan</p>
                </div>
                <div>
                    <button class="btn btn-primary" onclick="exportOrders()">
                        <i class="fas fa-download mr-2"></i>Export
                    </button>
                </div>
            </div>
        </div>
    </div>
    
    <!-- Filters -->
    <div class="card shadow mb-4">
        <div class="card-body">
            <form method="GET" class="row g-3">
                <div class="col-md-3">
                    <label class="form-label">Status</label>
                    <select name="status" class="form-select">
                        <option value="">Semua Status</option>
                        <option value="pending" <?php echo ($status_filter === 'pending') ? 'selected' : ''; ?>>Pending</option>
                        <option value="confirmed" <?php echo ($status_filter === 'confirmed') ? 'selected' : ''; ?>>Confirmed</option>
                        <option value="processing" <?php echo ($status_filter === 'processing') ? 'selected' : ''; ?>>Processing</option>
                        <option value="shipped" <?php echo ($status_filter === 'shipped') ? 'selected' : ''; ?>>Shipped</option>
                        <option value="delivered" <?php echo ($status_filter === 'delivered') ? 'selected' : ''; ?>>Delivered</option>
                        <option value="cancelled" <?php echo ($status_filter === 'cancelled') ? 'selected' : ''; ?>>Cancelled</option>
                    </select>
                </div>
                <div class="col-md-3">
                    <label class="form-label">Payment Status</label>
                    <select name="payment_status" class="form-select">
                        <option value="">Semua Payment</option>
                        <option value="pending" <?php echo ($payment_filter === 'pending') ? 'selected' : ''; ?>>Pending</option>
                        <option value="paid" <?php echo ($payment_filter === 'paid') ? 'selected' : ''; ?>>Paid</option>
                        <option value="failed" <?php echo ($payment_filter === 'failed') ? 'selected' : ''; ?>>Failed</option>
                        <option value="refunded" <?php echo ($payment_filter === 'refunded') ? 'selected' : ''; ?>>Refunded</option>
                    </select>
                </div>
                <div class="col-md-3">
                    <label class="form-label">Tanggal</label>
                    <input type="date" name="date_from" class="form-control" value="<?php echo $date_from; ?>">
                </div>
                <div class="col-md-3">
                    <label class="form-label">&nbsp;</label>
                    <div class="d-flex gap-2">
                        <input type="date" name="date_to" class="form-control" value="<?php echo $date_to; ?>">
                        <button type="submit" class="btn btn-primary">
                            <i class="fas fa-search"></i>
                        </button>
                        <a href="/views/admin/orders.php" class="btn btn-secondary">
                            <i class="fas fa-times"></i>
                        </a>
                    </div>
                </div>
            </form>
        </div>
    </div>
    
    <!-- Orders Table -->
    <div class="card shadow mb-4">
        <div class="card-header py-3">
            <div class="d-flex justify-content-between align-items-center">
                <h6 class="m-0 font-weight-bold text-primary">Daftar Pesanan</h6>
                <div>
                    <button class="btn btn-sm btn-secondary" onclick="selectAll()">
                        <i class="fas fa-check-square mr-1"></i>Select All
                    </button>
                    <div class="btn-group">
                        <button type="button" class="btn btn-sm btn-primary dropdown-toggle" data-bs-toggle="dropdown">
                            Bulk Actions
                        </button>
                        <ul class="dropdown-menu">
                            <li><a class="dropdown-item" href="#" onclick="bulkUpdateStatus('confirmed')">Mark as Confirmed</a></li>
                            <li><a class="dropdown-item" href="#" onclick="bulkUpdateStatus('processing')">Mark as Processing</a></li>
                            <li><a class="dropdown-item" href="#" onclick="bulkUpdateStatus('shipped')">Mark as Shipped</a></li>
                            <li><a class="dropdown-item" href="#" onclick="bulkUpdateStatus('delivered')">Mark as Delivered</a></li>
                            <li><hr class="dropdown-divider"></li>
                            <li><a class="dropdown-item text-danger" href="#" onclick="bulkUpdateStatus('cancelled')">Cancel Orders</a></li>
                        </ul>
                    </div>
                </div>
            </div>
        </div>
        <div class="card-body">
            <div class="table-responsive">
                <table class="table table-bordered" id="ordersTable">
                    <thead>
                        <tr>
                            <th width="30">
                                <input type="checkbox" id="selectAllCheckbox">
                            </th>
                            <th>Order #</th>
                            <th>Customer</th>
                            <th>Total</th>
                            <th>Status</th>
                            <th>Payment</th>
                            <th>Tanggal</th>
                            <th>Aksi</th>
                        </tr>
                    </thead>
                    <tbody>
                        <?php foreach ($orders as $order): ?>
                        <tr>
                            <td>
                                <input type="checkbox" class="order-checkbox" value="<?php echo $order['id']; ?>">
                            </td>
                            <td>
                                <a href="/views/admin/order_detail.php?id=<?php echo $order['id']; ?>" class="text-decoration-none">
                                    <?php echo htmlspecialchars($order['order_number']); ?>
                                </a>
                            </td>
                            <td>
                                <div>
                                    <strong><?php echo htmlspecialchars($order['user_name']); ?></strong>
                                    <br>
                                    <small class="text-muted"><?php echo htmlspecialchars($order['user_email']); ?></small>
                                </div>
                            </td>
                            <td>
                                <strong>Rp <?php echo number_format($order['total_amount'], 0, ',', '.'); ?></strong>
                            </td>
                            <td>
                                <select class="form-select form-select-sm status-select" 
                                        data-order-id="<?php echo $order['id']; ?>" 
                                        data-current-status="<?php echo $order['status']; ?>">
                                    <option value="pending" <?php echo ($order['status'] === 'pending') ? 'selected' : ''; ?>>Pending</option>
                                    <option value="confirmed" <?php echo ($order['status'] === 'confirmed') ? 'selected' : ''; ?>>Confirmed</option>
                                    <option value="processing" <?php echo ($order['status'] === 'processing') ? 'selected' : ''; ?>>Processing</option>
                                    <option value="shipped" <?php echo ($order['status'] === 'shipped') ? 'selected' : ''; ?>>Shipped</option>
                                    <option value="delivered" <?php echo ($order['status'] === 'delivered') ? 'selected' : ''; ?>>Delivered</option>
                                    <option value="cancelled" <?php echo ($order['status'] === 'cancelled') ? 'selected' : ''; ?>>Cancelled</option>
                                </select>
                            </td>
                            <td>
                                <span class="badge badge-<?php echo getPaymentBadgeClass($order['payment_status']); ?>">
                                    <?php echo ucfirst($order['payment_status']); ?>
                                </span>
                            </td>
                            <td>
                                <?php echo date('d/m/Y H:i', strtotime($order['created_at'])); ?>
                            </td>
                            <td>
                                <div class="btn-group">
                                    <a href="/views/admin/order_detail.php?id=<?php echo $order['id']; ?>" 
                                       class="btn btn-sm btn-primary" title="Detail">
                                        <i class="fas fa-eye"></i>
                                    </a>
                                    <button class="btn btn-sm btn-info" 
                                            onclick="printInvoice(<?php echo $order['id']; ?>)" 
                                            title="Print Invoice">
                                        <i class="fas fa-print"></i>
                                    </button>
                                    <?php if ($order['status'] !== 'cancelled' && $order['status'] !== 'delivered'): ?>
                                    <button class="btn btn-sm btn-danger" 
                                            onclick="cancelOrder(<?php echo $order['id']; ?>)" 
                                            title="Cancel">
                                        <i class="fas fa-times"></i>
                                    </button>
                                    <?php endif; ?>
                                </div>
                            </td>
                        </tr>
                        <?php endforeach; ?>
                    </tbody>
                </table>
            </div>
            
            <!-- Pagination -->
            <?php if ($total_pages > 1): ?>
            <nav aria-label="Page navigation">
                <ul class="pagination justify-content-center">
                    <?php for ($i = 1; $i <= $total_pages; $i++): ?>
                    <li class="page-item <?php echo ($i == $page) ? 'active' : ''; ?>">
                        <a class="page-link" href="?page=<?php echo $i; ?>&<?php echo http_build_query($_GET); ?>">
                            <?php echo $i; ?>
                        </a>
                    </li>
                    <?php endfor; ?>
                </ul>
            </nav>
            <?php endif; ?>
        </div>
    </div>
</div>

<!-- Cancel Order Modal -->
<div class="modal fade" id="cancelOrderModal" tabindex="-1">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title">Cancel Order</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <form id="cancelOrderForm">
                    <input type="hidden" id="cancelOrderId">
                    <div class="mb-3">
                        <label class="form-label">Alasan Pembatalan</label>
                        <textarea class="form-control" id="cancelReason" rows="3" required></textarea>
                    </div>
                </form>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Batal</button>
                <button type="button" class="btn btn-danger" onclick="confirmCancelOrder()">Cancel Order</button>
            </div>
        </div>
    </div>
</div>

<script>
// Status change handler
document.addEventListener('change', function(e) {
    if (e.target.classList.contains('status-select')) {
        const orderId = e.target.dataset.orderId;
        const newStatus = e.target.value;
        const currentStatus = e.target.dataset.currentStatus;
        
        if (newStatus !== currentStatus) {
            updateOrderStatus(orderId, newStatus, e.target);
        }
    }
});

function updateOrderStatus(orderId, status, selectElement) {
    fetch('/controllers/admin/OrderController.php', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            action: 'update_status',
            order_id: orderId,
            status: status
        })
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            selectElement.dataset.currentStatus = status;
            showAlert('Status pesanan berhasil diupdate', 'success');
        } else {
            // Revert selection
            selectElement.value = selectElement.dataset.currentStatus;
            showAlert(data.message || 'Gagal mengupdate status', 'danger');
        }
    })
    .catch(error => {
        console.error('Error:', error);
        selectElement.value = selectElement.dataset.currentStatus;
        showAlert('Terjadi kesalahan sistem', 'danger');
    });
}

function selectAll() {
    const checkboxes = document.querySelectorAll('.order-checkbox');
    const selectAllCheckbox = document.getElementById('selectAllCheckbox');
    
    checkboxes.forEach(checkbox => {
        checkbox.checked = !selectAllCheckbox.checked;
    });
    
    selectAllCheckbox.checked = !selectAllCheckbox.checked;
}

function bulkUpdateStatus(status) {
    const selectedOrders = Array.from(document.querySelectorAll('.order-checkbox:checked'))
                               .map(checkbox => checkbox.value);
    
    if (selectedOrders.length === 0) {
        showAlert('Pilih pesanan terlebih dahulu', 'warning');
        return;
    }
    
    if (!confirm(`Apakah Anda yakin ingin mengubah status ${selectedOrders.length} pesanan menjadi ${status}?`)) {
        return;
    }
    
    fetch('/controllers/admin/OrderController.php', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            action: 'bulk_update_status',
            order_ids: selectedOrders,
            status: status
        })
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            showAlert(data.message, 'success');
            setTimeout(() => {
                window.location.reload();
            }, 1000);
        } else {
            showAlert(data.message || 'Gagal mengupdate status', 'danger');
        }
    })
    .catch(error => {
        console.error('Error:', error);
        showAlert('Terjadi kesalahan sistem', 'danger');
    });
}

function cancelOrder(orderId) {
    document.getElementById('cancelOrderId').value = orderId;
    document.getElementById('cancelReason').value = '';
    
    const modal = new bootstrap.Modal(document.getElementById('cancelOrderModal'));
    modal.show();
}

function confirmCancelOrder() {
    const orderId = document.getElementById('cancelOrderId').value;
    const reason = document.getElementById('cancelReason').value;
    
    if (!reason.trim()) {
        showAlert('Alasan pembatalan harus diisi', 'warning');
        return;
    }
    
    fetch('/controllers/admin/OrderController.php', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            action: 'cancel_order',
            order_id: orderId,
            reason: reason
        })
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            showAlert('Pesanan berhasil dibatalkan', 'success');
            setTimeout(() => {
                window.location.reload();
            }, 1000);
        } else {
            showAlert(data.message || 'Gagal membatalkan pesanan', 'danger');
        }
    })
    .catch(error => {
        console.error('Error:', error);
        showAlert('Terjadi kesalahan sistem', 'danger');
    });
    
    const modal = bootstrap.Modal.getInstance(document.getElementById('cancelOrderModal'));
    modal.hide();
}

function printInvoice(orderId) {
    window.open(`/views/admin/print_invoice.php?id=${orderId}`, '_blank');
}

function exportOrders() {
    const params = new URLSearchParams(window.location.search);
    params.set('export', '1');
    
    window.location.href = '/controllers/admin/OrderController.php?' + params.toString();
}

function showAlert(message, type) {
    const alertContainer = document.createElement('div');
    alertContainer.className = `alert alert-${type} alert-dismissible fade show position-fixed`;
    alertContainer.style.cssText = 'top: 20px; right: 20px; z-index: 9999; min-width: 300px;';
    alertContainer.innerHTML = `
        ${message}
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    `;
    
    document.body.appendChild(alertContainer);
    
    setTimeout(() => {
        if (alertContainer.parentNode) {
            alertContainer.parentNode.removeChild(alertContainer);
        }
    }, 5000);
}
</script>

<?php 
function getPaymentBadgeClass($status) {
    switch ($status) {
        case 'pending': return 'warning';
        case 'paid': return 'success';
        case 'failed': return 'danger';
        case 'refunded': return 'info';
        default: return 'secondary';
    }
}

include '../layouts/admin_footer.php'; 
?>
```

### 4. Reporting System

**A. Reports Controller (controllers/admin/ReportsController.php)**
```php
<?php
require_once '../../config/database.php';
require_once '../../config/SessionManager.php';
require_once '../../middleware/AuthMiddleware.php';

class ReportsController {
    private $db;
    
    public function __construct() {
        $database = new Database();
        $this->db = $database->getConnection();
        
        SessionManager::start();
        AuthMiddleware::requireAdmin();
    }
    
    public function index() {
        $report_type = $_GET['type'] ?? 'sales';
        $date_from = $_GET['date_from'] ?? date('Y-m-01');
        $date_to = $_GET['date_to'] ?? date('Y-m-d');
        
        switch ($report_type) {
            case 'sales':
                $data = $this->getSalesReport($date_from, $date_to);
                break;
            case 'products':
                $data = $this->getProductReport($date_from, $date_to);
                break;
            case 'customers':
                $data = $this->getCustomerReport($date_from, $date_to);
                break;
            case 'inventory':
                $data = $this->getInventoryReport();
                break;
            default:
                $data = $this->getSalesReport($date_from, $date_to);
        }
        
        $pageTitle = 'Laporan';
        include '../../views/admin/reports.php';
    }
    
    private function getSalesReport($date_from, $date_to) {
        // Daily sales
        $query = "SELECT DATE(created_at) as date,
                         COUNT(*) as total_orders,
                         SUM(total_amount) as total_revenue,
                         AVG(total_amount) as avg_order_value
                 FROM orders 
                 WHERE DATE(created_at) BETWEEN :date_from AND :date_to
                 AND status != 'cancelled'
                 GROUP BY DATE(created_at)
                 ORDER BY date ASC";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindParam(':date_from', $date_from);
        $stmt->bindParam(':date_to', $date_to);
        $stmt->execute();
        $daily_sales = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        // Summary statistics
        $query = "SELECT COUNT(*) as total_orders,
                         SUM(total_amount) as total_revenue,
                         AVG(total_amount) as avg_order_value,
                         MIN(total_amount) as min_order,
                         MAX(total_amount) as max_order
                 FROM orders 
                 WHERE DATE(created_at) BETWEEN :date_from AND :date_to
                 AND status != 'cancelled'";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindParam(':date_from', $date_from);
        $stmt->bindParam(':date_to', $date_to);
        $stmt->execute();
        $summary = $stmt->fetch(PDO::FETCH_ASSOC);
        
        // Top selling products
        $query = "SELECT p.name, p.price,
                         SUM(oi.quantity) as total_sold,
                         SUM(oi.subtotal) as total_revenue
                 FROM order_items oi
                 JOIN products p ON oi.product_id = p.id
                 JOIN orders o ON oi.order_id = o.id
                 WHERE DATE(o.created_at) BETWEEN :date_from AND :date_to
                 AND o.status != 'cancelled'
                 GROUP BY p.id
                 ORDER BY total_sold DESC
                 LIMIT 10";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindParam(':date_from', $date_from);
        $stmt->bindParam(':date_to', $date_to);
        $stmt->execute();
        $top_products = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        return [
            'daily_sales' => $daily_sales,
            'summary' => $summary,
            'top_products' => $top_products
        ];
    }
    
    private function getProductReport($date_from, $date_to) {
        // Product performance
        $query = "SELECT p.id, p.name, p.price, p.stock,
                         COALESCE(SUM(oi.quantity), 0) as total_sold,
                         COALESCE(SUM(oi.subtotal), 0) as total_revenue,
                         p.stock + COALESCE(SUM(oi.quantity), 0) as initial_stock
                 FROM products p
                 LEFT JOIN order_items oi ON p.id = oi.product_id
                 LEFT JOIN orders o ON oi.order_id = o.id
                 AND DATE(o.created_at) BETWEEN :date_from AND :date_to
                 AND o.status != 'cancelled'
                 GROUP BY p.id
                 ORDER BY total_sold DESC";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindParam(':date_from', $date_from);
        $stmt->bindParam(':date_to', $date_to);
        $stmt->execute();
        $products = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        // Low stock products
        $query = "SELECT * FROM products WHERE stock <= 10 ORDER BY stock ASC";
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        $low_stock = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        return [
            'products' => $products,
            'low_stock' => $low_stock
        ];
    }
    
    private function getCustomerReport($date_from, $date_to) {
        // Customer statistics
        $query = "SELECT u.id, u.name, u.email,
                         COUNT(o.id) as total_orders,
                         SUM(o.total_amount) as total_spent,
                         AVG(o.total_amount) as avg_order_value,
                         MAX(o.created_at) as last_order_date
                 FROM users u
                 LEFT JOIN orders o ON u.id = o.user_id
                 AND DATE(o.created_at) BETWEEN :date_from AND :date_to
                 AND o.status != 'cancelled'
                 WHERE u.role = 'user'
                 GROUP BY u.id
                 HAVING total_orders > 0
                 ORDER BY total_spent DESC";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindParam(':date_from', $date_from);
        $stmt->bindParam(':date_to', $date_to);
        $stmt->execute();
        $customers = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        // New customers
        $query = "SELECT COUNT(*) as new_customers
                 FROM users 
                 WHERE role = 'user'
                 AND DATE(created_at) BETWEEN :date_from AND :date_to";
        
        $stmt = $this->db->prepare($query);
        $stmt->bindParam(':date_from', $date_from);
        $stmt->bindParam(':date_to', $date_to);
        $stmt->execute();
        $new_customers = $stmt->fetch(PDO::FETCH_ASSOC)['new_customers'];
        
        return [
            'customers' => $customers,
            'new_customers' => $new_customers
        ];
    }
    
    private function getInventoryReport() {
        // Current inventory status
        $query = "SELECT p.*, c.name as category_name,
                         COALESCE(SUM(oi.quantity), 0) as total_sold_all_time
                 FROM products p
                 LEFT JOIN categories c ON p.category_id = c.id
                 LEFT JOIN order_items oi ON p.id = oi.product_id
                 LEFT JOIN orders o ON oi.order_id = o.id
                 AND o.status != 'cancelled'
                 GROUP BY p.id
                 ORDER BY p.stock ASC";
        
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        $inventory = $stmt->fetchAll(PDO::FETCH_ASSOC);
        
        // Inventory summary
        $query = "SELECT 
                    COUNT(*) as total_products,
                    SUM(stock) as total_stock,
                    SUM(stock * price) as total_inventory_value,
                    COUNT(CASE WHEN stock = 0 THEN 1 END) as out_of_stock,
                    COUNT(CASE WHEN stock <= 10 THEN 1 END) as low_stock
                 FROM products";
        
        $stmt = $this->db->prepare($query);
        $stmt->execute();
        $summary = $stmt->fetch(PDO::FETCH_ASSOC);
        
        return [
            'inventory' => $inventory,
            'summary' => $summary
        ];
    }
    
    public function export() {
        $type = $_GET['type'] ?? 'sales';
        $format = $_GET['format'] ?? 'csv';
        $date_from = $_GET['date_from'] ?? date('Y-m-01');
        $date_to = $_GET['date_to'] ?? date('Y-m-d');
        
        switch ($type) {
            case 'sales':
                $data = $this->getSalesReport($date_from, $date_to);
                $this->exportSalesData($data, $format);
                break;
            case 'products':
                $data = $this->getProductReport($date_from, $date_to);
                $this->exportProductData($data, $format);
                break;
            case 'customers':
                $data = $this->getCustomerReport($date_from, $date_to);
                $this->exportCustomerData($data, $format);
                break;
        }
    }
    
    private function exportSalesData($data, $format) {
        if ($format === 'csv') {
            header('Content-Type: text/csv');
            header('Content-Disposition: attachment; filename="sales_report_' . date('Y-m-d') . '.csv"');
            
            $output = fopen('php://output', 'w');
            
            // Headers
            fputcsv($output, ['Tanggal', 'Total Pesanan', 'Total Revenue', 'Rata-rata Order']);
            
            // Data
            foreach ($data['daily_sales'] as $row) {
                fputcsv($output, [
                    $row['date'],
                    $row['total_orders'],
                    $row['total_revenue'],
                    $row['avg_order_value']
                ]);
            }
            
            fclose($output);
        }
    }
}

// Handle request
if (isset($_GET['export'])) {
    $controller = new ReportsController();
    $controller->export();
} else {
    $controller = new ReportsController();
    $controller->index();
}
?>
```

## Praktikum Hands-On

### Langkah 1: Setup Admin Layout
1. **Buat Admin Header dan Footer**
   - Implementasikan `views/layouts/admin_header.php`
   - Implementasikan `views/layouts/admin_footer.php`
   - Tambahkan Bootstrap 5 dan Chart.js

2. **Setup Admin Middleware**
   - Pastikan `AuthMiddleware::requireAdmin()` berfungsi
   - Test akses admin panel

### Langkah 2: Implementasi Dashboard
1. **Buat Dashboard Controller**
   - Implementasikan `DashboardController.php`
   - Test semua method statistik

2. **Buat Dashboard View**
   - Implementasikan dashboard dengan cards statistik
   - Tambahkan Chart.js untuk visualisasi
   - Test responsivitas

### Langkah 3: Product Management
1. **Implementasi CRUD Produk**
   - Create, Read, Update, Delete produk
   - Upload dan manajemen gambar
   - Bulk actions

2. **Testing Product Management**
   - Test semua fitur CRUD
   - Test upload gambar
   - Test filter dan search

### Langkah 4: Order Management
1. **Implementasi Order Management**
   - List orders dengan filter
   - Update status pesanan
   - Bulk actions

2. **Testing Order Management**
   - Test update status
   - Test filter dan pagination
   - Test bulk operations

### Langkah 5: Reporting System
1. **Implementasi Reports**
   - Sales report dengan chart
   - Product performance report
   - Customer analytics
   - Inventory report

2. **Testing Reports**
   - Test semua jenis laporan
   - Test export functionality
   - Test date filtering

## Tugas Individu

### Tugas 1: Enhanced Dashboard (Bobot: 25%)
**Objektif:** Meningkatkan dashboard dengan fitur advanced

**Requirements:**
1. **Real-time Updates**
   - Implementasikan auto-refresh setiap 30 detik
   - Tambahkan WebSocket untuk real-time notifications
   - Buat indicator untuk data yang sedang loading

2. **Advanced Charts**
   - Tambahkan comparison chart (bulan ini vs bulan lalu)
   - Implementasikan drill-down functionality
   - Buat interactive tooltips

3. **Quick Actions Widget**
   - Tambahkan shortcut untuk aksi cepat
   - Implementasikan drag & drop untuk reorder widgets
   - Buat customizable dashboard layout

**Deliverables:**
- Enhanced dashboard dengan real-time features
- Documentation untuk fitur baru
- Screenshot/video demo

### Tugas 2: Advanced Product Management (Bobot: 25%)
**Objektif:** Mengembangkan fitur product management yang lebih canggih

**Requirements:**
1. **Bulk Import/Export**
   - Implementasikan import produk dari CSV/Excel
   - Buat template untuk bulk import
   - Tambahkan validation dan error handling

2. **Product Variants**
   - Implementasikan sistem variant (size, color, dll)
   - Buat interface untuk manage variants
   - Update inventory tracking untuk variants

3. **Advanced Filtering**
   - Implementasikan advanced search dengan multiple criteria
   - Tambahkan saved filters
   - Buat export filtered results

**Deliverables:**
- Enhanced product management system
- Import/export functionality
- Documentation dan user guide

### Tugas 3: Comprehensive Reporting (Bobot: 25%)
**Objektif:** Membuat sistem reporting yang komprehensif

**Requirements:**
1. **Advanced Analytics**
   - Implementasikan cohort analysis
   - Buat customer lifetime value calculation
   - Tambahkan trend analysis

2. **Custom Reports**
   - Buat report builder interface
   - Implementasikan scheduled reports
   - Tambahkan email delivery untuk reports

3. **Data Visualization**
   - Implementasikan berbagai jenis chart
   - Buat interactive dashboards
   - Tambahkan export ke PDF

**Deliverables:**
- Advanced reporting system
- Custom report builder
- Automated report scheduling

### Tugas 4: System Administration (Bobot: 25%)
**Objektif:** Implementasi fitur administrasi sistem

**Requirements:**
1. **User Management**
   - Implementasikan CRUD untuk users
   - Buat role-based permissions
   - Tambahkan user activity logging

2. **System Settings**
   - Buat interface untuk system configuration
   - Implementasikan backup/restore functionality
   - Tambahkan system health monitoring

3. **Security Features**
   - Implementasikan two-factor authentication
   - Buat audit trail untuk admin actions
   - Tambahkan IP whitelist/blacklist

**Deliverables:**
- Complete admin panel
- Security implementation
- System monitoring tools

## Kriteria Evaluasi

### Aspek Teknis (60%)
- **Functionality (20%):** Semua fitur berfungsi sesuai requirements
- **Code Quality (20%):** Clean code, proper structure, documentation
- **Security (10%):** Implementasi security best practices
- **Performance (10%):** Optimasi query dan loading time

### Aspek UI/UX (25%)
- **Design (15%):** Interface yang menarik dan professional
- **Usability (10%):** Kemudahan penggunaan dan navigasi

### Aspek Dokumentasi (15%)
- **Technical Documentation (10%):** API docs, code comments
- **User Documentation (5%):** User guide dan tutorial

## Sumber Daya Tambahan

### Dokumentasi
- [Chart.js Documentation](https://www.chartjs.org/docs/)
- [Bootstrap 5 Admin Templates](https://getbootstrap.com/docs/5.0/examples/)
- [PHP Security Best Practices](https://www.php.net/manual/en/security.php)

### Tools dan Libraries
- **Chart.js:** Untuk visualisasi data
- **DataTables:** Untuk advanced table features
- **Select2:** Untuk enhanced select boxes
- **Moment.js:** Untuk date manipulation

### Best Practices

#### Security
- Selalu validasi dan sanitasi input
- Implementasikan CSRF protection
- Gunakan prepared statements
- Implement rate limiting

#### Performance
- Optimasi database queries
- Implementasikan caching
- Gunakan pagination untuk large datasets
- Optimize image loading

#### Code Organization
- Gunakan MVC pattern secara konsisten
- Implementasikan proper error handling
- Buat reusable components
- Follow PSR standards

---

## Navigasi
- [← Bab 5: E-Commerce Features dan Business Logic](bab5-ecommerce-logic.md)
- [→ Kembali ke Daftar Isi](../README.md)
- [→ Final Project](../PROJECT_FINAL.md)
- [→ Resources](../RESOURCES.md)

---

**Catatan:** Bab ini merupakan culmination dari semua pembelajaran sebelumnya. Pastikan untuk mengintegrasikan semua konsep yang telah dipelajari dan fokus pada pembuatan admin panel yang professional dan fungsional.