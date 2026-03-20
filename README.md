# Comunicación entre Microservicios con Spring Cloud OpenFeign

## Descripción del proyecto

Este proyecto implementa comunicación entre dos microservicios de un club deportivo usando **Spring Cloud OpenFeign**:

- **serviciocanchas** — gestiona las canchas disponibles (puerto `8081`)
- **servicioreservas** — gestiona las reservas de clientes (puerto `8082`)

Cuando un cliente crea una reserva, `servicioreservas` consulta a `serviciocanchas` para verificar que la cancha existe antes de guardarla en la base de datos. Esta comunicación se realiza de forma transparente mediante OpenFeign.

---

## ¿Qué es Spring Cloud OpenFeign?

OpenFeign es un cliente HTTP declarativo que permite que un microservicio llame a otro como si invocara un método Java local. En lugar de escribir manualmente la lógica HTTP (construir la URL, ejecutar la petición, parsear el JSON de respuesta), el desarrollador solo define una interfaz con anotaciones y Spring genera automáticamente el código de red en tiempo de ejecución.

### Comparación con RestTemplate

| | RestTemplate | WebClient | OpenFeign |
|---|---|---|---|
| Código HTTP | Manual (URL, método, parseo) | Manual con API fluida | Automático via anotaciones |
| Legibilidad | Verboso | Moderado | Limpio y declarativo |
| Modelo de ejecución | Síncrono (bloqueante) | Síncrono y asíncrono (reactivo) | Síncrono (bloqueante) |
| Integración Spring | Básica | Spring WebFlux | Nativa con Spring MVC |
| Manejo de errores | Manual | Via `onStatus()` / `onErrorMap()` | Via `FeignException` |
| Dependencia requerida | `spring-boot-starter-web` | `spring-boot-starter-webflux` | `spring-cloud-starter-openfeign` |
| Cuándo usarlo | Proyectos legacy o simples | Alta concurrencia o llamadas no bloqueantes | Microservicios con múltiples endpoints a consumir |

**Ejemplo con RestTemplate (código que se reemplaza):**
```java
String url = "http://localhost:8081/api/canchas/" + reserva.getCanchaId();
Cancha cancha = restTemplate.getForObject(url, Cancha.class);
```

**Equivalente con OpenFeign:**
```java
Cancha cancha = canchaClient.obtenerCanchaPorId(reserva.getCanchaId());
```

---

## Flujo de comunicación

```
Postman
  │
  │  POST /api/reservas
  ▼
ReservaController          (servicioreservas - 8082)
  │
  ▼
ReservaServicesImpl
  │  canchaClient.obtenerCanchaPorId(id)
  ▼
CanchaClient [Feign Proxy]
  │  GET http://localhost:8081/api/canchas/{id}
  ▼
CanchaController           (serviciocanchas - 8081)
  │  responde con JSON
  ▼
ReservaServicesImpl
  │  verifica que cancha != null
  │  si existe → reservaRepository.save(reserva)
  │  si no existe → lanza RuntimeException
  ▼
Postman recibe respuesta
```

---

## Cambios implementados

### 1. `pom.xml` de `servicioreservas`

#### Versiones compatibles

Ajustar el parent de Spring Boot a una versión compatible con Spring Cloud `2023.0.x`:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.6</version>
    <relativePath/>
</parent>
```

#### Propiedad de versión Spring Cloud

Dentro de `<properties>`:

```xml
<spring-cloud.version>2023.0.5</spring-cloud.version>
```

#### Dependencia OpenFeign

Dentro de `<dependencies>`:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

#### BOM de Spring Cloud

Después del bloque `<dependencies>`, antes de `<build>`:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

> **Nota de compatibilidad:** Spring Cloud y Spring Boot tienen versiones estrictamente compatibles. La combinación validada es Spring Boot `3.3.x` + Spring Cloud `2023.0.5`. Usar Spring Boot `4.x` o Spring Cloud `2024.x` produce errores de arranque.

---

### 2. `ServicioreservasApplication.java`

Agregar `@EnableFeignClients` para que Spring escanee y registre las interfaces Feign al arrancar:

```java
package com.clubdeportivo2.servicioreservas;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients(basePackages = "com.clubdeportivo2.servicioreservas.client")
public class ServicioreservasApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServicioreservasApplication.class, args);
    }
}
```

> Sin esta anotación, Spring ignora completamente la interfaz `CanchaClient` y no puede inyectarla, produciendo un error `NoSuchBeanDefinitionException` al arrancar.

---

### 3. Nuevo archivo: `client/CanchaClient.java`

Crear la carpeta `client` dentro del paquete principal y agregar la interfaz:

```
src/main/java/com/clubdeportivo2/servicioreservas/client/CanchaClient.java
```

```java
package com.clubdeportivo2.servicioreservas.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import com.clubdeportivo2.servicioreservas.model.dto.Cancha;

@FeignClient(name = "serviciocanchas", url = "http://localhost:8081")
public interface CanchaClient {

