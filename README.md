# IG Unfollow

Una página web de un solo archivo que fabrica **bookmarklets** (favoritos `javascript:`) para dejar de seguir en Instagram a quien no te sigue de vuelta — sin instalar nada, sin pedir tu contraseña y sin enviar tus datos a ningún servidor.

**En vivo:** https://willbytee-sudo.github.io/ig-unfollow/ · próximamente en `igunfollow.vercel.app`

---

## Qué hace

La página **no toca Instagram por sí sola**: su trabajo es generar cadenas `javascript:...` autocontenidas que el usuario arrastra a su barra de favoritos y luego ejecuta *dentro* de su propia pestaña de instagram.com, sobre la sesión que ya tiene abierta.

Ofrece tres motores principales, cada uno con su propio botón arrastrable y su perfil de riesgo:

| Modo | Botón (id) | Motor | Perfil de riesgo |
|------|-----------|-------|------------------|
| **Híbrido** — la forma más segura (recomendado) | `#bmHibrido` | Analiza por la API interna, pero deja de seguir con **clics reales** en el modal de Seguidos. Revisas la lista con foto antes de quitar. | El más seguro (más lento) |
| **IG Auto** — todo con clics | `#bmAuto` | 100 % DOM: hace scroll de los modales, calcula la diferencia y deja de seguir simulando escritura y clics. | Intermedio |
| **IG Analizador** — el más rápido | `#bmApiAuto` | Lee tus listas por la API privada de Instagram y da de baja con un `POST` directo por cuenta. | El que más arriesga tu cuenta |

Además hay un **modo manual** encadenado por si el automático falla: **Extractor** (raspa una lista abierta al portapapeles) → **Comparar** (pega Seguidos y Seguidores y calcula quién no te sigue) → **IG Unfollow** (recibe la lista y da de baja uno por uno).

---

## Cómo se usa

Es una herramienta **de escritorio**: se basa en arrastrar un botón a la barra de favoritos, algo que no existe en el celular (y ahí Instagram tampoco deja correr el bookmarklet). La página detecta el móvil y muestra un aviso para abrirla en la computadora.

Instalar toma tres pasos, una sola vez:

1. **Muestra la barra de favoritos** de tu navegador con `Ctrl + Shift + B` (en Mac, `⌘ + Shift + B`).
2. **Arrastra el botón verde** ("La forma más segura") hasta esa barra.
3. **Abre Instagram, entra a tu perfil** y haz clic en el favorito guardado. Se abre un panel que te muestra quién no te sigue de vuelta y te deja elegir a quién quitar.

> **Importante:** el favorito guarda una *foto fija* del código en el momento de arrastrarlo. Si después cambias el ritmo (o, en modo manual, la lista de usuarios), **vuelve a arrastrar el botón** — el nuevo valor solo se hornea al arrastrar.

Antes de arrastrar, elige el **ritmo** (cada cuánto deja de seguir). Por eso la sección de ritmo va *antes* de los botones. Si no puedes arrastrar, cada motor tiene un botón "copiar el código" que pone el bookmarklet en el portapapeles.

---

## Cómo funciona

### Por qué bookmarklets (política de mismo origen)

El sitio vive en GitHub Pages y se moverá a Vercel. Ninguno de esos orígenes puede leer las cookies de sesión de Instagram ni llamar a sus endpoints. El motor por API necesita `ds_user_id` y `csrftoken` de las cookies y hace `fetch(..., {credentials:'include'})` contra `/api/v1/friendships/...`. Eso **solo funciona si el código corre en el origen `instagram.com`**.

El bookmarklet es justamente el vehículo: al pulsar el favorito, el navegador ejecuta el `javascript:` en el contexto de la pestaña activa (Instagram), heredando su sesión y su mismo origen.

### Por qué el código va incrustado, no cargado

