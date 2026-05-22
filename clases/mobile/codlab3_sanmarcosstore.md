# 🛠️ CodLab N°3 — SanMarcosStore
### App Android con Material Design 3, Tema personalizado y Navegación entre pantallas

> 📌 **Objetivo del laboratorio:** Construir desde cero una app Android llamada **SanMarcosStore** que use:
> - Tu paleta personalizada generada en [Material Theme Builder](https://m3.material.io/theme-builder).
> - Componentes Material 3 (TopAppBar, Cards, Botones, TextField, LazyColumn, ListItem, Switches, NavigationBar).
> - Navegación entre 2 pantallas: **Tienda** y **Mi Perfil**.
> - Arquitectura modular (código organizado en múltiples archivos).

---

## 🎯 Pre-requisitos

- ✅ Android Studio **Hedgehog 2023.1.1** o superior (recomendado: Iguana o Koala).
- ✅ Archivo `material-theme.zip` descargado del [Theme Builder](https://m3.material.io/theme-builder).
- ✅ JDK 17 instalado (Android Studio lo trae por defecto).
- ✅ Emulador o dispositivo físico con Android 7.0+ (API 24).

---

## 📋 PASO 1 — Crear el proyecto

### 1.1 Abrir el wizard

**File → New → New Project** (o desde la pantalla de bienvenida: **"New Project"**).

### 1.2 ⚠️ ELEGIR EL TEMPLATE CORRECTO

Esta es la decisión más importante del laboratorio. Android Studio te muestra varios templates con nombres parecidos. **Solo uno es el correcto:**

| Template | ¿Lo elijo? | Razón |
|---|---|---|
| ✅ **Empty Activity** | **SÍ** | Compose limpio + MainActivity con tema vacío |
| ❌ Empty Views Activity | NO | Es XML viejo (sistema "Views"), no Compose |

> 🚨 **TRAMPA COMÚN:** Si eliges un template que contenga la palabra **"Views"** en el nombre, terminas con archivos `.xml` y `fragment_*.kt` — un mundo completamente distinto a lo que vimos en clase. **Cierra el proyecto y empieza de cero** si pasa esto.

**Cómo identificar el template correcto:**
- Su icono suele mostrar fragmentos de código `{ }`.
- Su descripción dice *"Creates a new empty activity with Jetpack Compose"*.
- Se llama **"Empty Activity"** a secas (sin "Views").

Selecciona **"Empty Activity"** → click **"Next"**.

### 1.3 Configurar el proyecto

Llena los campos así (copia exactamente):

| Campo | Valor |
|---|---|
| **Name** | `SanMarcosStore` |
| **Package name** | `com.example.sanmarcosstore` |
| **Save location** | (donde quieras guardarlo) |
| **Language** | Kotlin |
| **Minimum SDK** | API 24 (Android 7.0) |
| **Build configuration language** | Kotlin DSL (build.gradle.kts) |

Click **"Finish"** y espera 1–3 minutos al **Gradle sync inicial**.

### 1.4 Verificar que el proyecto nació con Compose

Abre `app/build.gradle.kts` y verifica que contenga estas líneas:

```kotlin
android {
    // ...
    buildFeatures {
        compose = true   // ← confirma que es Compose
    }
}

dependencies {
    // ...
    
    implementation("androidx.compose.material3:material3")
    implementation("androidx.activity:activity-compose:1.10.0")
    implementation("androidx.compose.ui:ui-text-google-fonts:1.7.8")
    implementation("androidx.compose.material:material-icons-extended:1.7.8")
    implementation("androidx.navigation:navigation-compose:2.8.0")

    // ...
}
```

Si **NO** ves estas líneas → elegiste el template incorrecto. Cierra el proyecto sin guardar y vuelve al paso 1.2.

---

## 📋 PASO 2 — Agregar las dependencias necesarias

El template **"Empty Activity"** no incluye navegación, Google Fonts ni iconos extendidos. Las agregamos manualmente.

Abre `app/build.gradle.kts` y dentro del bloque `dependencies { ... }` agrega estas **4 líneas**:

```kotlin
dependencies {
    // ... lo que ya tienes ...

    // Activity Compose (versión estable usada en clase)
    implementation("androidx.activity:activity-compose:1.10.0")

    // Tipografía descargable de Google Fonts (la usa el Theme Builder)
    implementation("androidx.compose.ui:ui-text-google-fonts:1.7.8")

    // Iconos extendidos de Material (Search, Favorite, ShoppingCart, Person, etc.)
    implementation("androidx.compose.material:material-icons-extended:1.7.8")

    // Navegación entre pantallas
    implementation("androidx.navigation:navigation-compose:2.8.0")
}
```

> 📌 **Versiones verificadas:** estas son las versiones exactas con las que el laboratorio se probó en clase. Si Android Studio te sugiere actualizar a una versión más nueva, **acepta** — son compatibles hacia atrás.

> 💡 **¿Por qué `material-icons-extended`?** Compose trae solo unos pocos iconos básicos por defecto. Para usar `Icons.Filled.Search`, `Icons.Filled.Favorite`, `Icons.Filled.ShoppingCart`, `Icons.Filled.Person` (todos los del laboratorio), necesitas el paquete extended con más de 2,000 iconos. Si lo olvidas, verás el error `Unresolved reference: Icons` al usarlos.

Click en **"Sync Now"** (banner amarillo en la parte superior) y espera a que termine la sincronización.

---

## 📋 PASO 3 — Integrar el tema del Material Theme Builder

### 3.1 Descomprimir el ZIP

Descomprime `material-theme.zip` en cualquier carpeta temporal. Vas a ver:

```
material-theme/
├── README.md
├── ui/theme/
│   ├── Color.kt    ← reemplaza el del proyecto
│   ├── Theme.kt    ← reemplaza el del proyecto
│   └── Type.kt     ← reemplaza el del proyecto (trae tipografía descargable)
└── res/values-v23/
    └── font_certs.xml    ← necesario para la tipografía Google Fonts
```

### 3.2 Cambiar el package en los 3 archivos del ZIP

> ⚠️ **¡Paso crítico!** El ZIP viene con el package `com.example.compose`, pero tu proyecto usa otro. Si no lo cambias, tendrás errores rojos por todos lados.

**Abre `Color.kt`** (del ZIP, con cualquier editor de texto) y cambia la primera línea:

```kotlin
// ANTES:
package com.example.compose

// DESPUÉS:
package com.example.sanmarcosstore.ui.theme
```

**Haz exactamente lo mismo en `Theme.kt` y `Type.kt`** — los 3 archivos deben tener:

```kotlin
package com.example.sanmarcosstore.ui.theme
```

### 3.3 Reemplazar los archivos en tu proyecto

En Android Studio, navega hasta `app/src/main/java/com/example/sanmarcosstore/ui/theme/`. Verás 3 archivos:
- `Color.kt`
- `Theme.kt`
- `Type.kt`

1. **Borra** `Color.kt` del proyecto → **arrastra** `Color.kt` del ZIP a esa carpeta. Cuando Android Studio pregunte si quieres copiar, di **"OK"**.
2. **Borra** `Theme.kt` del proyecto → **arrastra** `Theme.kt` del ZIP a esa carpeta.
3. **Borra** `Type.kt` del proyecto → **arrastra** `Type.kt` del ZIP a esa carpeta.

### 3.4 Copiar el recurso `font_certs.xml`

El `Type.kt` del ZIP usa una **tipografía descargable de Google Fonts** que necesita un archivo XML de certificados para funcionar.

1. En Android Studio cambia la vista del Project (esquina superior izquierda) de **"Android"** a **"Project"** — necesitas esta vista para navegar carpetas reales.
2. Navega hasta `app/src/main/res/`.
3. Si **NO existe** la carpeta `values-v23`, créala:
   - Click derecho en `res` → **New → Android Resource Directory**.
   - **Resource type:** `values`.
   - En **"Available qualifiers"** busca **"Version"** → click en `>>` → en la derecha pon **23**.
   - El nombre del directorio cambiará automáticamente a `values-v23`.
   - Click **OK**.
4. **Arrastra** `font_certs.xml` del ZIP a `app/src/main/res/values-v23/`.

### 3.5 ⚠️ Agregar el import faltante en `Type.kt` (ERROR COMÚN)

> 🚨 **¡Atención! Este es el error #1 del laboratorio.** El `Type.kt` del ZIP usa `R.array.com_google_android_gms_fonts_certs` pero **no incluye el import de `R`**. Si no lo agregas, verás este error rojo:
>
> ```
> Type.kt:14:20 Unresolved reference 'R'.
> ```

**Solución:** Abre el `Type.kt` que copiaste y agrega esta línea entre los demás `import`:

```kotlin
package com.example.sanmarcosstore.ui.theme

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
// ... otros imports que ya tiene el archivo ...
import com.example.sanmarcosstore.R   // ← AGREGAR ESTA LÍNEA
```

> 💡 **¿Por qué pasa esto?** El ZIP del Theme Builder se genera asumiendo el package `com.example.compose`. Cuando cambias el package al de tu app (paso 3.2), el `R` deja de resolverse automáticamente y hay que importarlo manualmente con la ruta correcta.

**Truco rápido en Android Studio:** coloca el cursor sobre la palabra `R` subrayada en rojo, presiona **`Alt + Enter`** (Windows/Linux) o **`Option + Enter`** (Mac) y elige **"Import"** → selecciona **`com.example.sanmarcosstore.R`**. Android Studio agrega el import automáticamente.

### 3.6 Verificar el nombre de la función del tema

Abre el `Theme.kt` que copiaste y busca cerca del final esta función:

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    // ... contenido ...
}
```

✅ **Déjala con el nombre `AppTheme` tal como viene del ZIP.** No la renombres.

> 💡 **¿Por qué mantenemos `AppTheme`?** Es el nombre estándar que genera el Material Theme Builder. Renombrarla por cada proyecto agrega trabajo innecesario y no aporta valor — el alcance del nombre ya está limitado por el package `com.example.sanmarcosstore.ui.theme`. Cuando importes `AppTheme` en `MainActivity.kt`, Android Studio sabrá exactamente cuál es porque pertenece a tu app.

### 3.7 Sincronizar Gradle

**File → Sync Project with Gradle Files** (o click en el botón con el icono de elefante en la barra superior).

✅ Si no aparecen errores rojos, el tema está integrado correctamente.

> 📌 **Resumen de archivos del ZIP integrados:**
> - ✅ `Color.kt` → `app/src/main/java/com/example/sanmarcosstore/ui/theme/`
> - ✅ `Theme.kt` → `app/src/main/java/com/example/sanmarcosstore/ui/theme/`
> - ✅ `Type.kt` → `app/src/main/java/com/example/sanmarcosstore/ui/theme/` (con `import com.example.sanmarcosstore.R` añadido)
> - ✅ `font_certs.xml` → `app/src/main/res/values-v23/`
> - ✅ Dependencia `ui-text-google-fonts:1.7.8` agregada a `build.gradle.kts`

---

## 📋 PASO 4 — Test rápido: "Hola SanMarcosStore"

Antes de armar la arquitectura, vamos a verificar que el tema funciona. Abre `MainActivity.kt` y reemplaza todo su contenido por:

```kotlin
package com.example.sanmarcosstore

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import com.example.sanmarcosstore.ui.theme.AppTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme(dynamicColor = false) {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    Text(
                        text = "¡Hola SanMarcosStore!",
                        style = MaterialTheme.typography.headlineMedium,
                        color = MaterialTheme.colorScheme.primary,
                        modifier = Modifier.padding(24.dp)
                    )
                }
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun PreviewSaludo() {
    AppTheme(dynamicColor = false) {
        Surface(color = MaterialTheme.colorScheme.background) {
            Text(
                "¡Hola SanMarcosStore!",
                style = MaterialTheme.typography.headlineMedium,
                color = MaterialTheme.colorScheme.primary,
                modifier = Modifier.padding(24.dp)
            )
        }
    }
}
```

### 🔧 ¿Por qué `dynamicColor = false`?

Si dejas `dynamicColor = true`, en Android 12+ Compose **ignora tu paleta** y usa los colores del wallpaper del usuario. Para **ver tu tema personalizado** lo dejamos en `false`.

### 📛 ¿Tu proyecto se llama distinto?

Este laboratorio usa el nombre `SanMarcosStore` (package `com.example.sanmarcosstore`). Si tú elegiste otro nombre al crear el proyecto (por ejemplo `illareklab.smstore`), debes **adaptar todos los imports** a tu package:

```kotlin
// EJEMPLO con otro package
import com.illareklab.smstore.ui.theme.AppTheme       // ← usa TU package
import com.illareklab.smstore.R                        // ← en Type.kt
```

> 💡 **Regla general:** todos los `import com.example.sanmarcosstore.*` de este documento se reemplazan por `import <TU_PACKAGE>.*`. Lo demás (los `import androidx.*`, los nombres de Composables como `AppTheme`, `Surface`, `Text`) **se quedan exactamente igual**.

### ▶️ Ejecutar

1. En la barra superior, abre el panel **"Preview"** (icono 👁️ a la derecha del editor).
2. Verás un saludo con el color **primary** de tu paleta sobre el fondo background.

✅ Si funciona → continuamos. Si no → revisa la sección **Troubleshooting** al final.

---

## 📋 PASO 5 — Crear la estructura de carpetas

Vamos a organizar el código en módulos por responsabilidad. La estructura final será:

```
com.example.sanmarcosstore/
├── MainActivity.kt               ← Entry point (15 líneas)
├── model/
│   └── Producto.kt               ← Modelo de datos
├── ui/
│   ├── components/
│   │   └── ProductoItem.kt       ← Componente reutilizable
│   ├── screens/
│   │   ├── TiendaScreen.kt       ← Pantalla 1
│   │   └── PerfilScreen.kt       ← Pantalla 2
│   ├── navigation/
│   │   └── AppNavigation.kt      ← Configuración de rutas
│   └── theme/                     ← (ya existe del paso 3)
│       ├── Color.kt
│       ├── Theme.kt
│       └── Type.kt
```

### Crear los packages

En el panel izquierdo (Project), expande `app/src/main/java/com/example/sanmarcosstore`. Click derecho sobre `sanmarcosstore`:

1. **New → Package** → escribe `model` → Enter.
2. **New → Package** → escribe `ui.components` → Enter.
3. **New → Package** → escribe `ui.screens` → Enter.
4. **New → Package** → escribe `ui.navigation` → Enter.

> 💡 Cuando escribes `ui.components` con punto, Android Studio crea la jerarquía anidada automáticamente.

---

## 📋 PASO 6 — Crear `Producto.kt` (modelo de datos)

Click derecho en el package `model` → **New → Kotlin Class/File** → escribe `Producto` → tipo **File**.

Pega esto:

```kotlin
package com.example.sanmarcosstore.model

data class Producto(
    val id: Int,
    val nombre: String,
    val precio: String,
    val categoria: String,
    val favorito: Boolean = false
)

// Datos de prueba para la lista
val productosDePrueba = listOf(
    Producto(1, "Café Premium 500g", "S/ 45.00", "Bebidas", true),
    Producto(2, "Chocolate artesanal 70%", "S/ 28.00", "Dulces"),
    Producto(3, "Pan de masa madre", "S/ 18.00", "Panadería"),
    Producto(4, "Miel de abeja orgánica", "S/ 35.00", "Endulzantes", true),
    Producto(5, "Mermelada de fresa", "S/ 22.00", "Conservas"),
    Producto(6, "Té verde matcha", "S/ 55.00", "Bebidas"),
    Producto(7, "Galletas de avena", "S/ 15.00", "Dulces"),
    Producto(8, "Aceite de oliva extra virgen", "S/ 68.00", "Aceites")
)
```

---

## 📋 PASO 7 — Crear `ProductoItem.kt` (componente reutilizable)

Click derecho en `ui.components` → **New → Kotlin Class/File** → `ProductoItem` (tipo File).

```kotlin
package com.example.sanmarcosstore.ui.components

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.size
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Favorite
import androidx.compose.material.icons.filled.ShoppingCart
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.Icon
import androidx.compose.material3.ListItem
import androidx.compose.material3.ListItemDefaults
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.example.sanmarcosstore.model.Producto

@Composable
fun ProductoItem(producto: Producto) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceContainerLow
        ),
        elevation = CardDefaults.cardElevation(defaultElevation = 1.dp)
    ) {
        ListItem(
            headlineContent = {
                Text(producto.nombre, style = MaterialTheme.typography.titleMedium)
            },
            supportingContent = {
                Text(producto.categoria, style = MaterialTheme.typography.bodySmall)
            },
            trailingContent = {
                Column(horizontalAlignment = Alignment.End) {
                    Text(
                        producto.precio,
                        style = MaterialTheme.typography.labelLarge,
                        color = MaterialTheme.colorScheme.primary
                    )
                    if (producto.favorito) {
                        Icon(
                            Icons.Filled.Favorite,
                            contentDescription = "Producto favorito",
                            tint = MaterialTheme.colorScheme.tertiary,
                            modifier = Modifier.size(20.dp)
                        )
                    }
                }
            },
            leadingContent = {
                Icon(
                    Icons.Filled.ShoppingCart,
                    contentDescription = null,
                    tint = MaterialTheme.colorScheme.primary
                )
            },
            colors = ListItemDefaults.colors(
                containerColor = MaterialTheme.colorScheme.surfaceContainerLow
            )
        )
    }
}
```

---

## 📋 PASO 8 — Crear `TiendaScreen.kt` (pantalla principal)

Click derecho en `ui.screens` → **New → Kotlin Class/File** → `TiendaScreen`.

```kotlin
package com.example.sanmarcosstore.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material.icons.filled.Error
import androidx.compose.material.icons.filled.Search
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.ElevatedButton
import androidx.compose.material3.FilledTonalButton
import androidx.compose.material3.FloatingActionButton
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.example.sanmarcosstore.model.productosDePrueba
import com.example.sanmarcosstore.ui.components.ProductoItem

