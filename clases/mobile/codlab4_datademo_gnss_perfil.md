# Lab 4 — GNSS y Perfil de Usuario

**Autor:** [illarek-lab](https://github.com/illarek-lab)
**Proyecto:** DemoData · `com.illareklab.demodata`
**Temas:** L1 (StateFlow, ViewModel, Navegación Compose) · L2 (Temas, Menús, Layouts, Containers, Adaptadores) · L3 (Corrutinas, Capa de Datos, Repositorio, DI manual)

> **Objetivo del laboratorio:** Comprender cómo se capturan, almacenan y visualizan coordenadas GNSS desde dos fuentes simultáneas (Google FLP y chip de hardware), y cómo se construye una pantalla de perfil con múltiples sub-vistas, toggle de modo noche persistente y un historial unificado de actividad.

---

## 1. Arquitectura del laboratorio

### Árbol de archivos del proyecto

```
app/src/main/
├── AndroidManifest.xml
└── java/com/illareklab/demodata/
    │
    ├── DemoDataApp.kt
    ├── MainActivity.kt
    │
    ├── data/
    │   ├── local/
    │   │   ├── AppDatabase.kt
    │   │   ├── dao/
    │   │   │   ├── GpsGoogleDao.kt
    │   │   │   └── GpsSensorsDao.kt
    │   │   └── entity/
    │   │       ├── GpsGoogleEntity.kt
    │   │       └── GpsSensorsEntity.kt
    │   ├── repository/
    │   │   └── GpsRepository.kt
    │   └── session/
    │       └── SessionManager.kt
    │
    ├── services/
    │   └── GpsCaptureService.kt
    │
    └── ui/
        ├── navigation/
        │   └── AppNavigation.kt
        ├── screens/
        │   ├── GpsScreen.kt
        │   └── ProfileScreen.kt
        ├── theme/
        │   ├── Color.kt
        │   ├── Theme.kt
        │   └── Type.kt
        └── viewmodel/
            ├── GpsViewModel.kt
            └── SessionViewModel.kt
```

### Diagrama de flujo de datos

```
GpsCaptureService (Foreground Service)
    │
    ├── FusedLocationProvider → GpsGoogleEntity → Room (gps_google)
    └── LocationManager chip → GpsSensorsEntity → Room (gps_sensors)
                                    │
                              GpsRepository
                                    │
                              GpsViewModel
                              ├── googlePoints: StateFlow<List<GpsGoogleEntity>>
                              ├── sensorsPoints: StateFlow<List<GpsSensorsEntity>>
                              └── comparativeHistory: StateFlow<List<ComparativeGpsRecord>>
                                    │
                              GpsScreen
                              └── ComparativeCaptureCard (dual-panel por instante)

SessionManager (DataStore)
    └── isDarkMode: Flow<Boolean?>
            │
    SessionViewModel
    └── isDarkMode: StateFlow<Boolean?>
            │
    ProfileScreen
    ├── ProfileMenu       → logout con AlertDialog
    ├── MyProfileScreen   → metadata + Switch modo noche
    └── MyActivityScreen  → lista de actividad registrada
```

---

## 2. Dependencias (`app/build.gradle.kts`)

Estas son las dependencias necesarias para el Lab 4. La mayoría ya están en el proyecto; se listan aquí para referencia.

### Archivo `gradle/libs.versions.toml` — entradas relevantes

```toml
[versions]
kotlin       = "2.2.10"
ksp          = "2.2.10-2.0.2"   # debe coincidir exactamente con la versión de Kotlin
androidxRoom = "2.7.0-alpha11"

[libraries]
androidx-room-runtime  = { group = "androidx.room", name = "room-runtime",  version.ref = "androidxRoom" }
androidx-room-ktx      = { group = "androidx.room", name = "room-ktx",      version.ref = "androidxRoom" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler",  version.ref = "androidxRoom" }

[plugins]
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

> **Nota:** Parche temporal, en `gradle.properties` añadir la fila `android.disallowKotlinSourceSets=false`.


```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.ksp)               // procesador de anotaciones Room
}

ksp {
    arg("room.generateKotlin", "true")    // Room emite código Kotlin puro
    arg("useK2", "true")                  // compilador K2
}

dependencies {
    // ── Compose BOM (gestiona versiones de todos los artefactos Compose) ──
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.material3)
    implementation("androidx.compose.material:material-icons-extended:1.7.8")
    implementation("androidx.navigation:navigation-compose:2.8.0")

    // ── Room (SQLite) ──
    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)          // extensiones suspend + Flow
    ksp(libs.androidx.room.compiler)                 // generador de código en tiempo de compilación

    // ── DataStore (reemplaza SharedPreferences) ──
    implementation("androidx.datastore:datastore-preferences:1.1.1")

    // ── ViewModel + lifecycle ──
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.6")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.6")

    // ── Google Fused Location Provider + bridge de coroutines ──
    implementation("com.google.android.gms:play-services-location:21.3.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.11.0")  // .await()

    // ── Coil (thumbnails de archivos en ProfileScreen → MyActivity) ──
    implementation("io.coil-kt:coil-compose:2.7.0")

    // ── Accompanist: gestión de permisos en runtime con Compose ──
    implementation("com.google.accompanist:accompanist-permissions:0.34.0")
}
```


> **Nota:** `room-compiler` va con `ksp(...)` en las dependencias, no con `implementation(...)`. Si se pone con `implementation` Room no genera el código y el proyecto no compila.

---

## 3. `AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application
        android:name=".DemoDataApp"
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.DemoData">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:label="@string/app_name"
            android:theme="@style/Theme.DemoData">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service
            android:name=".services.GpsCaptureService"
            android:enabled="true"
            android:exported="false"
            android:foregroundServiceType="location" />

    </application>

</manifest>
```

**`android:name=".DemoDataApp"`** le indica al sistema que, antes de crear cualquier `Activity` o `Service`, instancie esta clase como el objeto `Application` de todo el proceso. Gracias a eso, `GpsCaptureService` y `MainActivity` pueden obtener el repositorio con un simple cast `(application as DemoDataApp).gpsRepository` en lugar de cada uno inicializar su propia instancia de `AppDatabase`.

**Permisos declarados:**

| Permiso | Para qué se usa |
|---|---|
| `ACCESS_FINE_LOCATION` | Precisión GPS completa para FLP y LocationManager |
| `ACCESS_COARSE_LOCATION` | Ubicación por red (complemento del anterior) |
| `FOREGROUND_SERVICE` | Permite iniciar el servicio en primer plano |
| `FOREGROUND_SERVICE_LOCATION` | Tipo específico de servicio de primer plano (obligatorio desde Android 10) |
| `POST_NOTIFICATIONS` | Mostrar la notificación persistente del servicio (obligatorio desde Android 13) |

`foregroundServiceType="location"` en la declaración del `<service>` es obligatorio desde Android 10. Sin él, `startForeground()` lanza `MissingForegroundServiceTypeException` en tiempo de ejecución.

---

## 4. `DemoDataApp.kt` — Clase Application

```kotlin
package com.illareklab.demodata

import android.app.Application
import com.illareklab.demodata.data.local.AppDatabase
import com.illareklab.demodata.data.repository.GpsRepository
import com.illareklab.demodata.data.session.SessionManager

class DemoDataApp : Application() {

    // La BD se crea una sola vez cuando se accede por primera vez
    val database by lazy { AppDatabase.getDatabase(this) }

    // El repositorio se construye sobre la misma instancia de la BD
    val gpsRepository by lazy {
        GpsRepository(database.gpsGoogleDao(), database.gpsSensorsDao())
    }

