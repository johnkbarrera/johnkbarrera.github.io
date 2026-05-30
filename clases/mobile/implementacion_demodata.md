# Implementación DemoData — Guía técnica detallada

**Autor:** [illarek-lab](https://github.com/illarek-lab)

### App Android Kotlin Offline-First · Captura GNSS + Multimedia + Audio

> **Objetivo:** Documentar la arquitectura real del proyecto `DemoData` tal como está implementado en el repositorio. Esta guía es el complemento técnico de `arquitectura_fleet_offline.md`: mientras aquella describe el diseño conceptual, esta describe **lo que realmente existe en el código** con sus decisiones específicas.

> **Stack:** Kotlin + Jetpack Compose + Room v2 + DataStore + WorkManager + CameraX 1.6 + MediaRecorder + FusedLocationProvider + LocationManager + ViewModel + Coroutines/Flow · `minSdk = 29` · `compileSdk = 36`.

---

## 1. Diagrama de capas (implementación real)

```
┌──────────────────────────────────────────────────────────────┐
│                   UI (Compose Screens)                        │
│  LoginScreen · GpsScreen · MediaScreen · AudioScreen ·        │
│  SyncScreen · NotificationsScreen · ProfileScreen             │
│  (ProfileScreen tiene 3 sub-vistas: Menu / MyProfile /        │
│   MyActivity)                                                 │
└─────────────────────┬────────────────────────────────────────┘
                      │ observa StateFlow
                      ▼
┌──────────────────────────────────────────────────────────────┐
│                ViewModels (1 por pantalla)                    │
│  SessionViewModel · GpsViewModel · MediaViewModel ·           │
│  AudioViewModel · SyncViewModel                               │
└──────┬─────────────────────┬──────────────────────┬──────────┘
       │                     │                      │
       ▼                     ▼                      ▼
┌────────────┐    ┌─────────────────┐   ┌───────────────────┐
│ Room DAOs  │    │ FileStorage     │   │ SessionManager    │
│ fleet.db   │    │ (filesDir)      │   │ (DataStore Prefs) │
│ (version 2)│    │ photos/ videos/ │   │ is_logged_in      │
│            │    │ audios/         │   │ username          │
│            │    │                 │   │ dark_mode         │
└────────────┘    └─────────────────┘   └───────────────────┘
                          │
                  FileProvider registrado
                  para abrir archivos con
                  apps externas (galería,
                  reproductor, etc.)
```

**Diferencia clave respecto al diseño original:** la pestaña "Logout" fue reemplazada por **ProfileScreen**, que incorpora el logout dentro de un flujo de perfil completo con sub-pantallas.

---

## 2. Estructura real del proyecto

```
com.illareklab.demodata/
├── DemoDataApp.kt                ← Application class (DI manual con by lazy)
├── MainActivity.kt               ← Lee isDarkMode de DataStore y aplica al AppTheme
│
├── data/
│   ├── local/
│   │   ├── DemoDataDatabase.kt   ← version = 2, fallbackToDestructiveMigration()
│   │   ├── entity/
│   │   │   ├── GpsGoogleEntity.kt   (lat/lon/accuracy nullable)
│   │   │   ├── GpsSensorsEntity.kt  (lat/lon nullable — representa "sin señal")
│   │   │   ├── MediaEntity.kt
│   │   │   └── AudioEntity.kt
│   │   ├── dao/
│   │   │   ├── GpsGoogleDao.kt
│   │   │   ├── GpsSensorsDao.kt
│   │   │   ├── MediaDao.kt
│   │   │   └── AudioDao.kt
│   │   └── FileStorageManager.kt
│   │
│   ├── session/
│   │   └── SessionManager.kt     ← DataStore: is_logged_in + username + dark_mode
│   │
│   └── repository/
│       ├── GpsRepository.kt
│       ├── MediaRepository.kt
│       └── AudioRepository.kt
│
├── security/
│   └── PasswordHasher.kt         ← PBKDF2WithHmacSHA256, 120k iter (capa preparada)
│
├── services/
│   └── GpsCaptureService.kt      ← Foreground Service con coroutine loop (no callbacks)
│
├── workers/
│   └── DelayedNotificationWorker.kt
│
├── util/
│   └── AudioRecorderManager.kt   ← Wrapper de MediaRecorder (utility class)
│
└── ui/
    ├── Navigation.kt              ← Switch reactivo isLoggedIn → Login vs Scaffold
    ├── theme/
    │   ├── Theme.kt               ← Material 3 dinámico + 4 esquemas de contraste
    │   ├── Color.kt
    │   └── Type.kt
    ├── viewmodel/
    │   ├── SessionViewModel.kt    ← isLoggedIn + username + isDarkMode + setDarkMode()
    │   ├── GpsViewModel.kt        ← googlePoints + sensorsPoints + comparativeHistory
    │   ├── MediaViewModel.kt
    │   ├── AudioViewModel.kt      ← isRecording + elapsedSeconds + timer coroutine
    │   └── SyncViewModel.kt
    └── screens/
        ├── LoginScreen.kt         ← Con CircularProgressIndicator + estado "verificando"
        ├── GpsScreen.kt           ← Manejo de permisos + tarjetas comparativas dual-panel
        ├── MediaScreen.kt
        ├── AudioScreen.kt
        ├── SyncScreen.kt
        ├── NotificationsScreen.kt
        └── ProfileScreen.kt       ← 3 sub-vistas + toggle modo noche + Mi Actividad
```

---

## 3. Dependencias (`app/build.gradle.kts`)

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.ksp)
}

