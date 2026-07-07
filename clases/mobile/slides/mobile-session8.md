# Semana 8: Frameworks Backend para Apps Móviles
**Spring Boot · Ktor · Quarkus · FastAPI · Microservicios · OpenAPI**
Illarek-Lab (S11) — John K. Barrera

---

## Objetivos de la Sesión

Al finalizar la clase, serás capaz de:

- **Comparar frameworks backend 2026:** Spring Boot, Ktor, Quarkus, FastAPI — cuándo usar cada uno.
- **Spring Boot 3.3+:** Virtual Threads (Project Loom), GraalVM native image, Spring Security 6.2.
- **Ktor 2.3+:** DSL de routing con coroutines nativas, configuración sin anotaciones.
- **Microservicios:** diferencias con monolítico, API Gateway, Event-Driven Architecture con Kafka.
- **OpenAPI 3.1:** diseño API-First — el equipo Android genera el cliente Retrofit antes de que el backend esté listo.

---

## ¿Por qué importa el backend en una app móvil?

- La app móvil es tan buena como su backend: latencia, disponibilidad y escalabilidad vienen del servidor.
- El equipo Android/Kotlin necesita saber **qué esperar del backend** para diseñar la capa de red (Retrofit, auth, errores).
- En proyectos reales el mismo equipo toca frontend móvil **y** backend — hoy cubrimos el lado servidor.
- Semana pasada: ORM/DAO (cómo guardar datos). Hoy: **el framework que expone esos datos como API REST.**

---

## Comparativa de Frameworks Backend 2026

| Framework | Lenguaje | Startup | RAM Docker | Cuándo usarlo |
|---|---|---|---|---|
| **Spring Boot 3.3+** | Java/Kotlin | ~2s / 50ms (native) | 200MB / 50MB | Enterprise, ecosistema maduro |
| **Quarkus 3.8+** | Java/Kotlin | ~300ms / 50ms (native) | 120MB / 30MB | Cloud-native, Kubernetes-first |
| **Ktor 2.3+** | Kotlin | ~200ms | 80MB | Equipos Kotlin, APIs ligeras |
| **FastAPI 0.110+** | Python | ~500ms | 100MB | Equipos Python, backends IA/ML |
| **Go Echo 4.12+** | Go | ~10ms | 15MB | Alta concurrencia, microservicios |

Fuente: TechEmpower Framework Benchmarks · Stack Overflow Developer Survey 2024

---

## ¿Cómo elegir el framework correcto?

- **Spring Boot 3.3+** → equipo Java/Kotlin, integración empresarial, ecosistema maduro.
- **Quarkus 3.8+** → proyecto en Kubernetes, startup < 1 segundo requerido.
- **Ktor 2.3+** → equipo 100% Kotlin, control total sobre la configuración, sin "magia".
- **FastAPI** → equipo Python, el backend tiene componentes de IA/ML.
- **Go Echo** → máxima concurrencia con mínima RAM (microservicios de alta carga).
- **Regla:** elegir por el **equipo y contexto**, no solo por benchmarks.

---

## Spring Boot 3.3+ — El Framework Enterprise

- Framework más usado a nivel global (Stack Overflow Survey 2024).
- Basado en convención sobre configuración: starters, auto-configuración, `application.properties`.
- **Java 21 obligatorio** para aprovechar todas las novedades (Virtual Threads, Records, Sealed Classes).
- Tres grandes novedades en 2026:
  - **Virtual Threads** (Project Loom): escalar sin escribir código reactivo.
  - **GraalVM Native Image**: compilar a binario → startup ~50ms.
  - **Spring Security 6.2**: configuración DSL, OAuth2/JWT de primera clase.

---

## Virtual Threads — Project Loom (Java 21)

Problema tradicional: un hilo del pool bloqueado esperando I/O → no puede atender otras peticiones.

```properties
# application.properties — una línea activa virtual threads
spring.threads.virtual.enabled=true
```

```kotlin
@GetMapping("/productos")
fun getProductos(): List<Producto> {
    Thread.sleep(200)            // espera I/O — virtual thread se suspende sin bloquear el pool
    return repository.findAll()  // query a BD — también suspendible
}
```

- **Hilos del pool tradicional:** ~200 simultáneos.
- **Hilos virtuales:** millones simultáneos, gestionados por la JVM.
- Escribe código **síncrono/bloqueante** y obtienes rendimiento **asíncrono** — sin `async/await` ni Flows.

