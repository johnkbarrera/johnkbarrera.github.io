# Lab 5 — Multimedia, Audio, Login y Notificaciones

**Autor:** [illarek-lab](https://github.com/illarek-lab)
**Proyecto:** DemoData · `com.illareklab.demodata`
**Temas:** L1 (ViewModelProvider.Factory, sealed class) · L2 (Camera, ActivityResultContracts, FileProvider) · L3 (MediaRecorder, WorkManager, PasswordHasher)

> **Objetivo del laboratorio:** Extender la app DemoData con captura de fotos y videos (CameraX intent + FileProvider), grabación de audio (MediaRecorder), notificaciones programadas (WorkManager) y una pantalla de historial unificado que combina todos los tipos de datos capturados. El ciclo GNSS ya implementado en el Lab 4 se reutiliza sin cambios.

---

## 1. Arquitectura del laboratorio

### Árbol de archivos del proyecto

```
app/src/main/
├── AndroidManifest.xml
├── res/xml/
│   └── file_paths.xml
└── java/com/illareklab/demodata/
    │
    ├── DemoDataApp.kt          (FleetApp.kt)   ← actualizado Lab 5
    ├── MainActivity.kt                          ← actualizado Lab 5
    │
    ├── data/
    │   ├── local/
    │   │   ├── DemoDataDatabase.kt              ← actualizado Lab 5
    │   │   ├── FileStorageManager.kt            ← nuevo Lab 5
    │   │   ├── dao/
    │   │   │   ├── GpsGoogleDao.kt              (Lab 4)
    │   │   │   ├── GpsSensorsDao.kt             (Lab 4)
    │   │   │   ├── MediaDao.kt                  ← nuevo Lab 5
    │   │   │   └── AudioDao.kt                  ← nuevo Lab 5
    │   │   └── entity/
    │   │       ├── GpsGoogleEntity.kt           (Lab 4)
    │   │       ├── GpsSensorsEntity.kt          (Lab 4)
    │   │       ├── MediaEntity.kt               ← nuevo Lab 5
    │   │       └── AudioEntity.kt               ← nuevo Lab 5
    │   ├── repository/
    │   │   ├── GpsRepository.kt                 (Lab 4)
    │   │   ├── MediaRepository.kt               ← nuevo Lab 5
    │   │   └── AudioRepository.kt               ← nuevo Lab 5
    │   └── session/
    │       └── SessionManager.kt                ← actualizado Lab 5
    │
    ├── security/
    │   └── PasswordHasher.kt                    ← nuevo Lab 5
    │
    ├── services/
    │   └── GpsCaptureService.kt                 (Lab 4)
    │
    ├── util/
    │   └── AudioRecorderManager.kt              ← nuevo Lab 5
    │
    ├── workers/
    │   └── DelayedNotificationWorker.kt         ← nuevo Lab 5
    │
    └── ui/
        ├── Navigation.kt                        ← actualizado Lab 5
        ├── screens/
        │   ├── GpsScreen.kt                     (Lab 4)
        │   ├── LoginScreen.kt                   ← nuevo Lab 5
        │   ├── MediaScreen.kt                   ← nuevo Lab 5
        │   ├── AudioScreen.kt                   ← nuevo Lab 5
        │   ├── NotificationsScreen.kt           ← nuevo Lab 5
        │   ├── SyncScreen.kt                    ← nuevo Lab 5
        │   └── ProfileScreen.kt                 ← actualizado Lab 5
        ├── theme/
        │   ├── Color.kt
        │   ├── Theme.kt
        │   └── Type.kt
        └── viewmodel/
            ├── GpsViewModel.kt                  (Lab 4)
            ├── SessionViewModel.kt              ← actualizado Lab 5
            ├── MediaViewModel.kt                ← nuevo Lab 5
            ├── AudioViewModel.kt                ← nuevo Lab 5
            └── SyncViewModel.kt                 ← nuevo Lab 5
```

### Diagrama de flujo de datos

```
DemoDataApp
  ├── DemoDataDatabase (fleet.db, version 3)
  │     ├── gps_google, gps_sensors   (Lab 4)
  │     ├── media                     (Lab 5)
  │     └── audio                     (Lab 5)
  ├── FileStorageManager
  │     ├── filesDir/photos/
  │     ├── filesDir/videos/
  │     └── filesDir/audios/
  ├── GpsRepository   (Lab 4)
  ├── MediaRepository ──→ MediaViewModel ──→ MediaScreen
  ├── AudioRepository ──→ AudioViewModel ──→ AudioScreen
  └── SessionManager  ──→ SessionViewModel ──→ LoginScreen / Navigation

GpsCaptureService     (Lab 4)

DelayedNotificationWorker (WorkManager)
  └── NotificationsScreen ──→ WorkManager.enqueue(delay=10s) ──→ Worker.doWork()

SyncViewModel
  └── combine(gpsGoogle, gpsSensors, photos, videos, audios) ──→ SyncCounts
        └── SyncScreen

ProfileScreen.MyActivityScreen
  └── combine(googlePoints, sensorsPoints, allMedia, allAudios)
        └── sealed class ActivityItem ──→ LazyColumn + ActivityDetailDialog
                                             └── FileProvider ──→ Intent.ACTION_VIEW
```

---

## 2. Nuevas dependencias (`app/build.gradle.kts`)

Las dependencias del Lab 4 (Room, DataStore, Compose, Accompanist, Coil, FLP) permanecen iguales. Las nuevas para el Lab 5:

```kotlin
dependencies {
    // ── WorkManager (notificaciones diferidas, ejecución en background) ──
    implementation("androidx.work:work-runtime-ktx:2.9.0")
}
```

> **Nota:** `work-runtime-ktx` incluye las extensiones de coroutines para Workers. Si se necesita `CoroutineWorker` en lugar de `Worker`, basta con cambiar la clase base — sin dependencias adicionales.

---

## 3. `AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- ── Hardware de posicionamiento ── -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />

    <!-- ── Hardware inalámbrico complementario ── -->
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />

    <!-- ── Periféricos de captura multimedia ── -->
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />

    <!-- ── Servicio en primer plano + notificaciones (Android 13+) ── -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <!-- ── Conectividad de red para HTTP POST ── -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <!-- ── Declaración explícita de hardware ── -->
    <uses-feature android:name="android.hardware.camera"            android:required="false" />
    <uses-feature android:name="android.hardware.camera.autofocus"  android:required="false" />
    <uses-feature android:name="android.hardware.microphone"        android:required="false" />
    <uses-feature android:name="android.hardware.location"          android:required="false" />
    <uses-feature android:name="android.hardware.location.gps"      android:required="false" />
    <uses-feature android:name="android.hardware.bluetooth_le"      android:required="false" />
    <uses-feature android:name="android.hardware.wifi"              android:required="false" />

    <application
        android:name=".DemoDataApp"
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@drawable/data_icon"
        android:label="@string/app_name"
        android:roundIcon="@drawable/data_icon"
        android:supportsRtl="true"
        android:theme="@style/Theme.DemoData">

        <service
            android:name=".services.GpsCaptureService"
            android:foregroundServiceType="location"
            android:exported="false" />

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

        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>

    </application>

</manifest>
```

### Novedades respecto al Lab 4

| Elemento | Qué hace |
|---|---|
| `CAMERA` | Permite abrir la cámara del sistema |
| `RECORD_AUDIO` | Permite usar el micrófono con `MediaRecorder` |
| `INTERNET` + `ACCESS_NETWORK_STATE` | Preparado para el módulo de sincronización HTTP |
| `<uses-feature required="false">` | Declara hardware deseado sin hacerlo obligatorio — la app puede instalarse en dispositivos sin cámara |
| `<provider FileProvider>` | Expone archivos privados de `filesDir` con URIs `content://` que el sistema de cámara puede escribir |

### `res/xml/file_paths.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <files-path name="my_images"  path="photos/" />
    <files-path name="my_videos"  path="videos/" />
    <files-path name="my_audios"  path="audios/" />
</paths>
```

`<files-path>` mapea subdirectorios dentro de `context.filesDir`. El atributo `name` es el segmento que aparece en el URI `content://`; `path` es la ruta relativa dentro de `filesDir`. Sin esta declaración, `FileProvider.getUriForFile()` lanzaría `IllegalArgumentException`.

---

## 4. Componentes cubiertos en este lab

| Componente | Archivo | Nuevo/Actualizado |
|---|---|---|
| Clase Application | `DemoDataApp.kt` (FleetApp.kt) | Actualizado |
| Base de datos Room | `data/local/DemoDataDatabase.kt` | Actualizado (v3) |
| Gestor de archivos | `data/local/FileStorageManager.kt` | Nuevo |
| Entidad Media | `data/local/entity/MediaEntity.kt` | Nuevo |
| Entidad Audio | `data/local/entity/AudioEntity.kt` | Nuevo |
| DAO Media | `data/local/dao/MediaDao.kt` | Nuevo |
| DAO Audio | `data/local/dao/AudioDao.kt` | Nuevo |
| Repositorio Media | `data/repository/MediaRepository.kt` | Nuevo |
| Repositorio Audio | `data/repository/AudioRepository.kt` | Nuevo |
| Hasher de contraseñas | `security/PasswordHasher.kt` | Nuevo |
| Gestor de grabación | `util/AudioRecorderManager.kt` | Nuevo |
| Worker de notificaciones | `workers/DelayedNotificationWorker.kt` | Nuevo |
| SessionManager | `data/session/SessionManager.kt` | Actualizado |
| SessionViewModel | `ui/viewmodel/SessionViewModel.kt` | Actualizado |
| MediaViewModel | `ui/viewmodel/MediaViewModel.kt` | Nuevo |
| AudioViewModel | `ui/viewmodel/AudioViewModel.kt` | Nuevo |
| SyncViewModel | `ui/viewmodel/SyncViewModel.kt` | Nuevo |
| Navegación | `ui/Navigation.kt` | Actualizado (6 tabs + login gate) |
| Pantalla Login | `ui/screens/LoginScreen.kt` | Nuevo |
| Pantalla Multimedia | `ui/screens/MediaScreen.kt` | Nuevo |
| Pantalla Audio | `ui/screens/AudioScreen.kt` | Nuevo |
| Pantalla Notificaciones | `ui/screens/NotificationsScreen.kt` | Nuevo |
| Pantalla Sync | `ui/screens/SyncScreen.kt` | Nuevo |
| Pantalla Perfil | `ui/screens/ProfileScreen.kt` | Actualizado |
| Actividad principal | `MainActivity.kt` | Actualizado |
| Rutas FileProvider | `res/xml/file_paths.xml` | Nuevo |

---

## 5. `DemoDataApp.kt` — Clase Application actualizada

```kotlin
package com.illareklab.demodata

import android.app.Application
import com.illareklab.demodata.data.local.FileStorageManager
import com.illareklab.demodata.data.local.DemoDataDatabase
import com.illareklab.demodata.data.repository.AudioRepository
import com.illareklab.demodata.data.repository.GpsRepository
import com.illareklab.demodata.data.repository.MediaRepository
import com.illareklab.demodata.data.session.SessionManager

class DemoDataApp : Application() {

    val database     by lazy { DemoDataDatabase.getInstance(this) }
    val fileStorage  by lazy { FileStorageManager(this) }
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

Respecto al Lab 4, se agregan `fileStorage`, `mediaRepository` y `audioRepository`. El `FileStorageManager` se construye antes que los repositorios porque `MediaRepository` y `AudioRepository` lo reciben como dependencia para poder borrar archivos del disco al borrar un registro de Room.

---

## 6. `DemoDataDatabase.kt` — BD actualizada a versión 3

```kotlin
package com.illareklab.demodata.data.local

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import com.illareklab.demodata.data.local.dao.AudioDao
import com.illareklab.demodata.data.local.dao.GpsGoogleDao
import com.illareklab.demodata.data.local.dao.GpsSensorsDao
import com.illareklab.demodata.data.local.dao.MediaDao
import com.illareklab.demodata.data.local.entity.AudioEntity
import com.illareklab.demodata.data.local.entity.GpsGoogleEntity
import com.illareklab.demodata.data.local.entity.GpsSensorsEntity
import com.illareklab.demodata.data.local.entity.MediaEntity

@Database(
    entities = [
        GpsGoogleEntity::class,
        GpsSensorsEntity::class,
        MediaEntity::class,
        AudioEntity::class
    ],
    version = 3,
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
                ).fallbackToDestructiveMigration().build().also { INSTANCE = it }
            }
    }
}
```

**`fallbackToDestructiveMigration()`:** durante el desarrollo, cuando la versión de la BD sube de 2 → 3, Room destruye y recrea todas las tablas en lugar de intentar una migración. En producción esto significaría pérdida de datos, pero es aceptable en el contexto del laboratorio para evitar escribir scripts `Migration`.

---

## 7. `FileStorageManager.kt` — Gestión de archivos privados

```kotlin
package com.illareklab.demodata.data.local

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

    fun deleteFile(path: String): Boolean =
        File(path).takeIf { it.exists() }?.delete() ?: false

    fun fileSize(path: String): Long = File(path).length()
}
```

**¿Por qué `filesDir` en vez de `getExternalFilesDir`?**

`filesDir` es privado al proceso de la app: no requiere permisos de almacenamiento (eliminados en Android 10+), no es accesible por otras apps directamente, y se limpia automáticamente al desinstalar. `FileProvider` convierte estas rutas privadas en URIs `content://` para compartirlas temporalmente con la cámara del sistema.

