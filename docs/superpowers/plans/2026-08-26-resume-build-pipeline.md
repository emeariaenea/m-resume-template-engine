# Resume Build Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `resume build resumes/mariana.md --template trazo --paper a4` produces a one-page, ATS-parseable PDF, and fails the build with an actionable diagnostic when the content does not fit.

**Architecture:** Three Gradle modules. `engine` holds the domain and knows nothing about CLI or HTTP. `cli` is a Clikt adapter over the `ResumeEngine` facade. `server` is created empty here and filled by a later plan. Markdown in, Pebble template, Chrome headless out.

**Tech Stack:** Kotlin 2.1.0 on JDK 21, Gradle Kotlin DSL, Clikt, Pebble, flexmark-java, kaml, Playwright for Java, kotlin.test + kotest assertions.

**Spec:** `docs/superpowers/specs/2026-08-26-resume-template-engine-design.md`

## Global Constraints

- Base package: `dev.resumekit`. Modules: `dev.resumekit.engine`, `dev.resumekit.cli`, `dev.resumekit.server`.
- JDK 21. Kotlin JVM target 21.
- `engine/build.gradle.kts` MUST NOT declare Clikt or Ktor. This is the architectural boundary; a violation must fail compilation.
- `templates/` is read from the filesystem, never from `resources`. Editing a template must never require recompiling.
- Dates in the model are `java.time.YearMonth`. Never store formatted strings like `"feb. 2025"`.
- `end == null` means *Presente*.
- Every `Section` exposes `kind: String` — one of `"prose"`, `"timeline"`, `"keyvalues"`, `"inline"`. Pebble cannot type-check a sealed interface.
- Paper sizes: A4 = 210 × 297 mm (595 × 842 pt). Carta = 215.9 × 279.4 mm (612 × 792 pt).
- Parser errors always carry file and line number.
- Template CSS rule that MUST survive the port: `@media screen and (max-width: …)`, never a bare `@media (max-width: …)`. The print viewport is 210 mm and triggers bare narrow-screen rules.

---

### Task 1: Gradle skeleton with the module boundary

**Files:**
- Create: `settings.gradle.kts`
- Create: `build.gradle.kts`
- Create: `gradle/libs.versions.toml`
- Create: `engine/build.gradle.kts`, `cli/build.gradle.kts`, `server/build.gradle.kts`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/BoundarySmokeTest.kt`

**Interfaces:**
- Consumes: nothing.
- Produces: three Gradle modules named `engine`, `cli`, `server`. `cli` and `server` depend on `engine`; `engine` depends on neither.

- [ ] **Step 1: Write the version catalog**

`gradle/libs.versions.toml`:

```toml
[versions]
kotlin = "2.1.0"
clikt = "5.0.2"
pebble = "3.2.2"
flexmark = "0.64.8"
kaml = "0.67.0"
pdfbox = "3.0.3"
playwright = "1.49.0"
ktor = "3.0.3"
kotest = "5.9.1"

[libraries]
clikt      = { module = "com.github.ajalt.clikt:clikt",              version.ref = "clikt" }
pebble     = { module = "io.pebbletemplates:pebble",                 version.ref = "pebble" }
flexmark   = { module = "com.vladsch.flexmark:flexmark-all",         version.ref = "flexmark" }
kaml       = { module = "com.charleskorn.kaml:kaml",                 version.ref = "kaml" }
pdfbox     = { module = "org.apache.pdfbox:pdfbox",                  version.ref = "pdfbox" }
playwright = { module = "com.microsoft.playwright:playwright",       version.ref = "playwright" }
ktor-core  = { module = "io.ktor:ktor-server-core",                  version.ref = "ktor" }
ktor-cio   = { module = "io.ktor:ktor-server-cio",                   version.ref = "ktor" }
kotest     = { module = "io.kotest:kotest-assertions-core",          version.ref = "kotest" }
```

- [ ] **Step 2: Write the settings and root build files**

`settings.gradle.kts`:

```kotlin
rootProject.name = "m-resume-template-engine"
include("engine", "cli", "server")
```

`build.gradle.kts`:

```kotlin
plugins {
    kotlin("jvm") version "2.1.0" apply false
}

subprojects {
    apply(plugin = "org.jetbrains.kotlin.jvm")

    repositories { mavenCentral() }

    extensions.configure<org.jetbrains.kotlin.gradle.dsl.KotlinJvmProjectExtension> {
        jvmToolchain(21)
    }

    dependencies {
        "testImplementation"(kotlin("test"))
    }

    tasks.withType<Test> { useJUnitPlatform() }
}
```

- [ ] **Step 3: Write the three module build files**

`engine/build.gradle.kts` — note what is absent; that absence is the architecture:

```kotlin
dependencies {
    implementation(libs.pebble)
    implementation(libs.flexmark)
    implementation(libs.kaml)
    implementation(libs.pdfbox)
    implementation(libs.playwright)
    testImplementation(libs.kotest)
}
```

`cli/build.gradle.kts`:

```kotlin
dependencies {
    implementation(project(":engine"))
    implementation(libs.clikt)
    testImplementation(libs.kotest)
}
```

`server/build.gradle.kts`:

```kotlin
dependencies {
    implementation(project(":engine"))
    implementation(libs.ktor.core)
    implementation(libs.ktor.cio)
    testImplementation(libs.kotest)
}
```

- [ ] **Step 4: Write a smoke test that proves the toolchain runs**

`engine/src/test/kotlin/dev/resumekit/engine/BoundarySmokeTest.kt`:

```kotlin
package dev.resumekit.engine

import kotlin.test.Test
import kotlin.test.assertEquals

class BoundarySmokeTest {
    @Test
    fun `engine module compiles and tests run`() {
        assertEquals(21, Runtime.version().feature())
    }
}
```

- [ ] **Step 5: Run the build**

Run: `./gradlew test`
Expected: PASS, three modules configured.

- [ ] **Step 6: Verify the boundary is real**

Temporarily add `import io.ktor.server.engine.embeddedServer` at the top of `BoundarySmokeTest.kt`.
Run: `./gradlew :engine:compileTestKotlin`
Expected: FAIL with "Unresolved reference: ktor". **Remove the import again.** This is the whole point of three modules — confirm it once, by hand, now.

- [ ] **Step 7: Commit**

```bash
git add settings.gradle.kts build.gradle.kts gradle/libs.versions.toml engine cli server
git commit -m "build: Gradle skeleton with engine/cli/server module boundary"
```

---

### Task 2: Domain model

**Files:**
- Create: `engine/src/main/kotlin/dev/resumekit/engine/model/Resume.kt`
- Create: `engine/src/main/kotlin/dev/resumekit/engine/render/PaperSize.kt`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/model/ResumeTest.kt`

**Interfaces:**
- Consumes: Task 1's `engine` module.
- Produces: `Resume(basics, sections)`; `Basics(name, headline, email, phone, location, links)`; `Section` sealed interface with `title: String` and `kind: String`; `Prose(title, body)`, `Timeline(title, entries)`, `KeyValues(title, rows)`, `Inline(title, items)`; `Entry(role, org, location, start, end, bullets)`; `Row(label, items)`; `Link(label, url)`; `enum class PaperSize { A4, CARTA }` with `widthMm`, `heightMm`, `cssPageSize`.

- [ ] **Step 1: Write the failing test**

`engine/src/test/kotlin/dev/resumekit/engine/model/ResumeTest.kt`:

```kotlin
package dev.resumekit.engine.model

import dev.resumekit.engine.render.PaperSize
import java.time.YearMonth
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertNull

class ResumeTest {

    @Test
    fun `each section shape reports its own kind discriminator`() {
        val sections: List<Section> = listOf(
            Prose("Perfil", "texto"),
            Timeline("Experiencia", emptyList()),
            KeyValues("Competencias", emptyList()),
            Inline("Certificaciones", emptyList()),
        )
        assertEquals(
            listOf("prose", "timeline", "keyvalues", "inline"),
            sections.map { it.kind },
        )
    }

    @Test
    fun `a null end date means the role is current`() {
        val entry = Entry(
            role = "Ingeniero de Software",
            org = "Xetux Soluciones",
            location = null,
            start = YearMonth.of(2025, 2),
            end = null,
            bullets = listOf("Microservicios con Spring Boot"),
        )
        assertNull(entry.end)
        assertEquals(YearMonth.of(2025, 2), entry.start)
    }

    @Test
    fun `paper sizes carry their real dimensions`() {
        assertEquals(210.0, PaperSize.A4.widthMm)
        assertEquals(297.0, PaperSize.A4.heightMm)
        assertEquals("A4", PaperSize.A4.cssPageSize)
        assertEquals(215.9, PaperSize.CARTA.widthMm)
        assertEquals(279.4, PaperSize.CARTA.heightMm)
        assertEquals("Letter", PaperSize.CARTA.cssPageSize)
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `./gradlew :engine:test --tests '*ResumeTest*'`
Expected: FAIL, "Unresolved reference: Prose".

- [ ] **Step 3: Write the model**

`engine/src/main/kotlin/dev/resumekit/engine/model/Resume.kt`:

```kotlin
package dev.resumekit.engine.model

