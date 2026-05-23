# Lab 5 — Funcionalidades principales: Login, Multimedia, Audio y Notificaciones

**Autor:** [illarek-lab](https://github.com/illarek-lab)
**Proyecto:** DemoData · `com.illareklab.demodata`
**Temas:** L1 completo · L2 completo · L3 (Corrutinas, Capa de Datos, Repositorio, DI manual)

> **Objetivo del laboratorio:** Construir el esqueleto de navegación de la app, el formulario de login con estado de carga, la captura de fotos y videos con CameraX vía intents del sistema, la grabación de audio con `MediaRecorder` y un timer reactivo, y las notificaciones diferidas con WorkManager.

---

## 1. Arquitectura del laboratorio

```
MainActivity
└── AppTheme (darkTheme desde DataStore)
    └── Navigation()
        ├── LoginScreen  ←→  SessionViewModel
        └── MainScaffold
            ├── TopAppBar  (título + botón logout)
            ├── BottomNavigationBar (6 tabs)
            └── NavHost
                ├── GpsScreen      [Lab 4]
                ├── MediaScreen    ← este lab
                ├── AudioScreen    ← este lab
                ├── SyncScreen     [Lab 6]
                ├── NotificationsScreen ← este lab
                └── ProfileScreen  [Lab 4]

Capa de datos (este lab):
MediaRepository  ←→  MediaDao  ←→  Room (media)
AudioRepository  ←→  AudioDao  ←→  Room (audio)
FileStorageManager  →  filesDir/photos/ videos/ audios/
```

---

## 2. Dependencias (`app/build.gradle.kts`)

Este lab agrega **CameraX** y **WorkManager** a las dependencias del Lab 4. El resto ya existía.

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
    // ── Compose BOM ──
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.material3)
    implementation("androidx.compose.material:material-icons-extended:1.7.8")
    implementation("androidx.navigation:navigation-compose:2.8.0")
    implementation("androidx.compose.ui:ui-text-google-fonts:1.7.8")

    // ── Room ──
    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)
    ksp(libs.androidx.room.compiler)

    // ── DataStore ──
    implementation("androidx.datastore:datastore-preferences:1.1.1")

    // ── ViewModel + lifecycle ──
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.6")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.6")

    // ── Google FLP ──
    implementation("com.google.android.gms:play-services-location:21.3.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.11.0")

    // ── WorkManager: notificaciones diferidas que sobreviven al cierre de la app ──
    implementation("androidx.work:work-runtime-ktx:2.9.0")

    // ── CameraX: captura de fotos y videos ──
    val cameraxVersion = "1.6.0"
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")
    implementation("androidx.camera:camera-video:$cameraxVersion")

    // ── Coil: thumbnails en LazyColumn de MediaScreen ──
    implementation("io.coil-kt:coil-compose:2.7.0")

    // ── Accompanist: permisos de cámara y micrófono en Compose ──
    implementation("com.google.accompanist:accompanist-permissions:0.34.0")
}
```

### ¿Para qué se usa cada nueva dependencia en este lab?

| Dependencia | Uso en el lab |
|---|---|
| `work-runtime-ktx` | `DelayedNotificationWorker` + `OneTimeWorkRequestBuilder` en `NotificationsScreen` |
| `camera-core` | API base de CameraX (lifecycle, use cases) |
| `camera-camera2` | Backend de captura sobre la API Camera2 del OS |
| `camera-lifecycle` | Vincula CameraX al ciclo de vida del Composable |
| `camera-view` | `PreviewView` para mostrar el visor de cámara en pantalla |
| `camera-video` | `VideoCapture` use case para grabar video |
| `coil-compose` | `AsyncImage` carga thumbnails de fotos y videos en `MediaItemRow` |
| `accompanist-permissions` | `rememberPermissionState` para solicitar cámara y micrófono en runtime |

> **¿Por qué 5 artefactos de CameraX?** CameraX está modularizado: puedes incluir solo lo que necesitas. Tomar fotos usa `camera-core + camera-camera2 + camera-lifecycle`. Ver el visor requiere `camera-view`. Grabar video requiere `camera-video`. Incluirlos todos permite usar ambas funcionalidades sin restricciones.

### Permisos en `AndroidManifest.xml`

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />   <!-- Android 13+ -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />

<uses-feature android:name="android.hardware.camera"     android:required="false" />
<uses-feature android:name="android.hardware.microphone" android:required="false" />
```

`android:required="false"` hace que la app sea instalable en dispositivos sin cámara o micrófono. Los permisos se solicitan en runtime desde cada pantalla.

---

## 3. Componentes cubiertos en este lab

