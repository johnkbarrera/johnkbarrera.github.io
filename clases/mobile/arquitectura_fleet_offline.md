# Arquitectura Offline-First — App de Flotas

### Persistencia local con Jetpack Room + Preferences DataStore + filesDir

> **Objetivo:** Diseñar la arquitectura completa de una app Android Kotlin **100% offline** que gestione el ciclo de captura → almacenamiento local → consulta de información de flotas (GPS, multimedia, audios, notificaciones), con sesión persistente y 6 pestañas. **Sin endpoints ni HTTP**: todo se guarda en SQLite (Room) y en el directorio privado de la app (`filesDir`).

> **Stack:** Kotlin + Jetpack Compose + Room + DataStore + WorkManager + CameraX + MediaRecorder + FusedLocationProvider + ViewModel + Coroutines/Flow.

---

## 1. Diagrama de capas

```
┌─────────────────────────────────────────────────────────────┐
│                    UI (Compose Screens)                      │
│  LoginScreen · GpsScreen · MediaScreen · AudioScreen ·       │
│  SyncScreen · NotificationsScreen · LogoutScreen             │
└──────────────────────┬──────────────────────────────────────┘
                       │ observa StateFlow
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  ViewModels (1 por pantalla)                 │
│  SessionViewModel · GpsViewModel · MediaViewModel ·          │
│  AudioViewModel · SyncViewModel                              │
└──────────────────────┬──────────────────────────────────────┘
                       │ inyecta + invoca
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                       Repositorios                           │
│  GpsRepository · MediaRepository · AudioRepository           │
│  (fuente única de verdad — expone Flow<List<...>>)           │
└──────┬───────────────────┬────────────────────┬─────────────┘
       │                   │                    │
       ▼                   ▼                    ▼
┌────────────┐    ┌─────────────────┐   ┌───────────────────┐
│ Room DAOs  │    │ FileStorage     │   │ SessionManager    │
│ (SQLite)   │    │ (filesDir)      │   │ (DataStore Prefs) │
└────────────┘    └─────────────────┘   └───────────────────┘
```

**Principio clave (Offline-First):** los **DAOs** son la **única fuente de verdad** para la UI. Los servicios (GPS, captura de foto/video/audio) **escriben** en Room/filesDir; las pantallas **leen** desde Room vía `Flow`. Nunca pasamos datos crudos directamente del servicio a la UI: pasamos por la BD.

---

## 2. Estructura del proyecto

```
com.illareklab.fleet/
├── FleetApp.kt                    ← Application class (DI manual)
├── MainActivity.kt
│
├── data/
│   ├── local/
│   │   ├── FleetDatabase.kt
│   │   ├── entity/
│   │   │   ├── GpsGoogleEntity.kt
│   │   │   ├── GpsSensorsEntity.kt
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
│   │   └── SessionManager.kt      ← DataStore
│   │
│   └── repository/
│       ├── GpsRepository.kt
│       ├── MediaRepository.kt
│       └── AudioRepository.kt
│
├── services/
│   └── GpsCaptureService.kt       ← Foreground Service (cada 10s)
│
├── workers/
│   └── DelayedNotificationWorker.kt
│
└── ui/
    ├── Navigation.kt
    ├── theme/
    │   └── Theme.kt                ← AppTheme generado
    ├── viewmodel/
    │   ├── SessionViewModel.kt
    │   ├── GpsViewModel.kt
    │   ├── MediaViewModel.kt
    │   ├── AudioViewModel.kt
    │   └── SyncViewModel.kt
    └── screens/
        ├── LoginScreen.kt
        ├── GpsScreen.kt
        ├── MediaScreen.kt
        ├── AudioScreen.kt
        ├── SyncScreen.kt
        ├── NotificationsScreen.kt
        └── LogoutScreen.kt
```

---

## 3. Dependencias (`app/build.gradle.kts`)

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
    id("com.google.devtools.ksp") version "2.0.21-1.0.27"
}

dependencies {
    // ── Compose básico ──
    implementation(platform("androidx.compose:compose-bom:2024.09.03"))
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material:material-icons-extended:1.6.7")
    implementation("androidx.activity:activity-compose:1.9.2")

    // ── Navegación ──
    implementation("androidx.navigation:navigation-compose:2.8.0")

    // ── Room (SQLite) ──
    val roomVersion = "2.6.1"
    implementation("androidx.room:room-runtime:$roomVersion")
    implementation("androidx.room:room-ktx:$roomVersion")
    ksp("androidx.room:room-compiler:$roomVersion")

    // ── DataStore (Preferences) ──
    implementation("androidx.datastore:datastore-preferences:1.1.1")

    // ── ViewModel + lifecycle ──
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.6")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.6")

    // ── WorkManager ──
    implementation("androidx.work:work-runtime-ktx:2.9.1")

    // ── Localización (FLP) ──
    implementation("com.google.android.gms:play-services-location:21.3.0")

    // ── CameraX (fotos y videos) ──
    val cameraxVersion = "1.4.0"
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")
    implementation("androidx.camera:camera-video:$cameraxVersion")

    // ── Coil (thumbnails de fotos/videos en LazyColumn) ──
    implementation("io.coil-kt:coil-compose:2.7.0")

    // ── Permisos en Compose ──
    implementation("com.google.accompanist:accompanist-permissions:0.34.0")
}
```

### ¿Por qué KSP en vez de KAPT?

Room se basa en procesadores de anotaciones para generar el código del schema en tiempo de compilación. KAPT (el procesador clásico) ejecuta los procesadores **dentro de la JVM de Java**, lo que es **2-3 veces más lento** que KSP. KSP es el reemplazo moderno escrito en Kotlin nativo. Para Room 2.6+ es la opción recomendada por Google.

---

## 4. Capa de datos: Entidades Room

### 4.1 `GpsGoogleEntity` — Ubicación del FLP de Google

```kotlin
package com.illareklab.fleet.data.local.entity

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "gps_google")
data class GpsGoogleEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0L,

    val latitude: Double,
    val longitude: Double,

    // Metadatos del FLP que NO se obtienen del LocationManager crudo
    val accuracy: Float,         // precisión horizontal en metros
    val speed: Float? = null,    // m/s, null si no disponible
    val bearing: Float? = null,  // grados desde el norte

    val timestamp: Long          // System.currentTimeMillis() en UTC
)
```

### 4.2 `GpsSensorsEntity` — Ubicación cruda del hardware

```kotlin
package com.illareklab.fleet.data.local.entity

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "gps_sensors")
data class GpsSensorsEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0L,

    val latitude: Double,
    val longitude: Double,

    val provider: String,           // "gps", "network" o "passive"
    val altitude: Double? = null,   // metros sobre el nivel del mar
    val satellites: Int? = null,    // satélites usados (si lo expone el OS)

    val timestamp: Long
)
```

### 4.3 `MediaEntity` — Fotos y videos

```kotlin
package com.illareklab.fleet.data.local.entity

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "media")
data class MediaEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0L,

    val filePath: String,          // ruta absoluta dentro de filesDir
    val type: String,              // "PHOTO" o "VIDEO" (usar MediaType.name)
    val sizeBytes: Long,
    val durationMs: Long? = null,  // solo videos; null para fotos
    val widthPx: Int? = null,
    val heightPx: Int? = null,
    val timestamp: Long
)