---

## Spring Boot — Estructura de Proyecto Recomendada

```
src/main/kotlin/
├── config/       → SecurityConfig, CorsConfig, SwaggerConfig
├── controller/   → @RestController — solo routing y mapeo HTTP
├── service/      → Lógica de negocio  (≈ UseCase en Android)
├── repository/   → Spring Data JPA interfaces
├── entity/       → @Entity clases (tablas de BD)
├── dto/          → Data Transfer Objects (request / response)
├── mapper/       → Conversiones Entity ↔ DTO
└── exception/    → Excepciones custom + @ControllerAdvice
```

- **Controller** no contiene lógica de negocio — solo llama al **Service**.
- **Service** no sabe nada de HTTP — solo llama al **Repository**.
- Mismo principio de capas que vimos en `platform-api` (api / domain / infra).

---

## Spring Security 6.2 — Autenticación JWT

```kotlin
@Bean
fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
    http
        .csrf { it.disable() }               // APIs REST sin sesión no usan CSRF
        .authorizeHttpRequests {
            it.requestMatchers("/api/public/**").permitAll()
            it.requestMatchers("/api/v1/**").authenticated()
            it.requestMatchers("/admin/**").hasRole("ADMIN")
        }
        .oauth2ResourceServer { it.jwt { } }  // valida JWT de Firebase Auth / Auth0
    return http.build()
}
```

- **`/api/public/**`** → sin autenticación (login, registro, health check).
- **`/api/v1/**`** → requiere JWT válido en header `Authorization: Bearer <token>`.
- **`/admin/**`** → requiere JWT + rol `ADMIN`.
- El JWT lo emite Firebase Auth o Auth0 — Spring solo lo **valida**.

---

## Ktor 2.3+ — El Framework Kotlin-Native

- Desarrollado por JetBrains, usa **coroutines nativas** de Kotlin.
- Configuración por **DSL**, sin anotaciones (`@RestController`, `@Autowired` no existen).
- Menor "magia" que Spring: lo que no configuras explícitamente, no existe.
- Se instalan **plugins** (equivalente a starters): `ContentNegotiation`, `Authentication`, `CORS`, `CallLogging`.

```kotlin
embeddedServer(Netty, port = 8080) {
    install(ContentNegotiation) { json() }
    install(Authentication) { jwt("auth-jwt") { /* ... */ } }
    routing {
        get("/api/v1/productos") { call.respond(service.getAll()) }
        post("/api/v1/productos") {
            val req = call.receive<CreateProductoRequest>()
            call.respond(HttpStatusCode.Created, service.create(req))
        }
    }
}.start(wait = true)
```

---

## Monolítico vs Microservicios

| | Monolítico | Microservicios |
|---|---|---|
| **Estructura** | 1 backend, 1 base de datos | N servicios independientes, cada uno con su BD |
| **Desarrollo inicial** | Simple | Complejo (red, contratos, orquestación) |
| **Escalado** | Todo o nada | Por servicio (solo el de órdenes si hay pico) |
| **Fallos** | Un bug puede tumbar todo | Fallos aislados por servicio |
| **Testing local** | Fácil | Difícil (múltiples contenedores) |
| **Cuándo usarlo** | MVP, equipos pequeños, primer año del proyecto | Alta escala, equipos grandes, dominios bien definidos |

> **Regla de oro:** empezar monolítico, migrar a microservicios cuando el monolito duele.

---

## API Gateway Pattern

La app móvil siempre habla con **un solo punto de entrada**, no con cada microservicio directamente.

```
App Android / iOS
       ↓ HTTPS (un solo dominio: api.miapp.com)
   API Gateway
   (auth · rate limiting · routing · logging)
   ┌──────┬──────────┬──────────┬──────────┐
   ↓      ↓          ↓          ↓          ↓
 Auth  Productos  Órdenes  Usuarios   Notif.
```

- **Autenticación centralizada:** el gateway valida el JWT, los servicios no necesitan saber de auth.
- **Rate limiting:** protección contra abuso de la API desde la app.
- **Opciones:** Kong, AWS API Gateway, Nginx, Spring Cloud Gateway.

---

## Event-Driven Architecture con Kafka

En microservicios, en lugar de llamadas síncronas entre servicios, se publican **eventos**:

