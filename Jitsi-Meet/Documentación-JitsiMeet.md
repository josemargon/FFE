# Practica-Jitsi-Meet
## Introducción
El objetivo de esta práctica es la implementación un sistema de videoconferencia utilizando Jitsi Meet, un software libre que permite realizar reuniones en línea sin necesidad de cuentas o software adicional en el cliente. Esta práctica simula un escenario real en el que una empresa desea implantar su propia plataforma de videollamadas interna. 
## Entorno
Para realizar la práctica se utilizarán 3 máquinas virtuales instaladas todas ellas en hyper-V las cuales son las siguientes:
* Ubuntu Server 22.04 se utilizará como máquina anfitriona
* Ubuntu Desktop 22.04 se utilizará como máquina cliente
* Windows 10/11 se utilizará como máquina cliente
* Dominio público con certificado SSL
## Instalación y configuración
Instalaremos los SO de manera normal siguiendo los pasos que nos indiquen las propias instalaciones y una vez finalizadas, desde hyper-V les añadiremos un segundo adaptador de red, configurado para conexión interna.

![brr](/imagenes/Switch.png)
* ### Ubuntu Server 22.04
Para comenzar modificamos el archivo de configuración de red para tener conexión: /etc/netplan/01-netcfg.yaml

```
network:
  version: 2
  ethernets:
    eth0:
      addresses: [192.168.1.10/24]
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
    eth1:
      dhcp4: true
      dhcp4-overrides:
        route-metric: 100
```

Para trabajar mejor nos conectamos por ssh, para ello primero debemos instalarlo.
```
sudo apt install openssh-server
```
* ### Ubuntu Desktop 22.04
En este caso la red se ha configurado de igual manera que el Ubuntu Server.

```
network:
  version: 2
  ethernets:
    eth0:
      addresses: [192.168.1.20/24]
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
    eth1:
      dhcp4: true
      dhcp4-overrides:
        route-metric: 100
```

* ### Windows 10

* ### Dominio
Para tener un dominio con el que trabajar utilizaremos duckdns.org
![brr](imagenes/Jitsi/dominio.png)

## Instalación de Jitsi-meet
Una vez lo tenemos todo preparado podemos empezar a instalar todos los paquetes y repositorios necesarios para instalar Jitsi-Meet en nuestro Ubuntu Server.
Serán necesarios los paquetes gnupg2, nginx-full y curl.

```
sudo apt install gnupg2
sudo apt install nginx-full
sudo apt install curl
```

Para sistemas de ubuntu es necesario instalar un paquete de repositorio universal.

```
sudo apt-add-repository universe
```

Y hacemos un update para que se instale la última versión de cada paquete.

```
sudo apt update
```

* Configuramos el nombre de dominio para el host

```
sudo hostnamectl set-hostname jitsipractica.duckdns.org
```

También lo añadiremos en el archivo /etc/hosts
![brr](imagenes/Jitsi/host.png)

* Añadiremos el repositorio de prosody.

```
sudo curl -sL https://prosody.im/files/prosody-debian-packages.key -o /etc/apt/keyrings/prosody-debian-packages.key
echo "deb [signed-by=/etc/apt/keyrings/prosody-debian-packages.key] http://packages.prosody.im/debian $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/prosody-debian-packages.list
sudo apt install lua5.2
```

* Añadimos el repositorio de Jitsi.

```
curl -sL https://download.jitsi.org/jitsi-key.gpg.key | sudo sh -c 'gpg --dearmor > /usr/share/keyrings/jitsi-keyring.gpg'
echo "deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/" | sudo tee /etc/apt/sources.list.d/jitsi-stable.list
```

* Configuración de puertos

Configuraremos los siguientes puertos en el firewall.
80 TCP=> Para la verificación/renovación del certificado SSL con Let's Encrypt.

443 TCP=> Para acceder a Jitsi Meet.

10000 UDP=> Para reuniones de audio y video en red general.

22 TCPPara acceder a su servidor mediante SSH (cambie el puerto si no es el 22).

3478 UDP=> Para consultar el servidor de aturdimiento (coturn, opcional, necesita config.jscambio para habilitarlo).