enum class MediaType { PHOTO, VIDEO }
```

> **Decisión de diseño:** uso un único `MediaEntity` con un campo `type` discriminador, en vez de dos entidades separadas (`PhotoEntity` + `VideoEntity`). El consumidor (Sync, lista combinada) puede pedirlos juntos sin hacer `UNION`. Si en el futuro divergen los metadatos, se pueden separar.

### 4.4 `AudioEntity` — Grabaciones de audio

```kotlin
package com.illareklab.fleet.data.local.entity

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "audio")
data class AudioEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0L,

    val filePath: String,
    val durationMs: Long,
    val sizeBytes: Long,
    val format: String,     // "AAC" o "MP3"
    val timestamp: Long
)
```

---

## 5. Capa de datos: DAOs

### 5.1 `GpsGoogleDao.kt`

```kotlin
package com.illareklab.fleet.data.local.dao

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import com.illareklab.fleet.data.local.entity.GpsGoogleEntity
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

### 5.2 `GpsSensorsDao.kt`

```kotlin
package com.illareklab.fleet.data.local.dao

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import com.illareklab.fleet.data.local.entity.GpsSensorsEntity
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

### 5.3 `MediaDao.kt`

```kotlin
package com.illareklab.fleet.data.local.dao

import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.Query
import com.illareklab.fleet.data.local.entity.MediaEntity
import kotlinx.coroutines.flow.Flow

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

### 5.4 `AudioDao.kt`

```kotlin
package com.illareklab.fleet.data.local.dao

import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.Query
import com.illareklab.fleet.data.local.entity.AudioEntity
import kotlinx.coroutines.flow.Flow

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

### ¿Por qué `Flow<List<T>>` en vez de `suspend fun getAll(): List<T>`?

Una función `suspend` se ejecuta **una sola vez** y devuelve un snapshot. Si otro componente (el `GpsCaptureService`) inserta un registro nuevo, la UI no se entera.

`Flow<List<T>>` emite **automáticamente** cada vez que cambia la tabla observada. Room genera por debajo un trigger SQLite que notifica al Flow ante cualquier INSERT/UPDATE/DELETE. La UI se reactualiza sin llamar manualmente a `refresh()`.

Este es el corazón del patrón **Offline-First**: la BD es reactiva.

---

## 6. Database

```kotlin
package com.illareklab.fleet.data.local

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import com.illareklab.fleet.data.local.dao.AudioDao
import com.illareklab.fleet.data.local.dao.GpsGoogleDao
import com.illareklab.fleet.data.local.dao.GpsSensorsDao
import com.illareklab.fleet.data.local.dao.MediaDao
import com.illareklab.fleet.data.local.entity.AudioEntity
import com.illareklab.fleet.data.local.entity.GpsGoogleEntity
import com.illareklab.fleet.data.local.entity.GpsSensorsEntity
import com.illareklab.fleet.data.local.entity.MediaEntity

@Database(
    entities = [
        GpsGoogleEntity::class,
        GpsSensorsEntity::class,
        MediaEntity::class,
        AudioEntity::class
    ],
    version = 1,
    exportSchema = false
)
abstract class FleetDatabase : RoomDatabase() {

    abstract fun gpsGoogleDao(): GpsGoogleDao
    abstract fun gpsSensorsDao(): GpsSensorsDao
    abstract fun mediaDao(): MediaDao
    abstract fun audioDao(): AudioDao

    companion object {
        @Volatile private var INSTANCE: FleetDatabase? = null

        fun getInstance(context: Context): FleetDatabase =
            INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(
                    context.applicationContext,
                    FleetDatabase::class.java,
                    "fleet.db"
                ).build().also { INSTANCE = it }
            }
    }
}
```

### ¿Por qué el patrón Singleton con `@Volatile` y `synchronized`?

Abrir múltiples instancias de la misma BD SQLite causa **corrupción de archivos** y bloqueos. El doble check con `@Volatile` (visibilidad entre hilos) + `synchronized` (exclusión mutua durante la creación) garantiza que solo se crea **una** instancia incluso con múltiples hilos llamando al mismo tiempo. Es el patrón Singleton canónico de Java/Kotlin.

---

## 7. Sesión: DataStore Preferences

### 7.1 `SessionManager.kt`

```kotlin
package com.illareklab.fleet.data.session

import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.booleanPreferencesKey
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

// Extensión de Context — solo se crea una vez en todo el proceso
private val Context.sessionDataStore: DataStore<Preferences> by preferencesDataStore(
    name = "fleet_session"
)

class SessionManager(private val context: Context) {

    private companion object {
        val KEY_IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
        val KEY_USERNAME = stringPreferencesKey("username")
    }

    /** Flow reactivo del estado de sesión. La UI lo observa con collectAsState. */
    val isLoggedIn: Flow<Boolean> = context.sessionDataStore.data
        .map { prefs -> prefs[KEY_IS_LOGGED_IN] ?: false }

    val currentUsername: Flow<String?> = context.sessionDataStore.data
        .map { prefs -> prefs[KEY_USERNAME] }

    suspend fun login(username: String) {
        context.sessionDataStore.edit { prefs ->
            prefs[KEY_IS_LOGGED_IN] = true
            prefs[KEY_USERNAME] = username
        }
    }

