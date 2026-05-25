# Implementación de Servidor Web Nginx con PHP-FPM desde Código Fuente en Alma Linux

---

## TESOEM

**Integrante:** Nadir Uriel Rosas Millán 
**Integrante:** kevin Daniel Arollo Ugalde 
**Integrante:** Donovan Axl Jimenez Vargas

 

**Fecha:** 25 de mayo del 2026

---

# Objetivo General

Implementar un servidor web completo utilizando Nginx y PHP-FPM compilados desde código fuente en Alma Linux, configurados con SystemD y comunicación mediante socket UNIX para procesamiento dinámico de páginas PHP.

---

# Objetivos Específicos

1. Compilar Nginx versión 1.31.x desde código fuente con prefix en `/srv/nginx`
2. Compilar PHP versión 8.4.x desde código fuente con soporte FPM y extensiones GD, Intl y Date
3. Configurar usuario y grupo nginx para el servicio web
4. Implementar servicio SystemD para auto-inicio en `multi-user.target`
5. Configurar comunicación Nginx-PHP mediante socket UNIX en `/tmp/php84.sock`
6. Verificar funcionamiento con script PHP informativo

---

# Desarrollo del Proyecto

## 1. Preparación del Sistema (Alma Linux)

```bash
# Actualizar sistema
sudo dnf update -y

# Instalar herramientas de desarrollo
sudo dnf groupinstall "Development Tools" -y
sudo dnf install epel-release -y

# Instalar dependencias necesarias
sudo dnf install -y gcc make pcre-devel zlib-devel openssl-devel \
    libxml2-devel sqlite-devel libcurl-devel libpng-devel \
    libjpeg-turbo-devel freetype-devel bzip2-devel libxslt-devel \
    libzip-devel systemd-devel oniguruma-devel wget tar gd-devel \
    libwebp-devel
```

---

## 2. Crear Usuario y Grupo NGINX

```bash
# Crear grupo nginx
sudo groupadd -r nginx

# Crear usuario nginx sin shell (sin home)
sudo useradd -r -g nginx -s /sbin/nologin -M nginx

# Verificar creación
id nginx
```

---

## 3. Compilación e Instalación de Nginx 1.31.x

```bash
# Crear directorio prefix
sudo mkdir -p /srv/nginx

# Descargar Nginx
cd /tmp
wget https://nginx.org/download/nginx-1.31.8.tar.gz
tar -xzf nginx-1.31.8.tar.gz
cd nginx-1.31.8

# Configurar compilación
./configure --prefix=/srv/nginx \
    --user=nginx \
    --group=nginx \
    --pid-path=/srv/nginx/logs/nginx.pid \
    --error-log-path=/srv/nginx/logs/error.log \
    --http-log-path=/srv/nginx/logs/access.log \
    --with-http_ssl_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_random_index_module \
    --with-http_secure_link_module \
    --with-http_stub_status_module \
    --with-http_auth_request_module \
    --with-file-aio \
    --with-http_v2_module

# Compilar e instalar
make -j$(nproc)
sudo make install

# Crear directorios necesarios
sudo mkdir -p /srv/nginx/{conf.d,logs,sites-enabled,html}
```

---

## 4. Configuración de Nginx

### Crear archivo principal

```bash
sudo nano /srv/nginx/conf/nginx.conf
```

### Contenido de nginx.conf

```nginx
user nginx;
worker_processes auto;
error_log /srv/nginx/logs/error.log;
pid /srv/nginx/logs/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /srv/nginx/conf/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /srv/nginx/logs/access.log main;

    sendfile on;
    keepalive_timeout 65;

    include /srv/nginx/conf/conf.d/*.conf;
}
```

### Configuración del sitio

```bash
sudo nano /srv/nginx/conf/conf.d/default.conf
```

```nginx
server {
    listen 8080;
    server_name localhost;
    root /srv/nginx/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include /srv/nginx/conf/fastcgi_params;
        fastcgi_pass unix:/tmp/php84.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

---

## 5. Servicio SystemD para Nginx

```bash
sudo nano /etc/systemd/system/nginx.service
```

```ini
[Unit]
Description=Nginx HTTP Server Compiled from Source
After=network.target

[Service]
Type=forking
User=nginx
Group=nginx
PIDFile=/srv/nginx/logs/nginx.pid
ExecStartPre=/srv/nginx/sbin/nginx -t
ExecStart=/srv/nginx/sbin/nginx
ExecReload=/srv/nginx/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

---

## 6. Compilación de PHP 8.4.x con FPM

