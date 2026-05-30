# CodLab N°4 — Using&SavingDataSensors

### App Android con captura de sensores de hardware, Foreground Service, WorkManager y sincronización HTTP POST asíncrona

> **Objetivo del laboratorio:** Construir desde cero una app Android llamada **DataDemo** que use:
> - Arquitectura limpia modular dividida en capas (`data`, `security`, `ui`, `services`, `workers`).
> - Pantalla de autenticación local con control de la pila de navegación (*back stack*) y verificación de contraseña mediante **hash PBKDF2WithHmacSHA256** (estándar OWASP).
> - **Foreground Service** persistente que capture coordenadas GNSS y FLP cada 10 segundos.
> - **WorkManager** para disparar tareas diferidas (notificaciones programadas).
> - Pantalla de sincronización que envía registros por **HTTP POST** uno a uno y los elimina de la caché local con **animación de colapso vertical** tras la confirmación del servidor.

---

## Pre-requisitos

- Android Studio **Ladybug 2024.2.1** o superior (mínimo Jellyfish).
- JDK 17 (incluido en Android Studio).
- Emulador o dispositivo físico con **Android 8.0+ (API 26)**.
- Para probar GPS real: dispositivo físico o emulador con ubicación simulada activada.
- (Opcional) Endpoint HTTP de prueba (por ejemplo `https://webhook.site` o servidor local) para validar la sincronización.

---

## SECCIÓN 1 — Crear el proyecto

### 1.1 Abrir el wizard

**File → New → New Project** (o desde la pantalla de bienvenida: **"New Project"**).

### 1.2 Elegir el template correcto

Android Studio muestra varios templates con nombres similares. **Solo uno es el correcto** para este laboratorio:

| Template | ¿Lo elijo? | Razón |
|---|---|---|
| **Empty Activity** | Sí | Compose limpio + `MainActivity` con tema vacío. |
| Empty Views Activity | No | Es XML clásico (sistema *Views*), no Compose. |

> **Trampa común:** si eliges un template que contenga la palabra **"Views"** en el nombre, terminas con archivos `.xml` y `fragment_*.kt`, lo cual es incompatible con esta guía. **Cierra el proyecto y empieza de cero** si pasa esto.

**Cómo identificar el template correcto:**
- Su icono muestra fragmentos de código `{ }`.
- La descripción dice *"Creates a new empty activity with Jetpack Compose"*.
- Se llama **"Empty Activity"** a secas (sin "Views").

Selecciona **"Empty Activity"** y click en **"Next"**.

### 1.3 Configurar el proyecto

Llena los campos exactamente así:

| Campo | Valor |
|---|---|
| **Name** | `DataDemo` |
| **Package name** | `com.illareklab.data_demo` |
| **Save location** | (donde quieras guardarlo) |
| **Language** | Kotlin |
| **Minimum SDK** | API 26 (Android 8.0 Oreo) |
| **Build configuration language** | Kotlin DSL (`build.gradle.kts`) |

Click **"Finish"** y espera 1–3 minutos al **Gradle sync inicial**.

### 1.4 ¿Por qué API 26 como mínimo?

Es una decisión crítica de arquitectura. A partir de Android 8.0:

- Se introdujeron **límites estrictos** al consumo de batería en segundo plano.
- Los **Notification Channels** son obligatorios para mostrar cualquier notificación.
- Existen tipos específicos de Foreground Service (`location`, `camera`, `microphone`) que el sistema operativo exige declarar explícitamente.

Configurar API 26 nos asegura el acceso nativo a estos mecanismos. Sin ellos, el kernel de Android matará nuestra captura de sensores por motivos de privacidad/batería.

---

## SECCIÓN 2 — Declarar las dependencias

Abre `app/build.gradle.kts` y dentro del bloque `dependencies { ... }` agrega las siguientes librerías:

```kotlin
dependencies {
    // ... lo que ya tienes (compose-bom, material3, etc.) ...

    // ── Navegación ──
    implementation("androidx.navigation:navigation-compose:2.7.7")

    // ── Material Icons extendidos (Home, CloudSync, DeleteOutline, etc.) ──
    implementation("androidx.compose.material:material-icons-extended:1.6.7")

    // ── Hardware: ubicación de Google (Fused Location Provider) ──
    implementation("com.google.android.gms:play-services-location:21.2.0")

    // ── WorkManager: tareas diferidas y persistentes ──
    implementation("androidx.work:work-runtime-ktx:2.9.0")

    // ── CameraX (5 artefactos) ──
    val cameraxVersion = "1.3.3"
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")
    implementation("androidx.camera:camera-video:$cameraxVersion")

    // ── Permisos en tiempo de ejecución para Compose ──
    implementation("com.google.accompanist:accompanist-permissions:0.34.0")
}
```

Click en **"Sync Now"** (banner amarillo en la parte superior) y espera a que termine la sincronización.

### ¿Por qué cada dependencia?

| Librería | Justificación |
|---|---|
| `navigation-compose` | Permite mapear de forma desacoplada las pantallas (Login, Menú Principal, 5 Pestañas) sin instanciar múltiples Activities. |
| `material-icons-extended` | Compose trae solo unos pocos iconos básicos. Para usar `Icons.Default.CloudSync`, `Icons.Default.DeleteOutline`, etc., se necesita el paquete extendido con más de 2,000 iconos. |
| `play-services-location` | Acceso al **Fused Location Provider** (FLP) de Google, que combina GNSS, Wi-Fi y antenas celulares para obtener la mejor ubicación posible con menor consumo. |
| `work-runtime-ktx` | Garantiza la ejecución fiel de tareas diferidas (como notificaciones programadas) **aunque la app esté cerrada**. |
| `camera-*` | CameraX abstrae la complejidad de los distintos lentes y sensores del mercado, exponiendo un API ligado al ciclo de vida de Compose. |
| `accompanist-permissions` | Wrapper oficial de Google para gestionar permisos en runtime desde funciones `@Composable`. |

---

## SECCIÓN 3 — Configurar el `AndroidManifest.xml`

Abre `app/src/main/AndroidManifest.xml`. Vamos a hacer tres cambios:

1. Declarar los **permisos** necesarios para acceder a los periféricos.
2. Declarar las **características de hardware** (`<uses-feature>`) que la app usa pero **no** requiere obligatoriamente.
3. Registrar el **Foreground Service** y su tipo.

Reemplaza el contenido por:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

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

    <!-- ── Declaración explícita de hardware ──
         Indica a Google Play y a Lint que la app USA estos componentes
         pero NO los REQUIERE para funcionar (required="false"). Esto
         maximiza el alcance: tablets sin cámara, Chromebooks sin GPS y
         emuladores limitados podrán instalar la app sin filtros. -->
    <uses-feature
        android:name="android.hardware.camera"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.camera.autofocus"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.microphone"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.location"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.location.gps"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.bluetooth_le"
        android:required="false" />
    <uses-feature
        android:name="android.hardware.wifi"
        android:required="false" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/Theme.DataDemo"
        android:usesCleartextTraffic="true">

        <!-- Registro del servicio recolector de coordenadas -->
        <service
            android:name=".services.TrackingService"
            android:foregroundServiceType="location"
            android:exported="false" />

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.DataDemo">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

### ¿Por qué cada permiso?

| Permiso | Justificación |
|---|---|
| `ACCESS_FINE_LOCATION` | Requerido para leer el chip GNSS con precisión métrica. |
| `ACCESS_BACKGROUND_LOCATION` | Permite que la app siga leyendo posicionamiento aunque el usuario cambie a otra pestaña (Cámara, Audio). **Se solicita aparte en runtime.** |
| `FOREGROUND_SERVICE_LOCATION` | Obligatorio en Android 10+. Si el OS detecta un servicio que captura ubicación sin este permiso, **lo termina inmediatamente**. |
| `POST_NOTIFICATIONS` | Obligatorio en Android 13+ para mostrar cualquier notificación, incluida la del Foreground Service. |
| `usesCleartextTraffic="true"` | Permite tráfico HTTP sin TLS hacia tu endpoint de pruebas (por ejemplo `http://192.168.x.x`). En producción se omite. |

