# Configuración de Email en Producción

## Variables de Entorno Necesarias

Añadir las siguientes variables al servicio `backend` en `docker-compose.yml`:

```yaml
services:
  backend:
    image: eclipse-temurin:21-jre-alpine
    container_name: acainfo-backend
    restart: unless-stopped
    working_dir: /app
    volumes:
      - ./backend:/app
      - ./storage/materials:/app/materials
    command: java -jar app.jar
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/acainfodb
      SPRING_DATASOURCE_USERNAME: acainfo
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
      # Email Configuration (NUEVAS)
      GMAIL_USERNAME: ${GMAIL_USERNAME}
      GMAIL_APP_PASSWORD: ${GMAIL_APP_PASSWORD}
      APP_BASE_URL: https://acadeinfo.com
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
```

## Archivo .env en el Servidor

Crear o editar el archivo `.env` en `/opt/acainfo/`:

```bash
# Base de datos
DB_PASSWORD=tu_password_db

# JWT
JWT_SECRET=tu_secret_jwt

# Email (NUEVAS)
GMAIL_USERNAME=tu-email@gmail.com
GMAIL_APP_PASSWORD=xxxx-xxxx-xxxx-xxxx
```

## Cómo Obtener el App Password de Gmail

1. Ir a https://myaccount.google.com/
2. Seguridad → Verificación en 2 pasos (activar si no está)
3. Seguridad → Contraseñas de aplicaciones
4. Seleccionar "Correo" y "Otro (nombre personalizado)"
5. Escribir "AcaInfo" como nombre
6. Copiar la contraseña de 16 caracteres generada

## Aplicar Cambios

```bash
cd /opt/acainfo
docker-compose down
docker-compose up -d
```

## Verificar que Funciona

Revisar logs del backend:
```bash
docker logs acainfo-backend -f
```

Buscar líneas como:
- `Verification email sent to: usuario@red.ujaen.es`
- O errores de conexión SMTP si hay problemas
