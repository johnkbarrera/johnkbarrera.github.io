# Lab 6 — Sync Center: subida de datos a servicio web (en construcción)

**Autor:** [illarek-lab](https://github.com/illarek-lab)
**Proyecto:** DemoData · `com.illareklab.demodata`
**Temas:** L3 completo (HTTP/REST, Corrutinas, Repositorio, Retrofit, OkHttp, DI manual)

> **Estado:** La pantalla `SyncScreen` y el `SyncViewModel` ya están implementados y muestran los conteos reactivos de la BD. El botón "Sincronizar ahora" muestra un `Toast("Por implementar")`. **Este laboratorio completa esa implementación**, agregando la capa de red con Retrofit + OkHttp y conectando el botón al proceso real de subida de datos.

---

## 1. Qué existe hoy vs. qué se construye en este lab

```
Estado actual                         Estado final (este lab)
─────────────────────────────────     ──────────────────────────────────────
SyncScreen                            SyncScreen
└── SyncViewModel                     └── SyncViewModel  (extendido)
    └── counts: StateFlow<SyncCounts>     ├── counts: StateFlow<SyncCounts>
        (5 conteos en tiempo real)         ├── isSyncing: StateFlow<Boolean>
                                           └── lastResult: StateFlow<SyncResult?>
[Botón] Toast "Por implementar"
                                      [Botón] → SyncViewModel.sync()
                                                 → SyncRepository.syncAll()
                                                     ├── ApiService.postGpsGoogle(list)
                                                     ├── ApiService.postGpsSensors(list)
                                                     ├── ApiService.postMedia(list)
                                                     └── ApiService.postAudio(list)
                                                 → marca is_synced = true en Room
                                                 → SyncScreen muestra resultado
```

---

## 2. Arquitectura objetivo del lab

```
SyncScreen
└── SyncViewModel
    ├── isSyncing: StateFlow<Boolean>
    ├── lastResult: StateFlow<SyncResult?>
    └── sync() → SyncRepository.syncAll()
                    │
                    ├── GpsRepository.getUnsynced()    → Room SELECT WHERE is_synced = 0
                    ├── MediaRepository.getUnsynced()
                    └── AudioRepository.getUnsynced()
                    │
                    ├── ApiService.postGpsGoogle(list)  ─┐
                    ├── ApiService.postGpsSensors(list)  ├─ Retrofit + OkHttp
                    ├── ApiService.postMedia(list)       │
                    └── ApiService.postAudio(list)      ─┘
                    │
                    └── marcar is_synced = true en Room
```

---

## 3. Dependencias (`app/build.gradle.kts`)

Este lab agrega **Retrofit**, **OkHttp** y **Gson** al proyecto. Todo lo demás viene de Labs 4 y 5.

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.ksp)
}

ksp {
    arg("room.generateKotlin", "true")
    arg("useK2", "true")
}

dependencies {

    // ════════════════════════════════════════════════
    // ── Lab 4: GNSS + Perfil ──
    // ════════════════════════════════════════════════

    // Compose BOM: gestiona versiones de todos los artefactos Compose
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.material3)
    implementation("androidx.compose.material:material-icons-extended:1.7.8")
    implementation("androidx.navigation:navigation-compose:2.8.0")

    // Room: base de datos SQLite reactiva con Flow
    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)
    ksp(libs.androidx.room.compiler)          // genera el código en compilación (no implementation)

    // DataStore: almacenamiento clave-valor asíncrono (reemplaza SharedPreferences)
    implementation("androidx.datastore:datastore-preferences:1.1.1")

    // ViewModel + lifecycle: sobrevive rotaciones; collectAsStateWithLifecycle
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.6")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.6")

    // Google FLP + bridge de coroutines: .await() sobre Task<Location>
    implementation("com.google.android.gms:play-services-location:21.3.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.11.0")

    // Coil: thumbnails de archivos en ProfileScreen → MyActivity
    implementation("io.coil-kt:coil-compose:2.7.0")

    // Accompanist: rememberMultiplePermissionsState para permisos de ubicación
    implementation("com.google.accompanist:accompanist-permissions:0.34.0")

    // ════════════════════════════════════════════════
    // ── Lab 5: Multimedia, Audio y Notificaciones ──
    // ════════════════════════════════════════════════

    // WorkManager: notificaciones diferidas que sobreviven al cierre de la app
    implementation("androidx.work:work-runtime-ktx:2.9.0")

    // CameraX: captura de fotos y videos (5 artefactos necesarios)
    val cameraxVersion = "1.6.0"
    implementation("androidx.camera:camera-core:$cameraxVersion")      // API base
    implementation("androidx.camera:camera-camera2:$cameraxVersion")   // backend Camera2
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion") // vínculo con ciclo de vida
    implementation("androidx.camera:camera-view:$cameraxVersion")      // PreviewView
    implementation("androidx.camera:camera-video:$cameraxVersion")     // VideoCapture use case

    // ════════════════════════════════════════════════
    // ── Lab 6: Sync + HTTP ──
    // ════════════════════════════════════════════════

    // Retrofit: define ApiService con @POST; genera la implementación en runtime
    implementation("com.squareup.retrofit2:retrofit:2.11.0")

    // Converter Gson: serializa DTOs → JSON (request) y JSON → SyncBatchResponse (response)
    implementation("com.squareup.retrofit2:converter-gson:2.11.0")

    // OkHttp: cliente HTTP de bajo nivel que Retrofit usa internamente
    implementation("com.squareup.okhttp3:okhttp:4.12.0")

    // Logging Interceptor: imprime en Logcat el JSON enviado y la respuesta — solo para debug
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

    // Gson: motor de serialización JSON usado por el converter
    implementation("com.google.code.gson:gson:2.10.1")
}
```

### `gradle/libs.versions.toml` — entradas de Room (ya existentes, para referencia)

```toml
[versions]
kotlin       = "2.2.10"
ksp          = "2.2.10-2.0.2"
androidxRoom = "2.7.0-alpha11"