    suspend fun logout() {
        context.sessionDataStore.edit { prefs ->
            prefs.clear()
        }
    }
}
```

### ¿Por qué DataStore y no SharedPreferences?

| Aspecto | SharedPreferences | Preferences DataStore |
|---|---|---|
| API | Bloqueante (síncrono) | Asíncrona (Flow + suspend) |
| Hilo principal | Riesgo de ANR si se llama mal | Imposible bloquear el main thread |
| Reactividad | No emite cambios | Emite Flow ante cualquier cambio |
| Atomicidad de transacciones | Frágil | Garantizada |
| Recomendación oficial | Deprecado por DataStore | **Recomendado desde Android 11** |

El `Flow<Boolean>` de `isLoggedIn` es exactamente lo que necesitamos para que el `Navigation` reaccione **automáticamente** al login y logout sin tener que recargar manualmente la app.

---

## 8. Almacenamiento de archivos: `FileStorageManager`

Todos los archivos físicos van a `context.filesDir`, que es:
- **Privado** a la app (otras apps no pueden leerlo).
- **Sin permisos** especiales necesarios.
- **Limpiado automáticamente** al desinstalar.

```kotlin
package com.illareklab.fleet.data.local

import android.content.Context
import java.io.File

class FileStorageManager(private val context: Context) {

    private val photosDir = File(context.filesDir, "photos").apply { mkdirs() }
    private val videosDir = File(context.filesDir, "videos").apply { mkdirs() }
    private val audiosDir = File(context.filesDir, "audios").apply { mkdirs() }

    fun newPhotoFile(): File =
        File(photosDir, "photo_${System.currentTimeMillis()}.jpg")

    fun newVideoFile(): File =
        File(videosDir, "video_${System.currentTimeMillis()}.mp4")

    fun newAudioFile(extension: String = "m4a"): File =
        File(audiosDir, "audio_${System.currentTimeMillis()}.$extension")

    /** Elimina el archivo del disco. Retorna true si fue eliminado. */
    fun deleteFile(path: String): Boolean =
        File(path).takeIf { it.exists() }?.delete() ?: false

    /** Lee tamaño en bytes de un archivo existente. */
    fun fileSize(path: String): Long = File(path).length()
}
```

### ¿Por qué subdirectorios?

`filesDir/photos/`, `/videos/`, `/audios/` separan visualmente los tipos al inspeccionar con Device File Explorer. Además, permite eliminar todos los archivos de un tipo con un solo `deleteRecursively()` si se necesita.

---

## 9. Repositorios

Los repositorios son la **fachada** sobre Room + FileStorage. La UI nunca interactúa con DAOs directamente.

### 9.1 `GpsRepository.kt`

```kotlin
package com.illareklab.fleet.data.repository

import com.illareklab.fleet.data.local.dao.GpsGoogleDao
import com.illareklab.fleet.data.local.dao.GpsSensorsDao
import com.illareklab.fleet.data.local.entity.GpsGoogleEntity
import com.illareklab.fleet.data.local.entity.GpsSensorsEntity
import kotlinx.coroutines.flow.Flow

class GpsRepository(
    private val googleDao: GpsGoogleDao,
    private val sensorsDao: GpsSensorsDao
) {
    // ── Streams reactivos para la UI ──
    val googlePoints: Flow<List<GpsGoogleEntity>> = googleDao.observeAll()
    val sensorsPoints: Flow<List<GpsSensorsEntity>> = sensorsDao.observeAll()

    val googleCount: Flow<Int> = googleDao.observeCount()
    val sensorsCount: Flow<Int> = sensorsDao.observeCount()

    // ── Mutaciones (llamadas desde el servicio de captura) ──
    suspend fun saveGooglePoint(point: GpsGoogleEntity) = googleDao.insert(point)
    suspend fun saveSensorsPoint(point: GpsSensorsEntity) = sensorsDao.insert(point)

    suspend fun clearAll() {
        googleDao.deleteAll()
        sensorsDao.deleteAll()
    }
}
```

### 9.2 `MediaRepository.kt`

```kotlin
package com.illareklab.fleet.data.repository

import com.illareklab.fleet.data.local.FileStorageManager
import com.illareklab.fleet.data.local.dao.MediaDao
import com.illareklab.fleet.data.local.entity.MediaEntity
import com.illareklab.fleet.data.local.entity.MediaType
import kotlinx.coroutines.flow.Flow

class MediaRepository(
    private val mediaDao: MediaDao,
    private val fileStorage: FileStorageManager
) {
    val allMedia: Flow<List<MediaEntity>> = mediaDao.observeAll()
    val photoCount: Flow<Int> = mediaDao.observePhotoCount()
    val videoCount: Flow<Int> = mediaDao.observeVideoCount()

    /**
     * Registra una foto ya guardada en disco. La pantalla de cámara llama
     * a fileStorage.newPhotoFile() antes para obtener el path destino,
     * captura ahí, y luego invoca este método con el path resultante.
     */
    suspend fun registerPhoto(
        filePath: String,
        widthPx: Int,
        heightPx: Int
    ): Long = mediaDao.insert(
        MediaEntity(
            filePath = filePath,
            type = MediaType.PHOTO.name,
            sizeBytes = fileStorage.fileSize(filePath),
            widthPx = widthPx,
            heightPx = heightPx,
            timestamp = System.currentTimeMillis()
        )
    )

    suspend fun registerVideo(
        filePath: String,
        durationMs: Long
    ): Long = mediaDao.insert(
        MediaEntity(
            filePath = filePath,
            type = MediaType.VIDEO.name,
            sizeBytes = fileStorage.fileSize(filePath),
            durationMs = durationMs,
            timestamp = System.currentTimeMillis()
        )
    )

    /** Borra el archivo físico Y el registro en Room en una sola operación. */
    suspend fun delete(item: MediaEntity) {
        fileStorage.deleteFile(item.filePath)
        mediaDao.delete(item)
    }
}
```

### 9.3 `AudioRepository.kt`

```kotlin
package com.illareklab.fleet.data.repository

import com.illareklab.fleet.data.local.FileStorageManager
import com.illareklab.fleet.data.local.dao.AudioDao
import com.illareklab.fleet.data.local.entity.AudioEntity
import kotlinx.coroutines.flow.Flow

