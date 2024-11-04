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
header("Access-Control-Allow-Headers: Content-Type, Access-Control-Allow-Headers, typeization, X-Requested-With");

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

function validatemedicines($name, $type, $price, $stock) {
    $errors = [];
    if (empty($name)) {
        $errors[] = "Name is required";
    }
    if (empty($type)) {
        $errors[] = "Type is required";
    }
    if (empty($price)) {
        $errors[] = "Price is required";
    }
    if (!is_numeric($stock) || strlen($stock)) {
        $errors[] = "Stock must be a number";
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
                $stmt = $db->prepare("SELECT * FROM medicines WHERE name LIKE ? OR type LIKE ?");
                $searchTerm = "%$searchTerm%";
                $stmt->execute([$searchTerm, $searchTerm]);
                $medicines = $stmt->fetchAll();
                response(200, $medicines);
            } else {
                // Get specific medicines
                $id = $request[0];
                $stmt = $db->prepare("SELECT * FROM medicines WHERE id = ?");
                $stmt->execute([$id]);
                $medicines = $stmt->fetch();
                if ($medicines) {
                    response(200, $medicines);
                } else {
                    response(404, ["message" => "medicines not found"]);
                }
            }
        } else {
            // 5.2 Pagination
            $page = isset($_GET['page']) ? (int)$_GET['page'] : 1;
            $limit = isset($_GET['limit']) ? (int)$_GET['limit'] : 10;
            $offset = ($page - 1) * $limit;

            $stmt = $db->prepare("SELECT * FROM medicines LIMIT ? OFFSET ?");
            $stmt->bindValue(1, $limit, PDO::PARAM_INT);
            $stmt->bindValue(2, $offset, PDO::PARAM_INT);
            $stmt->execute();
            $medicines = $stmt->fetchAll();

            $totalStmt = $db->query("SELECT COUNT(*) FROM medicines");
            $total = $totalStmt->fetchColumn();

            response(200, [
                'medicines' => $medicines,
                'total' => $total,
                'page' => $page,
                'limit' => $limit
            ]);
        }
        break;
    
    case 'POST':
        $data = json_decode(file_get_contents("php://input"));
        $errors = validatemedicines($data->name ?? '', $data->type ?? '', $data->price ?? '', $data->stock ?? '');
        if (!empty($errors)) {
            response(400, ["errors" => $errors]);
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
        $errors = validatemedicines($data->name ?? '', $data->type ?? '', $data->price ?? '', $data->stock ?? '');
        if (!empty($errors)) {
            response(404, ["errors" => $errors]);
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
   a. menampilkan semua data
      - menggunakan method GET
      - dengan URL : `http://localhost/uts_medicine/medicines_api.php/`
      - lalu send
      - hasilnya :
        ```json
        {
          "medicines": [
        {
            "id": 1,
            "name": "Splasminal",
            "type": "Tablet",
            "price": "Rp 12000",
            "stock": 10
        },
        {
            "id": 2,
            "name": "Insto",
            "type": "Obat Tetes",
            "price": "Rp 17000",
            "stock": 6
        },
        {
            "id": 3,
            "name": "Polysilane",
            "type": "Syrup",
            "price": "Rp 20500",
            "stock": 9
        },
        {
            "id": 5,
            "name": "Betadine",
            "type": "Antiseptik",
            "price": "Rp 18000",
            "stock": 10
        },
        {
            "id": 6,
            "name": "OBH Combi Plus Mentol",
            "type": "Syrup",
            "price": "Rp 17500",
            "stock": 12
        }
          ],
          "total": 5,
          "page": 1,
          "limit": 10
         }
         ```
   b. mencari data dengan value type,
      - menggunakan method GET
      - dengan URL : `http://localhost/uts_medicine/medicines_api.php/search?&term=syrup`
      - lalu send
      - hasilnya:
        ```json
        [
          {
              "id": 3,
              "name": "Polysilane",
              "type": "Syrup",
              "price": "Rp 20500",
              "stock": 9
          },
          {
              "id": 6,
              "name": "OBH Combi Plus Mentol",
              "type": "Syrup",
              "price": "Rp 17500",
              "stock": 12
          }
        ]
        ```
   #### GET `/api/[object]/{id}`
   a. menampilkan data berdasarkan id,
      - menggunakan method GET
      - dengan URL : `http://localhost/uts_medicine/medicines_api.php/3`
      - lalu send
      - hasilnya :
        ```json
        {
          "id": 3,
          "name": "Polysilane",
          "type": "Syrup",
          "price": "Rp 20500",
          "stock": 9
        }
        ```
   b. response 404 jika data tidak ditemukan,
      - menggunakan method GET
      - dengan URL : `http://localhost/uts_medicine/medicines_api.php/15`
      - lalu send
      - hasilnya :
        ```json
        {
          "message": "medicines not found"
        }
        ```
   #### POST `/api/[object]`
   a. menambah data baru,
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
         ```json
         {
          "message": "Medicine created",
          "id": "7"
         }
         ```
   b. validasi input
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
   c. response 201 maka validasi input berhasil, hasilnya :
      ```json
      {
        "message": "Medicine created",
        "id": "9"
      }
      ```
   #### PUT `/api/[objek]/{id}`
   a. mengupdate data berdasarkan ID
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
      - hasilnya :
        ```json
        {
           "message": "Medicine updated"
        }
        ```
   b. validasi input
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
   c. Response 404 karena stock hanya bisa menggunakan angka, hasilnyanya :
      ```json
      {
        "errors": [
        "Stock must be a number"
        ]
      }
      ```
   #### DELETE `/api/[objek]/{id}`
   a. menghapus data berdasarkan ID
      - menggunakan method DELETE
      - dengan URL `http://localhost/uts_medicine/medicines_api.php/7`
      - lalu send
      - hasilnya :
        ```json
        {
          "message": "Medicine deleted"
        }
        ```
   b. Response 404 jika data tidak ditemukan
