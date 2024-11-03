# medicines_UTS

### Nama : Ranti Nurmala Sari
### NIM : 21.01.55.0016

Membuat web service untuk manajemen medicine menggunakan PHP dan MySQL dan menguji dengan menggunakan postman.
saya membuat database bernama medicines_db pada phpmyadmin dengan tabel yang bernama medicines berisikan id, name, type, price, stock

Dalam membuat web service ini saya menggunakan XAMPP (https://www.apachefriends.org/download.html),<br>
text editor visual studio code (https://code.visualstudio.com/download),<br>
dan postman untuk menguji API (https://code.visualstudio.com/download).

### Persiapan
1. Buka XAMPP dan hidupkan Apache dan MySQL
2. Buat folder baru bernama `uts_medicine` di dalam direktori `htdocs` XAMPP 

### Membuat Database
1. buka PhpMyAdmin pada brower (http://localhost/phpmyadmin/)
2. buat database baru bernama `medicines_db` atau buat dengan SQL
   ```sql
   CREATE DATABASE medicines_db;
   ```
4. pada database `medicines_db`, buka tab SQL dan jalankan query untuk membuat tabel dan menambahkan data
  ```sql
  CREATE TABLE medicines (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(100) NOT NULL,
    price VARCHAR(50) NOT NULL,
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
### Pengujian dengan postman
1. buka postman
2. implementasi endpoint API
   #### GET `/api/[object]`
   1. menampilkan semua data
      - menggunakan method GET
      - dengan URL : `http://localhost/uts_medicine/medicines_api.php/`
      - lalu send
   2. mencari data dengan value type,
      - menggunakan method GET
      - dengan URL : `http://localhost/uts_medicine/medicines_api.php/search?&term=syrup`
      - lalu send
   #### GET `/api/[object]/{id}`
   1. menampilkan data berdasarkan id,
      - menggunakan method GET
      - dengan URL : `http://localhost/uts_medicine/medicines_api.php/3`
      - lalu send
   2. response 404 jika data tidak ditemukan,
      - menggunakan method GET
      - dengan URL : `http://localhost/uts_medicine/medicines_api.php/9`
      - lalu send
      (data sampel saya hanya berisi 6 data)
   #### POST `/api/[object]`
   1. menambah data baru,
      - menggunakan method POST
      - dengan URL `http://localhost/uts_medicine/medicines_api.php`
      - pada Headers
        - key: Content-Type
        - value: application/json
      - pada Body
        - pilih "raw" dan "JSON"
        - masukkan:
          ```json
          {
            "name": "OBH Combi",
            "type": "Syrup",
            "price": "Rp 16500",
            "stock": 8
          }
          ```
       - lalu send
   2. validasi input
      - menggunakan method POST
      - dengan URL `http://localhost/uts_medicine/medicines_api.php`
      - pada Headers
        - key : Content-Type
        - value : application/json
      - pada Body
        - pilih "raw" dan "JSON"
        - masukkan :
          ```json
          {
            "name": "Panadol",
            "type": "Tablet",
            "price": "Rp 10000",
            "stock": 15
          }
      - lalu send
   3. response 201 maka validasi input berhasil, hasilnya :
      ```json
      {
        "message": "Medicine created",
        "id": "9"
      }
      ```
   #### PUT `/api/[objek]/{id}`
   1. mengupdate data berdasarkan ID
      - menggunakan method PUT
      - dengan URL `http://localhost/uts_medicine/medicines_api.php/9`
      - pada Headers
        - key : Content-Type
        - value : application/json
      - pada Body
        - pilih "raw" dan "JSON"
        - masukkan :
          ```json
          {        
            "name": "Panadol Biru",
            "type": "Tablet",
            "price": "Rp 10000",
            "stock": 15
          }
      - lalu send
   2. validasi input
      - menggunakan method PUT
      - dengan URL `http://localhost/uts_medicine/medicines_api.php/2`
      - pada Headers
        - key : Content-Type
        - value : application/json
      - pada Body
        - pilih "raw" dan "JSON"
        - masukkan :
          ```json
          {        
            "name": "Insto",
            "type": "Obat Tetes",
            "price": "Rp 17000",
            "stock": "lima"
          }
      - lalu send
   3. Response 404 karena stock hanya bisa menggunakan angka, hasilnyanya :
      ```json
      {
        "errors": [
        "Stock must be a number"
        ]
      }
      ```
   #### DELETE `/api/[objek]/{id}`
   1. menghapus data berdasarkan ID
      - menggunakan method DELETE
      - dengan URL `http://localhost/uts_medicine/medicines_api.php/7`
      - lalu send
   3. Response 404 jika data tidak ditemukan
