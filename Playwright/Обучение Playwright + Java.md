

--------------------------------------------------------------------------------

## 1. Тематический блок: Навигация и управление браузером

### 1.1. Архитектура и базовая инициализация

**Tags:** #playwright #java #architecture #setup **Senior Note:** Playwright работает через протокол **WebSocket**, обеспечивая двустороннюю связь с браузером в рамках одного соединения. Это делает его в **2–5 раз быстрее Selenium**, который ограничен накладными расходами HTTP-запросов (открытие/закрытие соединения на каждое действие).

При навигации через `page.navigate()` Playwright выполняет **5 автоматических проверок** (Actionability Checks): элемент прикреплен к DOM, видим, стабилен (не анимируется), не перекрыт другими элементами и включен.

```java
// Инициализация (используйте try-with-resources для авто-закрытия в простых скриптах)
try (Playwright playwright = Playwright.create()) {
    Browser browser = playwright.chromium().launch(new BrowserType.LaunchOptions()
        .setHeadless(false) // Включаем графический интерфейс для отладки
        .setSlowMo(500));    // Замедление для визуального контроля

    Page page = browser.newPage();
    // Автоматическое ожидание загрузки DOM (load state)
    page.navigate("https://example.com");
    
    System.out.println("Заголовок: " + page.title());
}
```

--------------------------------------------------------------------------------

### 1.2.  Контексты и изоляция сессий

**Tags:** #playwright #browser_context #isolation **Key Concept:** `BrowserContext` — это «суперсила» Playwright. Он создает полностью изолированный инкогнито-сеанс (свои куки, кэш, хранилище) без затрат на запуск нового процесса браузера. Это позволяет тестировать мультиролевые сценарии (например, Админ vs Клиент) в одном тесте.

```java
// Один браузер — несколько изолированных миров
BrowserContext adminContext = browser.newContext();
BrowserContext userContext = browser.newContext();

Page adminPage = adminContext.newPage();
adminPage.navigate("https://system.com/admin");

Page userPage = userContext.newPage();
userPage.navigate("https://system.com/app");
```

--------------------------------------------------------------------------------

### 1.3.  Эмуляция мобильных устройств

**Tags:** #playwright #mobile #emulation **Senior Note:** Playwright не просто меняет размер окна (resize), а полноценно эмулирует физические параметры устройства: Viewport, User Agent, Device Scale Factor и наличие тач-интерфейса.

```java
// Эмуляция iPhone 12 Pro на уровне контекста
BrowserContext context = browser.newContext(new Browser.NewContextOptions()
    .setUserAgent("Mozilla/5.0 (iPhone; CPU iPhone OS 14_4 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/14.0.3 Mobile/15E148 Safari/604.1")
    .setViewportSize(390, 844)
    .setDeviceScaleFactor(3)
    .setIsMobile(true)
    .setHasTouch(true));

Page page = context.newPage();
page.navigate("https://m.example.com");
```

--------------------------------------------------------------------------------

### 1.4.  Сохранение состояния (Storage State)

**Tags:** #playwright #performance #auth **Efficiency Stats:** Использование `Storage State` (сохранение cookies и localStorage в JSON) позволяет пропускать этап UI-логина в каждом тесте, что **экономит до 70% времени** выполнения тестовой сюиты.

```java
// 1. Сохраняем состояние после авторизации
context.storageState(new BrowserContext.StorageStateOptions()
    .setPath(Paths.get("state.json")));

// 2. Используем состояние в новых тестах
BrowserContext authContext = browser.newContext(new Browser.NewContextOptions()
    .setStorageStatePath(Paths.get("state.json")));
Page page = authContext.newPage();
page.navigate("https://example.com/dashboard"); // Вы уже внутри!
```

--------------------------------------------------------------------------------

## 2. Тематический блок: Поиск элементов (Локаторы)

### 2.1. Стратегии поиска (Locators)

**Tags:** #playwright #locators #best_practices **Senior Tip:** Используйте `getByRole` (Aria Role) — это самый стабильный метод, основанный на доступности (accessibility), который не ломается при изменении CSS-классов или структуры верстки.

