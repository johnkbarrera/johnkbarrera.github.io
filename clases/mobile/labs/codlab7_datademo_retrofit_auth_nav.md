# Lab 7 — Retrofit, Autenticación API y Navegación Anidada

**Autor:** [illarek-lab](https://github.com/illarek-lab)
**Proyecto:** DemoData · `com.illareklab.demodata`
**Branch:** `session--data-persistence`
**Temas:** L1 (Navegación anidada, LaunchedEffect) · L2 (SegmentedButton, ScrollableTabRow) · L3 (Retrofit, kotlinx.serialization, tokens JWT, Android ID)

> **Objetivo del laboratorio:** Reemplazar la autenticación hardcodeada del Lab 5 (`jkn/jkn`) por llamadas reales a una API REST usando Retrofit. Se añaden el registro de usuarios, almacenamiento de tokens JWT en DataStore, identificación del dispositivo con `ANDROID_ID`, y una navegación anidada que separa el flujo de autenticación del flujo principal de la app. `ProfileScreen` se expande con un explorador de registros que distingue datos locales de datos en nube.

---

## 1. Qué cambió respecto al Lab 5

### Archivos nuevos

| Archivo | Propósito |
|---|---|
| `data/remote/NetworkConstants.kt` | URL base y slug del proyecto (tenant) |
| `data/remote/RetrofitClient.kt` | Instancia singleton de Retrofit con OkHttp |
| `data/remote/ApiService.kt` | Interfaz con los 4 endpoints de autenticación |
| `data/remote/model/LoginRequest.kt` | Cuerpo JSON del login |
| `data/remote/model/RegisterRequest.kt` | Cuerpo JSON del registro |
| `data/remote/model/GoogleLoginRequest.kt` | Cuerpo JSON del login con Google |
| `data/remote/model/RefreshTokenRequest.kt` | Cuerpo JSON del refresh de tokens |
| `data/remote/model/TokenResponse.kt` | Respuesta JSON con los tokens |
| `ui/screens/RegisterScreen.kt` | Pantalla de registro de nuevo usuario |

### Archivos modificados

| Archivo | Qué cambió |
|---|---|
| `gradle/libs.versions.toml` | +Retrofit, +OkHttp, +kotlinx-serialization |
| `app/build.gradle.kts` | +plugin `kotlin.serialization`, +dependencias de red |
| `AndroidManifest.xml` | +`usesCleartextTraffic="true"` |
| `SessionManager.kt` | +`KEY_ACCESS_TOKEN`, +`KEY_REFRESH_TOKEN`, +`getDeviceId()`, +`updateTokens()` |
| `SessionViewModel.kt` | Login ya no es hardcodeado; llama a Retrofit. +`register()`, +`loginWithGoogle()`, +`refreshSession()` |
| `LoginScreen.kt` | Campo cambiado a email, +toggle visibilidad contraseña, +botón "Registrar usuario" |
| `Navigation.kt` | Navegación anidada con grafo `auth` y grafo `main`, +`LaunchedEffect` |
| `ProfileScreen.kt` | 6 sub-vistas, `RecordsExplorerScreen` con filtro local/nube, `NestedScreen` wrapper, `isRemote` en `ActivityItem` |

### Lo que se eliminó del Lab 5

- **Tabs Sync y Notificaciones de la `NavigationBar`:** en el Lab 5 la barra inferior tenía 6 tabs (GNSS, Media, Audio, Sync, Notif, Perfil). Ahora tiene solo 4 (GNSS, Media, Audio, Perfil). `SyncScreen` y `NotificationsScreen` se movieron al interior de `ProfileScreen` como sub-vistas navegables, porque son funcionalidades de gestión secundaria que no merecen un tab propio en la barra.
- **Credenciales hardcodeadas `jkn/jkn`** en `SessionViewModel.login()`: reemplazadas completamente por la llamada real a `RetrofitClient.apiService.login(...)`. Ya no existe ninguna comparación de strings en el ViewModel.
- **Sub-vista `MyActivityScreen`** en `ProfileScreen`: renombrada y rehecha como `RecordsExplorerScreen` con filtrado por fuente (local/nube/todo) y por categoría (GNSS, Fotos, Videos, Audios).

---

## 2. Árbol de archivos del proyecto (estado Lab 6)

```
app/src/main/
├── AndroidManifest.xml
├── res/xml/
│   └── file_paths.xml
└── java/com/illareklab/demodata/
    │
    ├── DemoDataApp.kt
    ├── MainActivity.kt
    │
    ├── data/
    │   ├── local/           (sin cambios desde Lab 5)
    │   ├── remote/          ← nuevo Lab 6
    │   │   ├── NetworkConstants.kt
    │   │   ├── RetrofitClient.kt
    │   │   ├── ApiService.kt
    │   │   └── model/
    │   │       ├── LoginRequest.kt
    │   │       ├── RegisterRequest.kt
    │   │       ├── GoogleLoginRequest.kt
    │   │       ├── RefreshTokenRequest.kt
    │   │       └── TokenResponse.kt
    │   ├── repository/      (sin cambios desde Lab 5)
    │   └── session/
    │       └── SessionManager.kt   ← actualizado Lab 6
    │
    ├── security/            (sin cambios)
    ├── services/            (sin cambios)
    ├── util/                (sin cambios)
    ├── workers/             (sin cambios)
    │
    └── ui/
        ├── Navigation.kt           ← actualizado Lab 6
        ├── screens/
        │   ├── GpsScreen.kt        (sin cambios)
        │   ├── LoginScreen.kt      ← actualizado Lab 6
        │   ├── RegisterScreen.kt   ← nuevo Lab 6
        │   ├── MediaScreen.kt      (sin cambios)
        │   ├── AudioScreen.kt      (sin cambios)
        │   ├── NotificationsScreen.kt (sin cambios, ahora accesible desde Perfil)
        │   ├── SyncScreen.kt       (sin cambios, ahora accesible desde Perfil)
        │   └── ProfileScreen.kt    ← actualizado Lab 6
        └── viewmodel/
            ├── SessionViewModel.kt  ← actualizado Lab 6
            └── (resto sin cambios)
```

---

## 3. `gradle/libs.versions.toml` — nuevas entradas de red

### Qué se agregó

```toml
[versions]
# ... versiones previas sin cambios ...
retrofit                 = "2.11.0"
okhttp                   = "4.12.0"
kotlinxSerializationJson = "1.7.3"

[libraries]
# ... librerías previas sin cambios ...
retrofit-core                   = { group = "com.squareup.retrofit2",           name = "retrofit",                          version.ref = "retrofit" }
retrofit-kotlin-serialization   = { group = "com.squareup.retrofit2",           name = "converter-kotlinx-serialization",   version.ref = "retrofit" }
okhttp-logging                  = { group = "com.squareup.okhttp3",             name = "logging-interceptor",               version.ref = "okhttp" }
kotlinx-serialization-json      = { group = "org.jetbrains.kotlinx",            name = "kotlinx-serialization-json",        version.ref = "kotlinxSerializationJson" }
```

**Por qué `converter-kotlinx-serialization` en vez de `converter-gson`:** `kotlinx.serialization` opera con clases `@Serializable` en tiempo de compilación (KSP genera el código de serialización), lo que es más rápido y seguro que Gson que usa reflexión en tiempo de ejecución. También es la opción oficial de Kotlin Multiplatform.

---

## 4. `app/build.gradle.kts` — plugin de serialización y dependencias de red

### Qué se agregó (diff respecto al Lab 5)

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.ksp)
    kotlin("plugin.serialization") version "2.2.10"   // ← NUEVO
}
```

```kotlin
dependencies {
    // ... dependencias previas del Lab 5 sin cambios ...

    // ── Network ──  ← NUEVO bloque
    implementation(libs.retrofit.core)
    implementation(libs.retrofit.kotlin.serialization)
    implementation(libs.okhttp.logging)
    implementation(libs.kotlinx.serialization.json)
}
```

**`kotlin("plugin.serialization")`:** este plugin de compilador genera el código de serialización/deserialización para las clases anotadas con `@Serializable` en tiempo de compilación. Sin él, `@Serializable` en los modelos de red causaría un error de compilación porque el compilador no sabría cómo procesar la anotación.

**Nota sobre CameraX:** el `build.gradle.kts` incluye 5 artefactos de CameraX (`camera-core`, `camera-camera2`, `camera-lifecycle`, `camera-view`, `camera-video`), preparados para reemplazar el uso de `ActivityResultContracts` en Lab 5 en una fase posterior.

---

## 5. `AndroidManifest.xml` — `usesCleartextTraffic`

### Qué cambió

```xml
<application
    android:name=".DemoDataApp"
    ...
    android:usesCleartextTraffic="true"   <!-- ← NUEVO -->
    android:theme="@style/Theme.DemoData">
