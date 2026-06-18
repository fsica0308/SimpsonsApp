# Análisis de errores detectados - SimpsonsApp

Este fork contiene el análisis de errores encontrados en el proyecto original `SimpsonsApp`.

El objetivo del análisis es identificar errores de código, configuración, arquitectura y buenas prácticas relacionados con Android, Kotlin, Jetpack Compose, Retrofit, Paging y Clean Architecture.

Cada error incluye:

* Archivo afectado.
* Línea o ubicación aproximada.
* Código con error, cuando corresponde.
* Explicación del problema.
* Posible solución.

---

# Errores principales encontrados

---

## Error 1 - Bloque `init` inválido en `Episode.kt`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/model/Episode.kt`
**Líneas:** 13 a 15

### Código con error

```kotlin
init {
    return Episode; //NO BORRAR
}
```

### Problema

Después de cerrar la `data class Episode`, queda declarado un bloque `init` suelto.

Esto no es válido en Kotlin, ya que un bloque `init` solo puede existir dentro del cuerpo de una clase. Además, `return Episode` tampoco es válido porque `Episode` es un tipo/modelo, no un valor que pueda retornarse de esa forma.

Este error impide que el archivo compile correctamente.

### Solución

Eliminar completamente el bloque `init`.

El archivo debería quedar únicamente con la definición de la `data class`:

```kotlin
data class Episode(
    val id: Int,
    val airdate: String,
    val episodeNumber: Int,
    val imagePath: String,
    val name: String,
    val season: Int,
    val synopsis: String
)
```

---

## Error 2 - Método mal nombrado en `EpisodeRepository`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/repository/EpisodeRepository.kt`
**Línea:** 8

### Código con error

```kotlin
fun get_episodes(): Flow<PagingData<Episode>>
```

### Problema

El método de la interfaz está declarado como `get_episodes()`, pero la clase que implementa el repositorio utiliza el método `getEpisodes()`.

Esto rompe el contrato entre la interfaz y su implementación, porque los nombres de los métodos no coinciden. Además, en Kotlin se recomienda usar nomenclatura `camelCase` para funciones.

### Solución

Renombrar el método en la interfaz:

```kotlin
fun getEpisodes(): Flow<PagingData<Episode>>
```

---

## Error 3 - `GetEpisodesUseCase` llama a un método inexistente

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/usecase/GetEpisodesUseCase.kt`
**Línea:** 13

### Código con error

```kotlin
return repository.get_episodes()
```

### Problema

El caso de uso llama al método `get_episodes()`, pero ese nombre no coincide con el método que debería exponer el repositorio.

Este error está relacionado con el error anterior. Al corregir el repositorio para que use `getEpisodes()`, también hay que actualizar esta llamada.

### Solución

Cambiar la llamada por:

```kotlin
return repository.getEpisodes()
```

---

## Error 4 - Falta importar `SimpsonsApi` en `EpisodeRepositoryImpl`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/repository/EpisodeRepositoryImpl.kt`
**Línea:** 18

### Código con error

```kotlin
private val simpsonsApi: SimpsonsApi,
```

### Problema

La clase `EpisodeRepositoryImpl` utiliza el tipo `SimpsonsApi`, pero el archivo no tiene el import correspondiente.

Esto provoca un error de referencia no resuelta al compilar.

### Solución

Agregar el import al inicio del archivo:

```kotlin
import com.example.simpsonsapp.data.remote.SimpsonsApi
```

---

## Error 5 - Retrofit se construye sin `baseUrl`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/di/DataModule.kt`
**Líneas:** 34 a 38

### Código con error

```kotlin
return Retrofit.Builder()
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)
```

### Problema

Retrofit necesita obligatoriamente una `baseUrl` para construir el cliente HTTP.

Sin esa configuración, la aplicación puede fallar en tiempo de ejecución al intentar crear o utilizar el servicio de red.

### Solución

Agregar la URL base antes del `.build()`:

```kotlin
return Retrofit.Builder()
    .baseUrl("https://thesimpsonsapi.com/")
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)
```

---

## Error 6 - Endpoint absoluto dentro de `@GET`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`
**Línea:** 106

### Código con error

```kotlin
@GET("https://thesimpsonsapi.com/api/episodes")
```

### Problema