android {
    namespace = "com.illareklab.demodata"
    compileSdk { version = release(36) { minorApiLevel = 1 } }

    defaultConfig {
        applicationId = "com.illareklab.demodata"
        minSdk = 29
        targetSdk = 36
        versionCode = 1
        versionName = "1.0"
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }
    buildFeatures { compose = true }
}

ksp {
    arg("room.generateKotlin", "true")  // Room genera código Kotlin puro (no Java)
    arg("useK2", "true")                // Compilador Kotlin 2.0
}

dependencies {
    // ── Compose BOM + material ──
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.material3)
    implementation("androidx.compose.material:material-icons-extended:1.7.8")
    implementation("androidx.navigation:navigation-compose:2.8.0")
    implementation("androidx.compose.ui:ui-text-google-fonts:1.7.8")

    // ── Room (SQLite) ──
    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)
    ksp(libs.androidx.room.compiler)

    // ── DataStore ──
    implementation("androidx.datastore:datastore-preferences:1.1.1")

    // ── ViewModel + lifecycle ──
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.6")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.6")

    // ── Google FLP + coroutines bridge ──
    implementation("com.google.android.gms:play-services-location:21.3.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.11.0")

    // ── WorkManager ──
    implementation("androidx.work:work-runtime-ktx:2.9.0")

    // ── CameraX ──
    val cameraxVersion = "1.6.0"
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")
    implementation("androidx.camera:camera-video:$cameraxVersion")

    // ── Coil (thumbnails) ──
    implementation("io.coil-kt:coil-compose:2.7.0")

    // ── Permisos en Compose ──
    implementation("com.google.accompanist:accompanist-permissions:0.34.0")
}
```

### Notas de configuración KSP

`room.generateKotlin=true` hace que Room emita código Kotlin puro en lugar de Java, lo que acelera la compilación y evita la necesidad de configurar el interoperador Java/Kotlin. `useK2=true` activa el compilador Kotlin 2.0 para el proceso de KSP, requisito de Room 2.7+.

---

## 4. Entidades Room

### 4.1 `GpsGoogleEntity` — Ubicación del FLP de Google

```kotlin
@Entity(tableName = "gps_google")
data class GpsGoogleEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0L,
    val latitude: Double?,          // null si FLP no logra fijar posición
    val longitude: Double?,         // null si FLP no logra fijar posición
    val accuracy: Float?,           // precisión horizontal en metros; null si no disponible
    val speed: Float? = null,       // m/s — null si el dispositivo no lo reporta
    val bearing: Float? = null,     // grados desde el norte
    val timestamp: Long             // System.currentTimeMillis() en UTC
)
```

### 4.2 `GpsSensorsEntity` — Sensor GNSS crudo (con soporte "sin señal")

```kotlin
@Entity(tableName = "gps_sensors")
data class GpsSensorsEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0L,
    val latitude: Double?,          // ← NULLABLE: null cuando el chip no tiene fix
    val longitude: Double?,         // ← NULLABLE
    val provider: String,           // "gps", "network" o "passive"
    val altitude: Double? = null,
    val satellites: Int? = null,
    val timestamp: Long
)
```

**Diferencia crítica con el diseño original:** tanto `GpsGoogleEntity` como `GpsSensorsEntity` tienen `latitude` y `longitude` **nullable**. En `GpsSensorsEntity` es esperable (el chip puede no tener fix). En `GpsGoogleEntity`, `accuracy` también es `Float?` porque en condiciones extremas el FLP puede retornar una ubicación sin dato de precisión.

### 4.3 `MediaEntity` y `AudioEntity`

Idénticas al diseño original. `MediaType` es un enum `{ PHOTO, VIDEO }` cuyo `.name` se persiste como `String` en Room.

---

## 5. Base de datos

```kotlin
@Database(
    entities = [GpsGoogleEntity::class, GpsSensorsEntity::class,
                MediaEntity::class, AudioEntity::class],
    version = 2,
    exportSchema = false
)
abstract class DemoDataDatabase : RoomDatabase() {