| Componente | Archivo |
|---|---|
| Application class + DI manual | `DemoDataApp.kt` |
| Base de datos | `data/local/DemoDataDatabase.kt` |
| Entidades Media y Audio | `data/local/entity/MediaEntity.kt`, `AudioEntity.kt` |
| DAOs Media y Audio | `data/local/dao/MediaDao.kt`, `AudioDao.kt` |
| Almacenamiento de archivos | `data/local/FileStorageManager.kt` |
| Repositorios Media y Audio | `data/repository/MediaRepository.kt`, `AudioRepository.kt` |
| ViewModels Media y Audio | `ui/viewmodel/MediaViewModel.kt`, `AudioViewModel.kt` |
| Navegación principal | `ui/Navigation.kt` |
| Login | `ui/screens/LoginScreen.kt` |
| Multimedia | `ui/screens/MediaScreen.kt` |
| Audio | `ui/screens/AudioScreen.kt` |
| Notificaciones | `ui/screens/NotificationsScreen.kt` |
| Worker de notificación | `workers/DelayedNotificationWorker.kt` |
| Tema y colores | `ui/theme/Theme.kt`, `Color.kt`, `Type.kt` |

---

## 3. `DemoDataApp` — DI manual

```kotlin
class DemoDataApp : Application() {

    val database    by lazy { DemoDataDatabase.getInstance(this) }
    val fileStorage by lazy { FileStorageManager(this) }
    val sessionManager by lazy { SessionManager(this) }

    val gpsRepository   by lazy { GpsRepository(database.gpsGoogleDao(), database.gpsSensorsDao()) }
    val mediaRepository by lazy { MediaRepository(database.mediaDao(), fileStorage) }
    val audioRepository by lazy { AudioRepository(database.audioDao(), fileStorage) }
}
```

`by lazy` garantiza que cada dependencia se crea **una sola vez** y solo cuando se necesita por primera vez. El grafo completo de dependencias es visible de un vistazo: base de datos → repositorios → disponibles para los ViewModels.

Los ViewModels acceden al grafo así:

```kotlin
val app = LocalContext.current.applicationContext as DemoDataApp
val vm: MediaViewModel = viewModel(
    factory = MediaViewModel.Factory(app.mediaRepository, app.fileStorage)
)
```

---

## 4. `DemoDataDatabase` — Singleton thread-safe

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

**`@Volatile`** garantiza que todos los hilos ven el valor más reciente de `INSTANCE` (visibilidad de memoria).
**`synchronized`** garantiza que solo un hilo a la vez ejecuta la creación (exclusión mutua).
El doble check (`INSTANCE ?:` antes y dentro del `synchronized`) evita sincronizar en cada lectura una vez que la instancia ya existe.

---

## 5. Entidades Room — Media y Audio

### `MediaEntity.kt`

```kotlin
@Entity(tableName = "media")
data class MediaEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0L,
    val filePath: String,           // ruta absoluta en filesDir/photos/ o videos/
    val type: String,               // MediaType.PHOTO.name o MediaType.VIDEO.name
    val sizeBytes: Long,
    val durationMs: Long? = null,   // solo videos; null para fotos
    val widthPx: Int? = null,
    val heightPx: Int? = null,
    val timestamp: Long
)

enum class MediaType { PHOTO, VIDEO }
```

**Decisión de diseño:** se usa una sola entidad con un campo discriminador `type` en lugar de dos entidades separadas (`PhotoEntity` + `VideoEntity`). Esto simplifica las queries: puedes pedir toda la multimedia junta (`observeAll()`) o filtrar por tipo (`observeByType("PHOTO")`).

### `AudioEntity.kt`

```kotlin
@Entity(tableName = "audio")
data class AudioEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0L,
    val filePath: String,       // ruta en filesDir/audios/
    val durationMs: Long,
    val sizeBytes: Long,
    val format: String,         // "AAC"
    val timestamp: Long
)
```

---

## 6. DAOs

### `MediaDao.kt`

```kotlin
@Dao
interface MediaDao {
    @Insert
    suspend fun insert(item: MediaEntity): Long

    @Query("SELECT * FROM media ORDER BY timestamp DESC")
    fun observeAll(): Flow<List<MediaEntity>>

    @Query("SELECT * FROM media WHERE type = :type ORDER BY timestamp DESC")
    fun observeByType(type: String): Flow<List<MediaEntity>>

    @Query("SELECT COUNT(*) FROM media WHERE type = 'PHOTO'")
    fun observePhotoCount(): Flow<Int>

    @Query("SELECT COUNT(*) FROM media WHERE type = 'VIDEO'")
    fun observeVideoCount(): Flow<Int>

    @Delete
    suspend fun delete(item: MediaEntity)
}
```

### `AudioDao.kt`

