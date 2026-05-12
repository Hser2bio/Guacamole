# Guacamole

## 1. Inicializar la base de datos

Arranca **solo MariaDB**:

```bash
docker compose up -d db
```

Genera el schema:

```bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > initdb.sql
```

Importa el schema en MariaDB:

```bash
docker exec -i guac-db mysql -uroot -pCAMBIA_ROOT_PASS guacamole_db < initdb.sql
```

---

## 2. Configurar SSH solo local

Edita la configuración SSH:

```bash
sudo nano /etc/ssh/sshd_config
```

Añade o modifica:

```text
ListenAddress 127.0.0.1
```

Reinicia SSH:

```bash
sudo systemctl restart ssh
```

Comprueba que escucha solo en localhost:

```bash
ss -tulpn | grep :22
```

Debe aparecer:

```text
127.0.0.1:22
```

⚠️ **No debe aparecer `0.0.0.0:22`.**

---

## 3. Certificado HTTPS

### Opción rápida (self-signed)

```bash
mkdir certs

openssl req -x509 -nodes -days 3650 \
-newkey rsa:4096 \
-keyout certs/key.pem \
-out certs/cert.pem
```

---

## 4. Configuración NGINX

Crea el archivo:

```bash
nano nginx.conf
```

Contenido:

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {

    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    server {

        listen 443 ssl http2;

        ssl_certificate /certs/cert.pem;
        ssl_certificate_key /certs/key.pem;

        location / {

            proxy_pass http://guacamole:8080/guacamole/;

            proxy_http_version 1.1;

            proxy_buffering off;

            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_read_timeout 86400;

        }
    }
}
```

---

## 5. Arrancar todo

```bash
docker compose up -d
```

---

## 6. Acceder

Abre en el navegador:

```text
https://IP_DEL_SERVIDOR
```

### Login inicial

```text
usuario: guacadmin
password: guacadmin
```

---

## 7. Crear conexión SSH local

Dentro de Guacamole:

- **New Connection**
- **Protocol:** SSH
- **Hostname:** `host.docker.internal`

---

## 8. Si Linux no resuelve `host.docker.internal`

Usa la IP del bridge Docker:

```bash
ip addr show docker0
```

Normalmente será:

```text
172.17.0.1
```