class AudioRepository(
    private val audioDao: AudioDao,
    private val fileStorage: FileStorageManager
) {
    val allAudios: Flow<List<AudioEntity>> = audioDao.observeAll()
    val count: Flow<Int> = audioDao.observeCount()

    suspend fun registerAudio(
        filePath: String,
        durationMs: Long,
        format: String = "AAC"
    ): Long = audioDao.insert(
        AudioEntity(
            filePath = filePath,
            durationMs = durationMs,
            sizeBytes = fileStorage.fileSize(filePath),
            format = format,
            timestamp = System.currentTimeMillis()
        )
    )

    suspend fun delete(item: AudioEntity) {
        fileStorage.deleteFile(item.filePath)
        audioDao.delete(item)
    }
}
```

### Principio de coordinación archivo + Room

Cuando se elimina un media, **siempre** se borra primero el archivo físico **y luego** el registro de Room. Si el orden fuera al revés y la app se cierra entre las dos operaciones, quedarían archivos huérfanos en disco. El repositorio garantiza la coordinación atómica desde el punto de vista del consumidor.

---

## 10. Application class (inyección manual)

En proyectos profesionales se usa Hilt o Koin. Para este lab — y para mantener visibilidad pedagógica del flujo — usamos **DI manual** vía una `Application` class.

```kotlin
package com.illareklab.fleet

import android.app.Application
import com.illareklab.fleet.data.local.FileStorageManager
import com.illareklab.fleet.data.local.FleetDatabase
import com.illareklab.fleet.data.repository.AudioRepository
import com.illareklab.fleet.data.repository.GpsRepository
import com.illareklab.fleet.data.repository.MediaRepository
import com.illareklab.fleet.data.session.SessionManager

class FleetApp : Application() {