import java.time.YearMonth

data class Resume(val basics: Basics, val sections: List<Section>)

data class Basics(
    val name: String,
    val headline: String,
    val email: String? = null,
    val phone: String? = null,
    val location: String? = null,
    val links: List<Link> = emptyList(),
)

data class Link(val label: String, val url: String)

/**
 * Pebble cannot branch on a Kotlin sealed type, so every shape exposes a
 * string discriminator that templates read as `section.kind`.
 */
sealed interface Section {
    val title: String
    val kind: String
}

data class Prose(override val title: String, val body: String) : Section {
    override val kind = "prose"
}

data class Timeline(override val title: String, val entries: List<Entry>) : Section {
    override val kind = "timeline"
}

data class KeyValues(override val title: String, val rows: List<Row>) : Section {
    override val kind = "keyvalues"
}

data class Inline(override val title: String, val items: List<String>) : Section {
    override val kind = "inline"
}

data class Entry(
    val role: String,
    val org: String? = null,
    val location: String? = null,
    val start: YearMonth,
    val end: YearMonth? = null,
    val bullets: List<String> = emptyList(),
)

data class Row(val label: String, val items: List<String>)
```

`engine/src/main/kotlin/dev/resumekit/engine/render/PaperSize.kt`:

```kotlin
package dev.resumekit.engine.render

