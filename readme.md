# Guía para Configurar un Servidor VNC Personal y Seguro en Ubuntu

## Resumen

Esta guía ofrece instrucciones detalladas para configurar un servidor de escritorio remoto (VNC) en Ubuntu, con un fuerte enfoque en la seguridad. Es ideal para usuarios que desean acceder a su entorno gráfico de forma remota, ya sea mediante un cliente VNC tradicional o desde el navegador web, utilizando conexiones seguras como túneles SSH y certificados SSL.

## Diagrama del Flujo de Conexión y Puertos

<div align="center">
  <table>
    <thead>
      <tr>
        <th style="text-align:center;">**Acceso Web (Nginx/noVNC)**</th>
        <th style="text-align:center;">**Acceso con Cliente VNC (Túnel SSH)**</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td style="text-align:center;">
          <pre>
    ┌─────────────────────────────┐
    │       Usuario Final         │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │       Navegador Web         │
    │   (puerto local 8080/443)   │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │         Internet            │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │     Dominio público         │
    │   (puertos 80/443 TCP)      │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │         Nginx               │
    │   (puertos 80/443 TCP)      │
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
          </pre>
        </td>
        <td style="text-align:center;">
          <pre>
┌─────────────────────────────┐
│       Cliente VNC           │
│   (puerto local 5901)       │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│       Túnel SSH             │
│   (puerto 22 TCP)           │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│         x11vnc              │
│   (puerto 5900 TCP)         │
└─────────────────────────────┘
          </pre>
        </td>
      </tr>
    </tbody>
  </table>
</div>

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
Para aumentar la seguridad, se recomienda deshabilitar la autenticación por contraseña y usar claves SSH.

> **Nota para usuarios de Windows**: Los comandos de esta sección son para terminales tipo Unix (Linux, macOS). Si usas Windows, necesitarás una herramienta como **Git Bash** (que viene con Git para Windows) o el **Subsistema de Windows para Linux (WSL)** para ejecutar estos comandos.



1.  **En tu máquina local**, genera un par de claves SSH si aún no tienes uno:
    ```bash
    ssh-keygen -t rsa -b 4096
    ```

2.  Copia tu clave pública al servidor (reemplaza `your_username` y `server_ip`):
    El comando `ssh-copy-id` automatiza la copia de tu clave pública (`~/.ssh/id_rsa.pub`) al archivo `~/.ssh/authorized_keys` en el servidor.
    ```bash
    ssh-copy-id your_username@server_ip
    ```
    **Si `ssh-copy-id` no está disponible** (común en Windows), puedes hacerlo manualmente. Ejecuta este comando desde tu máquina local:
    ```bash
    # Para Linux/macOS/Git Bash
    cat ~/.ssh/id_rsa.pub | ssh your_username@server_ip "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
    ```

3.  Ahora deberías poder iniciar sesión en tu servidor sin necesidad de una contraseña.