    // Inicialización perezosa: solo se crea al primer acceso
    val database by lazy { FleetDatabase.getInstance(this) }
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

Registrarla en el manifest:

```xml
<application
    android:name=".FleetApp"
    ... >
```

### ¿Cómo acceden los ViewModels?

```kotlin
// En cualquier @Composable:
val app = LocalContext.current.applicationContext as FleetApp
val viewModel: GpsViewModel = viewModel(
    factory = GpsViewModel.Factory(app.gpsRepository, app.sessionManager)
)
```

---

## 11. ViewModels

### 11.1 `SessionViewModel.kt`

```kotlin
package com.illareklab.fleet.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.illareklab.fleet.data.session.SessionManager
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

class SessionViewModel(
    private val sessionManager: SessionManager
) : ViewModel() {

    val isLoggedIn = sessionManager.isLoggedIn.stateIn(
        scope = viewModelScope,
        started = SharingStarted.Eagerly,
        initialValue = false
    )

    val username = sessionManager.currentUsername.stateIn(
        scope = viewModelScope,
        started = SharingStarted.Eagerly,
        initialValue = null
    )

    fun login(username: String, password: String, onResult: (Boolean) -> Unit) {
        // Para el laboratorio: credenciales locales fijas.
        // En producción se delega a un backend o EncryptedSharedPreferences.
        if (username == "admin" && password == "admin") {
            viewModelScope.launch {
                sessionManager.login(username)
                onResult(true)
            }
        } else {
            onResult(false)
        }
    }

    fun logout() {
        viewModelScope.launch {
            sessionManager.logout()
        }
    }

    class Factory(private val sessionManager: SessionManager) : ViewModelProvider.Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel> create(modelClass: Class<T>): T =
            SessionViewModel(sessionManager) as T
    }
}
```

### 11.2 `GpsViewModel.kt`

```kotlin
package com.illareklab.fleet.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.illareklab.fleet.data.repository.GpsRepository
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.stateIn

class GpsViewModel(
    private val gpsRepository: GpsRepository
) : ViewModel() {

    val googlePoints = gpsRepository.googlePoints.stateIn(
        viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList()
    )
    val sensorsPoints = gpsRepository.sensorsPoints.stateIn(
        viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList()
    )

    class Factory(private val gpsRepository: GpsRepository) : ViewModelProvider.Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel> create(modelClass: Class<T>): T =
            GpsViewModel(gpsRepository) as T
    }
}
```

### 11.3 `SyncViewModel.kt` (combina los conteos de todas las tablas)

```kotlin
package com.illareklab.fleet.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.illareklab.fleet.data.repository.AudioRepository
import com.illareklab.fleet.data.repository.GpsRepository
import com.illareklab.fleet.data.repository.MediaRepository
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.stateIn

data class SyncCounts(
    val gpsGoogle: Int = 0,
    val gpsSensors: Int = 0,
    val photos: Int = 0,
    val videos: Int = 0,
    val audios: Int = 0
) {
    val total: Int get() = gpsGoogle + gpsSensors + photos + videos + audios
}

class SyncViewModel(
    gpsRepository: GpsRepository,
    mediaRepository: MediaRepository,
    audioRepository: AudioRepository
) : ViewModel() {

    val counts = combine(
        gpsRepository.googleCount,
        gpsRepository.sensorsCount,
        mediaRepository.photoCount,
        mediaRepository.videoCount,
        audioRepository.count
    ) { g, s, p, v, a ->
        SyncCounts(g, s, p, v, a)
    }.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        SyncCounts()
    )

    class Factory(
        private val gps: GpsRepository,
        private val media: MediaRepository,
        private val audio: AudioRepository
    ) : ViewModelProvider.Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel> create(modelClass: Class<T>): T =
            SyncViewModel(gps, media, audio) as T
    }
}
```

### ¿Por qué `combine` y `stateIn`?

`combine` toma N flujos y emite un valor unificado cada vez que **cualquiera** cambia. La pantalla de Sync se reactualiza sola: agregas una foto en otra pestaña, regresas a Sync, el contador ya subió sin recargar.

`stateIn` convierte un `Flow` frío en un `StateFlow` caliente con un valor inicial, ideal para que Compose pueda renderizar inmediatamente con `collectAsStateWithLifecycle()`.

### 11.4 `MediaViewModel.kt` y `AudioViewModel.kt`

Estructura idéntica a `GpsViewModel`: exponen los flujos del repositorio, agregan métodos `suspend` para capturar/eliminar.

```kotlin
class MediaViewModel(
    private val mediaRepository: MediaRepository,
    private val fileStorage: FileStorageManager
) : ViewModel() {

    val mediaList = mediaRepository.allMedia.stateIn(
        viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList()
    )

    fun newPhotoFile() = fileStorage.newPhotoFile()
    fun newVideoFile() = fileStorage.newVideoFile()

    fun onPhotoCaptured(path: String, w: Int, h: Int) {
        viewModelScope.launch {
            mediaRepository.registerPhoto(path, w, h)
        }
    }

    fun onVideoCaptured(path: String, durationMs: Long) {
        viewModelScope.launch {
            mediaRepository.registerVideo(path, durationMs)
        }
    }

    fun delete(item: MediaEntity) {
        viewModelScope.launch { mediaRepository.delete(item) }
    }
    // Factory omitido por brevedad — sigue el mismo patrón
}
```

---

## 12. Servicio de captura GPS

`GpsCaptureService.kt` corre como Foreground Service y, cada 10 segundos, escribe en ambas tablas de Room.

```kotlin
package com.illareklab.fleet.services

import android.Manifest
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.Service
import android.content.Intent
import android.content.pm.PackageManager
import android.location.LocationManager
import android.os.IBinder
import androidx.core.app.NotificationCompat
import androidx.core.content.ContextCompat
import com.google.android.gms.location.FusedLocationProviderClient
import com.google.android.gms.location.LocationCallback
import com.google.android.gms.location.LocationRequest
import com.google.android.gms.location.LocationResult
import com.google.android.gms.location.LocationServices
import com.google.android.gms.location.Priority
import com.illareklab.fleet.FleetApp
import com.illareklab.fleet.data.local.entity.GpsGoogleEntity
import com.illareklab.fleet.data.local.entity.GpsSensorsEntity
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.launch

class GpsCaptureService : Service() {

    companion object {
        private const val CHANNEL_ID = "fleet_tracking"
        private const val NOTIF_ID = 100
        private const val INTERVAL_MS = 10_000L
    }

    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    private lateinit var fusedClient: FusedLocationProviderClient
    private lateinit var locationManager: LocationManager
    private val gpsRepo by lazy { (application as FleetApp).gpsRepository }

    private val flpCallback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            val loc = result.lastLocation ?: return
            scope.launch {
                gpsRepo.saveGooglePoint(
                    GpsGoogleEntity(
                        latitude = loc.latitude,
                        longitude = loc.longitude,
                        accuracy = loc.accuracy,
                        speed = if (loc.hasSpeed()) loc.speed else null,
                        bearing = if (loc.hasBearing()) loc.bearing else null,
                        timestamp = System.currentTimeMillis()
                    )
                )
            }
        }
    }

    private val rawListener = android.location.LocationListener { loc ->
        scope.launch {
            gpsRepo.saveSensorsPoint(
                GpsSensorsEntity(
                    latitude = loc.latitude,
                    longitude = loc.longitude,
                    provider = loc.provider ?: "unknown",
                    altitude = if (loc.hasAltitude()) loc.altitude else null,
                    timestamp = System.currentTimeMillis()
                )
            )
        }
    }

    override fun onCreate() {
        super.onCreate()
        createChannel()
        startForeground(NOTIF_ID, buildNotification())
        fusedClient = LocationServices.getFusedLocationProviderClient(this)
        locationManager = getSystemService(LOCATION_SERVICE) as LocationManager
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        if (!hasLocationPermission()) {
            stopSelf()
            return START_NOT_STICKY
        }

        try {
            // 1. FLP — fuente Google (filtrada y fusionada)
            val flpRequest = LocationRequest.Builder(
                Priority.PRIORITY_HIGH_ACCURACY, INTERVAL_MS
            ).build()
            fusedClient.requestLocationUpdates(flpRequest, flpCallback, mainLooper)

            // 2. LocationManager — fuente cruda de hardware
            locationManager.requestLocationUpdates(
                LocationManager.GPS_PROVIDER, INTERVAL_MS, 0f, rawListener
            )
        } catch (_: SecurityException) {
            stopSelf()
        }
        return START_STICKY
    }

    private fun hasLocationPermission() = ContextCompat.checkSelfPermission(
        this, Manifest.permission.ACCESS_FINE_LOCATION
    ) == PackageManager.PERMISSION_GRANTED

    private fun createChannel() {
        val channel = NotificationChannel(
            CHANNEL_ID, "Tracking de flota",
            NotificationManager.IMPORTANCE_LOW
        )
        (getSystemService(NOTIFICATION_SERVICE) as NotificationManager)
            .createNotificationChannel(channel)
    }

    private fun buildNotification() = NotificationCompat.Builder(this, CHANNEL_ID)
        .setContentTitle("Fleet: captura en curso")
        .setContentText("Grabando coordenadas cada 10 s…")
        .setSmallIcon(android.R.drawable.ic_menu_compass)
        .build()

    override fun onDestroy() {
        super.onDestroy()
        try {
            fusedClient.removeLocationUpdates(flpCallback)
            locationManager.removeUpdates(rawListener)
        } catch (_: Exception) {}
        scope.coroutineContext[Job]?.cancel()
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

---

## 13. Worker de notificación diferida

```kotlin
package com.illareklab.fleet.workers

import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import androidx.core.app.NotificationCompat
import androidx.work.Worker
import androidx.work.WorkerParameters

class DelayedNotificationWorker(
    context: Context,
    params: WorkerParameters
) : Worker(context, params) {

    companion object {
        const val INPUT_MESSAGE = "input_message"
        private const val CHANNEL_ID = "fleet_alerts"
    }

    override fun doWork(): Result {
        val message = inputData.getString(INPUT_MESSAGE) ?: "Mensaje vacío"

        val manager = applicationContext
            .getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager

        manager.createNotificationChannel(
            NotificationChannel(
                CHANNEL_ID, "Alertas Fleet",
                NotificationManager.IMPORTANCE_HIGH
            )
        )

        val notif = NotificationCompat.Builder(applicationContext, CHANNEL_ID)
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setContentTitle("Recordatorio Fleet")
            .setContentText(message)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .build()

        manager.notify(System.currentTimeMillis().toInt(), notif)
        return Result.success()
    }
}
```

---

## 14. Navegación basada en sesión

`Navigation.kt` observa el `isLoggedIn` del `SessionViewModel` y decide entre dos sub-grafos: `LoginScreen` o las 6 pestañas. La transición **automática** sucede cuando DataStore emite un cambio.

```kotlin
package com.illareklab.fleet.ui

import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.Logout
import androidx.compose.material.icons.filled.CameraAlt
import androidx.compose.material.icons.filled.CloudSync
import androidx.compose.material.icons.filled.LocationOn
import androidx.compose.material.icons.filled.Mic
import androidx.compose.material.icons.filled.Notifications
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.illareklab.fleet.FleetApp
import com.illareklab.fleet.ui.screens.*
import com.illareklab.fleet.ui.viewmodel.SessionViewModel

@Composable
fun Navigation() {
    val app = LocalContext.current.applicationContext as FleetApp
    val sessionVm: SessionViewModel = viewModel(
        factory = SessionViewModel.Factory(app.sessionManager)
    )

    val isLoggedIn by sessionVm.isLoggedIn.collectAsStateWithLifecycle()

    if (isLoggedIn) {
        MainScaffold(sessionVm)
    } else {
        LoginScreen(onSubmit = sessionVm::login)
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
private fun MainScaffold(sessionVm: SessionViewModel) {
    val nav = rememberNavController()
    var selected by remember { mutableIntStateOf(0) }
    val username by sessionVm.username.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Fleet — ${username ?: "?"}") },
                actions = {
                    IconButton(onClick = { sessionVm.logout() }) {
                        Icon(Icons.AutoMirrored.Filled.Logout, contentDescription = "Logout")
                    }
                }
            )
        },
        bottomBar = {
            NavigationBar {
                val tabs = listOf(
                    "gps" to (Icons.Default.LocationOn to "GPS"),
                    "media" to (Icons.Default.CameraAlt to "Multimedia"),
                    "audio" to (Icons.Default.Mic to "Audio"),
                    "sync" to (Icons.Default.CloudSync to "Sync"),
                    "notif" to (Icons.Default.Notifications to "Notif"),
                    "logout" to (Icons.AutoMirrored.Filled.Logout to "Salir")
                )
                tabs.forEachIndexed { idx, (route, iconLabel) ->
                    val (icon, label) = iconLabel
                    NavigationBarItem(
                        selected = selected == idx,
                        onClick = {
                            selected = idx
                            nav.navigate(route)
                        },
                        icon = { Icon(icon, contentDescription = null) },
                        label = { Text(label) }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(
            navController = nav,
            startDestination = "gps",
            modifier = Modifier.padding(padding)
        ) {
            composable("gps") { GpsScreen() }
            composable("media") { MediaScreen() }
            composable("audio") { AudioScreen() }
            composable("sync") { SyncScreen() }
            composable("notif") { NotificationsScreen() }
            composable("logout") { LogoutScreen(onLogout = sessionVm::logout) }
        }
    }
}
```

### El truco clave de la reactividad

Cuando `sessionVm.logout()` se llama (desde el botón de la pestaña 6 o desde la TopAppBar):

1. El método `suspend` actualiza DataStore.
2. DataStore emite un nuevo valor `false` por su `Flow`.
3. El `StateFlow` del ViewModel se actualiza.
4. `collectAsStateWithLifecycle()` en `Navigation` notifica a Compose.
5. Compose recompone, ve `isLoggedIn = false`, y muestra `LoginScreen` automáticamente.

**Sin navigation manual, sin `popBackStack()`, sin `Intent.FLAG_ACTIVITY_CLEAR_TASK`.** Solo flujo de datos. Esto es lo que hace la arquitectura tan limpia.

---

## 15. Estructura base de las pantallas

Esqueletos mínimos. Cada uno se completa con la UI específica (LazyColumn, TextField, FAB, etc.).

### 15.1 `LoginScreen.kt`

```kotlin
package com.illareklab.fleet.ui.screens

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.unit.dp

@Composable
fun LoginScreen(
    onSubmit: (username: String, password: String, onResult: (Boolean) -> Unit) -> Unit
) {
    var user by remember { mutableStateOf("") }
    var pass by remember { mutableStateOf("") }
    var error by remember { mutableStateOf("") }

    Column(
        modifier = Modifier.fillMaxSize().padding(32.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Fleet", style = MaterialTheme.typography.displayMedium)
        Spacer(Modifier.height(24.dp))
        OutlinedTextField(
            value = user, onValueChange = { user = it },
            label = { Text("Usuario") }, singleLine = true,
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = pass, onValueChange = { pass = it },
            label = { Text("Contraseña") }, singleLine = true,
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )
        if (error.isNotEmpty()) {
            Spacer(Modifier.height(8.dp))
            Text(error, color = MaterialTheme.colorScheme.error)
        }
        Spacer(Modifier.height(24.dp))
        Button(
            onClick = {
                error = ""
                onSubmit(user, pass) { ok ->
                    if (!ok) error = "Credenciales incorrectas"
                }
            },
            modifier = Modifier.fillMaxWidth().height(50.dp)
        ) { Text("Ingresar") }
    }
}
```

### 15.2 `GpsScreen.kt`

```kotlin
package com.illareklab.fleet.ui.screens

import android.content.Intent
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.illareklab.fleet.FleetApp
import com.illareklab.fleet.services.GpsCaptureService
import com.illareklab.fleet.ui.viewmodel.GpsViewModel

@Composable
fun GpsScreen() {
    val context = LocalContext.current
    val app = context.applicationContext as FleetApp
    val vm: GpsViewModel = viewModel(
        factory = GpsViewModel.Factory(app.gpsRepository)
    )

    val google by vm.googlePoints.collectAsStateWithLifecycle()
    val sensors by vm.sensorsPoints.collectAsStateWithLifecycle()

    var capturing by remember { mutableStateOf(false) }

    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        Button(
            onClick = {
                val intent = Intent(context, GpsCaptureService::class.java)
                if (!capturing) context.startForegroundService(intent)
                else context.stopService(intent)
                capturing = !capturing
            },
            modifier = Modifier.fillMaxWidth().height(50.dp)
        ) {
            Text(if (capturing) "Detener captura" else "Capturar coordenada (10 s)")
        }

        Spacer(Modifier.height(16.dp))

        Text("Google FLP: ${google.size} registros",
            style = MaterialTheme.typography.titleMedium)
        Text("Sensores: ${sensors.size} registros",
            style = MaterialTheme.typography.titleMedium)

        Spacer(Modifier.height(16.dp))

        // Combinamos los últimos N puntos de ambas tablas para mostrar
        LazyColumn(verticalArrangement = Arrangement.spacedBy(4.dp)) {
            items(google.take(10), key = { "g-${it.id}" }) { point ->
                Card { Text(
                    "[GOOGLE] ${point.latitude}, ${point.longitude}  ±${point.accuracy}m",
                    Modifier.padding(8.dp)
                ) }
            }
            items(sensors.take(10), key = { "s-${it.id}" }) { point ->
                Card { Text(
                    "[SENSORS:${point.provider}] ${point.latitude}, ${point.longitude}",
                    Modifier.padding(8.dp)
                ) }
            }
        }
    }
}
```

### 15.3 `MediaScreen.kt` — esqueleto

```kotlin
@Composable
fun MediaScreen() {
    val context = LocalContext.current
    val app = context.applicationContext as FleetApp
    val vm: MediaViewModel = viewModel(
        factory = MediaViewModel.Factory(app.mediaRepository, app.fileStorage)
    )
    val list by vm.mediaList.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Row {
            Button(onClick = { /* lanzar captura de foto con CameraX */ }) {
                Text("Foto")
            }
            Spacer(Modifier.width(8.dp))
            Button(onClick = { /* lanzar captura de video */ }) {
                Text("Video")
            }
        }
        Spacer(Modifier.height(16.dp))
        LazyColumn {
            items(list, key = { it.id }) { media ->
                ListItem(
                    headlineContent = { Text(media.type) },
                    supportingContent = {
                        Text("${media.sizeBytes / 1024} KB · ${media.filePath}")
                    },
                    leadingContent = {
                        AsyncImage(   // Coil
                            model = media.filePath,
                            contentDescription = null,
                            modifier = Modifier.size(48.dp)
                        )
                    }
                )
            }
        }
    }
}
```

### 15.4 `AudioScreen.kt` — esqueleto

```kotlin
@Composable
fun AudioScreen() {
    val app = LocalContext.current.applicationContext as FleetApp
    val vm: AudioViewModel = viewModel(
        factory = AudioViewModel.Factory(app.audioRepository, app.fileStorage)
    )
    val list by vm.audios.collectAsStateWithLifecycle()
    var recording by remember { mutableStateOf(false) }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Button(onClick = {
            if (recording) vm.stopRecording() else vm.startRecording()
            recording = !recording
        }) {
            Text(if (recording) "Detener" else "Grabar audio")
        }
        Spacer(Modifier.height(16.dp))
        LazyColumn {
            items(list, key = { it.id }) { audio ->
                ListItem(
                    headlineContent = { Text("${audio.durationMs / 1000} s") },
                    supportingContent = { Text(audio.filePath) }
                )
            }
        }
    }
}
```

### 15.5 `SyncScreen.kt` (con Toast "Por implementar")

```kotlin
package com.illareklab.fleet.ui.screens

import android.widget.Toast
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.illareklab.fleet.FleetApp
import com.illareklab.fleet.ui.viewmodel.SyncViewModel

@Composable
fun SyncScreen() {
    val context = LocalContext.current
    val app = context.applicationContext as FleetApp
    val vm: SyncViewModel = viewModel(
        factory = SyncViewModel.Factory(
            app.gpsRepository, app.mediaRepository, app.audioRepository
        )
    )
    val counts by vm.counts.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize().padding(24.dp)) {
        Button(
            onClick = {
                Toast.makeText(
                    context, "Por implementar", Toast.LENGTH_SHORT
                ).show()
            },
            modifier = Modifier.fillMaxWidth().height(50.dp)
        ) { Text("Sync") }

        Spacer(Modifier.height(24.dp))

        Text("Inventario local", style = MaterialTheme.typography.headlineSmall)
        Spacer(Modifier.height(12.dp))

        listOf(
            "GPS Google" to counts.gpsGoogle,
            "GPS Sensores" to counts.gpsSensors,
            "Fotos" to counts.photos,
            "Videos" to counts.videos,
            "Audios" to counts.audios
        ).forEach { (label, value) ->
            Card(modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                Row(
                    Modifier.padding(16.dp),
                    horizontalArrangement = Arrangement.SpaceBetween
                ) {
                    Text(label, style = MaterialTheme.typography.bodyLarge)
                    Text("$value", style = MaterialTheme.typography.titleLarge)
                }
            }
        }

        Spacer(Modifier.height(16.dp))
        Text(
            "Total de registros: ${counts.total}",
            style = MaterialTheme.typography.titleMedium
        )
    }
}
```

### 15.6 `NotificationsScreen.kt`

```kotlin
@Composable
fun NotificationsScreen() {
    val context = LocalContext.current
    var message by remember { mutableStateOf("") }
    var status by remember { mutableStateOf("") }

    Column(Modifier.fillMaxSize().padding(24.dp)) {
        OutlinedTextField(
            value = message, onValueChange = { message = it },
            label = { Text("Mensaje") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(Modifier.height(16.dp))
        Button(
            enabled = message.isNotBlank(),
            onClick = {
                val req = OneTimeWorkRequestBuilder<DelayedNotificationWorker>()
                    .setInitialDelay(10, TimeUnit.SECONDS)
                    .setInputData(
                        workDataOf(DelayedNotificationWorker.INPUT_MESSAGE to message)
                    )
                    .build()
                WorkManager.getInstance(context).enqueue(req)
                status = "Notificación programada (10 s). " +
                         "Funcionará incluso con la app cerrada."
            },
            modifier = Modifier.fillMaxWidth().height(50.dp)
        ) { Text("Programar notificación") }

        if (status.isNotEmpty()) {
            Spacer(Modifier.height(12.dp))
            Text(status, color = MaterialTheme.colorScheme.primary)
        }
    }
}
```

### 15.7 `LogoutScreen.kt`

```kotlin
@Composable
fun LogoutScreen(onLogout: () -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            "¿Cerrar la sesión actual?",
            style = MaterialTheme.typography.headlineSmall
        )
        Spacer(Modifier.height(24.dp))
        Button(
            onClick = onLogout,
            modifier = Modifier.fillMaxWidth().height(50.dp),
            colors = ButtonDefaults.buttonColors(
                containerColor = MaterialTheme.colorScheme.error
            )
        ) { Text("Cerrar sesión") }
    }
}
```

---

## 16. `MainActivity.kt`

```kotlin
package com.illareklab.fleet

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import com.illareklab.fleet.ui.Navigation
import com.illareklab.fleet.ui.theme.AppTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                Navigation()
            }
        }
    }
}
```

---

## 17. `AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Posicionamiento -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

    <!-- Captura multimedia -->
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />

    <!-- Servicios en primer plano + notificaciones -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <!-- Hardware declarado como opcional -->
    <uses-feature android:name="android.hardware.camera" android:required="false" />
    <uses-feature android:name="android.hardware.camera.autofocus" android:required="false" />
    <uses-feature android:name="android.hardware.microphone" android:required="false" />
    <uses-feature android:name="android.hardware.location.gps" android:required="false" />

    <application
        android:name=".FleetApp"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/Theme.Fleet">

        <service
            android:name=".services.GpsCaptureService"
            android:foregroundServiceType="location"
            android:exported="false" />

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