    abstract fun gpsGoogleDao(): GpsGoogleDao
    abstract fun gpsSensorsDao(): GpsSensorsDao
    abstract fun mediaDao(): MediaDao
    abstract fun audioDao(): AudioDao

    companion object {
        @Volatile private var INSTANCE: DemoDataDatabase? = null

        fun getInstance(context: Context): DemoDataDatabase =
            INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(
                    context.applicationContext,
                    DemoDataDatabase::class.java,
                    "fleet.db"
                ).fallbackToDestructiveMigration().build()
                 .also { INSTANCE = it }
            }
    }
}
```

### ¿Por qué version = 2 con `fallbackToDestructiveMigration()`?

`GpsSensorsEntity` fue modificada para hacer `latitude` y `longitude` nullables, lo que cambió el schema de SQLite (las columnas pasaron de `NOT NULL` a permitir `NULL`). Eso requirió incrementar la versión a 2.

`fallbackToDestructiveMigration()` descarta la BD si la versión instalada no coincide y la recrea limpia. Es la estrategia correcta para un entorno de laboratorio donde los datos de prueba son desechables. En producción se usaría `Migration(1, 2)` con `ALTER TABLE`.

---

## 6. Sesión: DataStore con preferencia de tema

### 6.1 `SessionManager.kt`

```kotlin
private val Context.sessionDataStore: DataStore<Preferences>
    by preferencesDataStore(name = "fleet_session")

class SessionManager(private val context: Context) {

    private companion object {
        val KEY_IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
        val KEY_USERNAME     = stringPreferencesKey("username")
        val KEY_DARK_MODE    = booleanPreferencesKey("dark_mode")
    }

    val isLoggedIn: Flow<Boolean> = context.sessionDataStore.data
        .map { it[KEY_IS_LOGGED_IN] ?: false }

    val currentUsername: Flow<String?> = context.sessionDataStore.data
        .map { it[KEY_USERNAME] }

    // null = seguir al sistema; true/false = forzar modo
    val isDarkMode: Flow<Boolean?> = context.sessionDataStore.data
        .map { it[KEY_DARK_MODE] }

    suspend fun login(username: String) {
        context.sessionDataStore.edit { prefs ->
            prefs[KEY_IS_LOGGED_IN] = true
            prefs[KEY_USERNAME] = username
        }
    }

    suspend fun setDarkMode(enabled: Boolean) {
        context.sessionDataStore.edit { it[KEY_DARK_MODE] = enabled }
    }

    suspend fun logout() {
        context.sessionDataStore.edit { prefs ->
            // Preservamos la preferencia de tema al cerrar sesión
            val currentTheme = prefs[KEY_DARK_MODE]
            prefs.clear()
            if (currentTheme != null) prefs[KEY_DARK_MODE] = currentTheme
        }
    }
}
```

**Detalle de diseño:** `logout()` limpia la sesión pero conserva `KEY_DARK_MODE`. Esto hace que el usuario mantenga su preferencia de tema aunque cierre sesión: no tiene sentido que el tema se resetee al hacer logout.

---

## 7. Seguridad: `PasswordHasher`

```kotlin
object PasswordHasher {
    private const val ALGORITMO = "PBKDF2WithHmacSHA256"
    private const val ITERACIONES = 120_000   // OWASP 2023 para SHA-256
    private const val LONGITUD_HASH_BITS = 256

    fun hash(password: String, salt: ByteArray): String {
        val spec = PBEKeySpec(password.toCharArray(), salt, ITERACIONES, LONGITUD_HASH_BITS)
        val factory = SecretKeyFactory.getInstance(ALGORITMO)
        val bytes = factory.generateSecret(spec).encoded
        spec.clearPassword()                  // limpia el array char[] en memoria
        return bytes.joinToString("") { "%02x".format(it) }
    }

    fun constantTimeEquals(a: String, b: String): Boolean {
        if (a.length != b.length) return false
        var diff = 0
        for (i in a.indices) diff = diff or (a[i].code xor b[i].code)
        return diff == 0
    }
}
```

**Estado actual:** el hasher está implementado pero el `SessionViewModel` usa credenciales hardcodeadas (`jkn/jkn`) para el laboratorio. La clase está lista para integrarse cuando se añada un backend o un almacén de credenciales cifrado (`EncryptedSharedPreferences`).

`constantTimeEquals` evita _timing attacks_: el tiempo de la comparación no revela en qué posición difieren los hashes.

---

## 8. Application class (DI manual)

```kotlin
class DemoDataApp : Application() {

    val database by lazy { DemoDataDatabase.getInstance(this) }
    val fileStorage by lazy { FileStorageManager(this) }
    val sessionManager by lazy { SessionManager(this) }