    @GetMapping("/api/canchas/{id}")
    Cancha obtenerCanchaPorId(@PathVariable("id") Long id);
}
```

> `@FeignClient` define el servicio externo a consumir. `url` apunta al host y puerto de `serviciocanchas`. El método mapea exactamente al endpoint `GET /api/canchas/{id}` del otro microservicio.

---

### 4. `ReservaServicesImpl.java`

Inyectar `CanchaClient` y usarlo en `crearReserva` para validar que la cancha existe antes de guardar la reserva:

```java
package com.clubdeportivo2.servicioreservas.services;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.clubdeportivo2.servicioreservas.client.CanchaClient;
import com.clubdeportivo2.servicioreservas.model.Reserva;
import com.clubdeportivo2.servicioreservas.model.dto.Cancha;
import com.clubdeportivo2.servicioreservas.repository.ReservaRepository;

import feign.FeignException;
import java.util.List;

@Service
public class ReservaServicesImpl implements ReservaServices {

    @Autowired
    private ReservaRepository reservaRepository;

    @Autowired
    private CanchaClient canchaClient;

    @Override
    public List<Reserva> listarReservas() {
        return reservaRepository.findAll();
    }

    @Override
    public Reserva crearReserva(Reserva reserva) {
        try {
            Cancha cancha = canchaClient.obtenerCanchaPorId(reserva.getCanchaId());
            if (cancha == null) {
                throw new RuntimeException("La cancha con id " + reserva.getCanchaId() + " no existe");
            }
        } catch (FeignException.NotFound e) {
            throw new RuntimeException("La cancha con id " + reserva.getCanchaId() + " no existe");
        }

        return reservaRepository.save(reserva);
    }

    @Override
    public List<Reserva> buscarReservasPorCancha(Long canchaId) {
        return reservaRepository.findByCanchaId(canchaId);
    }

    @Override
    public Cancha obtenerCanchaPorReserva(Long canchaId) {
        return canchaClient.obtenerCanchaPorId(canchaId);
    }
}
```

---

### 5. Nuevo archivo: `exception/CanchaNotFoundException.java` (en `serviciocanchas`)

```
src/main/java/com/clubdeportivo/serviciocanchas/exception/CanchaNotFoundException.java
```

```java
package com.clubdeportivo.serviciocanchas.exception;

public class CanchaNotFoundException extends RuntimeException {

    public CanchaNotFoundException(Long id) {
        super("No se encontró una cancha con el id: " + id);
    }
}
```

---

### 6. Nuevo archivo: `exception/GlobalExceptionHandler.java` (en `serviciocanchas`)

```
src/main/java/com/clubdeportivo/serviciocanchas/exception/GlobalExceptionHandler.java
```

```java
package com.clubdeportivo.serviciocanchas.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(CanchaNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleCanchaNotFound(CanchaNotFoundException ex) {
        Map<String, Object> error = new HashMap<>();
        error.put("timestamp", LocalDateTime.now().toString());
        error.put("status", 404);
        error.put("error", "Not Found");
        error.put("message", ex.getMessage());

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

---

### 7. `ServicesImpl.java` (en `serviciocanchas`)

Actualizar `buscarCancha` para lanzar la excepción en lugar de retornar `null`:

```java
@Override
public Cancha buscarCancha(Long id) {
    return canchaRepository.findById(id)
        .orElseThrow(() -> new CanchaNotFoundException(id));
}
```

---

## Estructura de archivos modificados / creados

```
servicioreservas/
├── pom.xml                                         ← versiones + dependencia Feign
├── src/main/java/com/clubdeportivo2/servicioreservas/
│   ├── ServicioreservasApplication.java            ← @EnableFeignClients
│   ├── client/
│   │   └── CanchaClient.java                       ← NUEVO - interfaz Feign
│   └── services/
│       └── ReservaServicesImpl.java                ← inyección + validación

serviciocanchas/
└── src/main/java/com/clubdeportivo/serviciocanchas/
    ├── exception/
    │   ├── CanchaNotFoundException.java             ← NUEVO
    │   └── GlobalExceptionHandler.java             ← NUEVO
    └── services/
        └── ServicesImpl.java                       ← lanza excepción en buscarCancha
```

---

## Cómo ejecutar

```bash
# Terminal 1 — levantar serviciocanchas primero
cd serviciocanchas
./mvnw spring-boot:run

# Terminal 2 — levantar servicioreservas
cd servicioreservas
./mvnw spring-boot:run
```

---

## Pruebas con Postman

### Crear reserva con cancha válida
```
POST http://localhost:8082/api/reservas
Content-Type: application/json

{
  "canchaId": 1,
  "nombreCliente": "Juan Pérez",
  "fecha": "2026-04-01",
  "horaInicio": "10:00:00",
  "horaFin": "11:00:00"
}
```
**Respuesta esperada:** `200 OK` con la reserva creada.

---

### Crear reserva con cancha inexistente
```
POST http://localhost:8082/api/reservas
Content-Type: application/json

{
  "canchaId": 999,
  "nombreCliente": "Juan Pérez",
  "fecha": "2026-04-01",
  "horaInicio": "10:00:00",
  "horaFin": "11:00:00"
}
```
**Respuesta esperada:** `500` con mensaje `"La cancha con id 999 no existe"`.

---

### Consultar detalle de cancha desde servicioreservas (via Feign)
```
GET http://localhost:8082/api/reservas/detalle-cancha/1
```
**Respuesta esperada:** `200 OK` con el objeto Cancha.

---

## Tecnologías utilizadas

- Java 21
- Spring Boot 3.3.6
- Spring Cloud 2023.0.5
- Spring Cloud OpenFeign
- Spring Data JPA
- MySQL
- Lombok
- Maven
