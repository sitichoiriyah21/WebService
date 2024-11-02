# WebService
Ini adalah tugas uts mata kuliah Web Service


#### Nama: Siti Choiriyah
#### NIM: 21.01.55.0019
#### Kuliner 
#### Tabel: menu makanan khas Purwodadi
- id (PK)
- name
- category
- price
- portion

## LANGKAH-LANGKAH

### 1. Persiapan Lingkungan
1. Buka XAMPP lalu klik start pada Apache dan mySQL
2. Buat folder baru bernama `rest_menus` di dalam direktori `htdocs` XAMPP

### 2. Membuat Database
1. Buka php My Admin (http://localhost/phpmyadmin)
2. Buat database baru bernama `culinary`
3. Pilih database `culinary`, lalu buka tab SQL
4. Jalankan query SQL berikut untuk membuat tabel dan menambahkan data sampel:

```sql
      CREATE TABLE menus (
      id INT AUTO_INCREMENT PRIMARY KEY,
      name VARCHAR(100) NOT NULL UNIQUE,
      category VARCHAR(50) NOT NULL,
      price DECIMAL(10,2) NOT NULL,
      porsi INT NOT NULL CHECK,
      INDEX idx_category (category) 
      );

      INSERT INTO menus(name, category, price, porsi) VALUES
      ('Garang Asem', 'makanan', '15000', '1'),
      ('Es potong', 'minuman', '5000', '1'),
      ('Sale Pisang', 'cemilan', '5000', '200');
```

### 3. Membuat file PHP untuk Web Service
1. Buka vs code
2. Buat file baru dan simpan sebagai `menus_api.php` di dalam folder `rest_menus`.
3. Salin dan tempel kode berikut dalam `menus_api.php`:

```php
<?php
header("Content-Type: application/json; charset=UTF-8");
header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Methods: GET, POST, PUT, DELETE");
header("Access-Control-Allow-Headers: Content-Type, Access-Control-Allow-Headers, Authorization, X-Requested-With");

$method = $_SERVER['REQUEST_METHOD'];
$request = [];

if (isset($_SERVER['PATH_INFO'])) {
    $request = explode('/', trim($_SERVER['PATH_INFO'],'/'));
}

function getConnection() {
    $host = 'localhost';
    $db   = 'culinary';
    $user = 'root';
    $pass = ''; // Ganti dengan password MySQL Anda jika ada
    $charset = 'utf8mb4';

    $dsn = "mysql:host=$host;dbname=$db;charset=$charset";
    $options = [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        PDO::ATTR_EMULATE_PREPARES   => false,
    ];
    try {
        return new PDO($dsn, $user, $pass, $options);
    } catch (\PDOException $e) {
        throw new \PDOException($e->getMessage(), (int)$e->getCode());
    }
}

function response($status, $data = NULL) {
    header("HTTP/1.1 " . $status);
    if ($data) {
        echo json_encode($data);
    }
    exit();
}

function validateBook($name, $category, $price, $porsi) {
    $errors = [];
    if (empty($name)) {
        $errors[] = "Name is required";
    }
    if (empty($category)) {
        $errors[] = "category is required";
    }
    if (empty($price)) {
        $errors[] = "price is required";
    }
    if (empty($porsi)) {
        $errors[] = "porsi is required";
    }
    return $errors;
}

$db = getConnection();

switch ($method) {
    case 'GET':
        if (!empty($request) && isset($request[0])) {
            if ($request[0] === 'search') {
                // 5.1 Search functionality
                $searchTerm = $_GET['term'] ?? '';
                $stmt = $db->prepare("SELECT * FROM menus WHERE name LIKE ? OR category LIKE ?");
                $searchTerm = "%$searchTerm%";
                $stmt->execute([$searchTerm, $searchTerm]);
                $menus = $stmt->fetchAll();
                response(200, $menus);
            } else {
                // Get specific book
                $id = $request[0];
                $stmt = $db->prepare("SELECT * FROM menus WHERE id = ?");
                $stmt->execute([$id]);
                $book = $stmt->fetch();
                if ($book) {
                    response(200, $book);
                } else {
                    response(404, ["message" => "menus not found"]);
                }
            }
        } else {
            // 5.2 Pagination
            $page = isset($_GET['page']) ? (int)$_GET['page'] : 1;
            $limit = isset($_GET['limit']) ? (int)$_GET['limit'] : 10;
            $offset = ($page - 1) * $limit;

            $stmt = $db->prepare("SELECT * FROM menus LIMIT ? OFFSET ?");
            $stmt->bindValue(1, $limit, PDO::PARAM_INT);
            $stmt->bindValue(2, $offset, PDO::PARAM_INT);
            $stmt->execute();
            $menus = $stmt->fetchAll();

            $totalStmt = $db->query("SELECT COUNT(*) FROM menus");
            $total = $totalStmt->fetchColumn();

            response(200, [
                'menus' => $menus,
                'total' => $total,
                'page' => $page,
                'limit' => $limit
            ]);
        }
        break;
    
    case 'POST':
        $data = json_decode(file_get_contents("php://input"));
        // 5.3 Stricter input validation
        $errors = validateBook($data->name ?? '', $data->category ?? '', $data->price ?? '', $data->porsi ?? '');
        if (!empty($errors)) {
            response(400, ["errors" => $errors]);
        }
        $sql = "INSERT INTO menus (name, category, price, porsi) VALUES (?, ?, ?, ?)";
        $stmt = $db->prepare($sql);
        if ($stmt->execute([$data->name, $data->category, $data->price, $data->porsi])) {
            response(201, ["message" => "menus created", "id" => $db->lastInsertId()]);
        } else {
            response(500, ["message" => "Failed to create menus"]);
        }
        break;
    
    case 'PUT':
        if (empty($request) || !isset($request[0])) {
            response(400, ["message" => "menus ID is required"]);
        }
        $id = $request[0];
        $data = json_decode(file_get_contents("php://input"));
        // 5.3 Stricter input validation
        $errors = validateBook($data->name ?? '', $data->category ?? '', $data->price ?? '', $data->porsi ?? '');
        if (!empty($errors)) {
            response(400, ["errors" => $errors]);
        }
        $sql = "UPDATE menus SET name = ?, category = ?, price = ?, porsi = ? WHERE id = ?";
        $stmt = $db->prepare($sql);
        if ($stmt->execute([$data->name, $data->category, $data->price, $data->porsi, $id])) {
            response(200, ["message" => "menus updated"]);
        } else {
            response(500, ["message" => "Failed to update menus"]);
        }
        break;
    
    case 'DELETE':
        if (empty($request) || !isset($request[0])) {
            response(400, ["message" => "menus ID is required"]);
        }
        $id = $request[0];
        $sql = "DELETE FROM menus WHERE id = ?";
        $stmt = $db->prepare($sql);
        if ($stmt->execute([$id])) {
            response(200, ["message" => "menus deleted"]);
        } else {
            response(500, ["message" => "Failed to delete menus"]);
        }
        break;
    
    default:
        response(405, ["message" => "Method not allowed"]);
        break;
}
?>
```
      

### 4. Pengujian dengan Postman
#### GET `/api/[objek]`
a. Menampilkan detail data berdasarkan ID
- Method: GET
- URL: `http://localhost/rest_menus/menus_api.php/3`
- Klik "Send"

![1](https://github.com/user-attachments/assets/1413a8d3-da40-4225-abb1-cafda121bdd7)

b. Response 404 jika data tidak ditemukan

#### GET `/api/[objek]/{id}`
a. Menampilkan semua data
- Method: GET
- URL: `http://localhost/rest_menus/menus_api.php`
- Klik "Send"

b. Pencarian berdasarkan nama
- Method: GET
- URL: `http://localhost/rest_menus/menus_api.php/search?term= es potong`
- Klik "Send"

#### POST `/api/[objek]`
a. Menambah data baru
- Method: POST
- URL: `http://localhost/rest_menus/menus_api.php`
- Headers: 
  - Key: Content-Type
  - Value: application/json
- Body:
  - Pilih "raw" dan "JSON"
  - Masukkan:
    ```json
    {
        "name": "Getuk",
        "category": "cemilan",
        "price": "5000.00",
        "porsi": "5"
    }
    ```
- Klik "Send"

b. Validasi Input
- Method: POST
- URL: `http://localhost/rest_menus/menus_api.php`
- Headers: 
  - Key: Content-Type
  - Value: application/json
- Body:
  - Pilih "raw" dan "JSON"
  - Masukkan:
    ```json
    {
        "name": "Getuk",
        "category": "",
        "price": "5000.00",
        "porsi": "5"
    }
    ```
- Klik "Send"
- akan menghasilkan eror karena category tidak valid/tidak diisi

#### PUT `/api/[objek]/{id}`
a. Mengupdate data berdasarkan ID
- Method: PUT
- URL: `http://localhost/rest_menus/menus_api.php/4` (asumsikan ID menus baru adalah 4)
- Headers: 
  - Key: Content-Type
  - Value: application/json
- Body:
  - Pilih "raw" dan "JSON"
  - Masukkan:
    ```json
    {
        "name": "Es Kul-kul",
        "category": "Minuman",
        "price": "5000.00",
        "porsi": "1"
    }
    ```
- Klik "Send"

b. Validasi input
c. Response 404 jika data tidak ditemukan

#### DELETE `/api/[objek]/{id}`
a. Menghapus data berdasarkan ID
- Method: DELETE
- URL: `http://localhost/rest_menus/menus_api.php/4` (untuk menghapus buku dengan ID 4)
- Klik "Send"


b. response 404 jika data tidak ditemukan
