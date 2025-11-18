# Guía Completa de Instalación y Configuración de GenieACS en Debian 12 Bookworm
Autor: AnthoAnton Fecha: 24 de octubre de 2025

## Introducción 

El Cliente TR-069 (Technical Report 069) es una implementación del CWMP (CPE WAN Management Protocol) que permite la gestión centralizada de dispositivos de usuario final. El ACS (Auto-Configuration Server) es el sistema utilizado para monitorizar, configurar y actualizar el firmware de estos dispositivos remotos, siendo GenieACS una implementación de código abierto de este sistema.

Esta guía detalla el proceso de instalación y configuración de GenieACS en Debian 12 Bookworm.

## Requisitos
Sistema operativo: Debian 12 Bookworm (instalación limpia).

# Pasos de Instalación y Configuración
## 1. Configuración de Repositorios

Edita el archivo de repositorios para añadir contrib y non-free:

```
# nano /etc/apt/sources.list
```

Asegúrate de que tus líneas se vean así (añadiendo contrib non-free):


```
deb http://deb.debian.org/debian/ bookworm main non-free-firmware contrib non-free
deb-src http://deb.debian.org/debian/ bookworm main non-free-firmware contrib non-free

deb http://security.debian.org/debian-security bookworm-security main non-free-firmware contrib non-free
deb-src http://security.debian.org/debian-security bookworm-security main non-free-firmware contrib non-free

deb http://deb.debian.org/debian/ bookworm-updates main non-free-firmware contrib non-free
deb-src http://deb.debian.org/debian/ bookworm-updates main non-free-firmware contrib non-free
```

Actualiza el repositorio e instala firmware si es necesario:

```
# apt update
# apt upgrade -y 
# apt install firmware-linux firmware-linux-free firmware-linux-nonfree -y
# reboot 
```
Reinicia para cargar nuevos módulos del kernel.

## 2. Instalación de Paquetes Requeridos

Instala paquetes esenciales:

```
# apt -y install gnupg gnupg2 curl wget rsyslog
```

## 3. Instalación de MongoDB

Configura el repositorio de MongoDB (Versión 7.0):

```
# curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
  http://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" \
  > /etc/apt/sources.list.d/mongodb-org-7.0.list

```

Recarga el repositorio e instala MongoDB:

```
# apt update
# apt -y install mongodb-org
```

Habilita e inicia el servicio, y verifica su estado:

```
# systemctl enable mongod
# systemctl start mongod
# systemctl status mongod
```

## 4. Instalación de Node.js y NPM
Instala Node.js y npm:

```
# apt -y install nodejs npm --no-install-recommends
# apt -y install node-mongodb
```

## 5. Instalación de GenieACS

Busca el paquete para verificar su existencia:

```
# npm search -g genieacs
```
Instala GenieACS globalmente:

```
# npm install -g genieacs
```

## 6. Configuración de Usuario y Directorios

Crea un usuario de sistema para ejecutar los demonios de GenieACS:

```
# useradd --system --no-create-home --user-group genieacs
```

Crea el directorio de configuración y extensiones:

```
# mkdir /opt/genieacs
# mkdir /opt/genieacs/ext
```

## 7. Archivo de Configuración de Entorno

Crea el archivo /opt/genieacs/genieacs.env para las variables de entorno:

```
# nano /opt/genieacs/genieacs.env
```
Añade las siguientes líneas:

```
GENIEACS_CWMP_ACCESS_LOG_FILE=/var/log/genieacs/genieacs-cwmp-access.log
GENIEACS_NBI_ACCESS_LOG_FILE=/var/log/genieacs/genieacs-nbi-access.log
GENIEACS_FS_ACCESS_LOG_FILE=/var/log/genieacs/genieacs-fs-access.log
GENIEACS_UI_ACCESS_LOG_FILE=/var/log/genieacs/genieacs-ui-access.log
GENIEACS_DEBUG_FILE=/var/log/genieacs/genieacs-debug.yaml
NODE_OPTIONS=--enable-source-maps
GENIEACS_EXT_DIR=/opt/genieacs/ext
```

Genera y añade un secreto JWT seguro:

```
# node -e "console.log(\"GENIEACS_UI_JWT_SECRET=\" + require('crypto').randomBytes(128).toString('hex'))" \
  >> /opt/genieacs/genieacs.env

```