```kotlin
// Servicio Órdenes → publica evento al crear una orden
kafkaTemplate.send("orden-creada", OrdenCreadaEvent(
    ordenId = orden.id, usuarioId = orden.usuarioId, productos = orden.productos
))
```

```kotlin
// Servicio Inventario → escucha el evento y descuenta stock automáticamente
@KafkaListener(topics = ["orden-creada"])
fun handleOrdenCreada(event: OrdenCreadaEvent) {
    event.productos.forEach { inventarioService.descontarStock(it.productoId, it.cantidad) }
}
```

- **Servicios desacoplados:** Órdenes no sabe que Inventario existe — solo publica.
- **Resiliencia:** si Inventario cae, el evento queda en Kafka y se procesa al volver.
- **Casos de uso:** notificaciones push, actualización de stock, auditoría, analytics.

---

## OpenAPI 3.1 — API-First Design

**API-First:** definir el contrato (YAML) **antes** de escribir código.

```yaml
openapi: 3.1.0
paths:
  /api/v1/productos:
    get:
      summary: Obtener lista de productos
      responses:
        '200':
          content:
            application/json:
              schema:
                type: array
                items: { $ref: '#/components/schemas/Producto' }
components:
  schemas:
    Producto:
      type: object
      required: [id, nombre, precio]
      properties:
        id:     { type: integer }
        nombre: { type: string, maxLength: 200 }
        precio: { type: number, minimum: 0 }
```

---

## OpenAPI — Beneficio para Apps Móviles

El equipo Android **genera el cliente Retrofit automáticamente** desde el YAML:

```bash
# OpenAPI Generator — genera cliente Kotlin/Retrofit desde el spec
openapi-generator-cli generate \
  -i openapi.yaml \
  -g kotlin \
  -o ./app/src/main/kotlin/api
```

```kotlin
// Código generado automáticamente — el dev Android no escribe esto a mano
interface ProductosApi {
    @GET("api/v1/productos")
    suspend fun getProductos(): List<ProductoDto>

    @POST("api/v1/productos")
    suspend fun createProducto(@Body body: CreateProductoRequest): ProductoDto
}
```

- **Frontend y backend trabajan en paralelo** desde el día 1.
- Cambios en el YAML → regenerar cliente → errores de compilación si algo rompe el contrato.
- Spring Boot genera `/docs` (Swagger UI) automáticamente desde las anotaciones del controller.

---

## Ejemplo Real — `platform-api` (FastAPI + SQLAlchemy)

El proyecto del lab usa **FastAPI** — el equivalente Python de Spring Boot:

| Concepto Spring Boot | Equivalente `platform-api` (FastAPI) |
|---|---|
| `@RestController` | `APIRouter` + `@router.get/post` |
| `@Service` | Clase `Service` inyectada vía `Depends()` |
| `@Repository` + JPA | `GeoEventORMRepository` + SQLAlchemy async |
| `application.properties` | `credentials/layout_example.env` |
| `@Autowired` | `Depends(get_geo_event_orm_repo)` |
| Swagger UI en `/swagger-ui` | Swagger UI en `/docs` (automático) |

- Misma arquitectura en capas: `api/ → domain/ ← infra/`
- Mismos patrones: Repository, Ports & Adapters, DTO, inyección de dependencias.
- **Lenguaje diferente, principios idénticos.**

---

## Resumen de la Sesión

| Concepto | Clave a recordar |
|---|---|
| **Spring Boot 3.3+** | Virtual Threads → escala bloqueante como reactivo. GraalVM → 50ms startup. |
| **Ktor 2.3+** | DSL Kotlin, coroutines nativas, sin anotaciones, máximo control. |
| **Quarkus 3.8+** | Cloud-native, Kubernetes-first, Dev Services automático. |
| **FastAPI** | Python async, `/docs` automático, mismo patrón Data Mapper que JPA. |
| **Microservicios** | Escala independiente + fallos aislados. Empezar monolítico, migrar cuando duele. |
| **API Gateway** | Un solo punto de entrada para la app móvil. Auth, rate limiting, routing centralizados. |
| **Event-Driven / Kafka** | Servicios desacoplados vía eventos. Resiliencia ante caídas. |
| **OpenAPI 3.1** | Contrato primero → cliente Retrofit generado automáticamente. Frontend y backend en paralelo. |
