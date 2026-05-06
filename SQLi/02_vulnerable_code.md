# 02. SQL Injection 공격 유형별 취약 코드 분석

## 1. 개요

이 문서는 SQL Injection(SQLi)의 주요 공격 유형을 기준으로 취약한 코드 구조와 대응 코드를 비교한다.

SQL Injection은 사용자 입력값이 SQL Query에 안전하지 않은 방식으로 포함될 때 발생할 수 있다.  
핵심 문제는 사용자의 입력값이 단순한 데이터가 아니라 SQL 문법의 일부로 해석될 수 있다는 점이다.

이 문서에서는 다음 흐름으로 정리한다.

```text
공격 유형 설명
   ↓
취약한 코드 예제
   ↓
공격 입력이 Query를 어떻게 바꾸는지 분석
   ↓
Prepared Statement 기반 대응 코드
```

---

## 2. 기본 실습 환경

예시는 PHP와 MySQL/MariaDB 환경을 기준으로 한다.

```text
OS: Windows
Environment: XAMPP
Web Server: Apache
Language: PHP
Database: MySQL / MariaDB
```

예시 테이블:

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(100) NOT NULL,
    email VARCHAR(100)
);

INSERT INTO users (username, password, email) VALUES
('admin', 'admin1234', 'admin@example.com'),
('guest', 'guest1234', 'guest@example.com');
```

예시 DB 연결 파일:

```php
<?php
$host = "localhost";
$user = "root";
$password = "";
$dbname = "sqli_lab";

$conn = mysqli_connect($host, $user, $password, $dbname);

if (!$conn) {
    die("Database connection failed");
}
?>
```

---

# 3. 인증 우회 SQL Injection

## 3.1 공격 유형 설명

인증 우회 SQL Injection은 로그인 기능에서 자주 설명되는 유형이다.

로그인 로직이 사용자의 `username`, `password` 값을 SQL Query 문자열에 직접 삽입하면, 공격자는 조건식을 조작해 인증 결과에 영향을 줄 수 있다.

취약한 로그인 Query는 보통 다음과 같은 형태를 가진다.

```sql
SELECT * FROM users
WHERE username = '입력값'
AND password = '입력값';
```

문제는 입력값 안에 SQL 문법이 포함될 수 있다는 점이다.

---

## 3.2 취약한 코드 예제

```php
<?php
require_once "db.php";

$username = $_POST['username'];
$password = $_POST['password'];

$sql = "SELECT * FROM users WHERE username = '$username' AND password = '$password'";
$result = mysqli_query($conn, $sql);

if (mysqli_num_rows($result) > 0) {
    echo "Login success";
} else {
    echo "Login failed";
}
?>
```

---

## 3.3 취약한 이유

문제 부분은 다음 코드다.

```php
$sql = "SELECT * FROM users WHERE username = '$username' AND password = '$password'";
```

사용자가 입력한 값이 SQL Query 문자열 안에 그대로 들어간다.

예를 들어 다음과 같은 입력이 들어간다고 가정한다.

```text
username: admin' -- -
password: anything
```

그러면 Query는 다음과 비슷한 형태가 될 수 있다.

```sql
SELECT * FROM users
WHERE username = 'admin' -- -'
AND password = 'anything';
```

`-- -` 뒤쪽은 주석으로 처리될 수 있으므로, password 조건이 무시될 가능성이 있다.

---

## 3.4 대응 코드 예제

```php
<?php
require_once "db.php";

$username = $_POST['username'];
$password = $_POST['password'];

$stmt = $conn->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
$stmt->bind_param("ss", $username, $password);
$stmt->execute();

$result = $stmt->get_result();