    // SessionManager también vive aquí para no duplicarlo en MainActivity
    val sessionManager by lazy { SessionManager(this) }
}
```

**¿Por qué una clase `Application`?**

Sin `DemoDataApp`, cada componente que necesita el repositorio construye su propia cadena:

```
GpsCaptureService  →  AppDatabase.getDatabase()  →  GpsRepository(dao1, dao2)
MainActivity       →  AppDatabase.getDatabase()  →  GpsRepository(dao1, dao2)
```

`AppDatabase` usa `@Volatile + synchronized` para garantizar que solo haya una instancia, pero cada componente duplica el código de inicialización. Con `DemoDataApp`:

```
DemoDataApp (proceso único)
    ├── database      by lazy   ← una sola BD para todo el proceso
    ├── gpsRepository by lazy   ← un solo repositorio compartido
    └── sessionManager by lazy  ← un solo gestor de sesión

GpsCaptureService  →  (application as DemoDataApp).gpsRepository
MainActivity       →  (application as DemoDataApp).gpsRepository
                      (application as DemoDataApp).sessionManager
```

**`by lazy`:** cada propiedad se inicializa solo cuando se accede por primera vez y el resultado se reutiliza en llamadas posteriores. Si el servicio arranca antes que la Activity, la BD se crea en ese momento; cuando la Activity la pide, ya existe — no se crea dos veces.

**Ciclo de vida:** `Application` vive mientras el proceso esté activo. Es el scope correcto para objetos de infraestructura como la BD o el DataStore, cuya vida debe superar a cualquier Activity o Service individual.

---

## 5. Componentes cubiertos en este lab

| Componente | Archivo |
|---|---|
| Manifiesto | `AndroidManifest.xml` |
| Clase Application | `DemoDataApp.kt` |
| Actividad principal | `MainActivity.kt` |
| Entidad GPS Google | `data/local/entity/GpsGoogleEntity.kt` |
| Entidad GPS Sensores | `data/local/entity/GpsSensorsEntity.kt` |
| Base de datos Room | `data/local/AppDatabase.kt` |
| DAOs GPS | `data/local/dao/GpsGoogleDao.kt`, `GpsSensorsDao.kt` |
| Repositorio GPS | `data/repository/GpsRepository.kt` |
| ViewModel GPS | `ui/viewmodel/GpsViewModel.kt` |
| Servicio de captura | `services/GpsCaptureService.kt` |
| Navegación principal | `ui/navigation/AppNavigation.kt` |
| Pantalla GNSS | `ui/screens/GpsScreen.kt` |
| SessionManager | `data/session/SessionManager.kt` |
| SessionViewModel | `ui/viewmodel/SessionViewModel.kt` |
| Pantalla Perfil | `ui/screens/ProfileScreen.kt` |

---

## 6. Capa de datos — Entidades GNSS

### 4.1 `GpsGoogleEntity` — Fuente Google FLP

```kotlin
package com.illareklab.demodata.data.local.entity

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "gps_google")
data class GpsGoogleEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0L,
    val latitude: Double?,      // null si FLP no logra fijar posición
    val longitude: Double?,     // null si FLP no logra fijar posición
    val accuracy: Float?,       // precisión horizontal en metros; null si no disponible
    val speed: Float? = null,   // m/s — null si el dispositivo no lo reporta
    val bearing: Float? = null, // grados desde el norte
    val timestamp: Long         // System.currentTimeMillis() en UTC
)
```

### 4.2 `GpsSensorsEntity` — Fuente chip GNSS

```kotlin
package com.illareklab.demodata.data.local.entity

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "gps_sensors")
data class GpsSensorsEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0L,
    val latitude: Double?,      // NULLABLE: null cuando no hay fix satelital
    val longitude: Double?,     // NULLABLE
    val provider: String,       // "gps", "network" o "passive"
    val altitude: Double? = null,
    val satellites: Int? = null,
    val timestamp: Long
)
```

**¿Por qué `latitude`, `longitude` y `accuracy` son nullable en ambas entidades?**

El comportamiento entre las dos fuentes es diferente pero ambas pueden no devolver datos:

- **`GpsSensorsEntity`:** el chip GNSS necesita visión directa a satélites. En interiores puede no tener fix. Si el timeout de 5 s pasa sin respuesta, guardamos `latitude = null`.
- **`GpsGoogleEntity`:** el FLP combina GPS, WiFi y redes móviles, y casi siempre devuelve una ubicación. Sin embargo, en condiciones extremas (modo avión, GPS desactivado completamente) puede no tener dato de precisión (`accuracy = null`) o incluso coordenadas. Hacer los campos nullable evita crashes inesperados y permite registrar el evento igualmente.

En ambos casos la UI muestra **"SIN SEÑAL"** en rojo cuando `latitude == null`.

### 4.3 `AppDatabase` — Singleton de Room

```kotlin
package com.illareklab.demodata.data.local

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import com.illareklab.demodata.data.local.dao.GpsGoogleDao
import com.illareklab.demodata.data.local.dao.GpsSensorsDao
import com.illareklab.demodata.data.local.entity.GpsGoogleEntity
import com.illareklab.demodata.data.local.entity.GpsSensorsEntity

