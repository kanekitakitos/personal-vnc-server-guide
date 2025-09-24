# Guía para Configurar un Servidor VNC Personal y Seguro en Ubuntu

## Resumen

Esta guía ofrece instrucciones detalladas para configurar un servidor de escritorio remoto (VNC) en Ubuntu, con un fuerte enfoque en la seguridad. Es ideal para usuarios que desean acceder a su entorno gráfico de forma remota, ya sea mediante un cliente VNC tradicional o desde el navegador web, utilizando conexiones seguras como túneles SSH y certificados SSL.

## Diagrama del Flujo de Conexión y Puertos

```
      ┌─────────────────────────────┐                         
      │        Usuario Final        │                         
      └─────────────┬───────────────┘                         
                    │                         
                    ▼                         
      ┌─────────────────────────────┐                         
      │      Navegador Web          │                         
      │  (puerto local 8080/443)    │                         
      └─────────────┬───────────────┘                         
                    │                         
                    ▼                         
      ┌─────────────────────────────┐                         
      │        Internet             │                         
      └─────────────┬───────────────┘                         
                    │                         
                    ▼                         
      ┌─────────────────────────────┐                         
      │      Dominio público        │                         
      │   (puertos 80/443 TCP)      │                         
      └─────────────┬───────────────┘                         
                    │                         
                    ▼                         
      ┌─────────────────────────────┐                         
      │         Nginx               │                         
      │  (puertos 80/443 TCP)       │                         
      └─────────────┬───────────────┘                         
                    │                         
                    ▼                         
      ┌─────────────────────────────┐                         
      │         noVNC               │                         
      │   (puerto 8081 TCP)         │                         
      └─────────────┬───────────────┘                         
                    │                                                  
                    ▼                                                  
      ┌─────────────────────────────┐                         
      │         x11vnc              │                         
      │   (puerto 5900 TCP)         │                         
      └─────────────────────────────┘                         

Alternativamente, para clientes VNC tradicionales:

┌─────────────────────────────┐
│        Cliente VNC          │
│    (puerto local 5901)      │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│      Túnel SSH              │
│  (puerto 22 TCP)            │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│        x11vnc               │
│    (puerto 5900 TCP)        │
└─────────────────────────────┘
```

**Resumen de puertos utilizados:**
- **22 TCP**: SSH (túnel seguro)
- **5900 TCP**: Servidor VNC (x11vnc)
- **8081 TCP**: Proxy Websockify/noVNC
- **8080 TCP**: Proxy local para acceso web vía túnel SSH
- **80 TCP**: HTTP (Nginx, redirige a HTTPS)
- **443 TCP**: HTTPS (Nginx, acceso seguro desde navegador)

## Requisitos Previos

Antes de comenzar, asegúrate de tener lo siguiente:

- **Servidor**: Una instancia de Ubuntu (se recomienda versión 20.04 o superior).
- **Encaminamiento de Puertos (si es necesario)**: Si tu servidor está detrás de un router (por ejemplo, en una red doméstica), deberás configurar el encaminamiento de puertos. Para la alternativa con Nginx, redirige los puertos `80` y `443` del router hacia la IP local de tu servidor.
- **Dominio público**: Para acceso web seguro, necesitas un dominio apuntando a la IP pública de tu servidor.

## Archivos de Configuración

Este repositorio incluye los siguientes archivos listos para usar:

-   `x11vnc.service`: Archivo de servicio `systemd` para ejecutar `x11vnc` de forma robusta.
-   `novnc.service`: Archivo de servicio `systemd` para ejecutar el proxy `noVNC`.
-   `vnc.conf`: Archivo de configuración de Nginx para exponer `noVNC` mediante proxy inverso con SSL.

## Índice