@Composable
fun TiendaScreen() {
    var busqueda by remember { mutableStateOf("") }
    var mostrarFormulario by remember { mutableStateOf(false) }

    Box(modifier = Modifier.fillMaxSize()) {
        Column(modifier = Modifier.fillMaxSize()) {

            // ── Campo de búsqueda ──
            OutlinedTextField(
                value = busqueda,
                onValueChange = { busqueda = it },
                label = { Text("Buscar producto") },
                placeholder = { Text("Ej: Café, Chocolate…") },
                leadingIcon = {
                    Icon(Icons.Filled.Search, contentDescription = null)
                },
                supportingText = {
                    Text("Escribe para filtrar la lista")
                },
                singleLine = true,
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp)
            )

            // ── Formulario "agregar producto rápido" (toggle por FAB) ──
            if (mostrarFormulario) {
                FormularioRapido()
            }

            // ── Lista filtrada ──
            val productosFiltrados = productosDePrueba.filter {
                it.nombre.contains(busqueda, ignoreCase = true) ||
                it.categoria.contains(busqueda, ignoreCase = true)
            }

            if (productosFiltrados.isEmpty()) {
                Box(
                    modifier = Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center
                ) {
                    Text(
                        "No se encontraron productos",
                        style = MaterialTheme.typography.bodyLarge,
                        color = MaterialTheme.colorScheme.onSurfaceVariant
                    )
                }
            } else {
                LazyColumn(
                    contentPadding = PaddingValues(
                        start = 16.dp,
                        end = 16.dp,
                        top = 8.dp,
                        bottom = 88.dp   // espacio para el FAB
                    ),
                    verticalArrangement = Arrangement.spacedBy(8.dp)
                ) {
                    items(productosFiltrados, key = { it.id }) { producto ->
                        ProductoItem(producto = producto)
                    }
                }
            }
        }

        // ── FAB para mostrar/ocultar formulario ──
        FloatingActionButton(
            onClick = { mostrarFormulario = !mostrarFormulario },
            containerColor = MaterialTheme.colorScheme.primary,
            contentColor = MaterialTheme.colorScheme.onPrimary,
            modifier = Modifier
                .align(Alignment.BottomEnd)
                .padding(16.dp)
        ) {
            Icon(Icons.Filled.Add, contentDescription = "Agregar producto")
        }
    }
}