|   |   |   |
|---|---|---|
|Метод|Почему используем|Пример|
|`getByRole`|Самый стабильный (семантика)|`page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Log in"))`|
|`getByText`|Поиск по видимому тексту|`page.getByText("Success")`|
|`locator("css")`|Для сложных технических путей|`page.locator(".submit-btn")`|

--------------------------------------------------------------------------------

### 2.2.  Работа с IFrames («Матрёшка»)

**Tags:** #playwright #iframe #nesting **Analogy:** Вложенные фреймы обрабатываются как матрёшка — через цепочку вызовов `frameLocator()`.

```java
// Доступ к элементу в глубоко вложенном фрейме
page.frameLocator("#parent-frame")
    .frameLocator("#child-frame")
    .getByRole(AriaRole.BUTTON)
    .click();
```

--------------------------------------------------------------------------------

### 2.3.  Динамические элементы и XPath

**Tags:** #playwright #xpath #dynamic_elements **Senior Warning:** Если ID генерируется динамически, используйте `contains` в XPath. Однако помните, что DOM динамичен: если локатор находит несколько элементов, используйте `.first()` или точные индексы, чтобы избежать ошибок неоднозначности.

```java
// Поиск по частичному совпадению ID
Locator dynamicIframe = page.locator("//iframe[contains(@id, 'frame-id-')]");
dynamicIframe.first().click(); // Берем первый найденный, если ID дублируются в дереве
```

--------------------------------------------------------------------------------

## 3. Тематический блок: Взаимодействие с элементами

### 3.1. Ввод текста и клики

**Tags:** #playwright #interaction #speed **Senior Note:** Метод `fill()` — лучший выбор для скорости, так как он мгновенно устанавливает значение. `type()` имитирует нажатие клавиш (посимвольно), что полезно только для тестирования живого поиска или полей с масками.

**5 Checks before Click:** Playwright проверяет, что элемент: 1. Прикреплен; 2. Видим; 3. Стабилен; 4. Не перекрыт; 5. Включен.

```java
// Быстрое заполнение (Best practice)
page.fill("#email", "senior@dev.com");

// Имитация медленного ввода (100мс задержка)
page.locator("#search").type("Playwright", new Locator.TypeOptions().setDelay(100));

page.click("#submit");
```

--------------------------------------------------------------------------------

### 3.2.  Клики с модификаторами (Ctrl/Shift)

**Tags:** #playwright #keyboard #interaction

```java
// Shift + Click для открытия ссылки в новой вкладке
page.click("a#terms", new Page.ClickOptions()
    .setModifiers(Arrays.asList(KeyboardModifier.SHIFT)));

// Ctrl + Click для множественного выбора (Multi-select)
page.click(".item-1", new Page.ClickOptions().setModifiers(Arrays.asList(KeyboardModifier.CONTROL)));
page.click(".item-2", new Page.ClickOptions().setModifiers(Arrays.asList(KeyboardModifier.CONTROL)));
```

--------------------------------------------------------------------------------

### 3.3.  Чекбоксы, Радиокнопки и Селекты

**Tags:** #playwright #forms #ui_elements **Senior Note:** Радиокнопки нельзя «отключить» (uncheck) программно — их состояние меняется только выбором другой опции в группе.

```java
// Чекбоксы
page.check("#agree");
assertThat(page.locator("#agree")).isChecked();

// Радиокнопки (лучше кликать по label для стабильности)
page.getByLabel("Male").check();

// Выпадающие списки (Select)
page.selectOption("#country", "RU"); // по value
page.selectOption("#country", new SelectOption().setLabel("Russia")); // по тексту
```

--------------------------------------------------------------------------------

### 3.4.  Hover и динамические меню

**Tags:** #playwright #hover #dynamic_ui **Concept:** Метод `hover()` триггерит появление скрытых элементов. Playwright автоматически дождется появления подменю в DOM.

```java
page.locator("#main-menu").hover();
// Взаимодействуем с появившимся пунктом
page.locator("#sub-menu-item").click();
```

--------------------------------------------------------------------------------

### 3.5.  Обработка диалогов (Alert/Confirm)

**Tags:** #playwright #dialogs **Critical Rule:** Обработчик `onDialog` должен быть зарегистрирован **ДО** того, как будет совершено действие, вызывающее диалог.