[libraries]
androidx-room-runtime  = { group = "androidx.room", name = "room-runtime",  version.ref = "androidxRoom" }
androidx-room-ktx      = { group = "androidx.room", name = "room-ktx",      version.ref = "androidxRoom" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler",  version.ref = "androidxRoom" }

[plugins]
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

> Retrofit, OkHttp y Gson se declaran con la versión inline (`"com.squareup.retrofit2:retrofit:2.11.0"`) en lugar de en el TOML porque son dependencias puntuales de un solo lab. En un proyecto de producción lo correcto es moverlas al TOML.

### ¿Para qué sirve cada nueva dependencia?

| Dependencia | Rol en el lab |
|---|---|
| `retrofit` | Define `ApiService` con anotaciones `@POST`; genera la implementación en tiempo de ejecución |
| `converter-gson` | Convierte los DTOs (`SyncBatchRequest`) a JSON en el POST y el JSON de respuesta a `SyncBatchResponse` |
| `okhttp` | Ejecuta la conexión TCP/TLS real; gestiona el pool de conexiones y los timeouts |
| `logging-interceptor` | Intercepta cada request/response y lo imprime en Logcat con nivel `BODY` — esencial para depurar |
| `gson` | Motor de serialización JSON usado por el converter; necesario aunque no se use directamente |

### Relación entre las librerías

```
Tu código (SyncRepository)
    ↓ llama
ApiService (interfaz Retrofit)
    ↓ implementada por
Retrofit (genera proxy dinámico)
    ↓ serializa DTOs con
Gson + GsonConverterFactory
    ↓ envía el request con
OkHttpClient
    ↓ interceptado por
HttpLoggingInterceptor → Logcat
    ↓ viaja por la red
Servidor REST
```

### Permisos en `AndroidManifest.xml` (ya presentes)

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

`INTERNET` es un permiso normal (no requiere aprobación del usuario en runtime). `ACCESS_NETWORK_STATE` permite verificar si hay conexión antes de intentar el POST, mejorando el mensaje de error cuando no hay red.

---

## 4. Migración de la BD — `is_synced`

Cada entidad necesita un campo `is_synced` para saber qué registros ya se subieron.

### 4.1 Actualizar entidades

```kotlin
// GpsGoogleEntity.kt
@Entity(tableName = "gps_google")
data class GpsGoogleEntity(
    // ... campos existentes ...
    val isSynced: Boolean = false    // ← nuevo campo
)
```

Repetir para `GpsSensorsEntity`, `MediaEntity` y `AudioEntity`.

### 4.2 Incrementar versión de la BD

```kotlin
@Database(
    entities = [...],
    version = 3,                    // ← era 2
    exportSchema = false
)
abstract class DemoDataDatabase : RoomDatabase() {

    companion object {
        private val MIGRATION_2_3 = object : Migration(2, 3) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("ALTER TABLE gps_google   ADD COLUMN isSynced INTEGER NOT NULL DEFAULT 0")
                db.execSQL("ALTER TABLE gps_sensors  ADD COLUMN isSynced INTEGER NOT NULL DEFAULT 0")
                db.execSQL("ALTER TABLE media        ADD COLUMN isSynced INTEGER NOT NULL DEFAULT 0")
                db.execSQL("ALTER TABLE audio        ADD COLUMN isSynced INTEGER NOT NULL DEFAULT 0")
            }
        }

        fun getInstance(context: Context): DemoDataDatabase =
            INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(...)
                    .addMigrations(MIGRATION_2_3)          // ← reemplaza fallbackToDestructive
                    .build().also { INSTANCE = it }
            }
    }
}
```

**¿Por qué `Migration` y no `fallbackToDestructiveMigration`?**

En el Lab 4 y 5 usamos `fallbackToDestructive` porque los datos de prueba son desechables. En este lab **los datos ya capturados son el objetivo de la sync** — borrarlos antes de subirlos anula el propósito del laboratorio. La migración explícita con `ALTER TABLE` preserva todos los registros.

---

## 5. Nuevas queries en los DAOs

```kotlin
// GpsGoogleDao.kt — agregar:
@Query("SELECT * FROM gps_google WHERE isSynced = 0 ORDER BY timestamp ASC")
suspend fun getUnsynced(): List<GpsGoogleEntity>

@Query("UPDATE gps_google SET isSynced = 1 WHERE id IN (:ids)")
suspend fun markSynced(ids: List<Long>)
```

Repetir para los otros 3 DAOs.

---

## 6. DTOs — Modelos para la red

Los DTOs (Data Transfer Objects) son los modelos que Gson serializa a JSON para el POST. Son distintos de las entidades de Room — no llevan `isSynced`, `id` auto-generado ni rutas de `filesDir`.

```kotlin
// data/remote/dto/GpsGoogleDto.kt
data class GpsGoogleDto(
    val latitude:  Double?,
    val longitude: Double?,
    val accuracy:  Float?,
    val speed:     Float?,
    val bearing:   Float?,
    val timestamp: Long
)

// data/remote/dto/GpsSensorsDto.kt
data class GpsSensorsDto(
    val latitude:  Double?,
    val longitude: Double?,
    val provider:  String,
    val altitude:  Double?,
    val timestamp: Long
)

// data/remote/dto/MediaDto.kt
data class MediaDto(
    val type:       String,   // "PHOTO" o "VIDEO"
    val sizeBytes:  Long,
    val durationMs: Long?,
    val timestamp:  Long
    // filePath NO se incluye: la ruta local no tiene sentido en el servidor
)

// data/remote/dto/AudioDto.kt
data class AudioDto(
    val durationMs: Long,
    val sizeBytes:  Long,
    val format:     String,
    val timestamp:  Long
)

// Wrappers para el batch POST
data class SyncBatchRequest(
    val gpsGoogle:  List<GpsGoogleDto>,
    val gpsSensors: List<GpsSensorsDto>,
    val media:      List<MediaDto>,
    val audio:      List<AudioDto>
)

data class SyncBatchResponse(
    val success:       Boolean,
    val processedCount: Int,
    val message:       String
)
```

---

## 7. `ApiService` — Interfaz Retrofit

```kotlin
// data/remote/ApiService.kt
interface ApiService {

    // Opción A: endpoint único (batch)
    @POST("api/v1/sync")
    suspend fun syncBatch(@Body request: SyncBatchRequest): Response<SyncBatchResponse>

    // Opción B: endpoints separados por tipo
    @POST("api/v1/gps/google")
    suspend fun postGpsGoogle(@Body list: List<GpsGoogleDto>): Response<SyncBatchResponse>

    @POST("api/v1/gps/sensors")
    suspend fun postGpsSensors(@Body list: List<GpsSensorsDto>): Response<SyncBatchResponse>

    @POST("api/v1/media")
    suspend fun postMedia(@Body list: List<MediaDto>): Response<SyncBatchResponse>

    @POST("api/v1/audio")
    suspend fun postAudio(@Body list: List<AudioDto>): Response<SyncBatchResponse>
}
```

> **Nota:** el instructor decide qué endpoints usar según el backend disponible. El resto de la arquitectura funciona igual en ambas opciones.

---

## 8. `NetworkManager` — OkHttp + Retrofit

```kotlin
// data/remote/NetworkManager.kt
object NetworkManager {

    private const val BASE_URL = "https://tu-servidor.ejemplo.com/"
    private const val TIMEOUT_SECONDS = 30L

    private val okHttpClient = OkHttpClient.Builder()
        .connectTimeout(TIMEOUT_SECONDS, TimeUnit.SECONDS)
        .readTimeout(TIMEOUT_SECONDS, TimeUnit.SECONDS)
        .writeTimeout(TIMEOUT_SECONDS, TimeUnit.SECONDS)
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = HttpLoggingInterceptor.Level.BODY   // ver JSON en Logcat
        })
        .build()

    val apiService: ApiService = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(okHttpClient)
        .addConverterFactory(GsonConverterFactory.create())
        .build()
        .create(ApiService::class.java)
}
```

El `HttpLoggingInterceptor` imprime en Logcat el JSON enviado y la respuesta recibida. Es imprescindible para depurar durante el lab. **Debe desactivarse en producción** (cambiando el `level` a `NONE`).

---

## 9. `SyncRepository` — Coordinador de la sincronización

```kotlin
// data/repository/SyncRepository.kt
class SyncRepository(
    private val gpsRepository:   GpsRepository,
    private val mediaRepository: MediaRepository,
    private val audioRepository: AudioRepository,
    private val apiService:      ApiService
) {
    sealed class SyncResult {
        data class Success(val count: Int) : SyncResult()
        data class Error(val message: String) : SyncResult()
    }

    suspend fun syncAll(): SyncResult {
        return try {
            // 1. Leer no sincronizados de Room
            val gpsGoogle  = gpsRepository.googleDao.getUnsynced()
            val gpsSensors = gpsRepository.sensorsDao.getUnsynced()
            val media      = mediaRepository.mediaDao.getUnsynced()
            val audio      = audioRepository.audioDao.getUnsynced()

            val totalPending = gpsGoogle.size + gpsSensors.size + media.size + audio.size
            if (totalPending == 0) return SyncResult.Success(0)

            // 2. Mapear entidades → DTOs
            val request = SyncBatchRequest(
                gpsGoogle  = gpsGoogle.map { GpsGoogleDto(it.latitude, it.longitude,
                                             it.accuracy, it.speed, it.bearing, it.timestamp) },
                gpsSensors = gpsSensors.map { GpsSensorsDto(it.latitude, it.longitude,
                                              it.provider, it.altitude, it.timestamp) },
                media      = media.map { MediaDto(it.type, it.sizeBytes, it.durationMs, it.timestamp) },
                audio      = audio.map { AudioDto(it.durationMs, it.sizeBytes, it.format, it.timestamp) }
            )

            // 3. POST al servidor
            val response = apiService.syncBatch(request)

            if (response.isSuccessful) {
                // 4. Marcar como sincronizados en Room
                gpsRepository.googleDao.markSynced(gpsGoogle.map { it.id })
                gpsRepository.sensorsDao.markSynced(gpsSensors.map { it.id })
                mediaRepository.mediaDao.markSynced(media.map { it.id })
                audioRepository.audioDao.markSynced(audio.map { it.id })

                SyncResult.Success(response.body()?.processedCount ?: totalPending)
            } else {
                SyncResult.Error("Error del servidor: ${response.code()} ${response.message()}")
            }
        } catch (e: IOException) {
            SyncResult.Error("Sin conexión a internet: ${e.message}")
        } catch (e: Exception) {
            SyncResult.Error("Error inesperado: ${e.message}")
        }
    }
}
```

**Decisión de diseño:** `markSynced` se llama **después** de confirmar que el servidor respondió `isSuccessful`. Si se marcara antes del POST y fallara la red, los registros quedarían marcados como sincronizados sin haberlo estado realmente.

---

## 10. `SyncViewModel` — Extendido con estado de red

```kotlin
// Agregar a SyncViewModel.kt:

sealed class SyncResult {
    data class Success(val count: Int) : SyncResult()
    data class Error(val message: String) : SyncResult()
    object Idle : SyncResult()
}

class SyncViewModel(
    gpsRepository:   GpsRepository,
    mediaRepository: MediaRepository,
    audioRepository: AudioRepository,
    private val syncRepository: SyncRepository    // ← nuevo
) : ViewModel() {

    // Conteos reactivos (ya existían)
    val counts = combine(...).stateIn(...)

    // Estado del proceso de sync
    private val _isSyncing = MutableStateFlow(false)
    val isSyncing = _isSyncing.asStateFlow()

    private val _lastResult = MutableStateFlow<SyncResult>(SyncResult.Idle)
    val lastResult = _lastResult.asStateFlow()

    fun sync() {
        if (_isSyncing.value) return    // previene doble tap
        viewModelScope.launch {
            _isSyncing.value = true
            _lastResult.value = syncRepository.syncAll()
            _isSyncing.value = false
        }
    }
}
```

---

## 11. `SyncScreen` — Botón funcional

```kotlin
@Composable
fun SyncScreen() {
    val counts     by vm.counts.collectAsStateWithLifecycle()
    val isSyncing  by vm.isSyncing.collectAsStateWithLifecycle()
    val lastResult by vm.lastResult.collectAsStateWithLifecycle()

    // Botón principal
    Button(
        onClick = { vm.sync() },
        enabled = !isSyncing && counts.total > 0,
        modifier = Modifier.fillMaxWidth().height(56.dp)
    ) {
        if (isSyncing) {
            CircularProgressIndicator(modifier = Modifier.size(24.dp), strokeWidth = 2.dp,
                color = MaterialTheme.colorScheme.onPrimary)
            Spacer(Modifier.width(8.dp))
            Text("Sincronizando…")
        } else {
            Icon(CloudUpload, null)
            Spacer(Modifier.width(8.dp))
            Text("Sincronizar ahora")
        }
    }

    // Resultado del último sync
    when (val result = lastResult) {
        is SyncResult.Success -> Card(
            colors = CardDefaults.cardColors(containerColor = colorScheme.primaryContainer)
        ) {
            Text("✓ ${result.count} registros sincronizados",
                color = colorScheme.onPrimaryContainer)
        }
        is SyncResult.Error -> Card(
            colors = CardDefaults.cardColors(containerColor = colorScheme.errorContainer)
        ) {
            Text("✗ ${result.message}", color = colorScheme.onErrorContainer)
        }
        SyncResult.Idle -> { /* primera vez, no mostrar nada */ }
    }

    // Conteos existentes (ya funcionaban)
    // ...
}
```

---

## 12. Actualizar `DemoDataApp` — Agregar `SyncRepository`

```kotlin
class DemoDataApp : Application() {

    // ... dependencias existentes ...

    val networkManager by lazy { NetworkManager }         // singleton object
    val syncRepository by lazy {
        SyncRepository(gpsRepository, mediaRepository, audioRepository, networkManager.apiService)
    }
}
```

Y la `Factory` del `SyncViewModel`:

```kotlin
class Factory(
    private val gps:    GpsRepository,
    private val media:  MediaRepository,
    private val audio:  AudioRepository,
    private val sync:   SyncRepository        // ← nuevo
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T =
        SyncViewModel(gps, media, audio, sync) as T
}
```

En `SyncScreen`:

```kotlin
val vm: SyncViewModel = viewModel(
    factory = SyncViewModel.Factory(
        app.gpsRepository, app.mediaRepository,
        app.audioRepository, app.syncRepository    // ← nuevo
    )
)
```

---

## 13. `AndroidManifest.xml` — Permisos de red

```xml
<!-- Ya presentes en el manifest del proyecto -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

Ambos permisos ya están declarados en el proyecto. `ACCESS_NETWORK_STATE` permite verificar si hay conexión antes de intentar el POST.

---

## 14. Verificación de conectividad (opcional)

```kotlin
// Agregar en SyncRepository.syncAll() antes del POST:
fun isConnected(context: Context): Boolean {
    val cm = context.getSystemService(CONNECTIVITY_SERVICE) as ConnectivityManager
    return cm.activeNetwork?.let { network ->
        cm.getNetworkCapabilities(network)
            ?.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) == true
    } ?: false
}
```

Sin esta verificación, el `IOException` del `try-catch` en `syncAll()` igual maneja la falta de red, pero con un mensaje menos descriptivo.

---

## 15. Flujo de datos completo del Lab 6

```
[SyncScreen] tap "Sincronizar ahora"
  → SyncViewModel.sync()
  → _isSyncing = true → botón muestra CircularProgressIndicator

  → SyncRepository.syncAll()
      │
      ├── Room: getUnsynced() × 4 tablas
      │
      ├── Mapeo Entity → DTO (sin rutas locales, sin isSynced)
      │
      ├── Retrofit → OkHttp → POST /api/v1/sync
      │   [OkHttp LoggingInterceptor imprime JSON en Logcat]
      │
      ├── si 2xx: Room markSynced() × 4 tablas
      │   → SyncResult.Success(count)
      │
      └── si error red: SyncResult.Error("Sin conexión…")
          si error servidor: SyncResult.Error("Error 500…")

  → _lastResult = SyncResult.Success/Error
  → _isSyncing = false
  → SyncScreen recompone:
      - Botón vuelve a "Sincronizar ahora"
      - Card verde "✓ N registros sincronizados"
        o Card roja "✗ mensaje de error"
  → counts.total sigue siendo el total en Room
    (los registros no se borran, solo se marcan isSynced = true)