    val gpsRepository by lazy {
        GpsRepository(database.gpsGoogleDao(), database.gpsSensorsDao())
    }
    val mediaRepository by lazy {
        MediaRepository(database.mediaDao(), fileStorage)
    }
    val audioRepository by lazy {
        AudioRepository(database.audioDao(), fileStorage)
    }
}
```

Registrada en el manifest como `android:name=".DemoDataApp"`. El `by lazy` garantiza que cada dependencia se crea una sola vez y solo cuando se necesita.

---

## 9. MainActivity — Dark mode desde DataStore

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val app = applicationContext as DemoDataApp
            val sessionVm: SessionViewModel = viewModel(
                factory = SessionViewModel.Factory(app.sessionManager)
            )

            val isDarkModePref by sessionVm.isDarkMode.collectAsState()
            // null = seguir al sistema; Boolean = forzar
            val darkTheme = isDarkModePref ?: isSystemInDarkTheme()

            AppTheme(darkTheme = darkTheme) {
                Navigation()
            }
        }
    }
}
```

`AppTheme` recibe el parámetro `darkTheme` en cada recomposición. Cuando el usuario cambia el toggle en `ProfileScreen → Mi Perfil`, DataStore emite, el Flow actualiza, el State cambia, y toda la app recompone con el nuevo esquema de colores instantáneamente — sin reiniciar la Activity.

---

## 10. Tema Material 3 completo

`Theme.kt` define **6 esquemas de color**:

| Esquema | Descripción |
|---|---|
| `lightScheme` | Tema claro estándar |
| `darkScheme` | Tema oscuro estándar |
| `mediumContrastLightColorScheme` | Claro con contraste medio (WCAG AA) |
| `highContrastLightColorScheme` | Claro con contraste alto (WCAG AAA) |
| `mediumContrastDarkColorScheme` | Oscuro con contraste medio |
| `highContrastDarkColorScheme` | Oscuro con contraste alto |

`AppTheme` activa **Dynamic Color** en Android 12+ (API 31): el sistema extrae la paleta del fondo de pantalla del usuario y la aplica automáticamente. Si el dispositivo no soporta Dynamic Color, usa los esquemas estáticos.

```kotlin
@Composable
fun AppTheme(darkTheme: Boolean = isSystemInDarkTheme(), dynamicColor: Boolean = true, ...) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> darkScheme
        else -> lightScheme
    }
    MaterialTheme(colorScheme = colorScheme, typography = AppTypography, content = content)
}
```

---

## 11. Navegación (6 pestañas)

```
LoginScreen
    │ isLoggedIn = true
    ▼
MainScaffold
  ├── Tab 0: GpsScreen          (GNSS)
  ├── Tab 1: MediaScreen        (Multimedia)
  ├── Tab 2: AudioScreen        (Audio)
  ├── Tab 3: SyncScreen         (Sync)
  ├── Tab 4: NotificationsScreen (Notif)
  └── Tab 5: ProfileScreen      (Perfil) ← reemplaza la pestaña Logout
```

La pestaña **Perfil** no lleva al logout directamente; en cambio, navega a una sub-pantalla con:
- **Mi Perfil:** metadatos del usuario + toggle de modo noche
- **Mi Actividad:** historial unificado de todos los registros (GNSS, foto, video, audio)
- **Botón "Cerrar sesión"** con `AlertDialog` de confirmación

---

## 12. `SessionViewModel`

```kotlin
class SessionViewModel(private val sessionManager: SessionManager) : ViewModel() {

    val isLoggedIn = sessionManager.isLoggedIn.stateIn(
        viewModelScope, SharingStarted.Eagerly, initialValue = false
    )
    val username = sessionManager.currentUsername.stateIn(
        viewModelScope, SharingStarted.Eagerly, initialValue = null
    )
    val isDarkMode = sessionManager.isDarkMode.stateIn(
        viewModelScope, SharingStarted.Eagerly, initialValue = null
    )

    fun login(username: String, password: String, onResult: (Boolean) -> Unit) {
        if (username == "jkn" && password == "jkn") {
            viewModelScope.launch {
                sessionManager.login(username)
                onResult(true)
            }
        } else {
            onResult(false)
        }
    }

    fun setDarkMode(enabled: Boolean) {
        viewModelScope.launch { sessionManager.setDarkMode(enabled) }
    }

    fun logout() {
        viewModelScope.launch { sessionManager.logout() }
    }
}
```

`SharingStarted.Eagerly` en `isLoggedIn`: el flow se inicia inmediatamente al crear el ViewModel, antes de que haya suscriptores. Esto garantiza que cuando `Navigation` se compone por primera vez ya tiene el valor correcto de DataStore, evitando el parpadeo de la pantalla de login antes de que cargue la sesión.

---

## 13. `LoginScreen`