### Sobre los `<uses-feature>` y el warning de Lint

Cuando declaras un permiso como `CAMERA`, `RECORD_AUDIO` o `ACCESS_FINE_LOCATION`, Android Studio (vía Lint) te muestra un warning del estilo:

> *Permission exists without corresponding hardware `<uses-feature>` tag.*

No es un error de compilación: la app compila igual. El warning te advierte que declaraste un permiso peligroso sin decir explícitamente si el hardware es obligatorio u opcional, y por defecto Android asume que **es obligatorio**. Esto tiene un efecto colateral importante en Google Play:

- **Tablets sin cámara** → no pueden ver tu app en la tienda.
- **Chromebooks sin GPS** → filtrados.
- **Dispositivos sin micrófono** → bloqueados.

La etiqueta `<uses-feature android:name="..." android:required="false" />` le dice al sistema: "*Mi app usa este hardware si está disponible, pero puede funcionar sin él*". Es una declaración de **inclusión**, no de exclusión.

| Atributo | Efecto |
|---|---|
| `required="true"` | La app no funciona sin este hardware. Play Store **filtra** los dispositivos que no lo tengan. |
| `required="false"` | La app prefiere este hardware pero se degrada elegantemente sin él. Permite instalación en **todos** los dispositivos. |

Para nuestro laboratorio **DataDemo** la decisión correcta es siempre `required="false"` porque la app tiene 5 pestañas y solo 2 dependen estrictamente del hardware multimedia. Un usuario sin cámara igual puede usar las pestañas de Sensores, Sincronización y Alertas.

> **En producción real:** si tu app **es** una app de cámara (tipo Instagram), pondrías `required="true"` en `android.hardware.camera` para que Play Store filtre dispositivos incompatibles automáticamente.

Como ahora declaramos `required="false"`, conviene verificar en runtime que el hardware exista antes de usarlo. Ejemplo dentro de `CamaraScreen.kt`:

```kotlin
val tieneCamara = context.packageManager
    .hasSystemFeature(PackageManager.FEATURE_CAMERA_ANY)

if (tieneCamara) {
    // mostrar CameraX
} else {
    Text("Este dispositivo no tiene cámara disponible.")
}
```

### Sobre `foregroundServiceType="location"`

Es un estándar de seguridad de las APIs modernas de Android. Si el sistema operativo detecta que un servicio captura ubicaciones y el manifiesto **no** declara explícitamente el tipo como `"location"`, el kernel termina la aplicación de inmediato por motivos de privacidad.

### ¿Por qué `.services.TrackingService` aparece subrayado en rojo?

Al guardar el manifest verás que `android:name=".services.TrackingService"` queda **subrayado en rojo** con el mensaje:

> *Class referenced in the manifest, `com.illareklab.data_demo.services.TrackingService`, was not found in the project or the libraries.*

Esto es **normal y esperado**: el archivo `TrackingService.kt` aún no existe (lo crearemos en la Sección 6). Android Studio resuelve referencias del manifest en tiempo de edición y avisa cuando la clase no existe todavía.

**No es un error de compilación.** El subrayado **desaparece automáticamente** apenas guardes el archivo `TrackingService.kt` en la Sección 6.

Si el subrayado te molesta visualmente, puedes adelantarte y crear el archivo como stub vacío:

1. Crea el package `services` (lo veremos en la siguiente sección de todas formas).
2. Crea `TrackingService.kt` con este contenido temporal:

```kotlin
package com.illareklab.data_demo.services

import android.app.Service
import android.content.Intent
import android.os.IBinder

class TrackingService : Service() {
    override fun onBind(intent: Intent?): IBinder? = null
}
```

3. Reemplazarás todo el contenido en la Sección 6 con la implementación real.

---

## SECCIÓN 4 — Crear la estructura de carpetas

Vamos a evitar el anti-patrón del código monolítico (clases todopoderosas) organizando todo en capas por responsabilidad. La estructura final será:

```
com.illareklab.data_demo/
├── MainActivity.kt              ← Entry point (≈ 15 líneas)
├── data/
│   └── NetworkManager.kt        ← Caché local + envío HTTP progresivo
├── security/
│   └── PasswordHasher.kt        ← Hashing PBKDF2WithHmacSHA256
├── services/
│   └── TrackingService.kt       ← Foreground Service (GNSS + FLP cada 10s)
├── workers/
│   └── NotificationWorker.kt    ← Tarea diferida de notificaciones
└── ui/
    ├── Navigation.kt            ← Enrutador (login + 5 pestañas)
    └── screens/
        ├── LoginScreen.kt       ← Acceso admin/admin (verificación por hash)
        ├── SensoresScreen.kt    ← Control del servicio de captura
        ├── CamaraScreen.kt      ← Estructura base CameraX
        ├── AudioScreen.kt       ← Estructura base MediaRecorder
        ├── SyncScreen.kt        ← Monitor + sincronización POST animada
        └── AlertasScreen.kt     ← Programación de notificaciones
```

### Crear los packages

En el panel izquierdo (Project), expande `app/src/main/java/com/illareklab/data_demo`. Click derecho sobre `data_demo`:

1. **New → Package** → escribe `data` → Enter.
2. **New → Package** → escribe `security` → Enter.
3. **New → Package** → escribe `services` → Enter.
4. **New → Package** → escribe `workers` → Enter.
5. **New → Package** → escribe `ui` → Enter.
6. Click derecho sobre `ui` → **New → Package** → escribe `screens` → Enter.

> Cuando escribes un nombre con punto (`ui.screens`), Android Studio crea la jerarquía anidada automáticamente.

---

## SECCIÓN 5 — Capa de datos (`data/NetworkManager.kt`)

Esta clase se encarga de tres responsabilidades:

1. Leer el archivo plano `historial_sensores.txt` y transformarlo en una lista de objetos.
2. Enviar cada objeto al servidor mediante HTTP POST.
3. Reescribir el archivo local restando los registros confirmados, para tolerar fallos de red.

Click derecho en `data` → **New → Kotlin Class/File** → `NetworkManager` → tipo **File**.

```kotlin
package com.illareklab.data_demo.data

import android.content.Context
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.io.File
import java.net.HttpURLConnection
import java.net.URL

// Estructura de datos que la UI consumirá de forma reactiva
data class LogMetrica(
    val id: Long,                  // timestamp como identificador único invariable
    val origen: String,            // "GNSS_PURO" o "FLP_INTELLIGENT"
    val coordenadas: String,       // "Lat: -12.04, Lng: -77.04"
    val textoPlanoOriginal: String // línea original tal cual fue escrita
)

class NetworkManager(private val context: Context) {

    private val file = File(context.getExternalFilesDir(null), "historial_sensores.txt")

    // Reemplazar por tu endpoint real o IP local de pruebas
    private val endpoint = "https://webhook.site/REEMPLAZAR-CON-TU-UUID"

    // ── 1. Lectura del historial plano → lista de objetos ────────────
    fun obtenerDatosLocales(): List<LogMetrica> {
        if (!file.exists() || file.length() == 0L) return emptyList()

        return file.readLines().mapNotNull { linea ->
            // Formato esperado: "timestamp | TIPO_SENSOR | Lat: ..., Lng: ..."
            val partes = linea.split(" | ")
            if (partes.size >= 3) {
                LogMetrica(
                    id = partes[0].toLongOrNull() ?: System.currentTimeMillis(),
                    origen = partes[1],
                    coordenadas = partes[2],
                    textoPlanoOriginal = linea
                )
            } else null
        }
    }

    // ── 2. Envío individual asíncrono ────────────────────────────────
    suspend fun enviarRegistroIndividual(metrica: LogMetrica): Boolean =
        withContext(Dispatchers.IO) {
            try {
                val urlConnection = (URL(endpoint).openConnection() as HttpURLConnection).apply {
                    requestMethod = "POST"
                    setRequestProperty("Content-Type", "text/plain; charset=utf-8")
                    doOutput = true
                    connectTimeout = 3000
                    readTimeout = 3000
                }

                urlConnection.outputStream.use { os ->
                    val bytes = metrica.textoPlanoOriginal.toByteArray(Charsets.UTF_8)
                    os.write(bytes, 0, bytes.size)
                }

                val responseCode = urlConnection.responseCode
                urlConnection.disconnect()

                responseCode == HttpURLConnection.HTTP_OK ||
                    responseCode == HttpURLConnection.HTTP_CREATED
            } catch (e: Exception) {
                false
            }
        }

    // ── 3. Reescritura segura del archivo físico ────────────────────
    fun actualizarArchivoLocal(metricasRestantes: List<LogMetrica>) {
        if (metricasRestantes.isEmpty()) {
            file.writeText("") // vaciado total cuando la cola llegó a cero
        } else {
            val nuevoContenido =
                metricasRestantes.joinToString("\n") { it.textoPlanoOriginal } + "\n"
            file.writeText(nuevoContenido)
        }
    }
}
```