if ($result->num_rows > 0) {
    echo "Login success";
} else {
    echo "Login failed";
}
?>
```

---

## 3.5 개선 포인트

| 항목 | 취약한 코드 | 대응 코드 |
|---|---|---|
| Query 생성 방식 | 문자열 결합 | Prepared Statement |
| 사용자 입력 처리 | SQL 문법과 섞임 | 값으로 바인딩 |
| 인증 조건 조작 가능성 | 높음 | 낮아짐 |
| 핵심 함수 | `mysqli_query()` | `prepare()`, `bind_param()`, `execute()` |

Prepared Statement를 사용하면 SQL Query 구조와 사용자 입력값이 분리된다.  
따라서 입력값에 SQL 문법이 포함되어도 Query 구조 자체를 바꾸기 어렵다.

---

# 4. Boolean-based SQL Injection

## 4.1 공격 유형 설명

Boolean-based SQL Injection은 참/거짓 조건에 따른 응답 차이를 이용하는 방식이다.

예를 들어 상품 조회 페이지에서 `id` 값이 SQL Query에 직접 들어간다고 가정한다.

정상 Query:

```sql
SELECT id, name, price FROM items WHERE id = 1;
```

공격자는 입력값에 조건식을 추가해 응답 차이를 확인할 수 있다.

```text
id=1 AND 1=1
id=1 AND 1=2
```

`1=1`은 참이고, `1=2`는 거짓이다.  
두 요청의 응답이 다르면 해당 입력값이 SQL 조건식에 영향을 주고 있다고 볼 수 있다.

---

## 4.2 취약한 코드 예제

```php
<?php
require_once "db.php";

$id = $_GET['id'];

$sql = "SELECT id, name, price FROM items WHERE id = $id";
$result = mysqli_query($conn, $sql);

while ($row = mysqli_fetch_assoc($result)) {
    echo "ID: " . $row['id'] . "<br>";
    echo "Name: " . $row['name'] . "<br>";
    echo "Price: " . $row['price'] . "<br>";
}
?>
```

---

## 4.3 취약한 이유

문제 부분:

```php
$sql = "SELECT id, name, price FROM items WHERE id = $id";
```

정상 입력:

```text
id=1
```

생성 Query:

```sql
SELECT id, name, price FROM items WHERE id = 1;
```

비정상 입력:

```text
id=1 AND 1=1
```

생성될 수 있는 Query:

```sql
SELECT id, name, price FROM items WHERE id = 1 AND 1=1;
```

다른 입력:

```text
id=1 AND 1=2
```

생성될 수 있는 Query:

```sql
SELECT id, name, price FROM items WHERE id = 1 AND 1=2;
```

이 두 요청의 결과가 다르면, 사용자의 입력값이 SQL 조건식으로 처리되고 있다는 단서가 된다.

---

## 4.4 대응 코드 예제

```php
<?php
require_once "db.php";

$id = $_GET['id'] ?? 0;

$stmt = $conn->prepare("SELECT id, name, price FROM items WHERE id = ?");
$stmt->bind_param("i", $id);
$stmt->execute();

$result = $stmt->get_result();

while ($row = $result->fetch_assoc()) {
    echo "ID: " . htmlspecialchars($row['id'], ENT_QUOTES, 'UTF-8') . "<br>";
    echo "Name: " . htmlspecialchars($row['name'], ENT_QUOTES, 'UTF-8') . "<br>";
    echo "Price: " . htmlspecialchars($row['price'], ENT_QUOTES, 'UTF-8') . "<br>";
}
?>
```

---

## 4.5 개선 포인트

```php
$stmt = $conn->prepare("SELECT id, name, price FROM items WHERE id = ?");
$stmt->bind_param("i", $id);
```

`?`는 placeholder이고, `$id`는 정수 값으로 바인딩된다.  
이 구조에서는 `1 AND 1=1` 같은 입력이 SQL 조건식으로 해석되기 어렵다.

---

# 5. Error-based SQL Injection

## 5.1 공격 유형 설명

Error-based SQL Injection은 DB 에러 메시지를 이용해 내부 정보를 확인하는 방식이다.

애플리케이션이 DB 에러를 사용자 화면에 그대로 출력하면 다음과 같은 정보가 노출될 수 있다.

- DBMS 종류
- 데이터베이스 이름
- Query 구조 일부
- 테이블 또는 컬럼 관련 단서

---

## 5.2 취약한 코드 예제

```php
<?php
require_once "db.php";

$id = $_GET['id'];

$sql = "SELECT id, name, price FROM items WHERE id = $id";
$result = mysqli_query($conn, $sql);

if (!$result) {
    die("Query Error: " . mysqli_error($conn));
}