// ────────────────────────────────────────────────────────────
// Formulario con los 5 tipos de botón de Material 3
// ────────────────────────────────────────────────────────────
@Composable
private fun FormularioRapido() {
    var nombreProducto by remember { mutableStateOf("") }
    val esError = nombreProducto.length > 30

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceContainer
        )
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                "Agregar producto rápido",
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.onSurface
            )

            Spacer(modifier = Modifier.height(8.dp))

            OutlinedTextField(
                value = nombreProducto,
                onValueChange = { nombreProducto = it },
                label = { Text("Nombre del producto") },
                supportingText = {
                    if (esError) {
                        Text(
                            "Máximo 30 caracteres",
                            color = MaterialTheme.colorScheme.error
                        )
                    } else {
                        Text("${nombreProducto.length}/30")
                    }
                },
                trailingIcon = {
                    if (esError) {
                        Icon(
                            Icons.Filled.Error,
                            contentDescription = "Error de longitud",
                            tint = MaterialTheme.colorScheme.error
                        )
                    }
                },
                isError = esError,
                singleLine = true,
                modifier = Modifier.fillMaxWidth()
            )

            Spacer(modifier = Modifier.height(12.dp))

            Text(
                "Acciones (los 5 niveles de énfasis):",
                style = MaterialTheme.typography.labelMedium
            )
            Spacer(modifier = Modifier.height(4.dp))

            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.spacedBy(4.dp)
            ) {
                Button(
                    onClick = { /* guardar */ },
                    modifier = Modifier.weight(1f)
                ) {
                    Text("Guardar", style = MaterialTheme.typography.labelSmall)
                }
                FilledTonalButton(
                    onClick = { /* borrador */ },
                    modifier = Modifier.weight(1f)
                ) {
                    Text("Borrador", style = MaterialTheme.typography.labelSmall)
                }
            }

            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.spacedBy(4.dp)
            ) {
                ElevatedButton(
                    onClick = { /* preview */ },
                    modifier = Modifier.weight(1f)
                ) {
                    Text("Vista previa", style = MaterialTheme.typography.labelSmall)
                }
                OutlinedButton(
                    onClick = { nombreProducto = "" },
                    modifier = Modifier.weight(1f)
                ) {
                    Text("Limpiar", style = MaterialTheme.typography.labelSmall)
                }
            }

            TextButton(onClick = { /* cancelar */ }) {
                Text("Cancelar")
            }
        }
    }
}
```

---

## 📋 PASO 9 — Crear `PerfilScreen.kt` (pantalla "Mi Perfil")

Click derecho en `ui.screens` → **New → Kotlin Class/File** → `PerfilScreen`.

```kotlin
package com.example.sanmarcosstore.ui.screens

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ExitToApp
import androidx.compose.material.icons.automirrored.filled.KeyboardArrowRight
import androidx.compose.material.icons.filled.Favorite
import androidx.compose.material.icons.filled.LocationOn
import androidx.compose.material.icons.filled.Person
import androidx.compose.material.icons.filled.Receipt
import androidx.compose.material.icons.filled.Settings
import androidx.compose.material3.ButtonDefaults
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.HorizontalDivider
import androidx.compose.material3.Icon
import androidx.compose.material3.ListItem
import androidx.compose.material3.ListItemDefaults
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Switch
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.unit.dp