> Sin `INTERNET` ni `usesCleartextTraffic` porque el laboratorio es **100% offline**. Si más adelante se agrega un endpoint, son las únicas dos líneas que faltan.

---

## 18. Resumen del flujo de datos

### Escritura (capturar GPS)

```
Botón "Capturar coordenada"
    → startForegroundService(GpsCaptureService)
    → cada 10 s, el servicio recibe coordenadas:
        - FLP → GpsRepository.saveGooglePoint() → Room (tabla gps_google)
        - LocationManager → GpsRepository.saveSensorsPoint() → Room (tabla gps_sensors)
    → Room emite a través del Flow
    → GpsViewModel.googlePoints / sensorsPoints emiten nueva lista
    → GpsScreen recompone automáticamente con los nuevos puntos
```

### Lectura (Sync Center muestra conteos)

```
SyncScreen entra a la composición
    → SyncViewModel.counts combina 5 flujos (uno por tipo)
    → cada Flow viene de un .observeCount() del DAO
    → cualquier INSERT/DELETE en cualquier tabla dispara una nueva emisión
    → la UI muestra el conteo actualizado sin recargar
```

### Sesión (login → logout)

```
LoginScreen → onSubmit("admin", "admin")
    → SessionViewModel.login()
    → SessionManager.login() → DataStore.edit { is_logged_in = true }
    → DataStore emite por Flow
    → SessionViewModel.isLoggedIn pasa a true
    → Navigation recompone, muestra MainScaffold

LogoutScreen → "Cerrar sesión"
    → SessionViewModel.logout()
    → DataStore.edit { clear() }
    → emite false
    → Navigation recompone, muestra LoginScreen
```

