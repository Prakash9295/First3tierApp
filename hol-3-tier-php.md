## Building a Simple 3-Tier Architecture with Docker Compose: A Hands-On Project Journey

![image](../images/3_tier_image.webp)

1. A frontend container
2. A backend container (With an API layer)
3. A MySQL database container

### Frontend App Container
**index.html**  
Make directory `frontend` and save below file as `index.html`

```bash
mkdir apache-php-mysql
cd apache-php-mysql
mkdir frontend
cd frontend
cat > index.html
```

```html
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>HTML Web Page</title>
  </head>
  <body>
    <h1>Hello, World!</h1>
  </body>
</html>
```
## Run apache docker container in detached mode

![image](../images/3_tier_frontend_apachephpmysql.webp)
```bash
docker run -d -p 3000:80 -v ./frontend:/usr/local/apache2/htdocs httpd:latest
```

## Main Project Directory
```bash
apache-php-mysql/
└── frontend
    └── index.html
```

### Backend App Container
![image](../images/3_tier_backend_apachephpmysql.webp)

**index.php**

Create folder `api` and copy & paste the below code snippet to `index.php`

```bash
mkdir api
cd api
cat > index.php
```
```php
<?php
// Set the response content type to JSON
header('Content-Type: application/json');
header('Access-Control-Allow-Origin: http://localhost:3000');
header('Access-Control-Allow-Methods: GET, POST, OPTIONS');

// Create a response array
$response = [
    [
        "_id" => 1,
        "todo" => "Todo 1"
    ],
    [
        "_id" => 2,
        "todo" => "Todo 2"
    ],
    [
        "_id" => 3,
        "todo" => "Todo 3"
    ],
    [
        "_id" => 4,
        "todo" => "Todo 4"
    ]
];

// Encode the response array as JSON
$jsonResponse = json_encode($response);

// Output the JSON response
echo $jsonResponse;
```

**Dockerfile**

```bash
FROM ubuntu:20.04

LABEL maintainer="kesaralive@gmail.com"
LABEL description="Apache / PHP development environment"

ARG DEBIAN_FRONTEND=newt
RUN apt-get update && apt-get install -y lsb-release && apt-get clean all
RUN apt install ca-certificates apt-transport-https software-properties-common -y
RUN add-apt-repository ppa:ondrej/php

RUN apt-get -y update && apt-get install -y \
apache2 \
php8.0 \
libapache2-mod-php8.0 \
php8.0-bcmath \
php8.0-gd \
php8.0-sqlite \
php8.0-mysql \
php8.0-curl \
php8.0-xml \
php8.0-mbstring \
php8.0-zip \
mcrypt \
nano

RUN apt-get install locales
RUN locale-gen fr_FR.UTF-8
RUN locale-gen en_US.UTF-8
RUN locale-gen de_DE.UTF-8

# config PHP
# we want a dev server which shows PHP errors
RUN sed -i -e 's/^error_reporting\s*=.*/error_reporting = E_ALL/' /etc/php/8.0/apache2/php.ini
RUN sed -i -e 's/^display_errors\s*=.*/display_errors = On/' /etc/php/8.0/apache2/php.ini
RUN sed -i -e 's/^zlib.output_compression\s*=.*/zlib.output_compression = Off/' /etc/php/8.0/apache2/php.ini

# to be able to use "nano" with shell on "docker exec -it [CONTAINER ID] bash"
ENV TERM xterm

# Apache conf
# allow .htaccess with RewriteEngine
RUN a2enmod rewrite
# to see live logs we do : docker logs -f [CONTAINER ID]
# without the following line we get "AH00558: apache2: Could not reliably determine the server's fully qualified domain name"
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf
# autorise .htaccess files
RUN sed -i '/<Directory \/var\/www\/>/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/apache2/apache2.conf

RUN chgrp -R www-data /var/www
RUN find /var/www -type d -exec chmod 775 {} +
RUN find /var/www -type f -exec chmod 664 {} +

EXPOSE 80

# start Apache2 on image start
CMD ["/usr/sbin/apache2ctl","-DFOREGROUND"]
```

```bash
apache-php-mysql/
├── Dockerfile
├── api
│   └── index.php
└── frontend
    └── index.html
```

Build backend docker image  
```bash
docker build -t simple-backend .
```

Run backend container

```bash
docker run -d -p 5000:80 -v ./api:/var/www/html simple-backend 
```

## Accessing the Backend from the Frontend
![image](../images/3_tier_apachephpmysql_access_backend.webp)

### Let’s update our frontend code as below to call the backend API

Update `index.html` with below code snippet