```java
// Регистрируем слушатель
page.onDialog(dialog -> {
    System.out.println("Message: " + dialog.message());
    dialog.accept("Custom Text for Prompt"); // Принять и ввести текст
});

page.click("#trigger-alert");
```

--------------------------------------------------------------------------------

### 3.6.  Загрузка и скачивание файлов

**Tags:** #playwright #files #download

```java
// 1. Включаем поддержку скачивания в контексте
BrowserContext context = browser.newContext(new Browser.NewContextOptions()
    .setAcceptDownloads(true));

// 2. Перехватываем событие загрузки
Download download = page.waitForDownload(() -> {
    page.getByText("Download Report").click();
});

download.saveAs(Paths.get("downloads/report.pdf"));
```

--------------------------------------------------------------------------------

## 4. Тематический блок: Ожидания и стабильность

### 4.1.  Умные ассерты (Assertions)

**Tags:** #playwright #assertions #stability **Senior Insight:** `Playwright Assertions` используют интеллектуальные ретраи. Сравнение: `textContent()` возвращает весь текст, включая скрытый (нужен `trim()`), а `innerText()` — только то, что видит пользователь. Используйте `assertThat(...).containsText()` для частичных проверок.

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

// Автоматический ретрай проверки до 5 секунд
assertThat(page.locator(".welcome-msg")).isVisible();
assertThat(page.locator(".status")).hasText("Completed");
```

--------------------------------------------------------------------------------

### 4.2.  Явные ожидания (Explicit Waits)

**Tags:** #playwright #waits #timeouts **Rule of Thumb:** Рекомендуемые тайм-ауты: **10–15 сек** для сетевых операций, **3–5 сек** для изменений UI.

```java
// Ожидание конкретного условия через JS
page.waitForFunction("() => document.querySelector('.loader').style.display === 'none'", 
    new Page.WaitForFunctionOptions().setTimeout(5000));

// Ожидание сетевого ответа
page.waitForResponse(response -> response.url().contains("/api/save") && response.status() == 200, 
    () -> page.click("#save-btn"));
```

--------------------------------------------------------------------------------

## 5. Тематический блок: Продвинутые инструменты и отладка

### 5.1.  Трассировка (Tracing)

**Tags:** #playwright #debug #tracing **Concept:** Трейс — это ZIP-архив с записью действий, сетевых логов и снимков DOM.

```java
context.tracing().start(new Tracing.StartOptions()
    .setScreenshots(true)
    .setSnapshots(true)
    .setSources(true));

// ... выполнение теста ...

context.tracing().stop(new Tracing.StopOptions()
    .setPath(Paths.get("trace.zip")));
```

--------------------------------------------------------------------------------

### 5.2.  Мокирование API (Network Interception)

**Tags:** #playwright #mocking #api

```java
// Перехват запроса и подмена ответа
page.route("**/api/v1/user", route -> {
    String mockJson = "{ \"name\": \"Test User\", \"role\": \"admin\" }";
    route.fulfill(new Route.FulfillOptions()
        .setStatus(200)
        .setBody(mockJson));
});
```

--------------------------------------------------------------------------------

### 5.3.  Allure и TestWatcher

**Tags:** #playwright #allure #reporting **Senior Practice:** Используйте `TestWatcher` для автоматических скриншотов только при падении.

```java
import io.qameta.allure.Attachment;
import org.junit.jupiter.api.extension.TestWatcher;

public class TestFailureWatcher implements TestWatcher {
    @Attachment(value = "Failure Screenshot", type = "image/png")
    public byte[] captureScreenshot(Page page) {
        return page.screenshot(new Page.ScreenshotOptions().setFullPage(true));
    }
}
```

--------------------------------------------------------------------------------

### 5.4.  Параллельный запуск и CI/CD

**Tags:** #playwright #ci_cd #parallel **ARCHITECTURAL WARNING:** Категорически **запрещено** использовать `static` Page или Browser объекты в параллельных тестах — это ведет к гонкам данных и падениям.

**junit-platform.properties:**

```properties
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
junit.jupiter.execution.parallel.config.fixed.parallelism = 4
```

**GitHub Actions (Precise snippet):**

```yaml
- name: Run Tests in XVFB
  run: |
    sudo apt-get install -y xvfb
    export DISPLAY=:99
    xvfb-run mvn test
```