# Implementación de NGINX y PHP-FPM desde Código Fuente

## Carátula

**Institución:** Tecnológico de Estudios Superiores del Oriente del Estado de México (TESOEM)  
**Materia:** Administración de Servidores Linux  
**Proyecto:** Implementación de servidor NGINX y PHP-FPM compilados desde código fuente  
**Alumno(s):** [Rodriguez Morales Angel Natahel]  
**Docente:** [Gustavo Moises Romero Gonzalez]  
**Fecha:** [25/05/26]  


---

# Objetivo General

Implementar un servidor web NGINX compilado desde código fuente junto con PHP-FPM versión 8.4.x en un sistema Linux, configurando la comunicación FastCGI mediante socket UNIX y gestionando ambos servicios con SystemD para garantizar su correcto funcionamiento y arranque automático.

---

# Objetivos Específicos

- Compilar e instalar NGINX versión 1.31.x desde código fuente.
- Configurar el prefijo de instalación en `/srv/nginx`.
- Crear y configurar el usuario y grupo del sistema `NGINX`.
- Implementar el servicio NGINX utilizando SystemD.
- Configurar el arranque automático del servicio.
- Compilar e instalar PHP versión 8.4.x desde código fuente.
- Habilitar soporte `php-fpm` para comunicación con NGINX.
- Configurar procesamiento de imágenes, fechas e internacionalización (`intl`).
- Configurar comunicación FastCGI mediante socket UNIX.
- Validar el funcionamiento mediante un script PHP.

---

# Desarrollo del Proyecto

# 1. Preparación del Sistema

Primero se actualizó el sistema operativo e instalaron las dependencias necesarias para la compilación.

```bash
sudo dnf update && sudo apt upgrade -y
```

## Instalación de dependencias

```bash
sudo dnf install -y build-essential gcc g++ make wget curl git \
libpcre3 libpcre3-dev zlib1g zlib1g-dev libssl-dev \
libxml2-dev libsqlite3-dev libcurl4-openssl-dev \
libjpeg-dev libpng-dev libwebp-dev libfreetype6-dev \
libonig-dev libzip-dev libicu-dev pkg-config autoconf \
bison re2c libxslt1-dev
```

---

# 2. Creación del Usuario y Grupo NGINX

Se creó el usuario y grupo del sistema que ejecutará el servicio NGINX.

```bash
sudo groupadd --system NGINX
sudo useradd --system --gid NGINX --shell /usr/sbin/nologin \
--home-dir /nonexistent NGINX
```

---

# 3. Descarga y Compilación de NGINX

## Descarga del código fuente

```bash
cd /usr/local/src
sudo wget https://nginx.org/download/nginx-1.31.0.tar.gz
sudo tar -xvzf nginx-1.31.0.tar.gz
cd nginx-1.31.0
```

## Configuración y compilación

```bash
sudo ./configure \
--prefix=/srv/nginx \
--user=NGINX \
--group=NGINX \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_stub_status_module
```

## Compilar e instalar

```bash
sudo make
sudo make install
```

## Verificación de instalación

```bash
/srv/nginx/sbin/nginx -v
```

---

# 4. Configuración del Servicio SystemD para NGINX

Se creó el archivo del servicio:

```bash
sudo nano /etc/systemd/system/nginx.service
```

## Contenido del archivo

```ini
[Unit]
Description=NGINX Web Server
After=network.target

[Service]
Type=forking
PIDFile=/srv/nginx/logs/nginx.pid
ExecStartPre=/srv/nginx/sbin/nginx -t
ExecStart=/srv/nginx/sbin/nginx
ExecReload=/srv/nginx/sbin/nginx -s reload
ExecStop=/srv/nginx/sbin/nginx -s quit
User=NGINX
Group=NGINX

[Install]
WantedBy=multi-user.target
```

