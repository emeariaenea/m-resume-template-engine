# Motor de plantillas de CV — diseño

**Fecha:** 2026-08-26
**Estado:** aprobado, pendiente de implementación

---

## 1. Qué es

Un motor local que toma los datos de un CV escritos en Markdown y produce un PDF
listo para enviar a un reclutador. Las plantillas son archivos HTML+CSS con
marcadores Pebble: se editan solas, se cambian sin tocar el motor.

Dos entradas: un `.md` con formato propio, o un PDF exportado del perfil de
LinkedIn (que se convierte en un `.md` borrador para revisión manual).

### Objetivos

1. Producir PDF de una página, aptos para ATS, en A4 y Carta.
2. Plantillas intercambiables, ligeras y editables sin recompilar.
3. Ingesta desde `.md` propio y desde PDF de LinkedIn.
4. Consolidar Kotlin como lenguaje.
5. Poder crecer a una aplicación web Next sin reescribir el motor.

### No objetivos

- Editor visual de plantillas.
- Alojamiento, cuentas de usuario o multiusuario.
- Envío o seguimiento de candidaturas.
- Analizar ofertas de empleo o sugerir contenido.

---

## 2. Decisiones y por qué

| Decisión | Alternativas descartadas | Motivo |
|---|---|---|
| Kotlin/JVM, sin Spring | TypeScript/Node | Consolidar Kotlin era un objetivo declarado. Node se descartó pese a tener mejor ecosistema de Markdown y PDF. |
| Tres módulos Gradle | Módulo único | La frontera del motor deja de ser disciplina y pasa a ser una regla que Gradle impone: un `import` de Ktor dentro de `engine` no compila. |
| Pebble | Thymeleaf, Mustache | Curva mínima (`{{ }}`, `{% if %}`, `{% for %}`, `{% include %}`) y sintaxis de la familia Jinja/Twig, reaprovechable en Nunjucks o Liquid. Thymeleaf se descartó al confirmar que no había conocimiento previo: su ventaja (la plantilla se abre sola en el navegador) queda cubierta por `resume serve`. |
| Chrome headless para el PDF | Typst, LaTeX | Las plantillas siguen siendo HTML+CSS. Validado en Windows con Edge Chromium durante el diseño. |
| Ingesta asistida desde LinkedIn | Automática | LinkedIn cambia de formato sin avisar. Un parser automático se rompe en silencio y produce un CV mal formado que no se detecta hasta enviarlo. |
| Ktor embebido desde el MVP | Añadirlo al llegar a Next | Sirve la vista previa hoy y es el mismo servidor HTTP que Next consumirá después. Evita el rediseño. |

---

## 3. Arquitectura

`engine` no conoce ni el CLI ni HTTP. `cli` y `server` son dos adaptadores
sobre la misma fachada. Next será un tercer consumidor del `server` que ya
existirá, no un motivo para tocar el motor.

```
        cli (Clikt)        server (Ktor)        [futuro: Next]
             \                  |                     |
              \                 |                     |
               +----> ResumeEngine (fachada) <--------+
                            |
                          engine
```

### Estructura del proyecto

```
m-resume-template-engine/
├─ settings.gradle.kts
├─ build.gradle.kts
├─ gradle/libs.versions.toml
│
├─ engine/                        no depende de Clikt ni de Ktor
├─ cli/                           depende de engine
├─ server/                        depende de engine
│
├─ templates/
│   └─ trazo/
│       ├─ resume.peb
│       └─ template.yaml
│
├─ resumes/
│   └─ mariana.md
│
├─ docs/superpowers/specs/
└─ out/                           en .gitignore
```

### Paquetes de `engine`

Paquete base: `dev.resumekit`.

```
engine/src/main/kotlin/dev/resumekit/engine/
├─ ResumeEngine.kt        fachada; lo único que cli y server tocan
├─ model/                 Resume, Basics, Section y sus cuatro formas
├─ input/                 ResumeSource, MarkdownSource, LinkedInPdfSource
├─ template/              TemplateCatalog, PebbleRenderer
└─ render/                PdfRenderer (Playwright), PaperSize, PageAudit
```