```bash
# Descargar PHP
cd /tmp
wget https://www.php.net/distributions/php-8.4.3.tar.gz
tar -xzf php-8.4.3.tar.gz
cd php-8.4.3

# Configurar compilación
./configure --prefix=/srv/nginx \
    --with-config-file-path=/srv/nginx/etc \
    --enable-fpm \
    --with-fpm-user=nginx \
    --with-fpm-group=nginx \
    --enable-mbstring \
    --enable-intl \
    --enable-gd \
    --with-freetype \
    --with-jpeg \
    --with-webp \
    --enable-exif \
    --enable-zip \
    --with-curl \
    --with-openssl \
    --enable-sockets

# Compilar e instalar
make -j$(nproc)
sudo make install
```

---

## 7. Configuración de PHP-FPM

```bash
sudo mkdir -p /srv/nginx/etc/php-fpm.d

sudo cp php.ini-production /srv/nginx/etc/php.ini
sudo cp /srv/nginx/etc/php-fpm.conf.default /srv/nginx/etc/php-fpm.conf
sudo cp /srv/nginx/etc/php-fpm.d/www.conf.default /srv/nginx/etc/php-fpm.d/www.conf
```

### Configuración principal

```ini
[global]
pid = /srv/nginx/logs/php-fpm.pid
error_log = /srv/nginx/logs/php-fpm.log
include=/srv/nginx/etc/php-fpm.d/*.conf
```

### Pool www

```ini
[www]
user = nginx
group = nginx
listen = /tmp/php84.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

---

## 8. Servicio SystemD para PHP-FPM

```bash
sudo nano /etc/systemd/system/php-fpm8.4.service
```

```ini
[Unit]
Description=PHP 8.4 FastCGI Process Manager
After=network.target

[Service]
Type=forking
User=nginx
Group=nginx
PIDFile=/srv/nginx/logs/php-fpm.pid
ExecStart=/srv/nginx/sbin/php-fpm --fpm-config /srv/nginx/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID
ExecStop=/bin/kill -SIGINT $MAINPID
PrivateTmp=true
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

---

## 9. Script de Prueba PHP

```bash
sudo nano /srv/nginx/html/info.php
```

```php
<!DOCTYPE html>
<html>
<head>
    <title>Información de Servidor</title>
</head>
<body>
    <h1>Servidor Funcionando Correctamente</h1>

    <p><strong>Fecha y Hora:</strong>
    <?php echo date('Y-m-d H:i:s'); ?></p>

    <p><strong>GD:</strong>
    <?php echo extension_loaded('gd') ? 'Habilitado' : 'No disponible'; ?></p>

    <p><strong>Intl:</strong>
    <?php echo extension_loaded('intl') ? 'Habilitado' : 'No disponible'; ?></p>

    <?php phpinfo(); ?>
</body>
</html>
```

```bash
sudo chown -R nginx:nginx /srv/nginx/html
sudo chmod -R 755 /srv/nginx/html
```

---

## 10. Iniciar Servicios

```bash
sudo systemctl daemon-reload

sudo systemctl enable nginx.service
sudo systemctl enable php-fpm8.4.service

sudo systemctl start nginx
sudo systemctl start php-fpm8.4

sudo systemctl status nginx
sudo systemctl status php-fpm8.4
```

---

## 11. Verificar Funcionamiento

```bash
# Verificar socket
ls -la /tmp/php84.sock

# Probar desde terminal
curl http://localhost:8080/info.php
```

---

# Conclusiones

Se logró implementar exitosamente un servidor Nginx compilado desde código fuente en Alma Linux utilizando PHP-FPM mediante socket UNIX.

La comunicación entre Nginx y PHP-FPM mediante `/tmp/php84.sock` funcionó correctamente, permitiendo el procesamiento dinámico de páginas PHP.

Los servicios fueron configurados en SystemD para auto-arranque y administración centralizada.

La verificación mediante `phpinfo()` confirmó el correcto funcionamiento de las extensiones y módulos instalados.

---

# Bibliografía 

CentOS Community. (2019). *Systemd Service Files*.  
https://www.freedesktop.org/wiki/Software/systemd/

Nginx, Inc. (2024). *NGINX Documentation*.  
https://nginx.org/en/docs/

PHP Group. (2024). *PHP 8.4 Manual*.  
https://www.php.net/manual/en/

Red Hat, Inc. (2024). *AlmaLinux 9 - System Administrator's Guide*.  
https://docs.almalinux.org/

Systemd Team. (2023). *systemd.service Documentation*.  
https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