enum class PaperSize(
    val widthMm: Double,
    val heightMm: Double,
    val cssPageSize: String,
) {
    A4(210.0, 297.0, "A4"),
    CARTA(215.9, 279.4, "Letter");

    companion object {
        fun parse(text: String): PaperSize = when (text.lowercase()) {
            "a4" -> A4
            "carta", "letter" -> CARTA
            else -> throw IllegalArgumentException(
                "Papel desconocido: '$text'. Use 'a4' o 'carta'."
            )
        }
    }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `./gradlew :engine:test --tests '*ResumeTest*'`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add engine/src
git commit -m "feat(engine): domain model with open sections and paper sizes"
```

---

### Task 3: Markdown front-matter into Basics

**Files:**
- Create: `engine/src/main/kotlin/dev/resumekit/engine/input/MarkdownSource.kt`
- Create: `engine/src/main/kotlin/dev/resumekit/engine/input/ResumeParseException.kt`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/input/FrontMatterTest.kt`
- Test fixture: `engine/src/test/resources/fixtures/minimal.md`

**Interfaces:**
- Consumes: Task 2's `Basics`, `Link`, `Resume`.
- Produces: `class MarkdownSource { fun parse(file: java.io.File): Resume }` and `class ResumeParseException(val file: String, val line: Int, message: String) : RuntimeException`. In this task `parse` returns `Resume(basics, emptyList())`; Task 4 fills `sections`.

- [ ] **Step 1: Write the fixture**

`engine/src/test/resources/fixtures/minimal.md`:

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
Texto de perfil.
```

- [ ] **Step 2: Write the failing test**

`engine/src/test/kotlin/dev/resumekit/engine/input/FrontMatterTest.kt`:

```kotlin
package dev.resumekit.engine.input

import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertFailsWith
import java.io.File

class FrontMatterTest {

    private fun fixture(name: String) =
        File(javaClass.classLoader.getResource("fixtures/$name")!!.toURI())

    @Test
    fun `reads basics out of the front matter`() {
        val resume = MarkdownSource().parse(fixture("minimal.md"))
        assertEquals("Mariana Pacheco", resume.basics.name)
        assertEquals("emeariaenea@gmail.com", resume.basics.email)
        assertEquals("+58 424 221 6100", resume.basics.phone)
        assertEquals("Venezuela", resume.basics.location)
        assertEquals(1, resume.basics.links.size)
        assertEquals("LinkedIn", resume.basics.links[0].label)
    }

    @Test
    fun `a missing front matter block fails on line 1`() {
        val tmp = File.createTempFile("sin-fm", ".md").apply {
            writeText("## Perfil\ntexto\n")
            deleteOnExit()
        }
        val e = assertFailsWith<ResumeParseException> { MarkdownSource().parse(tmp) }
        assertEquals(1, e.line)
        assertEquals(true, e.message!!.contains("front-matter"))
    }

    @Test
    fun `a missing name fails and names the field`() {
        val tmp = File.createTempFile("sin-nombre", ".md").apply {
            writeText("---\nheadline: X\n---\n\n## Perfil\ntexto\n")
            deleteOnExit()
        }
        val e = assertFailsWith<ResumeParseException> { MarkdownSource().parse(tmp) }
        assertEquals(true, e.message!!.contains("name"))
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `./gradlew :engine:test --tests '*FrontMatterTest*'`
Expected: FAIL, "Unresolved reference: MarkdownSource".

- [ ] **Step 4: Write the exception type**

`engine/src/main/kotlin/dev/resumekit/engine/input/ResumeParseException.kt`:

```kotlin
package dev.resumekit.engine.input

class ResumeParseException(
    val file: String,
    val line: Int,
    override val message: String,
) : RuntimeException("$file:$line — $message")
```

- [ ] **Step 5: Write the front-matter half of MarkdownSource**

`engine/src/main/kotlin/dev/resumekit/engine/input/MarkdownSource.kt`:

```kotlin
package dev.resumekit.engine.input

import dev.resumekit.engine.model.Basics
import dev.resumekit.engine.model.Link
import dev.resumekit.engine.model.Resume
import java.io.File

class MarkdownSource {

    /** Front-matter text plus the 1-based line where the body starts. */
    internal data class Split(val frontMatter: String, val body: String, val bodyStartLine: Int)

    fun parse(file: File): Resume {
        val split = split(file)
        val fm = parseFrontMatter(file.name, split.frontMatter)
        return Resume(basics = fm, sections = emptyList())
    }

    internal fun split(file: File): Split {
        val lines = file.readLines()
        if (lines.firstOrNull()?.trim() != "---") {
            throw ResumeParseException(
                file.name, 1,
                "falta el bloque front-matter; el archivo debe empezar con una línea '---'",
            )
        }
        val close = lines.drop(1).indexOfFirst { it.trim() == "---" }
        if (close < 0) {
            throw ResumeParseException(
                file.name, lines.size,
                "el front-matter se abre en la línea 1 pero nunca se cierra con '---'",
            )
        }
        val closeLine = close + 2 // 1-based index of the closing marker
        return Split(
            frontMatter = lines.subList(1, closeLine - 1).joinToString("\n"),
            body = lines.drop(closeLine).joinToString("\n"),
            bodyStartLine = closeLine + 1,
        )
    }

    private fun parseFrontMatter(fileName: String, yaml: String): Basics {
        val map = SimpleYaml.parse(yaml, fileName)
        val name = map.string("name")
            ?: throw ResumeParseException(fileName, 2, "falta el campo obligatorio 'name'")
        val headline = map.string("headline")
            ?: throw ResumeParseException(fileName, 2, "falta el campo obligatorio 'headline'")
        return Basics(
            name = name,
            headline = headline,
            email = map.string("email"),
            phone = map.string("phone"),
            location = map.string("location"),
            links = map.links("links"),
        )
    }
}
```

- [ ] **Step 6: Write the YAML helper**

`kaml` is strict about typed deserialization, which fights a loosely-shaped front matter. Wrap snakeyaml-style map access instead.

Append to `MarkdownSource.kt`:

```kotlin
internal class SimpleYaml(private val map: Map<String, Any?>) {

    fun string(key: String): String? = map[key]?.toString()?.takeIf { it.isNotBlank() }

    @Suppress("UNCHECKED_CAST")
    fun links(key: String): List<Link> {
        val raw = map[key] as? List<Map<String, Any?>> ?: return emptyList()
        return raw.mapNotNull { item ->
            val label = item["label"]?.toString() ?: return@mapNotNull null
            val url = item["url"]?.toString() ?: return@mapNotNull null
            Link(label, url)
        }
    }

    companion object {
        @Suppress("UNCHECKED_CAST")
        fun parse(yaml: String, fileName: String): SimpleYaml = try {
            val loaded = org.yaml.snakeyaml.Yaml().load<Any?>(yaml)
            SimpleYaml((loaded as? Map<String, Any?>) ?: emptyMap())
        } catch (e: org.yaml.snakeyaml.error.YAMLException) {
            throw ResumeParseException(fileName, 2, "front-matter YAML inválido: ${e.message}")
        }
    }
}
```

Add snakeyaml to `gradle/libs.versions.toml` (`snakeyaml = "2.3"`, module `org.yaml:snakeyaml`) and to `engine/build.gradle.kts`. `kaml` stays for `template.yaml` in Task 5.

- [ ] **Step 7: Run the test to verify it passes**

Run: `./gradlew :engine:test --tests '*FrontMatterTest*'`
Expected: PASS, three tests.

- [ ] **Step 8: Commit**

```bash
git add engine gradle/libs.versions.toml
git commit -m "feat(engine): parse resume front matter into Basics"
```

---

### Task 4: Section parsing and shape inference

**Files:**
- Modify: `engine/src/main/kotlin/dev/resumekit/engine/input/MarkdownSource.kt`
- Create: `engine/src/main/kotlin/dev/resumekit/engine/input/SectionParser.kt`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/input/SectionParserTest.kt`
- Test fixture: `engine/src/test/resources/fixtures/mariana.md`

**Interfaces:**
- Consumes: Task 3's `MarkdownSource.split`, `ResumeParseException`; Task 2's section shapes.
- Produces: `internal class SectionParser(fileName: String, bodyStartLine: Int) { fun parse(body: String): List<Section> }`. `MarkdownSource.parse` now returns populated `sections`.

- [ ] **Step 1: Write the fixture**

`engine/src/test/resources/fixtures/mariana.md` — the front matter from Task 1's fixture, then:

```markdown
## Perfil
Ingeniero en Informática con experiencia en el desarrollo de soluciones de software.

## Experiencia

### Ingeniero de Software — Xetux Soluciones
*2025-02 – presente*

- Microservicios con Java y Spring Boot.
- Liderazgo técnico de equipos ágiles.

### Arquitecto de Soluciones TI — Banco Exterior
*2024-03 – 2025-02*
Caracas, Venezuela

- Arquitecturas de software para soluciones empresariales.

## Competencias técnicas

- **Backend:** Java, Spring Boot, Kotlin
- **Infraestructura:** Docker, DevOps

## Certificaciones

- DevOps: Despliegue de microservicios con Docker
- Test Automation Foundations
```

- [ ] **Step 2: Write the failing test**

`engine/src/test/kotlin/dev/resumekit/engine/input/SectionParserTest.kt`:

```kotlin
package dev.resumekit.engine.input

import dev.resumekit.engine.model.Inline
import dev.resumekit.engine.model.KeyValues
import dev.resumekit.engine.model.Prose
import dev.resumekit.engine.model.Timeline
import java.io.File
import java.time.YearMonth
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertFailsWith
import kotlin.test.assertNull

class SectionParserTest {

    private fun fixture(name: String) =
        File(javaClass.classLoader.getResource("fixtures/$name")!!.toURI())

    private val resume = MarkdownSource().parse(fixture("mariana.md"))

    @Test
    fun `sections keep the order they appear in the file`() {
        assertEquals(
            listOf("Perfil", "Experiencia", "Competencias técnicas", "Certificaciones"),
            resume.sections.map { it.title },
        )
    }

    @Test
    fun `shape is inferred from the content under each heading`() {
        assertEquals(
            listOf("prose", "timeline", "keyvalues", "inline"),
            resume.sections.map { it.kind },
        )
    }

    @Test
    fun `a timeline entry splits role from organisation`() {
        val exp = resume.sections.filterIsInstance<Timeline>().single()
        val first = exp.entries.first()
        assertEquals("Ingeniero de Software", first.role)
        assertEquals("Xetux Soluciones", first.org)
        assertEquals(YearMonth.of(2025, 2), first.start)
        assertNull(first.end)
        assertEquals(2, first.bullets.size)
        assertNull(first.location)
    }

    @Test
    fun `an optional location line is captured`() {
        val exp = resume.sections.filterIsInstance<Timeline>().single()
        val second = exp.entries[1]
        assertEquals("Caracas, Venezuela", second.location)
        assertEquals(YearMonth.of(2025, 2), second.end)
    }

    @Test
    fun `key value rows split label from items`() {
        val skills = resume.sections.filterIsInstance<KeyValues>().single()
        assertEquals("Backend", skills.rows[0].label)
        assertEquals(listOf("Java", "Spring Boot", "Kotlin"), skills.rows[0].items)
    }

    @Test
    fun `inline sections keep their items verbatim`() {
        val certs = resume.sections.filterIsInstance<Inline>().single()
        assertEquals("DevOps: Despliegue de microservicios con Docker", certs.items[0])
    }

    @Test
    fun `prose sections keep their text`() {
        val perfil = resume.sections.filterIsInstance<Prose>().single()
        assertEquals(true, perfil.body.startsWith("Ingeniero en Informática"))
    }

    @Test
    fun `a timeline entry without dates fails on its own line`() {
        val tmp = File.createTempFile("sin-fechas", ".md").apply {
            writeText(
                """
                ---
                name: X
                headline: Y
                ---

                ## Experiencia

                ### Ingeniero de Software — Xetux

                - bullet
                """.trimIndent() + "\n"
            )
            deleteOnExit()
        }
        val e = assertFailsWith<ResumeParseException> { MarkdownSource().parse(tmp) }
        assertEquals(7, e.line)
        assertEquals(true, e.message.contains("rango de fechas"))
    }

    @Test
    fun `an explicit shape annotation overrides inference`() {
        val tmp = File.createTempFile("forzado", ".md").apply {
            writeText(
                """
                ---
                name: X
                headline: Y
                ---

                ## Notas {.prose}

                - esto sería inline sin la anotación
                """.trimIndent() + "\n"
            )
            deleteOnExit()
        }
        val r = MarkdownSource().parse(tmp)
        assertEquals("prose", r.sections.single().kind)
        assertEquals("Notas", r.sections.single().title)
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `./gradlew :engine:test --tests '*SectionParserTest*'`
Expected: FAIL — `resume.sections` is empty, so `sections.map` returns an empty list.

- [ ] **Step 4: Write the section parser**

`engine/src/main/kotlin/dev/resumekit/engine/input/SectionParser.kt`:

```kotlin
package dev.resumekit.engine.input

import dev.resumekit.engine.model.Entry
import dev.resumekit.engine.model.Inline
import dev.resumekit.engine.model.KeyValues
import dev.resumekit.engine.model.Prose
import dev.resumekit.engine.model.Row
import dev.resumekit.engine.model.Section
import dev.resumekit.engine.model.Timeline
import java.time.YearMonth

internal class SectionParser(
    private val fileName: String,
    private val bodyStartLine: Int,
) {

    private data class Block(val title: String, val forced: String?, val lines: List<Line>)
    private data class Line(val text: String, val number: Int)

    private val headingShape = Regex("""^##\s+(.+?)(?:\s+\{\.(prose|timeline|keyvalues|inline)})?\s*$""")
    private val entryHeading = Regex("""^###\s+(.+?)(?:\s+—\s+(.+))?\s*$""")
    private val dateRange = Regex("""^\*(\d{4})-(\d{2})\s*[–-]\s*(presente|(\d{4})-(\d{2}))\*$""",
        RegexOption.IGNORE_CASE)
    private val keyValueItem = Regex("""^-\s+\*\*(.+?):\*\*\s*(.+)$""")
    private val plainItem = Regex("""^-\s+(.+)$""")

    fun parse(body: String): List<Section> =
        blocks(body).map { toSection(it) }

    private fun blocks(body: String): List<Block> {
        val numbered = body.lines().mapIndexed { i, t -> Line(t, bodyStartLine + i) }
        val result = mutableListOf<Block>()
        var current: Block? = null
        for (line in numbered) {
            val m = headingShape.matchEntire(line.text)
            if (m != null) {
                current?.let { result += it }
                current = Block(m.groupValues[1].trim(), m.groupValues[2].ifBlank { null }, mutableListOf())
            } else if (current != null) {
                (current.lines as MutableList) += line
            }
        }
        current?.let { result += it }
        return result
    }

    private fun toSection(block: Block): Section {
        val meaningful = block.lines.filter { it.text.isNotBlank() }
        val shape = block.forced ?: when {
            meaningful.any { it.text.startsWith("### ") } -> "timeline"
            meaningful.isNotEmpty() && meaningful.all { keyValueItem.matches(it.text) } -> "keyvalues"
            meaningful.isNotEmpty() && meaningful.all { plainItem.matches(it.text) } -> "inline"
            else -> "prose"
        }
        return when (shape) {
            "timeline" -> Timeline(block.title, entries(block))
            "keyvalues" -> KeyValues(block.title, meaningful.map { row(it) })
            "inline" -> Inline(block.title, meaningful.map { plainItem.matchEntire(it.text)!!.groupValues[1] })
            else -> Prose(block.title, meaningful.joinToString(" ") { it.text.trim() })
        }
    }

    private fun row(line: Line): Row {
        val m = keyValueItem.matchEntire(line.text)!!
        return Row(m.groupValues[1], m.groupValues[2].split(",").map { it.trim() })
    }

    private fun entries(block: Block): List<Entry> {
        val result = mutableListOf<Entry>()
        var head: Line? = null
        var buffer = mutableListOf<Line>()

        fun flush() {
            head?.let { result += entry(it, buffer) }
            buffer = mutableListOf()
        }

        for (line in block.lines) {
            if (line.text.startsWith("### ")) {
                flush()
                head = line
            } else if (head != null && line.text.isNotBlank()) {
                buffer += line
            }
        }
        flush()
        return result
    }

    private fun entry(head: Line, rest: List<Line>): Entry {
        val h = entryHeading.matchEntire(head.text)
            ?: throw ResumeParseException(fileName, head.number, "encabezado de entrada mal formado")
        val dateLine = rest.firstOrNull { dateRange.matches(it.text) }
            ?: throw ResumeParseException(
                fileName, head.number,
                "falta el rango de fechas bajo \"${head.text.trim()}\"; " +
                    "se esperaba una línea como: *2025-02 – presente*",
            )
        val d = dateRange.matchEntire(dateLine.text)!!
        val start = YearMonth.of(d.groupValues[1].toInt(), d.groupValues[2].toInt())
        val end = if (d.groupValues[4].isBlank()) null
                  else YearMonth.of(d.groupValues[4].toInt(), d.groupValues[5].toInt())

        val afterDate = rest.dropWhile { it !== dateLine }.drop(1)
        val location = afterDate.firstOrNull { !plainItem.matches(it.text) }?.text?.trim()
        val bullets = rest.mapNotNull { plainItem.matchEntire(it.text)?.groupValues?.get(1) }

        return Entry(
            role = h.groupValues[1].trim(),
            org = h.groupValues[2].ifBlank { null }?.trim(),
            location = location,
            start = start,
            end = end,
            bullets = bullets,
        )
    }
}
```

- [ ] **Step 5: Wire the parser into MarkdownSource**

In `MarkdownSource.parse`, replace `sections = emptyList()`:

```kotlin
    fun parse(file: File): Resume {
        val split = split(file)
        val basics = parseFrontMatter(file.name, split.frontMatter)
        val sections = SectionParser(file.name, split.bodyStartLine).parse(split.body)
        return Resume(basics = basics, sections = sections)
    }
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `./gradlew :engine:test --tests '*SectionParserTest*'`
Expected: PASS, nine tests.

- [ ] **Step 7: Run the whole suite**

Run: `./gradlew test`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add engine
git commit -m "feat(engine): parse sections with shape inference and located errors"
```

---

### Task 5: Template catalog

**Files:**
- Create: `engine/src/main/kotlin/dev/resumekit/engine/template/TemplateCatalog.kt`
- Create: `templates/trazo/template.yaml`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/template/TemplateCatalogTest.kt`

**Interfaces:**
- Consumes: Task 2's `PaperSize`.
- Produces: `data class TemplateMeta(id, name, description, papers: Set<PaperSize>, autofit: Map<String, ClosedFloatingPointRange<Double>>, dir: java.io.File)`; `class TemplateCatalog(root: File) { fun list(): List<TemplateMeta>; fun get(id: String): TemplateMeta }`. `get` throws `IllegalArgumentException` listing available ids.

- [ ] **Step 1: Write the template metadata**

`templates/trazo/template.yaml`:

```yaml
name: Trazo
description: Línea de tiempo con nodos en las secciones cronológicas
papers: [a4, carta]
autofit:
  "--t-base": [8.6, 9.1]
  "--lh": [1.25, 1.34]
```

Autofit bounds are plain numbers; the unit lives in the template CSS (`pt` for `--t-base`, unitless for `--lh`). Mixing units into the YAML would force unit parsing for no gain.

- [ ] **Step 2: Write the failing test**

`engine/src/test/kotlin/dev/resumekit/engine/template/TemplateCatalogTest.kt`:

```kotlin
package dev.resumekit.engine.template

import dev.resumekit.engine.render.PaperSize
import java.io.File
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertFailsWith
import kotlin.test.assertTrue

class TemplateCatalogTest {

    private val catalog = TemplateCatalog(File("../templates"))

    @Test
    fun `finds trazo and reads its metadata`() {
        val meta = catalog.get("trazo")
        assertEquals("Trazo", meta.name)
        assertEquals(setOf(PaperSize.A4, PaperSize.CARTA), meta.papers)
    }

    @Test
    fun `reads autofit bounds`() {
        val meta = catalog.get("trazo")
        assertEquals(8.6, meta.autofit["--t-base"]!!.start)
        assertEquals(9.1, meta.autofit["--t-base"]!!.endInclusive)
    }

    @Test
    fun `an unknown id lists what is available`() {
        val e = assertFailsWith<IllegalArgumentException> { catalog.get("noexiste") }
        assertTrue(e.message!!.contains("trazo"))
    }

    @Test
    fun `list returns every template directory`() {
        assertTrue(catalog.list().map { it.id }.contains("trazo"))
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `./gradlew :engine:test --tests '*TemplateCatalogTest*'`
Expected: FAIL, "Unresolved reference: TemplateCatalog".

- [ ] **Step 4: Write the catalog**

`engine/src/main/kotlin/dev/resumekit/engine/template/TemplateCatalog.kt`:

```kotlin
package dev.resumekit.engine.template

import dev.resumekit.engine.render.PaperSize
import java.io.File

data class TemplateMeta(
    val id: String,
    val name: String,
    val description: String,
    val papers: Set<PaperSize>,
    val autofit: Map<String, ClosedFloatingPointRange<Double>>,
    val dir: File,
) {
    val templateFile: File get() = File(dir, "resume.peb")
}

class TemplateCatalog(private val root: File) {

    fun list(): List<TemplateMeta> =
        (root.listFiles { f: File -> f.isDirectory && File(f, "template.yaml").isFile } ?: emptyArray())
            .sortedBy { it.name }
            .map { read(it) }

    fun get(id: String): TemplateMeta =
        list().firstOrNull { it.id == id }
            ?: throw IllegalArgumentException(
                "Plantilla desconocida: '$id'. Disponibles: " +
                    list().joinToString(", ") { "${it.id} (${it.name})" }
            )

    @Suppress("UNCHECKED_CAST")
    private fun read(dir: File): TemplateMeta {
        val yaml = org.yaml.snakeyaml.Yaml()
            .load<Map<String, Any?>>(File(dir, "template.yaml").readText()) ?: emptyMap()

        val papers = (yaml["papers"] as? List<Any?> ?: listOf("a4"))
            .map { PaperSize.parse(it.toString()) }.toSet()

        val autofit = (yaml["autofit"] as? Map<String, Any?> ?: emptyMap())
            .mapValues { (_, v) ->
                val pair = v as List<Any?>
                (pair[0].toString().toDouble())..(pair[1].toString().toDouble())
            }

        return TemplateMeta(
            id = dir.name,
            name = yaml["name"]?.toString() ?: dir.name,
            description = yaml["description"]?.toString() ?: "",
            papers = papers,
            autofit = autofit,
            dir = dir,
        )
    }
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `./gradlew :engine:test --tests '*TemplateCatalogTest*'`
Expected: PASS, four tests.

- [ ] **Step 6: Commit**

```bash
git add engine templates
git commit -m "feat(engine): template catalog reading template.yaml from disk"
```

---

### Task 6: Port Trazo to Pebble and render HTML

**Files:**
- Create: `templates/trazo/resume.peb`
- Create: `engine/src/main/kotlin/dev/resumekit/engine/template/PebbleRenderer.kt`
- Create: `engine/src/main/kotlin/dev/resumekit/engine/template/DateFormat.kt`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/template/PebbleRendererTest.kt`

**Interfaces:**
- Consumes: Tasks 2, 4, 5.
- Produces: `class DateFormat(locale: java.util.Locale) { fun range(start: YearMonth, end: YearMonth?): String }`; `class PebbleRenderer { fun render(meta: TemplateMeta, resume: Resume, paper: PaperSize, overrides: Map<String, String> = emptyMap()): String }`.

**Source material:** the finished HTML lives at `C:\Users\Mariana\AppData\Local\Temp\claude\C--Users-Mariana-Documents-Projects-code-m-resume-template-engine\5ea13649-c872-4f92-a1d4-7a1765048f82\scratchpad\out\cv-mariana-pacheco-trazo-a4.html`. Copy it into `templates/trazo/resume.peb` verbatim first, then replace content with Pebble tags. Do not retype the CSS by hand.

- [ ] **Step 1: Write the failing test**

`engine/src/test/kotlin/dev/resumekit/engine/template/PebbleRendererTest.kt`:

```kotlin
package dev.resumekit.engine.template

import dev.resumekit.engine.input.MarkdownSource
import dev.resumekit.engine.render.PaperSize
import java.io.File
import java.time.YearMonth
import java.util.Locale
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class PebbleRendererTest {

    private val catalog = TemplateCatalog(File("../templates"))
    private val resume = MarkdownSource().parse(
        File(javaClass.classLoader.getResource("fixtures/mariana.md")!!.toURI())
    )
    private val html = PebbleRenderer().render(catalog.get("trazo"), resume, PaperSize.A4)

    @Test
    fun `date ranges render in Spanish with present as a word`() {
        val fmt = DateFormat(Locale.forLanguageTag("es"))
        assertEquals("feb. 2025 – Presente", fmt.range(YearMonth.of(2025, 2), null))
        assertEquals("mar. 2024 – feb. 2025",
            fmt.range(YearMonth.of(2024, 3), YearMonth.of(2025, 2)))
    }

    @Test
    fun `renders the name and every section title`() {
        assertTrue(html.contains("Mariana Pacheco"))
        assertTrue(html.contains("Experiencia"))
        assertTrue(html.contains("Certificaciones"))
    }

    @Test
    fun `renders one entry block per timeline entry`() {
        assertEquals(2, Regex("class=\"entry\"").findAll(html).count())
    }

    @Test
    fun `carries the A4 page size into the print rule`() {
        assertTrue(html.contains("size:A4"))
    }

    @Test
    fun `carta swaps the page size and the sheet width`() {
        val carta = PebbleRenderer().render(catalog.get("trazo"), resume, PaperSize.CARTA)
        assertTrue(carta.contains("size:Letter"))
        assertTrue(carta.contains("215.9mm"))
    }

    @Test
    fun `narrow screen rules stay scoped to screen`() {
        // Regression guard: a bare @media (max-width) also fires when printing,
        // because the paged viewport is 210mm. This stacked dates under the role.
        assertTrue(html.contains("@media screen and (max-width"))
        assertTrue(!Regex("""@media\s*\(max-width""").containsMatchIn(html))
    }

    @Test
    fun `token overrides reach the rendered CSS`() {
        val tight = PebbleRenderer().render(
            catalog.get("trazo"), resume, PaperSize.A4, mapOf("--lh" to "1.28"),
        )
        assertTrue(tight.contains("--lh:1.28"))
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `./gradlew :engine:test --tests '*PebbleRendererTest*'`
Expected: FAIL, "Unresolved reference: PebbleRenderer".

- [ ] **Step 3: Write the date formatter**

`engine/src/main/kotlin/dev/resumekit/engine/template/DateFormat.kt`:

```kotlin
package dev.resumekit.engine.template

import java.time.YearMonth
import java.time.format.DateTimeFormatter
import java.util.Locale

class DateFormat(private val locale: Locale = Locale.forLanguageTag("es")) {

    private val month = DateTimeFormatter.ofPattern("MMM yyyy", locale)

    private val present = when (locale.language) {
        "en" -> "Present"
        else -> "Presente"
    }

    fun one(value: YearMonth): String = value.format(month)

    fun range(start: YearMonth, end: YearMonth?): String =
        "${one(start)} – ${end?.let { one(it) } ?: present}"
}
```

- [ ] **Step 4: Port the template**

Copy the finished HTML to `templates/trazo/resume.peb`, then make exactly these substitutions. Everything else — every CSS rule — stays byte-for-byte.

Header:

```html
<h1 class="name">{{ cv.basics.name }}</h1>
<p class="headline">{{ cv.basics.headline }}</p>
<p class="contact">
  {% if cv.basics.email %}<span><a href="mailto:{{ cv.basics.email }}">{{ cv.basics.email }}</a></span>{% endif %}
  {% if cv.basics.phone %}<span><a href="tel:{{ cv.basics.phone | replace({' ': ''}) }}">{{ cv.basics.phone }}</a></span>{% endif %}
  {% if cv.basics.location %}<span>{{ cv.basics.location }}</span>{% endif %}
  {% for l in cv.basics.links %}<span><a href="{{ l.url }}">{{ l.url | replace({'https://': ''}) }}</a></span>{% endfor %}
</p>
```

Body — one loop replaces all six hand-written sections:

```html
{% for section in cv.sections %}
<section class="section">
  <h2>{{ section.title }}</h2>

  {% if section.kind == 'prose' %}
    <p class="summary">{{ section.body }}</p>

  {% elseif section.kind == 'timeline' %}
    {% for e in section.entries %}
      <div class="entry">
        <div class="entry-head">
          <h3 class="entry-title">{{ e.role }}{% if e.org %} <span class="org">— {{ e.org }}</span>{% endif %}</h3>
          <span class="entry-when">{{ fmt.range(e.start, e.end) }}</span>
        </div>
        {% if e.location %}<p class="entry-where">{{ e.location }}</p>{% endif %}
        {% if e.bullets is not empty %}
        <ul>{% for b in e.bullets %}<li>{{ b }}</li>{% endfor %}</ul>
        {% endif %}
      </div>
    {% endfor %}

  {% elseif section.kind == 'keyvalues' %}
    <dl class="rows">
      {% for r in section.rows %}
        <dt>{{ r.label }}</dt>
        <dd>{{ r.items | join(', ') }}</dd>
      {% endfor %}
    </dl>

  {% else %}
    <p class="certs">{% for i in section.items %}<span>{{ i }}</span>{% endfor %}</p>
  {% endif %}
</section>
{% endfor %}
```

Paper size and token overrides — replace the two hard-coded spots:

```html
@page{size:{{ paper.cssPageSize }};margin:0;}
```

```css
  --sheet-w:{{ paper.widthMm }}mm; --sheet-h:{{ paper.heightMm }}mm;
{% for name, value in overrides %}  {{ name }}:{{ value }};
{% endfor %}
```

Place the override loop **last** inside the `body{ … }` token block so it wins on cascade order.

Also delete the `<div class="picker">` block and the trailing `<script>` — those belonged to the comparison artifact, not to a template.

- [ ] **Step 5: Write the renderer**

`engine/src/main/kotlin/dev/resumekit/engine/template/PebbleRenderer.kt`:

```kotlin
package dev.resumekit.engine.template

import dev.resumekit.engine.model.Resume
import dev.resumekit.engine.render.PaperSize
import io.pebbletemplates.pebble.PebbleEngine
import io.pebbletemplates.pebble.loader.FileLoader
import java.io.StringWriter
import java.util.Locale

class PebbleRenderer(private val locale: Locale = Locale.forLanguageTag("es")) {

    fun render(
        meta: TemplateMeta,
        resume: Resume,
        paper: PaperSize,
        overrides: Map<String, String> = emptyMap(),
    ): String {
        require(paper in meta.papers) {
            "La plantilla '${meta.id}' no declara soporte para $paper. Declara: ${meta.papers}"
        }

        // Cache off: templates are edited live and must never be recompiled to reload.
        val engine = PebbleEngine.Builder()
            .loader(FileLoader().apply { prefix = meta.dir.absolutePath })
            .cacheActive(false)
            .strictVariables(false)
            .build()

        val out = StringWriter()
        engine.getTemplate("resume.peb").evaluate(
            out,
            mapOf(
                "cv" to resume,
                "paper" to paper,
                "fmt" to DateFormat(locale),
                "overrides" to overrides,
            ),
        )
        return out.toString()
    }
}
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `./gradlew :engine:test --tests '*PebbleRendererTest*'`
Expected: PASS, seven tests.

- [ ] **Step 7: Commit**

```bash
git add engine templates
git commit -m "feat(engine): render Trazo through Pebble for A4 and Carta"
```

---

### Task 7: Render HTML to PDF

**Files:**
- Create: `engine/src/main/kotlin/dev/resumekit/engine/render/PdfRenderer.kt`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/render/PdfRendererTest.kt`

**Interfaces:**
- Consumes: Task 2's `PaperSize`, Task 6's HTML string.
- Produces: `class PdfRenderer : AutoCloseable { fun render(html: String, target: java.io.File) }`. Throws `MissingBrowserException` with the install command when Chromium is absent.

- [ ] **Step 1: Write the failing test**

`engine/src/test/kotlin/dev/resumekit/engine/render/PdfRendererTest.kt`:

```kotlin
package dev.resumekit.engine.render

import java.io.File
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class PdfRendererTest {

    private fun mediaBox(pdf: File): Pair<Double, Double> {
        val m = Regex("""/MediaBox\s*\[\s*[\d.]+\s+[\d.]+\s+([\d.]+)\s+([\d.]+)""")
            .find(pdf.readBytes().toString(Charsets.ISO_8859_1))!!
        return m.groupValues[1].toDouble() to m.groupValues[2].toDouble()
    }

    @Test
    fun `writes a real PDF at the A4 media box`() {
        val out = File.createTempFile("a4", ".pdf").apply { deleteOnExit() }
        PdfRenderer().use {
            it.render("<html><head><style>@page{size:A4;margin:0}</style></head><body>x</body></html>", out)
        }
        assertTrue(out.length() > 0)
        val (w, h) = mediaBox(out)
        assertEquals(595.0, w, 2.0)
        assertEquals(842.0, h, 2.0)
    }

    @Test
    fun `writes a real PDF at the Carta media box`() {
        val out = File.createTempFile("carta", ".pdf").apply { deleteOnExit() }
        PdfRenderer().use {
            it.render("<html><head><style>@page{size:Letter;margin:0}</style></head><body>x</body></html>", out)
        }
        val (w, h) = mediaBox(out)
        assertEquals(612.0, w, 2.0)
        assertEquals(792.0, h, 2.0)
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `./gradlew :engine:test --tests '*PdfRendererTest*'`
Expected: FAIL, "Unresolved reference: PdfRenderer".

- [ ] **Step 3: Write the renderer**

`engine/src/main/kotlin/dev/resumekit/engine/render/PdfRenderer.kt`:

```kotlin
package dev.resumekit.engine.render

import com.microsoft.playwright.Browser
import com.microsoft.playwright.Page
import com.microsoft.playwright.Playwright
import java.io.File

class MissingBrowserException(cause: Throwable) : RuntimeException(
    """
    Chromium no está instalado para Playwright.
    Instálalo con:  ./gradlew :engine:run --args="install-browser"
    o directamente: mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install chromium"
    """.trimIndent(),
    cause,
)

class PdfRenderer : AutoCloseable {

    private val playwright: Playwright = Playwright.create()

    private val browser: Browser = try {
        playwright.chromium().launch()
    } catch (e: RuntimeException) {
        playwright.close()
        throw MissingBrowserException(e)
    }

    fun render(html: String, target: File) {
        target.parentFile?.mkdirs()
        browser.newPage().use { page: Page ->
            page.setContent(html)
            // Web fonts are remote; wait for them before laying out the page.
            page.waitForLoadState(com.microsoft.playwright.options.LoadState.NETWORKIDLE)
            page.pdf(
                Page.PdfOptions()
                    .setPath(target.toPath())
                    .setPreferCSSPageSize(true)   // honour @page, do not impose a size
                    .setPrintBackground(true)     // the rail nodes have a filled background
                    .setDisplayHeaderFooter(false),
            )
        }
    }

    override fun close() {
        browser.close()
        playwright.close()
    }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `./gradlew :engine:test --tests '*PdfRendererTest*'`
Expected: PASS. First run downloads Chromium (~150 MB) and may take several minutes.

- [ ] **Step 5: Commit**

```bash
git add engine
git commit -m "feat(engine): render HTML to PDF via Playwright honouring @page"
```

---

### Task 8: Output profiles

**Files:**
- Create: `engine/src/main/kotlin/dev/resumekit/engine/model/Profile.kt`
- Modify: `engine/src/main/kotlin/dev/resumekit/engine/input/MarkdownSource.kt`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/model/ProfileTest.kt`
- Test fixture: `engine/src/test/resources/fixtures/con-perfil.md`

**Interfaces:**
- Consumes: Tasks 2, 3, 4.
- Produces: `data class SectionLimit(val limit: Int?, val bulletsPerEntry: Int?)`; `data class Profile(name, maxPages: Int, sections: List<String>, limits: Map<String, SectionLimit>) { fun apply(resume: Resume): Resume }`; `MarkdownSource.parse` now also exposes `fun profiles(file: File): Map<String, Profile>`.

- [ ] **Step 1: Write the fixture**

`engine/src/test/resources/fixtures/con-perfil.md` — the body of `mariana.md`, with this front matter added after `location`:

```yaml
profiles:
  backend-1p:
    maxPages: 1
    sections: [Experiencia, Perfil]
    Experiencia: { limit: 1, bulletsPerEntry: 1 }
```

Note `sections` deliberately lists Experiencia before Perfil, and omits the other two.

- [ ] **Step 2: Write the failing test**

`engine/src/test/kotlin/dev/resumekit/engine/model/ProfileTest.kt`:

```kotlin
package dev.resumekit.engine.model

import dev.resumekit.engine.input.MarkdownSource
import java.io.File
import kotlin.test.Test
import kotlin.test.assertEquals

class ProfileTest {

    private fun fixture(name: String) =
        File(javaClass.classLoader.getResource("fixtures/$name")!!.toURI())

    private val file = fixture("con-perfil.md")
    private val resume = MarkdownSource().parse(file)
    private val profile = MarkdownSource().profiles(file).getValue("backend-1p")

    @Test
    fun `reads maxPages`() {
        assertEquals(1, profile.maxPages)
    }

    @Test
    fun `sections controls both selection and order`() {
        val out = profile.apply(resume)
        assertEquals(listOf("Experiencia", "Perfil"), out.sections.map { it.title })
    }

    @Test
    fun `limit trims a timeline to its most recent entries`() {
        val out = profile.apply(resume)
        val exp = out.sections.filterIsInstance<Timeline>().single()
        assertEquals(1, exp.entries.size)
        assertEquals("Ingeniero de Software", exp.entries[0].role)
    }

    @Test
    fun `bulletsPerEntry trims bullets`() {
        val out = profile.apply(resume)
        val exp = out.sections.filterIsInstance<Timeline>().single()
        assertEquals(1, exp.entries[0].bullets.size)
    }

    @Test
    fun `a file without profiles yields none`() {
        assertEquals(emptyMap(), MarkdownSource().profiles(fixture("mariana.md")))
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `./gradlew :engine:test --tests '*ProfileTest*'`
Expected: FAIL, "Unresolved reference: profiles".

- [ ] **Step 4: Write the profile**

`engine/src/main/kotlin/dev/resumekit/engine/model/Profile.kt`:

```kotlin
package dev.resumekit.engine.model

data class SectionLimit(val limit: Int? = null, val bulletsPerEntry: Int? = null)

data class Profile(
    val name: String,
    val maxPages: Int = 1,
    val sections: List<String> = emptyList(),
    val limits: Map<String, SectionLimit> = emptyMap(),
) {

    /** Selects, orders and trims. Never mutates the source resume. */
    fun apply(resume: Resume): Resume {
        val chosen =
            if (sections.isEmpty()) resume.sections
            else sections.mapNotNull { title -> resume.sections.firstOrNull { it.title == title } }

        return resume.copy(sections = chosen.map { trim(it) })
    }

    private fun trim(section: Section): Section {
        val rule = limits[section.title] ?: return section
        return when (section) {
            is Timeline -> Timeline(
                section.title,
                section.entries
                    .let { if (rule.limit != null) it.take(rule.limit) else it }
                    .map { e ->
                        if (rule.bulletsPerEntry != null) e.copy(bullets = e.bullets.take(rule.bulletsPerEntry))
                        else e
                    },
            )
            is Inline -> if (rule.limit != null) Inline(section.title, section.items.take(rule.limit)) else section
            is KeyValues -> if (rule.limit != null) KeyValues(section.title, section.rows.take(rule.limit)) else section
            is Prose -> section
        }
    }
}
```

- [ ] **Step 5: Read profiles in MarkdownSource**

Add to `MarkdownSource`:

```kotlin
    @Suppress("UNCHECKED_CAST")
    fun profiles(file: File): Map<String, Profile> {
        val raw = SimpleYaml.parse(split(file).frontMatter, file.name).raw["profiles"] as? Map<String, Any?>
            ?: return emptyMap()

        return raw.mapValues { (name, body) ->
            val map = body as? Map<String, Any?> ?: emptyMap()
            val limits = map.entries
                .filter { it.value is Map<*, *> }
                .associate { (title, cfg) ->
                    val c = cfg as Map<String, Any?>
                    title to SectionLimit(
                        limit = (c["limit"] as? Number)?.toInt(),
                        bulletsPerEntry = (c["bulletsPerEntry"] as? Number)?.toInt(),
                    )
                }
            Profile(
                name = name,
                maxPages = (map["maxPages"] as? Number)?.toInt() ?: 1,
                sections = (map["sections"] as? List<Any?>)?.map { it.toString() } ?: emptyList(),
                limits = limits,
            )
        }
    }
```

Expose the backing map on `SimpleYaml` by adding `val raw: Map<String, Any?> get() = map`.

- [ ] **Step 6: Run the test to verify it passes**

Run: `./gradlew :engine:test --tests '*ProfileTest*'`
Expected: PASS, five tests.

- [ ] **Step 7: Commit**

```bash
git add engine
git commit -m "feat(engine): output profiles select, order and trim sections"
```

---

### Task 9: Page audit

**Files:**
- Create: `engine/src/main/kotlin/dev/resumekit/engine/render/PageAudit.kt`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/render/PageAuditTest.kt`

**Interfaces:**
- Consumes: Task 8's `Profile`, Task 7's rendered PDF file.
- Produces: `data class AuditResult(val pages: Int, val fits: Boolean, val overflowing: List<String>, val diagnostic: String)`; `class PageAudit { fun audit(pdf: File, profile: Profile, resume: Resume): AuditResult }`. Uses PDFBox to count pages and to extract per-page text.

- [ ] **Step 1: Write the failing test**

`engine/src/test/kotlin/dev/resumekit/engine/render/PageAuditTest.kt`:

```kotlin
package dev.resumekit.engine.render

import dev.resumekit.engine.model.Inline
import dev.resumekit.engine.model.Profile
import dev.resumekit.engine.model.Prose
import dev.resumekit.engine.model.Basics
import dev.resumekit.engine.model.Resume
import java.io.File
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class PageAuditTest {

    private val resume = Resume(
        basics = Basics("Mariana Pacheco", "Ingeniera"),
        sections = listOf(Prose("Perfil", "texto"), Inline("Certificaciones", listOf("A", "B"))),
    )

    private fun pdfOf(html: String): File {
        val f = File.createTempFile("audit", ".pdf").apply { deleteOnExit() }
        PdfRenderer().use { it.render(html, f) }
        return f
    }

    @Test
    fun `a one page pdf fits a maxPages of one`() {
        val pdf = pdfOf("<html><head><style>@page{size:A4;margin:0}</style></head><body>Perfil</body></html>")
        val r = PageAudit().audit(pdf, Profile("p", maxPages = 1), resume)
        assertEquals(1, r.pages)
        assertTrue(r.fits)
    }

    @Test
    fun `a two page pdf fails and names the sections that cross the break`() {
        val pdf = pdfOf(
            """
            <html><head><style>@page{size:A4;margin:0}
            .p{height:290mm}</style></head>
            <body><div class="p">Perfil</div><div>Certificaciones</div></body></html>
            """.trimIndent()
        )
        val r = PageAudit().audit(pdf, Profile("p", maxPages = 1), resume)
        assertEquals(2, r.pages)
        assertTrue(!r.fits)
        assertTrue(r.overflowing.contains("Certificaciones"))
        assertTrue(r.diagnostic.contains("2 páginas (máximo 1)"))
        assertTrue(r.diagnostic.contains("maxPages 1 → 2"))
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `./gradlew :engine:test --tests '*PageAuditTest*'`
Expected: FAIL, "Unresolved reference: PageAudit".

- [ ] **Step 3: Write the audit**

`engine/src/main/kotlin/dev/resumekit/engine/render/PageAudit.kt`:

```kotlin
package dev.resumekit.engine.render

import dev.resumekit.engine.model.Profile
import dev.resumekit.engine.model.Resume
import dev.resumekit.engine.model.Timeline
import org.apache.pdfbox.Loader
import org.apache.pdfbox.text.PDFTextStripper
import java.io.File

data class AuditResult(
    val pages: Int,
    val fits: Boolean,
    val overflowing: List<String>,
    val diagnostic: String,
)

class PageAudit {

    fun audit(pdf: File, profile: Profile, resume: Resume): AuditResult {
        Loader.loadPDF(pdf).use { doc ->
            val pages = doc.numberOfPages
            if (pages <= profile.maxPages) {
                return AuditResult(pages, true, emptyList(), "✓ ${profile.name}: $pages página(s)")
            }

            val beyond = buildString {
                val stripper = PDFTextStripper()
                for (p in (profile.maxPages + 1)..pages) {
                    stripper.startPage = p
                    stripper.endPage = p
                    append(stripper.getText(doc))
                }
            }

            val overflowing = resume.sections.map { it.title }.filter { beyond.contains(it) }
            val excess = beyond.trim().length

            return AuditResult(pages, false, overflowing, diagnostic(profile, resume, pages, overflowing, excess))
        }
    }

    private fun diagnostic(
        profile: Profile,
        resume: Resume,
        pages: Int,
        overflowing: List<String>,
        excess: Int,
    ): String = buildString {
        appendLine("✗ ${profile.name}: $pages páginas (máximo ${profile.maxPages})")
        appendLine("  Cruzan el corte: ${overflowing.joinToString(", ").ifBlank { "(sin título identificable)" }}" +
            " (~$excess caracteres sobrantes)")
        appendLine("  Opciones:")
        resume.sections.filterIsInstance<Timeline>().forEach { t ->
            val current = profile.limits[t.title]?.limit ?: t.entries.size
            if (current > 1) appendLine("            ${t.title}.limit $current → ${current - 1}")
        }
        append("            maxPages ${profile.maxPages} → ${profile.maxPages + 1}")
    }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `./gradlew :engine:test --tests '*PageAuditTest*'`
Expected: PASS, two tests.

- [ ] **Step 5: Commit**

```bash
git add engine
git commit -m "feat(engine): audit page count and emit an actionable diagnostic"
```

---

### Task 10: ResumeEngine facade and the build command

**Files:**
- Create: `engine/src/main/kotlin/dev/resumekit/engine/ResumeEngine.kt`
- Create: `cli/src/main/kotlin/dev/resumekit/cli/Main.kt`
- Modify: `cli/build.gradle.kts` (add the `application` plugin)
- Create: `resumes/mariana.md`
- Test: `engine/src/test/kotlin/dev/resumekit/engine/ResumeEngineTest.kt`

**Interfaces:**
- Consumes: every prior task.
- Produces: `data class BuildRequest(source: File, templateId: String, papers: List<PaperSize>, profileName: String?, outDir: File, autofit: Boolean = false)`; `data class BuildOutput(val pdf: File, val audit: AuditResult)`; `class ResumeEngine(templatesRoot: File) { fun build(request: BuildRequest): List<BuildOutput>; fun templates(): List<TemplateMeta> }`. `build` throws `OverflowException` when any audit fails.

- [ ] **Step 1: Create the real resume data**

`resumes/mariana.md` — the fixture content from Task 4, using the real values: name `Mariana Pacheco`, headline `Ingeniero en Informática — Desarrollo Backend y Arquitectura de Software`, email `emeariaenea@gmail.com`, phone `"+58 424 221 6100"`, location `Venezuela`, LinkedIn `https://linkedin.com/in/emeariaenea`, and the four roles (Xetux, Banco Exterior, Keep Typing, Apps2Go) with the bullets already written in `cv-mariana-pacheco-trazo-a4.html`. Copy the text from that file; do not re-derive it from the LinkedIn PDF.

- [ ] **Step 2: Write the failing test**

`engine/src/test/kotlin/dev/resumekit/engine/ResumeEngineTest.kt`:

```kotlin
package dev.resumekit.engine

import dev.resumekit.engine.render.PaperSize
import java.io.File
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class ResumeEngineTest {

    private val engine = ResumeEngine(File("../templates"))
    private val source = File("../resumes/mariana.md")

    @Test
    fun `builds Mariana's real resume to one page in both papers`() {
        val out = File.createTempFile("outdir", "").let { it.delete(); it.mkdirs(); it }
        val results = engine.build(
            BuildRequest(
                source = source,
                templateId = "trazo",
                papers = listOf(PaperSize.A4, PaperSize.CARTA),
                profileName = null,
                outDir = out,
            )
        )
        assertEquals(2, results.size)
        results.forEach {
            assertTrue(it.pdf.exists())
            assertEquals(1, it.audit.pages)
            assertTrue(it.audit.fits)
        }
    }

    @Test
    fun `the produced pdf is parseable by an ATS`() {
        val out = File.createTempFile("ats", "").let { it.delete(); it.mkdirs(); it }
        val pdf = engine.build(
            BuildRequest(source, "trazo", listOf(PaperSize.A4), null, out)
        ).single().pdf

        val text = org.apache.pdfbox.Loader.loadPDF(pdf).use {
            org.apache.pdfbox.text.PDFTextStripper().getText(it)
        }

        listOf("Mariana Pacheco", "emeariaenea@gmail.com", "+58 424 221 6100",
               "Perfil", "Experiencia", "Certificaciones",
               "Java", "Spring Boot", "Kotlin", "Xetux").forEach {
            assertTrue(text.contains(it), "falta en el texto extraído: $it")
        }

        // Regression guard for the print media query: the date must sit on the
        // same extracted line as the role, not stacked underneath it.
        assertTrue(
            Regex("""Ingeniero de Software — Xetux Soluciones\s+feb\. 2025""").containsMatchIn(text)
        )
    }

    @Test
    fun `templates lists trazo`() {
        assertTrue(engine.templates().map { it.id }.contains("trazo"))
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `./gradlew :engine:test --tests '*ResumeEngineTest*'`
Expected: FAIL, "Unresolved reference: ResumeEngine".

- [ ] **Step 4: Write the facade**

`engine/src/main/kotlin/dev/resumekit/engine/ResumeEngine.kt`:

```kotlin
package dev.resumekit.engine

import dev.resumekit.engine.input.MarkdownSource
import dev.resumekit.engine.model.Profile
import dev.resumekit.engine.render.AuditResult
import dev.resumekit.engine.render.PageAudit
import dev.resumekit.engine.render.PaperSize
import dev.resumekit.engine.render.PdfRenderer
import dev.resumekit.engine.template.PebbleRenderer
import dev.resumekit.engine.template.TemplateCatalog
import dev.resumekit.engine.template.TemplateMeta
import java.io.File

data class BuildRequest(
    val source: File,
    val templateId: String,
    val papers: List<PaperSize>,
    val profileName: String? = null,
    val outDir: File,
    val autofit: Boolean = false,
)

data class BuildOutput(val pdf: File, val audit: AuditResult)

class OverflowException(val results: List<BuildOutput>) : RuntimeException(
    results.filter { !it.audit.fits }.joinToString("\n\n") { it.audit.diagnostic }
)

class ResumeEngine(templatesRoot: File) {

    private val catalog = TemplateCatalog(templatesRoot)
    private val markdown = MarkdownSource()
    private val pebble = PebbleRenderer()
    private val auditor = PageAudit()

    fun templates(): List<TemplateMeta> = catalog.list()

    fun build(request: BuildRequest): List<BuildOutput> {
        val meta = catalog.get(request.templateId)
        val full = markdown.parse(request.source)

        val profile = request.profileName?.let { name ->
            markdown.profiles(request.source)[name]
                ?: throw IllegalArgumentException(
                    "Perfil desconocido: '$name'. Disponibles: " +
                        markdown.profiles(request.source).keys.joinToString(", ").ifBlank { "(ninguno)" }
                )
        } ?: Profile(name = "por defecto")

        val resume = profile.apply(full)
        val stem = request.source.nameWithoutExtension

        val outputs = PdfRenderer().use { pdfRenderer ->
            request.papers.map { paper ->
                val html = pebble.render(meta, resume, paper)
                val suffix = if (paper == PaperSize.A4) "" else "-carta"
                val target = File(request.outDir, "$stem-${meta.id}$suffix.pdf")
                pdfRenderer.render(html, target)
                BuildOutput(target, auditor.audit(target, profile, resume))
            }
        }

        if (outputs.any { !it.audit.fits }) throw OverflowException(outputs)
        return outputs
    }
}
```

Autofit is declared in `BuildRequest` but not yet implemented; it is a later plan. Leaving the flag in the signature now avoids changing the facade later — the field is read by nothing, so no behaviour is implied.

- [ ] **Step 5: Run the test to verify it passes**

Run: `./gradlew :engine:test --tests '*ResumeEngineTest*'`
Expected: PASS, three tests.

- [ ] **Step 6: Write the CLI**

Add to `cli/build.gradle.kts`:

```kotlin
plugins { application }
application { mainClass.set("dev.resumekit.cli.MainKt") }
```

`cli/src/main/kotlin/dev/resumekit/cli/Main.kt`:

```kotlin
package dev.resumekit.cli

import com.github.ajalt.clikt.core.CliktCommand
import com.github.ajalt.clikt.core.PrintMessage
import com.github.ajalt.clikt.core.main
import com.github.ajalt.clikt.core.subcommands
import com.github.ajalt.clikt.parameters.arguments.argument
import com.github.ajalt.clikt.parameters.options.default
import com.github.ajalt.clikt.parameters.options.option
import com.github.ajalt.clikt.parameters.types.file
import dev.resumekit.engine.BuildRequest
import dev.resumekit.engine.OverflowException
import dev.resumekit.engine.ResumeEngine
import dev.resumekit.engine.render.PaperSize
import java.io.File
import kotlin.system.exitProcess

private fun engine() = ResumeEngine(File("templates"))

class Resume : CliktCommand(name = "resume") {
    override fun run() = Unit
}

class Build : CliktCommand(name = "build") {
    private val source by argument(help = "archivo .md del CV").file(mustExist = true)
    private val template by option("--template", help = "id de la plantilla").default("trazo")
    private val paper by option("--paper", help = "a4 | carta | ambos").default("a4")
    private val profile by option("--profile", help = "perfil de salida definido en el .md")
    private val out by option("--out").file().default(File("out"))

    override fun run() {
        val papers = when (paper.lowercase()) {
            "ambos" -> listOf(PaperSize.A4, PaperSize.CARTA)
            else -> listOf(PaperSize.parse(paper))
        }
        try {
            engine().build(BuildRequest(source, template, papers, profile, out)).forEach {
                echo("✓ ${it.pdf.path} — ${it.audit.pages} página(s)")
            }
        } catch (e: OverflowException) {
            echo(e.message, err = true)
            exitProcess(1)
        }
    }
}

class Templates : CliktCommand(name = "templates") {
    override fun run() {
        engine().templates().forEach { echo("${it.id.padEnd(12)} ${it.name} — ${it.description}") }
    }
}

fun main(args: Array<String>) = Resume().subcommands(Build(), Templates()).main(args)
```

- [ ] **Step 7: Run the CLI end to end**

Run: `./gradlew :cli:run --args="build resumes/mariana.md --template trazo --paper ambos"`
Expected: two `✓` lines, each reporting 1 page. Open `out/mariana-trazo.pdf` and confirm the dates sit on the same line as the role.

Run: `./gradlew :cli:run --args="templates"`
Expected: one line for `trazo`.

- [ ] **Step 8: Commit**

```bash
git add engine cli resumes
git commit -m "feat: resume build produces a one-page PDF in A4 and Carta"
```

---

## Self-review notes

**Spec coverage.** §3 architecture → Task 1. §4 model → Task 2. §5 md format → Tasks 3–4. §6 profiles and audit → Tasks 8–9. §7 template contract → Tasks 5–6. §8 CLI → Task 10. §11 errors → Tasks 3, 4, 5, 7, 9. §12 tests → every task.

**Deliberately deferred**, each needing its own plan because each is independently shippable:
- §9 LinkedIn ingestion (`resume import`) — needs `LinkedInPdfSource` and a `.md` writer.
- §10 Ktor preview server (`resume serve`) — the `server` module is created empty in Task 1.
- §6 `--autofit` — the flag exists in `BuildRequest`; the retry loop does not.

**Known sharp edges for the executor:**
- Task 7's first run downloads Chromium. Budget several minutes and a working network.
- Tests resolve `../templates` and `../resumes` relative to each module's project directory. If Gradle's test working directory differs on your machine, set `tasks.withType<Test> { workingDir = project.projectDir }`.
- The date-range regex in Task 4 accepts `–` (en dash) and `-` (hyphen). The fixtures use the en dash.