`templates/` vive en el sistema de archivos, **no** dentro del JAR. Si
estuviera en `resources`, editar una plantilla exigiría recompilar, lo que
rompe el hot-reload y la promesa de plantillas editables.

---

## 4. Modelo de datos

Un CV es una cabecera más una lista **ordenada** de secciones. Las secciones no
son campos fijos: son cuatro formas de presentación.

```kotlin
data class Resume(val basics: Basics, val sections: List<Section>)

data class Basics(
  val name: String,
  val headline: String,
  val email: String?,
  val phone: String?,
  val location: String?,
  val links: List<Link>,
)

sealed interface Section {
  val title: String
  val kind: String        // "prose" | "timeline" | "keyvalues" | "inline"
}                         // Pebble no puede hacer type-check: necesita el discriminador

data class Prose    (override val title: String, val body: String)            : Section
data class Timeline (override val title: String, val entries: List<Entry>)    : Section
data class KeyValues(override val title: String, val rows: List<Row>)         : Section
data class Inline   (override val title: String, val items: List<String>)     : Section

data class Entry(
  val role: String,
  val org: String?,
  val location: String?,
  val start: YearMonth,
  val end: YearMonth?,          // null = Presente
  val bullets: List<String>,
)

data class Row(val label: String, val items: List<String>)
data class Link(val label: String, val url: String)
```

**Consecuencia:** añadir una sección nueva —Proyectos, Idiomas,
Publicaciones, Voluntariado— es escribir un encabezado más en el `.md`. La
plantilla ya sabe dibujar una `Timeline`; no se toca ni una línea de CSS.

Las fechas se guardan como `YearMonth`, nunca como `"feb. 2025"`. El formato
depende del idioma y se resuelve al renderizar, así que el mismo `.md` puede
producir un CV en español y otro en inglés.

Los nombres de campo siguen de cerca el esquema **JSON Resume** para dejar
abierta una futura importación o exportación. No se adopta la librería.

---

## 5. Formato de entrada `.md`

Front-matter YAML para lo que ya es clave-valor; cuerpo Markdown para la prosa.

```markdown
---
name: Mariana Pacheco
headline: Ingeniero en Informática — Desarrollo Backend
email: emeariaenea@gmail.com
phone: "+58 424 221 6100"
location: Venezuela
links:
  - { label: LinkedIn, url: "https://linkedin.com/in/emeariaenea" }
---

## Perfil
Ingeniero en Informática con experiencia en...

## Experiencia

### Ingeniero de Software — Xetux Soluciones
*2025-02 – presente*

- Diseño y desarrollo de arquitecturas ágiles de microservicios...
- Implementación de una arquitectura de microservicios que mejoró...

## Competencias técnicas

- **Backend:** Java, Spring Boot, Kotlin, microservicios, API REST
- **Infraestructura:** Docker, DevOps, RPA

## Educación

### Grado en Ingeniería, Computer Science — Universidad «Alejandro de Humboldt»
*2019-09 – 2023-04*

## Certificaciones

- DevOps: Despliegue de microservicios con Docker
- Test Automation Foundations
```

### Cómo se infiere la forma de cada sección

Se decide por la estructura del contenido bajo el encabezado `##`, en este
orden:

1. Contiene encabezados `###` → `Timeline`.
2. Es una lista cuyos elementos empiezan con `**etiqueta:**` → `KeyValues`.
3. Es una lista simple → `Inline`.
4. Cualquier otra cosa → `Prose`.

Una sección puede forzar su forma con `## Título {.timeline}` cuando la
inferencia no acierte.

### Errores de análisis

El parser falla ruidosamente, con archivo y línea. Nunca adivina en silencio:

```
mariana.md:24 — falta el rango de fechas bajo "### Ingeniero de Software"
                se esperaba una línea como: *2025-02 – presente*
```

---

## 6. Perfiles de salida y desbordamiento