```

El resto del Manifest es idéntico al Lab 5.

**¿Por qué `usesCleartextTraffic="true"`?** Desde Android 9 (API 28), el SO bloquea por defecto las conexiones HTTP no cifradas. El atributo `usesCleartextTraffic="true"` en `<application>` permite tráfico HTTP en toda la app. Es necesario porque el servidor de desarrollo puede estar en `http://` en lugar de `https://`.

> **Para producción:** eliminar este atributo y usar solo HTTPS. Alternativamente, usar un `res/xml/network_security_config.xml` para permitir cleartext solo hacia dominios específicos de desarrollo.

El Manifest completo:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <uses-feature android:name="android.hardware.camera"           android:required="false" />
    <uses-feature android:name="android.hardware.camera.autofocus" android:required="false" />
    <uses-feature android:name="android.hardware.microphone"       android:required="false" />
    <uses-feature android:name="android.hardware.location"         android:required="false" />
    <uses-feature android:name="android.hardware.location.gps"     android:required="false" />
    <uses-feature android:name="android.hardware.bluetooth_le"     android:required="false" />
    <uses-feature android:name="android.hardware.wifi"             android:required="false" />

    <application
        android:name=".DemoDataApp"
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@drawable/data_icon"
        android:label="@string/app_name"
        android:roundIcon="@drawable/data_icon"
        android:supportsRtl="true"
        android:usesCleartextTraffic="true"
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

---

## 6. Capa de red — `data/remote/`

### 6.1 `NetworkConstants.kt`

```kotlin
package com.illareklab.demodata.data.remote

object NetworkConstants {
    const val BASE_URL     = "https://platform-api.kankunapaq.com/"
    // const val BASE_URL  = "http://illarek.org/"

    const val PROJECT_SLUG = "layout_example"
}
```

`PROJECT_SLUG` es el **tenant** de la API: permite que el mismo servidor sirva a múltiples proyectos. Todos los endpoints de autenticación reciben `{projectSlug}` como segmento de ruta (ej. `layout_example/auth/login`). Para usar la app con otro proyecto basta cambiar este valor sin tocar el resto del código.

### 6.2 `RetrofitClient.kt`

```kotlin
package com.illareklab.demodata.data.remote

import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.kotlinx.serialization.asConverterFactory

object RetrofitClient {
    private val json = Json {
        ignoreUnknownKeys = true
        coerceInputValues  = true
    }

    private val logging = HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
    }

    private val client = OkHttpClient.Builder()
        .addInterceptor(logging)
        .build()

    private val retrofit = Retrofit.Builder()
        .baseUrl(NetworkConstants.BASE_URL)
        .client(client)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()

    val apiService: ApiService = retrofit.create(ApiService::class.java)
}
```

**Puntos clave:**

- **`ignoreUnknownKeys = true`:** si el servidor devuelve campos extra que el modelo Kotlin no tiene declarados, la deserialización no falla. Útil cuando la API evoluciona y añade campos nuevos sin romper la app.
- **`coerceInputValues = true`:** si un campo nullable recibe `null` del JSON pero el campo Kotlin tiene valor por defecto, usa el default en lugar de lanzar excepción.
- **`HttpLoggingInterceptor.Level.BODY`:** en modo DEBUG imprime en Logcat la URL, headers y cuerpo completo de cada petición y respuesta. Para producción cambiar a `Level.NONE` para no loguear tokens en logs.
- **`object RetrofitClient`:** singleton de Kotlin. Solo existe una instancia de `OkHttpClient` y `Retrofit` en todo el proceso, lo que reutiliza el pool de conexiones HTTP y evita crear objetos costosos en cada llamada.

### 6.3 `ApiService.kt`

```kotlin
package com.illareklab.demodata.data.remote

import com.illareklab.demodata.data.remote.model.*
import retrofit2.Response
import retrofit2.http.Body
import retrofit2.http.POST
import retrofit2.http.Path

interface ApiService {

    @POST("{projectSlug}/auth/register")
    suspend fun register(
        @Path("projectSlug") projectSlug: String,
        @Body request: RegisterRequest
    ): Response<Unit>

    @POST("{projectSlug}/auth/login")
    suspend fun login(
        @Path("projectSlug") projectSlug: String,
        @Body request: LoginRequest
    ): Response<TokenResponse>

    @POST("{projectSlug}/auth/google")
    suspend fun loginWithGoogle(
        @Path("projectSlug") projectSlug: String,
        @Body request: GoogleLoginRequest
    ): Response<TokenResponse>

    @POST("{projectSlug}/auth/refresh-token")
    suspend fun refreshToken(
        @Path("projectSlug") projectSlug: String,
        @Body request: RefreshTokenRequest
    ): Response<TokenResponse>
}
```

**`suspend fun` en Retrofit:** desde Retrofit 2.6, las funciones `suspend` están soportadas nativamente. Retrofit suspende la coroutine mientras el hilo de red trabaja y la reanuda cuando llega la respuesta, sin necesidad de callbacks ni `Call<T>.enqueue()`.

**`Response<T>` vs `T` directo:** usar `Response<T>` permite inspeccionar el código HTTP (`response.isSuccessful`, `response.code()`, `response.errorBody()`). Si se usara `T` directamente y el servidor devuelve un 4xx/5xx, Retrofit lanzaría `HttpException`. Con `Response<T>` el control es explícito en el ViewModel.

**`@Path("{projectSlug}")`:** sustituye el segmento `{projectSlug}` en la URL por el valor del parámetro. Si `PROJECT_SLUG = "layout_example"`, la URL resultante es `https://platform-api.kankunapaq.com/layout_example/auth/login`.

### 6.4 Modelos de red

```kotlin
// LoginRequest.kt
package com.illareklab.demodata.data.remote.model

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class LoginRequest(
    val email: String,
    val password: String,
    @SerialName("device_id") val deviceId: String
)
```

```kotlin
// RegisterRequest.kt
package com.illareklab.demodata.data.remote.model

import kotlinx.serialization.Serializable

@Serializable
data class RegisterRequest(
    val email: String,
    val password: String
)
```

```kotlin
// GoogleLoginRequest.kt
package com.illareklab.demodata.data.remote.model

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class GoogleLoginRequest(
    val token: String,
    @SerialName("device_id") val deviceId: String
)
```

```kotlin
// RefreshTokenRequest.kt
package com.illareklab.demodata.data.remote.model

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class RefreshTokenRequest(
    @SerialName("refresh_token") val refreshToken: String,
    @SerialName("device_id")     val deviceId: String
)
```

```kotlin
// TokenResponse.kt
package com.illareklab.demodata.data.remote.model

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class TokenResponse(
    @SerialName("access_token")  val accessToken: String,
    @SerialName("refresh_token") val refreshToken: String,
    @SerialName("token_type")    val tokenType: String
)
```

**`@SerialName`:** mapea el nombre del campo Kotlin al nombre del campo JSON del servidor. El servidor espera `device_id` en snake_case; Kotlin usa `deviceId` en camelCase. Sin `@SerialName`, el campo JSON y el campo Kotlin deben tener exactamente el mismo nombre.