## Habilitar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable nginx
sudo systemctl start nginx
```

## Verificación del servicio

```bash
sudo systemctl status nginx
```

---

# 5. Descarga y Compilación de PHP 8.4.x

## Creación del usuario PHP

```bash
sudo groupadd --system PHP
sudo useradd --system --gid NGINX --shell /usr/sbin/nologin \
--home-dir /nonexistent PHP
```

## Descarga del código fuente

```bash
cd /usr/local/src
sudo wget https://www.php.net/distributions/php-8.4.0.tar.gz
sudo tar -xvzf php-8.4.0.tar.gz
cd php-8.4.0
```

## Configuración de compilación

```bash
sudo ./configure \
--prefix=/srv/nginx/php \
--enable-fpm \
--with-fpm-user=PHP \
--with-fpm-group=NGINX \
--with-openssl \
--with-zlib \
--enable-mbstring \
--with-curl \
--with-pdo-mysql \
--with-mysqli \
--with-jpeg \
--with-webp \
--with-freetype \
--enable-gd \
--enable-intl
```

## Compilar e instalar

```bash
sudo make
sudo make install
```

---

# 6. Configuración de PHP-FPM

## Copiar archivos de configuración

```bash
cd /srv/nginx/php/etc
sudo cp php-fpm.conf.default php-fpm.conf
sudo cp php-fpm.d/www.conf.default php-fpm.d/www.conf
```

## Configuración del socket UNIX

Editar el archivo:

```bash
sudo nano /srv/nginx/php/etc/php-fpm.d/www.conf
```

Modificar:

```ini
user = PHP
group = NGINX

listen = /tmp/php84.sock
listen.owner = NGINX
listen.group = NGINX
listen.mode = 0660
```

---

# 7. Configuración del Servicio SystemD para PHP-FPM

Crear el archivo:

```bash
sudo nano /etc/systemd/system/php-fpm8.4.service
```

## Contenido del archivo

```ini
[Unit]
Description=PHP 8.4 FastCGI Process Manager
After=network.target

[Service]
Type=simple
ExecStart=/srv/nginx/php/sbin/php-fpm --nodaemonize
ExecReload=/bin/kill -USR2 $MAINPID
User=PHP
Group=NGINX

[Install]
WantedBy=multi-user.target
```

## Habilitar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable php-fpm8.4
sudo systemctl start php-fpm8.4
```

## Verificación del servicio

```bash
sudo systemctl status php-fpm8.4
```

---

# 8. Configuración FastCGI entre NGINX y PHP-FPM

Editar el archivo principal de configuración de NGINX:

```bash
sudo nano /srv/nginx/conf/nginx.conf
```

## Configuración del servidor

```nginx
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout 65;

    server {
        listen 80;
        server_name localhost;

        root /srv/nginx/html;
        index index.php index.html;

        location / {
            try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
            fastcgi_pass unix:/tmp/php84.sock;
            fastcgi_index index.php;
            include fastcgi.conf;
        }
    }
}
```

## Reiniciar servicios

```bash
sudo systemctl restart nginx
sudo systemctl restart php-fpm8.4
```

---

# 9. Validación del Funcionamiento

## Crear archivo PHP de prueba

```bash
sudo nano /srv/nginx/html/phpinfo.php
```

## Contenido del archivo

```php
<?php
phpinfo();
?>
```

## Acceso desde navegador

```text
http://IP_DEL_SERVIDOR/phpinfo.php
```

Al acceder desde un navegador se mostró correctamente la página de información de PHP, validando la correcta comunicación entre NGINX y PHP-FPM mediante FastCGI usando socket UNIX.

---

# Evidencias

## Verificación de NGINX

```bash
systemctl status nginx
```

## Verificación de PHP-FPM

```bash
systemctl status php-fpm8.4
```

## Verificación del Socket UNIX

```bash
ls -l /tmp/php84.sock
```

---

# Conclusiones

Durante el desarrollo de este proyecto se logró implementar correctamente un servidor web NGINX compilado desde código fuente junto con PHP-FPM versión 8.4.x. Se configuró la comunicación FastCGI mediante socket UNIX, permitiendo una comunicación eficiente y segura entre ambos servicios.

Además, se implementó la administración de servicios mediante SystemD, logrando el arranque automático del servidor durante el inicio del sistema operativo. La compilación desde código fuente permitió personalizar completamente las opciones de instalación y habilitar módulos específicos necesarios para el funcionamiento del servidor.

Finalmente, se comprobó el correcto funcionamiento mediante la ejecución de un archivo `phpinfo.php`, demostrando que NGINX puede procesar correctamente archivos PHP utilizando PHP-FPM.

---

# Bibliografía

- NGINX. (2026). *NGINX Official Documentation*. Recuperado de https://nginx.org/en/docs/
- PHP Documentation Group. (2026). *PHP Manual*. Recuperado de https://www.php.net/manual/en/
- Linux Foundation. (2026). *SystemD Documentation*. Recuperado de https://www.freedesktop.org/wiki/Software/systemd/
- Shotts, W. E. (2019). *The Linux Command Line* (2nd ed.). No Starch Press.
- Nemeth, E., Snyder, G., Hein, T., & Whaley, B. (2017). *UNIX and Linux System Administration Handbook* (5th ed.). Pearson.