> **Nota**: Este paso es fundamental para la seguridad. Una vez que confirmes que puedes iniciar sesión con tu clave, considera deshabilitar la autenticación por contraseña en el archivo `/etc/ssh/sshd_config` de tu servidor.

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
    Asegúrate de que el directorio `.vnc` exista y luego crea el archivo de contraseña.
    ```bash
    x11vnc -storepasswd /home/your_username/.vnc/passwd
    # Reemplaza "your_username" por tu nombre de usuario
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
    > **Importante sobre la ruta `-auth`**: La ruta `-auth /run/user/1000/gdm/Xauthority` puede variar según tu sistema o gestor de pantalla. Si el servicio falla, puedes encontrar la ruta correcta iniciando sesión como tu usuario en el servidor y ejecutando `echo $XAUTHORITY`. Si eso no funciona, usa `find /var/run/ -name "*Xauthority*" 2>/dev/null` para localizar el archivo.

    El contenido del archivo `x11vnc.service` proporcionado en este repositorio es el siguiente:
    - **`ExecStart`**: Define el comando que se ejecutará.
        - `-display :0`: Conecta al escritorio principal.
        - `-auth`: Especifica la ruta de autenticación para el gestor de pantalla.
        - `-rfbauth`: Apunta al archivo de contraseña que creaste.
        - `-rfbport 5900`: Define el puerto en el que escuchará VNC.
        - `-forever -shared`: Mantiene la sesión activa y permite múltiples conexiones.
    - **`User`**: Asegura que el servicio se ejecute con los permisos de tu usuario, lo cual es crucial para acceder al escritorio gráfico.
    - **`Restart=always`**: Reinicia el servicio automáticamente si falla.


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
> **Importante**: No abras el puerto `5900` directamente en el firewall. La seguridad de este método se basa en que el VNC solo es accesible a través del túnel SSH o del proxy Nginx (HTTPS). Si tu firewall ya está activo, solo necesitas asegurarte de que el puerto 22 (SSH) esté permitido.

## Paso 4: Conexión Segura con Túnel SSH

Este es el método principal y más seguro para conectar. El túnel encripta todo el tráfico VNC entre tu máquina local y el servidor.

### Para Cliente VNC de Escritorio

En tu **máquina local**, crea un túnel que redirija el puerto `5901` local al `5900` del servidor.

```bash
ssh -L 5901:localhost:5900 your_username@server_ip
```
-   **`-L 5901:localhost:5900`**: Redirige las conexiones al puerto `5901` de tu máquina local (`localhost`) hacia el puerto `5900` en el servidor.

Mantén abierta la terminal donde ejecutaste este comando. Ahora, abre tu cliente VNC (como Remmina, VNC Viewer, etc.) y conéctate a la siguiente dirección:

-   **Servidor**: `localhost:5901`
-   **Contraseña**: La que creaste en el paso 2.

Tu conexión VNC ahora es segura y está completamente encriptada a través de SSH.

<a>
    <img src="https://github.com/kanekitakitos/personal-vnc-server-guide/blob/main/gif/parte_1.gif">
</a>

## Alternativa 1: Acceso Web con noVNC (vía Túnel SSH)

Este método te permite acceder a tu escritorio VNC desde un navegador web, manteniendo la seguridad de un túnel SSH.

**¿Cómo funciona?** El servidor VNC (`x11vnc`) usa un protocolo (RFB) que los navegadores no entienden. La solución es usar `noVNC`, que incluye un proxy (`websockify`) para "traducir" el tráfico VNC a WebSockets, un protocolo compatible con la web. La conexión seguirá siendo segura porque todo el tráfico pasará a través de nuestro túnel SSH.

### 1. Instalar noVNC en el Servidor

Clona el repositorio oficial de noVNC en el directorio personal de tu usuario.
```bash
git clone https://github.com/novnc/noVNC.git /home/your_username/noVNC
```

### 2. Crear y Habilitar el Servicio noVNC

Para asegurar que el proxy de noVNC se ejecute automáticamente, crearemos un servicio `systemd`.

1.  **Copia y personaliza el archivo de servicio**:
    Copia el archivo `novnc.service` de este repositorio a `/etc/systemd/system/`.
    ```bash
    sudo cp ruta/a/tu/repo/novnc.service /etc/systemd/system/novnc.service
    ```
    Abre el archivo y **reemplaza `your_username`** por tu usuario real.
    ```bash
    sudo nano /etc/systemd/system/novnc.service
    ```
    - **`ExecStart`**: Ejecuta el proxy. Es importante usar la ruta completa (`/home/your_username/...`) porque `systemd` no interpreta el atajo `~`.
        - `--listen 8081`: El proxy escucha en el puerto interno `8081`.
        - `--vnc localhost:5900`: El proxy se conecta al servidor `x11vnc` en el puerto `5900`.

2.  **Habilita e inicia el servicio**:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable novnc.service
    sudo systemctl start novnc.service
    sudo systemctl status novnc.service
    ```

### 3. Conectar desde tu Máquina Local

1.  **Crea el túnel SSH**:
    En tu **máquina local**, crea un túnel que redirija el puerto `8080` local al `8081` del servidor (donde escucha noVNC).
    ```bash
    ssh -L 8080:localhost:8081 your_username@server_ip
    ```

2.  **Accede desde el navegador**:
    Mantén la terminal del túnel abierta y visita la siguiente URL en tu navegador:
    `http://localhost:8080/vnc.html`

<a>
    <img src="https://github.com/kanekitakitos/personal-vnc-server-guide/blob/main/gif/parte_2.gif">
</a>

## Alternativa 2: Acceso Web con Nginx y SSL