Toda la coordinación es **reactiva**, sin observers manuales ni callbacks anidados.

---

## 19. Checklist de implementación

- [ ] Plugin KSP agregado en `build.gradle.kts`.
- [ ] Dependencias Room (3) y DataStore añadidas.
- [ ] 4 entidades creadas: `GpsGoogleEntity`, `GpsSensorsEntity`, `MediaEntity`, `AudioEntity`.
- [ ] 4 DAOs creados con métodos `Insert`, `observeAll`, `observeCount`, `delete`.
- [ ] `FleetDatabase` con singleton thread-safe.
- [ ] `SessionManager` exponiendo `isLoggedIn: Flow<Boolean>` desde DataStore.
- [ ] `FileStorageManager` con `filesDir/photos`, `videos`, `audios`.
- [ ] 3 repositorios coordinando Room + filesDir.
- [ ] `FleetApp : Application` registrada en el manifest, con DI por `by lazy`.
- [ ] 5 ViewModels con su Factory correspondiente.
- [ ] `GpsCaptureService` como Foreground Service tipo `location`.
- [ ] `DelayedNotificationWorker` extendiendo `Worker`.
- [ ] `Navigation` con switch reactivo basado en `isLoggedIn`.
- [ ] 7 pantallas: Login + 6 pestañas (GPS, Multimedia, Audio, Sync, Notificaciones, Logout).
- [ ] `MainActivity` usando `AppTheme`.
- [ ] Manifest con permisos + service registrado + `<uses-feature>`.
- [ ] App compila y ejecuta sin errores.
- [ ] Login persiste tras reiniciar la app.
- [ ] Logout desde TopAppBar O desde pestaña 6 vuelve al login.
- [ ] Captura de GPS escribe en ambas tablas, visible en SyncScreen.
- [ ] Sync Center refleja conteos actualizados en tiempo real.
- [ ] Notificación llega a los 10 s aunque la app esté cerrada.