El `.md` es el archivo maestro y **nunca se poda**: guarda todo el historial,
crezca lo que crezca. Lo que decide qué se publica es un perfil de salida.
Esto es adaptar el CV a cada vacante, que es lo que espera el reclutador.

```yaml
# en el front-matter del .md
profiles:
  backend-1p:
    maxPages: 1
    sections: [Perfil, Experiencia, Competencias técnicas, Educación, Certificaciones]
    Experiencia:     { limit: 4, bulletsPerEntry: 4 }
    Certificaciones: { limit: 5 }
```

```
resume build resumes/mariana.md --profile backend-1p
```

`sections` fija también el orden de salida. Una sección presente en el `.md` y
ausente de `sections` simplemente no se publica.

### Auditoría de páginas

Tras renderizar, el motor cuenta las páginas del PDF. Si excede `maxPages`,
**la build falla** con un diagnóstico accionable:

```
✗ backend-1p: 2 páginas (máximo 1)
  Cruzan el corte: Educación, Certificaciones (~180 caracteres sobrantes)
  Opciones: Experiencia.limit 4 → 3
            Certificaciones.limit 5 → 3
            maxPages 1 → 2
```

Es la automatización del bucle que se ejecutó a mano durante el diseño de las
plantillas.

### Autoajuste opcional

`--autofit` permite al motor apretar la tipografía y reintentar, pero **la
plantilla declara el suelo**:

```yaml
autofit:
  "--t-base": [8.6pt, 9.1pt]
  "--lh":     [1.25, 1.34]
```

Fuera de esos límites no baja: si al mínimo sigue sin caber, falla. Siempre
informa de lo aplicado (`autoajuste: --lh 1.34 → 1.28`). No se encoge la letra
en silencio.

### Dos páginas

`maxPages: 2` es un objetivo legítimo. La plantilla debe garantizar entonces:

- los bloques `.entry` no se parten por dentro (`break-inside: avoid`);
- un encabezado de sección nunca queda huérfano al pie (`break-after: avoid`);
- cabecera mínima opcional en la página 2 con el nombre.

---

## 7. Contrato de plantillas

```
templates/trazo/
├─ template.yaml
└─ resume.peb        HTML + CSS en un solo archivo
```

```yaml
# template.yaml
name: Trazo
description: Línea de tiempo con nodos en las secciones cronológicas
papers: [a4, carta]
autofit:
  "--t-base": [8.6pt, 9.1pt]
  "--lh":     [1.25, 1.34]
```

La plantilla recibe tres variables y nada más:

| Variable | Contenido |
|---|---|
| `cv` | el `Resume` ya filtrado por el perfil |
| `paper` | `a4` o `carta` |
| `fmt` | formateador de fechas según idioma |

```html
<h1 class="name">{{ cv.basics.name }}</h1>

{% for section in cv.sections %}
  <section class="section">
    <h2>{{ section.title }}</h2>
    {% if section.kind == 'timeline' %}
      {% for e in section.entries %}
        <div class="entry">
          <div class="entry-head">
            <h3 class="entry-title">{{ e.role }}<span class="org"> — {{ e.org }}</span></h3>
            <span class="entry-when">{{ fmt.range(e.start, e.end) }}</span>
          </div>
          <ul>{% for b in e.bullets %}<li>{{ b }}</li>{% endfor %}</ul>
        </div>
      {% endfor %}
    {% endif %}
  </section>
{% endfor %}
```

### Reglas de CSS que la plantilla debe cumplir

Aprendidas al construir las ocho plantillas de esta sesión:

1. **`@media screen and (max-width: …)`**, nunca `@media (max-width: …)`. Al
   imprimir, el viewport paginado mide 210 mm y dispara las reglas de pantalla
   estrecha. Este fallo apiló las fechas bajo el cargo en las ocho plantillas.
2. Tokens `--l-*` (claro) y `--d-*` (oscuro) por variante; un único bloque de
   mapeo. El acento claro y el oscuro son valores distintos: lo que se ve en
   pantalla oscura no es lo que se imprime.
