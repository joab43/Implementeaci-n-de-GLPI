# Plataforma de Gestión de tickets en un entorno virtual

---

Para el desarrollo de este proyecto desplegaremos una LEMP stack en un servidor Ubuntu 22.04LTS en el cual se integrara la plataforma GLPI para la gestión de tickets.

#### Requisitos Previos

- Ubuntu 22.04 o 24.04 LTS

- LEMP stack
  
  - Linux
  
  - Nginx
  
  - MariaDB
  
  - PHP

- GLPI

---

### Intalación de la base de datos con MariaDB

Una ven instalado MariaDB con el anterior comando, pasaremos a su configuración.

```bash
sudo apt install mariadb-server mariadeb-client
```

#### Paso 1 - Habilitar el servicio

`sudo systemctl enable mariadb-server`

```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb

sudo systemctl status mariadb
```

#### Paso 2 - Configuración de MariaDB

```bash
sudo mysql_secure_installation
```

> Responderemos las preguntas necesarias en el script de instalación de la base de datos de la siguiente forma.

- Set root password? -> *yes*. `Ingresa una contraseña para el usuario root de la base de datos`

- switch to unix_socket authentication -> yes

- Change the root password -> no

- Remove anonymous users? -> *yes*

- Disallow root login? -> *yes*

- Reload privilege tables -> *yes*

#### Paso 3 - Conexión con la base de datos

```bash
sudo mysql -u root -p
```

#### Paso 4 - Creación de las base de datos de GLPI

```sql
CREATE DATABASE glpidb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY 'tu_contraseña_segura';
GRANT ALL PRIVILEGES ON glpidb.* TO 'glpiuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Instalación de GLPI

```bash
cd /tmp
```

```bash
wget https://github.com/glpi-project/glpi/releases/download/10.0.15/glpi-10.0.15.tgz
tar -xvzf glpi-10.0.15.tgz
```

```bash
sudo mv glpi /var/www/
sudo chown -R www-data:www-data /var/www/glpi
sudo chmod -R 755 /var/www/glpi
```

### PHP

Ahora pasaremos con la instalación de PHP y todos los paquetes necesarios para el despliegue de GLPI.

```bash
sudo apt install php php-fpm php-mysql php-curl php-gd php-intl php-mbstring php-xml php-bz2 php-zip php-ldap php-apcu php-cli php-common php-opcache -y
```

> Esto instalara la versión 8.1 de PHP la cual es la versión por defecto para Ubuntu 22.04.



### Nginx

Pasaremos  a la configuración de Nginx con los siguientes pasos.

```bash
sudo apt install nginx
```

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

También habilitaremos el servicio en el firewall de ubuntu.

```bash
sudo ufw enable
sudo ufw allow 'Nginx full'
```

#### Paso 1 - Configuración de Nginx para GLPI

Primero crearemos el archivo de configuración de Nginx

```bash
sudo nano /etc/nginx/sites-available/glpi
```

El contenido de este archivo será el siguiente

```nginx
server {
    listen 80;
    server_name tu_dominio_o_ip; #direccion ip del servidor

    root /var/www/glpi; # ruta de el archivo que despliega la interfas de GLPI
    index index.php index.html;

    access_log /var/log/nginx/glpi_access.log;
    error_log /var/log/nginx/glpi_error.log;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock; # Ajustar a versión de PHP
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Seguridad para carpeta files/
    location ~ ^/files/ {
        deny all;
        return 403;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot|otf)$ {
        expires max;
        log_not_found off;
    }
}
```

#### Paso 2 - Activar el sitio Web

Guardaremos los cambios en el archivo, luego generaremos un enlace entre los siguientes directorios de la siguiente forma.

```bash
sudo ln -s /etc/nginx/sites-available/glpi /etc/nginx/sites-enabled/glpi
```

Por si acaso eliminaremos el directorio */default* para que nginx muestre el portal de GLPI.

```bash
sudo rm /etc/nginx/sites-enabled/default 
```

Comprobamos que la configuración de Nginx es la correcta

```bash
sudo nginx -t
```

Una vez comprobado que las configuraciones son correctas recargamos el servicio de Nginx

```bash
sudo systemctl reload nginx
```

#### Paso 3 - Ajustar Permisos Adicionales

```bash
sudo chmod -R g+w /var/www/glpi/files
sudo chmod -R g+w /var/www/glpi/config
```

### Instalación de GLPI desde la interfaz Web

Pasaremos a acceder a la interfaz web de GLPI con la ip de nuestro servidor.

Abriremos el navegador e ingresaremos en buscador la URL

`http://ip_del_servidor` veremos el portal de GLPI tal que así:

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-13-23-35-41-Screenshot_20250713_233522.png)

En el menú desplegable seleccionaremos el idioma español  y haremos click en el botón de `ok`.



En la siguiente pantalla veremos información de la licencia de GLPI.

Le daremos click en el botón`Continuar`.

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-13-23-39-36-Screenshot_20250713_233845.png)

Ahora nos mostrara la pantalla del inicio de la instalación, donde seleccionaremos el botón de instalar.

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-13-23-41-13-Screenshot_20250713_234104.png)

Nos moverá a una pantalla donde se realizara la comprobación de ciertos requisitos necesarios para la instalación de GLPI.

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-13-23-44-30-image.png)

Veremos que habrán tres requisitos de seguridad que no se cumplen, ahora pasaremos a corregir uno de ellos para posteriormente después de la instalación corregir los dos restantes.



#### Corrección de seguridad #1

Para corregir el problema de seguridad `Security configuration for sessions` haremos lo siguiente.

Pasaremos a editar el fichero `php.ini`

```bash
sudo nano /etc/php/8.1/fpm/php.ini
```

Y buscaremos la linea `session.cookie_httponly` y la dejaremos de la siguiente forma.

```ini
session.cookie_httponly = On
```

Guardaremos y luego reiniciaremos el servicio `php8.1-fpm`

```bash
sudo systemctl restart php8.1-fpm
```

### Verificación de Requisitos de GLPI

Ahora volveremos a verificar si se cumplió el requisito de seguridad.

Para esto, daremos click en el botón `Reintentar`

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-13-23-53-29-image.png) 

Ahora vemos que efectivamente se cumplió con el requisito de seguridad de GLPI.

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-13-23-55-06-image.png)

Ahora daremos click en el botón `Continuar` para empezar con la instalación.

Pasaremos a configurar la conexión a la base de datos de esta forma, llenando los campos solicitados

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-13-23-58-19-image.png)

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-14-00-03-21-image.png)

| Campo         | Valor Sugerido |
|:------------- | -------------- |
| Replica SQL   | localhost      |
| Usuario SQL   | glpiuser       |
| Password SQL  | contraseña     |
| Base de datos | glpidb         |

Después de haber rellenado los campos y seleccionado la base de datos, daremos click en `Continuar`.



Nos debe aparecer esta pantalla de confirmación de inicialización de la base de datos.

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-14-00-05-33-image.png)

Daremos click en `Continuar` hasta llegar a la ultima pantalla donde daremos click en `Utilizar GLPI`.

![](/home/yorch/.var/app/com.github.marktext.marktext/config/marktext/images/2025-07-14-00-07-05-image.png)