---

## 20. Decisiones arquitectónicas — resumen

| Decisión | Justificación |
|---|---|
| **Room en vez de archivos planos** | Consultas SQL, observabilidad reactiva con Flow, integridad referencial, migraciones versionadas. Es la diferencia entre "guardar datos" y "tener una base de datos". |
| **DataStore en vez de SharedPreferences** | API asíncrona, Flow reactivo, atomicidad de transacciones, recomendado oficialmente desde Android 11. |
| **`filesDir` en vez de `getExternalFilesDir()`** | Privacidad total: ninguna otra app puede acceder. No requiere permisos de almacenamiento. Limpieza automática al desinstalar. |
| **Repositorios entre ViewModel y DAO** | Permite cambiar la implementación (memoria, archivo, red) sin tocar la UI. Facilita testing. |
| **DI manual vía `Application`** | Visibilidad pedagógica del grafo de dependencias. Para producción con equipos grandes, migrar a **Hilt**. |
| **Navegación reactiva al `isLoggedIn`** | Elimina la necesidad de gestionar manualmente el back stack tras login/logout. La UI sigue al estado, no al revés. |
| **`Flow` en DAOs, `StateFlow` en ViewModels** | Flow = stream frío, eficiente para BD. StateFlow = stream caliente con valor inicial, ideal para Compose. `stateIn(viewModelScope, ...)` hace la conversión idiomáticamente. |
| **`SharingStarted.WhileSubscribed(5000)`** | Mantiene el Flow activo 5 segundos después de que la UI deja de observar, evitando re-suscripciones costosas durante rotaciones de pantalla. |
| **Una sola Activity** | Toda la navegación entre pantallas es de Compose. Activities adicionales son innecesarias y traen complicaciones de ciclo de vida. |
| **Foreground Service para GPS** | Garantiza ejecución continua incluso cuando el usuario navega a otras pestañas o pone la app en segundo plano. |
| **WorkManager para notificaciones diferidas** | Sobrevive al cierre de la app. Un `Handler.postDelayed` muere con el proceso. |

---

## 21. Próximos pasos (cuando agregues backend)

Cuando llegue el momento de conectar con un servidor:

1. **Agregar `NetworkManager`** con Retrofit + OkHttp.
2. **Agregar `INTERNET` al manifest** y configurar `usesCleartextTraffic` si el backend es HTTP.
3. **Extender los repositorios** con un método `syncToServer()` que: lea Room, haga POST por bloques, marque registros como sincronizados (agregar columna `is_synced: Boolean` a cada entidad).
4. **Migrar la BD a v2** con `Migration(1, 2)` añadiendo la columna.
5. **Reemplazar el Toast** del `SyncScreen` por la lógica real de sincronización (ya tienes los conteos listos).
6. **Considerar** `WorkManager` periódico (cada 15 min) para sync en background.

La arquitectura actual no necesita refactorizarse para soportar esto: los repositorios son la abstracción correcta.