```kotlin
@Dao
interface AudioDao {
    @Insert
    suspend fun insert(item: AudioEntity): Long

    @Query("SELECT * FROM audio ORDER BY timestamp DESC")
    fun observeAll(): Flow<List<AudioEntity>>

    @Query("SELECT COUNT(*) FROM audio")
    fun observeCount(): Flow<Int>

    @Delete
    suspend fun delete(item: AudioEntity)
}
```

---

## 7. `FileStorageManager`

```kotlin
class FileStorageManager(private val context: Context) {

    private val photosDir = File(context.filesDir, "photos").apply { mkdirs() }
    private val videosDir = File(context.filesDir, "videos").apply { mkdirs() }
    private val audiosDir = File(context.filesDir, "audios").apply { mkdirs() }

    fun newPhotoFile(): File = File(photosDir, "photo_${System.currentTimeMillis()}.jpg")
    fun newVideoFile(): File = File(videosDir, "video_${System.currentTimeMillis()}.mp4")
    fun newAudioFile(extension: String = "m4a"): File =
        File(audiosDir, "audio_${System.currentTimeMillis()}.$extension")

    fun deleteFile(path: String): Boolean =
        File(path).takeIf { it.exists() }?.delete() ?: false

    fun fileSize(path: String): Long = File(path).length()
}
```

**`context.filesDir`** es el almacenamiento privado de la app:
- No requiere permisos de almacenamiento externo.
- No es accesible por otras apps (seguridad).
- Se elimina automáticamente al desinstalar la app.

Los subdirectorios `photos/`, `videos/`, `audios/` separan los tipos visualmente y permiten borrar un tipo completo con `deleteRecursively()` si se necesitara.

---

## 8. Repositorios

### `MediaRepository.kt`

```kotlin
class MediaRepository(
    private val mediaDao: MediaDao,
    private val fileStorage: FileStorageManager
) {
    val allMedia:   Flow<List<MediaEntity>> = mediaDao.observeAll()
    val photoCount: Flow<Int> = mediaDao.observePhotoCount()
    val videoCount: Flow<Int> = mediaDao.observeVideoCount()

    suspend fun registerPhoto(filePath: String, widthPx: Int, heightPx: Int): Long =
        mediaDao.insert(MediaEntity(
            filePath = filePath,
            type     = MediaType.PHOTO.name,
            sizeBytes = fileStorage.fileSize(filePath),
            widthPx  = widthPx, heightPx = heightPx,
            timestamp = System.currentTimeMillis()
        ))

    suspend fun registerVideo(filePath: String, durationMs: Long): Long =
        mediaDao.insert(MediaEntity(
            filePath  = filePath,
            type      = MediaType.VIDEO.name,
            sizeBytes = fileStorage.fileSize(filePath),
            durationMs = durationMs,
            timestamp = System.currentTimeMillis()
        ))

    suspend fun delete(item: MediaEntity) {
        fileStorage.deleteFile(item.filePath)   // primero el archivo físico
        mediaDao.delete(item)                   // luego el registro en BD
    }
}
```

**Orden de borrado:** el archivo físico se elimina **antes** que el registro de Room. Si el orden fuera al revés y la app se cierra entre las dos operaciones, quedarían registros en Room apuntando a archivos inexistentes (referencias rotas). Al borrar el archivo primero, el peor caso es un registro huérfano en Room sin archivo, que es más fácil de limpiar.

### `AudioRepository.kt`

```kotlin
class AudioRepository(
    private val audioDao: AudioDao,
    private val fileStorage: FileStorageManager
) {
    val allAudios: Flow<List<AudioEntity>> = audioDao.observeAll()
    val count:     Flow<Int> = audioDao.observeCount()

    suspend fun registerAudio(filePath: String, durationMs: Long, format: String = "AAC"): Long =
        audioDao.insert(AudioEntity(
            filePath  = filePath,
            durationMs = durationMs,
            sizeBytes = fileStorage.fileSize(filePath),
            format    = format,
            timestamp = System.currentTimeMillis()
        ))

    suspend fun delete(item: AudioEntity) {
        fileStorage.deleteFile(item.filePath)
        audioDao.delete(item)
    }
}
```

---

## 9. `Navigation.kt` — Estructura de la app