Instagram sirve una Content-Security-Policy que **impide cargar scripts remotos**. Por eso ningún bookmarklet trae un pequeño "loader" que baje el motor desde Pages o Vercel: todo el motor va incrustado, literal, dentro del propio `javascript:` URI. Los motores solo hablan con recursos del mismo origen de Instagram (su API o el DOM) y nunca con un `https://` de terceros. Ese es el motivo de que cada bookmarklet sea un bloque enorme de JS en línea.

### Los tres motores por dentro

**Motor CLICS (IG Auto) — `getAutoFullCode()`.** 100 % DOM, sin API ni tokens. Al ejecutarse en tu perfil:
- **Fase 1–2:** abre los modales de *Seguidos* y *Seguidores* (`openModal`), espera a que aparezca el `div[role="dialog"]` (`waitDialog`) y recorre cada lista completa haciendo scroll con detección de estancamiento (`extract` → `Map<username, verificado>`).
- **Fase 3:** calcula `notBack` = seguidos que no están entre tus seguidores (comparación en minúsculas para no fallar por mayúsculas) y muestra una pantalla de acciones: deseguir todos, solo no verificados, editar la lista con checkboxes, o solo copiar.
- **Fase 4 — el deseguido real (`doUnfollow`):** por cada usuario escribe su nombre en el buscador del modal usando el **setter nativo** de `HTMLInputElement.value` (necesario porque React ignora la asignación directa) + un evento `input`, hace clic en **"Siguiendo"/"Following"** y luego en **"Dejar de seguir"/"Unfollow"**. Entre cada baja espera un intervalo **aleatorio** que imita comportamiento humano.

**Motor API + Híbrido — plantilla `#apiSrc`.** Un mismo bloque genera dos productos según la bandera `EXEC`:
- Lee las cookies (`ck()`), obtiene `ds_user_id` y `csrftoken` y arma la cabecera `H()` (`x-ig-app-id`, `x-csrftoken`, `x-requested-with`).
- **Pagina** ambas listas con `grab()` contra `/api/v1/friendships/{myId}/{following|followers}/` (hasta 600 páginas, con espera entre página y página).
- Calcula `NB` (no te siguen de vuelta) por PK y muestra una **lista revisable** con foto, insignia de verificado, buscador, botón "conservar verificados" y un contador vivo. Todo arranca marcado, con una confirmación (`confirm`) antes de ejecutar.
- Con `EXEC='api'` da de baja con `POST /api/v1/friendships/destroy/{pk}/` (rápido). Con `EXEC='click'` (Híbrido) abre el modal de Seguidos y reproduce los mismos clics que haría una persona (más seguro).
- Ambos respetan el **rate-limit de Instagram**: ante un `429` o `400` esperan 30 s y reintentan el mismo objetivo sin perderlo.

**Persistencia opcional.** El Analizador/Híbrido puede "recordar los desmarcados" en `localStorage` (clave `_ig_keep_forever`): a quien decidiste conservar no se te vuelve a proponer marcado.

---

## Arquitectura del código

Todo vive en **un único `index.html`** (~1791 líneas): sin `package.json`, sin framework, sin paso de build. El HTML, el CSS (embebido en `<style>`) y varios `<script>` conviven en el mismo archivo, que es a la vez el sitio de marketing y el "compilador" de los bookmarklets.

### Cómo se construyen los `href` "javascript:..."

Hay dos familias de motores y ambas terminan igual, con `'javascript:' + encodeURIComponent(code)`:

- **Motores DOM** (`getAutoFullCode`, `buildExtractorBM`, `buildUnfollowBM`) devuelven el código como *template string*. `buildUnfollowBM` **hornea los datos** dentro del código (`var users=${JSON.stringify(users)}`, delays literales); `buildAutoBM` inyecta el intervalo reemplazando el valor cableado del motor.
- **El motor API** se guarda como texto inerte en un bloque especial:

```html
<script type="text/plain" id="apiSrc">(function(){ ... })()</script>
```