### ¿Por qué `Dispatchers.IO`?

Android **bloquea por completo** la aplicación si se realiza una llamada HTTP en el hilo principal (Main Thread). El despachador `Dispatchers.IO` mueve la operación a un pool de hilos secundarios optimizado para entrada/salida, dejando libre el hilo de renderizado.

### ¿Por qué eliminación condicionada y defensiva?

Enviar todo el lote en una sola llamada y borrar el archivo a ciegas es un riesgo: si la red cae a la mitad, se pierden los registros no transmitidos. Procesando **línea por línea** y reescribiendo el archivo después de cada confirmación, garantizamos tolerancia a fallos. Si el Wi-Fi se desconecta, los registros pendientes permanecen intactos en el almacenamiento del dispositivo.

---

## SECCIÓN 6 — Capa de servicios (`services/TrackingService.kt`)

Este servicio corre **en primer plano** y captura coordenadas desde dos fuentes en paralelo:

- **GNSS puro** vía `LocationManager` (chip satelital directo).
- **FLP** (Fused Location Provider) de Google Play Services.

Click derecho en `services` → **New → Kotlin Class/File** → `TrackingService`.

```kotlin
package com.illareklab.data_demo.services

import android.Manifest
import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.Service
import android.content.Intent
import android.content.pm.PackageManager
import android.location.Location
import android.location.LocationListener
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
import java.io.File
import java.io.FileWriter

class TrackingService : Service() {

    companion object {
        private const val CANAL_ID = "track_datademo"
        private const val NOTIF_ID = 200
        private const val INTERVALO_MS = 10_000L  // 10 segundos
    }

    private lateinit var locationManager: LocationManager
    private lateinit var fusedLocationClient: FusedLocationProviderClient
    private lateinit var locationCallback: LocationCallback
    private lateinit var file: File

    // Listener del GNSS puro
    private val gnssListener = LocationListener { loc ->
        escribirRegistro("GNSS_PURO", loc.latitude, loc.longitude)
    }

    override fun onCreate() {
        super.onCreate()
        file = File(getExternalFilesDir(null), "historial_sensores.txt")
        crearCanalNotificacion()
        startForeground(NOTIF_ID, buildNotification())

        locationManager = getSystemService(LOCATION_SERVICE) as LocationManager
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        iniciarCapturas()
        return START_STICKY
    }

    private fun iniciarCapturas() {
        // Verificación defensiva de permisos antes de cualquier suscripción
        val tienePermiso = ContextCompat.checkSelfPermission(
            this, Manifest.permission.ACCESS_FINE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED

        if (!tienePermiso) {
            stopSelf()
            return
        }

        try {
            // 1. GNSS puro de hardware satelital (cada 10 s)
            locationManager.requestLocationUpdates(
                LocationManager.GPS_PROVIDER,
                INTERVALO_MS,
                0f,
                gnssListener
            )

            // 2. Fused Location Provider de Google (cada 10 s)
            val request = LocationRequest.Builder(
                Priority.PRIORITY_HIGH_ACCURACY,
                INTERVALO_MS
            ).build()

            locationCallback = object : LocationCallback() {
                override fun onLocationResult(result: LocationResult) {
                    result.lastLocation?.let { loc ->
                        escribirRegistro("FLP_INTELLIGENT", loc.latitude, loc.longitude)
                    }
                }
            }

            fusedLocationClient.requestLocationUpdates(
                request,
                locationCallback,
                mainLooper
            )
        } catch (e: SecurityException) {
            stopSelf()
        }
    }

    // Escritura sincronizada para evitar colisiones entre hilos GNSS y FLP
    @Synchronized
    private fun escribirRegistro(tipo: String, lat: Double, lng: Double) {
        val linea = "${System.currentTimeMillis()} | $tipo | Lat: $lat, Lng: $lng\n"
        FileWriter(file, true).use { it.write(linea) }
    }

    private fun crearCanalNotificacion() {
        val channel = NotificationChannel(
            CANAL_ID,
            "Tracking DataDemo",
            NotificationManager.IMPORTANCE_LOW
        )
        (getSystemService(NOTIFICATION_SERVICE) as NotificationManager)
            .createNotificationChannel(channel)
    }

    private fun buildNotification(): Notification =
        NotificationCompat.Builder(this, CANAL_ID)
            .setContentTitle("DataDemo: captura en curso")
            .setContentText("Grabando datos de posicionamiento cada 10 s…")
            .setSmallIcon(android.R.drawable.ic_menu_compass)
            .build()

    override fun onDestroy() {
        super.onDestroy()
        try {
            locationManager.removeUpdates(gnssListener)
            fusedLocationClient.removeLocationUpdates(locationCallback)
        } catch (_: Exception) { /* ignorado */ }
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

### ¿Por qué `START_STICKY`?

Obliga al sistema operativo a **revivir y reiniciar** el servicio automáticamente si en algún momento se ve forzado a cerrarlo para liberar memoria RAM. Sin esta bandera, una limpieza del kernel destruiría permanentemente la captura.

### ¿Por qué `@Synchronized` en `escribirRegistro`?

Las lecturas del chip GNSS y las del algoritmo FLP de Google se ejecutan **en paralelo, en hilos separados**. Si ambos intentan escribir simultáneamente en el archivo, se producen *race conditions* que corrompen el contenido. La anotación `@Synchronized` garantiza que solo un hilo a la vez pueda entrar al método.

> **Nota sobre la sintaxis:** En Kotlin, `synchronized` **no es un modificador de función** como en Java. Hay dos formas correctas: la anotación `@Synchronized` sobre el método, o el bloque `synchronized(lock) { ... }` dentro del cuerpo. Escribir `private synchronized fun ...` produce un error de compilación.

---

## SECCIÓN 7 — Capa de tareas diferidas (`workers/NotificationWorker.kt`)

`WorkManager` es la herramienta oficial de Jetpack para tareas que deben ejecutarse **incluso si la app está cerrada**. La usaremos para programar notificaciones temporales.

Click derecho en `workers` → **New → Kotlin Class/File** → `NotificationWorker`.

```kotlin
package com.illareklab.data_demo.workers

import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import androidx.core.app.NotificationCompat
import androidx.work.Worker
import androidx.work.WorkerParameters