Establece la propiedad y los permisos del archivo:

```
# chown genieacs: /opt/genieacs -R
# chmod 600 /opt/genieacs/genieacs.env
```

Crea el directorio de logs y establece su propiedad:

```
# mkdir /var/log/genieacs
# chown genieacs: /var/log/genieacs
```

## 8. Configuración de Servicios Systemd

Crea las unidades de servicio para los cuatro demonios de GenieACS:

genieacs-cwmp

```
# systemctl edit --force --full genieacs-cwmp
```

Contenido:

```
[Unit]
Description=GenieACS CWMP
After=network.target

[Service]
User=genieacs
EnvironmentFile=/opt/genieacs/genieacs.env
ExecStart=/usr/local/bin/genieacs-cwmp

[Install]
WantedBy=default.target
```

genieacs-nbi

```
# systemctl edit --force --full genieacs-nbi
```

Contenido:

```
[Unit]
Description=GenieACS NBI
After=network.target

[Service]
User=genieacs
EnvironmentFile=/opt/genieacs/genieacs.env
ExecStart=/usr/local/bin/genieacs-nbi

[Install]
WantedBy=default.target
```

genieacs-fs

```
# systemctl edit --force --full genieacs-fs
```

Contenido:

```
[Unit]
Description=GenieACS FS
After=network.target

[Service]
User=genieacs
EnvironmentFile=/opt/genieacs/genieacs.env
ExecStart=/usr/local/bin/genieacs-fs

[Install]
WantedBy=default.target
```

genieacs-ui

```
# systemctl edit --force --full genieacs-ui
```

Contenido:

```
[Unit]
Description=GenieACS UI
After=network.target

[Service]
User=genieacs
EnvironmentFile=/opt/genieacs/genieacs.env
ExecStart=/usr/local/bin/genieacs-ui

[Install]
WantedBy=default.target
```

## 9. Configuración de Logrotate

Crea un archivo de configuración para la rotación de logs:

```
# nano /etc/logrotate.d/genieacs
```

Contenido:

```
/var/log/genieacs/*.log /var/log/genieacs/*.yaml {
  daily
  rotate 30
  compress
  delaycompress
  dateext
}
```

## 10. Finalización y Verificación

Recarga el demonio de systemd, habilita, inicia y verifica el estado de los servicios:

```
# systemctl daemon-reload
# systemctl enable genieacs-cwmp genieacs-nbi genieacs-fs genieacs-ui
# systemctl start genieacs-cwmp genieacs-nbi genieacs-fs genieacs-ui
# systemctl status genieacs-cwmp genieacs-nbi genieacs-fs genieacs-ui --no-pager
```
Todos los servicios deberían mostrar un estado Active: active (running).

# Acceso a la Interfaz de Usuario:

Una vez que todos los servicios estén activos, puedes acceder a la interfaz web de GenieACS en tu navegador:

URL de Acceso: http://IP:3000

Credenciales de Acceso:

Usuario: admin

Contraseña: admin

# Pasos para agregar seguridad al api (Token)