3. `@page { size: A4 | Letter; margin: 0 }` y el margen real como `padding` de
   la hoja, para que el tamaño de papel sea un único punto de cambio.
4. `print-color-adjust: exact` en todo elemento cuyo fondo deba imprimirse.
5. Contraste mínimo 4.5:1 del acento sobre papel blanco.

### Tamaño de papel

`PaperSize` es un enum del modelo, no un detalle de la plantilla:

| | Milímetros | Puntos |
|---|---|---|
| A4 | 210 × 297 | 595 × 842 |
| Carta | 215.9 × 279.4 | 612 × 792 |

Cambiar de tamaño toca `@page` y dos tokens (`--sheet-w`, `--sheet-h`). Nada más.

---

## 8. CLI

```
resume build  resumes/mariana.md --template trazo --paper a4 --profile backend-1p --out out/
resume serve  resumes/mariana.md --template trazo --port 4321
resume import "LinkedIn.pdf" --out resumes/mariana.md
resume templates
```

`--paper` acepta `a4`, `carta` o `ambos`.

---

## 9. Ingesta desde LinkedIn

```
resume import "LinkedIn.pdf" --out resumes/mariana.md
```

1. PDFBox extrae el texto.
2. Una heurística lo trocea en secciones y las mapea a las cuatro formas.
3. Se escribe un `.md` **borrador** con marcas `<!-- REVISAR -->` donde la
   confianza es baja.
4. Se imprime un resumen de lo encontrado y lo no reconocido.

Nunca produce un PDF directamente. El `.md` es siempre la fuente de verdad y
pasa por revisión humana antes de generar nada.

---

## 10. Servidor Ktor

**Hoy:** `GET /` sirve el HTML renderizado; `GET /events` es un canal SSE; un
`WatchService` sobre `resumes/` y `templates/` dispara la recarga al guardar.

**Después:** `POST /render` recibe datos y devuelve el PDF. Ese es el endpoint
que consumirá Next, y no exige tocar el motor.

---

## 11. Errores con mensaje propio

| Situación | Mensaje |
|---|---|
| Front-matter inválido | archivo y línea del error de YAML |
| Sección sin fechas en una `Timeline` | archivo, línea y forma esperada |
| Plantilla inexistente | lista las disponibles con su nombre visible |
| Chromium no instalado | el comando exacto de instalación |
| Desbordamiento de páginas | el diagnóstico de la sección 6 |

---

## 12. Pruebas

La verificación manual de esta sesión se convierte en la suite.

| Nivel | Qué comprueba |
|---|---|
| `MarkdownSource` | `.md` de referencia → `Resume` esperado, incluida la inferencia de forma |
| `MarkdownSource` (errores) | cada error de la sección 11 produce archivo y línea correctos |
| `PebbleRenderer` | snapshot del HTML generado |
| `LinkedInPdfSource` | contra el PDF real guardado en recursos de prueba |
| Integración | renderiza a PDF y afirma: **1 página**; secciones en orden en el texto extraído; fechas en la misma línea que el cargo; MediaBox correcto según el papel |

El último test habría atrapado el fallo de la media query antes de que llegara
a producir un PDF equivocado.

---

## 13. Dependencias

| Uso | Biblioteca |
|---|---|
| CLI | Clikt |
| Plantillas | Pebble |
| Markdown | flexmark-java |
| YAML | kaml o snakeyaml |
| Leer PDF | Apache PDFBox |
| Generar PDF | Playwright for Java |
| Servidor | Ktor (CIO) |
| Pruebas | kotlin.test + kotest assertions |

Playwright descarga Chromium (~150 MB) en el primer uso. El arranque de la JVM
añade entre 0,4 y 1 segundo por comando. Distribuir requiere JRE o `jlink`.

---

## 14. Fuera de alcance de esta iteración

- Empaquetado y distribución (`jlink`, instalador).
- CV en varios idiomas desde un mismo `.md` (el modelo ya lo permite; falta el
  comando).
- La aplicación web Next.
- Importar o exportar JSON Resume.