En Retrofit, lo correcto es definir el dominio principal en el `baseUrl` del `Retrofit.Builder` y usar rutas relativas en los endpoints.

Tener la URL completa dentro de `@GET` duplica responsabilidades y hace que la configuración de red quede repartida en más de un lugar.

### Solución

Dejar el dominio en `DataModule.kt`:

```kotlin
.baseUrl("https://thesimpsonsapi.com/")
```

Y cambiar el endpoint por una ruta relativa:

```kotlin
@GET("api/episodes")
suspend fun getEpisodes(
    @Query("page") page: Int
): EpisodesResponse
```

---

## Error 7 - Condición incorrecta en paginación `PREPEND`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`
**Línea:** 36

### Código con error

```kotlin
?: return MediatorResult.Success(endOfPaginationReached = remoteKeys != null)
```

### Problema

Esta línea se ejecuta cuando `remoteKeys` es `null`, porque forma parte del operador Elvis `?:`.

Por lo tanto, evaluar `remoteKeys != null` en ese punto siempre va a devolver `false`.

En el caso de `PREPEND`, si no hay `remoteKeys`, significa que no hay información para seguir cargando páginas anteriores. Por eso, debería indicarse que ya se llegó al final de la paginación hacia atrás.

### Solución

Cambiar la condición por `true`:

```kotlin
?: return MediatorResult.Success(endOfPaginationReached = true)
```

---

## Error 8 - Condición incorrecta en paginación `APPEND`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`
**Línea:** 42

### Código con error

```kotlin
?: return MediatorResult.Success(endOfPaginationReached = remoteKeys != null)
```

### Problema

Este error es similar al de `PREPEND`, pero ocurre al intentar cargar páginas siguientes con `APPEND`.

La línea se ejecuta cuando `remoteKeys` es `null`, por lo tanto `remoteKeys != null` siempre va a ser `false`.

Esto puede provocar que Paging crea que todavía hay más páginas para cargar cuando en realidad no hay claves remotas disponibles para continuar.

### Solución

Cambiar la condición por:

```kotlin
?: return MediatorResult.Success(endOfPaginationReached = true)
```

---

## Error 9 - Side effect ejecutado directamente dentro de `MainScreen`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt`
**Líneas:** 50 a 53

### Código con error

```kotlin
if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
    viewModel.refreshSeasons()
}
```

### Problema

En Jetpack Compose no se deben ejecutar efectos secundarios directamente dentro del cuerpo de un composable.

El cuerpo de un composable puede recomponerse muchas veces. Si la condición se cumple en más de una recomposición, `viewModel.refreshSeasons()` puede ejecutarse repetidamente sin una acción directa del usuario.

Este tipo de lógica debería manejarse con APIs de efectos secundarios de Compose, como `LaunchedEffect`.

### Solución

Envolver la llamada dentro de un `LaunchedEffect`:

```kotlin
LaunchedEffect(episodes.loadState.refresh, seasons.isEmpty()) {
    if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
        viewModel.refreshSeasons()
    }
}
```

También se debe agregar el import:

```kotlin
import androidx.compose.runtime.LaunchedEffect
```

---

## Error 10 - `MainScreenTest` está desactualizado y no coincide con la pantalla real

**Archivo:** `app/src/androidTest/java/com/example/simpsonsapp/ui/main/MainScreenTest.kt`
**Líneas:** 18 y 23

### Código con error

```kotlin
composeTestRule.setContent { MainScreen(FAKE_DATA) }
```

```kotlin
FAKE_DATA.forEach { composeTestRule.onNodeWithText("Hello $it!").assertExists() }
```

### Problema

El test no coincide con la implementación actual de `MainScreen`.

La función real no recibe una lista de strings como `FAKE_DATA`. Su firma espera una función de navegación:

```kotlin
fun MainScreen(
    onNavigateToDetail: (Int) -> Unit,
    viewModel: MainViewModel = hiltViewModel()
)
```

Además, el test busca textos como `"Hello Sample1!"`, `"Hello Sample2!"`, etc., pero esos textos no existen en la pantalla real de la aplicación.

Esto indica que el test quedó de un template o de una versión anterior y no fue actualizado.

### Solución

Actualizar el test para llamar correctamente a `MainScreen`:

```kotlin
composeTestRule.setContent {
    MainScreen(onNavigateToDetail = {})
}
```