// Modelo simple para las opciones del menú
data class OpcionPerfil(
    val id: Int,
    val titulo: String,
    val descripcion: String,
    val icono: ImageVector
)

private val opcionesPerfil = listOf(
    OpcionPerfil(1, "Mis pedidos", "Historial de compras", Icons.Filled.Receipt),
    OpcionPerfil(2, "Mis favoritos", "Productos que te gustan", Icons.Filled.Favorite),
    OpcionPerfil(3, "Direcciones", "Gestiona tus envíos", Icons.Filled.LocationOn),
    OpcionPerfil(4, "Configuración", "Notificaciones, idioma", Icons.Filled.Settings)
)

@Composable
fun PerfilScreen() {
    var modoOscuro by remember { mutableStateOf(false) }
    var notificaciones by remember { mutableStateOf(true) }

    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {

        // ── Cabecera con avatar y datos del usuario ──
        item {
            Card(
                modifier = Modifier.fillMaxWidth(),
                colors = CardDefaults.cardColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                )
            ) {
                Row(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(20.dp),
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    // Avatar circular
                    Box(
                        modifier = Modifier
                            .size(72.dp)
                            .clip(CircleShape)
                            .background(MaterialTheme.colorScheme.primary),
                        contentAlignment = Alignment.Center
                    ) {
                        Icon(
                            Icons.Filled.Person,
                            contentDescription = "Foto de perfil",
                            tint = MaterialTheme.colorScheme.onPrimary,
                            modifier = Modifier.size(40.dp)
                        )
                    }

                    Spacer(modifier = Modifier.width(16.dp))

                    Column {
                        Text(
                            "Ana Martínez",
                            style = MaterialTheme.typography.titleLarge,
                            color = MaterialTheme.colorScheme.onPrimaryContainer
                        )
                        Text(
                            "ana.martinez@correo.com",
                            style = MaterialTheme.typography.bodyMedium,
                            color = MaterialTheme.colorScheme.onPrimaryContainer
                        )
                        Spacer(modifier = Modifier.height(4.dp))
                        Text(
                            "Cliente desde 2024",
                            style = MaterialTheme.typography.labelSmall,
                            color = MaterialTheme.colorScheme.onPrimaryContainer
                        )
                    }
                }
            }
        }

        // ── Estadísticas rápidas ──
        item {
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                EstadisticaCard("12", "Pedidos", Modifier.weight(1f))
                EstadisticaCard("5", "Favoritos", Modifier.weight(1f))
                EstadisticaCard("3", "Cupones", Modifier.weight(1f))
            }
        }

        // ── Título de sección "Mi cuenta" ──
        item {
            Text(
                "Mi cuenta",
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.onSurface,
                modifier = Modifier.padding(top = 8.dp, bottom = 4.dp)
            )
        }

        // ── Lista de opciones ──
        items(opcionesPerfil, key = { it.id }) { opcion ->
            Card(
                modifier = Modifier.fillMaxWidth(),
                colors = CardDefaults.cardColors(
                    containerColor = MaterialTheme.colorScheme.surfaceContainerLow
                )
            ) {
                ListItem(
                    headlineContent = { Text(opcion.titulo) },
                    supportingContent = { Text(opcion.descripcion) },
                    leadingContent = {
                        Icon(
                            opcion.icono,
                            contentDescription = null,
                            tint = MaterialTheme.colorScheme.primary
                        )
                    },
                    trailingContent = {
                        Icon(
                            Icons.AutoMirrored.Filled.KeyboardArrowRight,
                            contentDescription = "Ir a ${opcion.titulo}",
                            tint = MaterialTheme.colorScheme.onSurfaceVariant
                        )
                    },
                    colors = ListItemDefaults.colors(
                        containerColor = MaterialTheme.colorScheme.surfaceContainerLow
                    )
                )
            }
        }

        // ── Sección "Preferencias" con Switches ──
        item {
            Text(
                "Preferencias",
                style = MaterialTheme.typography.titleMedium,
                modifier = Modifier.padding(top = 8.dp, bottom = 4.dp)
            )
        }

        item {
            Card(
                colors = CardDefaults.cardColors(
                    containerColor = MaterialTheme.colorScheme.surfaceContainerLow
                )
            ) {
                Column {
                    ListItem(
                        headlineContent = { Text("Modo oscuro") },
                        supportingContent = { Text("Cambia el aspecto de la app") },
                        trailingContent = {
                            Switch(
                                checked = modoOscuro,
                                onCheckedChange = { modoOscuro = it }
                            )
                        },
                        colors = ListItemDefaults.colors(
                            containerColor = MaterialTheme.colorScheme.surfaceContainerLow
                        )
                    )
                    HorizontalDivider(
                        color = MaterialTheme.colorScheme.outlineVariant
                    )
                    ListItem(
                        headlineContent = { Text("Notificaciones") },
                        supportingContent = { Text("Recibe alertas de promociones") },
                        trailingContent = {
                            Switch(
                                checked = notificaciones,
                                onCheckedChange = { notificaciones = it }
                            )
                        },
                        colors = ListItemDefaults.colors(
                            containerColor = MaterialTheme.colorScheme.surfaceContainerLow
                        )
                    )
                }
            }
        }

        // ── Botón cerrar sesión ──
        item {
            Spacer(modifier = Modifier.height(16.dp))
            OutlinedButton(
                onClick = { /* cerrar sesión */ },
                modifier = Modifier.fillMaxWidth(),
                colors = ButtonDefaults.outlinedButtonColors(
                    contentColor = MaterialTheme.colorScheme.error
                )
            ) {
                Icon(
                    Icons.AutoMirrored.Filled.ExitToApp,
                    contentDescription = null
                )
                Spacer(modifier = Modifier.width(8.dp))
                Text("Cerrar sesión")
            }
        }
    }
}

