# Лабораторная работа: Непрерывная интеграция с использованием GitHub Actions и Docker

## Цель работы

Цель данной лабораторной работы — научиться настраивать процесс непрерывной интеграции (Continuous Integration) с помощью GitHub Actions для web-приложения, запущенного в Docker-контейнере. 

## Задание

1. Создать PHP web-приложение с базой данных SQLite.
2. Написать юнит-тесты для логики приложения.
3. Создать Dockerfile для запуска приложения в контейнере.
4. Настроить GitHub Actions для запуска тестов внутри контейнера.

## Ход работы

### 1. Создание структуры проекта

В корневом каталоге `containers08` я создала следующие папки:

```bash
containers08
├── site
│   ├── modules
│   ├── templates
│   ├── styles
├── sql
├── tests
├── Dockerfile
├── .github/workflows/main.yml
```

### 2. Создание Web-приложения

####  site/modules/database.php
```php
<?php
class Database {
    private $pdo;

    public function __construct($path) {
        $this->pdo = new PDO("sqlite:" . $path);
        $this->pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    }

    public function Execute($sql) {
        return $this->pdo->exec($sql);
    }

    public function Fetch($sql) {
        $stmt = $this->pdo->query($sql);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    public function Create($table, $data) {
        $columns = implode(", ", array_keys($data));
        $placeholders = ":" . implode(", :", array_keys($data));
        $stmt = $this->pdo->prepare("INSERT INTO $table ($columns) VALUES ($placeholders)");
        $stmt->execute($data);
        return $this->pdo->lastInsertId();
    }

    public function Read($table, $id) {
        $stmt = $this->pdo->prepare("SELECT * FROM $table WHERE id = :id");
        $stmt->execute(['id' => $id]);
        return $stmt->fetch(PDO::FETCH_ASSOC);
    }

    public function Update($table, $id, $data) {
        $sets = implode(", ", array_map(fn($k) => "$k = :$k", array_keys($data)));
        $data['id'] = $id;
        $stmt = $this->pdo->prepare("UPDATE $table SET $sets WHERE id = :id");
        return $stmt->execute($data);
    }

    public function Delete($table, $id) {
        $stmt = $this->pdo->prepare("DELETE FROM $table WHERE id = :id");
        return $stmt->execute(['id' => $id]);
    }

    public function Count($table) {
        $stmt = $this->pdo->query("SELECT COUNT(*) FROM $table");
        return $stmt->fetchColumn();
    }
}
```

####  site/modules/page.php
```php
<?php
class Page {
    private $template;

    public function __construct($template) {
        $this->template = file_get_contents($template);
    }

    public function Render($data) {
        $output = $this->template;
        foreach ($data as $key => $value) {
            $output = str_replace("{{ $key }}", $value, $output);
        }
        return $output;
    }
}
```

####  site/templates/index.tpl
```html
<!DOCTYPE html>
<html>
<head>
    <title>{{ title }}</title>
    <link rel="stylesheet" href="styles/style.css">
</head>
<body>
    <h1>{{ title }}</h1>
    <div>{{ content }}</div>
</body>
</html>
```

####  site/styles/style.css
```css
body {
    font-family: Arial, sans-serif;
    background-color: #f4f4f4;
    padding: 20px;
}
```

####  site/index.php
```php
<?php
require_once __DIR__ . '/modules/database.php';
require_once __DIR__ . '/modules/page.php';
require_once __DIR__ . '/config.php';

$db = new Database($config['db']['path']);
$page = new Page(__DIR__ . '/templates/index.tpl');
$pageId = $_GET['page'] ?? 1;
$data = $db->Read("page", $pageId);
echo $page->Render($data);
```

####  site/config.php
```php
<?php
$config = [
    "db" => [
        "path" => "/var/www/db/db.sqlite"
    ]
];
```

### 3. Подготовка базы данных (sql/schema.sql)
```sql
CREATE TABLE page (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT,
    content TEXT
);

INSERT INTO page (title, content) VALUES ('Page 1', 'Content 1');
INSERT INTO page (title, content) VALUES ('Page 2', 'Content 2');
INSERT INTO page (title, content) VALUES ('Page 3', 'Content 3');
```

### 4. Тестирование

####  tests/testframework.php
```php
<?php
function message($type, $message) {
    $time = date('Y-m-d H:i:s');
    echo "{$time} [{$type}] {$message}\n";
}

function info($message) { message('INFO', $message); }
function error($message) { message('ERROR', $message); }

function assertExpression($expression, $pass = 'Pass', $fail = 'Fail'): bool {
    if ($expression) {
        info($pass);
        return true;
    }
    error($fail);
    return false;
}

class TestFramework {
    private $tests = [];
    private $success = 0;

    public function add($name, $test) {
        $this->tests[$name] = $test;
    }

    public function run() {
        foreach ($this->tests as $name => $test) {
            info("Running test {$name}");
            if ($test()) $this->success++;
            info("End test {$name}");
        }
    }

    public function getResult() {
        return "{$this->success} / " . count($this->tests);
    }
}
```

