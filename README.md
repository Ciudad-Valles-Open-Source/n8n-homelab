# Instalación de n8n & postgres con docker y nginx usando NO-IP

Esta breve guía esta pensada para utilizarla en un Ubuntu Server, pero puedes adaptar los pasos a cualquier sistema operativo.

Si observas puntos a mejorar, puedes ayudarme con un PR

## Aclaraciones

Hay algunas fallas con esto, la más común es el no encontrar algunos `assets`, pues se buscan en la carpeta raíz, o redireccionarte a un path innexistente (basta con borrar el `N8N_PATH` que se puso demás), esto se soluciona si en lugar de usar el `N8N_PATH` se usa el `N8N_HOST` pero a este se le configurá un subdominio (ej, n8n.tu-sit.io) y en el NGINX en lugar de poner la redirección al path, se configurá la redirección al subdominio. Si tienes idea de como solucionarlo usando `N8N_PATH` o solucionar alguna otra falla que pueda existir, me encantaría que compartas tu solución con un PR.

¿Por qué se usa `N8N_PATH`? Si usas de forma gratuita NO-IP, te darás cuenta que no puedes usar subdominios (o almenos yo no tengo conocimiento de como usarlos con un dominio dns gratuito) y esta fue la forma que encontré para que funcionará.

## Configuración de NO-IP

Puedes guiarte viendo este [vídeo](https://www.youtube.com/watch?v=UNeTOHD21Vc) o siguiendo cualquier otra guía que hay en internet.

## Instalación de docker

Ejecuta los siguientes comandos en tu terminal:

```sh
# Dependencias básicas
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Crear carpeta de llaves
sudo mkdir -p /etc/apt/keyrings

# Agregar la llave GPG oficial de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Agregar el repositorio estable
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Actualizar repositorios
sudo apt-get update

# Instalar Docker CE + CLI + Containerd + Docker Compose plugin
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Habilitar e iniciar servicio
sudo systemctl enable docker
sudo systemctl start docker

# Comprobar instalación (Si esta activo, todo esta OK)
sudo systemctl status docker

# Extra, para evitar usar sudo en cada comando, se puede realizar lo siguiente:
sudo usermod -aG docker $USER
# Así el usuario actual tendrá permisos para ejecutar los comandos docker
```

## Configuración de docker-compose

Archivo ´docker-compose.yaml´:
```yaml
version: "3.8"

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n_server
    restart: unless-stopped
    ports:
      # Expone el puerto 5678 de n8n solo a localhost en el host.
      # Nginx (en el host) se conectará a esto.
      - "127.0.0.1:5678:5678"
    environment:
      # --- Conexión a Postgres ---
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres # El nombre del servicio de db
      - DB_POSTGRESDB_PORT=${POSTGRES_PORT}
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      
      # --- Configuración de n8n ---
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - N8N_PATH=${N8N_PATH}
      - N8N_WEBHOOK_URL=https://${N8N_HOST}${N8N_PATH}
	    - VUE_APP_URL_BASE_API=https://${N8N_HOST}${N8N_PATH}
      
      # --- Seguridad ---
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_AUTH_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_SECURE_COOKIE=true # Requerido para HTTPS
      
      # --- Zona Horaria (Ajusta si es necesario) ---
      - TZ=America/Mexico_City
	    # --- Telemetry (opcional)
	    - N8N_DIAGNOSTICS_ENABLED=false
      - N8N_PERSONALIZATION_ENABLED=false

    volumes:
      # Usamos la ruta del host donde pondremos los archivos de n8n (opcional).
      - /ruta/a/lugar/en/el/host/fisico/n8n_data:/home/node/.n8n
    networks:
      - n8n_net
    depends_on:
      postgres: # n8n no arrancará hasta que postgres esté listo
        condition: service_healthy

  postgres:
    image: postgres:18
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      # Usamos un volumen nombrado de Docker para los datos de la DB
      - pgdata:/var/lib/postgresql
    networks:
      - n8n_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
    driver: local

networks:
  n8n_net:
    driver: bridge
```

Variables de Entorno necesarias (archivo `.env`):
```dotenv
# --- Credenciales de Postgres ---
POSTGRES_USER=DB_USER
POSTGRES_PASSWORD=DB_PASSWORD
POSTGRES_DB=DB_NAME
POSTGRES_PORT=POSTGRES_PORT

# --- Credenciales de n8n Admin ---
N8N_AUTH_USER=N8N_USER
N8N_AUTH_PASSWORD=N8N_PASSWORD
N8N_HOST=N8N_HOST #-- Aquí solo va el dominio sin el protocolo (http o https), example.sytes.net, por ejemplo
N8N_PATH=N8N_PATH #-- En caso de tener algo en NGINX previamente, se asignará una ruta para N8N, también puedes omitir esto usando subdominios, pero necesitas tener un dominio en NO-IP para poder gestionarlo.

# --- Clave de Encriptación de n8n ---
N8N_ENCRYPTION_KEY=N8N_SECRET_KEY
```

El `N8N_ENCRYPTION_KEY` se puede generar con el comando: `openssl rand -hex 32` o poner los caracteres que se quieran.

## Configuración de NGINX

### Proceso previo

Verificar que en el router/modem esten los puertos 80 (HTTP) y 443 (HTTPS) abiertos y en el servidor se deben abrir también:
```sh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```
> Nota: No es necesario abrir el puerto `5678` en el firewall ni en el router/modem. NGINX (el host) es el único que necesitará acceder a él, y lo hace a través de `127.0.0.1` (localhost).

### Configuración de archivos NGINX

En el archivo `/etc/nginx/sites-available/default` (o tu archivo equivalente donde tengas la configuración de tu servidor) se deberá tener algo como lo siguiente:

```nginx
# Default server config
server {
  listen 80;
  server_name tu.server.name;
  return 301 https://$host$request_uri;
}

server {
  server_name tu.server.name;
  #SSL Config
  listen 443 ssl default_server;
  listen [::]:443 ssl default_server;

  ssl_certificate /root/to/cert.pam;
  ssl_certificate_key /root/to/cert.key;

  # ... Toda la configuración que tengas (si no tienes, puedes añadir una landing page), esto es opcional:
  location / {
    root /root/to/landing/page;
    index index.html;
    try_files $uri $uri/ =404;
  }

  # Sección para n8n
	location /n8n/ {
		proxy_pass http://127.0.0.1:5678/;
		proxy_http_version 1.1;

		# Soporte para WebSockets (GUI de n8n)
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";

		# Cabeceras necesarias
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;

		# Tiempos de espera
		proxy_read_timeout 3600s;
		proxy_send_timeout 3600s;
	}
}
```

Al terminar la edición, se deberá hacer un enlace simbólico de la siguiente manera:

```sh
sudo ln -sf /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
```

Una vez realizado, se prueba los archivos, se reinicia NGINX y se levanta el contenedor:

```sh
# Si el comando anterior dice: "syntax is ok"  y "test is successful", está listo.
sudo nginx -t

# Para reiniciar Nginx
sudo systemctl reload nginx
```

## Ejecución del contenedor

Por último, se debe ejecutar el comando `docker-compose up -d` para levantar el contenedor

## Extras

### Comandos docker útiles

```sh
# Verificar la versión del docker
docker compose version

# Ver todos los contenedores del docker
docker ps -a

# Ver los logs del contenedor
docker-compose logs -f <nombre de app>
# Ejemplo
# docker-compose logs -f postgres

# Bajar el contenedor:
docker-compose down

# Subir el contenedor (forzar la recreación)
docker-compose up -d --force-recreate
```