**`apply { mkdirs() }`:** crea el directorio en el momento en que la propiedad se inicializa (que es la primera vez que se usa `FileStorageManager`). Si el directorio ya existe, `mkdirs()` es una operación sin costo.

---

## 8. Capa de datos — Media (Foto y Video)

### 8.1 `MediaEntity.kt`

```kotlin
package com.illareklab.demodata.data.local.entity

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

`type` guarda el nombre del enum (`MediaType.name`) en lugar del ordinal para que la BD sea legible y sobreviva reordenamientos del enum.

### 8.2 `MediaDao.kt`

```kotlin
package com.illareklab.demodata.data.local.dao

import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.Query
import com.illareklab.demodata.data.local.entity.MediaEntity
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

`observeByType` recibe un `String` en lugar de `MediaType` para evitar que Room necesite un `TypeConverter`. La pantalla puede pasar `MediaType.PHOTO.name` directamente.

### 8.3 `MediaRepository.kt`

```kotlin
package com.illareklab.demodata.data.repository

import com.illareklab.demodata.data.local.FileStorageManager
import com.illareklab.demodata.data.local.dao.MediaDao
import com.illareklab.demodata.data.local.entity.MediaEntity
import com.illareklab.demodata.data.local.entity.MediaType
import kotlinx.coroutines.flow.Flow

class MediaRepository(
    private val mediaDao: MediaDao,
    private val fileStorage: FileStorageManager
) {
    val allMedia: Flow<List<MediaEntity>> = mediaDao.observeAll()
    val photoCount: Flow<Int> = mediaDao.observePhotoCount()
    val videoCount: Flow<Int> = mediaDao.observeVideoCount()

    suspend fun registerPhoto(
        filePath: String,
        widthPx: Int,
        heightPx: Int
    ): Long = mediaDao.insert(
        MediaEntity(
            filePath  = filePath,
            type      = MediaType.PHOTO.name,
            sizeBytes = fileStorage.fileSize(filePath),
            widthPx   = widthPx,
            heightPx  = heightPx,
            timestamp = System.currentTimeMillis()
        )
    )

    suspend fun registerVideo(
        filePath: String,
        durationMs: Long
    ): Long = mediaDao.insert(
        MediaEntity(
            filePath   = filePath,
            type       = MediaType.VIDEO.name,
            sizeBytes  = fileStorage.fileSize(filePath),
            durationMs = durationMs,
            timestamp  = System.currentTimeMillis()
        )
    )

    suspend fun delete(item: MediaEntity) {
        fileStorage.deleteFile(item.filePath)
        mediaDao.delete(item)
    }
}
```

`delete` borra primero el archivo físico y luego el registro en Room. Si se invirtiera el orden y la app crasheara entre ambas operaciones, quedaría un registro apuntando a un archivo inexistente.

### 8.4 `MediaViewModel.kt`

```kotlin
package com.illareklab.demodata.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.illareklab.demodata.data.local.FileStorageManager
import com.illareklab.demodata.data.local.entity.MediaEntity
import com.illareklab.demodata.data.repository.MediaRepository
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch
import java.io.File

class MediaViewModel(
    private val mediaRepository: MediaRepository,
    private val fileStorage: FileStorageManager
) : ViewModel() {

    val mediaList = mediaRepository.allMedia.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        emptyList()
    )

    val photoCount = mediaRepository.photoCount.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        0
    )

    val videoCount = mediaRepository.videoCount.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        0
    )

    fun newPhotoFile(): File = fileStorage.newPhotoFile()
    fun newVideoFile(): File = fileStorage.newVideoFile()

    fun onPhotoCaptured(filePath: String, widthPx: Int, heightPx: Int) {
        viewModelScope.launch {
            mediaRepository.registerPhoto(filePath, widthPx, heightPx)
        }
    }

    fun onVideoCaptured(filePath: String, durationMs: Long) {
        viewModelScope.launch {
            mediaRepository.registerVideo(filePath, durationMs)
        }
    }

    fun delete(item: MediaEntity) {
        viewModelScope.launch {
            mediaRepository.delete(item)
        }
    }

    class Factory(
        private val mediaRepository: MediaRepository,
        private val fileStorage: FileStorageManager
    ) : ViewModelProvider.Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel> create(modelClass: Class<T>): T =
            MediaViewModel(mediaRepository, fileStorage) as T
    }
}
```

**`ViewModelProvider.Factory`:** cuando un ViewModel necesita parámetros en el constructor que no puede obtener solo (como un repositorio), se usa una `Factory` que sabe cómo construirlo. La UI llama a `viewModel(factory = MediaViewModel.Factory(...))` y el sistema se encarga del ciclo de vida.

---

## 9. Capa de datos — Audio

### 9.1 `AudioEntity.kt`

```kotlin
package com.illareklab.demodata.data.local.entity

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

### 9.2 `AudioDao.kt`

```kotlin
package com.illareklab.demodata.data.local.dao

import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.Query
import com.illareklab.demodata.data.local.entity.AudioEntity
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

### 9.3 `AudioRepository.kt`

```kotlin
package com.illareklab.demodata.data.repository

import com.illareklab.demodata.data.local.FileStorageManager
import com.illareklab.demodata.data.local.dao.AudioDao
import com.illareklab.demodata.data.local.entity.AudioEntity
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
            filePath   = filePath,
            durationMs = durationMs,
            sizeBytes  = fileStorage.fileSize(filePath),
            format     = format,
            timestamp  = System.currentTimeMillis()
        )
    )

    suspend fun delete(item: AudioEntity) {
        fileStorage.deleteFile(item.filePath)
        audioDao.delete(item)
    }
}
```

### 9.4 `AudioRecorderManager.kt` — Adaptador de bajo nivel

```kotlin
package com.illareklab.demodata.util

import android.content.Context
import android.media.MediaMetadataRetriever
import android.media.MediaRecorder
import android.os.Build
import java.io.File
import java.io.IOException

class AudioRecorderManager(private val context: Context) {

    private var recorder: MediaRecorder? = null

    fun createAudioFile(): File {
        val audioDir = File(context.filesDir, "audios").apply {
            if (!exists()) mkdirs()
        }
        return File(audioDir, "AUDIO_${System.currentTimeMillis()}.m4a")
    }

    fun start(outputFile: File) {
        stop() // Evita fugas si ya había una grabación activa

        val createRecorder = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            MediaRecorder(context)
        } else {
            @Suppress("DEPRECATION")
            MediaRecorder()
        }

        recorder = createRecorder.apply {
            setAudioSource(MediaRecorder.AudioSource.MIC)
            setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
            setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
            setOutputFile(outputFile.absolutePath)
            try {
                prepare()
                start()
            } catch (e: IOException) {
                e.printStackTrace()
            } catch (e: IllegalStateException) {
                e.printStackTrace()
            }
        }
    }

    fun stop() {
        try {
            recorder?.apply {
                stop()
                release()
            }
        } catch (e: Exception) {
            e.printStackTrace()
        } finally {
            recorder = null
        }
    }

    fun getDuration(file: File): Long {
        if (!file.exists()) return 0L
        val retriever = MediaMetadataRetriever()
        return try {
            retriever.setDataSource(file.absolutePath)
            val durationStr = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_DURATION)
            durationStr?.toLong() ?: 0L
        } catch (e: Exception) {
            0L
        } finally {
            retriever.release()
        }
    }
}
```

