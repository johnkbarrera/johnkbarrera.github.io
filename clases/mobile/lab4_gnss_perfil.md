# Lab 4 — GNSS y Perfil de Usuario

**Autor:** [illarek-lab](https://github.com/illarek-lab)
**Proyecto:** DemoData · `com.illareklab.demodata`
**Temas:** L1 (StateFlow, ViewModel, Navegación Compose) · L2 (Temas, Menús, Layouts, Containers, Adaptadores) · L3 (Corrutinas, Capa de Datos, Repositorio, DI manual)

> **Objetivo del laboratorio:** Comprender cómo se capturan, almacenan y visualizan coordenadas GNSS desde dos fuentes simultáneas (Google FLP y chip de hardware), y cómo se construye una pantalla de perfil con múltiples sub-vistas, toggle de modo noche persistente y un historial unificado de actividad.

---

## 1. Arquitectura del laboratorio

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
    └── MyActivityScreen  → lista unificada de todos los registros
```

---

## 2. Dependencias (`app/build.gradle.kts`)

Estas son las dependencias necesarias para el Lab 4. La mayoría ya están en el proyecto; se listan aquí para referencia.

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

> **Nota:** `room-compiler` va con `ksp(...)` en las dependencias, no con `implementation(...)`. Si se pone con `implementation` Room no genera el código y el proyecto no compila.

---

## 3. Componentes cubiertos en este lab


| Componente | Archivo |
|---|---|
| Entidad GPS Google | `data/local/entity/GpsGoogleEntity.kt` |
| Entidad GPS Sensores | `data/local/entity/GpsSensorsEntity.kt` |
| DAOs GPS | `data/local/dao/GpsGoogleDao.kt`, `GpsSensorsDao.kt` |
| Repositorio GPS | `data/repository/GpsRepository.kt` |
| ViewModel GPS | `ui/viewmodel/GpsViewModel.kt` |
| Servicio de captura | `services/GpsCaptureService.kt` |
| Pantalla GNSS | `ui/screens/GpsScreen.kt` |
| SessionManager | `data/session/SessionManager.kt` |
| SessionViewModel | `ui/viewmodel/SessionViewModel.kt` |
| Pantalla Perfil | `ui/screens/ProfileScreen.kt` |

---

## 4. Capa de datos — Entidades GNSS

### 4.1 `GpsGoogleEntity` — Fuente Google FLP

```kotlin
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
@Entity(tableName = "gps_sensors")
data class GpsSensorsEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0L,
    val latitude: Double?,      // ← NULLABLE: null cuando no hay fix satelital
    val longitude: Double?,     // ← NULLABLE
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

---

## 5. DAOs

### `GpsGoogleDao.kt`

```kotlin
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

## 6. Repositorio

```kotlin
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

## 7. `GpsViewModel` — Historial comparativo

```kotlin
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
                google = gList.find { it.timestamp == ts },
                sensors = sList.find { it.timestamp == ts }
            )
        }
    }
    .flowOn(Dispatchers.Default)   // cálculo pesado fuera del hilo principal
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
}
```

### Puntos clave para la clase

**`combine`:** toma N flows y emite un valor nuevo cada vez que cualquiera cambia. Aquí combina las dos listas de Room en una sola vista comparativa sin duplicar consultas.

**`.flowOn(Dispatchers.Default)`:** el operador `combine` y la construcción del mapa se ejecutan en el pool de hilos de CPU, no en el Main Thread. Sin esto, listas largas causarían el warning "Skipped N frames" en la UI.

**`SharingStarted.WhileSubscribed(5_000)`:** el flow se mantiene activo 5 segundos después de que el último suscriptor desaparece. Esto evita re-suscripciones costosas a Room durante rotaciones de pantalla (el proceso de rotación desmonta y remonía el Composable en menos de 5 s).

**Emparejamiento por `timestamp`:** el `GpsCaptureService` captura ambas fuentes en el mismo ciclo y asigna el mismo valor de `System.currentTimeMillis()` a ambos registros, lo que hace el emparejamiento exacto.

---

## 8. `GpsCaptureService` — Loop con coroutines

```kotlin
class GpsCaptureService : Service() {

    companion object {
        private const val INTERVAL_MS = 10_000L
        private const val SENSOR_TIMEOUT_MS = 5_000L
    }

    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    private var captureJob: Job? = null
    private val gpsRepo by lazy { (application as DemoDataApp).gpsRepository }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        if (!hasLocationPermission()) { stopSelf(); return START_NOT_STICKY }

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

        // 1. Google FLP — always returns a location
        try {
            val loc = fusedClient
                .getCurrentLocation(Priority.PRIORITY_HIGH_ACCURACY, null)
                .await()                         // suspende sin bloquear el hilo
            loc?.let {
                gpsRepo.saveGooglePoint(GpsGoogleEntity(
                    latitude = it.latitude, longitude = it.longitude,
                    accuracy = it.accuracy,
                    speed    = if (it.hasSpeed()) it.speed else null,
                    bearing  = if (it.hasBearing()) it.bearing else null,
                    timestamp = now
                ))
            }
        } catch (e: Exception) { /* permisos revocados en runtime */ }

        // 2. Sensor GNSS — puede no tener fix; timeout de 5 s
        try {
            val sensorLoc = withTimeoutOrNull(SENSOR_TIMEOUT_MS) { getRawGpsLocation() }
            gpsRepo.saveSensorsPoint(GpsSensorsEntity(
                latitude  = sensorLoc?.latitude,    // null si timeout
                longitude = sensorLoc?.longitude,
                provider  = LocationManager.GPS_PROVIDER,
                altitude  = if (sensorLoc?.hasAltitude() == true) sensorLoc.altitude else null,
                timestamp = now
            ))
        } catch (e: Exception) { }
    }

    override fun onDestroy() {
        super.onDestroy()
        captureJob?.cancel()     // cancela el loop; el scope se limpia solo
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

---

## 9. `GpsScreen` — Permisos y tarjetas duales

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun GpsScreen() {
    val permisos = buildList {
        add(Manifest.permission.ACCESS_FINE_LOCATION)
        add(Manifest.permission.ACCESS_COARSE_LOCATION)
        if (Build.VERSION.SDK_INT >= TIRAMISU) add(Manifest.permission.POST_NOTIFICATIONS)
    }
    val estadoPermisos = rememberMultiplePermissionsState(permissions = permisos)

    // Bloqueo temprano: si no hay permisos mostramos Card de error y retornamos
    if (!estadoPermisos.allPermissionsGranted) {
        Card(colors = CardDefaults.cardColors(containerColor = errorContainer)) {
            Button(onClick = { estadoPermisos.launchMultiplePermissionRequest() }) {
                Text("Conceder permisos")
            }
        }
        return@Column
    }

    // Botón con color reactivo al estado
    Button(
        colors = ButtonDefaults.buttonColors(
            containerColor = if (capturando) colorScheme.error else colorScheme.primary
        )
    ) {
        Icon(if (capturando) Stop else PlayArrow, null)
        Text(if (capturando) "Detener captura" else "Capturar coordenada (cada 10 s)")
    }

    // Contadores en vivo con Cards semánticas
    Row {
        Card(colors = CardDefaults.cardColors(containerColor = primaryContainer)) {
            Text("Google FLP"); Text("${googlePoints.size}"); Text("registros")
        }
        Card(colors = CardDefaults.cardColors(containerColor = secondaryContainer)) {
            Text("Sensores GNSS"); Text("${sensorsPoints.size}"); Text("registros")
        }
    }

    // Lista reactiva del historial comparativo
    LazyColumn {
        items(items = history, key = { it.timestamp }) { record ->
            ComparativeCaptureCard(record, dateFormat)
        }
    }
}
```

### `ComparativeCaptureCard` — Dual panel

Cada tarjeta representa un **instante de captura** y muestra los datos de ambas fuentes lado a lado:

```
┌──────────────────────────────────────────────┐
│  Instante: 14:32:07          ID: 7432        │
├──────────────────┬───────────────────────────┤
│  GOOGLE FLP      │  SENSOR GNSS              │
│  19.43212        │  19.43198                 │
│  -99.13345       │  -99.13341                │
│  Acc: ±8m        │  Alt: 2240m               │
├──────────────────┼───────────────────────────┤
│  GOOGLE FLP      │  SENSOR GNSS (sin fix)    │
│  19.43212        │  SIN SEÑAL                │
│  -99.13345       │  Indoors                  │
└──────────────────┴───────────────────────────┘
```

La columna de sensores usa `MaterialTheme.colorScheme.error` cuando `latitude == null`, dando feedback visual inmediato de la calidad de señal.

---

## 10. `SessionManager` — Tres claves en DataStore

```kotlin
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

## 11. `SessionViewModel` — Dark mode reactivo

```kotlin
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

## 12. `ProfileScreen` — Tres sub-vistas con `sealed class`

```kotlin
private sealed class ProfileViewState {
    object Menu      : ProfileViewState()
    object MyProfile : ProfileViewState()
    object MyActivity: ProfileViewState()
}

@Composable
fun ProfileScreen(onLogout: () -> Unit, username: String?) {
    var viewState by remember { mutableStateOf<ProfileViewState>(ProfileViewState.Menu) }

    when (viewState) {
        ProfileViewState.Menu       -> ProfileMenu(
            username = username,
            onLogout = onLogout,
            onNavigateToProfile  = { viewState = ProfileViewState.MyProfile },
            onNavigateToActivity = { viewState = ProfileViewState.MyActivity }
        )
        ProfileViewState.MyProfile  -> MyProfileScreen(onBack = { viewState = ProfileViewState.Menu })
        ProfileViewState.MyActivity -> MyActivityScreen(onBack = { viewState = ProfileViewState.Menu })
    }
}
```

**¿Por qué `sealed class` en vez de rutas del `NavController`?**

Las sub-vistas de Perfil son un estado interno de la pantalla, no destinos independientes de la app. Usar `sealed class` + `remember { mutableStateOf() }` mantiene la navegación interna sin agregar rutas al `NavGraph`, lo que simplifica el back stack global.

### 11.1 `ProfileMenu`

```
Avatar (Icon Person 72dp)
Username (headlineMedium)
├── Card "Mi Perfil"    → MyProfileScreen
├── Card "Mi Actividad" → MyActivityScreen
└── Button "Cerrar sesión" (error color) → AlertDialog de confirmación
```

El `AlertDialog` previene cierres accidentales:

```kotlin
AlertDialog(
    title = { Text("¿Confirmar cierre de sesión?") },
    text  = { Text("Tus datos locales se conservan.") },
    confirmButton = {
        TextButton(onClick = onConfirm) {
            Text("Sí, cerrar sesión", color = colorScheme.error)
        }
    },
    dismissButton = { TextButton(onClick = onDismiss) { Text("Cancelar") } }
)
```

### 11.2 `MyProfileScreen` — Metadatos y toggle modo noche

```kotlin
val isDarkModePref by sessionVm.isDarkMode.collectAsStateWithLifecycle()
val isDark = isDarkModePref ?: isSystemInDarkTheme()

// Metadatos del usuario y del dispositivo
ProfileMetadataItem("Username", username ?: "N/A")
ProfileMetadataItem("Rol", "Administrador / Operador")
ProfileMetadataItem("Directorio Local", LocalContext.current.filesDir.absolutePath)
ProfileMetadataItem("Dispositivo", "${Build.MANUFACTURER} ${Build.MODEL}")
ProfileMetadataItem("Android Version", Build.VERSION.RELEASE)
ProfileMetadataItem("API Level", Build.VERSION.SDK_INT.toString())

// Toggle modo noche
Switch(
    checked = isDark,
    onCheckedChange = { sessionVm.setDarkMode(it) }
)
```

**"Directorio Local"** muestra la ruta absoluta de `filesDir` en el dispositivo real, por ejemplo `/data/user/0/com.illareklab.demodata/files`. Útil pedagógicamente para que los alumnos vean con Device File Explorer exactamente dónde se guardan las fotos, videos y audios capturados.

Un solo `onCheckedChange` desencadena toda la cadena reactiva:

```
Switch.onCheckedChange(true)
  → SessionViewModel.setDarkMode(true)
  → SessionManager.edit { dark_mode = true }
  → DataStore emite Boolean
  → isDarkMode StateFlow emite true
  → MainActivity.darkTheme = true
  → AppTheme(darkTheme = true)
  → toda la UI recompone con el esquema oscuro
```

### 11.3 `MyActivityScreen` — Lista unificada

```kotlin
val googlePoints by gpsRepo.googlePoints.collectAsStateWithLifecycle(emptyList())
val sensorsPoints by gpsRepo.sensorsPoints.collectAsStateWithLifecycle(emptyList())
val allMedia      by mediaRepo.allMedia.collectAsStateWithLifecycle(emptyList())
val allAudios     by audioRepo.allAudios.collectAsStateWithLifecycle(emptyList())

val combinedItems = remember(googlePoints, sensorsPoints, allMedia, allAudios) {
    mutableListOf<ActivityItem>().apply {
        addAll(googlePoints.map  { ActivityItem.GpsGoogle(it) })
        addAll(sensorsPoints.map { ActivityItem.GpsSensors(it) })
        addAll(allMedia.map      { ActivityItem.Media(it) })
        addAll(allAudios.map     { ActivityItem.Audio(it) })
    }.sortedByDescending { it.timestamp }
}
```

`remember(key1, key2, ...)` recalcula la lista combinada únicamente cuando alguno de sus inputs cambia. Sin `remember`, la lista se reconstruiría en cada recomposición.

### `ActivityDetailDialog` — Apertura con FileProvider

```kotlin
val uri = FileProvider.getUriForFile(
    context,
    "${context.packageName}.fileprovider",
    File(path)
)
val intent = Intent(Intent.ACTION_VIEW).apply {
    setDataAndType(uri, mimeType)      // "image/*", "video/*" o "audio/*"
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
context.startActivity(intent)
```

Sin el `FileProvider`, Android 7+ lanza `FileUriExposedException` al pasar una ruta `file://` a otra app. El `FileProvider` convierte la ruta privada de `filesDir` en un `content://` URI con permiso de lectura temporal.

---

## 13. Flujo de datos completo del laboratorio

### Ciclo GNSS

```
[GpsScreen] tap "Capturar"
  → startForegroundService(GpsCaptureService)
  → scope.launch { while(isActive) { performCaptures(); delay(10s) } }
      │
      ├── fusedClient.getCurrentLocation().await()   [suspende, no bloquea]
      │     → Room tabla gps_google (INSERT)
      │
      └── withTimeoutOrNull(5s) { getRawGpsLocation() }
            → Room tabla gps_sensors (INSERT, latitude puede ser null)
      │
Room emite Flow → GpsRepository → GpsViewModel
  → combine(googlePoints, sensorsPoints) → comparativeHistory
  → flowOn(Default) → StateFlow
  → GpsScreen recompone → ComparativeCaptureCard por instante
```

### Ciclo tema

```
[ProfileScreen] Switch ON
  → SessionViewModel.setDarkMode(true)
  → DataStore emite true
  → SessionViewModel.isDarkMode = true
  → MainActivity.darkTheme = true
  → AppTheme(darkTheme = true) → toda la UI en oscuro

[ProfileScreen] logout → AlertDialog → confirmar
  → SessionViewModel.logout()
  → DataStore.clear() preservando dark_mode
  → isLoggedIn = false
  → Navigation → LoginScreen (con tema oscuro conservado)
```

---

## 14. Conceptos de las listas de temas cubiertos

| Tema | Lista | Cómo se ve en este lab |
|---|---|---|
| StateFlow y UDF | L1 | `comparativeHistory`, `isDarkMode`: flujo unidireccional de datos desde DataStore hasta la UI |
| ViewModel | L1 | `GpsViewModel` con `combine + flowOn`; `SessionViewModel` con `setDarkMode` |
| Navegación Compose | L1 | Tab GNSS en `NavHost`; sub-vistas de Perfil con `sealed class` sin `NavController` |
| IU responsiva | L1 | `weight(1f)` en tarjetas duales; `fillMaxWidth`; `verticalScroll` en Perfil |
| Actividades y ciclos de vida | L1 | `GpsCaptureService.onDestroy()` cancela el loop; `onCleared()` implícito en ViewModel |
| Temas | L2 | Toggle Dark Mode → DataStore → `AppTheme` recompone sin reiniciar Activity |
| Menús | L2 | `TopAppBar` + `BottomNavBar` + opciones tipo menú en `ProfileMenu` |
| Layouts | L2 | `Column`, `Row`, `Card`, `LazyColumn` en ambas pantallas |
| Containers | L2 | `Card` con `primaryContainer`/`errorContainer`; `AlertDialog` de confirmación |
| Adaptadores | L2 | `LazyColumn + items()` con `key = { it.timestamp }` para animaciones eficientes |
| Corrutinas | L3 | Loop `while(isActive)`, `delay`, `withTimeoutOrNull`, `suspendCancellableCoroutine`, `await()` |
| Capa de Datos | L3 | Room (2 entidades GPS, 2 DAOs), DataStore (3 claves de sesión) |
| Repositorio | L3 | `GpsRepository`: fachada sobre 2 DAOs, única fuente de verdad para la UI |
| DI manual | L3 | `GpsViewModel.Factory(app.gpsRepository)`; `DemoDataApp by lazy` |

---

## 15. Ejercicios propuestos

1. **Cambiar el intervalo de captura** de 10 s a 5 s en `GpsCaptureService.INTERVAL_MS` y observar cómo `comparativeHistory` emite con mayor frecuencia.

2. **Agregar un campo `satellites`** a `GpsSensorsEntity`. Esto requiere: actualizar la entidad, incrementar la versión de la BD a 3, y agregar `Migration(2,3)` con `ALTER TABLE gps_sensors ADD COLUMN satellites INTEGER`.

3. **Mostrar la diferencia de precisión** entre FLP y sensor en `ComparativeCaptureCard`: calcular la distancia en metros entre ambas coordenadas con la fórmula de Haversine.

4. **Agregar un botón "Limpiar historial"** en `GpsScreen` que llame a `gpsRepository.clearAll()` y observe cómo `comparativeHistory` emite una lista vacía automáticamente.

---

## 16. Checklist del laboratorio

- [ ] `GpsGoogleEntity` y `GpsSensorsEntity` creadas con `@Entity`
- [ ] `GpsSensorsEntity` con `latitude: Double?` y `longitude: Double?` nullable
- [ ] `GpsGoogleDao` y `GpsSensorsDao` con `Flow<List<T>>` en las queries
- [ ] `GpsRepository` expone `googlePoints`, `sensorsPoints`, `googleCount`, `sensorsCount`
- [ ] `GpsViewModel` con `comparativeHistory` usando `combine + flowOn(Default)`
- [ ] `GpsCaptureService` registrado en `AndroidManifest` con `foregroundServiceType="location"`
- [ ] `GpsCaptureService` implementa loop de coroutine con `withTimeoutOrNull(5_000)`
- [ ] `GpsScreen` solicita permisos con `rememberMultiplePermissionsState`
- [ ] `GpsScreen` muestra `ComparativeCaptureCard` en `LazyColumn`
- [ ] `SessionManager` con clave `KEY_DARK_MODE`
- [ ] `SessionManager.logout()` preserva `dark_mode` al hacer `clear()`
- [ ] `SessionViewModel` expone `isDarkMode: StateFlow<Boolean?>`
- [ ] `MainActivity` aplica `isDarkMode` a `AppTheme`
- [ ] `ProfileScreen` con 3 sub-vistas (`sealed class ProfileViewState`)
- [ ] `MyProfileScreen` con `Switch` conectado a `SessionViewModel.setDarkMode()`
- [ ] `MyActivityScreen` con lista combinada ordenada por timestamp
- [ ] `ActivityDetailDialog` usa `FileProvider` para abrir archivos externos
- [ ] App compila y muestra coordenadas comparativas en tiempo real
- [ ] El toggle de modo noche persiste entre sesiones y sobrevive al logout