while ($row = mysqli_fetch_assoc($result)) {
    echo $row['id'] . " / " . $row['name'] . " / " . $row['price'] . "<br>";
}
?>
```

---

## 5.3 취약한 이유

취약점은 두 가지가 겹쳐서 발생한다.

첫 번째는 입력값이 SQL Query에 직접 들어간다는 점이다.

```php
$sql = "SELECT id, name, price FROM items WHERE id = $id";
```

두 번째는 DB 에러를 사용자에게 그대로 출력한다는 점이다.

```php
die("Query Error: " . mysqli_error($conn));
```

예를 들어 다음과 같은 입력이 들어갈 수 있다.

```text
id=1 AND updatexml(1,concat(0x7e,database(),0x7e),1)-- -
```

생성될 수 있는 Query:

```sql
SELECT id, name, price FROM items
WHERE id = 1 AND updatexml(1,concat(0x7e,database(),0x7e),1)-- -;
```

에러 메시지에 현재 데이터베이스 이름이 포함될 가능성이 있다.

예상 관찰 예시:

```text
XPATH syntax error: '~sqli_lab~'
```

---

## 5.4 대응 코드 예제

```php
<?php
require_once "db.php";

$id = $_GET['id'] ?? 0;

$stmt = $conn->prepare("SELECT id, name, price FROM items WHERE id = ?");
$stmt->bind_param("i", $id);
$stmt->execute();

$result = $stmt->get_result();

if (!$result) {
    error_log("Database query failed");
    die("Internal server error");
}

while ($row = $result->fetch_assoc()) {
    echo htmlspecialchars($row['id'], ENT_QUOTES, 'UTF-8') . " / ";
    echo htmlspecialchars($row['name'], ENT_QUOTES, 'UTF-8') . " / ";
    echo htmlspecialchars($row['price'], ENT_QUOTES, 'UTF-8') . "<br>";
}
?>
```

---

## 5.5 개선 포인트

| 문제 | 대응 |
|---|---|
| 입력값이 Query에 직접 삽입됨 | Prepared Statement 사용 |
| DB 에러가 화면에 출력됨 | 상세 에러는 서버 로그에 기록 |
| 내부 Query 정보가 노출될 수 있음 | 사용자에게 일반 메시지만 표시 |

실제 운영 환경에서는 상세한 DB 에러를 사용자에게 보여주지 않는 편이 좋다.

---

# 6. Union-based SQL Injection

## 6.1 공격 유형 설명

Union-based SQL Injection은 기존 `SELECT` Query에 `UNION SELECT`를 결합해 다른 Query 결과를 함께 출력하는 방식이다.

이 방식이 동작하려면 보통 다음 조건이 필요하다.

| 조건 | 설명 |
|---|---|
| 기존 Query가 SELECT 문이어야 함 | UNION은 SELECT 결과를 합치는 기능 |
| 컬럼 개수가 맞아야 함 | 앞뒤 SELECT의 컬럼 수가 같아야 함 |
| 타입이 호환되어야 함 | 각 위치의 데이터 타입이 맞아야 함 |
| 결과가 화면에 출력되어야 함 | 출력 위치가 있어야 확인 가능 |

---

## 6.2 취약한 코드 예제

```php
<?php
require_once "db.php";

$id = $_GET['id'];

$sql = "SELECT id, name, price FROM items WHERE id = $id";
$result = mysqli_query($conn, $sql);

while ($row = mysqli_fetch_assoc($result)) {
    echo "<h3>" . $row['name'] . "</h3>";
    echo "<p>Price: " . $row['price'] . "</p>";
}
?>
```

---

## 6.3 취약한 이유

기존 Query:

```sql
SELECT id, name, price FROM items WHERE id = 1;
```

공격 입력 예시:

```text
id=-1 UNION SELECT 1,database(),3-- -
```

생성될 수 있는 Query:

```sql
SELECT id, name, price FROM items
WHERE id = -1 UNION SELECT 1,database(),3-- -;
```

`id=-1`은 기존 결과를 비우기 위해 사용될 수 있다.  
이후 `UNION SELECT` 결과가 화면에 출력되는지 확인한다.

---

## 6.4 대응 코드 예제

```php
<?php
require_once "db.php";

$id = $_GET['id'] ?? 0;