**`MediaRecorder(context)` vs `MediaRecorder()`:** desde Android 12 (API 31) el constructor sin parámetros está deprecado — requiere el `Context` para asociar el grabador al proceso. El bloque `if (Build.VERSION.SDK_INT >= S)` mantiene compatibilidad con Android 11 y anteriores.

### 9.5 `AudioViewModel.kt`

```kotlin
package com.illareklab.demodata.ui.viewmodel

import android.content.Context
import android.media.MediaRecorder
import android.os.Build
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.illareklab.demodata.data.local.FileStorageManager
import com.illareklab.demodata.data.local.entity.AudioEntity
import com.illareklab.demodata.data.repository.AudioRepository
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch
import java.io.File

class AudioViewModel(
    private val context: Context,
    private val audioRepository: AudioRepository,
    private val fileStorage: FileStorageManager
) : ViewModel() {

    val audios = audioRepository.allAudios.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        emptyList()
    )

    val count = audioRepository.count.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        0
    )

    private val _isRecording    = MutableStateFlow(false)
    val isRecording = _isRecording.asStateFlow()

    private val _elapsedSeconds = MutableStateFlow(0)
    val elapsedSeconds = _elapsedSeconds.asStateFlow()

    private var recorder: MediaRecorder? = null
    private var currentFile: File? = null
    private var startTimeMs: Long = 0L
    private var timerJob: Job? = null

    fun startRecording(): Boolean {
        if (_isRecording.value) return false

        return try {
            val file = fileStorage.newAudioFile(extension = "m4a")
            currentFile = file

            val newRecorder = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
                MediaRecorder(context)
            } else {
                @Suppress("DEPRECATION")
                MediaRecorder()
            }

            newRecorder.apply {
                setAudioSource(MediaRecorder.AudioSource.MIC)
                setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
                setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
                setOutputFile(file.absolutePath)
                prepare()
                start()
            }
            recorder    = newRecorder
            startTimeMs = System.currentTimeMillis()
            _isRecording.value    = true
            _elapsedSeconds.value = 0

            timerJob = viewModelScope.launch {
                while (_isRecording.value) {
                    delay(1000)
                    _elapsedSeconds.value++
                }
            }
            true
        } catch (e: Exception) {
            cleanup()
            false
        }
    }

    fun stopRecording() {
        if (!_isRecording.value) return

        val file       = currentFile
        val durationMs = System.currentTimeMillis() - startTimeMs

        try {
            recorder?.apply {
                stop()
                release()
            }
            if (file != null && file.exists() && durationMs >= 1000L) {
                viewModelScope.launch {
                    audioRepository.registerAudio(
                        filePath   = file.absolutePath,
                        durationMs = durationMs,
                        format     = "AAC"
                    )
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

    fun delete(item: AudioEntity) {
        viewModelScope.launch { audioRepository.delete(item) }
    }

    private fun cleanup() {
        timerJob?.cancel()
        timerJob      = null
        recorder      = null
        currentFile   = null
        _isRecording.value    = false
        _elapsedSeconds.value = 0
    }

    override fun onCleared() {
        super.onCleared()
        if (_isRecording.value) {
            try {
                recorder?.apply { stop(); release() }
            } catch (_: Exception) { }
            currentFile?.takeIf { it.exists() }?.delete()
        }
    }

    class Factory(
        private val context: Context,
        private val audioRepository: AudioRepository,
        private val fileStorage: FileStorageManager
    ) : ViewModelProvider.Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel> create(modelClass: Class<T>): T =
            AudioViewModel(context, audioRepository, fileStorage) as T
    }
}
```

**Puntos clave:**

- El timer (`timerJob`) corre dentro del `viewModelScope` con un loop `while (_isRecording.value)`. Se cancela automáticamente cuando se llama `cleanup()`.
- `stopRecording` descarta grabaciones menores de 1 segundo para evitar archivos corruptos que harían fallar el codec AAC.
- `onCleared()` libera el micrófono si el ViewModel se destruye mientras graba (por ejemplo, al quitar la app de recientes).

---

## 10. Login y Sesión

### 10.1 `PasswordHasher.kt`

```kotlin
package com.illareklab.demodata.security

import javax.crypto.SecretKeyFactory
import javax.crypto.spec.PBEKeySpec

object PasswordHasher {

    private const val ALGORITMO           = "PBKDF2WithHmacSHA256"
    private const val ITERACIONES         = 120_000
    private const val LONGITUD_HASH_BITS  = 256

    fun hash(password: String, salt: ByteArray): String {
        val spec = PBEKeySpec(
            password.toCharArray(),
            salt,
            ITERACIONES,
            LONGITUD_HASH_BITS
        )
        val factory = SecretKeyFactory.getInstance(ALGORITMO)
        val bytes   = factory.generateSecret(spec).encoded

        spec.clearPassword()  // Borrado defensivo de la contraseña en memoria

        return bytes.joinToString("") { "%02x".format(it) }
    }

    fun constantTimeEquals(a: String, b: String): Boolean {
        if (a.length != b.length) return false
        var diff = 0
        for (i in a.indices) {
            diff = diff or (a[i].code xor b[i].code)
        }
        return diff == 0
    }
}
```

**PBKDF2WithHmacSHA256 — por qué 120 000 iteraciones:**

Cada iteración re-aplica HMAC-SHA256 sobre el resultado anterior. Con 120 000 iteraciones (recomendación OWASP 2023), un atacante que obtenga el hash necesita 120 000 hashes SHA-256 por cada intento de fuerza bruta, elevando el costo computacional en 5 órdenes de magnitud respecto a un simple SHA-256.

**Timing attack y `constantTimeEquals`:** una comparación `a == b` en Java/Kotlin se detiene en el primer carácter diferente — un atacante puede deducir cuántos caracteres coinciden midiendo la latencia. `constantTimeEquals` siempre itera todos los caracteres, haciendo el tiempo de respuesta constante e independiente del contenido.

> **En este laboratorio:** la validación usa credenciales fijas `jkn/jkn` sin salt ni BD de usuarios, lo que ilustra el flujo sin complejidad de infraestructura. `PasswordHasher` está incluido como referencia de buenas prácticas para cuando se conecte un backend real.

### 10.2 `SessionManager.kt`

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

private val Context.sessionDataStore: DataStore<Preferences> by preferencesDataStore(
    name = "fleet_session"
)

class SessionManager(private val context: Context) {

    private companion object {
        val KEY_IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
        val KEY_USERNAME     = stringPreferencesKey("username")
        val KEY_DARK_MODE    = booleanPreferencesKey("dark_mode")
    }

    val isLoggedIn: Flow<Boolean> = context.sessionDataStore.data
        .map { prefs -> prefs[KEY_IS_LOGGED_IN] ?: false }

    val currentUsername: Flow<String?> = context.sessionDataStore.data
        .map { prefs -> prefs[KEY_USERNAME] }

    val isDarkMode: Flow<Boolean?> = context.sessionDataStore.data
        .map { prefs -> prefs[KEY_DARK_MODE] }

    suspend fun login(username: String) {
        context.sessionDataStore.edit { prefs ->
            prefs[KEY_IS_LOGGED_IN] = true
            prefs[KEY_USERNAME]     = username
        }
    }

    suspend fun setDarkMode(enabled: Boolean) {
        context.sessionDataStore.edit { prefs ->
            prefs[KEY_DARK_MODE] = enabled
        }
    }

    suspend fun logout() {
        context.sessionDataStore.edit { prefs ->
            val currentTheme = prefs[KEY_DARK_MODE]
            prefs.clear()
            if (currentTheme != null) prefs[KEY_DARK_MODE] = currentTheme
        }
    }
}
```

El DataStore se llama `"fleet_session"` en lugar de `"session_prefs"` del Lab 4 — son instancias distintas e independientes en disco.

### 10.3 `SessionViewModel.kt` — con Factory y `login()`

```kotlin
package com.illareklab.demodata.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.illareklab.demodata.data.session.SessionManager
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

class SessionViewModel(
    private val sessionManager: SessionManager
) : ViewModel() {

    val isLoggedIn = sessionManager.isLoggedIn.stateIn(
        scope        = viewModelScope,
        started      = SharingStarted.Eagerly,
        initialValue = false
    )

    val username = sessionManager.currentUsername.stateIn(
        scope        = viewModelScope,
        started      = SharingStarted.Eagerly,
        initialValue = null
    )

    val isDarkMode = sessionManager.isDarkMode.stateIn(
        scope        = viewModelScope,
        started      = SharingStarted.Eagerly,
        initialValue = null
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

    class Factory(private val sessionManager: SessionManager) : ViewModelProvider.Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel> create(modelClass: Class<T>): T =
            SessionViewModel(sessionManager) as T
    }
}
```

**`login(onResult: (Boolean) -> Unit)`:** el callback devuelve el resultado en el hilo principal cuando la coroutine termina. `LoginScreen` lo usa para mostrar el error de credenciales incorrectas sin bloquear la UI.

**`SharingStarted.Eagerly`:** a diferencia de `WhileSubscribed`, Eagerly inicia la recolección del Flow inmediatamente cuando el ViewModel se crea, incluso si no hay suscriptores. Correcto para el estado de sesión porque necesita estar listo antes del primer frame.

### 10.4 `LoginScreen.kt`

```kotlin
package com.illareklab.demodata.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.unit.dp