class NotificationWorker(
    context: Context,
    params: WorkerParameters
) : Worker(context, params) {

    companion object {
        const val MSG_KEY = "MSG_KEY"
        private const val CANAL_ID = "datademo_alerts"
    }

    override fun doWork(): Result {
        val mensaje = inputData.getString(MSG_KEY) ?: "Alerta activada"

        val manager = applicationContext
            .getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager

        // El canal puede crearse muchas veces sin efectos secundarios
        val channel = NotificationChannel(
            CANAL_ID,
            "Recordatorios",
            NotificationManager.IMPORTANCE_HIGH
        )
        manager.createNotificationChannel(channel)

        val notification = NotificationCompat.Builder(applicationContext, CANAL_ID)
            .setSmallIcon(android.R.drawable.ic_lock_idle_alarm)
            .setContentTitle("Notificación temporal DataDemo")
            .setContentText(mensaje)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .build()

        manager.notify(System.currentTimeMillis().toInt(), notification)
        return Result.success()
    }
}
```

### ¿Por qué WorkManager en vez de un `Handler` o `Timer`?

Un `Handler.postDelayed` o un `Timer` viven dentro del proceso de la app. Si el usuario cierra la app **antes** de que dispare la tarea, la notificación nunca se muestra. `WorkManager` delega la programación al sistema operativo (a través de `JobScheduler`), garantizando la ejecución fielmente aunque el proceso ya no exista en memoria.

---

## SECCIÓN 8 — Enrutador de navegación (`ui/Navigation.kt`)

Este componente administra dos niveles de navegación:

1. **Navegación global:** entre la pantalla de login y la app principal.
2. **Navegación de pestañas:** las 5 secciones internas con `NavigationBar`.

Click derecho en `ui` → **New → Kotlin Class/File** → `Navigation`.

```kotlin
package com.illareklab.data_demo.ui

import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Call
import androidx.compose.material.icons.filled.CloudSync
import androidx.compose.material.icons.filled.Home
import androidx.compose.material.icons.filled.Notifications
import androidx.compose.material.icons.filled.PlayArrow
import androidx.compose.material3.Icon
import androidx.compose.material3.NavigationBar
import androidx.compose.material3.NavigationBarItem
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.illareklab.data_demo.ui.screens.AlertasScreen
import com.illareklab.data_demo.ui.screens.AudioScreen
import com.illareklab.data_demo.ui.screens.CamaraScreen
import com.illareklab.data_demo.ui.screens.LoginScreen
import com.illareklab.data_demo.ui.screens.SensoresScreen
import com.illareklab.data_demo.ui.screens.SyncScreen

@Composable
fun Navigation() {
    val globalNavController = rememberNavController()

    NavHost(
        navController = globalNavController,
        startDestination = "login"
    ) {
        composable("login") {
            LoginScreen(onLoginSuccess = {
                globalNavController.navigate("main_app") {
                    // Destruye el login del historial → "atrás" cierra la app
                    popUpTo("login") { inclusive = true }
                }
            })
        }
        composable("main_app") { MainNavigationScreen() }
    }
}

@Composable
fun MainNavigationScreen() {
    val tabsNavController = rememberNavController()
    var selectedTab by remember { mutableIntStateOf(0) }

    Scaffold(
        bottomBar = {
            NavigationBar {
                NavigationBarItem(
                    selected = selectedTab == 0,
                    onClick = {
                        selectedTab = 0
                        tabsNavController.navigate("sensores")
                    },
                    icon = { Icon(Icons.Default.Home, contentDescription = null) },
                    label = { Text("Sensores") }
                )
                NavigationBarItem(
                    selected = selectedTab == 1,
                    onClick = {
                        selectedTab = 1
                        tabsNavController.navigate("camara")
                    },
                    icon = { Icon(Icons.Default.PlayArrow, contentDescription = null) },
                    label = { Text("Cámara") }
                )
                NavigationBarItem(
                    selected = selectedTab == 2,
                    onClick = {
                        selectedTab = 2
                        tabsNavController.navigate("audio")
                    },
                    icon = { Icon(Icons.Default.Call, contentDescription = null) },
                    label = { Text("Audio") }
                )
                NavigationBarItem(
                    selected = selectedTab == 3,
                    onClick = {
                        selectedTab = 3
                        tabsNavController.navigate("sync")
                    },
                    icon = { Icon(Icons.Default.CloudSync, contentDescription = null) },
                    label = { Text("Sincronizar") }
                )
                NavigationBarItem(
                    selected = selectedTab == 4,
                    onClick = {
                        selectedTab = 4
                        tabsNavController.navigate("alertas")
                    },
                    icon = { Icon(Icons.Default.Notifications, contentDescription = null) },
                    label = { Text("Alertas") }
                )
            }
        }
    ) { paddingValues ->
        NavHost(
            navController = tabsNavController,
            startDestination = "sensores",
            modifier = Modifier.padding(paddingValues)
        ) {
            composable("sensores") { SensoresScreen() }
            composable("camara") { CamaraScreen() }
            composable("audio") { AudioScreen() }
            composable("sync") { SyncScreen() }
            composable("alertas") { AlertasScreen() }
        }
    }
}
```

### ¿Por qué `popUpTo("login") { inclusive = true }`?

Evita que, una vez iniciada la sesión, el usuario pueda regresar al login al presionar el botón físico/gestual de "atrás" de Android. Al marcarlo como `inclusive = true`, **limpiamos la pila completa**, haciendo que el sistema cierre la app correctamente si se intenta retroceder desde la pantalla principal.

---

## SECCIÓN 9 — Capa de seguridad y pantalla de acceso

### 9.1 ¿Por qué no comparar contraseñas en texto plano?

El instinto natural es escribir `if (password == "admin")`. Esto es **incorrecto** por dos motivos:

1. **La contraseña queda visible en el binario.** Cualquier persona con acceso al APK puede descompilarlo con herramientas como `apktool` o `jadx` y leer la cadena literal en segundos.
2. **No es una práctica transferible.** En el momento que la app se conecte a un backend real, las contraseñas se reciben **ya hasheadas** y la comparación debe hacerse contra el hash, no contra texto plano.

### 9.2 ¿Qué algoritmo de hashing se usa en Android?

| Algoritmo | ¿Sirve para contraseñas? | Comentario |
|---|---|---|
| MD5, SHA-1 | No | Rotos criptográficamente desde hace años. |
| SHA-256 puro | No | No es "inseguro" como hash general, pero es **demasiado rápido**: una GPU moderna calcula miles de millones de hashes por segundo, lo que permite atacar contraseñas por fuerza bruta. |
| **PBKDF2WithHmacSHA256** | **Sí** | Estándar recomendado por OWASP y Google. Aplica SHA-256 **miles de veces** sobre la contraseña + un *salt*, haciendo cada intento de fuerza bruta computacionalmente costoso. Viene **integrado en Android** vía `javax.crypto.SecretKeyFactory`, sin necesidad de dependencias externas. |
| BCrypt / Argon2 | Sí (mejores) | Más robustos aún, pero requieren librerías externas (`jBCrypt`, `argon2-jvm`). Para este laboratorio PBKDF2 es suficiente y nativo. |

> **Resumen:** `PBKDF2WithHmacSHA256` con salt + ~120 000 iteraciones es la recomendación oficial de **OWASP 2023** para almacenamiento de contraseñas en aplicaciones móviles que solo usan librerías estándar de la plataforma.

### 9.3 Crear `security/PasswordHasher.kt`

Click derecho en `security` → **New → Kotlin Class/File** → `PasswordHasher` → tipo **Object**.

```kotlin
package com.illareklab.data_demo.security

import javax.crypto.SecretKeyFactory
import javax.crypto.spec.PBEKeySpec

/**
 * Hashing de contraseñas con PBKDF2WithHmacSHA256.
 *
 * - 120 000 iteraciones (recomendación OWASP 2023 para SHA-256).
 * - Salida de 256 bits (32 bytes), serializada en hexadecimal.
 * - Comparación en tiempo constante (constantTimeEquals) para evitar
 *   ataques de temporización (timing attacks).
 */
object PasswordHasher {

    private const val ALGORITMO = "PBKDF2WithHmacSHA256"
    private const val ITERACIONES = 120_000
    private const val LONGITUD_HASH_BITS = 256