Esta configuración permite un acceso web directo y más profesional usando un dominio y un certificado SSL (`https`), eliminando la necesidad de crear un túnel SSH para cada conexión.

**¿Cómo funciona?** Nginx actuará como un **proxy inverso**. Escuchará en los puertos web estándar (80 para HTTP y 443 para HTTPS) y redirigirá de forma segura el tráfico al servicio `noVNC` que se ejecuta internamente en el puerto `8081`. Un certificado SSL de Let's Encrypt cifrará toda la comunicación entre el navegador del usuario y tu servidor.

### Requisitos Adicionales

-   Un **nombre de dominio** apuntando a la IP de tu servidor.
-   **Nginx** y **Certbot** instalados:
    ```bash
    sudo apt install nginx certbot python3-certbot-nginx -y
    ```

### 1. Configurar el Servicio noVNC

Si aún no lo has hecho en la "Alternativa 1", instala y habilita el servicio `noVNC`. Este servicio es necesario para que Nginx tenga un destino al cual redirigir el tráfico.

1.  **Instala noVNC**:
    ```bash
    git clone https://github.com/novnc/noVNC.git /home/your_username/noVNC
    ```
2.  **Habilita el servicio `systemd`**: Sigue los pasos de la **Alternativa 1, sección 2** para copiar, personalizar y habilitar `novnc.service`.

### 2. Configurar Nginx y Obtener Certificado SSL

1.  **Copia el archivo de configuración de Nginx**:
    ```bash
    sudo cp ruta/a/tu/repo/vnc.conf /etc/nginx/sites-available/vnc.conf
    ```
2.  **Edita el archivo** y reemplaza `your_domain.com` por tu dominio y `your_username` en la ruta de `root`.
    ```bash
    sudo nano /etc/nginx/sites-available/vnc.conf
    ```
3.  **Habilita la nueva configuración** creando un enlace simbólico:
    ```bash
    sudo ln -s /etc/nginx/sites-available/vnc.conf /etc/nginx/sites-enabled/
    ```
4.  **Obtén el certificado SSL**:
    Usa Certbot, que detectará la configuración de Nginx que acabas de crear, obtendrá el certificado y modificará `vnc.conf` automáticamente para usarlo.
    ```bash
    sudo certbot --nginx -d your_domain.com
    ```
5.  **Verifica y reinicia Nginx**:
    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```

### 3. Configurar Firewall y Router

Para que el tráfico de Internet llegue a Nginx, debes abrir los puertos web.

1.  **Abrir puertos en el Firewall UFW**:
    ```bash
    sudo ufw allow 'Nginx Full'
    # Esto es un atajo para abrir los puertos 80 (HTTP) y 443 (HTTPS)
    ```
2.  **Configurar el Encaminamiento de Puertos (Port Forwarding) en tu Router**:
    Este es un paso crucial si tu servidor está en una red doméstica. Accede a la interfaz de administración de tu router y crea dos reglas:
    -   **Regla 1 (HTTP)**: Redirige el tráfico del puerto externo `80` al puerto `80` de la IP local de tu servidor.
    -   **Regla 2 (HTTPS)**: Redirige el tráfico del puerto externo `443` al puerto `443` de la IP local de tu servidor.

### 4. Conectar desde el Navegador

¡Listo! Ahora puedes acceder a tu escritorio remoto de forma segura desde cualquier lugar visitando tu dominio en un navegador. La conexión se establecerá automáticamente a través de HTTPS.

`https://your_domain.com/vnc.html`

<a>
    <img src="https://github.com/kanekitakitos/personal-vnc-server-guide/blob/main/gif/parte_3.gif">
</a>

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

- **x11vnc**: [Documentación (man page)](https://github.com/LibVNC/x11vnc)
- **noVNC**: [Repositorio en GitHub](https://github.com/novnc/noVNC)
- **Nginx**: [Documentación Oficial](https://nginx.org/en/docs/)
- **Certbot**: [Documentación Oficial](https://certbot.eff.org/)
- **Fail2Ban**: [Sitio Oficial](https://www.fail2ban.org/wiki/index.php/Main_Page)

## Licencia

Este proyecto se distribuye bajo la Licencia MIT. Consulta el archivo `LICENSE` para más detalles.