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

## 9. Configuración de Log rotate

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

## Nota sobre Problemas de MongoDB: 

Si utilizas máquinas virtuales (como en Proxmox o VMware) y tienes problemas para iniciar MongoDB 5.0 o superior, es posible que tu CPU no soporte las instrucciones AVX/AVX2. En ese caso, se recomienda:

Cambiar el tipo de CPU de la VM a "host" (en Proxmox).

Si lo anterior no funciona, instalar una versión anterior de MongoDB (por ejemplo, 4.4, que no requiere AVX).

Fuente: [GenieACS](https://genieacs.com/)