```kotlin
@Composable
fun Navigation() {
    val app = LocalContext.current.applicationContext as DemoDataApp
    val sessionVm: SessionViewModel = viewModel(
        factory = SessionViewModel.Factory(app.sessionManager)
    )
    val isLoggedIn by sessionVm.isLoggedIn.collectAsStateWithLifecycle()

    // Switch reactivo: cuando DataStore emite, Compose recompone automáticamente
    if (isLoggedIn) MainScaffold(sessionVm) else LoginScreen(onSubmit = sessionVm::login)
}

@Composable
private fun MainScaffold(sessionVm: SessionViewModel) {
    val nav = rememberNavController()
    var selected by remember { mutableIntStateOf(0) }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("DemoData — ${username ?: "?"}") },
                actions = {
                    IconButton(onClick = { sessionVm.logout() }) {
                        Icon(Icons.AutoMirrored.Filled.Logout, "Logout")
                    }
                }
            )
        },
        bottomBar = {
            NavigationBar {
                val tabs = listOf(
                    "gps"     to (LocationOn   to "GNSS"),
                    "media"   to (CameraAlt    to "Multimedia"),
                    "audio"   to (Mic          to "Audio"),
                    "sync"    to (CloudSync    to "Sync"),
                    "notif"   to (Notifications to "Notif"),
                    "profile" to (Person       to "Perfil")
                )
                tabs.forEachIndexed { idx, (route, iconLabel) ->
                    val (icon, label) = iconLabel
                    NavigationBarItem(
                        selected = selected == idx,
                        onClick = { selected = idx; nav.navigate(route) },
                        icon = { Icon(icon, null) },
                        label = { Text(label) }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(navController = nav, startDestination = "gps", modifier = Modifier.padding(padding)) {
            composable("gps")     { GpsScreen() }
            composable("media")   { MediaScreen() }
            composable("audio")   { AudioScreen() }
            composable("sync")    { SyncScreen() }
            composable("notif")   { NotificationsScreen() }
            composable("profile") { ProfileScreen(onLogout = sessionVm::logout, username = username) }
        }
    }
}
```

### El truco del switch reactivo

```
SessionManager.logout() → DataStore { clear() }
  → Flow<Boolean> emite false
  → SessionViewModel.isLoggedIn = false
  → Navigation recompone → muestra LoginScreen

SessionViewModel.login("jkn", "jkn") → DataStore { is_logged_in = true }
  → Flow<Boolean> emite true
  → SessionViewModel.isLoggedIn = true
  → Navigation recompone → muestra MainScaffold
```

Sin `popBackStack()`, sin `Intent.FLAG_ACTIVITY_CLEAR_TASK`, sin gestión manual del back stack. **La UI sigue al estado, no al revés.**

---

## 10. `LoginScreen` — Formulario con estado de carga

```kotlin
@Composable
fun LoginScreen(onSubmit: (String, String, (Boolean) -> Unit) -> Unit) {
    var usuario    by remember { mutableStateOf("") }
    var password   by remember { mutableStateOf("") }
    var error      by remember { mutableStateOf("") }
    var verificando by remember { mutableStateOf(false) }

    Column(/* ... */) {
        Text("DemoData", style = displayMedium, color = colorScheme.primary)
        Text("Sistema de gestión de datos", style = bodyMedium)

        OutlinedTextField(
            value = usuario, onValueChange = { usuario = it },
            label = { Text("Usuario") },
            enabled = !verificando,   // deshabilitado durante la verificación
            singleLine = true,
            modifier = Modifier.fillMaxWidth()
        )
        OutlinedTextField(
            value = password, onValueChange = { password = it },
            label = { Text("Contraseña") },
            enabled = !verificando,
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )

        if (error.isNotEmpty()) Text(error, color = colorScheme.error)

        Button(
            onClick = {
                error = ""
                verificando = true
                onSubmit(usuario, password) { ok ->
                    verificando = false
                    if (!ok) error = "Credenciales incorrectas. Pruebe jkn/jkn."
                }
            },
            enabled = !verificando && usuario.isNotBlank() && password.isNotBlank()
        ) {
            if (verificando) CircularProgressIndicator(strokeWidth = 2.dp)
            else Text("Ingresar")
        }

        Text("Credenciales por defecto: jkn / jkn", style = bodySmall)
    }
}
```

**Estados del formulario:**

| Estado | `verificando` | `enabled` del botón | Contenido del botón |
|---|---|---|---|
| Vacío | false | false | "Ingresar" |
| Con texto | false | true | "Ingresar" |
| Enviando | true | false | `CircularProgressIndicator` |
| Error | false | true | "Ingresar" + texto de error |

---

## 11. `MediaScreen` — CameraX vía intents del sistema