El `type="text/plain"` es **deliberado**: el navegador no ejecuta ese bloque, solo lo conserva como texto legible. Dentro lleva tres marcadores de posición que no son JS real hasta que se sustituyen:

```js
var minD=__MIN__, maxD=__MAX__, EXEC='__EXEC__', KEEP='_ig_keep_forever';
```

`buildApiCode()` lee ese texto y rellena los placeholders (`__MIN__`, `__MAX__`, `__EXEC__`) antes de codificar la URL. `refreshApi()` mantiene vivos los dos `href` (`#bmApiAuto` con `exec='api'` y `#bmHibrido` con `exec='click'`) y se reengancha a cada cambio de ritmo; los motores DOM hacen lo análogo desde `update()`.

### El sistema de diseño

El CSS es literalmente el sistema de diseño heredado del sitio "Willito" (tokens de handoff de Claude Design). Sobre esa base se aplica un **bloque de re-tematización** que gana por orden de cascada: **fuerza el tema oscuro siempre** y remapea el acento lima original a **rosa Instagram**. Por eso los nombres de variable siguen siendo `--lime`, `--lime-ink`, etc., pero sus valores efectivos son rosados.

- **Paleta efectiva:** fondo `#0B0B0F`, tarjetas `#17171C`, texto `#F2F2F6`; acento `--lime` = `#E1306C`, texto de acento `#FF5C93`.
- **Firma visual — degradado Instagram** (`#F58529 → #E1306C → #C13584 → #833AB4`) recortado a texto en el título y la marca, y en los botones. Convención: **clics = degradado cálido, API = degradado púrpura**. La tarjeta recomendada rompe la regla a propósito con **verde** para marcarla como la opción segura.
- **Sombras como glow** (`0 0 40px ...`) en vez de sombras duras.
- **Tipografía "terminal":** `Unbounded` (títulos), `Space Grotesk` (cuerpo) y `JetBrains Mono` (kickers entre corchetes, chips, contadores, comandos). Kickers `[ … ]`, prompts con cursor parpadeante y `$`/`>` decorativos.
- **Tema oscuro fijo:** el `<head>` fija `data-theme="dark"` antes de pintar. Existe un script `themeToggle` heredado, pero **no hay elemento `#themeToggle`** en este HTML, así que queda inerte.

Los **paneles que inyectan los bookmarklets** dentro de Instagram usan sus propios estilos inline (fondos `#1a1a1a`/`#141418`, acento `#e1306c`, marca "Williams" en azul `#3b82f6`) — un sistema aparte, coherente en color pero independiente de las variables CSS de la página. Todos los assets tienen **fallback SVG vía `onerror`**, así que nada depende de un recurso externo que falle.

---

## Límites de Instagram y seguridad

Instagram no publica cifras oficiales, pero por comportamiento observado los rangos prácticos para dejar de seguir son:

- **Cuentas establecidas:** ~100–150 unfollows/día y ~20–30/hora.
- **Cuentas nuevas o "frías":** mucho menos, a veces 10–20/hora antes de recibir un aviso.
- **La ráfaga por hora es lo que más delata.** El pico de velocidad, no el total, es lo que dispara la detección.
- **Consecuencia:** *action block* temporal de 24 a 72 horas. Reincidir alarga el bloqueo.

La página lo resume en su aviso de seguridad: *"Dejar de seguir a mucha gente muy rápido puede causar un bloqueo temporal. El botón ya mete pausas aleatorias; si vas a limpiar cientos, hazlo por tandas y no todos los días."*

### Cómo el diseño mitiga esos límites

**Pausas aleatorias entre cada acción.** Es lo central: cada baja espera un intervalo aleatorio dentro del rango elegido (nunca un valor constante, que sería más fácil de detectar). El motor DOM trae un retardo base cableado que se reemplaza al construir el bookmarklet:

