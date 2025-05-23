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
В папке site/modules реализованы два основных класса: Database для работы с SQLite и Page для шаблонов страниц.

Класс Database

Класс Database инкапсулирует всю логику доступа к SQLite-базе данных. Он содержит методы для выполнения SQL-запросов (с возвратом и без), а также CRUD-операции.

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
Класс Page работает с шаблонами HTML. В нём два метода: конструктор загружает шаблон из файла, а метод Render() заменяет плейсхолдеры вроде {{ title }} и {{ content }} на реальные данные.
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
Этот файл связывает вместе конфигурацию, подключает базу и шаблон и выводит нужную страницу в зависимости от ID в URL.
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
Содержит путь к базе данных. Я указала абсолютный путь для контейнера: /var/www/db/db.sqlite.
```php
<?php
$config = [
    "db" => [
        "path" => "/var/www/db/db.sqlite"
    ]
];
```

### 3. Подготовка базы данных (sql/schema.sql)
В каталоге sql/ я создала файл schema.sql, который создаёт таблицу page и наполняет её тремя строками. Этот файл используется как при ручной инициализации, так и внутри Dockerfile для подготовки базы при сборке.
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
Я создала простой фреймворк TestFramework в testframework.php. В нём есть методы для логирования (info, error) и подсчёта успешных тестов. Далее в tests.php я реализовала тесты для всех методов класса Database, а также проверку работы класса Page.

Каждый тест изолирован и добавляется в фреймворк через метод add. После запуска результат отображается в консоль.
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
Я написала Dockerfile, в котором:

Устанавливается PHP и SQLite;

Копируется SQL-скрипт и создаётся база данных;

Копируется код приложения в контейнер;

Назначается том для базы (/var/www/db).

Команды RUN внутри Docker создают базу данных db.sqlite на этапе сборки.
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
Я создала файл .github/workflows/main.yml. Этот CI запускается на каждый push в ветку main.

Шаги:

Клонирование репозитория.

Сборка Docker-образа.

Создание и запуск контейнера.

Копирование тестов внутрь.

Запуск tests.php внутри контейнера.

Остановка и удаление контейнера.

Результат прогонки тестов отображается во вкладке Actions в репозитории GitHub. У меня тесты выполняются успешно (7 из 7).

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
---

## Ответы на вопросы

###  Что такое непрерывная интеграция?

Непрерывная интеграция (CI) — это практика автоматического запуска тестов при каждом изменении кода. Это позволяет находить ошибки на раннем этапе и облегчает командную разработку.

###  Зачем нужны юнит-тесты?

Юнит-тесты проверяют отдельные функции и классы. Они помогают убедиться, что изменения не нарушают текущую логику. Их стоит запускать при каждом коммите или pull request — и именно это я реализовала с помощью CI.

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