```kotlin
@Composable
fun MediaScreen() {
    var pendingFile by remember { mutableStateOf<File?>(null) }
    var videoStartTimeMs by remember { mutableStateOf(0L) }

    // ── Launcher para fotos (TakePicture contract) ──
    val photoLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.TakePicture()
    ) { success ->
        val file = pendingFile
        if (success && file != null && file.exists()) {
            vm.onPhotoCaptured(file.absolutePath, widthPx = 0, heightPx = 0)
        } else {
            file?.takeIf { it.exists() }?.delete()   // limpia si el usuario canceló
        }
        pendingFile = null
    }

    // ── Launcher para videos (CaptureVideo contract) ──
    val videoLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.CaptureVideo()
    ) { success ->
        val file = pendingFile
        if (success && file != null && file.exists()) {
            val durationMs = System.currentTimeMillis() - videoStartTimeMs
            vm.onVideoCaptured(file.absolutePath, durationMs)
        } else {
            file?.takeIf { it.exists() }?.delete()
        }
        pendingFile = null
    }

    // ── Botón Foto ──
    Button(onClick = {
        val file = vm.newPhotoFile()
        val uri  = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
        pendingFile = file
        photoLauncher.launch(uri)   // abre la cámara del sistema; la app duerme
    }) {
        Icon(PhotoCamera, null); Text("Foto")
    }

    // ── Botón Video ──
    Button(onClick = {
        val file = vm.newVideoFile()
        val uri  = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
        pendingFile = file
        videoStartTimeMs = System.currentTimeMillis()
        videoLauncher.launch(uri)
    }) {
        Icon(Videocam, null); Text("Video")
    }

    // ── Lista reactiva ──
    LazyColumn {
        items(items = mediaList, key = { it.id }) { media ->
            MediaItemRow(media = media, onDelete = { vm.delete(media) })
        }
    }
}
```

**Flujo de captura de foto:**

```
Botón Foto
  → vm.newPhotoFile()           → filesDir/photos/photo_TIMESTAMP.jpg (archivo vacío)
  → FileProvider.getUriForFile() → content://...fileprovider/my_images/photo_TIMESTAMP.jpg
  → photoLauncher.launch(uri)   → Android abre la app de cámara del sistema
  → usuario captura (o cancela)
  → photoLauncher callback { success: Boolean }
      si success → vm.onPhotoCaptured(path) → MediaRepository.registerPhoto() → Room INSERT
      si no      → file.delete() (limpieza)
```

**¿Por qué `FileProvider` en lugar de pasar la ruta directamente?**

Android 7+ prohíbe URIs `file://` en Intents entre apps. `FileProvider` genera un `content://` URI con permiso temporal de escritura para la app de cámara, que escribe la foto directamente en el archivo que creamos en `filesDir`.

### `MediaItemRow` — Thumbnail con Coil

```kotlin
AsyncImage(
    model = File(media.filePath),
    contentDescription = null,
    contentScale = ContentScale.Crop,
    modifier = Modifier
        .size(64.dp)
        .clip(RoundedCornerShape(8.dp))
)
```

Coil detecta automáticamente si el archivo es imagen o video y muestra el thumbnail correspondiente. El `key = { it.id }` en el `LazyColumn` permite a Compose reusar y animar los items eficientemente.

---

## 12. `AudioViewModel` — MediaRecorder con timer reactivo

```kotlin
class AudioViewModel(
    private val context: Context,
    private val audioRepository: AudioRepository,
    private val fileStorage: FileStorageManager
) : ViewModel() {

    val audios  = audioRepository.allAudios.stateIn(viewModelScope, WhileSubscribed(5_000), emptyList())
    val count   = audioRepository.count.stateIn(viewModelScope, WhileSubscribed(5_000), 0)

    private val _isRecording   = MutableStateFlow(false)
    val isRecording = _isRecording.asStateFlow()

    private val _elapsedSeconds = MutableStateFlow(0)
    val elapsedSeconds = _elapsedSeconds.asStateFlow()

    private var recorder: MediaRecorder? = null
    private var currentFile: File? = null
    private var startTimeMs = 0L
    private var timerJob: Job? = null

    fun startRecording(): Boolean {
        return try {
            val file = fileStorage.newAudioFile("m4a")
            currentFile = file

            // API 31+ requiere Context en el constructor
            val rec = if (Build.VERSION.SDK_INT >= S) MediaRecorder(context)
                      else @Suppress("DEPRECATION") MediaRecorder()

            rec.apply {
                setAudioSource(MediaRecorder.AudioSource.MIC)
                setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
                setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
                setOutputFile(file.absolutePath)
                prepare()
                start()
            }

            recorder = rec
            startTimeMs = System.currentTimeMillis()
            _isRecording.value = true
            _elapsedSeconds.value = 0

            // Timer que actualiza el contador cada segundo
            timerJob = viewModelScope.launch {
                while (_isRecording.value) {
                    delay(1_000)
                    _elapsedSeconds.value++
                }
            }
            true
        } catch (e: Exception) { cleanup(); false }
    }

    fun stopRecording() {
        val file = currentFile
        val durationMs = System.currentTimeMillis() - startTimeMs

        try {
            recorder?.apply { stop(); release() }

            // Validación: descartar si < 1 segundo (codec AAC puede fallar con grabaciones muy cortas)
            if (file != null && file.exists() && durationMs >= 1_000L) {
                viewModelScope.launch {
                    audioRepository.registerAudio(file.absolutePath, durationMs, "AAC")
                }
            } else {
                file?.takeIf { it.exists() }?.delete()
            }
        } catch (e: Exception) {
            file?.takeIf { it.exists() }?.delete()
        } finally {
            cleanup()
        }
    }

    private fun cleanup() {
        timerJob?.cancel(); timerJob = null
        recorder = null; currentFile = null
        _isRecording.value = false; _elapsedSeconds.value = 0
    }

    override fun onCleared() {
        super.onCleared()
        // Si el usuario cierra la app mientras graba, liberamos el micrófono
        if (_isRecording.value) {
            try { recorder?.apply { stop(); release() } } catch (_: Exception) {}
            currentFile?.takeIf { it.exists() }?.delete()
        }
    }
}
```