```html
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>HTML Web Page</title>
  </head>
  <body>
    <h1>Hello, World!</h1>
    <div id="todos"></div>

    <script>
      fetch("http://localhost:5000/")
        .then((response) => response.json())
        .then((data) => {
          const todos = document.getElementById("todos");

          data?.forEach((item, index) => {
            const todo = document.createElement("h2");
            todo.textContent = item._id + ". " + item.todo;
            todos.appendChild(todo);
          });
        })
        .catch((error) => {
          console.log("Error: ", error);
        });
    </script>
  </body>
</html>
```

### Adding a MySQL Database container
![image](../images/3_tier_apachephpmysql_add_db.webp)

Let’s add the sql dump file.
Create folder `db` and paste below code to `dump.sql` file

**dump.sql**

```sql
-- phpMyAdmin SQL Dump
-- version 5.2.1
-- https://www.phpmyadmin.net/
--
-- Host: database:3306

-- Server version: 8.0.33
-- PHP Version: 8.1.17

SET SQL_MODE = "NO_AUTO_VALUE_ON_ZERO";
START TRANSACTION;
SET time_zone = "+00:00";


/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;

--
-- Database: `todo_app`
--

-- --------------------------------------------------------

--
-- Table structure for table `todos`
--

CREATE TABLE `todos` (
  `_id` int NOT NULL,
  `todo` varchar(255) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

--
-- Dumping data for table `todos`
--

INSERT INTO `todos` (`_id`, `todo`) VALUES
(1, 'I will wake up at 4 in the morining.'),
(2, 'I will practice docker for 1 hour.'),
(3, 'I will give time for 2 hours javascript.'),
(4, 'Then I will have breakfast.'),
(5, 'I will give time for 3 hours php.');

--
-- Indexes for dumped tables
--

--
-- Indexes for table `todos`
--
ALTER TABLE `todos`
  ADD PRIMARY KEY (`_id`);

--
-- AUTO_INCREMENT for dumped tables
--

--
-- AUTO_INCREMENT for table `todos`
--
ALTER TABLE `todos`
  MODIFY `_id` int NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=6;
COMMIT;
```


```bash
apache-php-mysql/
├── Dockerfile
├── api
│   └── index.php
├── db
│   └── dump.sql
└── frontend
    └── index.html
```

Run the mysql docker

```bash
docker run -d -e MYSQL_DATABASE=todo_app -e MYSQL_USER=todo_admin -e MYSQL_PASSWORD=password -e MYSQL_ALLOW_EMPTY_PASSWORD=1 -v ./db:/docker-entrypoint-initdb.d mysql:latest

```

Once started the database, check if all the databases has created  
Frist exec into the db container
```bash
docker exec -it bd3b3c27ce3e bash

```

```sql
SHOW DATABASES;
```

### PhpMyAdmin to Access the MySQL Database Container

![image](../images/3_tier_apachephpmysql_addphpmyadmin.webp)


Run **phpmyadmin**

```bash
docker run -d \
    -p 8080:80 \
    -e PMA_HOST=bd3b3c27ce3e \
    -e PMA_PORT=3306 \
    phpmyadmin/phpmyadmin

```

### Wiring up the Backend API with Database container

![image](../images/3_tier_apachephpmysql_db_backend.webp)


Create a folder name `app` inside the `api` folder.Create `config.php` file and add below content.

**config.php**
```php
<?php 
define("DB_HOST", "database");
define("DB_USERNAME", "todo_admin");
define("DB_PASSWORD", "password");
define("DB_NAME", "todo_app");
```

**Database.php**
```php
<?php
class Database
{
    protected $connection = null;
    
    public function __construct()
    {
        try {
            $this->connection = new mysqli(DB_HOST, DB_USERNAME, DB_PASSWORD, DB_NAME);
            if (mysqli_connect_errno()) {
                throw new Exception("Database connection failed!");
            }
        } catch (Exception $e) {
            throw new Exception($e->getMessage());
        }
    }

    private function executeStatement($query = "", $params = [])
    {
        try {
            $stmt = $this->connection->prepare($query);
            if ($stmt === false) {
                throw new Exception("Statement preparation failure: " . $query);
            }
            if ($params) {
                $stmt->bind_param($params[0], $params[1]);
            }
            $stmt->execute();
            return $stmt;
        } catch (Exception $e) {
            throw new Exception($e->getMessage());
        }
    }

    public function select($query = "", $params = [])
    {
        try {
            $stmt = $this->executeStatement($query, $params);
            $result = $stmt->get_result()->fetch_all(MYSQLI_ASSOC);
            $stmt->close();
            return $result;
        } catch (Exception $e) {
            throw new Exception($e->getMessage());
        }
        return false;
    }
}
```