Y reemplazar las validaciones por textos reales de la pantalla, por ejemplo:

```kotlin
composeTestRule.onNodeWithText("The Simpsons Episodes").assertExists()
```

Idealmente, se debería crear una versión testeable/stateless del composable para poder pasarle datos mockeados sin depender directamente de `hiltViewModel()`.

---

# Observaciones adicionales

Además de los 10 errores principales, se detectaron otros puntos que también podrían corregirse o mejorarse.

---

## Observación adicional 1 - Ruta local inválida para Java en Gradle

**Archivo:** `gradle.properties`
**Ubicación:** propiedad `org.gradle.java.home`

### Código relacionado

```properties
org.gradle.java.home=/opt/homebrew/Cellar/openjdk@17/17.0.15/libexec/openjdk.jdk/Contents/Home
```

### Problema

El proyecto define una ruta absoluta local para el JDK de Gradle.

Esa ruta corresponde a una instalación específica de macOS con Homebrew, por lo tanto no existe en otras computadoras. Esto puede provocar que Android Studio o Gradle fallen al intentar sincronizar el proyecto.

El error mostrado es:

```text
Value '/opt/homebrew/Cellar/openjdk@17/17.0.15/libexec/openjdk.jdk/Contents/Home' given for org.gradle.java.home Gradle property is invalid (Java home supplied is invalid)
```

### Solución sugerida

Eliminar la línea del archivo `gradle.properties` o comentarla:

```properties
# org.gradle.java.home=/opt/homebrew/Cellar/openjdk@17/17.0.15/libexec/openjdk.jdk/Contents/Home
```

Luego, configurar el JDK desde Android Studio:

```text
File > Settings > Build, Execution, Deployment > Build Tools > Gradle > Gradle JDK
```

Seleccionar `Embedded JDK` o un JDK 17 válido instalado localmente.

---

## Observación adicional 2 - `SimpsonsApi` debería estar en un archivo propio

**Archivo actual:** `app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`
**Líneas:** 105 a 110

### Código relacionado

```kotlin
interface SimpsonsApi {
    @GET("https://thesimpsonsapi.com/api/episodes")
    suspend fun getEpisodes(
        @Query("page") page: Int
    ): EpisodesResponse
}
```

### Problema

La interfaz `SimpsonsApi` está declarada dentro del mismo archivo que `EpisodeRemoteMediator`.

Esto no necesariamente impide compilar por sí solo, pero es una mala decisión de arquitectura y organización del proyecto, porque mezcla responsabilidades distintas:

* `EpisodeRemoteMediator` debería encargarse de coordinar la carga de datos entre red, base local y Paging.
* `SimpsonsApi` debería encargarse únicamente de declarar los endpoints HTTP disponibles.

Desde el punto de vista de Clean Architecture y separación de responsabilidades, cada componente debe tener una responsabilidad clara y estar ubicado en el archivo/capa correspondiente.

### Solución sugerida

Mover la interfaz `SimpsonsApi` a un archivo propio:

```text
app/src/main/java/com/example/simpsonsapp/data/remote/SimpsonsApi.kt
```

Contenido sugerido:

```kotlin
package com.example.simpsonsapp.data.remote

import com.example.simpsonsapp.data.remote.model.EpisodesResponse
import retrofit2.http.GET
import retrofit2.http.Query

interface SimpsonsApi {
    @GET("api/episodes")
    suspend fun getEpisodes(
        @Query("page") page: Int
    ): EpisodesResponse
}
```

De esta forma, `EpisodeRemoteMediator.kt` queda enfocado solamente en la lógica de mediación entre la API, la base de datos local y Paging.

---

# Conclusión

Los errores principales encontrados afectan distintas partes del proyecto:

* Sintaxis y compilación Kotlin.
* Contratos entre capas de dominio y datos.
* Configuración de Retrofit.
* Lógica de paginación con Paging 3.
* Buenas prácticas de Jetpack Compose.
* Tests desactualizados.

También se detectaron observaciones adicionales relacionadas con:

* Configuración del entorno Gradle/JDK.
* Organización de archivos y separación de responsabilidades.

En conjunto, estos problemas afectan la compilación, la ejecución, la mantenibilidad y la calidad arquitectónica de la aplicación.