```kotlin
@Composable
fun LoginScreen(onSubmit: (String, String, (Boolean) -> Unit) -> Unit) {
    var usuario by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var error by remember { mutableStateOf("") }
    var verificando by remember { mutableStateOf(false) }

    Column(/* ... */) {
        Text("DemoData", style = MaterialTheme.typography.displayMedium)
        Text("Sistema de gestión de datos", style = MaterialTheme.typography.bodyMedium)
        // ... TextFields ...
        Button(
            onClick = {
                verificando = true
                onSubmit(usuario, password) { ok ->
                    verificando = false
                    if (!ok) error = "Credenciales incorrectas. Pruebe jkn/jkn."
                }
            },
            enabled = !verificando && usuario.isNotBlank() && password.isNotBlank()
        ) {
            if (verificando) CircularProgressIndicator(/* ... */)
            else Text("Ingresar")
        }
        Text("Credenciales por defecto: jkn / jkn", style = MaterialTheme.typography.bodySmall)
    }
}
```

**Mejoras sobre el esqueleto original:**
- `verificando` deshabilita los campos y el botón durante la validación.
- El botón muestra un `CircularProgressIndicator` en lugar del texto.
- El botón está deshabilitado si los campos están vacíos (previene envíos en blanco).

---

## 14. `GpsViewModel` — Historial comparativo

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
    .flowOn(Dispatchers.Default)   // ← el cálculo pesado va fuera del hilo principal
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
}
```

`comparativeHistory` empareja registros de ambas tablas por `timestamp`. Dado que el servicio captura ambas fuentes en el mismo ciclo (el campo `timestamp` se asigna una sola vez al inicio de `performCaptures()`), el emparejamiento es exacto.

`flowOn(Dispatchers.Default)` mueve el cálculo del mapa y los `find` fuera del hilo principal, evitando el warning "Skipped N frames" en la UI.

---

## 15. `GpsCaptureService` — Loop con coroutines

```kotlin
class GpsCaptureService : Service() {

    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    private var captureJob: Job? = null

    override fun onStartCommand(...): Int {
        if (captureJob == null) {
            captureJob = scope.launch {
                while (isActive) {
                    performCaptures()
                    delay(INTERVAL_MS)  // 10 s
                }
            }
        }
        return START_STICKY
    }

    private suspend fun performCaptures() {
        val now = System.currentTimeMillis()

        // 1. Google FLP: await() gracias a kotlinx-coroutines-play-services
        try {
            val flpLoc = fusedClient
                .getCurrentLocation(Priority.PRIORITY_HIGH_ACCURACY, null)
                .await()
            flpLoc?.let { gpsRepo.saveGooglePoint(GpsGoogleEntity(...)) }
        } catch (e: Exception) {}

        // 2. Sensor GNSS: timeout de 5 s; null si no hay fix
        try {
            val sensorLoc = withTimeoutOrNull(SENSOR_TIMEOUT_MS) { getRawGpsLocation() }
            gpsRepo.saveSensorsPoint(GpsSensorsEntity(
                latitude = sensorLoc?.latitude,   // null si timeout
                longitude = sensorLoc?.longitude,
                provider = LocationManager.GPS_PROVIDER,
                altitude = if (sensorLoc?.hasAltitude() == true) sensorLoc.altitude else null,
                timestamp = now
            ))
        } catch (e: Exception) {}
    }

    private suspend fun getRawGpsLocation(): Location? = suspendCancellableCoroutine { cont ->
        // API 30+: locationManager.getCurrentLocation(GPS_PROVIDER, ...)
        // API 29 fallback: requestLocationUpdates con listener efímero
    }