**¿Por qué el timer es una coroutine en vez de un `Handler`?**

Un `Handler.postDelayed` está atado al ciclo de vida del hilo donde se crea y debe cancelarse manualmente. La coroutine del timer vive en `viewModelScope`: se cancela automáticamente cuando el ViewModel se destruye, sin riesgo de fugas de memoria.

---

## 13. `AudioScreen` — UI de grabación

```kotlin
@Composable
fun AudioScreen() {
    val isRecording    by vm.isRecording.collectAsStateWithLifecycle()
    val elapsedSeconds by vm.elapsedSeconds.collectAsStateWithLifecycle()

    // Card con timer — cambia de color según estado
    Card(
        colors = CardDefaults.cardColors(
            containerColor = if (isRecording) colorScheme.errorContainer
                             else colorScheme.surfaceVariant
        )
    ) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            if (isRecording) Icon(FiberManualRecord, null, tint = colorScheme.error)
            Text(if (isRecording) "Grabando…" else "Detenido")
        }
        // Timer en formato mm:ss con fuente monoespaciada
        Text(
            text = String.format(Locale.US, "%02d:%02d",
                elapsedSeconds / 60, elapsedSeconds % 60),
            style = displayMedium,
            fontFamily = FontFamily.Monospace
        )
    }

    // Botón reactivo al estado de grabación
    Button(
        onClick = { if (isRecording) vm.stopRecording() else vm.startRecording() },
        colors = ButtonDefaults.buttonColors(
            containerColor = if (isRecording) colorScheme.error else colorScheme.primary
        )
    ) {
        Icon(if (isRecording) Stop else Mic, null)
        Text(if (isRecording) "Detener grabación" else "Iniciar grabación")
    }

    // Lista de grabaciones guardadas
    LazyColumn {
        items(items = audios, key = { it.id }) { audio ->
            AudioItemRow(audio = audio, onDelete = { vm.delete(audio) })
        }
    }
}
```

El timer muestra `00:00` → `00:01` → ... en tiempo real porque `elapsedSeconds` es un `StateFlow<Int>` que emite cada segundo desde la coroutine del ViewModel. La fuente monoespaciada (`FontFamily.Monospace`) evita que el layout "salte" al cambiar de `00:09` a `00:10`.

---

## 14. `NotificationsScreen` + `DelayedNotificationWorker`

### Pantalla

```kotlin
@Composable
fun NotificationsScreen() {
    var mensaje by remember { mutableStateOf("") }
    var ultimoEnvio by remember { mutableStateOf<String?>(null) }
    var contadorProgramadas by remember { mutableStateOf(0) }

    // Android 13+ requiere POST_NOTIFICATIONS en runtime
    val notifPermission = if (Build.VERSION.SDK_INT >= TIRAMISU)
        rememberPermissionState(POST_NOTIFICATIONS) else null

    OutlinedTextField(
        value = mensaje, onValueChange = { mensaje = it },
        label = { Text("Mensaje de la notificación") },
        minLines = 2, maxLines = 4,
        modifier = Modifier.fillMaxWidth()
    )

    Button(
        onClick = {
            val request = OneTimeWorkRequestBuilder<DelayedNotificationWorker>()
                .setInitialDelay(10, TimeUnit.SECONDS)
                .setInputData(workDataOf(DelayedNotificationWorker.INPUT_MESSAGE to mensaje))
                .build()
            WorkManager.getInstance(context).enqueue(request)
            ultimoEnvio = mensaje
            contadorProgramadas++
            mensaje = ""
        },
        enabled = mensaje.isNotBlank()
    ) {
        Icon(NotificationAdd, null)
        Text("Programar notificación (10 s)")
    }
}
```