ZIP: [nbi authentication](https://github.com/markabrahams/genieacs/archive/refs/heads/nbi-authentication.zip)

```
# mkdir nbi-authentication
```

```
# cd nbi-authentication
```
```
# wget https://github.com/markabrahams/genieacs/archive/refs/heads/nbi-authentication.zip
```
```
# unzip genieacs-nbi-authentication.zip
```
```
# cd genieacs-nbi-authentication
```
```
# npm i
```
```
# npm run build
```
```
# cp -r /dist/*  /usr/local/lib/node_modules/genieacs/
```

```
# node -e "console.log(\"GENIEACS_NBI_AUTHENTICATION_KEY=\" + require('crypto').randomBytes(128).toString('hex'))" \
  >> /opt/genieacs/genieacs.env

```

```
# sudo systemctl restart genieacs-nbi
```
```
# sudo systemctl status genieacs-nbi
```

# Cambiar puertos y host al servidor GenieACS
```
# sudo nano /opt/genieacs/genieacs.env
```
Contenido:
```
GENIEACS_UI_INTERFACE=127.0.0.1
GENIEACS_CWMP_INTERFACE=127.0.0.1
GENIEACS_NBI_INTERFACE=127.0.0.1
GENIEACS_FS_INTERFACE=127.0.0.1
GENIEACS_UI_PORT=3001
GENIEACS_CWMP_PORT=7541
GENIEACS_NBI_PORT=7551
GENIEACS_FS_PORT=7561

```
```
# sudo systemctl restart genieacs-ui genieacs-cwmp genieacs-nbi genieacs-fs
```

# Configuración Nginx

Instalar nginx


```
# sudo apt update
# sudo apt install nginx
# sudo systemctl status nginx
# sudo nano /etc/nginx/sites-available/acsempresa.conf

```
Contenido:
```
# ----------------------------------------------------
# 0. Redirección HTTP (Puerto 443) a HTTPS (Puerto 3001)
# ----------------------------------------------------
server {
    listen 443 ssl;
    server_name acsempresa.dominio.com;

    # Usa las rutas de tus certificados
    ssl_certificate     /etc/nginx/ssl/acsempresa.dominio.com/fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/acsempresa.dominio.com/certificado.key;

    # [Otras configuraciones SSL aquí]

    location / {
        # El puerto 443 redirige al backend de la UI en el puerto 3000 local
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Necesario para WebSockets
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# ----------------------------------------------------
# 1. Redirección HTTP (Puerto 80) a HTTPS (Puerto 3000)
# ----------------------------------------------------
server {
    listen 80;
    server_name acsempresa.dominio.com;

    # Redirige todo el tráfico HTTP al puerto de la UI en HTTPS (3000)
    location / {
        return 301 https://$host:3000$request_uri;
    }
}

# ----------------------------------------------------
# 2. Interfaz de Usuario (UI) - Puerto 3000
# URL: https://acsempresa.dominio.com:3000
# ----------------------------------------------------
server {
    listen 3000 ssl;
    server_name acsempresa.dominio.com;

    ssl_certificate     /etc/nginx/ssl/acsempresa.dominio.com/fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/acsempresa.dominio.com/certificado.key;

    # Configuración de seguridad SSL recomendada
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://127.0.0.1:3001; # Proxy a genieacs-ui
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Necesario para la funcionalidad de WebSockets de GenieACS UI
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# ----------------------------------------------------
# 3. CWMP, NBI y File Server (Puertos 7547, 7557, 7567)
# ----------------------------------------------------
# Usaremos un bloque para cada uno para mayor claridad y control.

# CWMP (TR-069) - Puerto 7547
server {
    listen 7547 ssl;
    server_name acsempresa.cominio.com;

    ssl_certificate     /etc/nginx/ssl/acsempresa.dominio.com/fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/acsempresa.dominio.com/certificado.key;

    location / {
        proxy_pass http://127.0.0.1:7541; # Proxy a genieacs-cwmp
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# NBI (NorthBound Interface/API) - Puerto 7557
server {
    listen 7557 ssl;
    server_name acsempresa.dominio.com;

    ssl_certificate     /etc/nginx/ssl/acsempresa.dominio.com/fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/acsempresa.dominio.com/certificado.key;

    location / {
        proxy_pass http://127.0.0.1:7551; # Proxy a genieacs-nbi
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# File Server (Firmware, etc.) - Puerto 7567
server {
    listen 7567 ssl;
    server_name acsempresa.dominio.com;

    ssl_certificate     /etc/nginx/ssl/acsempresa.dominio.com/fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/acsempresa.dominio.com/certificado.key;

    location / {
        proxy_pass http://127.0.0.1:7561; # Proxy a genieacs-fs
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}


```

```
# sudo ln -s /etc/nginx/sites-available/acsempresa.conf /etc/nginx/sites-enabled/
# sudo rm /etc/nginx/sites-enabled/default
# sudo nginx -t
# sudo systemctl reload nginx
```

# Nota sobre Problemas de MongoDB: 

Si utilizas máquinas virtuales (como en Proxmox o VMware) y tienes problemas para iniciar MongoDB 5.0 o superior, es posible que tu CPU no soporte las instrucciones AVX/AVX2. En ese caso, se recomienda:

Cambiar el tipo de CPU de la VM a "host" (en Proxmox).

Si lo anterior no funciona, instalar una versión anterior de MongoDB (por ejemplo, 4.4, que no requiere AVX).

Fuente: [GenieACS](https://genieacs.com/)
