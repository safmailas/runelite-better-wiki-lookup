# Better Wiki Lookup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a RuneLite Plugin Hub plugin that adds a minimap-area "lookup tool" toggle; while active, a left-click on any NPC / object / ground item / inventory item / interface widget opens the matching OSRS Wiki page in the browser, using a local Ollama model to resolve unknown interface text and remembering every resolution on disk.

**Architecture:** All RuneLite-coupled glue (event subscription, mouse listener, overlay, injection) lives in `BetterWikiLookupPlugin` and `WikiIconOverlay`. Everything else is a small, pure, unit-tested unit: `TargetExtractor` (click primitives → `LookupTarget`), `WikiUrls` (title/query → URL), `OllamaClient` (localhost HTTP → best-guess title), `ResolutionCache` (one JSON file, LRU, atomic writes), `Resolver` (cache → direct → AI → search pipeline), `LookupMode` (the single transient on/off flag). On a lookup, the plugin extracts a plain `LookupTarget` on the client thread, then hands resolution + browser-open to RuneLite's shared `ScheduledExecutorService`.

**Tech Stack:** Java 11, Gradle, RuneLite client (`net.runelite:client:latest.release`), Lombok, injected OkHttp + Gson, JUnit 4. Zero new runtime dependencies. Local AI via [Ollama](https://ollama.com) `/api/generate` (optional, off by default).

## Global Constraints

Every task's requirements implicitly include this section. Values are copied verbatim from `AGENTS.md` in the reference repo and the design spec.

- **Java 11 only.** `options.release.set(11)`. All code must be Java 11 compatible. CI additionally builds on JDK 21 to prove forward compatibility.
- **Zero new runtime dependencies.** Get HTTP and JSON via `@Inject OkHttpClient` and `@Inject Gson`. Never add `gson`, `guice`, `okhttp`, or other `runelite-client` transitives to `build.gradle`. Only test dependency is `junit:junit:4.12`.
- **Forbidden language features:** no reflection, no JNI/JNA, no `Unsafe`/LWJGL, no `Process`/`ProcessBuilder`, no dynamic class loading, no runtime code generation, no Java (de)serialization. `ResolutionCache` persists via Gson JSON, never `ObjectOutputStream`.
- **Disk I/O only under `RuneLite.RUNELITE_DIR/better-wiki-lookup/`.**
- **Threading:** never block network or disk I/O on the client thread; never `Thread.sleep()`; in `shutDown()` call `ScheduledFuture.cancel(false)` and `executor` is the injected shared one (do not shut it down); never `awaitTermination`.
- **Logging:** `log.debug()` for per-event/per-lookup logging; `log.info()` only for one-time startup/shutdown lines.
- **API usage:** open URLs with `net.runelite.client.util.LinkBrowser`, never `java.awt.Desktop`. Use `net.runelite.api.gameval` constants for any IDs — never hardcode magic numbers. Strip widget/menu markup with `net.runelite.client.util.Text`.
- **Config:** group name is the constant `"betterwikilookup"` — never rename without a migration. The `enableAi` item is **disabled by default** and carries a `warning=` string.
- **Menu rules:** the plugin must NOT add menu entries that send actions to the server. It only *consumes* an existing left-click via `MenuOptionClicked.consume()`. It must not inject input events.
- **Packaging:** BSD 2-Clause license with a header in every `.java` file. Rename everything from the template (package, classes, config group, `build.gradle` group, `settings.gradle` name, `runelite-plugin.properties`). Do NOT add a `META-INF/services/net.runelite.client.plugins.Plugin` file. Never commit build artifacts (`*.class`, `build/`, `out/`).
- **Testing philosophy:** pure logic classes only — no Mockito. RuneLite-coupled code (the `Plugin` subclass, the `Overlay`) is verified by the manual in-game matrix, not unit tests.
- **Completion:** you cannot verify in-game behaviour yourself and must never automate game input. After the code tasks, offer `./gradlew run` and wait for the user's in-game confirmation.

**Reference material:**
- Template / scaffold source: `~/runelite-ping-plugin` (same author, working Plugin Hub plugin).
- Design spec: `docs/superpowers/specs/2026-08-31-better-wiki-lookup-design.md`.

---

### Task 1: Repo scaffold, plugin skeleton, and config

**Files:**
- Create: `settings.gradle`
- Create: `build.gradle`
- Create: `runelite-plugin.properties`
- Create: `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`, `gradle/wrapper/gradle-wrapper.properties` (copied from template)
- Create: `LICENSE`
- Create: `checkstyle/checkstyle.xml`, `checkstyle/checkstyle-suppressions.xml` (copied from template if present, else minimal)
- Create: `.github/workflows/build.yml`
- Modify: `.gitignore`
- Create: `src/main/java/com/betterwikilookup/BetterWikiLookupPlugin.java`
- Create: `src/main/java/com/betterwikilookup/BetterWikiLookupConfig.java`
- Create: `src/test/java/com/betterwikilookup/BetterWikiLookupPluginTest.java`
- Create: `src/test/resources/logback-test.xml` (copied from template)

**Interfaces:**
- Consumes: nothing (first task).
- Produces:
  - `BetterWikiLookupConfig` interface with methods:
    `boolean deactivateAfterLookup()` (default `false`),
    `int maxCacheEntries()` (default `1000`),
    `boolean enableAi()` (default `false`),
    `String ollamaHost()` (default `"localhost"`),
    `int ollamaPort()` (default `11434`),
    `String ollamaModel()` (default `"llama3.2"`),
    and `String GROUP = "betterwikilookup"`.
  - `BetterWikiLookupPlugin extends net.runelite.client.plugins.Plugin` with a `@Provides BetterWikiLookupConfig` method.

- [ ] **Step 1: Copy the Gradle wrapper and static files from the template**

Run:
```bash
cd ~/runelite-better-wiki-lookup
cp ~/runelite-ping-plugin/gradlew ~/runelite-ping-plugin/gradlew.bat .
cp -R ~/runelite-ping-plugin/gradle .
mkdir -p src/test/resources
cp ~/runelite-ping-plugin/src/test/resources/logback-test.xml src/test/resources/logback-test.xml
cp -R ~/runelite-ping-plugin/checkstyle . 2>/dev/null || true
chmod +x gradlew
```
Expected: no errors; `gradle/wrapper/gradle-wrapper.jar` exists.

- [ ] **Step 2: Write `settings.gradle`**

```groovy
rootProject.name = 'better-wiki-lookup'
```

- [ ] **Step 3: Write `build.gradle`**

```groovy
plugins {
	id 'java'
}

repositories {
	mavenLocal()
	maven {
		url = 'https://repo.runelite.net'
		content {
			includeGroupByRegex("net\\.runelite.*")
		}
	}
	mavenCentral()
}

def runeLiteVersion = 'latest.release'
def pluginMainClass = 'com.betterwikilookup.BetterWikiLookupPluginTest'

dependencies {
	compileOnly group: 'net.runelite', name: 'client', version: runeLiteVersion

	compileOnly 'org.projectlombok:lombok:1.18.30'
	annotationProcessor 'org.projectlombok:lombok:1.18.30'

	testImplementation 'junit:junit:4.12'
	testImplementation group: 'net.runelite', name: 'client', version: runeLiteVersion
	testImplementation group: 'net.runelite', name: 'jshell', version: runeLiteVersion
}

group = 'com.betterwikilookup'

tasks.withType(JavaCompile).configureEach {
	options.encoding = 'UTF-8'
	options.release.set(11)
}

tasks.register('run', JavaExec) {
	classpath = sourceSets.test.runtimeClasspath
	mainClass = pluginMainClass
	jvmArgs "-ea"
	if (org.gradle.internal.os.OperatingSystem.current().isMacOsX()) {
		jvmArgs "--add-exports", "java.desktop/com.apple.eawt=ALL-UNNAMED",
			"--add-exports", "java.desktop/com.apple.eio=ALL-UNNAMED"
	}
	args "--developer-mode", "--debug"
}

tasks.register('shadowJar', Jar) {
	dependsOn configurations.testRuntimeClasspath
	manifest {
		attributes('Main-Class': pluginMainClass, 'Multi-Release': true)
	}
	duplicatesStrategy = DuplicatesStrategy.EXCLUDE
	from sourceSets.main.output
	from sourceSets.test.output
	from {
		configurations.testRuntimeClasspath.collect { file ->
			file.isDirectory() ? file : zipTree(file)
		}
	}
	exclude 'META-INF/INDEX.LIST'
	exclude 'META-INF/*.SF'
	exclude 'META-INF/*.DSA'
	exclude 'META-INF/*.RSA'
	exclude '**/module-info.class'
	group = BasePlugin.BUILD_GROUP
	archiveClassifier.set('shadow')
	archiveFileName.set("${rootProject.name}-${project.version}-all.jar")
}
```

- [ ] **Step 4: Write `runelite-plugin.properties`**

```properties
displayName=Better Wiki Lookup
author=safmailas
description=Toggle a lookup tool from the minimap, then click any entity or interface to open its OSRS Wiki page
tags=wiki,lookup,help,guide,search
plugins=com.betterwikilookup.BetterWikiLookupPlugin
version=
build=standard
```

- [ ] **Step 5: Write `LICENSE`**

Paste the standard BSD 2-Clause text with the first line:
```
Copyright (c) 2026, safmailas
```
(Use the exact text from https://opensource.org/license/bsd-2-clause — same wording as `~/runelite-ping-plugin/LICENSE`.)

- [ ] **Step 6: Extend `.gitignore`**

```gitignore
.gradle/
build/
out/
bin/
*.class
.idea/
*.iml
.DS_Store
```

- [ ] **Step 7: Write `.github/workflows/build.yml`**

```yaml
name: build
on:
  push:
  pull_request:
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java: [ 11, 21 ]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: ${{ matrix.java }}
      - name: Build
        run: ./gradlew build --stacktrace --console=plain
```

- [ ] **Step 8: Write `src/main/java/com/betterwikilookup/BetterWikiLookupConfig.java`**

```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import net.runelite.client.config.Config;
import net.runelite.client.config.ConfigGroup;
import net.runelite.client.config.ConfigItem;
import net.runelite.client.config.ConfigSection;
import net.runelite.client.config.Range;

@ConfigGroup(BetterWikiLookupConfig.GROUP)
public interface BetterWikiLookupConfig extends Config
{
	/**
	 * Config group name. Kept as a constant so it is never re-typed. Renaming this silently
	 * resets every user's saved settings, so it must never change without a migration.
	 */
	String GROUP = "betterwikilookup";

	String AI_SECTION = "ai";

	@ConfigItem(
		keyName = "deactivateAfterLookup",
		name = "One-shot lookup",
		description = "Turn the lookup tool off automatically after each lookup.",
		position = 0
	)
	default boolean deactivateAfterLookup()
	{
		return false;
	}

	@ConfigItem(
		keyName = "maxCacheEntries",
		name = "Remembered lookups",
		description = "How many resolved lookups to keep on disk before evicting the oldest.",
		position = 1
	)
	@Range(min = 50, max = 10000)
	default int maxCacheEntries()
	{
		return 1000;
	}

	@ConfigSection(
		name = "Local AI (Ollama)",
		description = "Optional: use a locally running Ollama model to resolve unknown interface text.",
		position = 2,
		closedByDefault = true
	)
	String aiSection = AI_SECTION;

	@ConfigItem(
		keyName = "enableAi",
		name = "Use local AI for unknown targets",
		description = "When on, unknown or ambiguous interface text is sent to a locally running Ollama "
			+ "instance to pick the best wiki page. When off, unknown targets open the wiki search page.",
		warning = "This sends the name or text of the thing you click to a local Ollama server "
			+ "(default http://localhost:11434). Nothing is sent if Ollama is not running.",
		section = AI_SECTION,
		position = 0
	)
	default boolean enableAi()
	{
		return false;
	}

	@ConfigItem(
		keyName = "ollamaHost",
		name = "Ollama host",
		description = "Host of the local Ollama server.",
		section = AI_SECTION,
		position = 1
	)
	default String ollamaHost()
	{
		return "localhost";
	}

	@ConfigItem(
		keyName = "ollamaPort",
		name = "Ollama port",
		description = "Port of the local Ollama server.",
		section = AI_SECTION,
		position = 2
	)
	@Range(min = 1, max = 65535)
	default int ollamaPort()
	{
		return 11434;
	}

	@ConfigItem(
		keyName = "ollamaModel",
		name = "Ollama model",
		description = "Model name to query. Must already be pulled in Ollama (e.g. run: ollama pull llama3.2).",
		section = AI_SECTION,
		position = 3
	)
	default String ollamaModel()
	{
		return "llama3.2";
	}
}
```

- [ ] **Step 9: Write `src/main/java/com/betterwikilookup/BetterWikiLookupPlugin.java`**

```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import com.google.inject.Provides;
import javax.inject.Inject;
import lombok.extern.slf4j.Slf4j;
import net.runelite.client.config.ConfigManager;
import net.runelite.client.plugins.Plugin;
import net.runelite.client.plugins.PluginDescriptor;

@Slf4j
@PluginDescriptor(
	name = "Better Wiki Lookup",
	description = "Toggle a lookup tool from the minimap, then click any entity or interface to open its OSRS Wiki page",
	tags = {"wiki", "lookup", "help", "guide", "search"}
)
public class BetterWikiLookupPlugin extends Plugin
{
	@Inject
	private BetterWikiLookupConfig config;

	@Override
	protected void startUp()
	{
		log.info("Better Wiki Lookup started");
	}

	@Override
	protected void shutDown()
	{
		log.info("Better Wiki Lookup stopped");
	}

	@Provides
	BetterWikiLookupConfig provideConfig(ConfigManager configManager)
	{
		return configManager.getConfig(BetterWikiLookupConfig.class);
	}
}
```

- [ ] **Step 10: Write `src/test/java/com/betterwikilookup/BetterWikiLookupPluginTest.java`**

```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import net.runelite.client.RuneLite;
import net.runelite.client.externalplugins.ExternalPluginManager;

public class BetterWikiLookupPluginTest
{
	@SuppressWarnings("unchecked")
	public static void main(String[] args) throws Exception
	{
		ExternalPluginManager.loadBuiltin(BetterWikiLookupPlugin.class);
		RuneLite.main(args);
	}
}
```

- [ ] **Step 11: Build**

Run: `./gradlew build --console=plain`
Expected: `BUILD SUCCESSFUL`. (First run downloads the RuneLite client jar; allow a few minutes.)

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "feat: scaffold Better Wiki Lookup plugin and config"
```

---

### Task 2: `WikiUrls` — title/query to wiki URL

**Files:**
- Create: `src/main/java/com/betterwikilookup/WikiUrls.java`
- Test: `src/test/java/com/betterwikilookup/WikiUrlsTest.java`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `WikiUrls.article(String pageTitle) -> String` — `https://oldschool.runescape.wiki/w/<Title_With_Underscores>`.
  - `WikiUrls.search(String query) -> String` — `https://oldschool.runescape.wiki/w/Special:Search?search=<encoded>&fulltext=1`.
  - `WikiUrls.BASE` = `"https://oldschool.runescape.wiki"` (package-private constant).

- [ ] **Step 1: Write the failing test**

`src/test/java/com/betterwikilookup/WikiUrlsTest.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import org.junit.Test;
import static org.junit.Assert.assertEquals;

public class WikiUrlsTest
{
	@Test
	public void articleReplacesSpacesWithUnderscores()
	{
		assertEquals("https://oldschool.runescape.wiki/w/Abyssal_whip", WikiUrls.article("Abyssal whip"));
	}

	@Test
	public void articleTrimsSurroundingWhitespace()
	{
		assertEquals("https://oldschool.runescape.wiki/w/Goblin", WikiUrls.article("  Goblin  "));
	}

	@Test
	public void articleKeepsWikiSafePunctuation()
	{
		assertEquals("https://oldschool.runescape.wiki/w/Vet'ion", WikiUrls.article("Vet'ion"));
		assertEquals("https://oldschool.runescape.wiki/w/Zulrah_(serpentine)", WikiUrls.article("Zulrah (serpentine)"));
	}

	@Test
	public void articlePercentEncodesUnsafeCharacters()
	{
		assertEquals("https://oldschool.runescape.wiki/w/A_%26_B", WikiUrls.article("A & B"));
	}

	@Test
	public void searchEncodesQueryWithPercentTwenty()
	{
		assertEquals(
			"https://oldschool.runescape.wiki/w/Special:Search?search=how%20to%20start%20dragon%20slayer&fulltext=1",
			WikiUrls.search("how to start dragon slayer"));
	}

	@Test
	public void searchEncodesAmpersandInQuery()
	{
		assertEquals(
			"https://oldschool.runescape.wiki/w/Special:Search?search=cats%20%26%20dogs&fulltext=1",
			WikiUrls.search("cats & dogs"));
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests 'com.betterwikilookup.WikiUrlsTest' --console=plain`
Expected: FAIL — `WikiUrls` does not exist / compilation error.

- [ ] **Step 3: Write minimal implementation**

`src/main/java/com/betterwikilookup/WikiUrls.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import java.io.UnsupportedEncodingException;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;

/**
 * Builds oldschool.runescape.wiki URLs. Pure functions, no state.
 */
public final class WikiUrls
{
	static final String BASE = "https://oldschool.runescape.wiki";

	private WikiUrls()
	{
	}

	/**
	 * Direct article URL. "Abyssal whip" becomes
	 * https://oldschool.runescape.wiki/w/Abyssal_whip . MediaWiki resolves redirects and
	 * capitalisation, and shows an inline search box for a missing title, so an imperfect
	 * title still lands somewhere useful.
	 */
	public static String article(String pageTitle)
	{
		String title = pageTitle.trim().replace(' ', '_');
		return BASE + "/w/" + encodePath(title);
	}

	/**
	 * Full-text search URL for a free-text query.
	 */
	public static String search(String query)
	{
		return BASE + "/w/Special:Search?search=" + encodeQuery(query.trim()) + "&fulltext=1";
	}

	private static String encodeQuery(String s)
	{
		try
		{
			return URLEncoder.encode(s, "UTF-8").replace("+", "%20");
		}
		catch (UnsupportedEncodingException e)
		{
			throw new AssertionError("UTF-8 always supported", e);
		}
	}

	// Percent-encode a path segment, but leave the characters MediaWiki itself leaves readable
	// in article URLs (underscore, parentheses, apostrophe, colon, comma, slash, dash, dot).
	private static String encodePath(String s)
	{
		StringBuilder out = new StringBuilder();
		for (byte rawByte : s.getBytes(StandardCharsets.UTF_8))
		{
			int b = rawByte & 0xff;
			char c = (char) b;
			boolean safe = (c >= 'A' && c <= 'Z') || (c >= 'a' && c <= 'z') || (c >= '0' && c <= '9')
				|| c == '_' || c == '-' || c == '.' || c == '/' || c == ':'
				|| c == '(' || c == ')' || c == ',' || c == '\'';
			if (safe)
			{
				out.append(c);
			}
			else
			{
				out.append('%').append(String.format("%02X", b));
			}
		}
		return out.toString();
	}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew test --tests 'com.betterwikilookup.WikiUrlsTest' --console=plain`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/betterwikilookup/WikiUrls.java src/test/java/com/betterwikilookup/WikiUrlsTest.java
git commit -m "feat: add WikiUrls for article and search URL building"
```

---

### Task 3: `ResolutionCache` — the one persistent store

**Files:**
- Create: `src/main/java/com/betterwikilookup/CacheEntry.java`
- Create: `src/main/java/com/betterwikilookup/ResolutionCache.java`
- Test: `src/test/java/com/betterwikilookup/ResolutionCacheTest.java`

**Interfaces:**
- Consumes: nothing (Gson supplied by caller; tests use `new com.google.gson.Gson()`).
- Produces:
  - `CacheEntry` value type: `String pageTitle` (nullable), `String url`, `String summary` (nullable), `String source`, `long savedAtEpochMs`; Lombok `@Value`.
  - `ResolutionCache(java.nio.file.Path dir, com.google.gson.Gson gson, int maxEntries)` — creates `dir`, loads `dir/resolutions.json` if present.
  - `Optional<CacheEntry> get(String key)`
  - `void put(String key, CacheEntry entry)` — marks dirty, applies LRU eviction.
  - `void flush()` — atomically writes to disk if dirty; safe to call repeatedly and on shutdown.

- [ ] **Step 1: Write the failing test**

`src/test/java/com/betterwikilookup/ResolutionCacheTest.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import com.google.gson.Gson;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Optional;
import org.junit.Rule;
import org.junit.Test;
import org.junit.rules.TemporaryFolder;
import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertTrue;

public class ResolutionCacheTest
{
	@Rule
	public TemporaryFolder tmp = new TemporaryFolder();

	private final Gson gson = new Gson();

	private CacheEntry entry(String title)
	{
		return new CacheEntry(title, "https://oldschool.runescape.wiki/w/" + title, "summary of " + title, "AI", 1L);
	}

	@Test
	public void persistsAndReloadsAcrossInstances() throws IOException
	{
		Path dir = tmp.getRoot().toPath();
		ResolutionCache a = new ResolutionCache(dir, gson, 100);
		a.put("npc|goblin", entry("Goblin"));
		a.flush();

		ResolutionCache b = new ResolutionCache(dir, gson, 100);
		Optional<CacheEntry> got = b.get("npc|goblin");
		assertTrue(got.isPresent());
		assertEquals("Goblin", got.get().getPageTitle());
	}

	@Test
	public void evictsLeastRecentlyUsedBeyondCapacity() throws IOException
	{
		ResolutionCache c = new ResolutionCache(tmp.getRoot().toPath(), gson, 2);
		c.put("k|a", entry("A"));
		c.put("k|b", entry("B"));
		c.get("k|a");                 // touch A so B is now least-recently-used
		c.put("k|c", entry("C"));     // evicts B

		assertTrue(c.get("k|a").isPresent());
		assertFalse(c.get("k|b").isPresent());
		assertTrue(c.get("k|c").isPresent());
	}

	@Test
	public void recoversFromCorruptFileAndKeepsBackup() throws IOException
	{
		Path dir = tmp.getRoot().toPath();
		Files.write(dir.resolve("resolutions.json"), "{ this is not json".getBytes(StandardCharsets.UTF_8));

		ResolutionCache c = new ResolutionCache(dir, gson, 100);

		assertFalse(c.get("anything").isPresent());
		assertTrue(Files.exists(dir.resolve("resolutions.json.bak")));
	}

	@Test
	public void flushWithoutChangesDoesNotCreateFile() throws IOException
	{
		Path dir = tmp.getRoot().toPath();
		ResolutionCache c = new ResolutionCache(dir, gson, 100);
		c.flush();
		assertFalse(Files.exists(dir.resolve("resolutions.json")));
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests 'com.betterwikilookup.ResolutionCacheTest' --console=plain`
Expected: FAIL — `CacheEntry` / `ResolutionCache` do not exist.

- [ ] **Step 3: Write minimal implementation**

`src/main/java/com/betterwikilookup/CacheEntry.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import javax.annotation.Nullable;
import lombok.Value;

/**
 * One remembered resolution. Serialised to JSON by {@link ResolutionCache}. Never use Java
 * serialisation for this type.
 */
@Value
class CacheEntry
{
	@Nullable
	String pageTitle;

	String url;

	@Nullable
	String summary;

	/** {@link ResolutionSource} name at the time it was resolved. */
	String source;

	long savedAtEpochMs;
}
```

`src/main/java/com/betterwikilookup/ResolutionCache.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import com.google.gson.Gson;
import com.google.gson.reflect.TypeToken;
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.IOException;
import java.lang.reflect.Type;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Optional;
import lombok.extern.slf4j.Slf4j;

/**
 * The only persistent state in the plugin: a bounded, least-recently-used map of
 * normalised-key -> {@link CacheEntry}, stored as a single JSON file under
 * {@code RUNELITE_DIR/better-wiki-lookup/resolutions.json}.
 *
 * <p>All disk access lives here. Every method is synchronised; callers may use it from the
 * client thread ({@link #get}) and a background executor ({@link #put}, {@link #flush}).
 */
@Slf4j
class ResolutionCache
{
	private static final String FILE_NAME = "resolutions.json";

	private final Path file;
	private final Path backup;
	private final Gson gson;
	private final int maxEntries;
	private final LinkedHashMap<String, CacheEntry> map;

	private boolean dirty;

	ResolutionCache(Path dir, Gson gson, int maxEntries) throws IOException
	{
		this.file = dir.resolve(FILE_NAME);
		this.backup = dir.resolve(FILE_NAME + ".bak");
		this.gson = gson;
		this.maxEntries = Math.max(1, maxEntries);
		this.map = new LinkedHashMap<String, CacheEntry>(16, 0.75f, true)
		{
			@Override
			protected boolean removeEldestEntry(Map.Entry<String, CacheEntry> eldest)
			{
				return size() > ResolutionCache.this.maxEntries;
			}
		};
		Files.createDirectories(dir);
		load();
	}

	synchronized Optional<CacheEntry> get(String key)
	{
		return Optional.ofNullable(map.get(key));
	}

	synchronized void put(String key, CacheEntry entry)
	{
		map.put(key, entry);
		dirty = true;
	}

	synchronized void flush()
	{
		if (!dirty)
		{
			return;
		}
		try
		{
			Path tmp = Files.createTempFile(file.getParent(), "resolutions", ".tmp");
			try (BufferedWriter w = Files.newBufferedWriter(tmp, StandardCharsets.UTF_8))
			{
				gson.toJson(map, w);
			}
			try
			{
				Files.move(tmp, file, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.ATOMIC_MOVE);
			}
			catch (IOException atomicUnsupported)
			{
				Files.move(tmp, file, StandardCopyOption.REPLACE_EXISTING);
			}
			dirty = false;
		}
		catch (IOException e)
		{
			log.debug("Could not write wiki lookup cache", e);
		}
	}

	private void load()
	{
		if (!Files.exists(file))
		{
			return;
		}
		try (BufferedReader r = Files.newBufferedReader(file, StandardCharsets.UTF_8))
		{
			Type type = new TypeToken<LinkedHashMap<String, CacheEntry>>()
			{
			}.getType();
			LinkedHashMap<String, CacheEntry> loaded = gson.fromJson(r, type);
			if (loaded != null)
			{
				loaded.forEach(map::put);
			}
		}
		catch (IOException | RuntimeException e)
		{
			log.warn("Wiki lookup cache was unreadable; starting fresh", e);
			try
			{
				Files.move(file, backup, StandardCopyOption.REPLACE_EXISTING);
			}
			catch (IOException moveFailed)
			{
				log.debug("Could not back up corrupt cache", moveFailed);
			}
		}
	}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew test --tests 'com.betterwikilookup.ResolutionCacheTest' --console=plain`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/betterwikilookup/CacheEntry.java src/main/java/com/betterwikilookup/ResolutionCache.java src/test/java/com/betterwikilookup/ResolutionCacheTest.java
git commit -m "feat: add ResolutionCache disk-backed LRU store"
```

---

### Task 4: `OllamaClient` — localhost model call

**Files:**
- Create: `src/main/java/com/betterwikilookup/OllamaResult.java`
- Create: `src/main/java/com/betterwikilookup/WikiPageGuesser.java`
- Create: `src/main/java/com/betterwikilookup/OllamaClient.java`
- Test: `src/test/java/com/betterwikilookup/OllamaClientTest.java`

**Interfaces:**
- Consumes: `com.google.gson.Gson`, `okhttp3.OkHttpClient` (both `@Inject`ed by the plugin; tests build a real `OkHttpClient` and a `new Gson()`).
- Produces:
  - `OllamaResult` value type: `String title`, `String summary`, `double confidence`; Lombok `@Value`.
  - `interface WikiPageGuesser { @Nullable OllamaResult guess(String rawText, String targetType); boolean available(); }`
  - `OllamaClient implements WikiPageGuesser`, constructed as
    `new OllamaClient(OkHttpClient http, Gson gson, java.util.function.Supplier<String> baseUrl, java.util.function.Supplier<String> model)`.
    `baseUrl` returns e.g. `"http://localhost:11434"`. `guess(...)` and `available()` are blocking — callers must run them off the client thread. Any failure returns `null` / `false`; never throws.

- [ ] **Step 1: Write the failing test**

`src/test/java/com/betterwikilookup/OllamaClientTest.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import com.google.gson.Gson;
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.util.function.Supplier;
import okhttp3.OkHttpClient;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertNull;
import static org.junit.Assert.assertTrue;

public class OllamaClientTest
{
	private HttpServer server;
	private String baseUrl;
	private volatile String responseBody = "";
	private volatile int responseCode = 200;

	@Before
	public void setUp() throws IOException
	{
		server = HttpServer.create(new InetSocketAddress("127.0.0.1", 0), 0);
		server.createContext("/", exchange ->
		{
			byte[] body = responseBody.getBytes(StandardCharsets.UTF_8);
			exchange.sendResponseHeaders(responseCode, body.length == 0 ? -1 : body.length);
			try (OutputStream os = exchange.getResponseBody())
			{
				os.write(body);
			}
		});
		server.start();
		baseUrl = "http://127.0.0.1:" + server.getAddress().getPort();
	}

	@After
	public void tearDown()
	{
		server.stop(0);
	}

	private OllamaClient client()
	{
		Supplier<String> url = () -> baseUrl;
		Supplier<String> model = () -> "llama3.2";
		return new OllamaClient(new OkHttpClient(), new Gson(), url, model);
	}

	@Test
	public void parsesWellFormedResponse()
	{
		responseBody = "{\"response\":\"{\\\"title\\\":\\\"Abyssal whip\\\",\\\"summary\\\":\\\"A weapon.\\\",\\\"confidence\\\":0.9}\"}";
		OllamaResult r = client().guess("whip", "ITEM");
		assertNotNull(r);
		assertEquals("Abyssal whip", r.getTitle());
		assertEquals("A weapon.", r.getSummary());
		assertEquals(0.9, r.getConfidence(), 0.001);
	}

	@Test
	public void returnsNullOnServerError()
	{
		responseCode = 500;
		responseBody = "boom";
		assertNull(client().guess("whip", "ITEM"));
	}

	@Test
	public void returnsNullWhenInnerJsonIsGarbage()
	{
		responseBody = "{\"response\":\"not json at all\"}";
		assertNull(client().guess("whip", "ITEM"));
	}

	@Test
	public void returnsNullWhenTitleMissing()
	{
		responseBody = "{\"response\":\"{\\\"summary\\\":\\\"x\\\"}\"}";
		assertNull(client().guess("whip", "ITEM"));
	}

	@Test
	public void availableIsTrueWhenServerResponds()
	{
		responseBody = "{}";
		assertTrue(client().available());
	}

	@Test
	public void availableIsFalseWhenNothingListening()
	{
		Supplier<String> deadUrl = () -> "http://127.0.0.1:1";
		OllamaClient dead = new OllamaClient(new OkHttpClient(), new Gson(), deadUrl, () -> "llama3.2");
		assertFalse(dead.available());
		assertNull(dead.guess("whip", "ITEM"));
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests 'com.betterwikilookup.OllamaClientTest' --console=plain`
Expected: FAIL — `OllamaClient` / `OllamaResult` / `WikiPageGuesser` do not exist.

- [ ] **Step 3: Write minimal implementation**

`src/main/java/com/betterwikilookup/OllamaResult.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import lombok.Value;

@Value
class OllamaResult
{
	String title;
	String summary;
	double confidence;
}
```

`src/main/java/com/betterwikilookup/WikiPageGuesser.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import javax.annotation.Nullable;

/**
 * Something that can guess the best wiki article title for a piece of game text.
 * Implementations are blocking and must be called off the client thread.
 */
interface WikiPageGuesser
{
	/**
	 * @return the best-guess article title plus a short summary, or {@code null} if the guesser
	 * is unavailable or produced nothing usable. Never throws.
	 */
	@Nullable
	OllamaResult guess(String rawText, String targetType);

	/**
	 * @return true if the backing service currently answers. Never throws.
	 */
	boolean available();
}
```

`src/main/java/com/betterwikilookup/OllamaClient.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import com.google.gson.Gson;
import com.google.gson.JsonObject;
import java.io.IOException;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;
import javax.annotation.Nullable;
import lombok.extern.slf4j.Slf4j;
import okhttp3.MediaType;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;

/**
 * Talks to a locally running Ollama instance via {@code POST /api/generate}. Blocking; call
 * from a background executor, never the client thread. Every failure mode returns {@code null}
 * / {@code false} rather than throwing, so the caller can fall through to wiki search.
 */
@Slf4j
class OllamaClient implements WikiPageGuesser
{
	private static final MediaType JSON = MediaType.get("application/json");
	private static final long READ_TIMEOUT_MS = 2500L;
	private static final long CONNECT_TIMEOUT_MS = 1000L;

	private final OkHttpClient http;
	private final Gson gson;
	private final Supplier<String> baseUrl;
	private final Supplier<String> model;

	OllamaClient(OkHttpClient http, Gson gson, Supplier<String> baseUrl, Supplier<String> model)
	{
		this.http = http.newBuilder()
			.connectTimeout(CONNECT_TIMEOUT_MS, TimeUnit.MILLISECONDS)
			.readTimeout(READ_TIMEOUT_MS, TimeUnit.MILLISECONDS)
			.callTimeout(READ_TIMEOUT_MS + CONNECT_TIMEOUT_MS + 500L, TimeUnit.MILLISECONDS)
			.build();
		this.gson = gson;
		this.baseUrl = baseUrl;
		this.model = model;
	}

	@Nullable
	@Override
	public OllamaResult guess(String rawText, String targetType)
	{
		JsonObject body = new JsonObject();
		body.addProperty("model", model.get());
		body.addProperty("prompt", buildPrompt(rawText, targetType));
		body.addProperty("stream", false);
		body.addProperty("format", "json");

		Request request = new Request.Builder()
			.url(baseUrl.get() + "/api/generate")
			.post(RequestBody.create(JSON, gson.toJson(body)))
			.build();

		try (Response response = http.newCall(request).execute())
		{
			if (!response.isSuccessful() || response.body() == null)
			{
				return null;
			}
			JsonObject outer = gson.fromJson(response.body().string(), JsonObject.class);
			if (outer == null || !outer.has("response"))
			{
				return null;
			}
			JsonObject inner = gson.fromJson(outer.get("response").getAsString(), JsonObject.class);
			if (inner == null || !inner.has("title"))
			{
				return null;
			}
			String title = inner.get("title").getAsString().trim();
			if (title.isEmpty())
			{
				return null;
			}
			String summary = inner.has("summary") ? inner.get("summary").getAsString().trim() : "";
			double confidence = inner.has("confidence") ? asDouble(inner.get("confidence").getAsString()) : 0.0;
			return new OllamaResult(title, summary, confidence);
		}
		catch (IOException | RuntimeException e)
		{
			log.debug("Ollama lookup failed for '{}'", rawText, e);
			return null;
		}
	}

	@Override
	public boolean available()
	{
		Request request = new Request.Builder().url(baseUrl.get() + "/api/tags").get().build();
		try (Response response = http.newCall(request).execute())
		{
			return response.isSuccessful();
		}
		catch (IOException e)
		{
			return false;
		}
	}

	private static double asDouble(String s)
	{
		try
		{
			return Double.parseDouble(s);
		}
		catch (NumberFormatException e)
		{
			return 0.0;
		}
	}

	private static String buildPrompt(String rawText, String targetType)
	{
		return "You map Old School RuneScape game content to the single best "
			+ "oldschool.runescape.wiki article title. The player clicked a " + targetType
			+ " labelled \"" + rawText + "\". "
			+ "Reply ONLY with JSON of the form "
			+ "{\"title\":\"<exact wiki article title>\",\"summary\":\"<40 words or fewer>\",\"confidence\":<0 to 1>}. "
			+ "If unsure, give your best guess with a low confidence.";
	}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew test --tests 'com.betterwikilookup.OllamaClientTest' --console=plain`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/betterwikilookup/OllamaResult.java src/main/java/com/betterwikilookup/WikiPageGuesser.java src/main/java/com/betterwikilookup/OllamaClient.java src/test/java/com/betterwikilookup/OllamaClientTest.java
git commit -m "feat: add OllamaClient localhost page guesser"
```

---

### Task 5: `TargetExtractor` — click primitives to `LookupTarget`

**Files:**
- Create: `src/main/java/com/betterwikilookup/TargetType.java`
- Create: `src/main/java/com/betterwikilookup/LookupTarget.java`
- Create: `src/main/java/com/betterwikilookup/TargetExtractor.java`
- Test: `src/test/java/com/betterwikilookup/TargetExtractorTest.java`

**Interfaces:**
- Consumes: `net.runelite.api.MenuAction` (enum — used directly, no mock needed).
- Produces:
  - `enum TargetType { NPC, GAME_OBJECT, GROUND_ITEM, ITEM, WIDGET_TEXT }`
  - `LookupTarget` value type: `TargetType type`, `String rawText`, `int id` (game id, or `-1`); Lombok `@Value`.
  - Static method:
    `Optional<LookupTarget> TargetExtractor.extract(MenuAction action, String option, String target, int identifier, int itemId, String widgetText, java.util.function.IntFunction<String> itemName)`
    - `itemName` maps an item id to its name; returns `null` if unknown. The plugin passes `id -> itemManager.getItemComposition(id).getName()`; tests pass a lambda.
    - Returns `Optional.empty()` when the click is not something worth looking up (walk-here, blank text).

- [ ] **Step 1: Write the failing test**

`src/test/java/com/betterwikilookup/TargetExtractorTest.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import java.util.Optional;
import java.util.function.IntFunction;
import net.runelite.api.MenuAction;
import org.junit.Test;
import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertTrue;

public class TargetExtractorTest
{
	private static final IntFunction<String> NO_ITEMS = id -> null;

	private static LookupTarget extract(MenuAction action, String option, String target, int id, int itemId, String widgetText)
	{
		Optional<LookupTarget> t = TargetExtractor.extract(action, option, target, id, itemId, widgetText, NO_ITEMS);
		assertTrue("expected a target", t.isPresent());
		return t.get();
	}

	@Test
	public void npcStripsColourTagsAndCombatLevel()
	{
		LookupTarget t = extract(MenuAction.NPC_SECOND_OPTION, "Attack", "<col=ffff00>Goblin</col>  (level-2)", 3242, -1, null);
		assertEquals(TargetType.NPC, t.getType());
		assertEquals("Goblin", t.getRawText());
		assertEquals(3242, t.getId());
	}

	@Test
	public void examineNpcIsAnNpcTarget()
	{
		LookupTarget t = extract(MenuAction.EXAMINE_NPC, "Examine", "Cow (level-2)", 1, -1, null);
		assertEquals(TargetType.NPC, t.getType());
		assertEquals("Cow", t.getRawText());
	}

	@Test
	public void gameObjectClassifiesAsObject()
	{
		LookupTarget t = extract(MenuAction.GAME_OBJECT_FIRST_OPTION, "Bank", "<col=00ffff>Bank booth</col>", 10355, -1, null);
		assertEquals(TargetType.GAME_OBJECT, t.getType());
		assertEquals("Bank booth", t.getRawText());
	}

	@Test
	public void groundItemClassifiesAsGroundItem()
	{
		LookupTarget t = extract(MenuAction.GROUND_ITEM_THIRD_OPTION, "Take", "<col=ff9040>Coins</col>", 995, -1, null);
		assertEquals(TargetType.GROUND_ITEM, t.getType());
		assertEquals("Coins", t.getRawText());
	}

	@Test
	public void inventoryItemPrefersItemManagerName()
	{
		IntFunction<String> items = id -> id == 4151 ? "Abyssal whip" : null;
		Optional<LookupTarget> t = TargetExtractor.extract(
			MenuAction.CC_OP, "Wield", "<col=ff9040>Abyssal whip</col>", 2, 4151, null, items);
		assertTrue(t.isPresent());
		assertEquals(TargetType.ITEM, t.get().getType());
		assertEquals("Abyssal whip", t.get().getRawText());
		assertEquals(4151, t.get().getId());
	}

	@Test
	public void widgetTextIsUsedWhenPresentAndNoEntityId()
	{
		LookupTarget t = extract(MenuAction.WIDGET_FIRST_OPTION, "Select", "", -1, -1, "<col=ff981f>Attack style</col>");
		assertEquals(TargetType.WIDGET_TEXT, t.getType());
		assertEquals("Attack style", t.getRawText());
		assertEquals(-1, t.getId());
	}

	@Test
	public void widgetFallsBackToTargetThenOption()
	{
		LookupTarget t = extract(MenuAction.WIDGET_FIRST_OPTION, "Prayers", "", -1, -1, "   ");
		assertEquals(TargetType.WIDGET_TEXT, t.getType());
		assertEquals("Prayers", t.getRawText());
	}

	@Test
	public void walkHereProducesNothing()
	{
		Optional<LookupTarget> t = TargetExtractor.extract(MenuAction.WALK, "Walk here", "", -1, -1, null, NO_ITEMS);
		assertFalse(t.isPresent());
	}

	@Test
	public void blankEverythingProducesNothing()
	{
		Optional<LookupTarget> t = TargetExtractor.extract(MenuAction.WIDGET_FIRST_OPTION, "", "", -1, -1, "  ", NO_ITEMS);
		assertFalse(t.isPresent());
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests 'com.betterwikilookup.TargetExtractorTest' --console=plain`
Expected: FAIL — types do not exist.

- [ ] **Step 3: Write minimal implementation**

`src/main/java/com/betterwikilookup/TargetType.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

enum TargetType
{
	NPC,
	GAME_OBJECT,
	GROUND_ITEM,
	ITEM,
	WIDGET_TEXT
}
```

`src/main/java/com/betterwikilookup/LookupTarget.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import lombok.Value;

/**
 * A cleaned, plain description of the thing the player clicked. Carries no RuneLite types so it
 * can cross from the client thread to a background executor freely.
 */
@Value
class LookupTarget
{
	TargetType type;

	/** Display text with markup and combat level removed. Never blank. */
	String rawText;

	/** Game id (NPC id, object id, item id) where meaningful, otherwise -1. */
	int id;
}
```

`src/main/java/com/betterwikilookup/TargetExtractor.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import java.util.Optional;
import java.util.function.IntFunction;
import java.util.regex.Pattern;
import javax.annotation.Nullable;
import net.runelite.api.MenuAction;
import net.runelite.client.util.Text;

/**
 * Turns the primitive fields of a clicked menu entry into a {@link LookupTarget}. Pure and
 * static: the plugin pulls the primitives off the RuneLite event on the client thread and calls
 * this; there is nothing here that needs a live client to test.
 */
final class TargetExtractor
{
	// Trailing "(level-92)" / "(lvl-92)" / "(level: 92)" produced by NPC menu targets.
	private static final Pattern COMBAT_LEVEL = Pattern.compile("\\s*\\((?:level|lvl)[-:]?\\s*\\d+\\)\\s*$",
		Pattern.CASE_INSENSITIVE);

	private static final Pattern WHITESPACE = Pattern.compile("\\s+");

	private TargetExtractor()
	{
	}

	static Optional<LookupTarget> extract(
		MenuAction action,
		String option,
		String target,
		int identifier,
		int itemId,
		@Nullable String widgetText,
		IntFunction<String> itemName)
	{
		switch (action)
		{
			case NPC_FIRST_OPTION:
			case NPC_SECOND_OPTION:
			case NPC_THIRD_OPTION:
			case NPC_FOURTH_OPTION:
			case NPC_FIFTH_OPTION:
			case EXAMINE_NPC:
			case WIDGET_TARGET_ON_NPC:
				return named(TargetType.NPC, target, identifier);

			case GAME_OBJECT_FIRST_OPTION:
			case GAME_OBJECT_SECOND_OPTION:
			case GAME_OBJECT_THIRD_OPTION:
			case GAME_OBJECT_FOURTH_OPTION:
			case GAME_OBJECT_FIFTH_OPTION:
			case EXAMINE_OBJECT:
			case WIDGET_TARGET_ON_GAME_OBJECT:
				return named(TargetType.GAME_OBJECT, target, identifier);

			case GROUND_ITEM_FIRST_OPTION:
			case GROUND_ITEM_SECOND_OPTION:
			case GROUND_ITEM_THIRD_OPTION:
			case GROUND_ITEM_FOURTH_OPTION:
			case GROUND_ITEM_FIFTH_OPTION:
			case EXAMINE_ITEM_GROUND:
			case WIDGET_TARGET_ON_GROUND_ITEM:
				return named(TargetType.GROUND_ITEM, target, identifier);

			case CC_OP:
			case CC_OP_LOW_PRIORITY:
			case WIDGET_TARGET_ON_WIDGET:
			case WIDGET_FIRST_OPTION:
			case WIDGET_SECOND_OPTION:
			case WIDGET_THIRD_OPTION:
			case WIDGET_FOURTH_OPTION:
			case WIDGET_FIFTH_OPTION:
				return widgetOrItem(target, option, identifier, itemId, widgetText, itemName);

			default:
				return Optional.empty();
		}
	}

	private static Optional<LookupTarget> named(TargetType type, String target, int identifier)
	{
		String cleaned = COMBAT_LEVEL.matcher(clean(target)).replaceAll("");
		cleaned = cleaned.trim();
		if (cleaned.isEmpty())
		{
			return Optional.empty();
		}
		return Optional.of(new LookupTarget(type, cleaned, identifier));
	}

	private static Optional<LookupTarget> widgetOrItem(
		String target,
		String option,
		int identifier,
		int itemId,
		@Nullable String widgetText,
		IntFunction<String> itemName)
	{
		if (itemId >= 0)
		{
			String name = itemName.apply(itemId);
			if (name != null && !name.isEmpty() && !"null".equalsIgnoreCase(name))
			{
				return Optional.of(new LookupTarget(TargetType.ITEM, clean(name), itemId));
			}
			String fromTarget = clean(target);
			if (!fromTarget.isEmpty())
			{
				return Optional.of(new LookupTarget(TargetType.ITEM, fromTarget, itemId));
			}
		}

		for (String candidate : new String[]{widgetText, target, option})
		{
			String text = clean(candidate);
			if (!text.isEmpty())
			{
				return Optional.of(new LookupTarget(TargetType.WIDGET_TEXT, text, -1));
			}
		}
		return Optional.empty();
	}

	private static String clean(@Nullable String s)
	{
		if (s == null)
		{
			return "";
		}
		String noTags = Text.removeTags(s).replace(' ', ' ');
		return WHITESPACE.matcher(noTags).replaceAll(" ").trim();
	}
}
```

> **Note for the implementer:** the `MenuAction` constants used here (`NPC_*_OPTION`, `GAME_OBJECT_*_OPTION`, `GROUND_ITEM_*_OPTION`, `EXAMINE_*`, `WIDGET_*_OPTION`, `WIDGET_TARGET_ON_*`, `CC_OP`, `CC_OP_LOW_PRIORITY`, `WALK`) are all long-standing in `net.runelite.api.MenuAction`. If the pinned `latest.release` renamed or removed one, the compiler will point at it — replace it with the current equivalent and adjust the corresponding test constant. Do not switch to numeric IDs.

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew test --tests 'com.betterwikilookup.TargetExtractorTest' --console=plain`
Expected: PASS (9 tests).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/betterwikilookup/TargetType.java src/main/java/com/betterwikilookup/LookupTarget.java src/main/java/com/betterwikilookup/TargetExtractor.java src/test/java/com/betterwikilookup/TargetExtractorTest.java
git commit -m "feat: add TargetExtractor for click classification"
```

---

### Task 6: `Resolver` — cache to direct to AI to search

**Files:**
- Create: `src/main/java/com/betterwikilookup/ResolutionSource.java`
- Create: `src/main/java/com/betterwikilookup/ResolvedPage.java`
- Create: `src/main/java/com/betterwikilookup/Resolver.java`
- Test: `src/test/java/com/betterwikilookup/ResolverTest.java`

**Interfaces:**
- Consumes: `ResolutionCache` (Task 3), `WikiPageGuesser` (Task 4), `TargetType` / `LookupTarget` (Task 5), `WikiUrls` (Task 2), `CacheEntry` (Task 3).
- Produces:
  - `enum ResolutionSource { CACHE, DIRECT, AI, SEARCH }`
  - `ResolvedPage` value type: `String url`, `String pageTitle` (nullable), `String summary` (nullable), `ResolutionSource source`; Lombok `@Value`.
  - `Resolver(ResolutionCache cache, WikiPageGuesser guesser, java.util.function.BooleanSupplier aiEnabled)`
  - `ResolvedPage Resolver.resolve(LookupTarget target)` — blocking (may call the guesser); writes fresh resolutions back to the cache. Call off the client thread.
  - `static String Resolver.cacheKey(LookupTarget target)` — `"<TYPE>|<lowercased, whitespace-collapsed rawText>"`.

- [ ] **Step 1: Write the failing test**

`src/test/java/com/betterwikilookup/ResolverTest.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import com.google.gson.Gson;
import java.io.IOException;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.BooleanSupplier;
import javax.annotation.Nullable;
import org.junit.Before;
import org.junit.Rule;
import org.junit.Test;
import org.junit.rules.TemporaryFolder;
import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNull;

public class ResolverTest
{
	@Rule
	public TemporaryFolder tmp = new TemporaryFolder();

	private ResolutionCache cache;

	@Before
	public void setUp() throws IOException
	{
		cache = new ResolutionCache(tmp.getRoot().toPath(), new Gson(), 100);
	}

	private static final class StubGuesser implements WikiPageGuesser
	{
		@Nullable
		OllamaResult next;
		final AtomicInteger calls = new AtomicInteger();

		@Nullable
		@Override
		public OllamaResult guess(String rawText, String targetType)
		{
			calls.incrementAndGet();
			return next;
		}

		@Override
		public boolean available()
		{
			return true;
		}
	}

	@Test
	public void namedEntityResolvesDirectAndIsCached()
	{
		StubGuesser guesser = new StubGuesser();
		Resolver resolver = new Resolver(cache, guesser, () -> true);

		ResolvedPage page = resolver.resolve(new LookupTarget(TargetType.NPC, "Goblin", 3242));

		assertEquals(ResolutionSource.DIRECT, page.getSource());
		assertEquals("https://oldschool.runescape.wiki/w/Goblin", page.getUrl());
		assertEquals(0, guesser.calls.get());

		ResolvedPage again = resolver.resolve(new LookupTarget(TargetType.NPC, "Goblin", 3242));
		assertEquals(ResolutionSource.CACHE, again.getSource());
	}

	@Test
	public void widgetTextWithAiDisabledFallsBackToSearch()
	{
		Resolver resolver = new Resolver(cache, new StubGuesser(), () -> false);

		ResolvedPage page = resolver.resolve(new LookupTarget(TargetType.WIDGET_TEXT, "Attack style", -1));

		assertEquals(ResolutionSource.SEARCH, page.getSource());
		assertEquals("https://oldschool.runescape.wiki/w/Special:Search?search=Attack%20style&fulltext=1", page.getUrl());
	}

	@Test
	public void widgetTextWithAiUsesGuesserResult()
	{
		StubGuesser guesser = new StubGuesser();
		guesser.next = new OllamaResult("Attack style", "Governs which stats you train.", 0.8);
		Resolver resolver = new Resolver(cache, guesser, () -> true);

		ResolvedPage page = resolver.resolve(new LookupTarget(TargetType.WIDGET_TEXT, "attack style", -1));

		assertEquals(ResolutionSource.AI, page.getSource());
		assertEquals("https://oldschool.runescape.wiki/w/Attack_style", page.getUrl());
		assertEquals("Governs which stats you train.", page.getSummary());
	}

	@Test
	public void widgetTextWithAiButNullGuessFallsBackToSearch()
	{
		StubGuesser guesser = new StubGuesser();
		guesser.next = null;
		Resolver resolver = new Resolver(cache, guesser, () -> true);

		ResolvedPage page = resolver.resolve(new LookupTarget(TargetType.WIDGET_TEXT, "some odd label", -1));
		assertEquals(ResolutionSource.SEARCH, page.getSource());
	}

	@Test
	public void cacheKeyNormalisesWhitespaceAndCaseWithinAType()
	{
		assertEquals(
			Resolver.cacheKey(new LookupTarget(TargetType.WIDGET_TEXT, "  Attack   Style ", -1)),
			Resolver.cacheKey(new LookupTarget(TargetType.WIDGET_TEXT, "attack style", -1)));
	}

	@Test
	public void cacheKeySeparatesByType()
	{
		assertNull(cache.get(Resolver.cacheKey(new LookupTarget(TargetType.ITEM, "Attack", -1))).orElse(null));
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests 'com.betterwikilookup.ResolverTest' --console=plain`
Expected: FAIL — `Resolver` / `ResolvedPage` / `ResolutionSource` do not exist.

- [ ] **Step 3: Write minimal implementation**

`src/main/java/com/betterwikilookup/ResolutionSource.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

enum ResolutionSource
{
	CACHE,
	DIRECT,
	AI,
	SEARCH
}
```

`src/main/java/com/betterwikilookup/ResolvedPage.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import javax.annotation.Nullable;
import lombok.Value;

@Value
class ResolvedPage
{
	String url;

	@Nullable
	String pageTitle;

	@Nullable
	String summary;

	ResolutionSource source;
}
```

`src/main/java/com/betterwikilookup/Resolver.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import java.util.Locale;
import java.util.Optional;
import java.util.function.BooleanSupplier;
import java.util.regex.Pattern;
import lombok.extern.slf4j.Slf4j;

/**
 * The resolution pipeline: cache hit, then a direct article for named entities, then the local
 * AI guesser for free interface text, then wiki full-text search as a guaranteed fallback.
 * Blocking (the AI step may do I/O) — run on a background executor, never the client thread.
 */
@Slf4j
class Resolver
{
	private static final Pattern WHITESPACE = Pattern.compile("\\s+");

	private final ResolutionCache cache;
	private final WikiPageGuesser guesser;
	private final BooleanSupplier aiEnabled;

	Resolver(ResolutionCache cache, WikiPageGuesser guesser, BooleanSupplier aiEnabled)
	{
		this.cache = cache;
		this.guesser = guesser;
		this.aiEnabled = aiEnabled;
	}

	ResolvedPage resolve(LookupTarget target)
	{
		String key = cacheKey(target);

		Optional<CacheEntry> hit = cache.get(key);
		if (hit.isPresent())
		{
			CacheEntry e = hit.get();
			return new ResolvedPage(e.getUrl(), e.getPageTitle(), e.getSummary(), ResolutionSource.CACHE);
		}

		ResolvedPage page = resolveFresh(target);
		cache.put(key, new CacheEntry(
			page.getPageTitle(),
			page.getUrl(),
			page.getSummary(),
			page.getSource().name(),
			System.currentTimeMillis()));
		log.debug("Resolved {} '{}' via {} -> {}", target.getType(), target.getRawText(), page.getSource(), page.getUrl());
		return page;
	}

	private ResolvedPage resolveFresh(LookupTarget target)
	{
		// Named game entities (NPC / object / item / ground item) are essentially always real
		// wiki article titles; MediaWiki handles redirects and near-misses gracefully.
		if (target.getType() != TargetType.WIDGET_TEXT)
		{
			return new ResolvedPage(
				WikiUrls.article(target.getRawText()), target.getRawText(), null, ResolutionSource.DIRECT);
		}

		// Free interface text: ask the local model if the user opted in.
		if (aiEnabled.getAsBoolean())
		{
			OllamaResult guess = guesser.guess(target.getRawText(), target.getType().name());
			if (guess != null)
			{
				return new ResolvedPage(
					WikiUrls.article(guess.getTitle()), guess.getTitle(), guess.getSummary(), ResolutionSource.AI);
			}
		}

		return new ResolvedPage(WikiUrls.search(target.getRawText()), null, null, ResolutionSource.SEARCH);
	}

	static String cacheKey(LookupTarget target)
	{
		String text = WHITESPACE.matcher(target.getRawText().trim()).replaceAll(" ").toLowerCase(Locale.ROOT);
		return target.getType().name() + "|" + text;
	}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew test --tests 'com.betterwikilookup.ResolverTest' --console=plain`
Expected: PASS (6 tests).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/betterwikilookup/ResolutionSource.java src/main/java/com/betterwikilookup/ResolvedPage.java src/main/java/com/betterwikilookup/Resolver.java src/test/java/com/betterwikilookup/ResolverTest.java
git commit -m "feat: add Resolver pipeline (cache/direct/AI/search)"
```

---

### Task 7: `LookupMode`, `WikiIconOverlay`, and plugin wiring

**Files:**
- Create: `src/main/java/com/betterwikilookup/LookupMode.java`
- Test: `src/test/java/com/betterwikilookup/LookupModeTest.java`
- Create: `src/main/java/com/betterwikilookup/WikiIconOverlay.java`
- Create: `src/main/resources/com/betterwikilookup/lookup_icon.png` (see Step 6)
- Modify: `src/main/java/com/betterwikilookup/BetterWikiLookupPlugin.java` (full rewrite below)

**Interfaces:**
- Consumes: everything from Tasks 2–6, plus RuneLite: `Client`, `ClientThread`, `OverlayManager`, `Overlay`, `MouseManager`, `MouseListener`, `ItemManager`, `OkHttpClient`, `Gson`, `ScheduledExecutorService`, `MenuOptionClicked`, `LinkBrowser`, `RuneLite.RUNELITE_DIR`, `ImageUtil`.
- Produces:
  - `LookupMode` — `boolean isActive()`, `void toggle()`, `void deactivate()`, `void afterLookup(boolean oneShot)`. The single owner of transient lookup state.
  - `WikiIconOverlay extends Overlay` — constructed with `(Client client, java.util.function.BooleanSupplier active, Runnable onClick)`; draws the icon, exposes `getBounds()` (inherited), toggles via the plugin's mouse listener.

- [ ] **Step 1: Write the failing test for `LookupMode`**

`src/test/java/com/betterwikilookup/LookupModeTest.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import org.junit.Test;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertTrue;

public class LookupModeTest
{
	@Test
	public void startsInactive()
	{
		assertFalse(new LookupMode().isActive());
	}

	@Test
	public void toggleFlips()
	{
		LookupMode m = new LookupMode();
		m.toggle();
		assertTrue(m.isActive());
		m.toggle();
		assertFalse(m.isActive());
	}

	@Test
	public void deactivateAlwaysTurnsOff()
	{
		LookupMode m = new LookupMode();
		m.toggle();
		m.deactivate();
		assertFalse(m.isActive());
	}

	@Test
	public void afterLookupHonoursOneShotSetting()
	{
		LookupMode m = new LookupMode();
		m.toggle();
		m.afterLookup(false);
		assertTrue(m.isActive());
		m.afterLookup(true);
		assertFalse(m.isActive());
	}
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `./gradlew test --tests 'com.betterwikilookup.LookupModeTest' --console=plain`
Expected: FAIL — `LookupMode` does not exist.

- [ ] **Step 3: Implement `LookupMode`**

`src/main/java/com/betterwikilookup/LookupMode.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

/**
 * The whole transient-state story for the plugin: is the lookup tool currently armed? Written
 * from the client thread (icon click, menu click) and read from the overlay render thread, so
 * the flag is {@code volatile}. Nothing else in the plugin holds lookup state.
 */
final class LookupMode
{
	private volatile boolean active;

	boolean isActive()
	{
		return active;
	}

	void toggle()
	{
		active = !active;
	}

	void deactivate()
	{
		active = false;
	}

	/**
	 * Call once a lookup has been dispatched. Turns the tool off when the user chose one-shot mode.
	 */
	void afterLookup(boolean oneShot)
	{
		if (oneShot)
		{
			active = false;
		}
	}
}
```

- [ ] **Step 4: Run it to verify it passes**

Run: `./gradlew test --tests 'com.betterwikilookup.LookupModeTest' --console=plain`
Expected: PASS (4 tests).

- [ ] **Step 5: Implement `WikiIconOverlay`**

`src/main/java/com/betterwikilookup/WikiIconOverlay.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import java.awt.Color;
import java.awt.Dimension;
import java.awt.Graphics2D;
import java.awt.image.BufferedImage;
import java.util.function.BooleanSupplier;
import javax.annotation.Nullable;
import net.runelite.client.ui.overlay.Overlay;
import net.runelite.client.ui.overlay.OverlayLayer;
import net.runelite.client.ui.overlay.OverlayPosition;
import net.runelite.client.util.ImageUtil;

/**
 * Draws the clickable "wiki lookup" icon just under the minimap. The plugin's mouse listener
 * hit-tests {@link #getBounds()} to toggle {@link LookupMode}; this overlay only paints.
 */
class WikiIconOverlay extends Overlay
{
	private static final int SIZE = 22;

	private final BooleanSupplier active;

	@Nullable
	private final BufferedImage icon;

	WikiIconOverlay(BooleanSupplier active)
	{
		this.active = active;
		this.icon = loadIcon();
		setPosition(OverlayPosition.TOP_RIGHT);
		setLayer(OverlayLayer.ABOVE_WIDGETS);
	}

	@Nullable
	private static BufferedImage loadIcon()
	{
		try
		{
			return ImageUtil.loadImageResource(WikiIconOverlay.class, "/com/betterwikilookup/lookup_icon.png");
		}
		catch (RuntimeException e)
		{
			return null;
		}
	}

	@Override
	public Dimension render(Graphics2D graphics)
	{
		boolean on = active.getAsBoolean();

		graphics.setColor(on ? new Color(0, 140, 0, 180) : new Color(40, 40, 40, 180));
		graphics.fillRoundRect(0, 0, SIZE, SIZE, 6, 6);
		graphics.setColor(on ? Color.GREEN : Color.LIGHT_GRAY);
		graphics.drawRoundRect(0, 0, SIZE, SIZE, 6, 6);

		if (icon != null)
		{
			graphics.drawImage(icon, (SIZE - icon.getWidth()) / 2, (SIZE - icon.getHeight()) / 2, null);
		}
		else
		{
			graphics.drawString("W", 7, 16);
		}

		return new Dimension(SIZE, SIZE);
	}
}
```

- [ ] **Step 6: Add the icon resource**

Create `src/main/resources/com/betterwikilookup/lookup_icon.png` — a 16×16 PNG (a small book/magnifier glyph). Keep it tiny and a real PNG (not a renamed JPEG).
Quick placeholder that satisfies the build now:
```bash
mkdir -p src/main/resources/com/betterwikilookup
# 16x16 transparent PNG placeholder; replace with a real glyph before submission.
printf 'iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAHElEQVQ4y2NgGAWjYBSMglEwCkbBKBgFo2AUAAAFQAABL1s2mQAAAABJRU5ErkJggg==' | base64 --decode > src/main/resources/com/betterwikilookup/lookup_icon.png
```
(`REPO_READINESS.md` in Task 8 lists "replace placeholder icon" as a gate.)

- [ ] **Step 7: Rewrite `BetterWikiLookupPlugin` with full wiring**

`src/main/java/com/betterwikilookup/BetterWikiLookupPlugin.java`:
```java
/*
 * Copyright (c) 2026, safmailas
 * All rights reserved.
 * Licensed under the BSD 2-Clause License. See LICENSE for details.
 */
package com.betterwikilookup;

import com.google.gson.Gson;
import com.google.inject.Provides;
import java.awt.event.MouseEvent;
import java.io.IOException;
import java.nio.file.Path;
import java.util.Optional;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;
import javax.inject.Inject;
import javax.swing.SwingUtilities;
import lombok.extern.slf4j.Slf4j;
import net.runelite.api.Client;
import net.runelite.api.MenuAction;
import net.runelite.api.MenuEntry;
import net.runelite.api.events.MenuOptionClicked;
import net.runelite.api.widgets.Widget;
import net.runelite.client.RuneLite;
import net.runelite.client.callback.ClientThread;
import net.runelite.client.config.ConfigManager;
import net.runelite.client.eventbus.Subscribe;
import net.runelite.client.game.ItemManager;
import net.runelite.client.input.MouseListener;
import net.runelite.client.input.MouseManager;
import net.runelite.client.plugins.Plugin;
import net.runelite.client.plugins.PluginDescriptor;
import net.runelite.client.ui.overlay.OverlayManager;
import net.runelite.client.util.LinkBrowser;
import okhttp3.OkHttpClient;

@Slf4j
@PluginDescriptor(
	name = "Better Wiki Lookup",
	description = "Toggle a lookup tool from the minimap, then click any entity or interface to open its OSRS Wiki page",
	tags = {"wiki", "lookup", "help", "guide", "search"}
)
public class BetterWikiLookupPlugin extends Plugin
{
	private static final String DATA_DIR = "better-wiki-lookup";
	private static final long FLUSH_PERIOD_SECONDS = 10L;

	@Inject
	private Client client;

	@Inject
	private ClientThread clientThread;

	@Inject
	private BetterWikiLookupConfig config;

	@Inject
	private OverlayManager overlayManager;

	@Inject
	private MouseManager mouseManager;

	@Inject
	private ItemManager itemManager;

	@Inject
	private ScheduledExecutorService executor;

	@Inject
	private OkHttpClient okHttpClient;

	@Inject
	private Gson gson;

	private final LookupMode mode = new LookupMode();

	private ResolutionCache cache;
	private Resolver resolver;
	private WikiIconOverlay overlay;
	private IconMouseListener mouseListener;
	private ScheduledFuture<?> flushFuture;

	@Override
	protected void startUp() throws IOException
	{
		Path dir = RuneLite.RUNELITE_DIR.toPath().resolve(DATA_DIR);
		cache = new ResolutionCache(dir, gson, config.maxCacheEntries());

		OllamaClient ollama = new OllamaClient(
			okHttpClient,
			gson,
			() -> "http://" + config.ollamaHost() + ":" + config.ollamaPort(),
			config::ollamaModel);

		resolver = new Resolver(cache, ollama, config::enableAi);

		overlay = new WikiIconOverlay(mode::isActive);
		overlayManager.add(overlay);

		mouseListener = new IconMouseListener();
		mouseManager.registerMouseListener(mouseListener);

		flushFuture = executor.scheduleWithFixedDelay(
			() -> cache.flush(), FLUSH_PERIOD_SECONDS, FLUSH_PERIOD_SECONDS, TimeUnit.SECONDS);

		log.info("Better Wiki Lookup started");
	}

	@Override
	protected void shutDown()
	{
		if (flushFuture != null)
		{
			flushFuture.cancel(false);
			flushFuture = null;
		}
		if (mouseListener != null)
		{
			mouseManager.unregisterMouseListener(mouseListener);
			mouseListener = null;
		}
		if (overlay != null)
		{
			overlayManager.remove(overlay);
			overlay = null;
		}
		if (cache != null)
		{
			cache.flush();
			cache = null;
		}
		resolver = null;
		mode.deactivate();
		log.info("Better Wiki Lookup stopped");
	}

	@Provides
	BetterWikiLookupConfig provideConfig(ConfigManager configManager)
	{
		return configManager.getConfig(BetterWikiLookupConfig.class);
	}

	@Subscribe
	public void onMenuOptionClicked(MenuOptionClicked event)
	{
		if (!mode.isActive() || resolver == null)
		{
			return;
		}

		MenuEntry entry = event.getMenuEntry();
		MenuAction action = event.getMenuAction();
		if (action == MenuAction.WALK)
		{
			return;
		}

		int itemId = safeItemId(entry);
		String widgetText = widgetText(entry);

		Optional<LookupTarget> target = TargetExtractor.extract(
			action,
			event.getMenuOption() == null ? "" : event.getMenuOption(),
			event.getMenuTarget() == null ? "" : event.getMenuTarget(),
			event.getId(),
			itemId,
			widgetText,
			this::itemName);

		if (!target.isPresent())
		{
			return;
		}

		event.consume();
		LookupTarget t = target.get();
		mode.afterLookup(config.deactivateAfterLookup());

		executor.execute(() ->
		{
			try
			{
				ResolvedPage page = resolver.resolve(t);
				LinkBrowser.browse(page.getUrl());
			}
			catch (RuntimeException e)
			{
				log.debug("Wiki lookup failed for '{}'", t.getRawText(), e);
			}
		});
	}

	private int safeItemId(MenuEntry entry)
	{
		try
		{
			return entry.getItemId();
		}
		catch (Throwable t)
		{
			return -1;
		}
	}

	private String widgetText(MenuEntry entry)
	{
		Widget w = entry.getWidget();
		return w == null ? null : w.getText();
	}

	private String itemName(int id)
	{
		if (id < 0)
		{
			return null;
		}
		try
		{
			return itemManager.getItemComposition(id).getName();
		}
		catch (RuntimeException e)
		{
			return null;
		}
	}

	/**
	 * Toggles lookup mode when the wiki icon overlay is left-clicked. Consumes only that click;
	 * every other mouse event passes straight through untouched.
	 */
	private final class IconMouseListener implements MouseListener
	{
		@Override
		public MouseEvent mousePressed(MouseEvent e)
		{
			if (SwingUtilities.isLeftMouseButton(e)
				&& overlay != null
				&& overlay.getBounds() != null
				&& overlay.getBounds().contains(e.getPoint()))
			{
				mode.toggle();
				log.debug("Lookup mode {}", mode.isActive() ? "on" : "off");
				e.consume();
			}
			return e;
		}

		@Override
		public MouseEvent mouseClicked(MouseEvent e)
		{
			return e;
		}

		@Override
		public MouseEvent mouseReleased(MouseEvent e)
		{
			return e;
		}

		@Override
		public MouseEvent mouseEntered(MouseEvent e)
		{
			return e;
		}

		@Override
		public MouseEvent mouseExited(MouseEvent e)
		{
			return e;
		}

		@Override
		public MouseEvent mouseDragged(MouseEvent e)
		{
			return e;
		}

		@Override
		public MouseEvent mouseMoved(MouseEvent e)
		{
			return e;
		}
	}
}
```

> **Notes for the implementer:**
> - `ScheduledExecutorService` is bound by `net.runelite.client` for injection (the World Hopper and many hub plugins inject it). Do not create your own and do not shut it down in `shutDown()` — only cancel `flushFuture`.
> - If `MenuEntry.getItemId()` is absent in the pinned client, `safeItemId` already degrades to `-1`; inventory items then resolve by their menu target text, which is still correct in the common case.
> - `MouseListener` is `net.runelite.client.input.MouseListener` (methods return `MouseEvent`). If the pinned client offers `MouseAdapter`, you may extend it instead and override only `mousePressed`.
> - `overlay.getBounds()` is populated by `OverlayManager` after the first render; before that it is `null` and the null check above handles it.

- [ ] **Step 8: Build and run the whole suite**

Run: `./gradlew build --console=plain`
Expected: `BUILD SUCCESSFUL`; all tests from Tasks 2–7 pass; `checkstyleMain` clean.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat: wire lookup toggle, overlay, and menu-click resolution"
```

---

### Task 8: Methodology, submission, and readiness docs

**Files:**
- Create: `docs/METHODOLOGY.md`
- Create: `docs/PLUGIN_HUB_SUBMISSION.md`
- Create: `docs/REPO_READINESS.md`
- Modify: `README.md` (link the three docs; update status)

**Interfaces:**
- Consumes: the finished plugin from Tasks 1–7.
- Produces: documentation only. No code.

- [ ] **Step 1: Write `docs/METHODOLOGY.md`**

````markdown
# AI-Assisted SDLC — how this plugin was built

This plugin was produced with an AI coding agent under a fixed, review-gated
process. The same process produced `runelite-ping-plugin`.

## Stages

1. **Brainstorm.** A single approved design emerges from a Q&A that pins down
   purpose, constraints, and success criteria before any code. Output: a design
   section the human signed off on.
2. **Spec.** The design is written to `docs/superpowers/specs/` and committed
   *before* implementation. It defines every unit, its interface, the data flow,
   error handling, and the test list.
3. **Plan.** `docs/superpowers/plans/` holds a task-by-task implementation plan.
   Each task is one unit, carries its own failing-test-first cycle, and ends at a
   commit a reviewer can accept or reject in isolation.
4. **Implement.** One fresh agent per task. The agent sees only its task's
   interface block plus the global constraints — not the whole codebase — which
   keeps changes small and forces the interfaces to be real.
5. **Review gates.**
   - Spec self-review (placeholders, contradictions, scope) before planning.
   - Plan self-review (spec coverage, type consistency) before implementing.
   - Per-task human review at each commit.
   - Pre-submission review against `AGENTS.md` and the Plugin Hub rules.
6. **In-game verification.** The agent never drives the game. The human runs
   `./gradlew run` and confirms the manual test matrix in `REPO_READINESS.md`.

## Artifacts kept in the repo

| Path | What |
|---|---|
| `docs/superpowers/specs/` | Approved design spec(s) |
| `docs/superpowers/plans/` | Task-by-task implementation plan(s) |
| `AGENTS.md` | Standing rules every agent must follow (copied from the template) |
| Git history | One commit per plan task, message prefixed `feat:` / `test:` / `docs:` |

## Why this shape

- The spec-before-code gate is where wrong assumptions get caught cheaply.
- One-unit-per-task keeps each diff inside a reviewer's working memory.
- Pure logic classes + thin RuneLite glue means the important behaviour is
  unit-tested and the un-testable part is small enough to eyeball.
````

- [ ] **Step 2: Write `docs/PLUGIN_HUB_SUBMISSION.md`**

````markdown
# Submitting Better Wiki Lookup to the RuneLite Plugin Hub

## Preconditions

Every box in `docs/REPO_READINESS.md` is checked, and `master` is pushed to
`https://github.com/safmailas/runelite-better-wiki-lookup`.

## Steps

1. Fork `https://github.com/runelite/plugin-hub`.
2. Branch: `git checkout -b better-wiki-lookup`.
3. Create `plugins/better-wiki-lookup` (no extension) with:
   ```
   repository=https://github.com/safmailas/runelite-better-wiki-lookup.git
   commit=<40-char hash of the release commit on master>
   ```
4. Commit: `git commit -m "Add Better Wiki Lookup plugin"`.
5. Push and open a PR against `runelite/plugin-hub`.
6. Watch the CI workflow on the PR. Fix any failure (usually checkstyle, a
   forbidden API, or an unpinned dependency) in *this* repo, push, then update
   the `commit=` line in the hub PR.

## PR description template

```
### Better Wiki Lookup

Adds a minimap-area toggle. While active, a left-click on any NPC, object,
ground item, inventory item, or interface widget opens the matching
oldschool.runescape.wiki page in the browser.

**Network:** none required. Named entities build a /w/<Title> URL locally;
unknown interface text opens /w/Special:Search. Optionally, if the user turns on
"Use local AI" (off by default), the clicked text is POSTed to a locally running
Ollama instance (default http://localhost:11434) to pick a better title. Nothing
is sent anywhere if Ollama is not running, and the plugin is fully functional
with the option off.

**Menus/input:** the plugin consumes the existing left-click via
MenuOptionClicked.consume() while the tool is active. It does not add menu
entries, does not send actions to the server, and does not inject input.

**Storage:** one JSON file under .runelite/better-wiki-lookup/ caching resolved
titles so repeat lookups skip the AI/search step.

**Dependencies:** none beyond runelite-client transitives (injected OkHttp + Gson).
```

## Review-risk checklist (answer these before opening the PR)

- [ ] `enableAi` defaults to `false` and has a `warning=` string. ✔ (Task 1)
- [ ] With AI off, no outbound network request is ever made by the plugin. ✔
- [ ] With AI on but Ollama down, behaviour is identical to AI off. ✔ (Task 4/6 tests)
- [ ] No `MenuOptionClicked` handler ever creates or re-targets a server action —
      it only calls `consume()`. ✔ (Task 7)
- [ ] No new runtime dependency in `build.gradle`. ✔
- [ ] No forbidden API (reflection, Process, serialization, dynamic loading). ✔
- [ ] File I/O is confined to `.runelite/better-wiki-lookup/`. ✔
````

- [ ] **Step 3: Write `docs/REPO_READINESS.md`**

````markdown
# Release readiness checklist

Gate for opening the Plugin Hub PR. Every box must be checked.

## Build & tests
- [ ] `./gradlew build` passes locally on JDK 11.
- [ ] GitHub Actions `build` is green on both JDK 11 and 21.
- [ ] All unit tests pass: `WikiUrlsTest`, `ResolutionCacheTest`, `OllamaClientTest`,
      `TargetExtractorTest`, `ResolverTest`, `LookupModeTest`.
- [ ] `./gradlew checkstyleMain` is clean.

## Packaging
- [ ] No `com.example` / `Example*` names anywhere; config group is `betterwikilookup`.
- [ ] No `META-INF/services/net.runelite.client.plugins.Plugin` file.
- [ ] No build artifacts committed (`git status` clean, `.gitignore` covers `build/`, `*.class`).
- [ ] `LICENSE` is BSD 2-Clause; every `.java` file has the licence header.
- [ ] `runelite-plugin.properties` `displayName`, `author`, `description`, `tags`, `plugins` all correct.
- [ ] `src/main/resources/com/betterwikilookup/lookup_icon.png` is a real, optimised PNG
      (placeholder from the plan **replaced**).
- [ ] Two screenshots committed under `docs/`: the icon in place; a resolved lookup opening the wiki.

## Manual in-game matrix (`./gradlew run`, dev account)

For each, with **AI off** then **AI on** (Ollama running `llama3.2`):

| Target | Expected |
|---|---|
| NPC (e.g. a Cow) | opens `/w/Cow` |
| Game object (e.g. a Bank booth) | opens `/w/Bank_booth` |
| Ground item | opens `/w/<item>` |
| Inventory item | opens `/w/<item>` |
| Skill Guide row (e.g. an Attack unlock) | AI off: search page; AI on: relevant article |
| Combat Options label ("Attack style") | AI off: search page; AI on: `/w/Attack_style` (or similar) |
| Settings/Options label | AI off: search; AI on: best-effort article |
| Empty ground while active | nothing happens, tool stays on |
| Second lookup of any above | opens instantly (cache hit) — confirm via `.runelite/better-wiki-lookup/resolutions.json` |
| Toggle icon | icon turns green/grey; a click while off does nothing |
| One-shot config on | tool turns off after one lookup |

- [ ] Matrix executed and all rows pass (confirmed by the human).

## Docs
- [ ] `docs/METHODOLOGY.md`, `docs/PLUGIN_HUB_SUBMISSION.md` complete.
- [ ] `README.md` links all three docs and states current status.
````

- [ ] **Step 4: Update `README.md`**

Replace the "Status" line and add a docs list:
```markdown
**Status:** implementation complete; pre-submission review pending.

## Docs
- [Design spec](docs/superpowers/specs/2026-08-31-better-wiki-lookup-design.md)
- [Implementation plan](docs/superpowers/plans/2026-09-01-better-wiki-lookup.md)
- [AI-SDLC methodology](docs/METHODOLOGY.md)
- [Plugin Hub submission runbook](docs/PLUGIN_HUB_SUBMISSION.md)
- [Release readiness checklist](docs/REPO_READINESS.md)
```

- [ ] **Step 5: Commit**

```bash
git add docs/METHODOLOGY.md docs/PLUGIN_HUB_SUBMISSION.md docs/REPO_READINESS.md README.md
git commit -m "docs: add methodology, submission runbook, and readiness checklist"
```

---

## Post-implementation (not a task — for the human)

Per `AGENTS.md`, do not declare the plugin done. Instead:

1. Run `./gradlew run` from `~/runelite-better-wiki-lookup`.
2. Log into the dev client (see https://github.com/runelite/runelite/wiki/Using-Jagex-Accounts).
3. Work through the manual matrix in `docs/REPO_READINESS.md`, AI off then AI on.
4. Only once every row passes, copy the `AGENTS.md` from the template repo, add the two
   screenshots, tag a release commit, and follow `docs/PLUGIN_HUB_SUBMISSION.md`.

---

## Self-Review

**1. Spec coverage**

| Spec section | Task |
|---|---|
| §2 Activation (icon toggle, persists, one-shot) | Task 7 (`LookupMode`, `WikiIconOverlay`, `IconMouseListener`) |
| §2 Lookup click (consume left-click, ignore right/drag/non-target) | Task 7 (`onMenuOptionClicked`) |
| §2 Resolution & open (executor, ~2s, `LinkBrowser`) | Task 7 dispatch + Task 4 timeouts |
| §3 `BetterWikiLookupPlugin` | Task 1 skeleton, Task 7 wiring |
| §3 `BetterWikiLookupConfig` | Task 1 |
| §3 `IconOverlay` | Task 7 (`WikiIconOverlay`) |
| §3 `LookupClickHandler` | Task 7 (folded into plugin as `onMenuOptionClicked` + `IconMouseListener`, per the "keep glue in the plugin" testing rule) |
| §3 `TargetExtractor` + `LookupTarget` types | Task 5 |
| §3 `Resolver` | Task 6 |
| §3 `OllamaClient` + `isAvailable()` | Task 4 (`guess` + `available`) |
| §3 `WikiUrls` | Task 2 |
| §3 `ResolutionCache` (one file, LRU, debounced save) | Task 3 (+ Task 7 schedules `flush()`) |
| §3 `ResolvedPage.source` = DIRECT/AI/SEARCH (+CACHE) | Task 6 (`ResolutionSource`) |
| §4 State management (one `volatile` flag; cache sole persistent owner) | Task 7 (`LookupMode`), Task 3 |
| §5 Data flow + normalized cache key | Task 6 (`cacheKey`) |
| §6 Error handling (every row) | Task 3 (corrupt/missing/write-fail/LRU), Task 4 (Ollama down/garbage), Task 6 (AI-off/null → search), Task 7 (blank target, browse failure) |
| §7 Testing (all five listed test classes + manual matrix) | Tasks 2–6 tests; Task 8 matrix. Extra: `LookupModeTest` (Task 7). |
| §8 Repo & build (JDK 11/21 CI, BSD-2, headers, icon, zero deps) | Task 1 + Task 8 checklist |
| §9 METHODOLOGY / PLUGIN_HUB_SUBMISSION / REPO_READINESS | Task 8 |
| §10 v2 deferrals | Not implemented, by design |

No gaps.

**2. Placeholder scan**

The Task 7 icon PNG is a base64 placeholder, but it is a *working* 16×16 PNG (build passes) and `REPO_READINESS.md` explicitly gates replacing it — this is a deliberate, tracked stub, not a plan hole. No "TBD", no "add error handling", no un-shown code. The `LICENSE` and `AGENTS.md` steps say "copy the exact text from <source>" rather than inlining ~1.5 KB of licence boilerplate — acceptable since the source is a specific file/URL.

**3. Type consistency**

- `WikiPageGuesser.guess(String, String)` / `available()` — defined Task 4, implemented by `OllamaClient` Task 4, consumed by `Resolver` Task 6, stubbed in `ResolverTest`. Consistent.
- `ResolutionCache` `get`/`put`/`flush` + `CacheEntry(pageTitle, url, summary, source, savedAtEpochMs)` — defined Task 3, used identically in Task 6 (`new CacheEntry(...)` with 5 args in that order) and Task 7 (`flush`). Consistent.
- `LookupTarget(TargetType, String, int)` / `TargetType` — defined Task 5, constructed in Task 6 tests and Task 7 with the same arg order. Consistent.
- `ResolvedPage(url, pageTitle, summary, source)` + `ResolutionSource{CACHE,DIRECT,AI,SEARCH}` — defined Task 6, consumed Task 7 (`page.getUrl()`). Consistent.
- `TargetExtractor.extract(MenuAction, String, String, int, int, String, IntFunction<String>)` — one signature, used identically in Task 5 tests and Task 7 call site. Consistent.
- `LookupMode` `isActive`/`toggle`/`deactivate`/`afterLookup(boolean)` — defined and consumed only in Task 7. Consistent.
- `WikiUrls.article` / `WikiUrls.search` — defined Task 2, consumed Task 6. Consistent.

No mismatches found.