// ────────────────────────────────────────────────────────────
// Componente auxiliar — card de estadística
// ────────────────────────────────────────────────────────────
@Composable
private fun EstadisticaCard(
    valor: String,
    etiqueta: String,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier,
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.tertiaryContainer
        )
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .padding(12.dp),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Text(
                valor,
                style = MaterialTheme.typography.headlineMedium,
                color = MaterialTheme.colorScheme.onTertiaryContainer
            )
            Text(
                etiqueta,
                style = MaterialTheme.typography.labelMedium,
                color = MaterialTheme.colorScheme.onTertiaryContainer
            )
        }
    }
}
```

---

## 📋 PASO 10 — Crear `AppNavigation.kt` (el router)

Click derecho en `ui.navigation` → **New → Kotlin Class/File** → `AppNavigation`.

```kotlin
package com.example.sanmarcosstore.ui.navigation

import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Person
import androidx.compose.material.icons.filled.Store
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.NavigationBar
import androidx.compose.material3.NavigationBarItem
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.material3.TopAppBarDefaults
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.navigation.NavDestination.Companion.hierarchy
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.currentBackStackEntryAsState
import androidx.navigation.compose.rememberNavController
import com.example.sanmarcosstore.ui.screens.PerfilScreen
import com.example.sanmarcosstore.ui.screens.TiendaScreen