```

---

## 16. Conceptos de la Lista 3 cubiertos

| Tema | L3 | Cómo se ve en este lab |
|---|---|---|
| HTTP y REST | L3 | `@POST` en `ApiService`; JSON con Gson; respuesta `Response<T>` |
| Corrutinas | L3 | `sync()` usa `viewModelScope.launch`; `syncAll()` es `suspend`; `await()` implícito en Retrofit |
| Capa de Datos | L3 | `Migration(2,3)` con `ALTER TABLE`; DAOs extendidos con `getUnsynced()` y `markSynced()` |
| Repositorio | L3 | `SyncRepository` orquesta 4 repositorios + API; single responsibility |
| Retrofit | L3 | `ApiService` con `@POST`; `GsonConverterFactory`; `Response<T>` |
| OkHttp | L3 | `OkHttpClient` con timeouts; `HttpLoggingInterceptor` |
| DI manual | L3 | `SyncRepository` en `DemoDataApp by lazy`; `SyncViewModel.Factory` con 4 dependencias |
| Views y Compose | L3 | `SyncScreen` con `CircularProgressIndicator` durante sync; `when(result)` para mostrar éxito/error |
| Firebase Tools | L3 | ❌ No implementado en este lab |

---

## 17. Lo que aún falta (trabajo futuro)

| Característica | Descripción |
|---|---|
| **Subida de archivos binarios** | Fotos/videos/audios requieren `Multipart` en Retrofit, no JSON. Necesita `@Multipart` + `@Part` en el `ApiService` |
| **WorkManager periódico** | Sync automático cada 15 min en background con `PeriodicWorkRequest` |
| **Token de autenticación** | `OkHttp Interceptor` que agrega `Authorization: Bearer <token>` a cada request |
| **`PasswordHasher` integrado** | Login real con PBKDF2: hashear la contraseña, compararla con el hash guardado en `EncryptedSharedPreferences` |
| **Firebase Auth** | Reemplazar las credenciales hardcodeadas `jkn/jkn` con Google Sign-In |
| **Firebase Firestore** | Alternativa a Retrofit para sincronización en tiempo real |
| **Reintentos automáticos** | `@Retry` con backoff exponencial cuando el servidor falla temporalmente |
| **Sync incremental** | Solo enviar registros nuevos desde la última sync, usando `lastSyncTimestamp` en DataStore |

---

## 18. Checklist del laboratorio

### Migración de la BD
- [ ] `isSynced: Boolean = false` agregado a las 4 entidades
- [ ] `DemoDataDatabase` versión 3
- [ ] `MIGRATION_2_3` con `ALTER TABLE` × 4 tablas
- [ ] `fallbackToDestructiveMigration()` reemplazado por `.addMigrations(MIGRATION_2_3)`

### Capa de red
- [ ] Dependencias Retrofit, OkHttp, Gson en `build.gradle.kts`
- [ ] DTOs creados: `GpsGoogleDto`, `GpsSensorsDto`, `MediaDto`, `AudioDto`, `SyncBatchRequest`, `SyncBatchResponse`
- [ ] `ApiService` con al menos un endpoint `@POST`
- [ ] `NetworkManager` con `OkHttpClient` (timeouts + logging) + `Retrofit`

### Capa de datos
- [ ] `getUnsynced()` y `markSynced(ids)` en los 4 DAOs
- [ ] `SyncRepository.syncAll()` lee Room → mapea DTOs → POST → marca sincronizado
- [ ] `SyncRepository` maneja `IOException` y errores HTTP correctamente

### ViewModel y UI
- [ ] `SyncViewModel` extendido con `isSyncing` y `lastResult`
- [ ] `SyncViewModel.sync()` protegido contra doble tap (`if (_isSyncing.value) return`)
- [ ] `SyncRepository` registrado en `DemoDataApp by lazy`
- [ ] `SyncScreen` muestra `CircularProgressIndicator` durante la sync
- [ ] `SyncScreen` muestra card verde en éxito y card roja en error
- [ ] `SyncScreen` deshabilita el botón si `counts.total == 0` o `isSyncing == true`
- [ ] App compila, conecta al endpoint, y los registros se marcan `isSynced = true` en Room
