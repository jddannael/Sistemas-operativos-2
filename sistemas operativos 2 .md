# Tecnológico de Estudios Superiores del Oriente del Estado de México (TESOEM)

## Reporte Técnico: Compilación y Configuración de Servidores Web Dedicados

---

### Datos Generales
* **Institución:** Tecnológico de Estudios Superiores del Oriente del Estado de México
* **División:** Ingeniería en Sistemas Computacionales
* **Asignatura:** TALLER DE SISTEMAS OPERATIVOS
* **Nombre de integrantes:**
* 1-Gonzales Robledo Daniela
* 2-Lopez Velazquez Angélica 
* 3-Salas Vilchis Josué Dannael

* **Profesores:** Gustavo Moises Romero Gonzalez
* **Fecha:** Mayo 2026

---

## 1. Objetivo General
Implementar un entorno de servidor web modular, seguro y de alto rendimiento en el sistema operativo AlmaLinux OS 10 mediante la compilación desde código fuente de NGINX y PHP 8.4, interconectados localmente a través de un Socket UNIX para eliminar la sobrecarga de red y garantizar la persistencia del entorno frente a reinicios del sistema.

## 2. Objetivos Específicos
* Compilar NGINX desde su código fuente distribuidor, centralizando la instalación en el Prefix `/srv/nginx` bajo la administración del usuario y grupo del sistema `nginx`.
* Compilar PHP versión 8.4 desde su código fuente, habilitando el Administrador de Procesos FastCGI (`php-fpm`) de manera integrada en el mismo Prefix `/srv/nginx`.
* Incorporar soporte nativo para el procesamiento de imágenes (`gd`, `jpeg`, `webp`, `freetype`) e internacionalización y manejo de formatos de fecha (`intl`) en la compilación de PHP.
* Configurar una comunicación FastCGI basada estrictamente en un Socket UNIX en la ruta `/tmp/php84.sock`, mitigando el uso innecesario de puertos TCP locales.
* Automatizar el inicio y la gestión de la infraestructura mediante la creación y registro de Slices de Servicio de SystemD (`nginx.service` y `php-fpm8.4.service`) integrados en el `multi-user.target`.
* Diagnosticar y resolver restricciones de seguridad y permisos de ejecución derivados de SELinux (error de SystemD `status=203/EXEC`) configurando un entorno permisivo persistente.

---

## 3. Desarrollo del Proyecto

### A. Preparación de Dependencias del Sistema
Para garantizar que los scripts de configuración (`./configure`) no fallaran por la ausencia de librerías de desarrollo en AlmaLinux 10, se ejecutó el siguiente comando global de preparación, asegurando la activación del repositorio CRB para las dependencias avanzadas:

```bash
sudo dnf update -y
sudo dnf config-manager --set-enabled crb
sudo dnf groupinstall "Development Tools" -y
sudo dnf install -y epel-release wget pcre-devel pcre2-devel zlib-devel openssl-devel \
    libxml2-devel sqlite-devel libpng-devel libjpeg-turbo-devel freetype-devel \
    libwebp-devel libcurl-devel libffi-devel libicu-devel systemd-devel oniguruma-devel
```
Se crearon las cuentas de servicio del sistema restrictivas y sin acceso a shell (/sbin/nologin):
```bash
sudo groupadd nginx
sudo useradd -g nginx -s /sbin/nologin -r nginx
sudo useradd -g nginx -s /sbin/nologin -r php
```
Compilación e Instalación de NGINX
Se descargó el código fuente estable de NGINX, se asignó el Prefix de instalación en la ruta solicitada y se ejecutó la compilación:
```bash
cd /usr/src
sudo wget [http://nginx.org/download/nginx-1.26.0.tar.gz](http://nginx.org/download/nginx-1.26.0.tar.gz)
sudo tar -xzvf nginx-1.26.0.tar.gz
cd nginx-1.26.0
sudo ./configure \
    --prefix=/srv/nginx \
    --user=nginx \
    --group=nginx \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-threads \
    --with-file-aio
sudo make && sudo make install
```
Compilación e Instalación de PHP 8.4
Se descargó y extrajo la versión 8.4.1 de PHP, activando el soporte para FPM, las credenciales del pool, y las extensiones obligatorias de imágenes (gd) e internacionalización (intl):
```bash
cd /usr/src
sudo wget [https://www.php.net/distributions/php-8.4.1.tar.gz](https://www.php.net/distributions/php-8.4.1.tar.gz)
sudo tar -xzvf php-8.4.1.tar.gz
cd php-8.4.1
sudo ./configure \
    --prefix=/srv/nginx \
    --mandir=/srv/nginx/man \
    --enable-fpm \
    --with-fpm-user=php \
    --with-fpm-group=nginx \
    --with-mysqli \
    --with-pdo-mysql \
    --with-openssl \
    --with-zlib \
    --with-curl \
    --enable-gd \
    --with-jpeg \
    --with-webp \
    --with-freetype \
    --enable-intl \
    --enable-mbstring \
    --enable-soap \
    --enable-calendar
sudo make -j4 && sudo make install
```
Configuración de los Servicios e Interconexión
Se configuró el archivo del pool principal de PHP-FPM (/srv/nginx/etc/php-fpm.d/www.conf) para inicializar el pool [www] y escuchar peticiones mediante el socket local:
```bash
[www]
user = php
group = nginx
listen = /tmp/php84.sock
listen.owner = php
listen.group = nginx
listen.mode = 0660
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```
En el archivo /srv/nginx/conf/nginx.conf se direccionó la interpretación de scripts PHP hacia el descriptor del socket UNIX:

Nginx
```bash
location ~ \.php$ {
    fastcgi_pass   unix:/tmp/php84.sock;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include        fastcgi_params;
}
```
Resolución de Mitigaciones de Seguridad (SELinux)
Al reiniciar la máquina virtual bajo el nuevo núcleo de AlmaLinux 10, se detectó que el servicio fallaba con el código de error status=203/EXEC.

Diagnóstico: Las políticas restrictivas nativas de SELinux en AlmaLinux 10 bloquearon el entorno de ejecución debido a que los binarios se encuentran en una ruta personalizada (/srv/nginx) no indexada en los contextos de seguridad por defecto del sistema.

Para solucionarlo de forma definitiva se configuró el estado de SELinux en modo permisivo editando el archivo /etc/selinux/config

```bash
SELINUX=permissive
```
Posteriormente, se recargaron los daemons de SystemD, se habilitó el auto-arranque y se levantaron los servicios en el sistema:
```bash
sudo systemctl daemon-reload
sudo systemctl enable nginx.service php-fpm8.4.service
sudo systemctl start php-fpm8.4.service nginx.service
```

# 4. Conclusiones
La compilación de infraestructura de servidores a partir de su código fuente original representa una ventaja competitiva crítica en la administración de sistemas, permitiendo omitir módulos innecesarios y optimizar el rendimiento térmico y de procesamiento del hardware real o virtualizado. La implementación del esquema de comunicación mediante FastCGI sobre Sockets UNIX, en sustitución de los puertos TCP loopback tradicionales, reduce drásticamente los tiempos de respuesta y la sobrecarga del stack de red interna del núcleo de Linux. Finalmente, el proceso de resolución de problemas con respecto a los bloqueos de ejecución aplicados por SELinux en AlmaLinux 10 subraya la necesidad de un control preciso sobre los permisos y contextos de seguridad del software en sistemas operativos empresariales de última generación basados en RHEL.

# 5. Bibliografía (Formato APA 7ma Edición)
AlmaLinux OS Foundation. (2026). AlmaLinux OS 10 Deployment and Configuration Guide. AlmaLinux Docs. https://docs.almalinux.org/

The Nginx Organization. (2026). Nginx Documentation: Core installation from sources. Nginx Enterprise. https://nginx.org/en/docs/install.html

The PHP Group. (2026). PHP Manual: FastCGI Process Manager (FPM) Configuration. PHP Documentation. https://www.php.net/manual/en/install.fpm.configuration.php
