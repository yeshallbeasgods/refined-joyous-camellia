## Project Overview

A Java + Playwright test automation project targeting [Sauce Demo](https://www.saucedemo.com) (e2e) and the OpenWeather API (API/data quality). Built as a portfolio piece demonstrating API testing depth, data quality validation, and Allure reporting — patterns directly relevant to MarTech/platform SDET roles.

**Status:** early scaffolding — structure and conventions below will fill in as tickets complete.

---

## Build and Test Commands

```bash
# Compile and run all tests
./gradlew test

# Run smoke (e2e) tests only - needs a tag-filter property wired into
# build.gradle's `test { useJUnitPlatform { includeTags ... } }` once tests
# with @Tag("smoke") exist; not wired yet since there are no tests to tag.
./gradlew test -Ptags=smoke

# Run API tests only
./gradlew test -Ptags=api

# Run a single test method - Gradle's built-in test filter, no extra config needed
./gradlew test --tests "CartTest.testAddSingleItemToCart"

# Run with browser visible (not headless) - needs `test { systemProperty
# 'headless', ... }` forwarding added in build.gradle once BrowserFactory
# reads it; a `-D` on the gradle command line only sets a property on the
# Gradle daemon, not the forked test JVM, unless forwarded explicitly.
./gradlew test -Dheadless=false

# Generate and view Allure report
./gradlew allureReport
./gradlew allureServe
```

(Project was built with Gradle + Groovy DSL, not Maven - see [build.gradle](build.gradle) and [settings.gradle](settings.gradle). Always run the wrapper (`./gradlew`), never a bare `gradle`, so everyone uses the exact pinned Gradle version.)

---

## Architecture

```
src/test/java/
  config/
    ConfigManager.java       # Loads .env — always go through this, never System.getenv() directly
  pages/
    BasePage.java            # All e2e page objects extend this
    LoginPage.java, InventoryPage.java, CartPage.java, CheckoutPage.java
  api/
    OpenWeatherClient.java   # Typed RestAssured client
    models/                  # POJOs for JSON deserialization (Jackson)
  tests/
    e2e/                     # Extends TestBase — needs a browser
    api/                     # Does NOT extend TestBase — no browser needed
  utils/
    BrowserFactory.java      # Creates Playwright/Browser/Context/Page
    TestBase.java            # JUnit 5 lifecycle (@BeforeEach/@AfterEach), screenshot on failure
  resources/
    allure.properties
    test-data/                # JSON arrays for parametrized tests (cities, users, products)
```

---

## Naming Conventions

- **Locators**: use `page.locator("[data-test='...']")` — `data-test` attributes preferred, defined inside page object methods, never inline in test classes
- **Page objects**: extend `BasePage`, constructor takes a `Page` instance
- **API models**: one POJO per JSON object level (e.g. `WeatherResponse`, `Main`, `Weather`, `Coord` — not one flat class), annotated with `@JsonIgnoreProperties(ignoreUnknown = true)`
- **Test classes**: e2e tests extend `TestBase`; API tests do not
- **Config**: all environment values (URLs, credentials, API keys) go through `ConfigManager` — never hardcoded, never read directly from `System.getenv()` in a test or page object
- **Parametrized/array-driven tests**: use `@ParameterizedTest` + `@MethodSource` returning `Stream<Arguments>` backed by `List<>` — this project intentionally showcases Java collection/array handling
- **Tags**: every test method tagged `@Tag("smoke")` or `@Tag("api")`

---

## Allure Conventions

- `@Epic` at the class level — one per target system (`"Sauce Demo"`, `"OpenWeather API"`)
- `@Feature` at the class level — one per functional area (`"Login"`, `"Cart"`, `"Weather Data Quality"`)
- `@Story` and `@Description` on individual test methods
- On failure, `TestBase` attaches a screenshot and the current page URL to the Allure report automatically — don't duplicate this manually in test methods

---

## What Not To Do

- **No `Thread.sleep()`.** Use Playwright's built-in auto-waiting or explicit `page.waitForURL()` / `locator.waitFor()` calls.
- **No hardcoded credentials, API keys, or URLs.** Everything sensitive or environment-specific goes through `ConfigManager` and `.env`.
- **No raw string selectors in test classes.** Selectors belong inside page object methods only.
- **No `System.getenv()` outside `ConfigManager`.**
- **No mixing `TestBase` into API tests.** API tests are stateless HTTP calls — they don't need a browser lifecycle.

---

## Fixtures / Lifecycle

There's no conftest equivalent — setup and teardown live in JUnit 5 lifecycle annotations inside `TestBase`:

| Class | Extends | Needs browser? |
|---|---|---|
| e2e test classes (`LoginTest`, `CartTest`) | `TestBase` | Yes — `@BeforeEach` creates Playwright/Browser/Context/Page, `@AfterEach` tears down |
| API test classes (`OpenWeatherTest`) | none | No — instantiate `OpenWeatherClient` directly |

---

## Debugging

**Local:**
- Run a single failing test with `--tests "ClassName.methodName" -Dheadless=false` to watch it happen
- On failure, `TestBase` writes a screenshot to `build/screenshots/{testName}.png` and attaches it to Allure automatically

**CI:**
- Allure results are uploaded as a workflow artifact on every run (`build/allure-results`)
- Download the artifact and run `allure serve <path>` locally to view

---

## Documentation

- Update this file as architecture decisions get made (page object patterns, API client conventions, parallel execution setup)
- Update `README.md` roadmap checkboxes as tickets close