@Composable
fun LoginScreen(
    onSubmit: (username: String, password: String, onResult: (Boolean) -> Unit) -> Unit
) {
    var usuario    by remember { mutableStateOf("") }
    var password   by remember { mutableStateOf("") }
    var error      by remember { mutableStateOf("") }
    var verificando by remember { mutableStateOf(false) }

    Column(
        modifier            = Modifier.fillMaxSize().padding(32.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            "DemoData",
            style = MaterialTheme.typography.displayMedium,
            color = MaterialTheme.colorScheme.primary
        )
        Text(
            "Sistema de gestión de datos",
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(modifier = Modifier.height(48.dp))

        OutlinedTextField(
            value         = usuario,
            onValueChange = { usuario = it },
            label         = { Text("Usuario") },
            singleLine    = true,
            enabled       = !verificando,
            modifier      = Modifier.fillMaxWidth()
        )
        Spacer(modifier = Modifier.height(8.dp))

        OutlinedTextField(
            value               = password,
            onValueChange       = { password = it },
            label               = { Text("Contraseña") },
            singleLine          = true,
            enabled             = !verificando,
            visualTransformation = PasswordVisualTransformation(),
            modifier            = Modifier.fillMaxWidth()
        )

        if (error.isNotEmpty()) {
            Spacer(modifier = Modifier.height(8.dp))
            Text(error, color = MaterialTheme.colorScheme.error)
        }

        Spacer(modifier = Modifier.height(24.dp))
        Button(
            onClick = {
                error      = ""
                verificando = true
                onSubmit(usuario, password) { ok ->
                    verificando = false
                    if (!ok) error = "Credenciales incorrectas. Pruebe jkn/jkn."
                }
            },
            enabled  = !verificando && usuario.isNotBlank() && password.isNotBlank(),
            modifier = Modifier.fillMaxWidth().height(50.dp)
        ) {
            if (verificando) {
                CircularProgressIndicator(
                    modifier    = Modifier.height(24.dp),
                    strokeWidth = 2.dp,
                    color       = MaterialTheme.colorScheme.onPrimary
                )
            } else {
                Text("Ingresar")
            }
        }

        Spacer(modifier = Modifier.height(16.dp))
        Text(
            "Credenciales por defecto: jkn / jkn",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.outline
        )
    }
}
```

`PasswordVisualTransformation()` reemplaza cada carácter del campo por `•` en la pantalla sin cifrar el valor del state — el state sigue siendo texto plano en memoria. Para producción, combinar con `SecureString` o limpiar el estado tras el login.

---

## 11. `Navigation.kt` — Login gate y 6 tabs

```kotlin
package com.illareklab.demodata.ui

import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.Logout
import androidx.compose.material.icons.filled.CameraAlt
import androidx.compose.material.icons.filled.CloudSync
import androidx.compose.material.icons.filled.LocationOn
import androidx.compose.material.icons.filled.Mic
import androidx.compose.material.icons.filled.Notifications
import androidx.compose.material.icons.filled.Person
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.illareklab.demodata.DemoDataApp
import com.illareklab.demodata.ui.screens.*
import com.illareklab.demodata.ui.viewmodel.SessionViewModel

@Composable
fun Navigation() {
    val app = LocalContext.current.applicationContext as DemoDataApp
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
    val nav      = rememberNavController()
    var selected by remember { mutableIntStateOf(0) }
    val username by sessionVm.username.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(
                title   = { Text("DemoData — ${username ?: "?"}") },
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
                    "gps"     to (Icons.Default.LocationOn   to "GNSS"),
                    "media"   to (Icons.Default.CameraAlt    to "Multimedia"),
                    "audio"   to (Icons.Default.Mic          to "Audio"),
                    "sync"    to (Icons.Default.CloudSync    to "Sync"),
                    "notif"   to (Icons.Default.Notifications to "Notif"),
                    "profile" to (Icons.Default.Person       to "Perfil")
                )
                tabs.forEachIndexed { idx, (route, iconLabel) ->
                    val (icon, label) = iconLabel
                    NavigationBarItem(
                        selected = selected == idx,
                        onClick  = { selected = idx; nav.navigate(route) },
                        icon     = { Icon(icon, contentDescription = null) },
                        label    = { Text(label) }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(
            navController    = nav,
            startDestination = "gps",
            modifier         = Modifier.padding(padding)
        ) {
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

**Login gate:** `Navigation()` comprueba `isLoggedIn` antes de mostrar el `Scaffold`. Cuando cambia a `false` (logout), Compose recompone y muestra `LoginScreen` — sin navegar a ninguna ruta explícita.

**`viewModel(factory = ...)` en un Composable:** en este proyecto los ViewModels no se crean en `MainActivity` sino directamente en los Composables que los necesitan usando `LocalContext.current.applicationContext as DemoDataApp` para obtener las dependencias. Este patrón evita pasar los ViewModels por toda la jerarquía de parámetros.

---

## 12. `MediaScreen.kt` — Captura de foto y video

```kotlin
package com.illareklab.demodata.ui.screens

import android.Manifest
import android.net.Uri
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.PhotoCamera
import androidx.compose.material.icons.filled.Videocam
import androidx.compose.material.icons.outlined.Delete
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.core.content.FileProvider
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import coil.compose.AsyncImage
import com.google.accompanist.permissions.ExperimentalPermissionsApi
import com.google.accompanist.permissions.isGranted
import com.google.accompanist.permissions.rememberPermissionState
import com.illareklab.demodata.DemoDataApp
import com.illareklab.demodata.data.local.entity.MediaEntity
import com.illareklab.demodata.data.local.entity.MediaType
import com.illareklab.demodata.ui.viewmodel.MediaViewModel
import java.io.File
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun MediaScreen() {
    val context = LocalContext.current
    val app = context.applicationContext as DemoDataApp
    val vm: MediaViewModel = viewModel(
        factory = MediaViewModel.Factory(app.mediaRepository, app.fileStorage)
    )

    val mediaList        by vm.mediaList.collectAsStateWithLifecycle()
    val cameraPermission = rememberPermissionState(Manifest.permission.CAMERA)

    var pendingFile       by remember { mutableStateOf<File?>(null) }
    var pendingType       by remember { mutableStateOf<MediaType?>(null) }
    var videoStartTimeMs  by remember { mutableStateOf(0L) }

    // ── Launcher para fotos ──
    val photoLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.TakePicture()
    ) { success ->
        val file = pendingFile
        if (success && file != null && file.exists()) {
            vm.onPhotoCaptured(file.absolutePath, widthPx = 0, heightPx = 0)
        } else {
            file?.takeIf { it.exists() }?.delete()
        }
        pendingFile = null
        pendingType = null
    }

    // ── Launcher para videos ──
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
        pendingType = null
    }

    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        Text("Multimedia", style = MaterialTheme.typography.headlineSmall)
        Text(
            "Fotos y videos guardados en filesDir",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(modifier = Modifier.height(16.dp))

        if (!cameraPermission.status.isGranted) {
            Card(
                modifier = Modifier.fillMaxWidth(),
                colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.errorContainer)
            ) {
                Column(modifier = Modifier.padding(16.dp)) {
                    Text(
                        "Esta pantalla necesita permiso de cámara.",
                        color = MaterialTheme.colorScheme.onErrorContainer
                    )
                    Spacer(modifier = Modifier.height(8.dp))
                    Button(onClick = { cameraPermission.launchPermissionRequest() }) {
                        Text("Conceder permiso")
                    }
                }
            }
            return@Column
        }

        Row(modifier = Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(
                onClick = {
                    val file = vm.newPhotoFile()
                    val uri  = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
                    pendingFile = file
                    pendingType = MediaType.PHOTO
                    photoLauncher.launch(uri)
                },
                modifier = Modifier.weight(1f).height(56.dp)
            ) {
                Icon(Icons.Default.PhotoCamera, contentDescription = null)
                Spacer(modifier = Modifier.width(8.dp))
                Text("Foto")
            }
            Button(
                onClick = {
                    val file = vm.newVideoFile()
                    val uri  = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
                    pendingFile      = file
                    pendingType      = MediaType.VIDEO
                    videoStartTimeMs = System.currentTimeMillis()
                    videoLauncher.launch(uri)
                },
                modifier = Modifier.weight(1f).height(56.dp)
            ) {
                Icon(Icons.Default.Videocam, contentDescription = null)
                Spacer(modifier = Modifier.width(8.dp))
                Text("Video")
            }
        }

        Spacer(modifier = Modifier.height(16.dp))
        Text("${mediaList.size} elementos capturados", style = MaterialTheme.typography.titleSmall)
        Spacer(modifier = Modifier.height(8.dp))

        if (mediaList.isEmpty()) {
            Box(modifier = Modifier.fillMaxWidth().padding(32.dp), contentAlignment = Alignment.Center) {
                Text(
                    "Aún no has capturado nada. Tap en Foto o Video.",
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.outline
                )
            }
        } else {
            LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                items(items = mediaList, key = { it.id }) { media ->
                    MediaItemRow(media = media, onDelete = { vm.delete(media) })
                }
            }
        }
    }
}

@Composable
private fun MediaItemRow(media: MediaEntity, onDelete: () -> Unit) {
    val dateFormat = remember { SimpleDateFormat("dd/MM HH:mm:ss", Locale.getDefault()) }

    Card(modifier = Modifier.fillMaxWidth()) {
        Row(modifier = Modifier.padding(12.dp), verticalAlignment = Alignment.CenterVertically) {
            AsyncImage(
                model              = File(media.filePath),
                contentDescription = null,
                contentScale       = ContentScale.Crop,
                modifier           = Modifier.size(64.dp).clip(RoundedCornerShape(8.dp))
            )
            Spacer(modifier = Modifier.width(12.dp))
            Column(modifier = Modifier.weight(1f)) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Icon(
                        imageVector = if (media.type == MediaType.PHOTO.name)
                            Icons.Default.PhotoCamera else Icons.Default.Videocam,
                        contentDescription = null,
                        tint = MaterialTheme.colorScheme.primary,
                        modifier = Modifier.size(16.dp)
                    )
                    Spacer(modifier = Modifier.width(4.dp))
                    Text(media.type, style = MaterialTheme.typography.labelMedium, color = MaterialTheme.colorScheme.primary)
                }
                Text(
                    "${media.sizeBytes / 1024} KB" + (media.durationMs?.let { " · ${it / 1000}s" } ?: ""),
                    style = MaterialTheme.typography.bodyMedium
                )
                Text(dateFormat.format(Date(media.timestamp)), style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.outline)
            }
            IconButton(onClick = onDelete) {
                Icon(Icons.Outlined.Delete, contentDescription = "Eliminar", tint = MaterialTheme.colorScheme.error)
            }
        }
    }
}
```

### Flujo de captura con `ActivityResultContracts`

```
tap "Foto"
  → vm.newPhotoFile()         ← crea File vacío en filesDir/photos/
  → FileProvider.getUriForFile() ← convierte la ruta privada en content://URI
  → photoLauncher.launch(uri) ← abre la cámara del sistema
       │
  [usuario toma la foto]
       │
  photoLauncher callback(success = true)
  → vm.onPhotoCaptured(path, w, h)
      → mediaRepository.registerPhoto()
          → Room INSERT ← Flow emite → LazyColumn recompone
```

**`pendingFile`:** la URI pasada al launcher solo apunta al archivo destino. Cuando el callback llega, hay que verificar que `file.exists()` porque el usuario puede cancelar sin tomar la foto, en cuyo caso el archivo permanece vacío y debe eliminarse.

---

## 13. `AudioScreen.kt` — Grabadora con timer reactivo

```kotlin
package com.illareklab.demodata.ui.screens

import android.Manifest
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.AudioFile
import androidx.compose.material.icons.filled.FiberManualRecord
import androidx.compose.material.icons.filled.Mic
import androidx.compose.material.icons.filled.Stop
import androidx.compose.material.icons.outlined.Delete
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.google.accompanist.permissions.ExperimentalPermissionsApi
import com.google.accompanist.permissions.isGranted
import com.google.accompanist.permissions.rememberPermissionState
import com.illareklab.demodata.DemoDataApp
import com.illareklab.demodata.data.local.entity.AudioEntity
import com.illareklab.demodata.ui.viewmodel.AudioViewModel
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun AudioScreen() {
    val context = LocalContext.current
    val app     = context.applicationContext as DemoDataApp
    val vm: AudioViewModel = viewModel(
        factory = AudioViewModel.Factory(
            context.applicationContext,
            app.audioRepository,
            app.fileStorage
        )
    )

    val audios         by vm.audios.collectAsStateWithLifecycle()
    val isRecording    by vm.isRecording.collectAsStateWithLifecycle()
    val elapsedSeconds by vm.elapsedSeconds.collectAsStateWithLifecycle()
    val micPermission  = rememberPermissionState(Manifest.permission.RECORD_AUDIO)

    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        Text("Audio", style = MaterialTheme.typography.headlineSmall)
        Text(
            "Grabaciones AAC guardadas en filesDir",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(modifier = Modifier.height(16.dp))

        if (!micPermission.status.isGranted) {
            Card(
                modifier = Modifier.fillMaxWidth(),
                colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.errorContainer)
            ) {
                Column(modifier = Modifier.padding(16.dp)) {
                    Text("Esta pantalla necesita permiso de micrófono.", color = MaterialTheme.colorScheme.onErrorContainer)
                    Spacer(modifier = Modifier.height(8.dp))
                    Button(onClick = { micPermission.launchPermissionRequest() }) { Text("Conceder permiso") }
                }
            }
            return@Column
        }

        // ── Card con el timer ──
        Card(
            modifier = Modifier.fillMaxWidth(),
            colors   = CardDefaults.cardColors(
                containerColor = if (isRecording)
                    MaterialTheme.colorScheme.errorContainer
                else
                    MaterialTheme.colorScheme.surfaceVariant
            )
        ) {
            Column(
                modifier            = Modifier.fillMaxWidth().padding(24.dp),
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    if (isRecording) {
                        Icon(Icons.Default.FiberManualRecord, contentDescription = null, tint = MaterialTheme.colorScheme.error)
                        Spacer(modifier = Modifier.width(8.dp))
                    }
                    Text(
                        text  = if (isRecording) "Grabando…" else "Detenido",
                        style = MaterialTheme.typography.titleLarge
                    )
                }
                Spacer(modifier = Modifier.height(8.dp))
                Text(
                    text       = String.format(Locale.US, "%02d:%02d", elapsedSeconds / 60, elapsedSeconds % 60),
                    style      = MaterialTheme.typography.displayMedium,
                    fontFamily = FontFamily.Monospace
                )
            }
        }

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick  = { if (isRecording) vm.stopRecording() else vm.startRecording() },
            modifier = Modifier.fillMaxWidth().height(56.dp),
            colors   = ButtonDefaults.buttonColors(
                containerColor = if (isRecording)
                    MaterialTheme.colorScheme.error
                else
                    MaterialTheme.colorScheme.primary
            )
        ) {
            Icon(if (isRecording) Icons.Default.Stop else Icons.Default.Mic, contentDescription = null)
            Spacer(modifier = Modifier.width(8.dp))
            Text(if (isRecording) "Detener grabación" else "Iniciar grabación")
        }

        Spacer(modifier = Modifier.height(16.dp))
        Text("${audios.size} grabaciones guardadas", style = MaterialTheme.typography.titleSmall)
        Spacer(modifier = Modifier.height(8.dp))

        if (audios.isEmpty()) {
            Box(modifier = Modifier.fillMaxWidth().padding(32.dp), contentAlignment = Alignment.Center) {
                Text(
                    "Aún no hay grabaciones. Tap en Iniciar grabación.",
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.outline
                )
            }
        } else {
            LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                items(items = audios, key = { it.id }) { audio ->
                    AudioItemRow(audio = audio, onDelete = { vm.delete(audio) })
                }
            }
        }
    }
}

@Composable
private fun AudioItemRow(audio: AudioEntity, onDelete: () -> Unit) {
    val dateFormat = remember { SimpleDateFormat("dd/MM HH:mm:ss", Locale.getDefault()) }

    Card(modifier = Modifier.fillMaxWidth()) {
        Row(modifier = Modifier.padding(12.dp), verticalAlignment = Alignment.CenterVertically) {
            Box(modifier = Modifier.size(48.dp), contentAlignment = Alignment.Center) {
                Icon(Icons.Default.AudioFile, contentDescription = null, tint = MaterialTheme.colorScheme.primary, modifier = Modifier.size(40.dp))
            }
            Spacer(modifier = Modifier.width(12.dp))
            Column(modifier = Modifier.weight(1f)) {
                Text("${audio.durationMs / 1000} seg · ${audio.format}", style = MaterialTheme.typography.titleSmall)
                Text("${audio.sizeBytes / 1024} KB", style = MaterialTheme.typography.bodyMedium)
                Text(dateFormat.format(Date(audio.timestamp)), style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.outline)
            }
            IconButton(onClick = onDelete) {
                Icon(Icons.Outlined.Delete, contentDescription = "Eliminar", tint = MaterialTheme.colorScheme.error)
            }
        }
    }
}
```

---

## 14. `NotificationsScreen.kt` + `DelayedNotificationWorker.kt`

### `NotificationsScreen.kt`

```kotlin
package com.illareklab.demodata.ui.screens

import android.Manifest
import android.os.Build
import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.NotificationAdd
import androidx.compose.material.icons.filled.Schedule
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import androidx.work.workDataOf
import com.google.accompanist.permissions.ExperimentalPermissionsApi
import com.google.accompanist.permissions.isGranted
import com.google.accompanist.permissions.rememberPermissionState
import com.illareklab.demodata.workers.DelayedNotificationWorker
import java.util.concurrent.TimeUnit

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun NotificationsScreen() {
    val context = LocalContext.current
    var mensaje             by remember { mutableStateOf("") }
    var ultimoEnvio         by remember { mutableStateOf<String?>(null) }
    var contadorProgramadas by remember { mutableStateOf(0) }

    val notifPermission = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        rememberPermissionState(Manifest.permission.POST_NOTIFICATIONS)
    } else null

    val tienePermiso = notifPermission?.status?.isGranted ?: true

    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        Text("Notificaciones", style = MaterialTheme.typography.headlineSmall)
        Text(
            "Programa notificaciones locales con WorkManager",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(modifier = Modifier.height(16.dp))

        if (!tienePermiso) {
            Card(
                modifier = Modifier.fillMaxWidth(),
                colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.errorContainer)
            ) {
                Column(modifier = Modifier.padding(16.dp)) {
                    Text("Se requiere permiso para mostrar notificaciones.", color = MaterialTheme.colorScheme.onErrorContainer)
                    Spacer(modifier = Modifier.height(8.dp))
                    Button(onClick = { notifPermission?.launchPermissionRequest() }) { Text("Conceder permiso") }
                }
            }
            return@Column
        }

        OutlinedTextField(
            value         = mensaje,
            onValueChange = { mensaje = it },
            label         = { Text("Mensaje de la notificación") },
            placeholder   = { Text("Ej: Revisar inventario") },
            modifier      = Modifier.fillMaxWidth(),
            minLines      = 2,
            maxLines      = 4
        )

        Spacer(modifier = Modifier.height(16.dp))

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
            enabled  = mensaje.isNotBlank(),
            modifier = Modifier.fillMaxWidth().height(56.dp)
        ) {
            Icon(Icons.Default.NotificationAdd, contentDescription = null)
            Spacer(modifier = Modifier.width(8.dp))
            Text("Programar notificación (10 s)")
        }

        Spacer(modifier = Modifier.height(16.dp))

        Card(
            modifier = Modifier.fillMaxWidth(),
            colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer)
        ) {
            Column(modifier = Modifier.padding(16.dp)) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Icon(Icons.Default.Schedule, contentDescription = null, tint = MaterialTheme.colorScheme.onPrimaryContainer)
                    Spacer(modifier = Modifier.width(8.dp))
                    Text(
                        "Notificaciones programadas en esta sesión: $contadorProgramadas",
                        style = MaterialTheme.typography.titleSmall,
                        color = MaterialTheme.colorScheme.onPrimaryContainer
                    )
                }
                if (ultimoEnvio != null) {
                    Spacer(modifier = Modifier.height(8.dp))
                    Text("Último mensaje: \"$ultimoEnvio\"", style = MaterialTheme.typography.bodyMedium, color = MaterialTheme.colorScheme.onPrimaryContainer)
                }
            }
        }
    }
}
```

### `DelayedNotificationWorker.kt`

```kotlin
package com.illareklab.demodata.workers

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
            NotificationChannel(CHANNEL_ID, "Alertas DemoData", NotificationManager.IMPORTANCE_HIGH)
        )

        val notif = NotificationCompat.Builder(applicationContext, CHANNEL_ID)
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setContentTitle("Recordatorio DemoData")
            .setContentText(message)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .build()

        manager.notify(System.currentTimeMillis().toInt(), notif)
        return Result.success()
    }
}
```

### ¿Por qué WorkManager en vez de `Handler.postDelayed`?

| `Handler.postDelayed` | `WorkManager + setInitialDelay` |
|---|---|
| Cancelado si la app se cierra | Persiste aunque la app muera |
| No sobrevive reboots | Puede reencolar tras reboot con `BOOT_COMPLETED` |
| No tiene garantías en Doze Mode | El sistema lo reagrupa respetando Doze |
| Sin datos de entrada | `inputData` pasa datos serializados al Worker |

`WorkManager` delega al `JobScheduler` (Android 5+) o `AlarmManager` según la versión del SO, respetando las políticas de ahorro de batería del dispositivo.

---

## 15. `SyncScreen.kt` + `SyncViewModel.kt`

### `SyncViewModel.kt`

```kotlin
package com.illareklab.demodata.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.illareklab.demodata.data.repository.AudioRepository
import com.illareklab.demodata.data.repository.GpsRepository
import com.illareklab.demodata.data.repository.MediaRepository
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.stateIn

data class SyncCounts(
    val gpsGoogle: Int = 0,
    val gpsSensors: Int = 0,
    val photos: Int    = 0,
    val videos: Int    = 0,
    val audios: Int    = 0
) {
    val total: Int get() = gpsGoogle + gpsSensors + photos + videos + audios
}

class SyncViewModel(
    gpsRepository:   GpsRepository,
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
        private val gps:   GpsRepository,
        private val media: MediaRepository,
        private val audio: AudioRepository
    ) : ViewModelProvider.Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel> create(modelClass: Class<T>): T =
            SyncViewModel(gps, media, audio) as T
    }
}
```

`combine` con 5 flows emite cada vez que cualquiera de los contadores cambia. `total` es una propiedad calculada del data class — no ocupa columna en Room, se deriva en memoria.

### `SyncScreen.kt`

```kotlin
package com.illareklab.demodata.ui.screens

import android.widget.Toast
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.illareklab.demodata.DemoDataApp
import com.illareklab.demodata.ui.viewmodel.SyncViewModel

@Composable
fun SyncScreen() {
    val context = LocalContext.current
    val app     = context.applicationContext as DemoDataApp
    val vm: SyncViewModel = viewModel(
        factory = SyncViewModel.Factory(app.gpsRepository, app.mediaRepository, app.audioRepository)
    )

    val counts by vm.counts.collectAsStateWithLifecycle()

    Column(
        modifier = Modifier.fillMaxSize().verticalScroll(rememberScrollState()).padding(16.dp)
    ) {
        Text("Sync Center", style = MaterialTheme.typography.headlineSmall)
        Text(
            "Inventario de registros locales pendientes",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick  = { Toast.makeText(context, "Por implementar", Toast.LENGTH_SHORT).show() },
            modifier = Modifier.fillMaxWidth().height(56.dp)
        ) {
            Icon(Icons.Default.CloudUpload, contentDescription = null)
            Spacer(modifier = Modifier.width(8.dp))
            Text("Sincronizar ahora")
        }

        Spacer(modifier = Modifier.height(8.dp))
        Text(
            "El servidor se integrará en una fase posterior.",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.outline
        )

        Spacer(modifier = Modifier.height(24.dp))

        Card(
            modifier = Modifier.fillMaxWidth(),
            colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer)
        ) {
            Row(
                modifier              = Modifier.fillMaxWidth().padding(20.dp),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.CenterVertically
            ) {
                Column {
                    Text("Total de registros locales", style = MaterialTheme.typography.titleMedium, color = MaterialTheme.colorScheme.onPrimaryContainer)
                    Text("Suma de todas las categorías", style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.onPrimaryContainer)
                }
                Text("${counts.total}", style = MaterialTheme.typography.displaySmall, color = MaterialTheme.colorScheme.onPrimaryContainer)
            }
        }

        Spacer(modifier = Modifier.height(16.dp))
        Text("Desglose por tipo", style = MaterialTheme.typography.titleSmall)
        Spacer(modifier = Modifier.height(8.dp))

        CategoryRow(Icons.Default.LocationOn, "GNSS Google FLP",  counts.gpsGoogle)
        Spacer(modifier = Modifier.height(8.dp))
        CategoryRow(Icons.Default.Sensors,    "GNSS Sensores HW", counts.gpsSensors)
        Spacer(modifier = Modifier.height(8.dp))
        CategoryRow(Icons.Default.PhotoCamera,"Fotos",            counts.photos)
        Spacer(modifier = Modifier.height(8.dp))
        CategoryRow(Icons.Default.Videocam,   "Videos",           counts.videos)
        Spacer(modifier = Modifier.height(8.dp))
        CategoryRow(Icons.Default.AudioFile,  "Audios",           counts.audios)
    }
}

@Composable
private fun CategoryRow(icon: ImageVector, label: String, count: Int) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Row(modifier = Modifier.fillMaxWidth().padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
            Icon(icon, contentDescription = null, tint = MaterialTheme.colorScheme.primary)
            Spacer(modifier = Modifier.width(12.dp))
            Text(label, style = MaterialTheme.typography.bodyLarge, modifier = Modifier.weight(1f))
            Text(
                "$count",
                style = MaterialTheme.typography.titleLarge,
                color = if (count > 0) MaterialTheme.colorScheme.primary else MaterialTheme.colorScheme.outline
            )
        }
    }
}
```

---

## 16. `ProfileScreen.kt` — Historial unificado con `sealed class`

```kotlin
package com.illareklab.demodata.ui.screens

import android.content.Intent
import androidx.compose.foundation.clickable
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.Logout
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.core.content.FileProvider
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import coil.compose.AsyncImage
import com.illareklab.demodata.DemoDataApp
import com.illareklab.demodata.data.local.entity.AudioEntity
import com.illareklab.demodata.data.local.entity.GpsGoogleEntity
import com.illareklab.demodata.data.local.entity.GpsSensorsEntity
import com.illareklab.demodata.data.local.entity.MediaEntity
import com.illareklab.demodata.data.local.entity.MediaType
import com.illareklab.demodata.ui.viewmodel.SessionViewModel
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.io.File
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

@Composable
fun ProfileScreen(onLogout: () -> Unit, username: String? = null) {
    val app = LocalContext.current.applicationContext as DemoDataApp
    val sessionVm: SessionViewModel = viewModel(
        factory = SessionViewModel.Factory(app.sessionManager)
    )

    var viewState by remember { mutableStateOf<ProfileViewState>(ProfileViewState.Menu) }

    when (viewState) {
        ProfileViewState.Menu -> ProfileMenu(
            username            = username,
            onLogout            = onLogout,
            onNavigateToProfile  = { viewState = ProfileViewState.MyProfile },
            onNavigateToActivity = { viewState = ProfileViewState.MyActivity }
        )
        ProfileViewState.MyProfile -> MyProfileScreen(
            username  = username,
            sessionVm = sessionVm,
            onBack    = { viewState = ProfileViewState.Menu }
        )
        ProfileViewState.MyActivity -> MyActivityScreen(
            onBack = { viewState = ProfileViewState.Menu }
        )
    }
}

private sealed class ProfileViewState {
    object Menu       : ProfileViewState()
    object MyProfile  : ProfileViewState()
    object MyActivity : ProfileViewState()
}

@Composable
private fun ProfileMenu(
    username: String?,
    onLogout: () -> Unit,
    onNavigateToProfile: () -> Unit,
    onNavigateToActivity: () -> Unit
) {
    var mostrarConfirmacion by remember { mutableStateOf(false) }

    Column(
        modifier            = Modifier.fillMaxSize().padding(24.dp).verticalScroll(rememberScrollState()),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Icon(Icons.Default.Person, contentDescription = null, modifier = Modifier.size(72.dp), tint = MaterialTheme.colorScheme.primary)
        Spacer(modifier = Modifier.height(16.dp))
        Text(text = username ?: "Usuario", style = MaterialTheme.typography.headlineMedium)
        Spacer(modifier = Modifier.height(32.dp))

        MenuOption(icon = Icons.Default.Person,  title = "Mi Perfil",    subtitle = "Ver metadatos del usuario",                  onClick = onNavigateToProfile)
        Spacer(modifier = Modifier.height(12.dp))
        MenuOption(icon = Icons.Default.History, title = "Mi Actividad", subtitle = "Registros locales de GNSS y multimedia",     onClick = onNavigateToActivity)
        Spacer(modifier = Modifier.height(32.dp))

        Button(
            onClick  = { mostrarConfirmacion = true },
            modifier = Modifier.fillMaxWidth().height(56.dp),
            colors   = ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.error)
        ) {
            Icon(Icons.AutoMirrored.Filled.Logout, contentDescription = null)
            Spacer(modifier = Modifier.size(8.dp))
            Text("Cerrar sesión")
        }
    }

    if (mostrarConfirmacion) {
        LogoutDialog(onConfirm = onLogout, onDismiss = { mostrarConfirmacion = false })
    }
}

@Composable
private fun MenuOption(icon: ImageVector, title: String, subtitle: String, onClick: () -> Unit) {
    Card(
        modifier = Modifier.fillMaxWidth().clickable(onClick = onClick),
        colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceVariant)
    ) {
        Row(modifier = Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
            Icon(icon, contentDescription = null, tint = MaterialTheme.colorScheme.primary)
            Spacer(modifier = Modifier.width(16.dp))
            Column(modifier = Modifier.weight(1f)) {
                Text(title,    style = MaterialTheme.typography.titleMedium)
                Text(subtitle, style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
            Icon(Icons.Default.ChevronRight, contentDescription = null)
        }
    }
}

@Composable
private fun MyProfileScreen(username: String?, sessionVm: SessionViewModel, onBack: () -> Unit) {
    val isDarkModePref by sessionVm.isDarkMode.collectAsStateWithLifecycle()
    val isDark = isDarkModePref ?: isSystemInDarkTheme()

    Column(modifier = Modifier.fillMaxSize().padding(24.dp).verticalScroll(rememberScrollState())) {
        Text("Mi Perfil", style = MaterialTheme.typography.headlineSmall)
        Spacer(modifier = Modifier.height(24.dp))

        ProfileMetadataItem("Username",       username ?: "N/A")
        ProfileMetadataItem("Rol",            "Administrador / Operador")
        ProfileMetadataItem("Directorio Local", LocalContext.current.filesDir.absolutePath)

        Row(
            modifier              = Modifier.fillMaxWidth().padding(vertical = 12.dp),
            verticalAlignment     = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Icon(Icons.Default.DarkMode, contentDescription = null, tint = MaterialTheme.colorScheme.primary)
                Spacer(modifier = Modifier.width(16.dp))
                Column {
                    Text("Modo Noche", style = MaterialTheme.typography.titleMedium)
                    Text(
                        if (isDarkModePref == null) "Siguiendo al sistema"
                        else if (isDark) "Activado" else "Desactivado",
                        style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.onSurfaceVariant
                    )
                }
            }
            Switch(checked = isDark, onCheckedChange = { sessionVm.setDarkMode(it) })
        }
        HorizontalDivider()

        Spacer(modifier = Modifier.height(16.dp))
        ProfileMetadataItem("Dispositivo",      "${android.os.Build.MANUFACTURER} ${android.os.Build.MODEL}")
        ProfileMetadataItem("Android Version",  android.os.Build.VERSION.RELEASE)
        ProfileMetadataItem("API Level",        android.os.Build.VERSION.SDK_INT.toString())

        Spacer(modifier = Modifier.height(32.dp))
        Button(onClick = onBack, modifier = Modifier.fillMaxWidth()) { Text("Volver") }
    }
}

@Composable
private fun ProfileMetadataItem(label: String, value: String) {
    Column(modifier = Modifier.padding(vertical = 8.dp)) {
        Text(label, style = MaterialTheme.typography.labelMedium, color = MaterialTheme.colorScheme.primary)
        Text(value, style = MaterialTheme.typography.bodyLarge)
        HorizontalDivider(modifier = Modifier.padding(top = 8.dp))
    }
}

@Composable
private fun MyActivityScreen(onBack: () -> Unit) {
    val context  = LocalContext.current
    val app      = context.applicationContext as DemoDataApp

    val googlePoints  by app.gpsRepository.googlePoints.collectAsStateWithLifecycle(emptyList())
    val sensorsPoints by app.gpsRepository.sensorsPoints.collectAsStateWithLifecycle(emptyList())
    val allMedia      by app.mediaRepository.allMedia.collectAsStateWithLifecycle(emptyList())
    val allAudios     by app.audioRepository.allAudios.collectAsStateWithLifecycle(emptyList())

    var combinedItems by remember { mutableStateOf<List<ActivityItem>>(emptyList()) }

    LaunchedEffect(googlePoints, sensorsPoints, allMedia, allAudios) {
        withContext(Dispatchers.Default) {
            val items = mutableListOf<ActivityItem>()
            items.addAll(googlePoints.map  { ActivityItem.GpsGoogle(it) })
            items.addAll(sensorsPoints.map { ActivityItem.GpsSensors(it) })
            items.addAll(allMedia.map      { ActivityItem.Media(it) })
            items.addAll(allAudios.map     { ActivityItem.Audio(it) })
            items.sortByDescending { it.timestamp }
            combinedItems = items
        }
    }

    var detailItem by remember { mutableStateOf<ActivityItem?>(null) }

    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Text("Mi Actividad", style = MaterialTheme.typography.headlineSmall, modifier = Modifier.weight(1f))
            TextButton(onClick = onBack) { Text("Cerrar") }
        }
        Spacer(modifier = Modifier.height(16.dp))

        LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp), modifier = Modifier.weight(1f)) {
            items(combinedItems) { item ->
                ActivityRow(item, onClick = { detailItem = item })
            }
        }
    }

    if (detailItem != null) {
        ActivityDetailDialog(item = detailItem!!, onDismiss = { detailItem = null })
    }
}

sealed class ActivityItem {
    abstract val timestamp: Long
    abstract val label: String
    abstract val icon: ImageVector

    data class GpsGoogle(val data: GpsGoogleEntity) : ActivityItem() {
        override val timestamp = data.timestamp
        override val label     = "GNSS Google"
        override val icon      = Icons.Default.LocationOn
    }
    data class GpsSensors(val data: GpsSensorsEntity) : ActivityItem() {
        override val timestamp = data.timestamp
        override val label     = "GNSS Sensor"
        override val icon      = Icons.Default.LocationOn
    }
    data class Media(val data: MediaEntity) : ActivityItem() {
        override val timestamp = data.timestamp
        override val label     = data.type
        override val icon      = if (data.type == MediaType.PHOTO.name) Icons.Default.PhotoCamera else Icons.Default.Videocam
    }
    data class Audio(val data: AudioEntity) : ActivityItem() {
        override val timestamp = data.timestamp
        override val label     = "Audio"
        override val icon      = Icons.Default.AudioFile
    }
}

@Composable
private fun ActivityRow(item: ActivityItem, onClick: () -> Unit) {
    val dateFormat = remember { SimpleDateFormat("dd/MM HH:mm:ss", Locale.getDefault()) }
    val isNoSignal = item is ActivityItem.GpsSensors && item.data.latitude == null

    Card(modifier = Modifier.fillMaxWidth().clickable(onClick = onClick)) {
        Row(modifier = Modifier.padding(12.dp), verticalAlignment = Alignment.CenterVertically) {
            Icon(
                item.icon,
                contentDescription = null,
                tint = if (isNoSignal) MaterialTheme.colorScheme.error else MaterialTheme.colorScheme.primary
            )
            Spacer(modifier = Modifier.width(16.dp))
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text  = if (isNoSignal) "${item.label} (Sin señal)" else item.label,
                    style = MaterialTheme.typography.titleSmall,
                    color = if (isNoSignal) MaterialTheme.colorScheme.error else Color.Unspecified
                )
                Text(dateFormat.format(Date(item.timestamp)), style = MaterialTheme.typography.bodySmall)
            }
            Icon(Icons.Default.ChevronRight, contentDescription = null)
        }
    }
}

@Composable
private fun ActivityDetailDialog(item: ActivityItem, onDismiss: () -> Unit) {
    val context    = LocalContext.current
    val dateFormat = remember { SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault()) }

    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(item.label) },
        text  = {
            Column(modifier = Modifier.verticalScroll(rememberScrollState())) {
                Text("Fecha: ${dateFormat.format(Date(item.timestamp))}")
                Spacer(modifier = Modifier.height(8.dp))

                when (item) {
                    is ActivityItem.GpsGoogle -> {
                        Text("Lat: ${item.data.latitude}")
                        Text("Lon: ${item.data.longitude}")
                        Text("Accuracy: ±${item.data.accuracy}m")
                        item.data.speed?.let { Text("Velocidad: $it m/s") }
                    }
                    is ActivityItem.GpsSensors -> {
                        if (item.data.latitude != null) {
                            Text("Lat: ${item.data.latitude}")
                            Text("Lon: ${item.data.longitude}")
                            item.data.altitude?.let { Text("Altitud: ${it}m") }
                        } else {
                            Text("Estado: SIN SEÑAL", color = MaterialTheme.colorScheme.error)
                            Text("Causa: Probable lugar cerrado (sin vista a satélites)")
                        }
                        Text("Provider: ${item.data.provider}")
                    }
                    is ActivityItem.Media -> {
                        Text("Tamaño: ${item.data.sizeBytes / 1024} KB")
                        Spacer(modifier = Modifier.height(8.dp))
                        AsyncImage(
                            model              = File(item.data.filePath),
                            contentDescription = null,
                            modifier           = Modifier.fillMaxWidth().height(200.dp).clip(RoundedCornerShape(8.dp)),
                            contentScale       = ContentScale.Crop
                        )
                        Spacer(modifier = Modifier.height(16.dp))
                        Button(onClick = {
                            openFile(context, item.data.filePath,
                                if (item.data.type == MediaType.PHOTO.name) "image/*" else "video/*")
                        }) {
                            Text(if (item.data.type == MediaType.PHOTO.name) "Ver Foto" else "Reproducir Video")
                        }
                    }
                    is ActivityItem.Audio -> {
                        Text("Duración: ${item.data.durationMs / 1000}s")
                        Text("Tamaño: ${item.data.sizeBytes / 1024} KB")
                        Spacer(modifier = Modifier.height(16.dp))
                        Button(onClick = { openFile(context, item.data.filePath, "audio/*") }) {
                            Text("Reproducir Audio")
                        }
                    }
                }
            }
        },
        confirmButton = { TextButton(onClick = onDismiss) { Text("Cerrar") } }
    )
}

private fun openFile(context: android.content.Context, path: String, mimeType: String) {
    try {
        val uri    = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", File(path))
        val intent = Intent(Intent.ACTION_VIEW).apply {
            setDataAndType(uri, mimeType)
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        context.startActivity(intent)
    } catch (_: Exception) { }
}

@Composable
private fun LogoutDialog(onConfirm: () -> Unit, onDismiss: () -> Unit) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title   = { Text("¿Confirmar cierre de sesión?") },
        text    = { Text("Volverás a la pantalla de login. Tus datos locales se conservan.") },
        confirmButton = {
            TextButton(onClick = onConfirm) {
                Text("Sí, cerrar sesión", color = MaterialTheme.colorScheme.error)
            }
        },
        dismissButton = { TextButton(onClick = onDismiss) { Text("Cancelar") } }
    )
}
```

### `sealed class ActivityItem` — polimorfismo sin `instanceof`

```
ActivityItem (sealed)
  ├── GpsGoogle  → label="GNSS Google", icon=LocationOn
  ├── GpsSensors → label="GNSS Sensor", icon=LocationOn (rojo si sin señal)
  ├── Media      → label=type, icon=PhotoCamera|Videocam
  └── Audio      → label="Audio",       icon=AudioFile
```

El `when(item)` en `ActivityDetailDialog` es exhaustivo: el compilador garantiza que todas las subclases estén cubiertas. Si se agrega un nuevo tipo `ActivityItem.Bluetooth`, el código no compilará hasta que se añada su `is ActivityItem.Bluetooth ->` branch.

**`LaunchedEffect` + `withContext(Dispatchers.Default)`:** la combinación de 4 listas se ejecuta fuera del Main Thread. `LaunchedEffect` se re-lanza cada vez que alguno de los 4 keys cambia, garantizando que `combinedItems` siempre esté actualizado.

---

## 17. `MainActivity.kt`

```kotlin
package com.illareklab.demodata

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.lifecycle.viewmodel.compose.viewModel
import com.illareklab.demodata.ui.Navigation
import com.illareklab.demodata.ui.theme.AppTheme
import com.illareklab.demodata.ui.viewmodel.SessionViewModel

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val app = applicationContext as DemoDataApp
            val sessionVm: SessionViewModel = viewModel(
                factory = SessionViewModel.Factory(app.sessionManager)
            )

            val isDarkModePref by sessionVm.isDarkMode.collectAsState()
            val darkTheme      = isDarkModePref ?: isSystemInDarkTheme()

            AppTheme(darkTheme = darkTheme) {
                Navigation()
            }
        }
    }
}
```

`MainActivity` es ahora más simple que en el Lab 4 — no inicializa repositorios directamente. `DemoDataApp` los tiene todos y cualquier Composable puede acceder a ellos con `LocalContext.current.applicationContext as DemoDataApp`.

---

## 18. Flujo de datos completo del laboratorio

### Ciclo de captura multimedia

```
[MediaScreen] tap "Foto"
  → vm.newPhotoFile()
      → FileStorageManager.newPhotoFile()
          → File("filesDir/photos/photo_TIMESTAMP.jpg")
  → FileProvider.getUriForFile() → content://URI
  → photoLauncher.launch(uri)  ← cámara del sistema escribe en el archivo
  → callback(success=true)
      → vm.onPhotoCaptured(path, w, h)
          → mediaRepository.registerPhoto()
              → mediaDao.insert(MediaEntity)
  → Room Flow emite → mediaList StateFlow → LazyColumn recompone
```

### Ciclo de grabación de audio

```
[AudioScreen] tap "Iniciar grabación"
  → vm.startRecording()
      → FileStorageManager.newAudioFile()
      → MediaRecorder.prepare() + start()
      → timerJob: delay(1s) → _elapsedSeconds++  ← cada segundo
  → [usuario toca "Detener"]
  → vm.stopRecording()
      → MediaRecorder.stop() + release()
      → audioRepository.registerAudio(path, durationMs)
          → audioDao.insert(AudioEntity)
  → Room Flow emite → audios StateFlow → LazyColumn recompone
```

### Ciclo de notificación diferida

```
[NotificationsScreen] tap "Programar"
  → OneTimeWorkRequestBuilder<DelayedNotificationWorker>()
       .setInitialDelay(10, TimeUnit.SECONDS)
       .setInputData(workDataOf("input_message" to mensaje))
       .build()
  → WorkManager.enqueue(request)
  → [10 segundos después, incluso con la app cerrada]
  → DelayedNotificationWorker.doWork()
      → NotificationManager.createNotificationChannel()
      → NotificationCompat.Builder(...).build()
      → manager.notify(...)  ← aparece en la barra de notificaciones
  → Result.success()
```

### Ciclo de sesión

```
[LoginScreen] usuario="jkn", password="jkn"
  → sessionVm.login("jkn", "jkn", onResult)
      → if (credenciales ok) sessionManager.login("jkn")
          → DataStore: is_logged_in=true, username="jkn"
          → isLoggedIn Flow emite true
          → Navigation: if(isLoggedIn) MainScaffold else LoginScreen
          → recompone → muestra MainScaffold con 6 tabs

[Perfil] tap "Cerrar sesión" → confirmar
  → sessionVm.logout()
      → DataStore.clear() preservando dark_mode
      → isLoggedIn Flow emite false
      → Navigation recompone → LoginScreen
```

---

## 19. Conceptos de las listas de temas cubiertos

| Tema | Lista | Cómo se ve en este lab |
|---|---|---|
| StateFlow y UDF | L1 | `isRecording`, `elapsedSeconds`, `mediaList`, `counts`: todos son StateFlow que la UI observa sin llamar `refresh()` |
| ViewModel + Factory | L1 | `MediaViewModel.Factory`, `AudioViewModel.Factory`, `SyncViewModel.Factory`: permiten pasar dependencias al constructor |
| sealed class | L1 | `ActivityItem` con 4 subtipos; `when(item)` exhaustivo garantizado por el compilador |
| Temas | L2 | `isDarkModePref ?: isSystemInDarkTheme()` en `MyProfileScreen` y `MainActivity` |
| Cámara con Activity Contracts | L2 | `TakePicture` y `CaptureVideo` launchers + `FileProvider` para URI segura |
| Layouts | L2 | `LazyColumn` con `key = { it.id }`, `Card`, `Row`, `Column` anidados |
| Containers | L2 | `AlertDialog` para detalle de item, `Card` con color reactivo al estado de grabación |
| Adaptadores | L2 | `items(mediaList, key = { it.id })` y `items(audios, key = { it.id })` |
| MediaRecorder | L3 | `setAudioSource → setOutputFormat → setAudioEncoder → prepare → start → stop → release` |
| WorkManager | L3 | `OneTimeWorkRequestBuilder + setInitialDelay + setInputData → Worker.doWork()` |
| FileProvider | L3 | `getUriForFile()` + `file_paths.xml` + `FLAG_GRANT_READ_URI_PERMISSION` |
| Capa de Datos | L3 | 4 entidades Room, 4 DAOs, 3 repositorios; `FileStorageManager` para archivos físicos |
| DI manual | L3 | `DemoDataApp` centraliza toda la infraestructura; Composables la obtienen via `applicationContext as DemoDataApp` |

---

## 20. Ejercicios propuestos

1. **Mostrar miniaturas de video** en `MediaItemRow`: `AsyncImage` de Coil intenta cargar el primer frame de un video por `File(path)`. Para verificar, grabar un video corto y revisar que aparece la miniatura.

2. **Agregar contador en tiempo real del tamaño** de grabación: `MediaRecorder.getMaxAmplitude()` devuelve la amplitud de la muestra actual. Mostrarlo en la `Card` del timer como una barra de nivel de volumen.

3. **Cancelar un WorkRequest programado**: guardar el `UUID` del `WorkRequest` y mostrar un botón "Cancelar" que llame a `WorkManager.getInstance(context).cancelWorkById(uuid)`.

4. **Exportar CSV del historial**: en `MyActivityScreen`, agregar un botón que itere `combinedItems`, genere una cadena CSV con timestamp, tipo y ruta, y la guarde en `FileStorageManager` como `export_TIMESTAMP.csv`.

5. **Validar credenciales con PBKDF2**: usar `PasswordHasher.hash()` para hashear la contraseña del usuario con un salt fijo y compararla con `constantTimeEquals` en lugar de la comparación de strings directa de `SessionViewModel.login()`.

---

## 21. Checklist del laboratorio

- [ ] `DemoDataApp` expone `fileStorage`, `mediaRepository` y `audioRepository` como `by lazy`
- [ ] `DemoDataDatabase` versión 3 con `MediaEntity` y `AudioEntity` en `entities`
- [ ] `DemoDataDatabase` usa `fallbackToDestructiveMigration()` para el laboratorio
- [ ] `FileStorageManager` crea directorios `photos/`, `videos/`, `audios/` con `apply { mkdirs() }`
- [ ] `MediaEntity` guarda `type` como `String` (nombre del enum, no el ordinal)
- [ ] `MediaDao` tiene `observeByType(type: String)`, `observePhotoCount()`, `observeVideoCount()`
- [ ] `MediaRepository.delete()` borra el archivo físico ANTES del registro de Room
- [ ] `MediaViewModel` tiene `Factory` y helpers `newPhotoFile()` / `newVideoFile()`
- [ ] `AudioEntity` incluye `durationMs`, `sizeBytes` y `format`
- [ ] `AudioViewModel.stopRecording()` descarta grabaciones < 1 segundo
- [ ] `AudioViewModel.onCleared()` libera el micrófono si se destruye mientras graba
- [ ] `AndroidManifest` declara `<provider FileProvider>` con `file_paths.xml`
- [ ] `file_paths.xml` mapea `photos/`, `videos/`, `audios/` con `<files-path>`
- [ ] `MediaScreen` usa `TakePicture` y `CaptureVideo` con `rememberLauncherForActivityResult`
- [ ] `MediaScreen` llama `FileProvider.getUriForFile()` antes de lanzar la cámara
- [ ] `AudioScreen` muestra timer `MM:SS` con `FontFamily.Monospace`
- [ ] `NotificationsScreen` usa `OneTimeWorkRequestBuilder + setInitialDelay(10, TimeUnit.SECONDS)`
- [ ] `DelayedNotificationWorker.doWork()` crea canal y muestra notificación con `PRIORITY_HIGH`
- [ ] `SyncViewModel` usa `combine(5 flows)` para producir `SyncCounts` con `total: Int`
- [ ] `ProfileScreen.MyActivityScreen` usa `LaunchedEffect + withContext(Default)` para la fusión
- [ ] `sealed class ActivityItem` cubre los 4 tipos y `when` es exhaustivo
- [ ] `ActivityDetailDialog` abre archivos con `FileProvider + Intent.ACTION_VIEW`
- [ ] `SessionViewModel` tiene `Factory` y función `login(username, password, onResult)`
- [ ] `Navigation` muestra `LoginScreen` cuando `isLoggedIn == false` y `MainScaffold` cuando es `true`
- [ ] La app compila, hace login con `jkn/jkn`, captura foto/video/audio y los muestra en historial