    /**
     * Calcula el hash de [password] usando el [salt] proporcionado y lo
     * devuelve como cadena hexadecimal en minúsculas.
     */
    fun hash(password: String, salt: ByteArray): String {
        val spec = PBEKeySpec(
            password.toCharArray(),
            salt,
            ITERACIONES,
            LONGITUD_HASH_BITS
        )
        val factory = SecretKeyFactory.getInstance(ALGORITMO)
        val bytes = factory.generateSecret(spec).encoded

        // Borrado defensivo del array intermedio en memoria
        spec.clearPassword()

        return bytes.joinToString("") { "%02x".format(it) }
    }

    /**
     * Comparación de hashes en **tiempo constante**: el tiempo de respuesta
     * no depende de en qué byte difieren los strings, lo que evita que un
     * atacante deduzca el hash midiendo latencias.
     */
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

> **¿Por qué `constantTimeEquals` y no `==`?** El operador `==` (que en Kotlin invoca a `String.equals`) **devuelve `false` apenas encuentra el primer byte distinto**. Un atacante puede medir esos microsegundos de diferencia y deducir, byte por byte, cuál es el hash correcto. La comparación bit a bit recorriendo todos los bytes elimina esta filtración temporal.

### 9.4 Crear `ui/screens/LoginScreen.kt`

Click derecho en `ui.screens` → **New → Kotlin Class/File** → `LoginScreen`.

```kotlin
package com.illareklab.data_demo.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.unit.dp
import com.illareklab.data_demo.security.PasswordHasher
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext

// ── Credenciales almacenadas (nunca en texto plano) ─────────────────
// Usuario válido (en producción vendría de una base de datos)
private const val USUARIO_VALIDO = "admin"

// Salt de 16 bytes ASCII: "DataDemoSalt0123"
// En producción, cada usuario tendría su propio salt aleatorio.
private val SALT_DEMO: ByteArray = byteArrayOf(
    0x44, 0x61, 0x74, 0x61, 0x44, 0x65, 0x6d, 0x6f,
    0x53, 0x61, 0x6c, 0x74, 0x30, 0x31, 0x32, 0x33
)

// Hash precomputado de "admin" con el SALT_DEMO + 120 000 iteraciones.
// Para regenerarlo, ejecute PasswordHasher.hash("nuevoPass", SALT_DEMO)
// una vez en el debugger y pegue el resultado aquí.
private const val HASH_PASSWORD_ADMIN =
    "70f5d2c5f1b01c28f3750bfd7d6a5b9e2fd1e80b99286319bfd32626e29f40be"

@Composable
fun LoginScreen(onLoginSuccess: () -> Unit) {
    val scope = rememberCoroutineScope()

    var usuario by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var errorMensaje by remember { mutableStateOf("") }
    var verificando by remember { mutableStateOf(false) }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(32.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            "Sistema DataDemo",
            style = MaterialTheme.typography.headlineLarge,
            color = MaterialTheme.colorScheme.primary
        )
        Spacer(modifier = Modifier.height(32.dp))

        OutlinedTextField(
            value = usuario,
            onValueChange = { usuario = it },
            label = { Text("Usuario") },
            singleLine = true,
            enabled = !verificando,
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(modifier = Modifier.height(8.dp))

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Contraseña") },
            singleLine = true,
            enabled = !verificando,
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )

        if (errorMensaje.isNotEmpty()) {
            Spacer(modifier = Modifier.height(8.dp))
            Text(errorMensaje, color = MaterialTheme.colorScheme.error)
        }

        Spacer(modifier = Modifier.height(24.dp))
        Button(
            onClick = {
                errorMensaje = ""
                verificando = true
                scope.launch {
                    // El hashing PBKDF2 con 120 000 iteraciones tarda
                    // ~100-300 ms. Se ejecuta en Dispatchers.Default
                    // para no bloquear el hilo de UI.
                    val coincide = withContext(Dispatchers.Default) {
                        val hashIngresado = PasswordHasher.hash(password, SALT_DEMO)
                        usuario == USUARIO_VALIDO &&
                            PasswordHasher.constantTimeEquals(
                                hashIngresado,
                                HASH_PASSWORD_ADMIN
                            )
                    }
                    verificando = false
                    if (coincide) {
                        onLoginSuccess()
                    } else {
                        errorMensaje = "Credenciales incorrectas. Pruebe admin/admin."
                    }
                }
            },
            enabled = !verificando && usuario.isNotBlank() && password.isNotBlank(),
            modifier = Modifier
                .fillMaxWidth()
                .height(50.dp)
        ) {
            Text(if (verificando) "Verificando…" else "Ingresar")
        }
    }
}
```

### 9.5 ¿Cómo regenerar el hash si cambia la contraseña?

El valor `HASH_PASSWORD_ADMIN` está **precomputado**. Si quiere cambiar la contraseña (por ejemplo a `"miClaveSegura2026"`), siga estos pasos:

1. Ponga un breakpoint temporal al inicio de `LoginScreen`.
2. Ejecute la app y abra la **Evaluate Expression** de Android Studio (`Alt + F8`).
3. Escriba:
   ```kotlin
   PasswordHasher.hash("miClaveSegura2026", SALT_DEMO)
   ```
4. Copie la cadena resultante y reemplace el valor de `HASH_PASSWORD_ADMIN`.
5. Remueva el breakpoint y vuelva a ejecutar.

> **Nota didáctica:** En esta versión usamos un salt fijo y un único usuario porque el laboratorio se enfoca en captura de sensores, no en gestión de identidad. En una app real:
> - Cada usuario tiene un **salt aleatorio único** (mínimo 16 bytes) generado con `SecureRandom`.
> - El hash + salt se almacenan en una base de datos (Room, SQLite, o backend remoto).
> - Las contraseñas **jamás** viajan ni se guardan en texto plano, ni siquiera por instantes.
> - Para apps verdaderamente sensibles, considere `androidx.security.crypto.EncryptedSharedPreferences` o el Android Keystore.

---

## SECCIÓN 10 — Pantalla de control de sensores (`ui/screens/SensoresScreen.kt`)

Esta pantalla incluye **manejo de permisos en runtime**: en Android 6.0+ los permisos peligrosos no se conceden automáticamente solo por declararlos en el manifest; deben pedirse al usuario en el momento de uso.

Click derecho en `ui.screens` → **New → Kotlin Class/File** → `SensoresScreen`.

```kotlin
package com.illareklab.data_demo.ui.screens

import android.Manifest
import android.content.Intent
import android.os.Build
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import com.google.accompanist.permissions.ExperimentalPermissionsApi
import com.google.accompanist.permissions.rememberMultiplePermissionsState
import com.illareklab.data_demo.services.TrackingService

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun SensoresScreen() {
    val context = LocalContext.current

    // Lista dinámica de permisos según versión de Android
    val permisos = buildList {
        add(Manifest.permission.ACCESS_FINE_LOCATION)
        add(Manifest.permission.ACCESS_COARSE_LOCATION)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            add(Manifest.permission.POST_NOTIFICATIONS)
        }
    }

    val estadoPermisos = rememberMultiplePermissionsState(permissions = permisos)

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            "Pestaña 1 — Hardware de posicionamiento",
            style = MaterialTheme.typography.headlineSmall
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(
            "Captura continua GNSS + FLP cada 10 segundos.",
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(modifier = Modifier.height(24.dp))

        if (estadoPermisos.allPermissionsGranted) {
            Button(
                onClick = {
                    val intent = Intent(context, TrackingService::class.java)
                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                        context.startForegroundService(intent)
                    } else {
                        context.startService(intent)
                    }
                },
                modifier = Modifier
                    .fillMaxWidth()
                    .height(50.dp)
            ) {
                Text("Iniciar captura continua (10 s)")
            }

            Spacer(modifier = Modifier.height(12.dp))

            OutlinedButton(
                onClick = {
                    context.stopService(Intent(context, TrackingService::class.java))
                },
                modifier = Modifier
                    .fillMaxWidth()
                    .height(50.dp)
            ) {
                Text("Detener captura")
            }
        } else {
            Text(
                "Esta pantalla necesita permisos de ubicación y notificación.",
                style = MaterialTheme.typography.bodyMedium
            )
            Spacer(modifier = Modifier.height(12.dp))
            Button(
                onClick = { estadoPermisos.launchMultiplePermissionRequest() },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Conceder permisos")
            }
        }
    }
}
```

### ¿Por qué `startForegroundService` y no `startService`?

A partir de Android 8.0 (API 26), el sistema **no permite** que una app en segundo plano inicie un servicio normal. La invocación correcta es `startForegroundService`, y el servicio **dispone de 5 segundos** para llamar a `startForeground(...)` y mostrar su notificación; si no lo hace, el OS lanza `ForegroundServiceDidNotStartInTimeException` y termina la app.

---

## SECCIÓN 11 — Pantalla de sincronización animada (`ui/screens/SyncScreen.kt`)

Esta es la pantalla más importante del laboratorio. Combina:

- Lectura asíncrona de la caché local al abrir la pestaña.
- Lista reactiva (`mutableStateListOf`) que Compose observa automáticamente.
- Envío secuencial por HTTP POST con feedback de progreso.
- Eliminación visual elemento por elemento con animación `shrinkVertically`.
- Reescritura del archivo físico tras cada confirmación del servidor.

Click derecho en `ui.screens` → **New → Kotlin Class/File** → `SyncScreen`.

```kotlin
package com.illareklab.data_demo.ui.screens

import androidx.compose.animation.AnimatedVisibility
import androidx.compose.animation.core.tween
import androidx.compose.animation.shrinkVertically
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.CloudUpload
import androidx.compose.material.icons.filled.DeleteOutline
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateListOf
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import com.illareklab.data_demo.data.LogMetrica
import com.illareklab.data_demo.data.NetworkManager
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

@Composable
fun SyncScreen() {
    val context = LocalContext.current
    val scope = rememberCoroutineScope()
    val networkManager = remember { NetworkManager(context) }

    // Lista reactiva observable directamente por Compose
    val listaMetricas = remember { mutableStateListOf<LogMetrica>() }
    var estaSincronizando by remember { mutableStateOf(false) }
    var mensajeEstado by remember { mutableStateOf("Caché local sincronizable") }

    // Carga asíncrona al abrir la pestaña
    LaunchedEffect(Unit) {
        listaMetricas.clear()
        listaMetricas.addAll(networkManager.obtenerDatosLocales())
        if (listaMetricas.isEmpty()) {
            mensajeEstado = "No existen registros guardados localmente."
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        Text(
            "Monitor de almacenamiento",
            style = MaterialTheme.typography.headlineMedium
        )
        Text(
            "Registros temporales en la memoria del dispositivo",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.outline
        )

        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                estaSincronizando = true
                scope.launch {
                    val colaOriginal = ArrayList(listaMetricas)

                    for (metrica in colaOriginal) {
                        mensajeEstado = "Transmitiendo ID: ${metrica.id}…"
                        val exito = networkManager.enviarRegistroIndividual(metrica)

                        if (exito) {
                            // Delay perceptible para evidenciar la animación
                            delay(300)

                            // Remueve de la UI → gatilla el refresco asíncrono
                            listaMetricas.remove(metrica)

                            // Limpia el archivo físico inmediatamente
                            networkManager.actualizarArchivoLocal(listaMetricas)
                        } else {
                            mensajeEstado =
                                "Transmisión fallida. Sincronización pausada para proteger la caché."
                            break
                        }
                    }

                    estaSincronizando = false
                    if (listaMetricas.isEmpty()) {
                        mensajeEstado = "Almacenamiento temporal completamente limpio."
                    }
                }
            },
            enabled = listaMetricas.isNotEmpty() && !estaSincronizando,
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(Icons.Default.CloudUpload, contentDescription = null)
            Spacer(modifier = Modifier.width(8.dp))
            Text(
                if (estaSincronizando) "Vaciando memoria…"
                else "Sincronizar y limpiar caché (POST)"
            )
        }

        Spacer(modifier = Modifier.height(12.dp))
        Text(
            mensajeEstado,
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.primary
        )
        Spacer(modifier = Modifier.height(16.dp))

        // Lista reactiva con llaves únicas → Compose sabe exactamente qué item animar
        LazyColumn(
            modifier = Modifier.fillMaxSize(),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(items = listaMetricas, key = { it.id }) { metrica ->
                AnimatedVisibility(
                    visible = true,
                    exit = shrinkVertically(animationSpec = tween(durationMillis = 300))
                ) {
                    Card(
                        modifier = Modifier.fillMaxWidth(),
                        colors = CardDefaults.cardColors(
                            containerColor = MaterialTheme.colorScheme.surfaceVariant
                        )
                    ) {
                        Row(
                            modifier = Modifier.padding(16.dp),
                            verticalAlignment = Alignment.CenterVertically
                        ) {
                            Column(modifier = Modifier.weight(1f)) {
                                Text(
                                    metrica.origen,
                                    style = MaterialTheme.typography.titleSmall,
                                    color = MaterialTheme.colorScheme.secondary
                                )
                                Text(
                                    metrica.coordenadas,
                                    style = MaterialTheme.typography.bodyMedium
                                )
                            }
                            Icon(
                                Icons.Default.DeleteOutline,
                                contentDescription = null,
                                tint = MaterialTheme.colorScheme.error.copy(alpha = 0.5f)
                            )
                        }
                    }
                }
            }
        }
    }
}
```

### ¿Por qué `mutableStateListOf` y `key = { it.id }`?

Las listas comunes de Kotlin (`List<T>`, `ArrayList<T>`) **no notifican** a Compose cuando su contenido cambia. Una `mutableStateListOf<T>` sí lo hace: cada `add`/`remove` dispara la recomposición automáticamente.

El parámetro `key = { it.id }` indica a Compose la **identidad estable** de cada item. Sin esto, al eliminar la posición 2 de la lista, Compose interpretaría que cambiaron los items de las posiciones 3, 4, 5… (porque ahora son 2, 3, 4…) y los animaría a todos. Con la llave, sabe que solo desapareció el item con `id = X`, y aplica el `shrinkVertically` únicamente a esa tarjeta sin que parpadeen las demás.

### ¿Por qué el `delay(300)`?

Es puramente didáctico: hace visible la animación de colapso en el aula. En producción se omite para maximizar el throughput de la sincronización.

---

## SECCIÓN 12 — Pantallas restantes (estructura base)

Las tres pantallas siguientes se incluyen como placeholders para completar la compilación. Cada una puede expandirse después con su lógica específica (CameraX, MediaRecorder, encolado de Workers).

### 12.1 `CamaraScreen.kt`

```kotlin
package com.illareklab.data_demo.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun CamaraScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            "Pestaña 2 — Cámara",
            style = MaterialTheme.typography.headlineSmall
        )
        Text(
            "Aquí integrará el PreviewView de CameraX para captura de foto y video.",
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}
```

### 12.2 `AudioScreen.kt`

```kotlin
package com.illareklab.data_demo.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun AudioScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            "Pestaña 3 — Audio",
            style = MaterialTheme.typography.headlineSmall
        )
        Text(
            "Aquí integrará MediaRecorder para captura de audio del micrófono.",
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}
```

### 12.3 `AlertasScreen.kt`

Esta pantalla sí incluye una implementación funcional: encolar un `NotificationWorker` con un mensaje personalizado, programado para ejecutarse 10 segundos después.

```kotlin
package com.illareklab.data_demo.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import androidx.work.workDataOf
import com.illareklab.data_demo.workers.NotificationWorker
import java.util.concurrent.TimeUnit

@Composable
fun AlertasScreen() {
    val context = LocalContext.current
    var mensaje by remember { mutableStateOf("") }
    var confirmacion by remember { mutableStateOf("") }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Top
    ) {
        Text(
            "Pestaña 5 — Alertas diferidas",
            style = MaterialTheme.typography.headlineSmall
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(
            "Programa una notificación que se dispara 10 segundos después, incluso si cierra la app.",
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(modifier = Modifier.height(16.dp))

        OutlinedTextField(
            value = mensaje,
            onValueChange = { mensaje = it },
            label = { Text("Mensaje de la alerta") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(modifier = Modifier.height(16.dp))

        Button(
            onClick = {
                val request = OneTimeWorkRequestBuilder<NotificationWorker>()
                    .setInitialDelay(10, TimeUnit.SECONDS)
                    .setInputData(workDataOf(NotificationWorker.MSG_KEY to mensaje))
                    .build()
                WorkManager.getInstance(context).enqueue(request)
                confirmacion = "Alerta encolada. Llegará en 10 s aunque cierre la app."
            },
            enabled = mensaje.isNotBlank(),
            modifier = Modifier
                .fillMaxWidth()
                .height(50.dp)
        ) {
            Text("Programar notificación")
        }

        if (confirmacion.isNotEmpty()) {
            Spacer(modifier = Modifier.height(12.dp))
            Text(
                confirmacion,
                color = MaterialTheme.colorScheme.primary,
                style = MaterialTheme.typography.bodyMedium
            )
        }
    }
}
```

---

## SECCIÓN 13 — Simplificar `MainActivity.kt`

Abre `MainActivity.kt` (en la raíz del paquete) y reemplaza **todo** su contenido por esta versión minimalista:

```kotlin
package com.illareklab.data_demo

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.MaterialTheme
import com.illareklab.data_demo.ui.Navigation

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                // El entry point cede el control al módulo de enrutamiento gráfico
                Navigation()
            }
        }
    }
}
```

> **Separación de responsabilidades:** `MainActivity` ya no decide qué pantalla mostrar ni gestiona estado. Solo aplica el tema y delega al enrutador. Esta es la clave de la arquitectura modular: cada capa hace una sola cosa.

---

## SECCIÓN 14 — Ejecutar y verificar

### 14.1 Build completo

**Build → Rebuild Project**. Esto te dirá si hay errores antes de ejecutar.

### 14.2 Ejecutar

1. Si no tienes emulador: **Tools → Device Manager → Create Virtual Device** → elige **Pixel 7** + **API 34** (Android 14).
2. Click ▶ **Run** (o `Shift+F10`).
3. Si pruebas en emulador, activa ubicación simulada: **Extended controls → Location → Set location**.

### 14.3 Comportamiento esperado

| Acción | Resultado esperado |
|---|---|
| Abrir la app | Aparece la pantalla **Login** centrada con el título "Sistema DataDemo". |
| Ingresar credenciales incorrectas | Aparece mensaje rojo "Credenciales incorrectas. Pruebe admin/admin." |
| Ingresar `admin` / `admin` y tap "Ingresar" | Por ~100-300 ms el botón muestra "Verificando…" (mientras se calcula el PBKDF2), luego salta a la app principal con la pestaña **Sensores** activa. |
| Tap "atrás" desde la pantalla principal | La app se cierra (no vuelve al login). |
| Tap "Conceder permisos" en Sensores | El sistema muestra el diálogo nativo de permisos. |
| Tap "Iniciar captura continua" | Aparece notificación persistente "DataDemo: captura en curso" en la barra de estado. |
| Esperar 30 segundos | El archivo `historial_sensores.txt` contiene al menos 3 líneas de cada origen. |
| Cambiar a pestaña **Sincronizar** | Se muestra la lista de capturas como tarjetas. |
| Tap "Sincronizar y limpiar caché" | Las tarjetas desaparecen una por una con animación de colapso. |
| Cambiar a pestaña **Alertas**, escribir mensaje y programar | Tras 10 segundos llega una notificación de alta prioridad. |

### 14.4 Inspeccionar el archivo capturado

Para verificar el contenido del archivo `historial_sensores.txt`:

1. **View → Tool Windows → Device Explorer**.
2. Navega a `/storage/emulated/0/Android/data/com.illareklab.data_demo/files/`.
3. Doble click en `historial_sensores.txt`.

Verás líneas con el formato:

```
1716240000123 | GNSS_PURO | Lat: -12.046, Lng: -77.043
1716240000456 | FLP_INTELLIGENT | Lat: -12.046, Lng: -77.042
```

---

## Troubleshooting

| Problema | Causa probable | Solución |
|---|---|---|
| `Unresolved reference: Icons` o `Icons.Default.CloudSync` en rojo | Falta la dependencia de iconos extendidos. | Agregar `implementation("androidx.compose.material:material-icons-extended:1.6.7")` y sincronizar. |
| Error `synchronized is not a modifier` | `synchronized` se usó como modificador de función, sintaxis válida solo en Java. | Reemplazar `private synchronized fun X(...)` por `@Synchronized private fun X(...)`. |
| Error `16dp` o `Unresolved reference: dp` | Se escribió `16dp` (junto) en vez de `16.dp` (con punto). | Reemplazar todos los `Ndp` por `N.dp` y verificar el import `androidx.compose.ui.unit.dp`. |
| Error `onResult does not override anything` en `LocationCallback` | El callback se llama `onLocationResult`, no `onResult`. | Cambiar la firma a `override fun onLocationResult(result: LocationResult)`. |
| La app crashea con `ForegroundServiceDidNotStartInTimeException` | El servicio no llamó a `startForeground(...)` en los 5 segundos posteriores a su arranque. | Mover la llamada `startForeground(NOTIF_ID, buildNotification())` a `onCreate()` (no a `onStartCommand`). |
| La app crashea con `SecurityException: Need ACCESS_FINE_LOCATION` | Se inició el servicio antes de que el usuario otorgue permisos. | Verificar `estadoPermisos.allPermissionsGranted` antes del `startForegroundService(...)`. |
| El servicio arranca pero no escribe nada en el archivo | El emulador no tiene ubicación simulada activada. | **Extended controls → Location** → escribe coordenadas y presiona **Send**. |
| Las tarjetas de `SyncScreen` no se animan al eliminarse | Falta el parámetro `key = { it.id }` en `items(...)`. | Agregar la lambda de llave para que Compose distinga cada item. |
| `HTTP POST` siempre devuelve `false` | URL del endpoint inválida, o falta el permiso `INTERNET`. | Verificar `<uses-permission android:name="android.permission.INTERNET" />` y que la URL sea accesible desde el dispositivo. |
| Conexión HTTP simple falla con `CLEARTEXT communication not permitted` | Android 9+ bloquea por defecto el tráfico HTTP sin TLS. | Agregar `android:usesCleartextTraffic="true"` al tag `<application>` del manifest. |
| La notificación del Foreground Service no aparece en Android 13+ | Falta el permiso `POST_NOTIFICATIONS`. | Solicitarlo en runtime junto con los de ubicación (ya incluido en `SensoresScreen`). |
| `WorkManager` no dispara la notificación | El dispositivo está en **modo ahorro de batería** extremo. | Quitar la app de la lista de "optimización de batería" en Configuración → Apps → DataDemo → Batería. |
| `popUpTo` no limpia el login | Falta `inclusive = true`. | Asegurarse que `popUpTo("login") { inclusive = true }` esté presente en el `navigate`. |
| Warning `Permission exists without corresponding hardware <uses-feature> tag` subrayado en rojo/amarillo | Lint detecta un permiso peligroso sin su declaración de hardware correspondiente. | Agregar el bloque de `<uses-feature android:required="false" />` después de los `<uses-permission>` (ver Sección 3). El warning es informativo, **no impide compilar**, pero sí filtra dispositivos en Google Play. |
| `.services.TrackingService` subrayado en rojo en el manifest | El archivo `TrackingService.kt` todavía no existe (se crea en Sección 6). | Avanzar con la guía hasta la Sección 6: al guardar el archivo, el subrayado desaparece. O crear primero un stub vacío (ver final de Sección 3). |
| El login responde con "Credenciales incorrectas" usando `admin/admin` | El `SALT_DEMO` o el `HASH_PASSWORD_ADMIN` en `LoginScreen.kt` no coinciden con los valores precomputados. | Verificar que el salt sea exactamente `byteArrayOf(0x44, 0x61, 0x74, 0x61, 0x44, 0x65, 0x6d, 0x6f, 0x53, 0x61, 0x6c, 0x74, 0x30, 0x31, 0x32, 0x33)` y el hash exactamente `70f5d2c5f1b01c28f3750bfd7d6a5b9e2fd1e80b99286319bfd32626e29f40be`. |
| El botón "Ingresar" congela la app por 1-2 segundos | Se está ejecutando `PasswordHasher.hash(...)` en el hilo principal. | Envolver la llamada en `withContext(Dispatchers.Default) { ... }` como muestra el código de Sección 9.4. |
| `NoSuchAlgorithmException: PBKDF2WithHmacSHA256` | Equipo con una versión muy antigua de Android (pre-API 26) o emulador roto. | Verificar `minSdk = 26`. El algoritmo está disponible nativamente desde Android 8.0. |
| Sigue sin compilar después de varios intentos | Caché de Android Studio corrupto. | **File → Invalidate Caches… → Invalidate and Restart**. |

---

## Checklist final

- [ ] Proyecto creado con template **Empty Activity** (con Compose).
- [ ] `Minimum SDK` configurado en **API 26**.
- [ ] Dependencia `navigation-compose:2.7.7` agregada.
- [ ] Dependencia `play-services-location:21.2.0` agregada.
- [ ] Dependencia `work-runtime-ktx:2.9.0` agregada.
- [ ] 5 dependencias de CameraX agregadas.
- [ ] Dependencia `material-icons-extended:1.6.7` agregada.
- [ ] Dependencia `accompanist-permissions:0.34.0` agregada.
- [ ] Permisos `ACCESS_FINE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`, `POST_NOTIFICATIONS`, `INTERNET` declarados en el manifest.
- [ ] Service registrado con `foregroundServiceType="location"`.
- [ ] Bloque de `<uses-feature>` con `required="false"` agregado para `camera`, `microphone`, `location.gps`, `bluetooth_le` y `wifi`.
- [ ] Atributo `usesCleartextTraffic="true"` agregado al `<application>` (solo si se usa HTTP).
- [ ] Packages `data`, `services`, `workers`, `ui`, `ui.screens` creados.
- [ ] `NetworkManager.kt` con `LogMetrica`, `obtenerDatosLocales`, `enviarRegistroIndividual`, `actualizarArchivoLocal`.
- [ ] `TrackingService.kt` con `@Synchronized` en `escribirRegistro`.
- [ ] `NotificationWorker.kt` extendiendo `Worker`.
- [ ] `Navigation.kt` con `NavHost` global + `MainNavigationScreen` con 5 pestañas.
- [ ] `PasswordHasher.kt` con `PBKDF2WithHmacSHA256`, 120 000 iteraciones y `constantTimeEquals`.
- [ ] `LoginScreen.kt` que verifica la contraseña **comparando hashes** (nunca texto plano) sobre `Dispatchers.Default`.
- [ ] `SensoresScreen.kt` con manejo de permisos en runtime.
- [ ] `SyncScreen.kt` con `mutableStateListOf` + `key = { it.id }` + `AnimatedVisibility`.
- [ ] `CamaraScreen.kt`, `AudioScreen.kt` (estructura base) creados.
- [ ] `AlertasScreen.kt` con encolado de `OneTimeWorkRequest`.
- [ ] `MainActivity.kt` reducido a ~15 líneas.
- [ ] App compila sin errores ni warnings rojos.
- [ ] Login funciona y limpia el back stack.
- [ ] Las 5 pestañas son navegables desde la barra inferior.
- [ ] El servicio escribe líneas cada 10 s en `historial_sensores.txt`.
- [ ] La pestaña Sincronizar elimina tarjetas con animación tras cada POST.
- [ ] La pestaña Alertas dispara la notificación a los 10 s.

---

## Ejercicios sugeridos (para entrega)

### Nivel 1 — Personalización

1. Cambia el **endpoint** de `NetworkManager` por un servicio real de prueba (`https://webhook.site` te asigna una URL única). Captura las primeras 5 transmisiones exitosas.
2. Cambia la contraseña `admin` por una **tuya propia** regenerando el hash con `PasswordHasher.hash(...)` (ver Sección 9.5) y actualizando `HASH_PASSWORD_ADMIN`. Documenta el proceso con captura del Evaluate Expression.

### Nivel 2 — Funcionalidad

3. Implementa el **PreviewView** real de CameraX en `CamaraScreen` con un botón para tomar foto y guardarla en almacenamiento externo.
4. Implementa la **grabación de audio** con `MediaRecorder` en `AudioScreen` y muestra el tiempo transcurrido en pantalla.
5. Agrega un **switch global** en `SensoresScreen` que permita elegir entre intervalos de 5, 10 o 30 segundos.

### Nivel 3 — Avanzado

6. Reemplaza el archivo plano `historial_sensores.txt` por una base de datos **Room** y adapta `NetworkManager` para consultar la tabla.
7. Implementa **reintentos exponenciales** en `enviarRegistroIndividual` (1 s, 2 s, 4 s) antes de dar la transmisión por fallida.
8. Convierte `NotificationWorker` en un **PeriodicWorkRequest** cada 15 minutos que solo dispare la notificación si hay registros pendientes en la caché.
9. Usa `ViewModel` y `StateFlow` para compartir el estado de la lista de capturas entre `SensoresScreen` y `SyncScreen`, eliminando la lectura redundante del archivo.
10. Implementa **registro de nuevos usuarios** con `EncryptedSharedPreferences` (de `androidx.security.crypto`): cada usuario obtiene un salt aleatorio único generado con `SecureRandom.getInstanceStrong()`, y tanto el salt como el hash se persisten cifrados con una llave del Android Keystore.

---

## Recursos

| Recurso | URL |
|---|---|
| Foreground Services (oficial) | https://developer.android.com/develop/background-work/services/foreground-services |
| Fused Location Provider | https://developer.android.com/develop/sensors-and-location/location/request-updates |
| WorkManager | https://developer.android.com/develop/background-work/background-tasks/persistent |
| CameraX | https://developer.android.com/training/camerax |
| Navigation Compose | https://developer.android.com/jetpack/compose/navigation |
| Accompanist Permissions | https://google.github.io/accompanist/permissions/ |
| Webhook.site (endpoint de pruebas) | https://webhook.site |
| OWASP Password Storage Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html |
| SecretKeyFactory (Android docs) | https://developer.android.com/reference/javax/crypto/SecretKeyFactory |
| EncryptedSharedPreferences | https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences |

---

## Entregable

Sube a tu repositorio GitHub:

1. **Carpeta completa del proyecto** (sin `build/`, `.gradle/`, `.idea/`).
2. **Capturas de pantalla** de:
   - Pantalla Login.
   - Pestaña Sensores con la notificación persistente visible en la barra de estado.
   - Pestaña Sincronizar **antes** y **después** de la sincronización (lista poblada vs. vacía).
   - Pestaña Alertas con la notificación recibida.
3. **Captura del Device Explorer** mostrando el archivo `historial_sensores.txt` con al menos 6 líneas (3 de cada origen).
4. **Captura de Webhook.site** (o el endpoint que uses) mostrando los POST recibidos.
5. **README.md** con: nombre, descripción breve, capturas principales, ejercicios completados (cuáles del Nivel 1/2/3) y cualquier modificación destacable.

> Con esto has construido una app Android moderna que orquesta hardware, persistencia local, tareas diferidas, navegación modular y sincronización tolerante a fallos. Es la base real de cualquier sistema de telemetría, fitness tracker o aplicación IoT del mercado.