$stmt = $conn->prepare("SELECT id, name, price FROM items WHERE id = ?");
$stmt->bind_param("i", $id);
$stmt->execute();

$result = $stmt->get_result();

while ($row = $result->fetch_assoc()) {
    echo "<h3>" . htmlspecialchars($row['name'], ENT_QUOTES, 'UTF-8') . "</h3>";
    echo "<p>Price: " . htmlspecialchars($row['price'], ENT_QUOTES, 'UTF-8') . "</p>";
}
?>
```

---

## 6.5 개선 포인트

Prepared Statement를 사용하면 `UNION SELECT` 문자열이 SQL 문법으로 결합되지 않고 하나의 입력값으로 처리된다.

또한 출력 시 `htmlspecialchars()`를 사용하면 DB에서 가져온 값이 HTML로 해석되는 위험도 줄일 수 있다.  
단, `htmlspecialchars()`는 SQL Injection 대응책이 아니라 출력 인코딩에 해당한다.

---

# 7. Time-based Blind SQL Injection

## 7.1 공격 유형 설명

Time-based Blind SQL Injection은 화면 응답 차이가 거의 없을 때 응답 시간 차이를 이용해 SQL 조건 결과를 추론하는 방식이다.

예를 들어 조건이 참이면 DB에서 지연 함수를 실행하고, 거짓이면 바로 응답하게 만들 수 있다.

---

## 7.2 취약한 코드 예제

```php
<?php
require_once "db.php";

$id = $_GET['id'];

$sql = "SELECT id, name, price FROM items WHERE id = $id";
$result = mysqli_query($conn, $sql);

echo "Request completed";
?>
```

이 코드는 Query 결과를 자세히 보여주지 않지만, 입력값이 SQL Query에 직접 들어가므로 Blind SQLi 가능성이 남아 있다.

---

## 7.3 취약한 이유

입력 예시:

```text
id=1 AND IF(1=1,SLEEP(5),0)
```

생성될 수 있는 Query:

```sql
SELECT id, name, price FROM items
WHERE id = 1 AND IF(1=1,SLEEP(5),0);
```

조건이 참일 때 응답이 지연된다면, 공격자는 응답 시간을 통해 조건 결과를 추론할 수 있다.

---

## 7.4 대응 코드 예제

```php
<?php
require_once "db.php";

$id = $_GET['id'] ?? 0;

$stmt = $conn->prepare("SELECT id, name, price FROM items WHERE id = ?");
$stmt->bind_param("i", $id);
$stmt->execute();

echo "Request completed";
?>
```

---

## 7.5 개선 포인트

Blind SQLi는 화면에 직접 데이터가 보이지 않아도 발생할 수 있다.  
따라서 결과를 출력하지 않는다고 해서 안전하다고 볼 수 없다.

중요한 점은 Query 생성 구조다.

```text
문제: 입력값이 SQL Query 문자열에 직접 삽입됨
대응: 입력값을 Parameter로 바인딩
```

---

# 8. Second-order SQL Injection

## 8.1 공격 유형 설명

Second-order SQL Injection은 입력 시점에는 공격이 바로 발생하지 않지만, 저장된 값이 나중에 다른 SQL Query에 사용될 때 발생할 수 있는 유형이다.

예를 들어 회원가입 때 저장한 `username`이 나중에 관리자 페이지의 Query에 직접 들어가는 경우를 생각할 수 있다.

---

## 8.2 취약한 코드 예제

### 회원가입 코드

```php
<?php
require_once "db.php";

$username = $_POST['username'];
$email = $_POST['email'];

$stmt = $conn->prepare("INSERT INTO users (username, email) VALUES (?, ?)");
$stmt->bind_param("ss", $username, $email);
$stmt->execute();

echo "User registered";
?>
```

이 코드는 저장 시점에는 Prepared Statement를 사용하고 있다.

하지만 나중에 관리자 페이지에서 다음과 같이 사용하면 문제가 생길 수 있다.

### 관리자 조회 코드

```php
<?php
require_once "db.php";

$username = $_GET['username'];

$sql = "SELECT * FROM users WHERE username = '$username'";
$result = mysqli_query($conn, $sql);