**todos.php**

```php
<?php 
require_once "./app/Database.php";

class Todo extends Database
{
    public function getTodos($limit)
    {
        return $this->select("SELECT * FROM todos");
    }
}
```

To connect all together, let’s update our `index.php` by replacing with the below code snippet.

**index.php**

```php
<?php
// Set the response content type to JSON
header('Content-Type: application/json');
header('Access-Control-Allow-Origin: http://localhost:3000');
header('Access-Control-Allow-Methods: GET, POST, OPTIONS');

require "./app/config.php";
require_once "./app/todos.php";

$todoModel = new Todo();
$response = $todoModel->getTodos(10);
// Create a response array

// Encode the response array as JSON
$jsonResponse = json_encode($response);

// Output the JSON response
echo $jsonResponse;
```

## Troubleshooting  

## Few commands
```bash
apt-get install vim
apt-get install php-mysql
sudo service apache2 restart

sudo service php-fpm restart
## Install ps command
apt install procps


```
### Check if php module is loaded
```php
<?php
$extensionName = 'mysqli'; // Replace this with the name of the extension you want to check

if (extension_loaded($extensionName)) {
    echo "The $extensionName extension is available.\n";
} else {
    echo "The $extensionName extension is not available.\n";
}
?>
```
### Enter into docker namespace
```bash
sudo nsenter --target $(docker inspect --format '{{.State.Pid}}' focused_euclid) --mount --uts --ipc --net --pid
```

### Show ip address of all the running containers
```bash
docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -q)
```

### Iptables enable logging
```bash
sudo iptables -A INPUT -j LOG --log-prefix "INPUT: "
```

### tcpdump
```bash
tcpdump -i ens5 port 5000
sudo tcpdump -i <interface> not port 22
tcpdump -vvv -n -i ens5 tcp and not port 22 and not port 80 and not port 53 and not port 443

```
### Docker run with container name
```bash
docker run -d --name fe -v ./frontend:/usr/local/apache2/htdocs -p 3000:80 httpd:latest
```

### Enable the MySQL Extension
Uncomment `extension=mysqli` in `/etc/php/{version}/apache2/php.ini` file

### Docker start with --add-hosts
```bash
docker run --rm=True --add-host=testing.example.com:10.0.0.1 -it ubuntu /bin/bash
docker run --hostname test --ip 10.1.2.3 ubuntu:14.04 
```

## Run docker in host network
```bash
docker run --network=host <image_name>

```

## .htaccess file
```bash
Header add Access-Control-Allow-Origin "*"
Header add Access-Control-Allow-Methods: "GET,POST,OPTIONS,DELETE,PUT"
```
```bash
Header add Access-Control-Allow-Origin "*"
Header add Access-Control-Allow-Headers "origin, x-requested-with, content-type"
Header add Access-Control-Allow-Methods "PUT, GET, POST, DELETE, OPTIONS"
```
## [Check](https://stackoverflow.com/questions/14467673/enable-cors-in-htaccess)

### Run container in separate network with the ip specified
Some time following error might be seen
> docker: Error response from daemon: invalid config for network testnetwork: invalid endpoint settings: user specified IP address is supported only when connecting to networks with user configured subnets. See 'docker run --help'.

```bash
docker network create --subnet=192.168.0.0/16 mynetwork
docker run --net mynetwork --ip 192.168.0.2 -d myimage

docker network create my_network_name --subnet=10.11.0.0/16
docker run -it --net my_network_name --ip 10.11.0.10 nginx
```

### Steps
```bash
docker run -d -e MYSQL_DATABASE=todo_app -e MYSQL_USER=todo_admin -e MYSQL_PASSWORD=password -e MYSQL_ALLOW_EMPTY_PASSWORD=1 -v ./db:/docker-entrypoint-initdb.d --name=db --ip 172.17.0.2 mysql:latest

docker run -d --name=be -p 5000:80 -v ./api:/var/www/html --add-host=database:172.17.0.2 --ip 172.17.0.3 be

docker run -d -p 3000:80 -v ./frontend:/usr/local/apache2/htdocs --name=fe --add-host=backend:172.17.0.4 httpd:latest
docker run -d -p 8080:80 --add-host=database:172.17.0.2 -e PMA_HOST=database -e PMA_PORT=3306 --ip 172.17.0.5 phpmyadmin/phpmyadmin

```
## References
1. [github repo](https://github.com/DeepakBomjan/apache-php-mysql)
2. [Building a Simple 3-Tier Architecture with Docker Compose: A Hands-On Project Journey](https://medium.com/@kesaralive/getting-started-with-docker-compose-hands-on-project-experience-e562ab07e24c)