### Worker

```kotlin
class DelayedNotificationWorker(context: Context, params: WorkerParameters) : Worker(context, params) {

    companion object {
        const val INPUT_MESSAGE = "input_message"
    }

    override fun doWork(): Result {
        val message = inputData.getString(INPUT_MESSAGE) ?: "Mensaje vacío"

        val manager = applicationContext.getSystemService(NOTIFICATION_SERVICE) as NotificationManager
        manager.createNotificationChannel(
            NotificationChannel("fleet_alerts", "Alertas DemoData", IMPORTANCE_HIGH)
        )

        manager.notify(
            System.currentTimeMillis().toInt(),
            NotificationCompat.Builder(applicationContext, "fleet_alerts")
                .setSmallIcon(android.R.drawable.ic_dialog_info)
                .setContentTitle("Recordatorio DemoData")
                .setContentText(message)
                .setPriority(PRIORITY_HIGH)
                .setAutoCancel(true)
                .build()
        )
        return Result.success()
    }
}
```

**¿Por qué WorkManager y no `Handler.postDelayed`?**

| `Handler.postDelayed` | WorkManager |
|---|---|
| Vive en el proceso de la app | Delegado al `JobScheduler` del OS |
| Muere si el proceso se cierra | Sobrevive al cierre de la app |
| No garantiza ejecución mínima | Garantizado por el sistema |
| Fácil de implementar | Requiere `Worker` + registro |

Para notificaciones que deben llegar aunque el usuario cierre la app, WorkManager es la única opción correcta.

---

## 15. Tema Material 3 — `AppTheme`

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        // Dynamic Color: extrae la paleta del fondo de pantalla del usuario (Android 12+)
        dynamicColor && Build.VERSION.SDK_INT >= S -> {
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> darkScheme    // esquema oscuro estático
        else -> lightScheme        // esquema claro estático
    }
    MaterialTheme(colorScheme = colorScheme, typography = AppTypography, content = content)
}
```

El tema se aplica en `MainActivity`, que recibe `darkTheme` desde el `SessionViewModel`:

```kotlin
val isDarkModePref by sessionVm.isDarkMode.collectAsState()
val darkTheme = isDarkModePref ?: isSystemInDarkTheme()   // null = seguir al sistema

AppTheme(darkTheme = darkTheme) { Navigation() }
```

---

## 16. Flujos de datos completos del laboratorio

### Foto/Video

```
[MediaScreen] tap "Foto"
  → vm.newPhotoFile() → filesDir/photos/photo_TIMESTAMP.jpg (vacío)
  → FileProvider URI → photoLauncher.launch(uri)
  → Camera app del sistema escribe la foto en el archivo
  → callback { success = true }
  → vm.onPhotoCaptured(path) → MediaRepository.registerPhoto() → Room INSERT
  → Flow<List<MediaEntity>> emite → mediaList StateFlow → LazyColumn recompone
```

### Audio

```
[AudioScreen] tap "Iniciar grabación"
  → AudioViewModel.startRecording()
  → MediaRecorder.start() (toma control del micrófono)
  → timerJob: delay(1s) → _elapsedSeconds++ → AudioScreen actualiza timer

[AudioScreen] tap "Detener grabación"
  → AudioViewModel.stopRecording()
  → MediaRecorder.stop() + release()
  → si durationMs >= 1000: AudioRepository.registerAudio() → Room INSERT
  → Flow<List<AudioEntity>> emite → audios StateFlow → LazyColumn recompone
```

### Notificación

```
[NotificationsScreen] tap "Programar notificación"
  → OneTimeWorkRequestBuilder con delay de 10 s
  → WorkManager.enqueue(request)   ← delega al JobScheduler del OS
  → [10 segundos después, aunque la app esté cerrada]
  → DelayedNotificationWorker.doWork()
  → NotificationManager.notify()
  → Notificación visible en la barra del sistema