    override fun onDestroy() {
        super.onDestroy()
        captureJob?.cancel()
    }
}
```

**Diferencias clave con el diseño original:**

| Diseño original | Implementación real |
|---|---|
| `LocationCallback` + `LocationListener` registrados permanentemente | Loop de coroutine con `delay(10 s)` y llamadas únicas por ciclo |
| FLP: streaming continuo | FLP: `getCurrentLocation()` (one-shot, sin callback permanente) |
| Sensor: listener permanente | Sensor: `suspendCancellableCoroutine` con `withTimeoutOrNull(5 s)` |
| No registra "sin señal" | Persiste `latitude = null` cuando el chip no tiene fix |

El loop de coroutine es más predecible que callbacks permanentes: cada ciclo es atómico, no hay overlap entre capturas, y la lógica de "captura → espera → captura" es lineal y legible.

---

## 16. `AudioViewModel` — Estado de grabación y timer

```kotlin
class AudioViewModel(
    private val context: Context,
    private val audioRepository: AudioRepository,
    private val fileStorage: FileStorageManager
) : ViewModel() {

    val audios = audioRepository.allAudios.stateIn(...)
    val count  = audioRepository.count.stateIn(...)

    private val _isRecording = MutableStateFlow(false)
    val isRecording = _isRecording.asStateFlow()

    private val _elapsedSeconds = MutableStateFlow(0)
    val elapsedSeconds = _elapsedSeconds.asStateFlow()

    private var recorder: MediaRecorder? = null
    private var currentFile: File? = null
    private var startTimeMs: Long = 0L
    private var timerJob: Job? = null

    fun startRecording(): Boolean {
        val file = fileStorage.newAudioFile(extension = "m4a")
        val newRecorder = if (Build.VERSION.SDK_INT >= S) MediaRecorder(context)
                          else @Suppress("DEPRECATION") MediaRecorder()
        newRecorder.apply {
            setAudioSource(MediaRecorder.AudioSource.MIC)
            setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
            setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
            setOutputFile(file.absolutePath)
            prepare(); start()
        }
        recorder = newRecorder; currentFile = file
        startTimeMs = System.currentTimeMillis()
        _isRecording.value = true

        timerJob = viewModelScope.launch {
            while (_isRecording.value) { delay(1000); _elapsedSeconds.value++ }
        }
        return true
    }

    fun stopRecording() {
        val durationMs = System.currentTimeMillis() - startTimeMs
        recorder?.apply { stop(); release() }

        // Solo registrar si >= 1 segundo (el codec AAC necesita mínimo de datos)
        if (currentFile != null && currentFile!!.exists() && durationMs >= 1000L) {
            viewModelScope.launch {
                audioRepository.registerAudio(currentFile!!.absolutePath, durationMs, "AAC")
            }
        } else {
            currentFile?.takeIf { it.exists() }?.delete()
        }
        cleanup()
    }

    override fun onCleared() {
        // Si el usuario abandona la app mientras graba, libera el micrófono
        if (_isRecording.value) { recorder?.apply { stop(); release() }; currentFile?.delete() }
    }
}
```

**Validación ≥ 1 segundo:** el codec AAC puede lanzar `IllegalStateException` si se llama a `stop()` sin haber grabado suficientes frames. La validación elimina el archivo corrupto en lugar de registrar basura en Room.

---

## 17. `GpsScreen` — Permisos y tarjetas comparativas

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

    if (!estadoPermisos.allPermissionsGranted) {
        // Muestra Card de error con botón "Conceder permisos"
        return@Column
    }

    // ... botón Capturar + contadores + LazyColumn con ComparativeCaptureCard ...
}
```

`ComparativeCaptureCard` muestra una tarjeta por instante de captura con dos columnas:
- **GOOGLE FLP:** lat/lon con 5 decimales + precisión en metros
- **SENSOR GNSS:** lat/lon o **"SIN SEÑAL"** en rojo si `latitude == null`

El color del botón cambia: azul (primary) cuando está parado, rojo (error) cuando está capturando — feedback visual inmediato.

---

## 18. `ProfileScreen` — 3 sub-vistas

```
ProfileScreen
├── ProfileViewState.Menu          ← punto de entrada
│   ├── avatar + username
│   ├── → Mi Perfil
│   ├── → Mi Actividad
│   └── [Cerrar sesión] → AlertDialog de confirmación
│
├── ProfileViewState.MyProfile
│   ├── Username / Rol
│   ├── Directorio Local (ruta real de filesDir en el dispositivo)
│   ├── Switch "Modo Noche" (lee/escribe isDarkMode en DataStore)
│   ├── Dispositivo / Android Version / API Level
│   └── [Volver]
│
└── ProfileViewState.MyActivity
    ├── Lista combinada de todos los registros (GNSS + Foto + Video + Audio)
    │   ordenada por timestamp descendente
    ├── ActivityRow → icono + label + fecha
    └── click → ActivityDetailDialog
        ├── GpsGoogle: lat/lon/accuracy/speed
        ├── GpsSensors: lat/lon/altitud o "SIN SEÑAL"
        ├── Media: thumbnail (Coil) + botón "Ver Foto/Reproducir Video"
        └── Audio: duración/tamaño + botón "Reproducir Audio"
```

`MyActivityScreen` colecciona directamente desde los repositorios (no necesita ViewModel propio porque solo lee, no tiene lógica de mutación).

`ActivityDetailDialog` usa `FileProvider` para abrir archivos con apps externas:

```kotlin
val uri = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
val intent = Intent(Intent.ACTION_VIEW).apply {
    setDataAndType(uri, mimeType)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
context.startActivity(intent)
```

---

## 19. `FileProvider` — Compartir archivos externos

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <files-path name="my_images" path="photos/" />
    <files-path name="my_videos" path="videos/" />
    <files-path name="my_audios" path="audios/" />
</paths>
```

Sin el `FileProvider`, pasar un `file://` URI a otra app en Android 7+ lanza `FileUriExposedException`. El `FileProvider` genera un `content://` URI seguro con permisos de lectura temporal, permitiendo que la galería o el reproductor del sistema abran los archivos guardados en `filesDir`.