5349 TCP=> Para comunicaciones de video/audio de red de respaldo sobre TCP (por ejemplo, cuando UDP está bloqueado), proporcionado por coturn.

```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 10000/udp
sudo ufw allow 22/tcp
sudo ufw allow 3478/udp
sudo ufw allow 5349/tcp
sudo ufw enable
```

Instalamos Jitsi-Meet y seguimos los pasos que se nos van mostrando, cuando pida tipo de certificado seleccionaremos la opción de certificado auto-firmado.

```
sudo apt install jitsi-meet
```

* Configuración de Prosody

Ahora configuraremos el modo de acceso a las reuniones con prosody accediendo a este archivo. "/etc/prosody/conf.avail/jitsipractica.duckdns.org.cfg.lua".
Para habilitar la autenticación dentro del apartado VirtualHost añdiremos lo siguiente.

```
VirtualHost "jitsipractica.duckdns.org"
    authentication = "internal_hashed"
```

Para habilitar el inicio de sesión anonimo para invitados, a continuación añadimos lo siguiente.

```
VirtualHost "guest.jitsipractica.duckdns.org"
    authentication = "anonymous"
    c2s_require_encryption = false
```

* Configuración de Jitsi-Meet

Modificar el siguiente arechivo /etc/jitsi/meet/jitsipractica.duckdns.org-config.js

```
var config = {
    hosts: {
            domain: 'jitsipractica.duckdns.org',
            anonymousdomain: 'guest.jitsipractica.duckdns.org',
            ...
        },
        ...
}
```

* Configuración de Jicofo

Dentro del archivo de configuración de jicofo /etc/jitsi/jicofo/jicofo.conf añadimos una nueva sección de autenticación donde especificaremos el dominio. 

```
jicofo {
  authentication: {
    enabled: true
    type: XMPP
    login-url: jitsi-meet.example.com
 }
 ...
```

* Creación de usuarios en Prosody

Para crear usuarios ejecutaremos lo siguiente.

```
sudo prosodyctl register <username> jitsi-meet.example.com <password>
```

Y reiniciaremos todo.

```
systemctl restart prosody
systemctl restart jicofo
systemctl restart jitsi-videobridge2
```

## Personalización de Jitsi-meet

* Instalación de npm y code

Ejecutamos lo siguiente para descargar lal ultimas versiones de npm y node, que los utilizaremos para poder trabajar con java script.

```
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt-get update
sudo apt-get install nodejs -y
```

* Clonar Jitsi-meet

```
cd ~
git clone https://github.com/jitsi/jitsi-meet.git
cd ~/jitsi-meet/
npm install
```

* Clonar lib-jitsi-meet y vincularlo con jitsi-meet

  * Clonar
 
```
cd ~
git clone https://github.com/jitsi/lib-jitsi-meet.git
```

  * Eliminar archivo actual


```
sudo rm -R ~/jitsi-meet/node_modules/lib-jitsi-meet
```

  * Crear enlace simbolico

```
ln -s ~/lib-jitsi-meet ~/jitsi-meet/node_modules/lib-jitsi-meet
```

  * Actualizar y crear lib-jitsi-meet

```
cd ~/lib-jitsi-meet
npm update
npm run build
```

* Construir Jitsi-meet

**IMPORTANTE TENER MÁS DE 8GB DE RAM**

```
cd ~/jitsi-meet/
make
```

* Configurar nginx para usar la carpeta local

Modificamos el archivo /etc/nginx/sites-available/jitsipractica.duckdns.org.conf

```
server
{
    listen jitsipractica.duckdns.org:443 ssl http2;
    server_name jitsipractica.duckdns.org;
    ...
    ssl_certificate /etc/ssl/jitsipractica.duckdns.org.crt;
    ssl_certificate_key /etc/ssl/jitsipractica.duckdns.org.key;

    # Comment the next line out, so you can revert later, if needed
    #root /usr/share/jitsi-meet;
    
    # Add this new line below
    root /home/jose/jitsi-meet;
...
}
```

* Reiniciar nginx

```
sudo service nginx restart
```