// Cada pantalla se identifica con una "ruta" única
sealed class Ruta(
    val ruta: String,
    val etiqueta: String,
    val icono: ImageVector
) {
    data object Tienda : Ruta("tienda", "Tienda", Icons.Filled.Store)
    data object Perfil : Ruta("perfil", "Mi Perfil", Icons.Filled.Person)
}

private val pestañas = listOf(Ruta.Tienda, Ruta.Perfil)

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    val currentBackStack by navController.currentBackStackEntryAsState()
    val currentDestination = currentBackStack?.destination

    // Título dinámico según la pantalla activa
    val tituloActual = when (currentDestination?.route) {
        Ruta.Tienda.ruta -> "SanMarcos Store"
        Ruta.Perfil.ruta -> "Mi Perfil"
        else -> "SanMarcos Store"
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text(tituloActual) },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        },
        bottomBar = {
            NavigationBar(
                containerColor = MaterialTheme.colorScheme.surfaceContainer
            ) {
                pestañas.forEach { pestaña ->
                    val seleccionada = currentDestination
                        ?.hierarchy?.any { it.route == pestaña.ruta } == true

                    NavigationBarItem(
                        selected = seleccionada,
                        onClick = {
                            navController.navigate(pestaña.ruta) {
                                // Evita acumular instancias en el back stack
                                popUpTo(navController.graph.startDestinationId) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        },
                        icon = {
                            Icon(pestaña.icono, contentDescription = pestaña.etiqueta)
                        },
                        label = { Text(pestaña.etiqueta) }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = Ruta.Tienda.ruta,
            modifier = Modifier.padding(padding)
        ) {
            composable(Ruta.Tienda.ruta) { TiendaScreen() }
            composable(Ruta.Perfil.ruta) { PerfilScreen() }
        }
    }
}
```

### 🔧 ¿Qué hacen `popUpTo` + `launchSingleTop` + `restoreState`?

Son las **3 banderas estándar** del patrón de bottom navigation. Juntas hacen que:
- ✅ Al cambiar de tab, no se acumulen pantallas en el historial.
- ✅ Cada tab "recuerde" su estado al volver (por ejemplo, la búsqueda escrita).
- ✅ El botón "atrás" del sistema cierre la app desde el tab raíz, en vez de saltar entre tabs.

---

## 📋 PASO 11 — Simplificar `MainActivity.kt`

Vuelve a `MainActivity.kt` y reemplaza **todo** su contenido por esta versión minimalista:

```kotlin
package com.example.sanmarcosstore

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import com.example.sanmarcosstore.ui.navigation.AppNavigation
import com.example.sanmarcosstore.ui.theme.AppTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme(dynamicColor = false) {
                AppNavigation()
            }
        }
    }
}
```

> 🎯 **MainActivity ya no decide qué pantalla mostrar.** Solo aplica el tema y entrega el control al sistema de navegación. Esa separación de responsabilidades es la clave de la arquitectura modular.

---

## 📋 PASO 12 — Ejecutar y verificar

### 12.1 Build completo

**Build → Rebuild Project** (esto te dirá si hay errores antes de ejecutar).

### 12.2 Ejecutar

1. Si no tienes emulador: **Tools → Device Manager → Create Virtual Device** → elige **Pixel 7** + **API 34** (Android 14).
2. Click ▶️ **Run** (o `Shift+F10`).

### 12.3 ✅ Comportamiento esperado

| Acción | Resultado esperado |
|---|---|
| Abrir la app | Aparece la pantalla **Tienda** con la lista de productos sobre fondo crema rosado |
| Tap en el FAB ➕ | Aparece/desaparece el formulario "Agregar producto rápido" con los 5 botones |
| Escribir en el buscador | La lista se filtra en tiempo real |
| Tap en tab "Mi Perfil" (abajo) | Aparece pantalla con avatar, estadísticas, opciones y switches |
| Toggle Switches | Cambian visualmente |
| Volver a tab "Tienda" | La búsqueda escrita **se conserva** |

---

## 🐛 Troubleshooting

| Problema | Causa | Solución |
|---|---|---|
| 🔥 **`Type.kt:14:20 Unresolved reference 'R'`** ⭐ | El `Type.kt` del ZIP usa `R` pero no lo importa | Agregar `import com.example.sanmarcosstore.R` al inicio del archivo (paso 3.5). Truco: `Alt+Enter` sobre la `R` roja → Import |
| `Unresolved reference: AppTheme` | No cambiaste el package del ZIP | Verifica que `Color.kt`, `Theme.kt` y `Type.kt` tengan `package com.example.sanmarcosstore.ui.theme` |
| `Unresolved reference: primaryLight` | El package de `Color.kt` no coincide con el de `Theme.kt` | Los 3 archivos deben tener exactamente el mismo package |
| `Unresolved reference: Icons` o `Icons.Filled.Search` en rojo | Falta la dependencia de iconos extendidos | Agrega `implementation("androidx.compose.material:material-icons-extended:1.7.8")` en `build.gradle.kts` y sincroniza |
| `Unresolved reference: GoogleFont` | Falta la dependencia | Agrega `implementation("androidx.compose.ui:ui-text-google-fonts:1.7.8")` en `build.gradle.kts` |
| `Resource not found: com_google_android_gms_fonts_certs` | No copiaste `font_certs.xml` | Verifica que el archivo esté en `app/src/main/res/values-v23/font_certs.xml` |
| Los colores se ven azules en vez de terracota | `dynamicColor = true` en Android 12+ | Ponlo en `false` en `MainActivity.kt` |
| `cannot find symbol AppTheme` | Faltó el import en MainActivity | Asegúrate que el import diga exactamente `import com.example.sanmarcosstore.ui.theme.AppTheme` (usa tu nombre de proyecto si es distinto) |
| Falta `@OptIn(ExperimentalMaterial3Api::class)` | Algunos componentes (TopAppBar) son experimentales | Agrégalo arriba del `@Composable` que los usa |
| El preview no se renderiza | Falta sincronizar Gradle | **File → Sync Project with Gradle Files** |
| Error `data object` no compila | Versión vieja de Kotlin | Cambia `data object` por `object` en `AppNavigation.kt` |
| `NavigationBar` no se ve | Olvidaste la dependencia de Navigation | Verifica `androidx.navigation:navigation-compose:2.8.0` en `build.gradle.kts` y sincroniza |
| Crash al iniciar la app | Falta el import de algún Composable | Revisa la pestaña Logcat (parte inferior) para ver la línea exacta |
| Sigue sin compilar después de varios intentos | Cache de Android Studio corrupto | **File → Invalidate Caches… → Invalidate and Restart** |

> ⭐ **El error marcado** es el más común del laboratorio. Si te aparece, sigue exactamente el paso 3.5.

---

## ✅ Checklist final

- [ ] Proyecto creado con template **Empty Activity** (con Compose).
- [ ] Dependencia `navigation-compose:2.8.0` agregada.
- [ ] Dependencia `ui-text-google-fonts:1.7.8` agregada.
- [ ] Dependencia `material-icons-extended:1.7.8` agregada.
- [ ] Dependencia `activity-compose:1.10.0` agregada.
- [ ] `Color.kt`, `Theme.kt` y `Type.kt` del ZIP integrados con package `com.example.sanmarcosstore.ui.theme`.
- [ ] `font_certs.xml` copiado a `res/values-v23/`.
- [ ] `import com.example.sanmarcosstore.R` agregado en `Type.kt`.
- [ ] Función del tema mantenida como `AppTheme` (nombre que viene del ZIP).
- [ ] Import correcto en `MainActivity`: `import com.example.sanmarcosstore.ui.theme.AppTheme`.
- [ ] Test "Hola SanMarcosStore" funciona con colores correctos.
- [ ] Packages `model`, `ui.components`, `ui.screens`, `ui.navigation` creados.
- [ ] Archivo `Producto.kt` con modelo + datos de prueba.
- [ ] Archivo `ProductoItem.kt` reutilizable.
- [ ] Archivo `TiendaScreen.kt` con buscador, FAB, formulario y lista.
- [ ] Archivo `PerfilScreen.kt` con avatar, estadísticas, opciones y switches.
- [ ] Archivo `AppNavigation.kt` con `NavHost` y `NavigationBar`.
- [ ] `MainActivity.kt` reducido a ~15 líneas.
- [ ] App compila sin errores ni warnings rojos.
- [ ] Ambas pantallas accesibles desde la barra inferior.
- [ ] Estado de cada pantalla se conserva al cambiar de tab.

---

## 🎓 Ejercicios sugeridos (para entrega)

### 🟢 Nivel 1 — Personalización

1. Cambia el **color seed** en [Material Theme Builder](https://m3.material.io/theme-builder) por uno distinto (verde, azul, naranja). Reemplaza solo `Color.kt`. Documenta cómo cambia la app.
2. Cambia los datos de tu perfil en `PerfilScreen` por **tus datos reales** (nombre, correo, año).

### 🟡 Nivel 2 — Funcionalidad

3. Agrega un **tercer tab** "Carrito" con icono 🛒. Debe mostrar al menos una `Card` que diga "Tu carrito está vacío".
4. Haz que el icono de favorito (corazón ❤️) en `ProductoItem` sea **clickable** y cambie el estado del producto. (Pista: usa `mutableStateListOf` o un `ViewModel`.)
5. Agrega filtros por **categoría** usando `FilterChip` arriba de la lista en `TiendaScreen`.

### 🔴 Nivel 3 — Avanzado

6. Al hacer click en un `ProductoItem`, navega a una **pantalla de detalle** que reciba el ID del producto como parámetro de ruta.
7. Implementa **modo oscuro real**: que el Switch del perfil cambie el tema de toda la app (pista: usa `CompositionLocalProvider` o un `ViewModel` compartido).
8. Persiste el estado de los favoritos con **DataStore** para que sobrevivan al cierre de la app.

---

## 📚 Recursos

| Recurso | URL |
|---|---|
| Material Theme Builder | https://m3.material.io/theme-builder |
| Componentes Material 3 | https://developer.android.com/jetpack/compose/components |
| Navigation Compose | https://developer.android.com/jetpack/compose/navigation |
| Roles de color M3 | https://m3.material.io/styles/color/roles |
| Compose Samples (GitHub) | https://github.com/android/compose-samples |
| Material Icons | https://fonts.google.com/icons |

---

## 📝 Entregable

Sube a tu repositorio GitHub:

1. **Carpeta completa del proyecto** (sin `build/`, `.gradle/`, `.idea/`).
2. **Capturas de pantalla** de:
   - Pantalla Tienda (con productos visibles).
   - Pantalla Tienda con formulario abierto (mostrando los 5 botones).
   - Pantalla Mi Perfil (completa, hacer scroll).
3. **README.md** con: nombre, descripción breve, captura principal, ejercicios completados (cuáles del Nivel 1/2/3).

> 🎉 **¡Listo!** Has construido una app Android moderna con arquitectura escalable, tema personalizado y navegación entre múltiples pantallas. Este es el punto de partida real de cualquier app profesional en 2026.