```js
fullCode = fullCode.replace('5000+Math.random()*10000', minD + '+Math.random()*' + (maxD - minD));
```

**Preset "seguro" por defecto.** Tres presets de ritmo, con "seguro" activo al cargar:

| Preset | min / max | Intervalo por persona |
|--------|-----------|----------------------|
| 🛡️ seguro | 8000 / 20000 | 8–20 s |
| ⚖️ normal | 4000 / 12000 | 4–12 s |
| ⚡ rápido | 2000 / 5000 | 2–5 s |

Los sliders manuales permiten afinar hasta 60 s. Un estimador muestra el tiempo previsto (en "seguro", 50 cuentas ≈ 12 min, 200 ≈ 47 min) para dimensionar cada tanda.

**Respeto al freno del propio Instagram (modo API).** Ante un `429` o `400`, el motor espera 30 s y reintenta sin perder el elemento.

> **Nota honesta:** las pausas reducen la *ráfaga*, pero la herramienta **no** aplica un límite diario automático. La protección real depende de que el usuario haga tandas y no lo use a diario. Empieza siempre en modo **seguro**.

---

## Privacidad

El modelo de privacidad es fuerte porque **no hay backend**:

- **Todo corre en tu navegador.** No hay servidor propio: *"todo corre en tu navegador · nada se envía a ningún servidor"*.
- **Nunca pide tu contraseña.** Usa la sesión que ya tienes abierta: lee las cookies existentes de tu propio Instagram y las usa solo para hablar con `instagram.com` desde la misma pestaña. Nunca las transmite a terceros.
- **Sin telemetría ni build.** Las listas de Seguidos/Seguidores se procesan en memoria y no se persisten fuera del navegador (salvo la lista opcional "recordar desmarcados", que se guarda en tu propio `localStorage`).
- **Única dependencia externa:** Google Fonts (una petición a `fonts.googleapis.com` al cargar; no envía datos del usuario).

---

## Despliegue

Es un **único `index.html` autocontenido** que se sirve como archivo estático. Referencia un asset local `williams.jpg` con fallback SVG inline, así que si la imagen falta la página no se rompe — pero conviene subirlo junto al HTML para que se vea el avatar real.

### GitHub Pages (actual)

- Publicado en **https://willbytee-sudo.github.io/ig-unfollow/**.
- Basta con que `index.html` (y `williams.jpg`) estén en la raíz de la rama publicada (o `/docs`). No requiere Actions ni compilación.
- Checklist: subir ambos archivos, activar Pages sobre la rama correcta y verificar que la fuente de Google carga.

### Vercel (planeado — `igunfollow.vercel.app`)

- Encaja igual: proyecto estático, framework preset **"Other"**, sin build command, directorio de salida = raíz.
- Ventaja: dominio más corto y despliegues automáticos por push.
- Sin variables de entorno ni secretos que configurar (no hay backend).
- Recomendación: mantener **una sola fuente de verdad** del HTML para que ambos hosts sirvan lo mismo y no diverjan.

---

## Descargo de responsabilidad

- Esta herramienta **no está afiliada, asociada ni respaldada por Instagram ni por Meta Platforms, Inc.** "Instagram" es marca de Meta. Es un proyecto independiente hecho por Williams.
- **Automatizar acciones** como dejar de seguir de forma masiva **puede ir en contra de los Términos de Uso de Instagram** y derivar en bloqueos temporales o, en casos extremos, en restricciones de la cuenta.
- **Úsala bajo tu propia responsabilidad y con moderación.** No nos hacemos responsables de bloqueos, pérdidas ni sanciones sobre tu cuenta. Empieza siempre en modo **seguro**, hazlo **por tandas** y **no todos los días**.
- Todo el proceso ocurre **en tu navegador**: no pedimos tu contraseña ni almacenamos tus datos.

---

## Créditos

Hecho por **Williams** · [@willbytee-sudo](https://github.com/willbytee-sudo)
