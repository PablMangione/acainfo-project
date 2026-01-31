# Plan: Restricción de Dominios y Verificación de Email

> **Estado: IMPLEMENTADO** - Todas las fases han sido completadas.

## Objetivo
Implementar seguridad en el registro de estudiantes:
1. **Restricción de dominios**: Solo `@red.ujaen.es` o `@gmail.com`
2. **Verificación de email**: Correo de confirmación antes de activar cuenta

## Servicio SMTP: Gmail
- Usar Gmail SMTP con "App Password"
- Límite: 500 emails/día (suficiente para uso académico)

---

## Fase 1: Restricción de Dominios de Email

### 1.1 Nueva excepción
**Crear**: `back/src/main/java/com/acainfo/user/domain/exception/InvalidEmailDomainException.java`

### 1.2 Modificar AuthService.java
**Archivo**: `back/src/main/java/com/acainfo/user/application/service/AuthService.java`
- Añadir método `isAllowedDomain(String email)`
- Validar dominio antes de crear usuario
- Lanzar `InvalidEmailDomainException` si dominio no permitido

### 1.3 Handler de excepción
**Modificar**: `back/src/main/java/com/acainfo/shared/infrastructure/exception/GlobalExceptionHandler.java`
- Añadir handler para `InvalidEmailDomainException` → HTTP 400

---

## Fase 2: Infraestructura de Email

### 2.1 Dependencia Maven
**Modificar**: `back/pom.xml`
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

### 2.2 Configuración SMTP
**Modificar**: `back/src/main/resources/application.properties`
```properties
# Email Configuration
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=${GMAIL_USERNAME:}
spring.mail.password=${GMAIL_APP_PASSWORD:}
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true

# Verification settings
app.email.from=${GMAIL_USERNAME:noreply@acainfo.com}
app.email.verification.expiration-hours=24
app.email.verification.base-url=${APP_BASE_URL:http://localhost:5173}/verify-email
```

**Modificar**: `back/src/main/resources/application-dev.properties`
```properties
# Mock email in development (log instead of send)
app.email.mock=true
```

### 2.3 Propiedades de configuración
**Crear**: `back/src/main/java/com/acainfo/shared/infrastructure/config/EmailProperties.java`

### 2.4 Puerto de salida (arquitectura hexagonal)
**Crear**: `back/src/main/java/com/acainfo/shared/application/port/out/EmailSenderPort.java`
```java
public interface EmailSenderPort {
    void sendVerificationEmail(String to, String userName, String verificationLink);
}
```

### 2.5 Adaptador SMTP
**Crear**: `back/src/main/java/com/acainfo/shared/infrastructure/adapter/out/email/SmtpEmailAdapter.java`
- Implementar `EmailSenderPort`
- Usar `JavaMailSender`
- Construir email HTML con link de verificación
- Si `app.email.mock=true`, solo loguear (desarrollo)

---

## Fase 3: Sistema de Tokens de Verificación

### 3.1 Entidad EmailVerificationToken
**Crear**: `back/src/main/java/com/acainfo/security/verification/EmailVerificationToken.java`

Campos (siguiendo patrón de `RefreshToken.java`):
- `Long id`
- `Long userId`
- `String token` (UUID, unique)
- `LocalDateTime expiresAt`
- `LocalDateTime createdAt`
- `boolean used`

Tabla: `email_verification_tokens`

### 3.2 Repository
**Crear**: `back/src/main/java/com/acainfo/security/verification/EmailVerificationTokenRepository.java`
- `findByToken(String token)`
- `findValidToken(String token, LocalDateTime now)`
- `deleteByUserId(Long userId)`
- `deleteExpiredTokens(LocalDateTime now)`

### 3.3 Service
**Crear**: `back/src/main/java/com/acainfo/security/verification/EmailVerificationService.java`
- `createVerificationToken(Long userId)` → String token
- `validateToken(String token)` → EmailVerificationToken
- `markAsUsed(String token)`

### 3.4 Excepción de token inválido
**Crear**: `back/src/main/java/com/acainfo/security/verification/InvalidVerificationTokenException.java`

---

## Fase 4: Modificar Flujo de Registro

### 4.1 Nueva excepción
**Crear**: `back/src/main/java/com/acainfo/user/domain/exception/EmailNotVerifiedException.java`

### 4.2 Nuevo Use Case
**Crear**: `back/src/main/java/com/acainfo/user/application/port/in/VerifyEmailUseCase.java`
```java
public interface VerifyEmailUseCase {
    void verifyEmail(String token);
}
```

**Crear**: `back/src/main/java/com/acainfo/user/application/port/in/ResendVerificationUseCase.java`
```java
public interface ResendVerificationUseCase {
    void resendVerification(String email);
}
```

### 4.3 Modificar AuthService
**Archivo**: `back/src/main/java/com/acainfo/user/application/service/AuthService.java`

**Cambios en `register()`**:
1. Añadir validación de dominio (fase 1)
2. Cambiar `status` de `ACTIVE` a `PENDING_ACTIVATION`
3. Generar token de verificación
4. Enviar email de verificación