```

---

## 17. Conceptos de las listas cubiertos

| Tema | Lista | Cómo se ve en este lab |
|---|---|---|
| Navegación y arquitectura | L1 | `Navigation.kt`: `NavHost` con 6 `composable()`, switch reactivo login/main |
| Actividades y ciclos de vida | L1 | `MainActivity` única; `AudioViewModel.onCleared()` libera el micrófono |
| StateFlow y UDF | L1 | `isRecording`, `elapsedSeconds`, `mediaList`, `audios`: datos fluyen en una sola dirección |
| LiveData | L1 | ❌ No se usa — reemplazado por `StateFlow` + `collectAsStateWithLifecycle` |
| ViewModel | L1 | `MediaViewModel`, `AudioViewModel` con `Factory`; ciclo de vida gestionado por Compose |
| Navegación Compose | L1 | `rememberNavController()`, `NavHost`, `composable()`, `BottomNavigationBar` |
| IU responsiva | L1 | `weight(1f)` en botones de Media; `LazyColumn` para listas de longitud variable |
| Temas | L2 | `AppTheme` con Dynamic Color; `darkTheme` desde DataStore vía `SessionViewModel` |
| Menús | L2 | `TopAppBar` + `BottomNavigationBar` definidos en `Navigation.kt` |
| Actividades | L2 | Single-Activity pattern; `GpsCaptureService` como componente de servicio |
| Layouts | L2 | `Column`, `Row`, `Scaffold` en todas las pantallas; `LazyColumn` en Media y Audio |
| Containers | L2 | `Card` en Media y Audio; `Scaffold` como contenedor principal |
| Formularios | L2 | `LoginScreen`: `OutlinedTextField` + validación + `CircularProgressIndicator` |
| Adaptadores | L2 | `LazyColumn + items(key = { it.id })` en Media y Audio — equivale a `RecyclerView + DiffUtil` |
| Corrutinas | L3 | `viewModelScope.launch` en Audio; `timerJob` con `delay(1s)`; `onCleared()` cancela todo |
| Capa de Datos | L3 | Room: `MediaEntity`, `AudioEntity`, `MediaDao`, `AudioDao`; DataStore en SessionManager |
| Repositorio | L3 | `MediaRepository`, `AudioRepository`: coordinan Room + disco; la UI nunca toca DAOs |
| DI manual | L3 | `DemoDataApp by lazy`; `MediaViewModel.Factory`; `AudioViewModel.Factory` |
| Views y Compose | L3 | 100% Jetpack Compose; `AsyncImage` (Coil) para thumbnails; sin XML |

---

## 18. Ejercicios propuestos

1. **Añadir un chip de filtro** en `MediaScreen` para mostrar solo fotos o solo videos usando `observeByType()` del `MediaDao`.

2. **Mostrar la duración real del audio** usando `AudioRecorderManager.getDuration(file)` con `MediaMetadataRetriever` en lugar de calcular `stop_time - start_time`.

3. **Limitar las notificaciones a 5** en `NotificationsScreen`: deshabilitar el botón si `contadorProgramadas >= 5`.

4. **Agregar un campo `username`** al `Worker` de notificación y mostrarlo en el título: `"Recordatorio para $username"`.

5. **Implementar swipe-to-delete** en la lista de audios usando `SwipeToDismiss` de Material 3.

---

## 19. Checklist del laboratorio

- [ ] `DemoDataApp` registrada como `android:name=".DemoDataApp"` en el manifest
- [ ] `DemoDataDatabase` con 4 entidades y singleton thread-safe
- [ ] `MediaEntity` con campo discriminador `type` (PHOTO/VIDEO)
- [ ] `AudioEntity` con `format = "AAC"` y `durationMs`
- [ ] `MediaDao` con `observeAll()`, `observeByType()`, `observePhotoCount()`, `observeVideoCount()`, `delete()`
- [ ] `AudioDao` con `observeAll()`, `observeCount()`, `delete()`
- [ ] `FileStorageManager` crea `photos/`, `videos/`, `audios/` en `filesDir`
- [ ] `MediaRepository.delete()` borra archivo físico antes del registro en Room
- [ ] `AudioRepository.registerAudio()` construye `AudioEntity` completa con `sizeBytes`
- [ ] `Navigation.kt` con `NavHost` de 6 rutas + switch reactivo `isLoggedIn`
- [ ] `LoginScreen` con `verificando` + `CircularProgressIndicator` + validación de campos
- [ ] `MediaScreen` con `rememberLauncherForActivityResult` para foto y video
- [ ] `MediaScreen` usa `FileProvider` para generar URIs de cámara
- [ ] `AudioViewModel.startRecording()` inicia `MediaRecorder` y coroutine del timer
- [ ] `AudioViewModel.stopRecording()` valida `durationMs >= 1000` antes de guardar
- [ ] `AudioViewModel.onCleared()` libera el micrófono si la grabación estaba activa
- [ ] `AudioScreen` muestra timer en formato `mm:ss` con `FontFamily.Monospace`
- [ ] `NotificationsScreen` solicita permiso `POST_NOTIFICATIONS` en Android 13+
- [ ] `DelayedNotificationWorker` registrado en el manifest y encola con `WorkManager`
- [ ] `AppTheme` aplica Dynamic Color en Android 12+ y esquemas estáticos en versiones anteriores
- [ ] App compila sin errores y se puede tomar foto, grabar audio y recibir notificación