---

## 20. `AndroidManifest.xml` — Permisos ampliados

```xml
<!-- Posicionamiento -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />

<!-- Hardware inalámbrico complementario (para GNSS multi-fuente) -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />

<!-- Multimedia -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />

<!-- Foreground Service + notificaciones (Android 13+) -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<!-- Red (preparado para backend futuro) -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

`ACCESS_BACKGROUND_LOCATION` es necesario si el servicio GPS sigue capturando cuando la app está completamente en segundo plano (pantalla apagada). Requiere aprobación explícita del usuario en Android 10+.

`BLUETOOTH_SCAN` + WiFi están presentes porque el Fused Location Provider puede usar Bluetooth y WiFi para mejorar la precisión en interiores (fuentes alternativas al GPS).

---

## 21. Flujo de datos completo

### Ciclo GNSS

```
[GpsScreen] Botón "Capturar"
  → startForegroundService(GpsCaptureService)
  → scope.launch { while(isActive) { performCaptures(); delay(10_000) } }
  │
  ├── fusedClient.getCurrentLocation().await()
  │     → GpsRepository.saveGooglePoint() → Room tabla gps_google
  │
  └── withTimeoutOrNull(5_000) { getRawGpsLocation() }
        → GpsRepository.saveSensorsPoint(lat=null si timeout)
              → Room tabla gps_sensors
  │
Room emite Flow → GpsViewModel.comparativeHistory
  → combine() + flowOn(Default) → StateFlow
  → GpsScreen recompone → ComparativeCaptureCard por instante
```

### Ciclo de audio

```
[AudioScreen] Botón "Grabar"
  → AudioViewModel.startRecording()
  → MediaRecorder inicia (formato MPEG_4/AAC)
  → timerJob actualiza _elapsedSeconds cada segundo → UI muestra "Xs"

[AudioScreen] Botón "Detener"
  → AudioViewModel.stopRecording()
  → MediaRecorder.stop() + release()
  → si durationMs >= 1000: AudioRepository.registerAudio() → Room
  → si no: File.delete() (descarte silencioso)
  → Room emite → audios StateFlow → AudioScreen recompone con nuevo item
```

### Ciclo de sesión + tema

```
[LoginScreen] onSubmit("jkn", "jkn")
  → SessionViewModel.login()
  → SessionManager.login() → DataStore { is_logged_in=true, username="jkn" }
  → DataStore emite → isLoggedIn=true
  → Navigation recompone → MainScaffold

[ProfileScreen → Mi Perfil] Switch "Modo Noche"
  → SessionViewModel.setDarkMode(true)
  → SessionManager.setDarkMode() → DataStore { dark_mode=true }
  → isDarkMode emite → MainActivity.darkTheme=true
  → AppTheme recompone con darkScheme → toda la UI cambia de color

[ProfileScreen] "Cerrar sesión" → AlertDialog → confirmar
  → SessionViewModel.logout()
  → SessionManager.logout() → DataStore clear (pero preserva dark_mode)
  → isLoggedIn=false → Navigation recompone → LoginScreen
```

---

## 22. `AudioRecorderManager` — Utility class

```kotlin
class AudioRecorderManager(private val context: Context) {

    private var recorder: MediaRecorder? = null

    fun createAudioFile(): File =
        File(File(context.filesDir, "audios").apply { mkdirs() },
             "AUDIO_${System.currentTimeMillis()}.m4a")

    fun start(outputFile: File) {
        stop()  // evita fugas si ya había una grabación
        recorder = (if (SDK_INT >= S) MediaRecorder(context) else MediaRecorder()).apply {
            setAudioSource(MediaRecorder.AudioSource.MIC)
            setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
            setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
            setOutputFile(outputFile.absolutePath)
            prepare(); start()
        }
    }

    fun stop() {
        try { recorder?.apply { stop(); release() } }
        catch (e: Exception) { e.printStackTrace() }
        finally { recorder = null }
    }