while ($row = mysqli_fetch_assoc($result)) {
    echo $row['username'] . " / " . $row['email'] . "<br>";
}
?>
```

---

## 8.3 취약한 이유

Second-order SQLi의 핵심은 다음과 같다.

```text
1. 악의적인 문자열이 DB에 저장됨
2. 저장 당시에는 바로 실행되지 않음
3. 이후 다른 기능에서 해당 값을 SQL Query에 직접 삽입
4. 그 시점에 SQL Injection 가능성 발생
```

즉, 저장할 때만 안전하게 처리하는 것으로는 부족하다.  
DB에서 꺼낸 값이라도 다시 Query에 넣을 때는 안전하게 바인딩해야 한다.

---

## 8.4 대응 코드 예제

```php
<?php
require_once "db.php";

$username = $_GET['username'];

$stmt = $conn->prepare("SELECT * FROM users WHERE username = ?");
$stmt->bind_param("s", $username);
$stmt->execute();

$result = $stmt->get_result();

while ($row = $result->fetch_assoc()) {
    echo htmlspecialchars($row['username'], ENT_QUOTES, 'UTF-8') . " / ";
    echo htmlspecialchars($row['email'], ENT_QUOTES, 'UTF-8') . "<br>";
}
?>
```

---

## 8.5 개선 포인트

| 상황 | 주의점 |
|---|---|
| 사용자 입력값 | 신뢰하면 안 됨 |
| DB에 저장된 값 | 다시 Query에 넣을 때도 신뢰하면 안 됨 |
| 내부 시스템 값 | 출처와 변환 과정을 확인해야 함 |

Query에 들어가는 모든 값은 Prepared Statement로 처리하는 편이 좋다.

---

# 9. 취약 코드 패턴 정리

SQL Injection 취약 코드에는 공통적인 패턴이 있다.

## 9.1 문자열 결합

```php
$sql = "SELECT * FROM users WHERE username = '$username'";
```

## 9.2 숫자 입력값 직접 삽입

```php
$sql = "SELECT * FROM items WHERE id = $id";
```

## 9.3 정렬 기준 직접 삽입

```php
$order = $_GET['order'];
$sql = "SELECT * FROM items ORDER BY $order";
```

`ORDER BY`나 테이블명, 컬럼명은 Prepared Statement의 placeholder로 처리하기 어려운 경우가 있다.  
이 경우에는 허용 목록 기반 검증이 필요하다.

대응 예시:

```php
$allowed = ['name', 'price', 'id'];
$order = $_GET['order'] ?? 'id';

if (!in_array($order, $allowed, true)) {
    $order = 'id';
}

$sql = "SELECT id, name, price FROM items ORDER BY $order";
$result = mysqli_query($conn, $sql);
```

---

# 10. 대응 방식 요약

| 취약 패턴 | 문제 | 대응 |
|---|---|---|
| 입력값 문자열 결합 | SQL 문법으로 해석될 수 있음 | Prepared Statement |
| 숫자값 직접 삽입 | 조건식 삽입 가능성 | 정수 바인딩 |
| 에러 직접 출력 | 내부 정보 노출 가능성 | 일반 에러 메시지 사용 |
| 저장값 재사용 | Second-order SQLi 가능성 | 재사용 시에도 바인딩 |
| 컬럼명 직접 삽입 | Query 구조 조작 가능성 | 허용 목록 검증 |

---

# 11. 정리

SQL Injection 취약 코드의 핵심 문제는 다음과 같다.

```text
사용자 입력값 또는 신뢰할 수 없는 값이 SQL Query 문자열에 직접 결합된다.
```

안전한 코드의 핵심은 다음과 같다.

```text
SQL Query 구조와 입력값을 분리한다.
```

주요 대응 방식:

- Prepared Statement 사용
- Parameter Binding 사용
- 입력값 타입 검증
- 허용 목록 기반 검증
- DB 에러 메시지 제한
- 출력 시 HTML 인코딩 적용
- DB 계정 최소 권한 적용

---

## 참고자료

- MITRE CWE-89: Improper Neutralization of Special Elements used in an SQL Command  
  https://cwe.mitre.org/data/definitions/89.html

- OWASP SQL Injection Prevention Cheat Sheet  
  https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html

- PortSwigger Web Security Academy - SQL Injection  
  https://portswigger.net/web-security/sql-injection
