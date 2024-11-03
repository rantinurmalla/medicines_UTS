# medicines_UTS

### Nama : Ranti Nurmala Sari
### NIM : 21.01.55.0016

Dalam membuat web service ini saya menggunakan XAMPP (https://www.apachefriends.org/download.html),<br>
text editor visual studio code (https://code.visualstudio.com/download),<br>
dan postman untuk menguji API (https://code.visualstudio.com/download).

### Persiapan
1. Buka XAMPP dan hidupkan Apache dan MySQL
2. Buat folder baru bernama `uts_medicine` di dalam direktori `htdocs` XAMPP 

### Membuat Database
1. buka PhpMyAdmin pada brower (http://localhost/phpmyadmin/)
2. buat database baru bernama `medicines_db` atau buat dengan SQL ```sql CREATE DATABASE medicines_db;```
3. pada database `medicines_db`, buka tab SQL dan jalankan query untuk membuat tabel dan menambahkan data
  ```sql
  CREATE TABLE medicines (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(255) NOT NULL,
    price VARCHAR(255) NOT NULL,
    stock INT(11) NOT NULL
  );
  
  INSERT INTO medicines (name, type, price, stock) VALUES
    ('Splasminal', 'Tablet', 'Rp 12000', '10'),
    ('Insto', 'Obat Tetes', 'Rp 17000' , '6'),
    ('Polysilane', 'Syrup', 'Rp 20500' , '9'),
    ('Panadol','Tablet','Rp 11000','12'),
    ('Betadine','Antiseptik','Rp 18000','10');
  ```
### Membuat file PHP untuk Web Service
1. buka text editor kesayangan
2. buat file baru bernama `medicines_api.php`di dalam folder `uts_medicine`
3. salin dan simpan kode berikut ke dalam `medicine_api.php` :
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
    $db   = 'medicines_db';
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

$db = getConnection();

switch ($method) {
    case 'GET':
        if (!empty($request) && isset($request[0])) {
            $id = $request[0];
            $stmt = $db->prepare("SELECT * FROM medicines WHERE id = ?");
            $stmt->execute([$id]);
            $Medicine = $stmt->fetch();
            if ($Medicine) {
                response(200, $Medicine);
            } else {
                response(404, ["message" => "Medicine not found"]);
            }
        } else {
            $stmt = $db->query("SELECT * FROM medicines");
            $medicines = $stmt->fetchAll();
            response(200, $medicines);
        }
        break;
    
    case 'POST':
        $data = json_decode(file_get_contents("php://input"));
        if (!isset($data->name) || !isset($data->type) || !isset($data->price) || !isset($data->stock)) {
            response(400, ["message" => "Missing required fields"]);
        }
        $sql = "INSERT INTO medicines (name, type, price, stock) VALUES (?, ?, ?, ?)";
        $stmt = $db->prepare($sql);
        if ($stmt->execute([$data->name, $data->type, $data->price, $data->stock])) {
            response(201, ["message" => "Medicine created", "id" => $db->lastInsertId()]);
        } else {
            response(500, ["message" => "Failed to create Medicine"]);
        }
        break;
    
    case 'PUT':
        if (empty($request) || !isset($request[0])) {
            response(400, ["message" => "Medicine ID is required"]);
        }
        $id = $request[0];
        $data = json_decode(file_get_contents("php://input"));
        if (!isset($data->name) || !isset($data->type) || !isset($data->price) || !isset($data->stock)) {
            response(400, ["message" => "Missing required fields"]);
        }
        $sql = "UPDATE medicines SET name = ?, type = ?, price = ?, stock = ? WHERE id = ?";
        $stmt = $db->prepare($sql);
        if ($stmt->execute([$data->name, $data->type, $data->price, $data->stock, $id])) {
            response(200, ["message" => "Medicine updated"]);
        } else {
            response(500, ["message" => "Failed to update Medicine"]);
        }
        break;
    
    case 'DELETE':
        if (empty($request) || !isset($request[0])) {
            response(400, ["message" => "Medicine ID is required"]);
        }
        $id = $request[0];
        $sql = "DELETE FROM medicines WHERE id = ?";
        $stmt = $db->prepare($sql);
        if ($stmt->execute([$id])) {
            response(200, ["message" => "Medicine deleted"]);
        } else {
            response(500, ["message" => "Failed to delete Medicine"]);
        }
        break;
    
    default:
        response(405, ["message" => "Method not allowed"]);
        break;
}
?>
```