    fun getDuration(file: File): Long {
        val retriever = MediaMetadataRetriever()
        return try {
            retriever.setDataSource(file.absolutePath)
            retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_DURATION)?.toLong() ?: 0L
        } catch (e: Exception) { 0L }
        finally { retriever.release() }
    }
}
```

Esta clase está disponible como alternativa al manejo directo en `AudioViewModel`. Encapsula la lógica del `MediaRecorder` y extrae la duración real del archivo con `MediaMetadataRetriever` (más preciso que calcular `stop_time - start_time`).

---

## 23. Decisiones arquitectónicas del proyecto real

| Decisión | Justificación |
|---|---|
| **`GpsSensorsEntity` con lat/lon nullable** | Permite registrar el evento "sin señal GPS" como un registro real en BD, no solo omitirlo. La UI puede mostrar cuántas veces el dispositivo estuvo sin fix. |
| **Loop de coroutine en GPS en vez de callbacks** | El flujo es lineal: captura FLP → captura sensor → delay → repetir. Sin overlap entre ciclos, sin necesidad de desregistrar listeners al final. |
| **`withTimeoutOrNull(5 s)` para el chip GPS** | El chip GPS puede tardar indefinidamente en dar fix. 5 s es el balance entre esperar suficiente y no bloquear el siguiente ciclo de 10 s. |
| **`performCaptures()` usa el mismo `now`** | Ambas fuentes (FLP y sensor) comparten el mismo timestamp para que `comparativeHistory` pueda emparejarlos exactamente. |
| **`comparativeHistory` con `flowOn(Default)`** | La construcción del mapa de registros emparejados opera sobre listas potencialmente largas. Moverlo a `Dispatchers.Default` evita frames perdidos en la UI. |
| **Pestaña Perfil en lugar de Logout** | Más UX: el logout está detrás de un `AlertDialog` de confirmación, evitando cierres accidentales. Además integra el historial de actividad y preferencias. |
| **Dark mode persistido en DataStore** | La preferencia de tema se mantiene entre sesiones. Se conserva incluso al hacer logout. `MainActivity` la aplica antes de renderizar cualquier UI. |
| **DB version 2 con `fallbackToDestructiveMigration`** | El schema cambió (lat/lon nullable). En laboratorio es aceptable resetear la BD. La nota pendiente es migrar a `Migration(1,2)` con `ALTER TABLE` para producción. |
| **FileProvider para abrir archivos** | Android 7+ prohíbe `file://` URIs. El FileProvider mapea `filesDir/photos`, `videos`, `audios` a `content://` URIs seguros para compartir con galería/reproductor. |
| **`PasswordHasher` preparado pero no integrado** | La infraestructura de seguridad existe: PBKDF2 + salt + comparación en tiempo constante. La integración completa requiere almacenar el hash del usuario, lo que es trabajo de un sprint futuro. |

---

## 24. Checklist de estado actual

### Implementado y funcional
- [x] `DemoDataApp` registrada, DI manual con `by lazy`
- [x] `DemoDataDatabase` version 2, 4 entidades, 4 DAOs
- [x] `SessionManager` con `isLoggedIn`, `username`, `isDarkMode`
- [x] `FileStorageManager` con `photos/`, `videos/`, `audios/`
- [x] 3 repositorios coordinando Room + filesDir
- [x] `GpsCaptureService` con loop de coroutine + FLP + sensor con timeout
- [x] `GpsSensorsEntity` con lat/lon nullable (registro de "sin señal")
- [x] `GpsViewModel.comparativeHistory` con emparejamiento por timestamp
- [x] `GpsScreen` con manejo de permisos y tarjetas comparativas dual-panel
- [x] `AudioViewModel` con `isRecording`, `elapsedSeconds`, validación ≥ 1 s
- [x] `ProfileScreen` con 3 sub-vistas (Menu / MyProfile / MyActivity)
- [x] Toggle de modo noche persistido en DataStore + aplicado en `MainActivity`
- [x] `ActivityDetailDialog` con apertura de archivos vía FileProvider
- [x] `PasswordHasher` PBKDF2 con comparación en tiempo constante
- [x] Material 3 Dynamic Color con 4 esquemas de contraste
- [x] `DelayedNotificationWorker` con WorkManager
- [x] Manifest con permisos extendidos (INTERNET, BT, WiFi) + FileProvider

### Pendiente / futuro
- [ ] Integrar `PasswordHasher` en el flujo de login real
- [ ] Migración explícita DB v1→v2 con `Migration` (en lugar de destructiva)
- [ ] Añadir `SyncScreen` con lógica real de envío HTTP
- [ ] `AccessBackgroundLocation` requiere justificación al usuario en Android 10+
- [ ] Tests: DAOs (Room in-memory), `SessionManager`, `GpsViewModel`

---

## 25. Próximos pasos para conectar backend

La arquitectura actual ya está preparada. Solo falta:

1. **`NetworkManager`** con Retrofit + OkHttp (el permiso `INTERNET` ya existe en el manifest).
2. **Columna `is_synced: Boolean`** en cada entidad → `Migration(2, 3)`.
3. **Método `syncToServer()`** en cada repositorio: lee registros no sincronizados, hace POST, marca como sincronizados.
4. **Reemplazar el Toast** de `SyncScreen` por la lógica real (los conteos reactivos ya están).
5. **`WorkManager` periódico** (cada 15 min) para sync en background.
6. **`PasswordHasher` + EncryptedSharedPreferences** para almacenar credenciales locales cifradas.

Ninguno de estos pasos requiere refactorizar la arquitectura: los repositorios son la abstracción correcta y los ViewModels no necesitan cambiar.