####  tests/tests.php (фрагмент)
```php
<?php
require_once __DIR__ . '/testframework.php';
require_once __DIR__ . '/../config.php';
require_once __DIR__ . '/../modules/database.php';
require_once __DIR__ . '/../modules/page.php';

$tests = new TestFramework();

$tests->add('Database connection', function () {
    global $config;
    $db = new Database($config['db']['path']);
    return assertExpression($db instanceof Database, "Database created");
});

$tests->run();
echo $tests->getResult();
```

### 5. Dockerfile
```Dockerfile
FROM php:7.4-fpm as base

RUN apt-get update && \
    apt-get install -y sqlite3 libsqlite3-dev && \
    docker-php-ext-install pdo_sqlite

VOLUME ["/var/www/db"]
COPY sql/schema.sql /var/www/db/schema.sql

RUN echo "prepare database" && \
    cat /var/www/db/schema.sql | sqlite3 /var/www/db/db.sqlite && \
    chmod 777 /var/www/db/db.sqlite && \
    rm -rf /var/www/db/schema.sql && \
    echo "database is ready"

COPY site /var/www/html
```

### 6. GitHub Actions CI (main.yml)
```yaml
name: CI

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build the Docker image
        run: docker build -t containers08 .

      - name: Create `container`
        run: docker create --name container --volume database:/var/www/db containers08

      - name: Copy tests to the container
        run: docker cp ./tests container:/var/www/html

      - name: Up the container
        run: docker start container

      - name: Run tests
        run: docker exec container php /var/www/html/tests/tests.php

      - name: Stop the container
        run: docker stop container

      - name: Remove the container
        run: docker rm container
```

# Лабораторная работа: Непрерывная интеграция с использованием GitHub Actions и Docker

## Цель работы

Цель данной лабораторной работы — научиться настраивать процесс непрерывной интеграции (соntinuous integration) с помощью GitHub Actions для web-приложения, запущенного в Docker.

## Задача

1. Создать PHP web-приложение с базой SQLite.
2. Написать тесты.
3. Настроить GitHub Actions для запуска тестов в Docker-контейнере.

## Ход работы

### Структура проекта

Я создала папку `containers08`, где было размещено web-приложение в `site/`, SQL-схема в `sql/schema.sql`, тесты в `tests/`, а также файл `Dockerfile` и workflow `.github/workflows/main.yml`.

### Web-приложение на PHP

Web-структура:
- `modules/database.php` — реализует CRUD-операции на SQLite;
- `modules/page.php` — отвечает за отображение HTML-шаблонов;
- `templates/index.tpl` — HTML-шаблон с {{ title }} и {{ content }};
- `index.php` — основная точка входа.

### SQL-схема

`schema.sql` создаёт таблицу `page` и добавляет 3 тестовые записи. База создаётся прямо в Dockerfile.

### Тесты

Я написала класс `TestFramework` и добавила тесты на все методы `Database` и `Page`. Тесты выполняются локально и в CI.

### Dockerfile

- Собирает среду PHP + SQLite.
- Копирует проект и SQL.
- Создаёт базу `db.sqlite` сразу при сборке.

### GitHub Actions (main.yml)

Workflow:
- Собирает Docker-образ;
- Запускает контейнер;
- Копирует `tests/`;
- Запускает `tests.php`;
- Удаляет контейнер.

Проверка в GitHub Actions прошла успешно — все 7 тестов прошли.

---

## Ответы на вопросы

###  Что такое непрерывная интеграция?

CI — это автоматическая проверка кода при каждом push/изменении. Тесты запускаются сами, что удобно и предотвращает ошибки.

###  Зачем нужны юнит-тесты?

Юнит-тесты проверяют отдельные функции/классы. Я запускаю их при каждой проверке CI (а значит — постоянно).

###  Что добавить для pull request?

```yaml
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
```

###  Что добавить для удаления Docker-образа?

```yaml
- name: Remove Docker image
  run: docker rmi containers08
```

## Выводы

Я научилась:
- запускать web-приложение через Docker;
- настраивать автоматический CI с GitHub Actions;
- исправлять ошибки по логам CI;
- работать с SQLite и PHP-тестами.

Все шаги прошли успешно, CI зелёный, приложение работает.