@Database(
    entities = [GpsGoogleEntity::class, GpsSensorsEntity::class],
    version = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {

    abstract fun gpsGoogleDao(): GpsGoogleDao
    abstract fun gpsSensorsDao(): GpsSensorsDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "demo_data_db"
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

`@Volatile` garantiza que todos los hilos lean el mismo valor de `INSTANCE`. El bloque `synchronized` evita que dos hilos creen dos instancias simultáneamente. Room no permite múltiples instancias de la misma BD en el mismo proceso.

`DemoDataApp` llama a `AppDatabase.getDatabase(this)` una única vez y expone la instancia resultante como propiedad `database`. Tanto `MainActivity` como `GpsCaptureService` acceden a la misma BD a través de `(application as DemoDataApp).gpsRepository`.

---

## 7. DAOs

### `GpsGoogleDao.kt`

```kotlin
package com.illareklab.demodata.data.local.dao

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import com.illareklab.demodata.data.local.entity.GpsGoogleEntity
import kotlinx.coroutines.flow.Flow

@Dao
interface GpsGoogleDao {
    @Insert
    suspend fun insert(item: GpsGoogleEntity): Long

    @Query("SELECT * FROM gps_google ORDER BY timestamp DESC")
    fun observeAll(): Flow<List<GpsGoogleEntity>>

    @Query("SELECT COUNT(*) FROM gps_google")
    fun observeCount(): Flow<Int>

    @Query("DELETE FROM gps_google")
    suspend fun deleteAll()
}
```

### `GpsSensorsDao.kt`

```kotlin
package com.illareklab.demodata.data.local.dao

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import com.illareklab.demodata.data.local.entity.GpsSensorsEntity
import kotlinx.coroutines.flow.Flow

@Dao
interface GpsSensorsDao {
    @Insert
    suspend fun insert(item: GpsSensorsEntity): Long

    @Query("SELECT * FROM gps_sensors ORDER BY timestamp DESC")
    fun observeAll(): Flow<List<GpsSensorsEntity>>

    @Query("SELECT COUNT(*) FROM gps_sensors")
    fun observeCount(): Flow<Int>

    @Query("DELETE FROM gps_sensors")
    suspend fun deleteAll()
}
```

**¿Por qué `Flow<List<T>>` en vez de `suspend fun getAll(): List<T>`?**

`suspend fun` se ejecuta una sola vez y devuelve un snapshot estático. Si el `GpsCaptureService` inserta una nueva coordenada, la pantalla no se entera.

`Flow<List<T>>` emite automáticamente cada vez que la tabla cambia. Room genera internamente un trigger SQLite que notifica al Flow ante cualquier `INSERT`, `UPDATE` o `DELETE`. La UI se actualiza sola — sin llamar `refresh()` manualmente.

---

## 8. Repositorio

```kotlin
package com.illareklab.demodata.data.repository

import com.illareklab.demodata.data.local.dao.GpsGoogleDao
import com.illareklab.demodata.data.local.dao.GpsSensorsDao
import com.illareklab.demodata.data.local.entity.GpsGoogleEntity
import com.illareklab.demodata.data.local.entity.GpsSensorsEntity
import kotlinx.coroutines.flow.Flow

class GpsRepository(
    private val googleDao: GpsGoogleDao,
    private val sensorsDao: GpsSensorsDao
) {
    val googlePoints: Flow<List<GpsGoogleEntity>> = googleDao.observeAll()
    val sensorsPoints: Flow<List<GpsSensorsEntity>> = sensorsDao.observeAll()

    val googleCount: Flow<Int> = googleDao.observeCount()
    val sensorsCount: Flow<Int> = sensorsDao.observeCount()

    suspend fun saveGooglePoint(point: GpsGoogleEntity) = googleDao.insert(point)
    suspend fun saveSensorsPoint(point: GpsSensorsEntity) = sensorsDao.insert(point)

    suspend fun clearAll() {
        googleDao.deleteAll()
        sensorsDao.deleteAll()
    }
}
```

El repositorio es la **única fachada** entre la UI y los DAOs. El `GpsCaptureService` escribe a través del repositorio; el `GpsViewModel` lee a través del repositorio. Ningún componente toca los DAOs directamente.

---

## 9. `GpsViewModel` — Historial comparativo

```kotlin
package com.illareklab.demodata.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.illareklab.demodata.data.local.entity.GpsGoogleEntity
import com.illareklab.demodata.data.local.entity.GpsSensorsEntity
import com.illareklab.demodata.data.repository.GpsRepository
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.flowOn
import kotlinx.coroutines.flow.stateIn

// Estructura intermedia para emparejar ambos registros en un mismo instante
data class ComparativeGpsRecord(
    val timestamp: Long,
    val google: GpsGoogleEntity?,
    val sensors: GpsSensorsEntity?
)

class GpsViewModel(private val gpsRepository: GpsRepository) : ViewModel() {

    val googlePoints = gpsRepository.googlePoints.stateIn(
        viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList()
    )

    val sensorsPoints = gpsRepository.sensorsPoints.stateIn(
        viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList()
    )

    // Combina ambos canales asíncronos en una sola lista unificada
    val comparativeHistory = combine(
        gpsRepository.googlePoints,
        gpsRepository.sensorsPoints
    ) { gList, sList ->
        val allTimestamps = (gList.map { it.timestamp } + sList.map { it.timestamp })
            .distinct()
            .sortedDescending()

        allTimestamps.map { ts ->
            ComparativeGpsRecord(
                timestamp = ts,
                google  = gList.find { it.timestamp == ts },
                sensors = sList.find { it.timestamp == ts }
            )
        }
    }
        .flowOn(Dispatchers.Default)   // Envía el cálculo pesado fuera del hilo de la UI
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
}
```

### Puntos clave para la clase

**`combine`:** toma N flows y emite un valor nuevo cada vez que cualquiera cambia. Aquí combina las dos listas de Room en una sola vista comparativa sin duplicar consultas.

**`.flowOn(Dispatchers.Default)`:** el operador `combine` y la construcción del mapa se ejecutan en el pool de hilos de CPU, no en el Main Thread. Sin esto, listas largas causarían el warning "Skipped N frames" en la UI.

**`SharingStarted.WhileSubscribed(5_000)`:** el flow se mantiene activo 5 segundos después de que el último suscriptor desaparece. Esto evita re-suscripciones costosas a Room durante rotaciones de pantalla (el proceso de rotación desmonta y remonta el Composable en menos de 5 s).

**Emparejamiento por `timestamp`:** el `GpsCaptureService` captura ambas fuentes en el mismo ciclo y asigna el mismo valor de `System.currentTimeMillis()` a ambos registros, lo que hace el emparejamiento exacto.

**Sin `ViewModelProvider.Factory`:** el `GpsViewModel` se instancia directamente en `MainActivity` pasando el repositorio como constructor:

```kotlin
val gpsViewModel = GpsViewModel(gpsRepository)
```

---

## 10. `GpsCaptureService` — Loop con coroutines

```kotlin
package com.illareklab.demodata.services

import android.annotation.SuppressLint
import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.Service
import android.content.Context
import android.content.Intent
import android.location.Location
import android.location.LocationManager
import android.os.Build
import android.os.IBinder
import androidx.core.app.NotificationCompat
import com.illareklab.demodata.DemoDataApp
import com.illareklab.demodata.data.local.entity.GpsGoogleEntity
import com.illareklab.demodata.data.local.entity.GpsSensorsEntity
import com.google.android.gms.location.LocationServices
import com.google.android.gms.location.Priority
import kotlinx.coroutines.*
import kotlinx.coroutines.tasks.await

class GpsCaptureService : Service() {

    companion object {
        private const val INTERVAL_MS        = 10_000L
        private const val SENSOR_TIMEOUT_MS  = 5_000L
        private const val NOTIFICATION_ID    = 1001
        private const val CHANNEL_ID         = "gps_capture_channel"
    }

    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    private var captureJob: Job? = null

    // El repositorio ya existe en DemoDataApp — solo lo referenciamos
    private val gpsRepo by lazy { (application as DemoDataApp).gpsRepository }

    private val fusedClient by lazy {
        LocationServices.getFusedLocationProviderClient(applicationContext)
    }

    private val locationManager by lazy {
        getSystemService(Context.LOCATION_SERVICE) as LocationManager
    }

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
        startForeground(NOTIFICATION_ID, createNotification())
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        if (captureJob == null) {
            captureJob = scope.launch {
                while (isActive) {
                    performCaptures()
                    delay(INTERVAL_MS)
                }
            }
        }
        return START_STICKY
    }

    private suspend fun performCaptures() {
        val now = System.currentTimeMillis()     // mismo timestamp para ambas fuentes

        // 1. Google FLP — siempre devuelve ubicación cuando hay conectividad
        try {
            val loc = fusedClient
                .getCurrentLocation(Priority.PRIORITY_HIGH_ACCURACY, null)
                .await()                         // suspende sin bloquear el hilo
            loc?.let {
                gpsRepo.saveGooglePoint(GpsGoogleEntity(
                    latitude  = it.latitude,
                    longitude = it.longitude,
                    accuracy  = it.accuracy,
                    speed     = if (it.hasSpeed()) it.speed else null,
                    bearing   = if (it.hasBearing()) it.bearing else null,
                    timestamp = now
                ))
            }
        } catch (e: Exception) {
            // Manejo de excepciones en caso de revocación de permisos o fallas del proveedor
        }

        // 2. Sensor de Hardware puro — puede no tener fix; timeout de 5 s
        try {
            val sensorLoc = withTimeoutOrNull(SENSOR_TIMEOUT_MS) { getRawGpsLocation() }
            gpsRepo.saveSensorsPoint(GpsSensorsEntity(
                latitude  = sensorLoc?.latitude,   // null si se cumple el timeout
                longitude = sensorLoc?.longitude,
                provider  = LocationManager.GPS_PROVIDER,
                altitude  = if (sensorLoc?.hasAltitude() == true) sensorLoc.altitude else null,
                timestamp = now
            ))
        } catch (e: Exception) {
            // Failsafe pasivo
        }
    }

    // Adaptador suspendible para emular el flujo lineal sobre la API legacy de LocationManager
    @SuppressLint("MissingPermission")
    private suspend fun getRawGpsLocation(): Location? = suspendCancellableCoroutine { continuation ->
        val listener = android.location.LocationListener { location ->
            if (continuation.isActive) continuation.resume(location, null)
        }

        try {
            locationManager.requestLocationUpdates(
                LocationManager.GPS_PROVIDER,
                0L,
                0f,
                listener,
                mainLooper
            )
        } catch (e: Exception) {
            if (continuation.isActive) continuation.resumeWith(Result.failure(e))
        }

        continuation.invokeOnCancellation {
            locationManager.removeUpdates(listener)
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        captureJob?.cancel()     // Cancela de inmediato el bucle de la corutina al apagar el servicio
    }

    private fun createNotification(): Notification {
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Captura GNSS Activa")
            .setContentText("Registrando coordenadas en paralelo cada 10s...")
            .setSmallIcon(android.R.drawable.ic_menu_mylocation)
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .build()
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "Servicio GNSS",
                NotificationManager.IMPORTANCE_LOW
            )
            getSystemService(NotificationManager::class.java)
                ?.createNotificationChannel(channel)
        }
    }
}
```

### ¿Por qué un loop de coroutine en vez de callbacks?

| Callbacks (diseño original) | Loop de coroutine (implementación real) |
|---|---|
| `LocationCallback` activo permanentemente | Solicitud puntual por ciclo (`getCurrentLocation`) |
| `LocationListener` activo permanentemente | `suspendCancellableCoroutine` efímero con timeout |
| Necesita `removeUpdates` en `onDestroy` | Solo `captureJob?.cancel()` |
| Difícil de razonar (concurrencia implícita) | Flujo lineal: captura → espera → captura |
| No registra "sin señal" | Guarda `lat = null` si el timeout expira |

El loop de coroutine hace el código **secuencial y predecible**: cada ciclo termina antes de empezar el siguiente, sin overlap entre capturas.

### `getRawGpsLocation` — `suspendCancellableCoroutine`

La API del `LocationManager` es basada en callbacks (API legacy). `suspendCancellableCoroutine` la convierte en una función suspendible:

1. Registra un `LocationListener` efímero.
2. Cuando llega la primera ubicación, llama `continuation.resume(location, null)` y suspende hasta que `withTimeoutOrNull` la cancele o la ubicación llegue.
3. `invokeOnCancellation` elimina el listener cuando el timeout de 5 s expira, evitando fugas de memoria.

---

## 11. `AppNavigation` — NavHost con BottomBar

```kotlin
package com.illareklab.demodata.ui.navigation

import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Person
import androidx.compose.material.icons.filled.Place
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.navigation.NavDestination.Companion.hierarchy
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.currentBackStackEntryAsState
import androidx.navigation.compose.rememberNavController
import com.illareklab.demodata.ui.screens.GpsScreen
import com.illareklab.demodata.ui.screens.ProfileScreen
import com.illareklab.demodata.ui.viewmodel.GpsViewModel
import com.illareklab.demodata.ui.viewmodel.SessionViewModel

sealed class Ruta(val ruta: String, val etiqueta: String, val icono: ImageVector) {
    object Gps    : Ruta("gps",    "Captura GNSS", Icons.Default.Place)
    object Perfil : Ruta("perfil", "Mi Perfil",    Icons.Default.Person)
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppNavigation(gpsViewModel: GpsViewModel, sessionViewModel: SessionViewModel) {
    val navController = rememberNavController()
    val currentBackStack by navController.currentBackStackEntryAsState()
    val currentDestination = currentBackStack?.destination

    val pestañas = listOf(Ruta.Gps, Ruta.Perfil)

    val tituloActual = when (currentDestination?.route) {
        Ruta.Gps.ruta    -> "Laboratorio 4: GNSS Dual"
        Ruta.Perfil.ruta -> "Panel de Usuario"
        else             -> "DemoData"
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text(tituloActual) },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor    = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        },
        bottomBar = {
            NavigationBar(containerColor = MaterialTheme.colorScheme.surfaceContainer) {
                pestañas.forEach { pestaña ->
                    val seleccionada = currentDestination?.hierarchy
                        ?.any { it.route == pestaña.ruta } == true
                    NavigationBarItem(
                        selected = seleccionada,
                        onClick  = {
                            navController.navigate(pestaña.ruta) {
                                popUpTo(navController.graph.startDestinationId) { saveState = true }
                                launchSingleTop = true
                                restoreState    = true
                            }
                        },
                        icon  = { Icon(pestaña.icono, contentDescription = pestaña.etiqueta) },
                        label = { Text(pestaña.etiqueta) }
                    )
                }
            }
        }
    ) { paddingValues ->
        NavHost(
            navController    = navController,
            startDestination = Ruta.Gps.ruta,
            modifier         = Modifier.padding(paddingValues)
        ) {
            composable(Ruta.Gps.ruta) {
                GpsScreen(viewModel = gpsViewModel)
            }
            composable(Ruta.Perfil.ruta) {
                ProfileScreen(
                    sessionVm = sessionViewModel,
                    onLogout  = { sessionViewModel.logout() }
                )
            }
        }
    }
}
```

**Puntos clave:**

- `sealed class Ruta` centraliza las rutas, etiquetas e íconos de la barra de navegación. Agregar un nuevo tab solo requiere una nueva subclase y añadirla a `pestañas`.
- `popUpTo + saveState + restoreState` evita acumular destinos en el back stack al cambiar de tab. El estado de cada tab se preserva al volver a él.
- `launchSingleTop = true` evita crear instancias duplicadas si el usuario toca el tab ya activo.
- Los ViewModels se pasan por parámetro desde `MainActivity`, no se instancian dentro de los composables con `viewModel()`, porque la DI es manual.

---

## 12. `GpsScreen.kt` — Permisos y tarjetas duales

```kotlin
package com.illareklab.demodata.ui.screens

import android.Manifest
import android.content.Intent
import android.os.Build
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.PlayArrow
import androidx.compose.material.icons.filled.Stop
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.illareklab.demodata.data.local.entity.GpsGoogleEntity
import com.illareklab.demodata.data.local.entity.GpsSensorsEntity
import com.illareklab.demodata.ui.viewmodel.ComparativeGpsRecord
import com.illareklab.demodata.ui.viewmodel.GpsViewModel
import com.illareklab.demodata.services.GpsCaptureService
import com.google.accompanist.permissions.ExperimentalPermissionsApi
import com.google.accompanist.permissions.rememberMultiplePermissionsState
import java.text.SimpleDateFormat
import java.util.*

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun GpsScreen(viewModel: GpsViewModel) {
    val context = LocalContext.current

    // Lista de permisos requeridos por el laboratorio
    val permisos = buildList {
        add(Manifest.permission.ACCESS_FINE_LOCATION)
        add(Manifest.permission.ACCESS_COARSE_LOCATION)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            add(Manifest.permission.POST_NOTIFICATIONS)
        }
    }
    val estadoPermisos = rememberMultiplePermissionsState(permissions = permisos)

    // Estado local para saber si el servicio está corriendo
    var capturando by remember { mutableStateOf(false) }

    // Recolectamos los datos reactivos del ViewModel
    val googlePoints  by viewModel.googlePoints.collectAsStateWithLifecycle()
    val sensorsPoints by viewModel.sensorsPoints.collectAsStateWithLifecycle()
    val history       by viewModel.comparativeHistory.collectAsStateWithLifecycle()

    val dateFormat = remember { SimpleDateFormat("HH:mm:ss", Locale.getDefault()) }

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // Bloqueo temprano: si no hay permisos mostramos Card de error y retornamos
        if (!estadoPermisos.allPermissionsGranted) {
            Card(
                colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.errorContainer),
                modifier = Modifier.fillMaxWidth()
            ) {
                Column(
                    modifier = Modifier.padding(16.dp),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Text(
                        text  = "Se requieren permisos de ubicación para este laboratorio.",
                        style = MaterialTheme.typography.bodyMedium,
                        color = MaterialTheme.colorScheme.onErrorContainer
                    )
                    Spacer(modifier = Modifier.height(8.dp))
                    Button(onClick = { estadoPermisos.launchMultiplePermissionRequest() }) {
                        Text("Conceder permisos")
                    }
                }
            }
            return@Column
        }

        // Botón de control reactivo con cambio de color semántico
        Button(
            onClick = {
                capturando = !capturando
                val intent = Intent(context, GpsCaptureService::class.java)
                if (capturando) context.startForegroundService(intent)
                else            context.stopService(intent)
            },
            colors   = ButtonDefaults.buttonColors(
                containerColor = if (capturando) MaterialTheme.colorScheme.error
                                 else            MaterialTheme.colorScheme.primary
            ),
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(
                if (capturando) Icons.Default.Stop else Icons.Default.PlayArrow,
                contentDescription = null
            )
            Spacer(modifier = Modifier.width(8.dp))
            Text(if (capturando) "Detener captura" else "Capturar coordenada (cada 10 s)")
        }

        // Contadores en vivo utilizando tarjetas contenedoras
        Row(
            modifier              = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            Card(
                colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer),
                modifier = Modifier.weight(1f)
            ) {
                Column(modifier = Modifier.padding(12.dp), horizontalAlignment = Alignment.CenterHorizontally) {
                    Text("Google FLP",   style = MaterialTheme.typography.titleSmall)
                    Text("${googlePoints.size}", style = MaterialTheme.typography.headlineMedium, fontWeight = FontWeight.Bold)
                    Text("registros",   style = MaterialTheme.typography.labelSmall)
                }
            }
            Card(
                colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.secondaryContainer),
                modifier = Modifier.weight(1f)
            ) {
                Column(modifier = Modifier.padding(12.dp), horizontalAlignment = Alignment.CenterHorizontally) {
                    Text("Sensores GNSS", style = MaterialTheme.typography.titleSmall)
                    Text("${sensorsPoints.size}", style = MaterialTheme.typography.headlineMedium, fontWeight = FontWeight.Bold)
                    Text("registros",    style = MaterialTheme.typography.labelSmall)
                }
            }
        }

        Text("Historial Comparativo", style = MaterialTheme.typography.titleMedium, fontWeight = FontWeight.Bold)

        // Lista optimizada usando claves basadas en el timestamp
        LazyColumn(
            verticalArrangement = Arrangement.spacedBy(8.dp),
            modifier            = Modifier.fillMaxSize()
        ) {
            items(items = history, key = { it.timestamp }) { record ->
                ComparativeCaptureCard(record, dateFormat)
            }
        }
    }
}

@Composable
fun ComparativeCaptureCard(record: ComparativeGpsRecord, dateFormat: SimpleDateFormat) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceContainerLow)
    ) {
        Column(modifier = Modifier.padding(12.dp)) {
            Text(
                text       = "Instante: ${dateFormat.format(Date(record.timestamp))}",
                style      = MaterialTheme.typography.labelMedium,
                fontWeight = FontWeight.Bold,
                color      = MaterialTheme.colorScheme.secondary
            )
            HorizontalDivider(modifier = Modifier.padding(vertical = 8.dp))
            Row(modifier = Modifier.fillMaxWidth()) {

                // Panel Izquierdo: Google FLP
                Column(modifier = Modifier.weight(1f)) {
                    Text("GOOGLE FLP", style = MaterialTheme.typography.labelSmall,
                         fontWeight = FontWeight.Bold, color = MaterialTheme.colorScheme.primary)
                    if (record.google != null) {
                        Text("Lat: ${record.google.latitude}",    style = MaterialTheme.typography.bodySmall)
                        Text("Lon: ${record.google.longitude}",   style = MaterialTheme.typography.bodySmall)
                        Text("Prec: ±${record.google.accuracy}m", style = MaterialTheme.typography.bodySmall)
                    } else {
                        Text("Buscando...", style = MaterialTheme.typography.bodySmall,
                             color = MaterialTheme.colorScheme.onSurfaceVariant)
                    }
                }

                // Panel Derecho: Chip de Hardware (Sensores)
                Column(modifier = Modifier.weight(1f)) {
                    Text("SENSOR GNSS", style = MaterialTheme.typography.labelSmall,
                         fontWeight = FontWeight.Bold, color = MaterialTheme.colorScheme.tertiary)
                    if (record.sensors != null) {
                        if (record.sensors.latitude != null) {
                            Text("Lat: ${record.sensors.latitude}",         style = MaterialTheme.typography.bodySmall)
                            Text("Lon: ${record.sensors.longitude}",        style = MaterialTheme.typography.bodySmall)
                            Text("Alt: ${record.sensors.altitude ?: 0.0}m", style = MaterialTheme.typography.bodySmall)
                        } else {
                            // Feedback visual inmediato en rojo si no hay fijación satelital
                            Text("SIN SEÑAL",        style = MaterialTheme.typography.bodySmall,
                                 fontWeight = FontWeight.Bold, color = MaterialTheme.colorScheme.error)
                            Text("Indoors / Sin Fix", style = MaterialTheme.typography.bodySmall,
                                 color = MaterialTheme.colorScheme.error)
                        }
                    } else {
                        Text("Buscando...", style = MaterialTheme.typography.bodySmall,
                             color = MaterialTheme.colorScheme.onSurfaceVariant)
                    }
                }
            }
        }
    }
}
```

### Diseño dual panel

```
┌──────────────────────────────────────────────┐
│  Instante: 14:32:07                          │
├──────────────────┬───────────────────────────┤
│  GOOGLE FLP      │  SENSOR GNSS              │
│  Lat: -12.04318  │  Lat: -12.04305           │
│  Lon: -77.02824  │  Lon: -77.02819           │
│  Prec: ±8m       │  Alt: 154.0m              │
├──────────────────┼───────────────────────────┤
│  GOOGLE FLP      │  SENSOR GNSS              │
│  Lat: -12.04318  │  SIN SEÑAL                │
│  Lon: -77.02824  │  Indoors / Sin Fix        │
└──────────────────┴───────────────────────────┘
```

La columna de sensores usa `MaterialTheme.colorScheme.error` cuando `latitude == null`, dando feedback visual inmediato de la calidad de señal.

---

## 13. `SessionManager` — Tres claves en DataStore

```kotlin
package com.illareklab.demodata.data.session

import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.booleanPreferencesKey
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

// Extensión para instanciar DataStore de forma global en el Contexto
val Context.sessionDataStore: DataStore<Preferences> by preferencesDataStore(name = "session_prefs")

class SessionManager(private val context: Context) {

    private companion object {
        val KEY_IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
        val KEY_USERNAME     = stringPreferencesKey("username")
        val KEY_DARK_MODE    = booleanPreferencesKey("dark_mode")   // ← nueva en este lab
    }

    val isLoggedIn: Flow<Boolean> = context.sessionDataStore.data
        .map { it[KEY_IS_LOGGED_IN] ?: false }

    val currentUsername: Flow<String?> = context.sessionDataStore.data
        .map { it[KEY_USERNAME] }

    // null = seguir al sistema operativo; true/false = forzar modo
    val isDarkMode: Flow<Boolean?> = context.sessionDataStore.data
        .map { it[KEY_DARK_MODE] }

    suspend fun setDarkMode(enabled: Boolean) {
        context.sessionDataStore.edit { it[KEY_DARK_MODE] = enabled }
    }

    suspend fun login(username: String) {
        context.sessionDataStore.edit { prefs ->
            prefs[KEY_IS_LOGGED_IN] = true
            prefs[KEY_USERNAME]     = username
        }
    }

    suspend fun logout() {
        context.sessionDataStore.edit { prefs ->
            val currentTheme = prefs[KEY_DARK_MODE]   // preservar antes de clear
            prefs.clear()
            if (currentTheme != null) prefs[KEY_DARK_MODE] = currentTheme
        }
    }
}
```

**¿Por qué preservar `dark_mode` al hacer logout?** El tema es una preferencia visual del usuario sobre el dispositivo, no de la sesión. Resetear el tema al cerrar sesión sería sorpresivo y frustrante.

---

## 14. `SessionViewModel` — Dark mode reactivo

```kotlin
package com.illareklab.demodata.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.illareklab.demodata.data.session.SessionManager
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

class SessionViewModel(private val sessionManager: SessionManager) : ViewModel() {

    val isLoggedIn = sessionManager.isLoggedIn.stateIn(
        viewModelScope, SharingStarted.Eagerly, false
    )

    val username = sessionManager.currentUsername.stateIn(
        viewModelScope, SharingStarted.Eagerly, null
    )

    val isDarkMode = sessionManager.isDarkMode.stateIn(
        viewModelScope, SharingStarted.Eagerly, null   // null = sistema
    )

    fun setDarkMode(enabled: Boolean) {
        viewModelScope.launch { sessionManager.setDarkMode(enabled) }
    }

    fun logout() {
        viewModelScope.launch { sessionManager.logout() }
    }
}
```

`isDarkMode` viaja de DataStore → `SessionManager.Flow` → `SessionViewModel.StateFlow` → `MainActivity` → `AppTheme`. Cuando el usuario cambia el `Switch` en `ProfileScreen`, el tema de toda la app recompone en el mismo frame, sin reiniciar la `Activity`.

---

## 15. `ProfileScreen.kt` — Tres sub-vistas con `sealed class`

```kotlin
package com.illareklab.demodata.ui.screens

import android.os.Build
import androidx.compose.foundation.background
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ExitToApp
import androidx.compose.material.icons.automirrored.filled.KeyboardArrowRight
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.illareklab.demodata.ui.viewmodel.SessionViewModel

// Definición de las 3 sub-vistas internas del perfil usando sealed class
private sealed class ProfileViewState {
    object Menu       : ProfileViewState()
    object MyProfile  : ProfileViewState()
    object MyActivity : ProfileViewState()
}

data class OpcionPerfil(val id: Int, val titulo: String, val descripcion: String, val icono: ImageVector)

// ── PANTALLA RAÍZ: máquina de estados interna ──
@Composable
fun ProfileScreen(sessionVm: SessionViewModel, onLogout: () -> Unit) {
    var viewState by remember { mutableStateOf<ProfileViewState>(ProfileViewState.Menu) }
    val username by sessionVm.username.collectAsStateWithLifecycle()

    when (viewState) {
        is ProfileViewState.Menu -> ProfileMenu(
            username             = username ?: "Estudiante San Marcos",
            onNavigateToProfile  = { viewState = ProfileViewState.MyProfile },
            onNavigateToActivity = { viewState = ProfileViewState.MyActivity },
            onLogoutClick        = onLogout
        )
        is ProfileViewState.MyProfile -> MyProfileScreen(
            sessionVm = sessionVm,
            username  = username ?: "N/A",
            onBack    = { viewState = ProfileViewState.Menu }
        )
        is ProfileViewState.MyActivity -> MyActivityScreen(
            onBack = { viewState = ProfileViewState.Menu }
        )
    }
}

// ── SUB-PANTALLA 1: MENÚ PRINCIPAL DEL PERFIL ──
@Composable
private fun ProfileMenu(
    username: String,
    onNavigateToProfile: () -> Unit,
    onNavigateToActivity: () -> Unit,
    onLogoutClick: () -> Unit
) {
    var mostrarDialogo by remember { mutableStateOf(false) }

    val opciones = remember {
        listOf(
            OpcionPerfil(1, "Mis datos",              "Información del estudiante y dispositivo", Icons.Default.Person),
            OpcionPerfil(2, "Historial de Actividad", "Registros consolidados del sistema",       Icons.Default.Receipt)
        )
    }

    Column(modifier = Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {

        // Cabecera con Avatar Circular
        Card(
            modifier = Modifier.fillMaxWidth(),
            colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer)
        ) {
            Row(modifier = Modifier.fillMaxWidth().padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
                Box(
                    modifier         = Modifier.size(64.dp).clip(CircleShape).background(MaterialTheme.colorScheme.primary),
                    contentAlignment = Alignment.Center
                ) {
                    Icon(Icons.Default.Person, contentDescription = null,
                         tint = MaterialTheme.colorScheme.onPrimary, modifier = Modifier.size(36.dp))
                }
                Spacer(modifier = Modifier.width(16.dp))
                Column {
                    Text(username, style = MaterialTheme.typography.titleLarge,
                         fontWeight = FontWeight.Bold, color = MaterialTheme.colorScheme.onPrimaryContainer)
                    Text("9no Ciclo — UNMSM", style = MaterialTheme.typography.bodyMedium,
                         color = MaterialTheme.colorScheme.onPrimaryContainer)
                }
            }
        }

        // Opciones de menú como Cards navegables
        opciones.forEach { opcion ->
            Card(
                onClick  = { if (opcion.id == 1) onNavigateToProfile() else onNavigateToActivity() },
                modifier = Modifier.fillMaxWidth(),
                colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceContainerLow)
            ) {
                ListItem(
                    headlineContent  = { Text(opcion.titulo,       fontWeight = FontWeight.SemiBold) },
                    supportingContent = { Text(opcion.descripcion) },
                    leadingContent   = { Icon(opcion.icono, contentDescription = null, tint = MaterialTheme.colorScheme.primary) },
                    trailingContent  = { Icon(Icons.AutoMirrored.Filled.KeyboardArrowRight, contentDescription = null) },
                    colors           = ListItemDefaults.colors(containerColor = MaterialTheme.colorScheme.surfaceContainerLow)
                )
            }
        }

        Spacer(modifier = Modifier.weight(1f))

        // Botón de Cerrar Sesión semántico
        OutlinedButton(
            onClick  = { mostrarDialogo = true },
            modifier = Modifier.fillMaxWidth(),
            colors   = ButtonDefaults.outlinedButtonColors(contentColor = MaterialTheme.colorScheme.error)
        ) {
            Icon(Icons.AutoMirrored.Filled.ExitToApp, contentDescription = null)
            Spacer(modifier = Modifier.width(8.dp))
            Text("Cerrar sesión")
        }
    }

    // AlertDialog de confirmación para evitar cierres accidentales
    if (mostrarDialogo) {
        AlertDialog(
            onDismissRequest = { mostrarDialogo = false },
            title   = { Text("¿Confirmar cierre de sesión?") },
            text    = { Text("Tus preferencias visuales del dispositivo se conservarán.") },
            confirmButton = {
                TextButton(onClick = { mostrarDialogo = false; onLogoutClick() }) {
                    Text("Sí, cerrar", color = MaterialTheme.colorScheme.error)
                }
            },
            dismissButton = {
                TextButton(onClick = { mostrarDialogo = false }) { Text("Cancelar") }
            }
        )
    }
}

// ── SUB-PANTALLA 2: MIS DATOS Y CONFIGURACIÓN DE MODO OSCURO ──
@Composable
private fun MyProfileScreen(sessionVm: SessionViewModel, username: String, onBack: () -> Unit) {
    val context = LocalContext.current
    val isDarkModePref by sessionVm.isDarkMode.collectAsStateWithLifecycle()
    val isDark = isDarkModePref ?: isSystemInDarkTheme()

    Column(modifier = Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            IconButton(onClick = onBack) { Icon(Icons.Default.ArrowBack, contentDescription = "Regresar") }
            Text("Mis Datos", style = MaterialTheme.typography.titleLarge, fontWeight = FontWeight.Bold)
        }

        // Metadatos pedagógicos del sistema y almacenamiento local
        ProfileMetadataItem("Nombre de Usuario",        username)
        ProfileMetadataItem("Rol de Acceso",            "Estudiante / Evaluador")
        ProfileMetadataItem("Directorio Local Interno", context.filesDir.absolutePath)
        ProfileMetadataItem("Fabricante del Equipo",    Build.MANUFACTURER.uppercase())
        ProfileMetadataItem("Modelo del Dispositivo",   Build.MODEL)
        ProfileMetadataItem("Versión de Android",       "Android ${Build.VERSION.RELEASE} (API ${Build.VERSION.SDK_INT})")

        HorizontalDivider(modifier = Modifier.padding(vertical = 8.dp))

        // Card para el control del Modo Oscuro Persistente
        Card(
            modifier = Modifier.fillMaxWidth(),
            colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceContainerLow)
        ) {
            ListItem(
                headlineContent   = { Text("Modo oscuro", fontWeight = FontWeight.SemiBold) },
                supportingContent = { Text("Forzar aspecto visual nocturno") },
                trailingContent   = {
                    Switch(
                        checked         = isDark,
                        onCheckedChange = { sessionVm.setDarkMode(it) }
                    )
                },
                colors = ListItemDefaults.colors(containerColor = MaterialTheme.colorScheme.surfaceContainerLow)
            )
        }
    }
}

// ── SUB-PANTALLA 3: HISTORIAL DE ACTIVIDAD ──
@Composable
private fun MyActivityScreen(onBack: () -> Unit) {
    Column(modifier = Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            IconButton(onClick = onBack) { Icon(Icons.Default.ArrowBack, contentDescription = "Regresar") }
            Text("Historial de Actividad", style = MaterialTheme.typography.titleLarge, fontWeight = FontWeight.Bold)
        }

        Card(
            modifier = Modifier.fillMaxWidth(),
            colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceContainerHigh)
        ) {
            Box(modifier = Modifier.fillMaxWidth().padding(24.dp), contentAlignment = Alignment.Center) {
                Text(
                    text  = "No hay archivos multimedia registrados en este ciclo.",
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        }
    }
}

// ── COMPONENTE AUXILIAR: FILA DE METADATO ──
@Composable
private fun ProfileMetadataItem(label: String, value: String) {
    Column(modifier = Modifier.fillMaxWidth().padding(horizontal = 4.dp)) {
        Text(label, style = MaterialTheme.typography.labelSmall,
             color = MaterialTheme.colorScheme.primary, fontWeight = FontWeight.Bold)
        Text(value, style = MaterialTheme.typography.bodyMedium, fontWeight = FontWeight.Medium)
        Spacer(modifier = Modifier.height(4.dp))
    }
}
```

**¿Por qué `sealed class` en vez de rutas del `NavController`?**

Las sub-vistas de Perfil son estado interno de una sola pantalla, no destinos independientes de la app. Usar `sealed class` + `remember { mutableStateOf() }` elimina la necesidad de rutas adicionales en el `NavGraph`, simplifica el back stack global y mantiene el ViewModel en el mismo scope de composición.

**Cadena reactiva del Switch de modo oscuro:**

```
Switch.onCheckedChange(true)
  → SessionViewModel.setDarkMode(true)
  → SessionManager.edit { dark_mode = true }
  → DataStore emite Boolean
  → isDarkMode StateFlow emite true
  → MainActivity: usarModoOscuro = true
  → AppTheme(darkTheme = true)
  → toda la UI recompone con el esquema oscuro
```

**"Directorio Local Interno"** muestra la ruta `/data/user/0/com.illareklab.demodata/files` — permite ver con Device File Explorer exactamente dónde se almacenan los datos locales de la app.

---

## 16. `MainActivity` — Punto de entrada y DI manual

```kotlin
package com.illareklab.demodata

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.illareklab.demodata.ui.navigation.AppNavigation
import com.illareklab.demodata.ui.theme.AppTheme
import com.illareklab.demodata.ui.viewmodel.GpsViewModel
import com.illareklab.demodata.ui.viewmodel.SessionViewModel

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 1. Los objetos de infraestructura ya viven en DemoDataApp
        val app = application as DemoDataApp

        // 2. Instanciación de los ViewModels inyectando sus dependencias desde DemoDataApp
        val gpsViewModel     = GpsViewModel(app.gpsRepository)
        val sessionViewModel = SessionViewModel(app.sessionManager)

        setContent {
            // 3. Recolección reactiva de la preferencia del Modo Oscuro de DataStore
            val isDarkModePref by sessionViewModel.isDarkMode.collectAsStateWithLifecycle()
            val usarModoOscuro = isDarkModePref ?: isSystemInDarkTheme()

            // 4. Inyección del estado dinámico al tema de Material Design 3
            AppTheme(darkTheme = usarModoOscuro, dynamicColor = false) {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color    = MaterialTheme.colorScheme.background
                ) {
                    AppNavigation(
                        gpsViewModel     = gpsViewModel,
                        sessionViewModel = sessionViewModel
                    )
                }
            }
        }
    }
}
```

**Patrón de DI con `DemoDataApp`:**

```
DemoDataApp (proceso único, vive antes que cualquier Activity/Service)
  ├── database      by lazy  → AppDatabase singleton
  ├── gpsRepository by lazy  → GpsRepository(dao1, dao2)
  └── sessionManager by lazy → SessionManager(context)
            │
MainActivity.onCreate()
  ├── val app = application as DemoDataApp
  ├── GpsViewModel(app.gpsRepository)     → ViewModel GPS
  └── SessionViewModel(app.sessionManager) → ViewModel de sesión
            │
       AppNavigation(gpsVm, sessionVm)
            ├── GpsScreen(viewModel = gpsVm)
            └── ProfileScreen(sessionVm = sessionVm, onLogout = { sessionVm.logout() })

GpsCaptureService.onCreate()
  └── gpsRepo = (application as DemoDataApp).gpsRepository  ← misma instancia
```

No se usa `ViewModelProvider` ni `hiltViewModel()` — los ViewModels se pasan directamente como parámetros hasta las pantallas que los necesitan. `DemoDataApp` actúa como contenedor de dependencias manual sin agregar ningún framework externo.

---

## 17. Flujo de datos completo del laboratorio

### Ciclo GNSS

```
[GpsScreen] tap "Capturar coordenada"
  → capturando = true
  → context.startForegroundService(Intent(GpsCaptureService))
      │
  GpsCaptureService.onCreate()  → createNotificationChannel() → startForeground()
  GpsCaptureService.onStartCommand()
  → scope.launch { while(isActive) { performCaptures(); delay(10s) } }
      │
      ├── fusedClient.getCurrentLocation().await()    [suspende sin bloquear]
      │     → Room INSERT gps_google
      │
      └── withTimeoutOrNull(5s) { getRawGpsLocation() }
            └── suspendCancellableCoroutine → LocationManager.requestLocationUpdates
            → Room INSERT gps_sensors  (latitude = null si timeout expira)
      │
Room Flow emite → GpsRepository → GpsViewModel
  → combine(googlePoints, sensorsPoints) { ... }
  → flowOn(Dispatchers.Default)
  → StateFlow emite ComparativeGpsRecord
  → GpsScreen recompone → ComparativeCaptureCard por instante
```

### Ciclo tema

```
[MyProfileScreen] Switch ON
  → SessionViewModel.setDarkMode(true)
  → SessionManager.sessionDataStore.edit { dark_mode = true }
  → DataStore emite Preferences
  → isDarkMode Flow emite true
  → SessionViewModel.isDarkMode StateFlow emite true
  → MainActivity: usarModoOscuro = true
  → AppTheme(darkTheme = true) → toda la UI recompone con esquema oscuro

[ProfileMenu] botón "Cerrar sesión" → AlertDialog → "Sí, cerrar"
  → SessionViewModel.logout()
  → DataStore.clear() preservando KEY_DARK_MODE
  → isLoggedIn Flow emite false
```

---

## 18. Conceptos de las listas de temas cubiertos

| Tema | Lista | Cómo se ve en este lab |
|---|---|---|
| StateFlow y UDF | L1 | `comparativeHistory`, `isDarkMode`: flujo unidireccional desde DataStore/Room hasta la UI |
| ViewModel | L1 | `GpsViewModel` con `combine + flowOn`; `SessionViewModel` con `setDarkMode` |
| Navegación Compose | L1 | `NavHost` con dos tabs en `AppNavigation`; sub-vistas de Perfil con `sealed class` sin `NavController` |
| IU responsiva | L1 | `weight(1f)` en tarjetas duales; `fillMaxWidth`; `Arrangement.spacedBy` en columnas |
| Actividades y ciclos de vida | L1 | `GpsCaptureService.onDestroy()` cancela el loop; `onCleared()` implícito en ViewModel |
| Temas | L2 | Toggle Dark Mode → DataStore → `AppTheme` recompone sin reiniciar Activity |
| Menús | L2 | `TopAppBar` + `NavigationBar` + opciones tipo menú en `ProfileMenu` con `ListItem` |
| Layouts | L2 | `Column`, `Row`, `Card`, `LazyColumn` en ambas pantallas |
| Containers | L2 | `Card` con `primaryContainer`/`errorContainer`; `AlertDialog` de confirmación |
| Adaptadores | L2 | `LazyColumn + items()` con `key = { it.timestamp }` para animaciones eficientes |
| Corrutinas | L3 | Loop `while(isActive)`, `delay`, `withTimeoutOrNull`, `suspendCancellableCoroutine`, `.await()` |
| Capa de Datos | L3 | Room (2 entidades GPS, 2 DAOs, `AppDatabase` singleton), DataStore (3 claves de sesión) |
| Repositorio | L3 | `GpsRepository`: fachada sobre 2 DAOs, única fuente de verdad para la UI |
| DI manual | L3 | Instanciación explícita en `MainActivity.onCreate()`; ViewModels pasados por parámetro |

---

## 19. Ejercicios propuestos

1. **Cambiar el intervalo de captura** de 10 s a 5 s en `GpsCaptureService.INTERVAL_MS` y observar cómo `comparativeHistory` emite con mayor frecuencia.

2. **Mostrar el campo `satellites`** en `ComparativeCaptureCard`. El campo `satellites: Int?` ya existe en `GpsSensorsEntity`; solo falta mostrarlo en la UI como `"Satélites: ${record.sensors.satellites ?: "N/D"}"`.

3. **Mostrar la diferencia de precisión** entre FLP y sensor en `ComparativeCaptureCard`: calcular la distancia en metros entre ambas coordenadas con la fórmula de Haversine.

4. **Agregar un botón "Limpiar historial"** en `GpsScreen` que llame a `gpsRepository.clearAll()` y observe cómo `comparativeHistory` emite una lista vacía automáticamente.

5. **Poblar `MyActivityScreen`** con los registros reales de GPS: inyectar el `GpsViewModel` en `ProfileScreen` y pasar `googlePoints` y `sensorsPoints` a `MyActivityScreen` para mostrar un historial unificado con `LazyColumn`.

---

## 20. Checklist del laboratorio

- [ ] `GpsGoogleEntity` y `GpsSensorsEntity` creadas con `@Entity` en `data/local/entity/`
- [ ] `GpsSensorsEntity` con `latitude: Double?`, `longitude: Double?` nullable y campo `satellites: Int?`
- [ ] `AppDatabase` singleton con `@Database(entities = [...], version = 1)`
- [ ] `GpsGoogleDao` y `GpsSensorsDao` con `Flow<List<T>>` en las queries
- [ ] `GpsRepository` expone `googlePoints`, `sensorsPoints`, `googleCount`, `sensorsCount`
- [ ] `GpsViewModel` con `comparativeHistory` usando `combine + flowOn(Dispatchers.Default)`
- [ ] `GpsCaptureService` registrado en `AndroidManifest` con `foregroundServiceType="location"`
- [ ] `GpsCaptureService` implementa `onCreate()` con `startForeground()` y canal de notificación
- [ ] `GpsCaptureService` implementa loop de coroutine con `withTimeoutOrNull(5_000)` y `suspendCancellableCoroutine`
- [ ] `GpsScreen(viewModel: GpsViewModel)` solicita permisos con `rememberMultiplePermissionsState`
- [ ] `GpsScreen` muestra `ComparativeCaptureCard` en `LazyColumn` con `key = { it.timestamp }`
- [ ] `AppNavigation` con `sealed class Ruta`, `TopAppBar`, `NavigationBar` y dos composables en `NavHost`
- [ ] `SessionManager` con clave `KEY_DARK_MODE` y extensión `Context.sessionDataStore`
- [ ] `SessionManager.logout()` preserva `dark_mode` al hacer `clear()`
- [ ] `SessionViewModel` expone `isDarkMode: StateFlow<Boolean?>`
- [ ] `MainActivity` inicializa `AppDatabase`, `GpsRepository`, `SessionManager` y los dos ViewModels manualmente
- [ ] `MainActivity` aplica `isDarkModePref ?: isSystemInDarkTheme()` a `AppTheme`
- [ ] `ProfileScreen(sessionVm, onLogout)` con 3 sub-vistas (`sealed class ProfileViewState`)
- [ ] `MyProfileScreen` con `Switch` conectado a `SessionViewModel.setDarkMode()` y metadatos del dispositivo
- [ ] `MyActivityScreen` con estructura base (extensible con lista real de GPS)
- [ ] App compila y muestra coordenadas comparativas en tiempo real
- [ ] El toggle de modo noche persiste entre sesiones y sobrevive al logout