**`device_id` en `LoginRequest` y `GoogleLoginRequest` pero no en `RegisterRequest`:** el registro no necesita identificar el dispositivo porque aún no hay sesión activa. El login y el refresh sí lo incluyen para que el servidor pueda asociar tokens a dispositivos específicos (útil para revocar sesiones remotamente por dispositivo).

---

## 7. `SessionManager.kt` — Tokens y Android ID

### Qué se eliminó del Lab 5
- `suspend fun login(username: String)` — firmado cambia: ahora recibe también los dos tokens.

### Qué se añadió
- `KEY_ACCESS_TOKEN` y `KEY_REFRESH_TOKEN` como claves de DataStore.
- Flows `accessToken` y `refreshToken`.
- `fun getDeviceId()` — lee `Settings.Secure.ANDROID_ID`.
- `suspend fun updateTokens(access, refresh)` — renueva tokens sin tocar el resto de la sesión.
- `suspend fun login(username, access, refresh)` — nueva firma que guarda los tres valores en una sola operación atómica.

```kotlin
package com.illareklab.demodata.data.session

import android.annotation.SuppressLint
import android.content.Context
import android.provider.Settings
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
        val KEY_IS_LOGGED_IN   = booleanPreferencesKey("is_logged_in")
        val KEY_USERNAME       = stringPreferencesKey("username")
        val KEY_ACCESS_TOKEN   = stringPreferencesKey("access_token")    // ← nuevo Lab 6
        val KEY_REFRESH_TOKEN  = stringPreferencesKey("refresh_token")   // ← nuevo Lab 6
        val KEY_DARK_MODE      = booleanPreferencesKey("dark_mode")
    }

    val isLoggedIn: Flow<Boolean> = context.sessionDataStore.data
        .map { prefs -> prefs[KEY_IS_LOGGED_IN] ?: false }

    val currentUsername: Flow<String?> = context.sessionDataStore.data
        .map { prefs -> prefs[KEY_USERNAME] }

    val accessToken: Flow<String?> = context.sessionDataStore.data    // ← nuevo Lab 6
        .map { prefs -> prefs[KEY_ACCESS_TOKEN] }

    val refreshToken: Flow<String?> = context.sessionDataStore.data   // ← nuevo Lab 6
        .map { prefs -> prefs[KEY_REFRESH_TOKEN] }

    val isDarkMode: Flow<Boolean?> = context.sessionDataStore.data
        .map { prefs -> prefs[KEY_DARK_MODE] }

    @SuppressLint("HardwareIds")
    fun getDeviceId(): String {
        return Settings.Secure.getString(
            context.contentResolver,
            Settings.Secure.ANDROID_ID
        ) ?: "unknown_device"
    }

    // Firma actualizada: ahora persiste los tokens junto al username
    suspend fun login(username: String, access: String, refresh: String) {
        context.sessionDataStore.edit { prefs ->
            prefs[KEY_IS_LOGGED_IN]  = true
            prefs[KEY_USERNAME]      = username
            prefs[KEY_ACCESS_TOKEN]  = access
            prefs[KEY_REFRESH_TOKEN] = refresh
        }
    }

    // Renueva solo los tokens sin tocar el resto de la sesión
    suspend fun updateTokens(access: String, refresh: String) {
        context.sessionDataStore.edit { prefs ->
            prefs[KEY_ACCESS_TOKEN]  = access
            prefs[KEY_REFRESH_TOKEN] = refresh
        }
    }

    suspend fun setDarkMode(enabled: Boolean) {
        context.sessionDataStore.edit { prefs -> prefs[KEY_DARK_MODE] = enabled }
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

**`Settings.Secure.ANDROID_ID`:** identificador único de 64 bits asignado por el SO al instalarse la app por primera vez. Persiste entre reinicios pero cambia si se hace factory reset o se reinstala la app en algunos dispositivos. Es la forma estándar de identificar un dispositivo sin pedir permisos adicionales. `@SuppressLint("HardwareIds")` suprime el aviso de Lint sobre privacidad ya que el uso es para identificación de sesión, no para tracking publicitario.

**`DataStore.edit{}` es atómico:** todos los `prefs[KEY_X] = value` dentro de un bloque `edit{}` se persisten juntos en disco. Si la app crashea a mitad, no se guardará ningún campo parcial.

---

## 8. `SessionViewModel.kt` — Llamadas reales a la API

### Qué se eliminó del Lab 5
- La función `login()` con comparación `if (username == "jkn" && password == "jkn")` — eliminada completamente.

### Qué se añadió
- Importaciones de `RetrofitClient`, `NetworkConstants` y todos los modelos de red.
- `login()` ahora lanza una coroutine que llama a `RetrofitClient.apiService.login()`.
- `register()` — nuevo endpoint de registro.
- `loginWithGoogle()` — nuevo endpoint de OAuth con Google.
- `refreshSession()` — usa `sessionManager.refreshToken.firstOrNull()` para renovar tokens.

```kotlin
package com.illareklab.demodata.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.illareklab.demodata.data.session.SessionManager
import com.illareklab.demodata.data.remote.NetworkConstants
import com.illareklab.demodata.data.remote.RetrofitClient
import com.illareklab.demodata.data.remote.model.*
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.firstOrNull
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

    fun login(email: String, password: String, onResult: (Boolean) -> Unit) {
        viewModelScope.launch {
            try {
                val response = RetrofitClient.apiService.login(
                    projectSlug = NetworkConstants.PROJECT_SLUG,
                    request     = LoginRequest(
                        email    = email,
                        password = password,
                        deviceId = sessionManager.getDeviceId()
                    )
                )
                if (response.isSuccessful && response.body() != null) {
                    val body = response.body()!!
                    sessionManager.login(email, body.accessToken, body.refreshToken)
                    onResult(true)
                } else {
                    onResult(false)
                }
            } catch (e: Exception) {
                onResult(false)
            }
        }
    }

    fun register(email: String, password: String, onResult: (Boolean) -> Unit) {
        viewModelScope.launch {
            try {
                val response = RetrofitClient.apiService.register(
                    projectSlug = NetworkConstants.PROJECT_SLUG,
                    request     = RegisterRequest(email, password)
                )
                onResult(response.isSuccessful)
            } catch (e: Exception) {
                onResult(false)
            }
        }
    }

    fun loginWithGoogle(googleToken: String, onResult: (Boolean) -> Unit) {
        viewModelScope.launch {
            try {
                val response = RetrofitClient.apiService.loginWithGoogle(
                    projectSlug = NetworkConstants.PROJECT_SLUG,
                    request     = GoogleLoginRequest(
                        token    = googleToken,
                        deviceId = sessionManager.getDeviceId()
                    )
                )
                if (response.isSuccessful && response.body() != null) {
                    val body = response.body()!!
                    sessionManager.login("Google User", body.accessToken, body.refreshToken)
                    onResult(true)
                } else {
                    onResult(false)
                }
            } catch (e: Exception) {
                onResult(false)
            }
        }
    }

    fun refreshSession(onResult: (Boolean) -> Unit) {
        viewModelScope.launch {
            try {
                val currentRefresh = sessionManager.refreshToken.firstOrNull()
                if (currentRefresh != null) {
                    val response = RetrofitClient.apiService.refreshToken(
                        projectSlug = NetworkConstants.PROJECT_SLUG,
                        request     = RefreshTokenRequest(
                            refreshToken = currentRefresh,
                            deviceId     = sessionManager.getDeviceId()
                        )
                    )
                    if (response.isSuccessful && response.body() != null) {
                        val body = response.body()!!
                        sessionManager.updateTokens(body.accessToken, body.refreshToken)
                        onResult(true)
                        return@launch
                    }
                }
                onResult(false)
            } catch (e: Exception) {
                onResult(false)
            }
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

**Manejo de errores en red:** cada función tiene un bloque `try/catch` que atrapa excepciones de red (`UnknownHostException`, `SocketTimeoutException`, etc.) y llama `onResult(false)`. La UI recibe siempre un `Boolean` limpio sin necesidad de saber si falló por red o por credenciales incorrectas.

**`sessionManager.refreshToken.firstOrNull()`:** `firstOrNull()` recolecta una única emisión del Flow y suspende hasta obtenerla. Es la forma correcta de leer un valor puntual de DataStore dentro de una coroutine sin abrir una suscripción permanente.

**Patrón access/refresh token:** el servidor devuelve dos tokens. El `access_token` tiene vida corta (minutos) y se adjunta en el header `Authorization: Bearer` de cada petición autenticada. Cuando expira, se usa el `refresh_token` (vida larga) para pedir uno nuevo sin que el usuario tenga que loguearse de nuevo. `refreshSession()` implementa esta renovación.

---

## 9. `LoginScreen.kt` — Email, visibilidad de contraseña y registro

### Qué se eliminó del Lab 5
- El label del campo de usuario era `"Usuario"` — ahora es `"Email"`.
- El hint `"Credenciales por defecto: jkn / jkn"` — eliminado porque ya no hay credenciales hardcodeadas.

### Qué se añadió
- Toggle de visibilidad de contraseña con `Visibility`/`VisibilityOff`.
- Botón `OutlinedButton` "Registrar usuario" que navega a `RegisterScreen`.
- Parámetro `onRegisterNavigate: () -> Unit` en la firma del composable.

```kotlin
package com.illareklab.demodata.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Visibility
import androidx.compose.material.icons.filled.VisibilityOff
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.text.input.VisualTransformation
import androidx.compose.ui.unit.dp

@Composable
fun LoginScreen(
    onSubmit: (username: String, password: String, onResult: (Boolean) -> Unit) -> Unit,
    onRegisterNavigate: () -> Unit
) {
    var usuario by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var passwordVisible by remember { mutableStateOf(false) }
    var error by remember { mutableStateOf("") }
    var verificando by remember { mutableStateOf(false) }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(32.dp),
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
            value = usuario,
            onValueChange = { usuario = it },
            label = { Text("Email") },
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
            visualTransformation = if (passwordVisible) VisualTransformation.None else PasswordVisualTransformation(),
            trailingIcon = {
                val icon = if (passwordVisible) Icons.Default.Visibility else Icons.Default.VisibilityOff
                val description = if (passwordVisible) "Ocultar contraseña" else "Mostrar contraseña"

                IconButton(onClick = { passwordVisible = !passwordVisible }) {
                    Icon(imageVector = icon, contentDescription = description)
                }
            },
            modifier = Modifier.fillMaxWidth()
        )

        if (error.isNotEmpty()) {
            Spacer(modifier = Modifier.height(8.dp))
            Text(error, color = MaterialTheme.colorScheme.error)
        }

        Spacer(modifier = Modifier.height(24.dp))
        Button(
            onClick = {
                error = ""
                verificando = true
                onSubmit(usuario, password) { ok ->
                    verificando = false
                    if (!ok) error = "Credenciales incorrectas. Revisa tu email y contraseña."
                }
            },
            enabled = !verificando && usuario.isNotBlank() && password.isNotBlank(),
            modifier = Modifier
                .fillMaxWidth()
                .height(50.dp)
        ) {
            if (verificando) {
                CircularProgressIndicator(
                    modifier = Modifier.height(24.dp),
                    strokeWidth = 2.dp,
                    color = MaterialTheme.colorScheme.onPrimary
                )
            } else {
                Text("Ingresar")
            }
        }

        Spacer(modifier = Modifier.height(12.dp))

        OutlinedButton(
            onClick = onRegisterNavigate,
            enabled = !verificando,
            modifier = Modifier
                .fillMaxWidth()
                .height(50.dp)
        ) {
            Text("Registrar usuario")
        }

        Spacer(modifier = Modifier.height(16.dp))
        Text(
            "Usa tus credenciales de Platform API",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.outline
        )
    }
}
```

**Toggle de visibilidad:** `VisualTransformation.None` muestra el texto plano; `PasswordVisualTransformation()` lo enmascara con `•`. El estado `passwordVisible` controla cuál se aplica. El `trailingIcon` cambia el ícono acorde al estado actual.

---

## 10. `RegisterScreen.kt` — Nueva pantalla de registro

```kotlin
package com.illareklab.demodata.ui.screens

import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ArrowBack
import androidx.compose.material.icons.filled.Visibility
import androidx.compose.material.icons.filled.VisibilityOff
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.text.input.VisualTransformation
import androidx.compose.ui.unit.dp

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RegisterScreen(
    onBack:   () -> Unit,
    onSubmit: (email: String, pass: String, onResult: (Boolean) -> Unit) -> Unit
) {
    var email                   by remember { mutableStateOf("") }
    var password                by remember { mutableStateOf("") }
    var confirmPassword         by remember { mutableStateOf("") }
    var passwordVisible         by remember { mutableStateOf(false) }
    var confirmPasswordVisible  by remember { mutableStateOf(false) }
    var loading                 by remember { mutableStateOf(false) }
    var error                   by remember { mutableStateOf("") }

    Scaffold(
        topBar = {
            TopAppBar(
                title            = { Text("Nuevo Usuario") },
                navigationIcon   = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "Volver")
                    }
                }
            )
        }
    ) { padding ->
        Column(
            modifier            = Modifier.fillMaxSize().padding(padding).padding(32.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            OutlinedTextField(
                value         = email,
                onValueChange = { email = it },
                label         = { Text("Email") },
                modifier      = Modifier.fillMaxWidth(),
                singleLine    = true,
                enabled       = !loading
            )
            Spacer(modifier = Modifier.height(8.dp))

            OutlinedTextField(
                value               = password,
                onValueChange       = { password = it },
                label               = { Text("Contraseña") },
                modifier            = Modifier.fillMaxWidth(),
                singleLine          = true,
                visualTransformation = if (passwordVisible) VisualTransformation.None
                                       else PasswordVisualTransformation(),
                trailingIcon        = {
                    IconButton(onClick = { passwordVisible = !passwordVisible }) {
                        Icon(if (passwordVisible) Icons.Default.Visibility else Icons.Default.VisibilityOff, null)
                    }
                },
                enabled = !loading
            )
            Spacer(modifier = Modifier.height(8.dp))

            OutlinedTextField(
                value               = confirmPassword,
                onValueChange       = { confirmPassword = it },
                label               = { Text("Confirmar Contraseña") },
                modifier            = Modifier.fillMaxWidth(),
                singleLine          = true,
                visualTransformation = if (confirmPasswordVisible) VisualTransformation.None
                                       else PasswordVisualTransformation(),
                trailingIcon        = {
                    IconButton(onClick = { confirmPasswordVisible = !confirmPasswordVisible }) {
                        Icon(if (confirmPasswordVisible) Icons.Default.Visibility else Icons.Default.VisibilityOff, null)
                    }
                },
                enabled = !loading
            )

            if (error.isNotEmpty()) {
                Spacer(modifier = Modifier.height(8.dp))
                Text(error, color = MaterialTheme.colorScheme.error)
            }

            Spacer(modifier = Modifier.height(24.dp))

            Button(
                onClick  = {
                    if (password != confirmPassword) {
                        error = "Las contraseñas no coinciden"
                        return@Button
                    }
                    error   = ""
                    loading = true
                    onSubmit(email, password) { success ->
                        loading = false
                        if (!success) error = "Error al registrar. Intente con otro email."
                    }
                },
                modifier = Modifier.fillMaxWidth().height(50.dp),
                enabled  = !loading && email.isNotBlank() && password.isNotBlank() && confirmPassword.isNotBlank()
            ) {
                if (loading) {
                    CircularProgressIndicator(modifier = Modifier.size(24.dp), strokeWidth = 2.dp)
                } else {
                    Text("Crear cuenta")
                }
            }
        }
    }
}
```

**Validación cliente `password != confirmPassword`:** se verifica antes de llamar a la API para no gastar una petición de red en un error evitable. `return@Button` sale del lambda `onClick` sin ejecutar el resto.

---

## 11. `Navigation.kt` — Navegación anidada y `LaunchedEffect`

### Qué se eliminó del Lab 5
- El simple `if (isLoggedIn) MainScaffold(sessionVm) else LoginScreen(...)` — reemplazado por un `NavHost` con grafos anidados.
- Tabs `"sync"` y `"notif"` de la `NavigationBar` — movidos al interior de `ProfileScreen`.
- `composable("sync") { SyncScreen() }` y `composable("notif") { NotificationsScreen() }` del `NavHost` interno.

### Qué se añadió
- `rootNavController` que maneja la transición auth ↔ main.
- `navigation(startDestination = "login", route = "auth")` — grafo anidado de autenticación.
- `composable("register")` dentro del grafo `auth`.
- `LaunchedEffect(isLoggedIn)` para navegar reactivamente cuando cambia el estado de sesión.

```kotlin
package com.illareklab.demodata.ui

import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.Logout
import androidx.compose.material.icons.filled.CameraAlt
import androidx.compose.material.icons.filled.LocationOn
import androidx.compose.material.icons.filled.Mic
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
import androidx.navigation.compose.navigation
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

    val rootNavController = rememberNavController()
    val isLoggedIn by sessionVm.isLoggedIn.collectAsStateWithLifecycle()

    // Navega reactivamente cada vez que isLoggedIn cambia
    LaunchedEffect(isLoggedIn) {
        if (isLoggedIn) {
            rootNavController.navigate("main") {
                popUpTo("auth") { inclusive = true }
            }
        } else {
            rootNavController.navigate("auth") {
                popUpTo(0) { inclusive = true }
            }
        }
    }

    NavHost(
        navController    = rootNavController,
        startDestination = if (isLoggedIn) "main" else "auth"
    ) {
        // ── Grafo de Autenticación ──
        navigation(startDestination = "login", route = "auth") {
            composable("login") {
                LoginScreen(
                    onSubmit           = sessionVm::login,
                    onRegisterNavigate = { rootNavController.navigate("register") }
                )
            }
            composable("register") {
                RegisterScreen(
                    onBack   = { rootNavController.popBackStack() },
                    onSubmit = { email, pass, onResult ->
                        sessionVm.register(email, pass) { success ->
                            onResult(success)
                            if (success) rootNavController.popBackStack()
                        }
                    }
                )
            }
        }

        // ── Pantalla Principal ──
        composable("main") {
            MainScaffold(sessionVm)
        }
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
                    "gps"     to (Icons.Default.LocationOn to "GNSS"),
                    "media"   to (Icons.Default.CameraAlt  to "Media"),
                    "audio"   to (Icons.Default.Mic        to "Audio"),
                    "profile" to (Icons.Default.Person     to "Perfil")
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
            composable("profile") { ProfileScreen(onLogout = sessionVm::logout, username = username) }
        }
    }
}
```

### Navegación anidada — `navigation(route = "auth")`

```
rootNavController
├── "auth"  (grafo anidado)
│   ├── "login"    → LoginScreen
│   └── "register" → RegisterScreen
└── "main"
      └── MainScaffold (navController interno)
            ├── "gps"
            ├── "media"
            ├── "audio"
            └── "profile"
```

**`navigation(startDestination = "login", route = "auth")`:** agrupa rutas bajo un mismo sub-grafo. `popUpTo("auth") { inclusive = true }` elimina todo el grafo de autenticación del back stack cuando el login es exitoso — el botón "Atrás" no puede regresar a `LoginScreen` desde la app principal.

**`LaunchedEffect(isLoggedIn)`:** se ejecuta en la composición y cada vez que `isLoggedIn` cambia. Si el token caduca y el backend lo invalida, `SessionManager.logout()` hará que `isLoggedIn` emita `false`, el `LaunchedEffect` se disparará y navegará automáticamente a `"auth"` sin que ningún composable de pantalla tenga que saber de ello.

**`popUpTo(0) { inclusive = true }`:** elimina absolutamente todo el back stack (índice 0 = raíz). Garantiza que al hacer logout, el usuario no pueda presionar "Atrás" para regresar a la pantalla principal.

---

## 12. `ProfileScreen.kt` — 6 sub-vistas y explorador de registros

### Qué se eliminó del Lab 5
- Sub-vista `MyActivityScreen` — reemplazada por `RecordsExplorerScreen`, que agrega filtros por fuente y por categoría.
- El `sealed class ActivityItem` no tenía `isRemote` — se agregó como campo abstracto a todos los subtipos.

### Qué se añadió
- `enum class RecordsSource { LOCAL, REMOTE, ALL }` — tres modos de visualización.
- `sealed class ProfileViewState` ahora tiene 6 estados: `Menu`, `MyProfile`, `LocalRecords`, `AllRecords`, `Sync`, `Notifications`.
- `RecordsExplorerScreen` — lista filtrada con `SegmentedButton` (LOCAL/CLOUD/ALL) y `ScrollableTabRow` (Todos/GNSS/Fotos/Videos/Audios).
- `NestedScreen` — wrapper que embebe `SyncScreen()` y `NotificationsScreen()` dentro de `ProfileScreen` con un header "← Volver".
- `ProfileMenu` con 5 opciones navegables (antes tenía 2).
- `MyProfileScreen` muestra ahora el `Android ID` del dispositivo.
- `ActivityRow` diferencia visualmente con `colorScheme.tertiary` los ítems remotos y muestra un chip "Cloud".

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
import androidx.compose.material.icons.automirrored.filled.List
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

enum class RecordsSource { LOCAL, REMOTE, ALL }

@Composable
fun ProfileScreen(onLogout: () -> Unit, username: String? = null) {
    val app = LocalContext.current.applicationContext as DemoDataApp
    val sessionVm: SessionViewModel = viewModel(
        factory = SessionViewModel.Factory(app.sessionManager)
    )
    var viewState by remember { mutableStateOf<ProfileViewState>(ProfileViewState.Menu) }

    when (viewState) {
        ProfileViewState.Menu          -> ProfileMenu(
            username                = username,
            onLogout                = onLogout,
            onNavigateToProfile      = { viewState = ProfileViewState.MyProfile },
            onNavigateToLocal        = { viewState = ProfileViewState.LocalRecords },
            onNavigateToAll          = { viewState = ProfileViewState.AllRecords },
            onNavigateToSync         = { viewState = ProfileViewState.Sync },
            onNavigateToNotifications = { viewState = ProfileViewState.Notifications }
        )
        ProfileViewState.MyProfile     -> MyProfileScreen(username = username, sessionVm = sessionVm, onBack = { viewState = ProfileViewState.Menu })
        ProfileViewState.LocalRecords  -> RecordsExplorerScreen(title = "Registros locales", allowedSource = RecordsSource.LOCAL, onBack = { viewState = ProfileViewState.Menu })
        ProfileViewState.AllRecords    -> RecordsExplorerScreen(title = "Todos los registros", allowedSource = RecordsSource.ALL, onBack = { viewState = ProfileViewState.Menu })
        ProfileViewState.Sync          -> NestedScreen(title = "Sincronización",  onBack = { viewState = ProfileViewState.Menu }) { SyncScreen() }
        ProfileViewState.Notifications -> NestedScreen(title = "Notificaciones",  onBack = { viewState = ProfileViewState.Menu }) { NotificationsScreen() }
    }
}

private sealed class ProfileViewState {
    object Menu : ProfileViewState()
    object MyProfile : ProfileViewState()
    object LocalRecords : ProfileViewState()
    object AllRecords : ProfileViewState()
    object Sync : ProfileViewState()
    object Notifications : ProfileViewState()
}

@Composable
private fun ProfileMenu(
    username: String?,
    onLogout: () -> Unit,
    onNavigateToProfile: () -> Unit,
    onNavigateToLocal: () -> Unit,
    onNavigateToAll: () -> Unit,
    onNavigateToSync: () -> Unit,
    onNavigateToNotifications: () -> Unit
) {
    var mostrarConfirmacion by remember { mutableStateOf(false) }

    Column(
        modifier            = Modifier.fillMaxSize().padding(24.dp).verticalScroll(rememberScrollState()),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Icon(Icons.Default.Person, null, modifier = Modifier.size(72.dp), tint = MaterialTheme.colorScheme.primary)
        Spacer(modifier = Modifier.height(16.dp))
        Text(text = username ?: "Usuario", style = MaterialTheme.typography.headlineMedium)
        Spacer(modifier = Modifier.height(32.dp))

        MenuOption(Icons.Default.Person,              "Mi Perfil",          "Metadatos y configuración de tema",         onNavigateToProfile)
        Spacer(modifier = Modifier.height(12.dp))
        MenuOption(Icons.Default.History,             "Registros locales",  "Datos almacenados en este dispositivo",     onNavigateToLocal)
        Spacer(modifier = Modifier.height(12.dp))
        MenuOption(Icons.AutoMirrored.Filled.List,    "Todos los registros","Explorador local + nube (API)",             onNavigateToAll)
        Spacer(modifier = Modifier.height(12.dp))
        MenuOption(Icons.Default.CloudSync,           "Sincronización",     "Subir registros al servidor remoto",        onNavigateToSync)
        Spacer(modifier = Modifier.height(12.dp))
        MenuOption(Icons.Default.Notifications,       "Notificaciones",     "Programar y gestionar notificaciones",      onNavigateToNotifications)
        Spacer(modifier = Modifier.height(32.dp))

        Button(
            onClick  = { mostrarConfirmacion = true },
            modifier = Modifier.fillMaxWidth().height(56.dp),
            colors   = ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.error)
        ) {
            Icon(Icons.AutoMirrored.Filled.Logout, null)
            Spacer(modifier = Modifier.size(8.dp))
            Text("Cerrar sesión")
        }
    }
    if (mostrarConfirmacion) LogoutDialog(onConfirm = onLogout, onDismiss = { mostrarConfirmacion = false })
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
private fun RecordsExplorerScreen(title: String, allowedSource: RecordsSource, onBack: () -> Unit) {
    val context      = LocalContext.current
    val app          = context.applicationContext as DemoDataApp

    val googlePoints  by app.gpsRepository.googlePoints.collectAsStateWithLifecycle(emptyList())
    val sensorsPoints by app.gpsRepository.sensorsPoints.collectAsStateWithLifecycle(emptyList())
    val allMedia      by app.mediaRepository.allMedia.collectAsStateWithLifecycle(emptyList())
    val allAudios     by app.audioRepository.allAudios.collectAsStateWithLifecycle(emptyList())

    var selectedTab  by remember { mutableIntStateOf(0) }
    val tabs         = listOf("Todos", "GNSS", "Fotos", "Videos", "Audios")
    var sourceFilter by remember { mutableStateOf(if (allowedSource == RecordsSource.ALL) RecordsSource.ALL else RecordsSource.LOCAL) }
    var detailItem   by remember { mutableStateOf<ActivityItem?>(null) }

    Column(modifier = Modifier.fillMaxSize()) {
        Row(modifier = Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
            Text(title, style = MaterialTheme.typography.headlineSmall, modifier = Modifier.weight(1f))
            TextButton(onClick = onBack) { Text("Cerrar") }
        }

        // Filtro LOCAL / NUBE / TODO — solo visible en modo AllRecords
        if (allowedSource == RecordsSource.ALL) {
            SingleChoiceSegmentedButtonRow(
                modifier = Modifier.fillMaxWidth().padding(horizontal = 16.dp, vertical = 8.dp)
            ) {
                SegmentedButton(
                    selected = sourceFilter == RecordsSource.ALL,
                    onClick  = { sourceFilter = RecordsSource.ALL },
                    shape    = SegmentedButtonDefaults.itemShape(0, 3),
                    icon     = { Icon(Icons.AutoMirrored.Filled.List, null, Modifier.size(16.dp)) }
                ) { Text("Todo",  style = MaterialTheme.typography.labelSmall) }
                SegmentedButton(
                    selected = sourceFilter == RecordsSource.LOCAL,
                    onClick  = { sourceFilter = RecordsSource.LOCAL },
                    shape    = SegmentedButtonDefaults.itemShape(1, 3),
                    icon     = { Icon(Icons.Default.Storage, null, Modifier.size(16.dp)) }
                ) { Text("Local", style = MaterialTheme.typography.labelSmall) }
                SegmentedButton(
                    selected = sourceFilter == RecordsSource.REMOTE,
                    onClick  = { sourceFilter = RecordsSource.REMOTE },
                    shape    = SegmentedButtonDefaults.itemShape(2, 3),
                    icon     = { Icon(Icons.Default.Cloud, null, Modifier.size(16.dp)) }
                ) { Text("Nube",  style = MaterialTheme.typography.labelSmall) }
            }
        }

        ScrollableTabRow(selectedTabIndex = selectedTab, edgePadding = 16.dp) {
            tabs.forEachIndexed { index, t ->
                Tab(selected = selectedTab == index, onClick = { selectedTab = index }, text = { Text(t) })
            }
        }

        val filteredItems = remember(selectedTab, sourceFilter, googlePoints, sensorsPoints, allMedia, allAudios) {
            val localItems = mutableListOf<ActivityItem>().apply {
                addAll(googlePoints.map  { ActivityItem.GpsGoogle(it,  isRemote = false) })
                addAll(sensorsPoints.map { ActivityItem.GpsSensors(it, isRemote = false) })
                addAll(allMedia.map      { ActivityItem.Media(it,      isRemote = false) })
                addAll(allAudios.map     { ActivityItem.Audio(it,      isRemote = false) })
            }

            // Datos remotos simulados — placeholder hasta integrar API de consulta
            val remoteItems = if (sourceFilter != RecordsSource.LOCAL) listOf(
                ActivityItem.GpsGoogle(GpsGoogleEntity(id = 999, latitude = -12.0463, longitude = -77.0427, accuracy = 5f, timestamp = System.currentTimeMillis() - 86400000), isRemote = true),
                ActivityItem.Media(MediaEntity(id = 888, filePath = "", type = "PHOTO", sizeBytes = 1024, timestamp = System.currentTimeMillis() - 43200000), isRemote = true)
            ) else emptyList()

            val combined = when (sourceFilter) {
                RecordsSource.LOCAL  -> localItems
                RecordsSource.REMOTE -> remoteItems
                RecordsSource.ALL    -> localItems + remoteItems
            }

            val filtered = when (selectedTab) {
                0    -> combined
                1    -> combined.filter { it is ActivityItem.GpsGoogle || it is ActivityItem.GpsSensors }
                2    -> combined.filter { it is ActivityItem.Media && it.label == "PHOTO" }
                3    -> combined.filter { it is ActivityItem.Media && it.label == "VIDEO" }
                4    -> combined.filter { it is ActivityItem.Audio }
                else -> combined
            }
            filtered.sortedByDescending { it.timestamp }
        }

        LazyColumn(
            modifier            = Modifier.weight(1f).padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(filteredItems) { item ->
                ActivityRow(item, onClick = { detailItem = item })
            }
        }
    }

    if (detailItem != null) {
        ActivityDetailDialog(item = detailItem!!, onDismiss = { detailItem = null })
    }
}

@Composable
private fun NestedScreen(title: String, onBack: () -> Unit, content: @Composable () -> Unit) {
    Column(modifier = Modifier.fillMaxSize()) {
        Row(
            modifier          = Modifier.padding(horizontal = 8.dp, vertical = 4.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            TextButton(onClick = onBack) { Text("← Volver") }
            Text(title, style = MaterialTheme.typography.titleMedium)
        }
        HorizontalDivider()
        content()
    }
}

@Composable
private fun MenuOption(icon: ImageVector, title: String, subtitle: String, onClick: () -> Unit) {
    Card(
        modifier = Modifier.fillMaxWidth().clickable(onClick = onClick),
        colors   = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceVariant)
    ) {
        Row(modifier = Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
            Icon(icon, null, tint = MaterialTheme.colorScheme.primary)
            Spacer(modifier = Modifier.width(16.dp))
            Column(modifier = Modifier.weight(1f)) {
                Text(title,    style = MaterialTheme.typography.titleMedium)
                Text(subtitle, style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
            Icon(Icons.Default.ChevronRight, null)
        }
    }
}

@Composable
private fun MyProfileScreen(username: String?, sessionVm: SessionViewModel, onBack: () -> Unit) {
    val isDarkModePref by sessionVm.isDarkMode.collectAsStateWithLifecycle()
    val isDark         = isDarkModePref ?: isSystemInDarkTheme()
    val context        = LocalContext.current
    val androidId      = android.provider.Settings.Secure.getString(
        context.contentResolver, android.provider.Settings.Secure.ANDROID_ID
    )

    Column(modifier = Modifier.fillMaxSize().padding(24.dp).verticalScroll(rememberScrollState())) {
        Text("Mi Perfil", style = MaterialTheme.typography.headlineSmall)
        Spacer(modifier = Modifier.height(24.dp))

        ProfileMetadataItem("Username",         username ?: "N/A")
        ProfileMetadataItem("Rol",              "Administrador / Operador")
        ProfileMetadataItem("Directorio Local", context.filesDir.absolutePath)

        Row(
            modifier              = Modifier.fillMaxWidth().padding(vertical = 12.dp),
            verticalAlignment     = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Icon(Icons.Default.DarkMode, null, tint = MaterialTheme.colorScheme.primary)
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
        ProfileMetadataItem("Android ID",       androidId ?: "N/A")   // ← nuevo Lab 6

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

sealed class ActivityItem {
    abstract val timestamp: Long
    abstract val label: String
    abstract val icon: ImageVector
    abstract val isRemote: Boolean           // ← nuevo Lab 6

    data class GpsGoogle(val data: GpsGoogleEntity, override val isRemote: Boolean) : ActivityItem() {
        override val timestamp = data.timestamp
        override val label     = "GNSS Google"
        override val icon      = Icons.Default.LocationOn
    }
    data class GpsSensors(val data: GpsSensorsEntity, override val isRemote: Boolean) : ActivityItem() {
        override val timestamp = data.timestamp
        override val label     = "GNSS Sensor"
        override val icon      = Icons.Default.LocationOn
    }
    data class Media(val data: MediaEntity, override val isRemote: Boolean) : ActivityItem() {
        override val timestamp = data.timestamp
        override val label     = data.type
        override val icon      = if (data.type == MediaType.PHOTO.name) Icons.Default.PhotoCamera else Icons.Default.Videocam
    }
    data class Audio(val data: AudioEntity, override val isRemote: Boolean) : ActivityItem() {
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
                item.icon, null,
                tint = when {
                    isNoSignal    -> MaterialTheme.colorScheme.error
                    item.isRemote -> MaterialTheme.colorScheme.tertiary  // azul/verde para nube
                    else          -> MaterialTheme.colorScheme.primary
                }
            )
            Spacer(modifier = Modifier.width(16.dp))
            Column(modifier = Modifier.weight(1f)) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Text(
                        text  = if (isNoSignal) "${item.label} (Sin señal)" else item.label,
                        style = MaterialTheme.typography.titleSmall,
                        color = if (isNoSignal) MaterialTheme.colorScheme.error else Color.Unspecified
                    )
                    if (item.isRemote) {
                        Spacer(modifier = Modifier.width(8.dp))
                        SuggestionChip(
                            onClick  = {},
                            label    = { Text("Cloud", style = MaterialTheme.typography.labelSmall) },
                            modifier = Modifier.height(20.dp)
                        )
                    }
                }
                Text(dateFormat.format(Date(item.timestamp)), style = MaterialTheme.typography.bodySmall)
            }
            Icon(Icons.Default.ChevronRight, null)
        }
    }
}

@Composable
private fun ActivityDetailDialog(item: ActivityItem, onDismiss: () -> Unit) {
    val context    = LocalContext.current
    val dateFormat = remember { SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault()) }

    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(item.label + if (item.isRemote) " (Nube)" else " (Local)") },
        text  = {
            Column(modifier = Modifier.verticalScroll(rememberScrollState())) {
                Text("Fecha:  ${dateFormat.format(Date(item.timestamp))}")
                Text("Origen: ${if (item.isRemote) "Servidor Externo" else "Memoria del Dispositivo"}")
                Spacer(modifier = Modifier.height(8.dp))

                when (item) {
                    is ActivityItem.GpsGoogle -> {
                        if (item.data.latitude != null) {
                            Text("Lat: ${item.data.latitude}")
                            Text("Lon: ${item.data.longitude}")
                            Text("Accuracy: ±${item.data.accuracy}m")
                        } else {
                            Text("Estado: SIN SEÑAL", color = MaterialTheme.colorScheme.error)
                        }
                    }
                    is ActivityItem.GpsSensors -> {
                        if (item.data.latitude != null) {
                            Text("Lat: ${item.data.latitude}")
                            Text("Lon: ${item.data.longitude}")
                            item.data.altitude?.let { Text("Altitud: ${it}m") }
                        } else {
                            Text("Estado: SIN SEÑAL", color = MaterialTheme.colorScheme.error)
                        }
                        Text("Provider: ${item.data.provider}")
                    }
                    is ActivityItem.Media -> {
                        Text("Tamaño: ${item.data.sizeBytes / 1024} KB")
                        if (!item.isRemote) {
                            Spacer(modifier = Modifier.height(8.dp))
                            AsyncImage(
                                model        = File(item.data.filePath),
                                contentDescription = null,
                                modifier     = Modifier.fillMaxWidth().height(200.dp).clip(RoundedCornerShape(8.dp)),
                                contentScale = ContentScale.Crop
                            )
                            Spacer(modifier = Modifier.height(16.dp))
                            Button(onClick = {
                                openFile(context, item.data.filePath,
                                    if (item.data.type == MediaType.PHOTO.name) "image/*" else "video/*")
                            }) {
                                Text(if (item.data.type == MediaType.PHOTO.name) "Ver Foto" else "Reproducir Video")
                            }
                        } else {
                            Text("Archivo alojado en la nube. Pendiente integración CDN.", style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.outline)
                        }
                    }
                    is ActivityItem.Audio -> {
                        Text("Duración: ${item.data.durationMs / 1000}s")
                        if (!item.isRemote) {
                            Button(onClick = { openFile(context, item.data.filePath, "audio/*") }, modifier = Modifier.padding(top = 16.dp)) {
                                Text("Reproducir Audio")
                            }
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
        confirmButton = { TextButton(onClick = onConfirm) { Text("Sí, cerrar sesión", color = MaterialTheme.colorScheme.error) } },
        dismissButton = { TextButton(onClick = onDismiss) { Text("Cancelar") } }
    )
}
```

**`NestedScreen`:** es un composable genérico que acepta `content: @Composable () -> Unit` como parámetro. Permite embeber cualquier pantalla existente (`SyncScreen`, `NotificationsScreen`) sin modificarlas. Esta técnica — pasar composables como lambdas — es el patrón "slot API" de Compose.

**`remember(selectedTab, sourceFilter, ...)` sin `LaunchedEffect`:** el recalculo de `filteredItems` se hace de forma síncrona en el hilo de composición porque usa `remember` con múltiples keys. Es aceptable para listas pequeñas. Si las listas fueran grandes, se movería a `LaunchedEffect + withContext(Dispatchers.Default)` como en Lab 5.

**Datos remotos simulados:** el bloque `remoteItems` contiene registros de ejemplo hardcodeados (`id = 999`, `id = 888`) para demostrar el filtrado LOCAL/NUBE sin necesitar un endpoint real de consulta. En producción se reemplazaría por una llamada a `apiService.getRecords(...)`.

---

## 13. Flujo completo del laboratorio

### Flujo de login

```
[LoginScreen] usuario ingresa email + password
  → sessionVm.login(email, password, onResult)
      → viewModelScope.launch
          → RetrofitClient.apiService.login(
                projectSlug = "layout_example",
                LoginRequest(email, password, deviceId = ANDROID_ID)
             )
          → HTTP POST → https://platform-api.kankunapaq.com/layout_example/auth/login
          → Response<TokenResponse>
              if (isSuccessful)
                → sessionManager.login(email, accessToken, refreshToken)
                    → DataStore: is_logged_in=true, username=email,
                                 access_token=..., refresh_token=...
                → isLoggedIn Flow emite true
                → LaunchedEffect(isLoggedIn) en Navigation.kt se dispara
                → rootNavController.navigate("main") { popUpTo("auth") inclusive=true }
                → MainScaffold con 4 tabs
```

### Flujo de registro

```
[LoginScreen] → tap "Registrar usuario"
  → rootNavController.navigate("register")
  → [RegisterScreen]
      → verifica password == confirmPassword (cliente)
      → sessionVm.register(email, password, onResult)
          → RetrofitClient.apiService.register(RegisterRequest(email, password))
          → HTTP POST → .../layout_example/auth/register
          if (isSuccessful)
              → rootNavController.popBackStack() ← vuelve a LoginScreen
              → usuario hace login normalmente
```

### Flujo de refresh token

```
[SessionViewModel.refreshSession()]
  → sessionManager.refreshToken.firstOrNull() → lee el token guardado en DataStore
  → RetrofitClient.apiService.refreshToken(
        RefreshTokenRequest(refreshToken, deviceId)
    )
  → HTTP POST → .../layout_example/auth/refresh-token
  if (isSuccessful)
    → sessionManager.updateTokens(newAccess, newRefresh)
        → DataStore: solo actualiza access_token y refresh_token
        → el resto de la sesión permanece intacto
```

---

## 14. Conceptos de las listas de temas cubiertos

| Tema | Lista | Cómo se ve en este lab |
|---|---|---|
| Navegación anidada | L1 | `navigation(route = "auth")` contiene login/register; `popUpTo("auth") inclusive=true` limpia el back stack al autenticar |
| `LaunchedEffect` | L1 | `LaunchedEffect(isLoggedIn)` navega reactivamente cuando cambia el estado de sesión |
| `SegmentedButton` | L2 | Filtro LOCAL/NUBE/TODO en `RecordsExplorerScreen` |
| `ScrollableTabRow` | L2 | Tabs Todos/GNSS/Fotos/Videos/Audios en el explorador de registros |
| Slot API de Compose | L2 | `NestedScreen(content: @Composable () -> Unit)` embebe pantallas existentes |
| Toggle de visibilidad | L2 | `VisualTransformation.None` vs `PasswordVisualTransformation()` con `Visibility`/`VisibilityOff` |
| Retrofit + kotlinx.serialization | L3 | `ApiService` con `suspend fun`, `@Path`, `@Body`; modelos con `@Serializable` y `@SerialName` |
| `OkHttpClient` + `HttpLoggingInterceptor` | L3 | Logs de red en Logcat con `Level.BODY` |
| Tokens JWT (access + refresh) | L3 | Guardados en DataStore; `updateTokens()` renueva sin borrar la sesión |
| `Settings.Secure.ANDROID_ID` | L3 | Identificador único del dispositivo enviado en cada petición de autenticación |
| Multi-tenant API | L3 | `{projectSlug}` como segmento de ruta; `NetworkConstants.PROJECT_SLUG` centraliza el tenant |
| `usesCleartextTraffic` | L3 | Habilita HTTP para APIs de desarrollo; eliminar en producción |

---

## 15. Ejercicios propuestos

1. **Mostrar el error del servidor en `LoginScreen`:** `response.errorBody()?.string()` devuelve el cuerpo de error del servidor (ej. `{"message": "Invalid credentials"}`). Parsear con `Json.decodeFromString<ErrorResponse>(body)` y mostrar el mensaje en lugar del genérico.

2. **Interceptor de autorización:** crear un `Interceptor` de OkHttp que añada el header `Authorization: Bearer $accessToken` a todas las peticiones autenticadas. Esto evita pasar el token manualmente a cada llamada.

3. **Refresh automático con `Authenticator`:** `OkHttpClient.Builder().authenticator(...)` permite interceptar respuestas 401 y llamar a `refreshSession()` de forma transparente antes de reintentar la petición fallida.

4. **Validar formato de email:** antes de llamar a la API, verificar que `email.contains("@")` y que la longitud sea razonable. Mostrar error en el campo con `isError = true` en `OutlinedTextField`.

5. **Persistir `PROJECT_SLUG` en DataStore:** permitir que el usuario ingrese el slug desde una pantalla de configuración en lugar de estar hardcodeado en `NetworkConstants`.

---

## 16. Checklist del laboratorio

- [ ] `gradle/libs.versions.toml` tiene entradas para `retrofit`, `okhttp` y `kotlinxSerializationJson`
- [ ] `build.gradle.kts` tiene `kotlin("plugin.serialization")` en `plugins`
- [ ] `build.gradle.kts` tiene las 4 dependencias de red (`retrofit-core`, `retrofit-kotlin-serialization`, `okhttp-logging`, `kotlinx-serialization-json`)
- [ ] `AndroidManifest.xml` tiene `android:usesCleartextTraffic="true"`
- [ ] `NetworkConstants` define `BASE_URL` y `PROJECT_SLUG`
- [ ] `RetrofitClient` es un `object` con `ignoreUnknownKeys = true` y `HttpLoggingInterceptor.Level.BODY`
- [ ] `ApiService` tiene 4 endpoints `suspend fun` con `@POST`, `@Path` y `@Body`
- [ ] Todos los modelos de red tienen `@Serializable` y campos con snake_case usan `@SerialName`
- [ ] `SessionManager` persiste `KEY_ACCESS_TOKEN` y `KEY_REFRESH_TOKEN` en DataStore
- [ ] `SessionManager.getDeviceId()` usa `Settings.Secure.ANDROID_ID`
- [ ] `SessionManager.login()` acepta `username, access, refresh` en su nueva firma
- [ ] `SessionManager.updateTokens()` renueva solo los tokens sin tocar el resto
- [ ] `SessionViewModel.login()` llama a Retrofit (no hay comparación hardcodeada)
- [ ] `SessionViewModel` tiene `register()`, `loginWithGoogle()` y `refreshSession()`
- [ ] `LoginScreen` tiene campo `Email`, toggle de visibilidad y botón "Registrar usuario"
- [ ] `RegisterScreen` valida `password == confirmPassword` antes de llamar a la API
- [ ] `Navigation.kt` tiene `navigation(route = "auth")` con `composable("login")` y `composable("register")`
- [ ] `Navigation.kt` tiene `LaunchedEffect(isLoggedIn)` con `popUpTo` que limpia el back stack
- [ ] La `NavigationBar` tiene 4 tabs (sin Sync ni Notif)
- [ ] `ProfileScreen` tiene `sealed class ProfileViewState` con 6 estados
- [ ] `RecordsExplorerScreen` filtra por `RecordsSource` (LOCAL/REMOTE/ALL) con `SegmentedButton`
- [ ] `RecordsExplorerScreen` filtra por categoría con `ScrollableTabRow`
- [ ] `NestedScreen` embebe `SyncScreen()` y `NotificationsScreen()` desde `ProfileScreen`
- [ ] `ActivityItem` tiene `isRemote: Boolean` en todos los subtipos
- [ ] `ActivityRow` diferencia visualmente ítems remotos con `colorScheme.tertiary` y chip "Cloud"
- [ ] `MyProfileScreen` muestra el `Android ID` del dispositivo
- [ ] La app compila, hace login con credenciales reales de la API y muestra el email en el `TopAppBar`