**Cambios en `authenticate()`**:
- Añadir validación: si `status == PENDING_ACTIVATION` → lanzar `EmailNotVerifiedException`

**Nuevos métodos**:
- `verifyEmail(String token)` - implementar `VerifyEmailUseCase`
- `resendVerification(String email)` - implementar `ResendVerificationUseCase`

### 4.4 Nuevos endpoints
**Modificar**: `back/src/main/java/com/acainfo/user/infrastructure/adapter/in/rest/AuthController.java`

```java
@GetMapping("/verify-email")
public ResponseEntity<MessageResponse> verifyEmail(@RequestParam String token)

@PostMapping("/resend-verification")
public ResponseEntity<MessageResponse> resendVerification(@RequestBody @Valid ResendVerificationRequest request)
```

### 4.5 DTOs nuevos
**Crear**: `back/src/main/java/com/acainfo/user/infrastructure/adapter/in/rest/dto/ResendVerificationRequest.java`
**Crear**: `back/src/main/java/com/acainfo/user/infrastructure/adapter/in/rest/dto/MessageResponse.java` (si no existe)

---

## Fase 5: Frontend

### 5.1 Modificar validación de registro
**Modificar**: `acainfo-front/src/features/auth/components/RegisterForm.tsx`
- Añadir validación de dominio en schema Zod
- Añadir texto informativo sobre dominios permitidos
- Redirigir a página de verificación pendiente tras registro exitoso

### 5.2 Nueva página: Verificación Pendiente
**Crear**: `acainfo-front/src/features/auth/pages/VerificationPendingPage.tsx`
- Mensaje: "Hemos enviado un email de verificación a tu correo"
- Instrucciones para revisar spam
- Botón: "Reenviar email de verificación"
- Link: "Volver al login"

### 5.3 Nueva página: Verificar Email
**Crear**: `acainfo-front/src/features/auth/pages/VerifyEmailPage.tsx`
- Ruta: `/verify-email?token=xxx`
- Leer token de URL
- Llamar API para verificar
- Mostrar resultado (éxito/error)
- Redirigir a login tras éxito

### 5.4 Actualizar tipos
**Modificar**: `acainfo-front/src/features/auth/types/auth.types.ts`
- Añadir tipos para nuevos endpoints

### 5.5 Actualizar API
**Modificar**: `acainfo-front/src/features/auth/services/authApi.ts`
```typescript
verifyEmail: (token: string) => apiClient.get(`/auth/verify-email?token=${token}`)
resendVerification: (email: string) => apiClient.post('/auth/resend-verification', { email })
```

### 5.6 Actualizar router
**Modificar**: `acainfo-front/src/app/router.tsx` (o equivalente)
- Añadir ruta `/verification-pending`
- Añadir ruta `/verify-email`

### 5.7 Modificar página de login
**Modificar**: `acainfo-front/src/features/auth/pages/LoginPage.tsx`
- Detectar error `EMAIL_NOT_VERIFIED`
- Mostrar mensaje con opción de reenviar verificación

---

## Archivos a Crear/Modificar

### Backend - Crear (12 archivos)
1. `domain/exception/InvalidEmailDomainException.java`
2. `domain/exception/EmailNotVerifiedException.java`
3. `shared/infrastructure/config/EmailProperties.java`
4. `shared/application/port/out/EmailSenderPort.java`
5. `shared/infrastructure/adapter/out/email/SmtpEmailAdapter.java`
6. `security/verification/EmailVerificationToken.java`
7. `security/verification/EmailVerificationTokenRepository.java`
8. `security/verification/EmailVerificationService.java`
9. `security/verification/InvalidVerificationTokenException.java`
10. `user/application/port/in/VerifyEmailUseCase.java`
11. `user/application/port/in/ResendVerificationUseCase.java`
12. `user/infrastructure/adapter/in/rest/dto/ResendVerificationRequest.java`

### Backend - Modificar (6 archivos)
1. `pom.xml` - añadir spring-boot-starter-mail
2. `application.properties` - configuración SMTP
3. `application-dev.properties` - mock email
4. `AuthService.java` - flujo registro/login/verificación
5. `AuthController.java` - nuevos endpoints
6. `GlobalExceptionHandler.java` - nuevos handlers

### Frontend - Crear (2 archivos)
1. `pages/VerificationPendingPage.tsx`
2. `pages/VerifyEmailPage.tsx`

### Frontend - Modificar (5 archivos)
1. `components/RegisterForm.tsx`
2. `pages/LoginPage.tsx`
3. `services/authApi.ts`
4. `types/auth.types.ts`
5. `router.tsx`

---

## Configuración Gmail SMTP (Producción)

Para usar Gmail SMTP necesitas:
1. Cuenta Gmail
2. Activar verificación en 2 pasos
3. Generar "App Password" en: Cuenta Google → Seguridad → Contraseñas de aplicaciones
4. Configurar variables de entorno:
   - `GMAIL_USERNAME=tu-email@gmail.com`
   - `GMAIL_APP_PASSWORD=xxxx-xxxx-xxxx-xxxx`