- [Paso 1: Conexión Segura con Claves SSH](#paso-1-conexión-segura-con-claves-ssh-recomendado)
- [Paso 2: Instalar y Configurar el Servidor VNC](#paso-2-instalar-y-configurar-el-servidor-vnc-en-ubuntu)
- [Paso 3: Configurar el Firewall (UFW)](#paso-3-configurar-el-firewall-ufw)
- [Paso 4: Conexión Segura con Túnel SSH](#paso-4-conexión-segura-con-túnel-ssh)
- [Alternativa 1: Conectar con noVNC (vía Túnel SSH)](#alternativa-1-conectar-con-novnc-vía-túnel-ssh)
- [Alternativa 2: Acceso Web con Nginx y SSL](#alternativa-2-acceso-web-con-nginx-y-ssl)
- [Estructura de Archivos Esperada](#estructura-de-archivos-esperada)
- [Notas de Seguridad](#notas-de-seguridad)
- [Solución de Problemas](#solución-de-problemas)
- [Referencias y Enlaces Útiles](#referencias-y-enlaces-útiles)
- [Licencia](#licencia)

## Paso 1: Conexión Segura con Claves SSH (Recomendado)

(El contenido de esta sección no ha cambiado)

## Paso 2: Instalar y Configurar el Servidor VNC en Ubuntu

### Instalar el gestor de pantalla

```bash
sudo apt-get update
sudo apt-get install lightdm
```
Después de instalar, reinicia el servidor (`sudo reboot`).

### Instalar el servidor VNC (x11vnc)

```bash
sudo apt install x11vnc
```

### Crear y habilitar el servicio VNC

1.  **Crear el archivo de contraseña VNC**:
    Por seguridad, usa un archivo de contraseñas en vez de ponerla en el script.
    ```bash
    # Reemplaza "your_username" por tu nombre de usuario
    sudo -u your_username mkdir -p /home/your_username/.vnc
    x11vnc -storepasswd /home/your_username/.vnc/passwd
    sudo chown -R your_username:your_username /home/your_username/.vnc
    ```

2.  **Copiar y personalizar el archivo de servicio**:
    Copia el archivo `x11vnc.service` de este repositorio a `/etc/systemd/system/`.
    ```bash
    sudo cp ruta/a/tu/repo/x11vnc.service /etc/systemd/system/x11vnc.service
    ```
    Abre el archivo y **reemplaza todas las instancias de `your_username`** por tu usuario real.
    ```bash
    sudo nano /etc/systemd/system/x11vnc.service
    ```

3.  **Habilitar e iniciar el servicio**:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable x11vnc.service
    sudo systemctl start x11vnc.service
    sudo systemctl status x11vnc.service
    ```

## Paso 3: Configurar el Firewall (UFW)

```bash
sudo ufw allow ssh
sudo ufw enable
``` 
> **Importante**: No abras el puerto 5900 directamente en el firewall. La seguridad de este método se basa en que el VNC solo es accesible mediante túnel SSH o proxy Nginx (HTTPS). Abre los puertos 80 y 443 solo si usas la alternativa web.

## Paso 4: Conexión Segura con Túnel SSH

Este paso es necesario tanto para clientes de escritorio como para `noVNC` si no usas Nginx.

### Para Cliente VNC de Escritorio

En tu **máquina local**, crea un túnel que redirija el puerto `5901` local al `5900` del servidor.

```bash
ssh -L 5901:localhost:5900 your_username@server_ip
```

Luego, conecta tu cliente VNC a `localhost:5901`.

### Para noVNC (Alternativa 1)

En tu **máquina local**, crea un túnel que redirija el puerto `8080` local al `8081` del servidor.

```bash
ssh -L 8080:localhost:8081 your_username@server_ip
```

## Alternativa 1: Conectar con noVNC (vía Túnel SSH)

1.  **Clona noVNC en el servidor**:
    ```bash
    git clone https://github.com/novnc/noVNC.git ~/noVNC
    ```
2.  **Inicia el proxy Websockify en el servidor** (o usa el servicio `novnc.service`):
    ```bash
    ~/noVNC/utils/novnc_proxy --listen 8081 --vnc localhost:5900
    ```
3.  **Crea el túnel SSH** (ver paso 4).
4.  **Conecta desde el navegador**:
    Abre `http://localhost:8080/vnc.html` en tu navegador local.

## Alternativa 2: Acceso Web con Nginx y SSL

Esta configuración permite acceso web directo y seguro mediante una URL `https`.

### Requisitos Adicionales

-   Un **nombre de dominio** apuntando a la IP de tu servidor.
-   **Nginx** y **Certbot** instalados:
    ```bash
    sudo apt install nginx certbot python3-certbot-nginx
    ```

### 1. Obtener el Certificado SSL

```bash
sudo certbot --nginx -d your_domain.com
```

### 2. Configurar y habilitar los servicios

1.  **Servicio noVNC**: Copia el archivo `novnc.service` de este repositorio a `/etc/systemd/system/` y personalízalo.
    ```bash
    sudo cp ruta/a/tu/repo/novnc.service /etc/systemd/system/novnc.service
    sudo nano /etc/systemd/system/novnc.service # Reemplaza your_username
    ```
    Luego, habilítalo e inícialo:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable novnc.service
    sudo systemctl start novnc.service
    ```

2.  **Configuración de Nginx**: Copia el archivo `vnc.conf` a `/etc/nginx/sites-available/`.
    ```bash
    sudo cp ruta/a/tu/repo/vnc.conf /etc/nginx/sites-available/vnc.conf
    ```
    Edita el archivo y reemplaza `your_domain.com` por tu dominio y `your_username` en la ruta de `root`.
    ```bash
    sudo nano /etc/nginx/sites-available/vnc.conf
    ```

3.  **Habilita la configuración de Nginx**:
    ```bash
    sudo ln -s /etc/nginx/sites-available/vnc.conf /etc/nginx/sites-enabled/
    sudo nginx -t
    sudo systemctl restart nginx
    ```

4.  **Abre el puerto HTTPS en el firewall**:
    ```bash
    sudo ufw allow https
    ```
 
### 3. Conecta desde el Navegador

Accede a tu escritorio remoto de forma segura visitando `https://your_domain.com/vnc.html`.

## Estructura de Archivos Esperada

Después de seguir la guía, la estructura de archivos y directorios relevantes debería verse así:

```
/etc/
├── systemd/
│   └── system/
│       ├── x11vnc.service
│       └── novnc.service
├── nginx/
│   ├── sites-available/
│   │   └── vnc.conf
│   └── sites-enabled/
│       └── vnc.conf -> /etc/nginx/sites-available/vnc.conf
└── ufw/
    └── ufw.conf

/home/your_username/
├── .vnc/
│   └── passwd
└── noVNC/
    ├── ... (archivos de noVNC)
```

## Notas de Seguridad

- **Fail2Ban**: Instala y configura `fail2ban` para proteger tu servidor SSH contra ataques de fuerza bruta.
  ```bash
  sudo apt install fail2ban
  ```
- **Actualizaciones del Sistema**: Mantén tu sistema operativo y todos los paquetes actualizados regularmente.
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```
- **Restricción de IP**: Si tienes una IP estática, considera restringir el acceso SSH solo a tu IP en la configuración del firewall UFW.
  ```bash
  sudo ufw allow from TU_IP_ESTATICA to any port 22
  ```

## Solución de Problemas

- **Problemas de Autenticación**:
  - Asegúrate de que la contraseña VNC se haya creado correctamente y que el archivo `passwd` tenga los permisos correctos.
  - Verifica que el servicio `x11vnc.service` se esté ejecutando con el usuario correcto (`your_username`).
- **Puertos Ocupados**:
  - Si un servicio no inicia, revisa si el puerto que intenta usar ya está ocupado con `sudo lsof -i :PUERTO`.
- **Logs de Errores**:
  - Para `x11vnc`: `sudo journalctl -u x11vnc.service`
  - Para `noVNC`: `sudo journalctl -u novnc.service`
  - Para `Nginx`: `cat /var/log/nginx/error.log`

## Referencias y Enlaces Útiles

- **x11vnc**: [Documentación (man page)](https://www.karlrunge.com/x11vnc/x11vnc_opts.html)
- **noVNC**: [Repositorio en GitHub](https://github.com/novnc/noVNC)
- **Nginx**: [Documentación Oficial](https://nginx.org/en/docs/)
- **Certbot**: [Documentación Oficial](https://certbot.eff.org/docs/)
- **Fail2Ban**: [Sitio Oficial](https://www.fail2ban.org/wiki/index.php/Main_Page)

## Licencia

Este proyecto se distribuye bajo la Licencia MIT. Consulta el archivo `LICENSE` para más detalles.