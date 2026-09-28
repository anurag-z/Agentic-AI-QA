# Playwright C# NUnit — Complete Reference
> Hierarchical master reference for RAG retrieval and interview prep.  
> Stack: .NET 8 · Microsoft.Playwright 1.61.0 · NUnit 3  
> All APIs are async — every method ends with `Async` and returns a `Task` or `Task<T>`.

---

## Table of Contents
1. [Test Infrastructure](#1-test-infrastructure)
2. [Browser / Context / Page Lifecycle](#2-browser--context--page-lifecycle)
3. [Locators](#3-locators)
4. [Actions](#4-actions)
5. [Assertions (Expect)](#5-assertions-expect)
6. [Waits and Auto-Waiting](#6-waits-and-auto-waiting)
7. [Network Interception](#7-network-interception)
8. [API Testing](#8-api-testing)
9. [HAR Recording and Replay](#9-har-recording-and-replay)
10. [Screenshots and Visual Regression](#10-screenshots-and-visual-regression)
11. [JavaScript Evaluation](#11-javascript-evaluation)
12. [Keyboard](#12-keyboard)
13. [Mouse](#13-mouse)
14. [Dialogs](#14-dialogs)
15. [Multi-Tab / Popups](#15-multi-tab--popups)
16. [Downloads](#16-downloads)
17. [File Upload](#17-file-upload)
18. [Emulation](#18-emulation)
19. [Authentication and Storage State](#19-authentication-and-storage-state)
20. [Tracing](#20-tracing)
21. [IBrowserContext Methods](#21-ibrowsercontext-methods)
22. [Frames and iFrames](#22-frames-and-iframes)
23. [Page Object Model (POM)](#23-page-object-model-pom)
24. [Configuration, Parallelism, and CI/CD](#24-configuration-parallelism-and-cicd)

---

## 1. Test Infrastructure

### 1.1 Base class hierarchy

```
PageTest          ← provides Page, Context, Browser, BrowserType, Playwright
  └── ContextTest ← same; override ContextOptions() to change context
        └── BrowserTest ← lets you create multiple contexts
              └── PlaywrightTest ← full control; manage browser yourself
```

- `BaseTest : PageTest` — project-level base; overrides `ContextOptions()` to load `StorageStatePath`.
- Every test class inherits `BaseTest` (or `PageTest` directly).
- `[SetUp]` / `[TearDown]` are per-test; `[OneTimeSetUp]` / `[OneTimeTearDown]` are per-fixture.

### 1.2 GlobalSetup — one-time login for the whole suite

```csharp
[SetUpFixture]
public class GlobalSetup
{
    public static string TestEmail    => Environment.GetEnvironmentVariable("NOTES_TEST_EMAIL")!;
    public static string TestPassword => Environment.GetEnvironmentVariable("NOTES_TEST_PASSWORD")!;

    [OneTimeSetUp]
    public async Task RunBeforeAllTests()
    {
        using var playwright = await Playwright.CreateAsync();
        await using var browser = await playwright.Chromium.LaunchAsync(new() { Headless = true });
        var context = await browser.NewContextAsync();
        var page    = await context.NewPageAsync();

        // UI login
        await page.GotoAsync("https://example.com/login");
        await page.GetByLabel("Email").FillAsync(TestEmail);
        await page.GetByLabel("Password").FillAsync(TestPassword);
        await page.GetByRole(AriaRole.Button, new() { Name = "Login" }).ClickAsync();
        await page.WaitForURLAsync("**/app");

        // persist auth
        await context.StorageStateAsync(new() { Path = "playwright/.auth/user.json" });
    }
}
```

### 1.3 BaseTest — loads storageState for every test

```csharp
public class BaseTest : PageTest
{
    // Called by PageTest before every test — customize context here
    public override BrowserNewContextOptions ContextOptions() => new()
    {
        StorageStatePath = "playwright/.auth/user.json"
    };

    // Hook-points for subclasses — called by SetUp/TearDown below
    protected virtual Task OnSetUpAsync()    => Task.CompletedTask;
    protected virtual Task OnTearDownAsync() => Task.CompletedTask;

    [SetUp]    public async Task SetUp()    => await OnSetUpAsync();
    [TearDown] public async Task TearDown() => await OnTearDownAsync();
}
```

### 1.4 Overriding launch options per test class

```csharp
// Change browser or launch settings for one class
public override BrowserNewContextOptions ContextOptions() => new()
{
    StorageStatePath  = "playwright/.auth/user.json",
    ViewportSize      = new() { Width = 1920, Height = 1080 },
    ColorScheme       = ColorScheme.Dark,
    Locale            = "en-US",
    TimezoneId        = "America/New_York",
};
```

---

## 2. Browser / Context / Page Lifecycle

```
IPlaywright  →  IBrowserType (Chromium/Firefox/WebKit)
                    ↓ LaunchAsync
              IBrowser
                    ↓ NewContextAsync
              IBrowserContext  ← cookies, routes, headers, auth, permissions
                    ↓ NewPageAsync
              IPage  ← URL, DOM, events, one tab
```

### 2.1 Manual browser management (BrowserTest)

```csharp
public class MyTests : BrowserTest
{
    private IBrowserContext _context1 = null!;
    private IBrowserContext _context2 = null!;

    [SetUp]
    public async Task CreateContexts()
    {
        _context1 = await Browser.NewContextAsync(new() { StorageStatePath = "user1.json" });
        _context2 = await Browser.NewContextAsync(new() { StorageStatePath = "user2.json" });
    }

    [TearDown]
    public async Task DisposeContexts()
    {
        await _context1.CloseAsync();
        await _context2.CloseAsync();
    }
}
```

### 2.2 Key page navigation methods

```csharp
await Page.GotoAsync("https://example.com");
await Page.GotoAsync("https://example.com", new() { WaitUntil = WaitUntilState.NetworkIdle });
await Page.ReloadAsync();
await Page.GoBackAsync();
await Page.GoForwardAsync();
string url   = Page.Url;
string title = await Page.TitleAsync();
```

---

## 3. Locators

Locators describe how to find an element. They are **lazy** — no DOM query runs until an action or assertion uses them. They **auto-retry** with built-in wait.

### 3.1 Seven built-in semantic locators (prefer these)

```csharp
// 1. Role — matches ARIA role
Page.GetByRole(AriaRole.Button, new() { Name = "Submit" })
Page.GetByRole(AriaRole.Link,   new() { Name = /sign in/i })   // Regex
Page.GetByRole(AriaRole.Textbox, new() { Name = "Email", Exact = true })

// 2. Label text — matches <label> text or aria-label
Page.GetByLabel("Password")
Page.GetByLabel("Password", new() { Exact = true })

// 3. Placeholder — matches placeholder attribute
Page.GetByPlaceholder("Enter email")

// 4. Text — matches visible text content
Page.GetByText("Sign in")
Page.GetByText("Sign in", new() { Exact = true })

// 5. Alt text — matches alt attribute on images
Page.GetByAltText("Company logo")

// 6. Title — matches title attribute or tooltip
Page.GetByTitle("Close dialog")

// 7. Test ID — matches data-testid (or custom attribute)
Page.GetByTestId("submit-button")   // default: data-testid
```

### 3.2 CSS and XPath selectors

```csharp
// CSS — flexible but fragile with deep paths
Page.Locator("button.btn-primary")
Page.Locator("#username")
Page.Locator("[data-cy='login-btn']")
Page.Locator("input[type='email']")

// XPath
Page.Locator("//button[@type='submit']")
Page.Locator("xpath=//div[@class='modal']")

// XPath axes
//ancestor::  //parent::  //child::  //following-sibling::  //preceding-sibling::
```

### 3.3 Locator filtering

```csharp
// Filter by text
Page.GetByRole(AriaRole.Listitem).Filter(new() { HasText = "Product" })

// Filter by child element
Page.GetByRole(AriaRole.Listitem).Filter(new() { Has = Page.GetByRole(AriaRole.Button) })

// Filter out items
Page.GetByRole(AriaRole.Listitem).Filter(new() { HasNot = Page.GetByText("Out of stock") })

// Combining filters (chain)
Page.GetByRole(AriaRole.Row)
    .Filter(new() { HasText = "Jane" })
    .Filter(new() { Has = Page.GetByRole(AriaRole.Checkbox, new() { Checked = true }) })
```

### 3.4 Combining locators

```csharp
// .And() — element must match BOTH locators
Page.GetByRole(AriaRole.Button).And(Page.GetByText("Submit"))

// .Or() — element matches EITHER locator
Page.GetByRole(AriaRole.Button, new() { Name = "OK" })
    .Or(Page.GetByRole(AriaRole.Button, new() { Name = "Accept" }))
```

### 3.5 Chaining / scoping

```csharp
// Scope: narrow to a parent element first
var form    = Page.Locator("form#login");
var emailFld = form.GetByLabel("Email");      // searches only inside form
var submitBtn = form.GetByRole(AriaRole.Button, new() { Name = "Submit" });
```

### 3.6 Indexing into multiple matches

```csharp
Page.GetByRole(AriaRole.Listitem).First         // first match
Page.GetByRole(AriaRole.Listitem).Last          // last match
Page.GetByRole(AriaRole.Listitem).Nth(2)        // 0-based index

// Get all matches as a list
var items = await Page.GetByRole(AriaRole.Listitem).AllAsync();
foreach (var item in items)
    Console.WriteLine(await item.TextContentAsync());
```

### 3.7 Stability ranking (best → worst)

1. `GetByRole` (semantic + stable)
2. `GetByLabel` / `GetByPlaceholder`
3. `GetByTestId` (requires `data-testid` in app)
4. `GetByText` / `GetByAltText` / `GetByTitle`
5. CSS attribute selectors `[data-cy=...]`
6. CSS class / id
7. XPath (brittle, last resort)

---

## 4. Actions

All actions **auto-wait** for actionability before executing.

### 4.1 Click and keyboard input

```csharp
await locator.ClickAsync();
await locator.ClickAsync(new() { Button = MouseButton.Right });
await locator.ClickAsync(new() { Modifiers = new[] { KeyboardModifier.Shift } }); // Shift+click
await locator.ClickAsync(new() { Modifiers = new[] { KeyboardModifier.Control } }); // Ctrl+click
await locator.ClickAsync(new() { ClickCount = 2 });   // double-click (equivalent to DblClickAsync)
await locator.DblClickAsync();
await locator.ClickAsync(new() { Position = new() { X = 10, Y = 5 } });  // click offset

// Text input
await locator.FillAsync("text");                  // clears first, then types at once
await locator.PressSequentiallyAsync("text");     // key-by-key (triggers keydown/keypress/keyup)
await locator.PressAsync("Enter");                // single key press
await locator.PressAsync("Control+A");            // modifier combo
await locator.ClearAsync();                       // clear input
```

**FillAsync vs PressSequentiallyAsync:**
- `FillAsync` — sets value directly, fastest, no key events.
- `PressSequentiallyAsync` — fires all keyboard events; use for autocomplete, masking.

### 4.2 Hover and drag

```csharp
await locator.HoverAsync();
await locator.HoverAsync(new() { Position = new() { X = 50, Y = 20 } });

// Drag using Locator.DragToAsync
await Page.Locator("#source").DragToAsync(Page.Locator("#target"));
await Page.Locator("#source").DragToAsync(Page.Locator("#target"),
    new() { SourcePosition = new() { X = 5, Y = 5 },
            TargetPosition = new() { X = 50, Y = 50 } });
```

### 4.3 Form controls

```csharp
// Select dropdown
await locator.SelectOptionAsync("value");
await locator.SelectOptionAsync(new[] { "a", "b" });                    // multi-select by value
await locator.SelectOptionAsync(new SelectOptionValue { Label = "Blue" });
await locator.SelectOptionAsync(new SelectOptionValue { Index = 2 });

// Checkbox / radio
await locator.CheckAsync();
await locator.UncheckAsync();
await locator.SetCheckedAsync(true);
bool isChecked = await locator.IsCheckedAsync();

// Focus / blur
await locator.FocusAsync();
await locator.BlurAsync();
```

### 4.4 Scroll

```csharp
// Scroll element into view
await locator.ScrollIntoViewIfNeededAsync();

// Wheel scroll on page
await Page.Mouse.WheelAsync(0, 500);

// Wheel scroll on locator
await locator.WheelAsync(0, 300);
```

### 4.5 Touch

```csharp
// Requires HasTouch = true on context
await locator.TapAsync();
await locator.TapAsync(new() { Position = new() { X = 10, Y = 10 } });
```

### 4.6 Actionability checks (auto-wait prerequisites)

| Check | Meaning |
|---|---|
| Attached | element is in the DOM |
| Visible | not `display:none`, `visibility:hidden`, zero size |
| Stable | not animating / moving |
| Enabled | not disabled |
| Editable | not readonly (for fill) |
| Receives events | not covered by another element |

---

## 5. Assertions (Expect)

Assertions **auto-retry** (default timeout 5 s) until the condition is met or timeout expires.

### 5.1 Page assertions

```csharp
await Expect(Page).ToHaveURLAsync("https://example.com/app");
await Expect(Page).ToHaveURLAsync(new Regex(".*/dashboard"));
await Expect(Page).ToHaveTitleAsync("Dashboard");
await Expect(Page).ToHaveTitleAsync(new Regex(".*Dashboard.*"));
```

### 5.2 Visibility and presence

```csharp
await Expect(locator).ToBeVisibleAsync();
await Expect(locator).ToBeHiddenAsync();
await Expect(locator).ToBeAttachedAsync();      // in DOM, may be hidden
await Expect(locator).Not.ToBeVisibleAsync();   // negation via .Not.
```

### 5.3 Text content

```csharp
await Expect(locator).ToHaveTextAsync("exact text");
await Expect(locator).ToHaveTextAsync(new Regex(".*partial.*"));
await Expect(locator).ToContainTextAsync("partial");
await Expect(locator).ToHaveInnerHTMLAsync("<b>bold</b>");
```

### 5.4 Input state

```csharp
await Expect(locator).ToHaveValueAsync("expected value");
await Expect(locator).ToBeCheckedAsync();
await Expect(locator).Not.ToBeCheckedAsync();
await Expect(locator).ToBeEditableAsync();
await Expect(locator).ToBeEnabledAsync();
await Expect(locator).ToBeDisabledAsync();
await Expect(locator).ToBeEmptyAsync();         // empty input / no children
await Expect(locator).ToBeFocusedAsync();
```

### 5.5 Attribute and CSS

```csharp
await Expect(locator).ToHaveAttributeAsync("href", "https://example.com");
await Expect(locator).ToHaveAttributeAsync("aria-label", new Regex("close.*"));
await Expect(locator).ToHaveClassAsync("active");
await Expect(locator).ToHaveCSSAsync("color", "rgb(255, 0, 0)");
await Expect(locator).ToHaveIdAsync("submit-btn");
```

### 5.6 Count and selection

```csharp
await Expect(locator).ToHaveCountAsync(5);
await Expect(locator).ToHaveValuesAsync(new[] { "a", "b" });  // multi-select
```

### 5.7 ARIA and JS property

```csharp
await Expect(locator).ToHaveAccessibleNameAsync("Submit");
await Expect(locator).ToHaveAccessibleDescriptionAsync("Description");
await Expect(locator).ToHaveRoleAsync(AriaRole.Button);
await Expect(locator).ToHaveJSPropertyAsync("checked", true);
```

### 5.8 Soft assertions — continue after failure

```csharp
// Collect failures, assert all at end
var softAssert = new SoftAssertions();
await softAssert.Expect(locator1).ToBeVisibleAsync();
await softAssert.Expect(locator2).ToHaveTextAsync("foo");
softAssert.AssertAll();   // throws if any failed
```

### 5.9 API response assertions

```csharp
var response = await Page.APIRequest.GetAsync("https://api.example.com/data");
await Expect(response).ToBeOKAsync();                // status 200-299
```

---

## 6. Waits and Auto-Waiting

### 6.1 Auto-waiting (built-in — prefer this)

Every action (`ClickAsync`, `FillAsync`, etc.) and assertion (`Expect(…).To*Async`) automatically waits for:
- Element to be attached, visible, stable, enabled, editable, receives events
- Default timeout: 30 s for actions, 5 s for assertions (configurable)

### 6.2 Explicit wait methods

```csharp
// Wait for a URL pattern (navigation)
await Page.WaitForURLAsync("**/dashboard");
await Page.WaitForURLAsync(new Regex(".*/users/\\d+"));

// Wait for load state
await Page.WaitForLoadStateAsync();                         // default = Load
await Page.WaitForLoadStateAsync(LoadState.NetworkIdle);
await Page.WaitForLoadStateAsync(LoadState.DOMContentLoaded);

// Wait for a specific selector to appear
await Page.WaitForSelectorAsync(".modal");
await Page.WaitForSelectorAsync(".spinner", new() { State = WaitForSelectorState.Hidden });

// Wait for a JS expression to return truthy
await Page.WaitForFunctionAsync("document.readyState === 'complete'");
await Page.WaitForFunctionAsync("() => window.__loaded === true");

// Wait for a locator condition
await locator.WaitForAsync();                               // visible (default)
await locator.WaitForAsync(new() { State = WaitForSelectorState.Hidden });
```

### 6.3 Anti-pattern: WaitForTimeoutAsync

```csharp
// AVOID — fixed sleep; makes tests slow and flaky
await Page.WaitForTimeoutAsync(3000);

// USE INSTEAD — wait for a real condition
await Expect(locator).ToBeVisibleAsync();
await Page.WaitForLoadStateAsync(LoadState.NetworkIdle);
```

---

## 7. Network Interception

### 7.1 RouteAsync — intercept requests

```csharp
// Intercept by glob
await Page.RouteAsync("**/api/users", async route =>
{
    await route.FulfillAsync(new()
    {
        Status      = 200,
        ContentType = "application/json",
        Body        = """[{"id":1,"name":"Alice"}]"""
    });
});

// Intercept by Regex
await Page.RouteAsync(new Regex(".*/api/.*"), async route =>
{
    await route.AbortAsync();               // block the request
});

// Intercept by predicate (lambda on request object)
await Page.RouteAsync(r => r.Url.Contains("/api/"), async route =>
{
    await route.ContinueAsync();            // pass through unchanged
});
```

### 7.2 FulfillAsync options

```csharp
await route.FulfillAsync(new()
{
    Status        = 200,
    ContentType   = "application/json",
    Body          = """{"key":"value"}""",
    Headers       = new Dictionary<string, string> { ["X-Custom"] = "test" },
    Path          = "fixtures/response.json",   // load body from file
});
```

### 7.3 ContinueAsync — modify request before forwarding

```csharp
await Page.RouteAsync("**/api/**", async route =>
{
    await route.ContinueAsync(new()
    {
        Method  = "POST",
        Headers = new Dictionary<string, string>(route.Request.Headers)
                  { ["Authorization"] = "Bearer token" },
        PostData = """{"key":"modified"}"""
    });
});
```

### 7.4 FetchAsync — modify real response

```csharp
await Page.RouteAsync("**/api/data", async route =>
{
    var realResponse = await route.FetchAsync();           // make actual network call
    var json = await realResponse.JsonAsync<JsonElement>();
    // modify json …
    await route.FulfillAsync(new()
    {
        Response = realResponse,
        Body     = modifiedJson
    });
});
```

### 7.5 WaitForRequestAsync / WaitForResponseAsync

```csharp
// Task-first, action-second pattern
var requestTask  = Page.WaitForRequestAsync("**/api/submit");
var responseTask = Page.WaitForResponseAsync("**/api/submit");

await Page.GetByRole(AriaRole.Button, new() { Name = "Save" }).ClickAsync();

var request  = await requestTask;
var response = await responseTask;

Console.WriteLine(request.PostData);
Console.WriteLine(response.Status);
Console.WriteLine(await response.TextAsync());
```

### 7.6 Event listeners for all requests

```csharp
Page.Request  += (_, req)  => Console.WriteLine($">> {req.Method} {req.Url}");
Page.Response += (_, resp) => Console.WriteLine($"<< {resp.Status} {resp.Url}");
Page.RequestFailed += (_, req) => Console.WriteLine($"!! {req.Url}");
```

### 7.7 AbortAsync — error categories

```csharp
await route.AbortAsync("blockedbyclient");  // or "connectionrefused", "timedout", "failed"
```

---

## 8. API Testing

### 8.1 Create an API request context

```csharp
// Standalone — no browser needed
using var playwright   = await Playwright.CreateAsync();
var apiContext = await playwright.APIRequest.NewContextAsync(new()
{
    BaseURL         = "https://api.example.com",
    ExtraHTTPHeaders = new Dictionary<string, string>
    {
        ["x-auth-token"] = token,
        ["Accept"]       = "application/json"
    }
});
```

### 8.2 HTTP methods

```csharp
// GET
var resp = await apiContext.GetAsync("/users");
Assert.That(resp.Ok, Is.True);
var body = await resp.JsonAsync<List<UserDto>>();

// POST — JSON body
var resp = await apiContext.PostAsync("/users", new APIRequestContextOptions
{
    DataObject = new { name = "Alice", email = "alice@example.com" }
});

// POST — form-encoded
var resp = await apiContext.PostAsync("/users/login", new APIRequestContextOptions
{
    Form = new Dictionary<string, string>
    {
        ["email"]    = "user@example.com",
        ["password"] = "secret"
    }
});

// PATCH / PUT / DELETE
var resp = await apiContext.PatchAsync("/users/1",   new APIRequestContextOptions { DataObject = update });
var resp = await apiContext.PutAsync("/users/1",     new APIRequestContextOptions { DataObject = full });
var resp = await apiContext.DeleteAsync("/users/1");
```

### 8.3 Reading response

```csharp
int    status  = resp.Status;
bool   ok      = resp.Ok;                         // true if 200-299
string text    = await resp.TextAsync();
byte[] bytes   = await resp.BodyAsync();
var    json    = await resp.JsonAsync<MyDto>();
var    headers = resp.Headers;
```

### 8.4 Use Page's request context (shares cookies)

```csharp
// Uses same cookies/session as the browser page
var resp = await Page.APIRequest.GetAsync("https://api.example.com/me");
```

---

## 9. HAR Recording and Replay

### 9.1 Record HAR

```csharp
var context = await Browser.NewContextAsync(new()
{
    RecordHarPath = "network.har",
    RecordHarUrlFilter = "**/api/**"   // optional — record only matching URLs
});
// run tests …
await context.CloseAsync();   // HAR file is written on close
```

### 9.2 Replay from HAR

```csharp
await Context.RouteFromHARAsync("network.har", new()
{
    Url      = "**/api/**",
    NotFound = HarNotFound.Fallback,  // Fallback = go to real network | Abort = fail request
    Update   = false                  // false = replay mode
});
```

### 9.3 Update (re-record) HAR

```csharp
await Context.RouteFromHARAsync("network.har", new()
{
    Url    = "**/api/**",
    Update = true   // overwrite HAR with fresh recorded traffic
});
```

---

## 10. Screenshots and Visual Regression

### 10.1 Full-page and element screenshots

```csharp
// Full page
await Page.ScreenshotAsync(new() { Path = "full.png", FullPage = true });

// Viewport only (default)
await Page.ScreenshotAsync(new() { Path = "viewport.png" });

// Element only
await locator.ScreenshotAsync(new() { Path = "element.png" });

// As byte array (no save)
byte[] png = await Page.ScreenshotAsync();
```

### 10.2 Visual regression — ToHaveScreenshotAsync

```csharp
// First run (no baseline): creates .png baseline in __snapshots__/
await Expect(Page).ToHaveScreenshotAsync("homepage.png");

// Subsequent runs: compare against baseline
await Expect(Page).ToHaveScreenshotAsync("homepage.png", new()
{
    MaxDiffPixels      = 100,        // allow up to 100 pixels different
    MaxDiffPixelRatio  = 0.01,       // OR allow up to 1% of pixels different
    Threshold          = 0.2,        // per-pixel tolerance (0-1)
    Animations         = ScreenshotAnimations.Disabled,
    Mask               = new[] { Page.Locator(".timestamp") }  // mask dynamic areas
});

// Element screenshot comparison
await Expect(locator).ToHaveScreenshotAsync("button.png");
```

### 10.3 Update baseline snapshots

```bash
dotnet test -- Playwright.UpdateSnapshots=true
```

---

## 11. JavaScript Evaluation

### 11.1 EvaluateAsync — returns value

```csharp
// Returns a C# value
var title   = await Page.EvaluateAsync<string>("document.title");
var count   = await Page.EvaluateAsync<int>("document.querySelectorAll('li').length");
var visible = await Page.EvaluateAsync<bool>("() => document.querySelector('.modal') !== null");

// Pass arguments to script
var text = await Page.EvaluateAsync<string>("selector => document.querySelector(selector).textContent",
    "#username");

// Multi-line with arrow function
var result = await Page.EvaluateAsync<string>(@"([sel, attr]) => {
    const el = document.querySelector(sel);
    return el ? el.getAttribute(attr) : null;
}", new[] { ".btn", "data-id" });
```

### 11.2 EvaluateHandleAsync — returns IJSHandle

```csharp
// Use when you need to pass a JS object back to another evaluate
var jsHandle = await Page.EvaluateHandleAsync("() => window");
```

### 11.3 Locator.EvaluateAsync — run JS on a specific element

```csharp
// element is the resolved DOM node
var classes  = await locator.EvaluateAsync<string>("el => el.className");
var rect     = await locator.EvaluateAsync<JsonElement>("el => el.getBoundingClientRect()");
var value    = await locator.EvaluateAsync<string>("el => el.value");

// Set a property
await locator.EvaluateAsync("el => el.value = 'new value'");
await locator.EvaluateAsync("el => el.classList.add('active')");
await locator.EvaluateAsync("el => el.style.border = '2px solid red'");
```

### 11.4 localStorage and sessionStorage

```csharp
// Read
var token = await Page.EvaluateAsync<string>("localStorage.getItem('auth_token')");
var data  = await Page.EvaluateAsync<string>("sessionStorage.getItem('session_id')");

// Write
await Page.EvaluateAsync("localStorage.setItem('key', 'value')");

// Clear
await Page.EvaluateAsync("localStorage.clear()");

// Pre-fill before page load (via AddInitScriptAsync)
await Context.AddInitScriptAsync("localStorage.setItem('feature_flag', 'true')");
```

### 11.5 DOM queries (legacy — prefer Locators)

```csharp
var el = await Page.QuerySelectorAsync("input#email");   // nullable IElementHandle
if (el != null) await el.FillAsync("test@test.com");

// ElementHandle methods
await el.ClickAsync();
var text = await el.TextContentAsync();
```

---

## 12. Keyboard

### 12.1 Key press on locator (preferred)

```csharp
await locator.PressAsync("Enter");
await locator.PressAsync("Tab");
await locator.PressAsync("Escape");
await locator.PressAsync("Control+A");
await locator.PressAsync("Shift+Tab");
await locator.PressAsync("ArrowDown");
```

### 12.2 Page.Keyboard — low-level control

```csharp
// Hold key, perform action, release
await Page.Keyboard.DownAsync("Shift");
await Page.GetByRole(AriaRole.Link, new() { Name = "Item" }).ClickAsync();
await Page.Keyboard.UpAsync("Shift");

// Type characters (fires keydown/keypress/keyup per key)
await Page.Keyboard.TypeAsync("Hello World");

// Insert text at once (no key events)
await Page.Keyboard.InsertTextAsync("émoji 🎉");

// Single key press with modifier
await Page.Keyboard.PressAsync("Control+Z");   // Ctrl+Z undo
await Page.Keyboard.PressAsync("Meta+A");      // Cmd+A on Mac
```

### 12.3 Common key names

`Enter` · `Tab` · `Escape` · `Backspace` · `Delete` · `ArrowUp/Down/Left/Right`  
`Home` · `End` · `PageUp` · `PageDown` · `F1`–`F12`  
`Control` · `Shift` · `Alt` · `Meta`

---

## 13. Mouse

### 13.1 High-level drag (prefer over low-level)

```csharp
await Page.Locator("#draggable").DragToAsync(Page.Locator("#droppable"));
```

### 13.2 Page.Mouse — low-level control

```csharp
// Move to coordinates
await Page.Mouse.MoveAsync(100, 200);

// Click at absolute coordinates
await Page.Mouse.ClickAsync(100, 200);
await Page.Mouse.ClickAsync(100, 200, new() { Button = MouseButton.Right });

// Double-click
await Page.Mouse.DblClickAsync(100, 200);

// Hold button for drag
await Page.Mouse.MoveAsync(100, 200);
await Page.Mouse.DownAsync();
await Page.Mouse.MoveAsync(300, 400);
await Page.Mouse.UpAsync();

// Wheel scroll
await Page.Mouse.WheelAsync(0, 500);    // deltaX, deltaY
```

---

## 14. Dialogs

**Critical rule: register the handler BEFORE the action that triggers the dialog.**  
An unhandled dialog blocks page execution indefinitely.

### 14.1 Event handler pattern

```csharp
// Register before trigger
Page.Dialog += async (_, dialog) =>
{
    // Read dialog properties
    string type    = dialog.Type;          // "alert" | "confirm" | "prompt" | "beforeunload"
    string message = dialog.Message;
    string defVal  = dialog.DefaultValue;  // prompt only

    // Respond
    await dialog.AcceptAsync();            // OK button
    await dialog.AcceptAsync("text");      // prompt: submit this text
    await dialog.DismissAsync();           // Cancel button
};

// Trigger
await Page.EvaluateAsync("() => alert('Hello')");
```

### 14.2 Dialog types

| Type | `AcceptAsync` result | `DismissAsync` result |
|---|---|---|
| `alert` | closes | closes |
| `confirm` | JS gets `true` | JS gets `false` |
| `prompt` | JS gets typed string | JS gets `null` |
| `beforeunload` | navigation proceeds | navigation cancelled |

### 14.3 RunAndWaitForDialog (one-shot)

```csharp
var dialog = await Page.RunAndWaitForDialogAsync(
    async () => await Page.GetByRole(AriaRole.Button, new() { Name = "Delete" }).ClickAsync()
);
Assert.That(dialog.Type, Is.EqualTo("confirm"));
await dialog.AcceptAsync();
```

---

## 15. Multi-Tab / Popups

### 15.1 Open second tab programmatically

```csharp
var tab2 = await Context.NewPageAsync();
await tab2.GotoAsync("https://example.com/other");
Assert.That(Context.Pages.Count, Is.EqualTo(2));
await tab2.CloseAsync();
```

### 15.2 WaitForPopupAsync — task-first pattern

```csharp
// 1. Register the wait BEFORE the action
var popupTask = Page.WaitForPopupAsync();

// 2. Trigger the popup
await Page.GetByRole(AriaRole.Link, new() { Name = "Open" }).ClickAsync();

// 3. Await the result
var popup = await popupTask;
await popup.WaitForLoadStateAsync();
Assert.That(popup.Url, Does.Contain("target"));
```

### 15.3 RunAndWaitForPopupAsync — lambda pattern

```csharp
var popup = await Page.RunAndWaitForPopupAsync(async () =>
    await Page.GetByRole(AriaRole.Link, new() { Name = "Open" }).ClickAsync()
);
await popup.WaitForLoadStateAsync();
```

### 15.4 Context.WaitForPageAsync

```csharp
var pageTask = Context.WaitForPageAsync();
await Page.ClickAsync("#open-app-button");
var newPage = await pageTask;
```

### 15.5 Context.Page event

```csharp
IPage? capturedPage = null;
Context.Page += (_, newPage) => capturedPage = newPage;  // fires for every new page

var tab2 = await Context.NewPageAsync();
Assert.That(capturedPage, Is.Not.Null);
```

### 15.6 BringToFrontAsync — switch focus

```csharp
await tab1.BringToFrontAsync();   // make tab1 the active foreground tab
await tab2.BringToFrontAsync();
```

### 15.7 Context.Pages — list all tabs

```csharp
IReadOnlyList<IPage> pages = Context.Pages;
Console.WriteLine($"Open tabs: {pages.Count}");
```

---

## 16. Downloads

### 16.1 RunAndWaitForDownloadAsync

```csharp
var download = await Page.RunAndWaitForDownloadAsync(async () =>
    await Page.GetByRole(AriaRole.Button, new() { Name = "Download" }).ClickAsync()
);

// Filename suggested by the server
string filename = download.SuggestedFilename;

// Save to a specific path
await download.SaveAsAsync($"downloads/{filename}");

// Get the temp path (valid until download.DeleteAsync or context close)
string? tmpPath = await download.PathAsync();

// Check for failure
string? error = await download.FailureAsync();
Assert.That(error, Is.Null);
```

### 16.2 WaitForDownloadAsync — task-first pattern

```csharp
var downloadTask = Page.WaitForDownloadAsync();
await Page.GetByRole(AriaRole.Button, new() { Name = "Export" }).ClickAsync();
var download = await downloadTask;
```

---

## 17. File Upload

### 17.1 `<input type="file">` — direct

```csharp
// Single file
await Page.GetByLabel("Upload file").SetInputFilesAsync("path/to/file.pdf");

// Multiple files
await Page.Locator("input[type='file']").SetInputFilesAsync(new[]
{
    "file1.csv",
    "file2.csv"
});

// Clear selection
await locator.SetInputFilesAsync(new string[] { });
```

### 17.2 Custom upload button — RunAndWaitForFileChooserAsync

```csharp
var fileChooser = await Page.RunAndWaitForFileChooserAsync(async () =>
    await Page.GetByRole(AriaRole.Button, new() { Name = "Choose File" }).ClickAsync()
);

await fileChooser.SetFilesAsync("upload/document.pdf");
```

### 17.3 WaitForFileChooserAsync — task-first pattern

```csharp
var chooserTask = Page.WaitForFileChooserAsync();
await Page.ClickAsync("#upload-btn");
var chooser = await chooserTask;
await chooser.SetFilesAsync("doc.pdf");
```

---

## 18. Emulation

### 18.1 Device presets

```csharp
var device = Playwright.Devices["iPhone 13"];   // also "Pixel 5", "iPad Pro 11", etc.

var context = await Browser.NewContextAsync(new BrowserNewContextOptions
{
    UserAgent       = device.UserAgent,
    ViewportSize    = device.ViewportSize,
    DeviceScaleFactor = device.DeviceScaleFactor,
    IsMobile        = device.IsMobile,
    HasTouch        = device.HasTouch,
});
```

### 18.2 BrowserNewContextOptions — all emulation settings

```csharp
var context = await Browser.NewContextAsync(new BrowserNewContextOptions
{
    ViewportSize      = new ViewportSize { Width = 375, Height = 812 },
    IsMobile          = true,
    HasTouch          = true,
    DeviceScaleFactor = 3,
    UserAgent         = "Mozilla/5.0 (iPhone; ...)",
    ColorScheme       = ColorScheme.Dark,          // Light | Dark | NoPreference
    Locale            = "fr-FR",
    TimezoneId        = "Europe/Paris",
    Offline           = false,
    Geolocation       = new Geolocation { Latitude = 51.5, Longitude = -0.1 },
    Permissions       = new[] { "geolocation" },
    StorageStatePath  = "playwright/.auth/user.json",
    RecordHarPath     = "network.har",
});
```

### 18.3 Change emulation during a test

```csharp
// Resize viewport
await Page.SetViewportSizeAsync(1280, 720);

// Change media features (dark/light, print)
await Page.EmulateMediaAsync(new() { ColorScheme = ColorScheme.Dark });
await Page.EmulateMediaAsync(new() { Media = Media.Print });

// Change geolocation
await Context.SetGeolocationAsync(new() { Latitude = 48.8566, Longitude = 2.3522 });
await Context.GrantPermissionsAsync(new[] { "geolocation" });

// Simulate offline
await Context.SetOfflineAsync(true);
await Context.SetOfflineAsync(false);
```

---

## 19. Authentication and Storage State

### 19.1 Save and reuse auth

```csharp
// After successful login — save to file
await Context.StorageStateAsync(new() { Path = "playwright/.auth/user.json" });

// Load at context creation
public override BrowserNewContextOptions ContextOptions() => new()
{
    StorageStatePath = "playwright/.auth/user.json"
};
```

> `StorageStateAsync` saves **cookies + localStorage**. `sessionStorage` is NOT included.

### 19.2 Multiple roles

```csharp
// GlobalSetup — create auth files per role
await SaveStorageState(adminEmail, adminPass, "playwright/.auth/admin.json");
await SaveStorageState(userEmail,  userPass,  "playwright/.auth/user.json");

// In test class — load per role
public override BrowserNewContextOptions ContextOptions() => new()
{
    StorageStatePath = "playwright/.auth/admin.json"
};
```

### 19.3 Auth types with IAPIRequestContext

| Auth Type | How to set |
|---|---|
| JWT / Bearer | `ExtraHTTPHeaders["Authorization"] = "Bearer <token>"` |
| Basic Auth | `HttpCredentials = new() { Username = "u", Password = "p" }` |
| API Key | `ExtraHTTPHeaders["x-api-key"] = key` |
| Cookie | `Context.AddCookiesAsync(...)` |
| NTLM | `HttpCredentials = new() { Username = ..., Domain = "corp" }` |

```csharp
// Bearer token via API context
var apiContext = await Playwright.APIRequest.NewContextAsync(new()
{
    BaseURL = "https://api.example.com",
    ExtraHTTPHeaders = new Dictionary<string, string>
    {
        ["Authorization"] = $"Bearer {token}"
    }
});

// Basic auth via context options
var context = await Browser.NewContextAsync(new()
{
    HttpCredentials = new HttpCredentials { Username = "admin", Password = "secret" }
});
```

### 19.4 Persistent context (keeps state between runs)

```csharp
var context = await playwright.Chromium.LaunchPersistentContextAsync(
    "C:/user-data",    // profile directory — state persists on disk
    new() { Headless = false }
);
var page = await context.NewPageAsync();
```

---

## 20. Tracing

### 20.1 Code-level tracing

```csharp
// Start
await Context.Tracing.StartAsync(new()
{
    Screenshots = true,   // screenshot at each action
    Snapshots   = true,   // DOM snapshot at each step
    Sources     = true    // embed C# source files
});

// Run test …

// Stop and save
await Context.Tracing.StopAsync(new() { Path = "trace.zip" });
```

### 20.2 Chunk tracing (multi-section)

```csharp
await Context.Tracing.StartAsync(new() { Screenshots = true, Snapshots = true });

await Context.Tracing.StartChunkAsync(new() { Title = "login" });
// … login actions …
await Context.Tracing.StopChunkAsync(new() { Path = "login.zip" });

await Context.Tracing.StartChunkAsync(new() { Title = "checkout" });
// … checkout actions …
await Context.Tracing.StopChunkAsync(new() { Path = "checkout.zip" });
```

### 20.3 runsettings / CLI

```xml
<!-- playwright.runsettings -->
<TestOptions>
  <Tracing>retain-on-failure</Tracing>
</TestOptions>
```

```bash
dotnet test -- Playwright.TestOptions.Tracing=retain-on-failure
# values: off | on | retain-on-failure | on-all-retries
```

### 20.4 View a trace

```bash
pwsh bin/Debug/net8.0/playwright.ps1 show-trace trace.zip
```

---

## 21. IBrowserContext Methods

### 21.1 Route methods

```csharp
// Intercept across ALL pages in this context
await Context.RouteAsync("**/api/**", async route => await route.FulfillAsync(new() { ... }));

// Replay from HAR
await Context.RouteFromHARAsync("network.har", new() { Url = "**/api/**", Update = false });

// Remove routes
await Context.UnrouteAsync("**/api/**");
await Context.UnrouteAllAsync();
```

### 21.2 Cookies

```csharp
var cookies = await Context.CookiesAsync();                   // all cookies
var cookies = await Context.CookiesAsync(new[] { "https://example.com" }); // filtered

await Context.AddCookiesAsync(new[]
{
    new Cookie { Name = "session", Value = "abc", Domain = "example.com", Path = "/" }
});

await Context.ClearCookiesAsync();
```

### 21.3 Network — headers and offline

```csharp
await Context.SetExtraHTTPHeadersAsync(new Dictionary<string, string>
{
    ["Authorization"] = "Bearer token",
    ["x-api-key"]     = "key"
});

await Context.SetOfflineAsync(true);
await Context.SetOfflineAsync(false);
```

### 21.4 Geolocation and permissions

```csharp
await Context.SetGeolocationAsync(new() { Latitude = 51.5074, Longitude = -0.1278 });
await Context.GrantPermissionsAsync(new[] { "geolocation" });
await Context.GrantPermissionsAsync(new[] { "camera", "microphone" }, origin: "https://example.com");
await Context.ClearPermissionsAsync();
```

### 21.5 Script injection

```csharp
// Runs before EVERY page load in this context
await Context.AddInitScriptAsync("window.__testMode = true;");
await Context.AddInitScriptAsync(scriptPath: "inject.js");
```

### 21.6 StorageState and close

```csharp
await Context.StorageStateAsync(new() { Path = "playwright/.auth/user.json" });
await Context.CloseAsync();   // trace and HAR are written on close
```

### 21.7 Context vs Page route scope

| | `Page.RouteAsync` | `Context.RouteAsync` |
|---|---|---|
| Scope | one tab only | all pages in context |
| Use for | single-page intercept | multi-tab / shared mock |

---

## 22. Frames and iFrames

### 22.1 FrameLocator (recommended)

```csharp
// By CSS selector for the iframe element
var frame  = Page.FrameLocator("iframe#editor");
var button = frame.GetByRole(AriaRole.Button, new() { Name = "Save" });
await button.ClickAsync();

// Nested frames
var outer = Page.FrameLocator("#outer-frame");
var inner = outer.FrameLocator("#inner-frame");
await inner.GetByLabel("Email").FillAsync("test@test.com");
```

### 22.2 IFrame (legacy — prefer FrameLocator)

```csharp
var frame = Page.Frame("frameName");                          // by name attribute
var frame = Page.FrameByUrl("**/embedded*");                  // by URL glob
var frame = Page.Frames[1];                                   // by index

// Navigate and interact
await frame.GotoAsync("https://example.com");
await frame.ClickAsync("button.submit");
var text = await frame.TextContentAsync("h1");
```

### 22.3 Shadow DOM

```csharp
// CSS selectors automatically pierce Shadow DOM
await Page.Locator("my-component button").ClickAsync();

// Explicit pierce operator
await Page.Locator("my-custom-element >> css=button").ClickAsync();
```

---

## 23. Page Object Model (POM)

### 23.1 BaseTest

```csharp
public class BaseTest : PageTest
{
    public override BrowserNewContextOptions ContextOptions() => new()
    {
        StorageStatePath = "playwright/.auth/user.json"
    };
}
```

### 23.2 BasePage

```csharp
public abstract class BasePage
{
    protected readonly IPage Page;
    protected BasePage(IPage page) => Page = page;
}
```

### 23.3 Page object — locators + action methods

```csharp
public class LoginPage : BasePage
{
    // Private locators — never exposed outside this class
    private ILocator EmailInput    => Page.GetByLabel("Email");
    private ILocator PasswordInput => Page.GetByLabel("Password");
    private ILocator SubmitButton  => Page.GetByRole(AriaRole.Button, new() { Name = "Login" });
    private ILocator ErrorMessage  => Page.GetByTestId("error-message");

    public LoginPage(IPage page) : base(page) { }

    // Navigation returns next page object (page chaining)
    public async Task<NotesPage> LoginAsync(string email, string password)
    {
        await EmailInput.FillAsync(email);
        await PasswordInput.FillAsync(password);
        await SubmitButton.ClickAsync();
        await Page.WaitForURLAsync("**/app");
        return new NotesPage(Page);
    }

    // Methods that don't navigate return Task or Task<T>
    public async Task<string> GetErrorMessageAsync() =>
        await ErrorMessage.TextContentAsync() ?? "";
}
```

### 23.4 Page chaining in tests

```csharp
[Test]
public async Task CreateNote_Success()
{
    var loginPage    = new LoginPage(Page);
    var notesPage    = await loginPage.LoginAsync(email, password);
    var noteFormPage = await notesPage.ClickAddNoteAsync();
    await noteFormPage.FillTitleAsync("My Note");
    await noteFormPage.FillDescriptionAsync("Content here");
    notesPage = await noteFormPage.SaveAsync();
    await Expect(notesPage.GetNoteCard("My Note")).ToBeVisibleAsync();
}
```

### 23.5 Return type guide

| Scenario | Return type |
|---|---|
| Navigate to next page | `Task<NextPage>` |
| Get text or value | `Task<string>` |
| Get state (checked, visible) | `Task<bool>` |
| Get count | `Task<int>` |
| Just action (fill, click) | `Task` |
| Get locator (for assertions) | `ILocator` (sync property) |

### 23.6 Where assertions live

- **In page objects**: only when the assertion is intrinsic to the action (e.g., wait for URL after navigation).
- **In test methods**: all business-logic and result assertions.

---

## 24. Configuration, Parallelism, and CI/CD

### 24.1 runsettings file

```xml
<?xml version="1.0" encoding="utf-8"?>
<RunSettings>
  <Playwright>
    <BrowserName>chromium</BrowserName>         <!-- chromium | firefox | webkit -->
    <LaunchOptions>
      <Headless>true</Headless>
      <SlowMo>0</SlowMo>
      <Channel>chrome</Channel>                 <!-- use installed Chrome -->
    </LaunchOptions>
    <TestOptions>
      <Tracing>retain-on-failure</Tracing>      <!-- off|on|retain-on-failure|on-all-retries -->
      <Screenshot>only-on-failure</Screenshot>  <!-- off|on|only-on-failure -->
      <Video>retain-on-failure</Video>
    </TestOptions>
  </Playwright>
  <NUnit>
    <NumberOfTestWorkers>4</NumberOfTestWorkers>
  </NUnit>
</RunSettings>
```

### 24.2 CLI overrides — dotnet test flags

```bash
dotnet test -- Playwright.BrowserName=firefox
dotnet test -- Playwright.LaunchOptions.Headless=true
dotnet test -- Playwright.LaunchOptions.SlowMo=500
dotnet test -- Playwright.LaunchOptions.Channel=chrome
dotnet test -- Playwright.TestOptions.Tracing=on
dotnet test -- Playwright.TestOptions.Screenshot=on
dotnet test -- Playwright.TestOptions.Video=on
dotnet test -- Playwright.UpdateSnapshots=true
dotnet test -- NUnit.NumberOfTestWorkers=4
```

### 24.3 Parallelism attributes

```csharp
// Run all tests in assembly in parallel (most aggressive)
[assembly: Parallelizable(ParallelScope.All)]

// Run test fixtures in parallel, tests within fixture serial
[assembly: Parallelizable(ParallelScope.Fixtures)]

// Per-class override — this fixture runs serially even if assembly is parallel
[Parallelizable(ParallelScope.None)]
public class OrderDependentTests : BaseTest { }
```

**Rule**: Each test needs its own `IPage`/`IBrowserContext` (via `PageTest` or manual `Browser.NewContextAsync`) — never share a page between parallel tests.

### 24.4 GitHub Actions CI

```yaml
name: Playwright Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: mcr.microsoft.com/playwright/dotnet:v1.61.0-noble
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: dotnet test
        env:
          NOTES_TEST_EMAIL:    ${{ secrets.TEST_EMAIL }}
          NOTES_TEST_PASSWORD: ${{ secrets.TEST_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-traces
          path: playwright/traces/
```

### 24.5 Docker

```dockerfile
FROM mcr.microsoft.com/playwright/dotnet:v1.61.0-noble
WORKDIR /app
COPY Migrated.csproj .
RUN dotnet restore
COPY . .
RUN dotnet build --no-restore --configuration Release
ENV NOTES_TEST_EMAIL=""
ENV NOTES_TEST_PASSWORD=""
ENTRYPOINT ["dotnet", "test", "--no-build", "--configuration", "Release"]
```

```bash
# Build
docker build -t playwright-tests .

# Run
docker run --rm \
  -e NOTES_TEST_EMAIL=user@example.com \
  -e NOTES_TEST_PASSWORD=secret \
  -v $(pwd)/results:/app/playwright \
  playwright-tests
```

> Docker has no display server — `Headless` must be `true` in `playwright.runsettings` or passed via `-- Playwright.LaunchOptions.Headless=true`.

### 24.6 Debugging tools

```bash
# Playwright Inspector — step through test, highlight locators
PWDEBUG=1 dotnet test --filter "TestName"

# On Windows PowerShell
$env:PWDEBUG=1; dotnet test --filter "TestName"

# Console API logging (verbose)
PWDEBUG=console dotnet test

# Slow motion — add delay between actions (ms)
dotnet test -- Playwright.LaunchOptions.SlowMo=1000

# View trace
pwsh bin/Debug/net8.0/playwright.ps1 show-trace trace.zip

# Install/reinstall browser binaries
pwsh bin/Debug/net8.0/playwright.ps1 install
pwsh bin/Debug/net8.0/playwright.ps1 install chromium
```

---

## Quick-Look Interview Cheat Cards

### Locator priority order
`GetByRole` > `GetByLabel/Placeholder` > `GetByTestId` > `GetByText/Alt/Title` > CSS attribute > CSS class/id > XPath

### Auto-wait vs explicit wait
- **Auto-wait**: built into every action and Expect assertion — no code needed.
- **Explicit**: `WaitForURLAsync`, `WaitForLoadStateAsync`, `WaitForFunctionAsync`, `WaitForSelectorAsync`.
- **Anti-pattern**: `WaitForTimeoutAsync` (fixed sleep).

### Task-first pattern (dialogs, popups, downloads)
Always register/create the wait task **before** the action that triggers the event:
```csharp
var task = Page.WaitForPopupAsync();     // register first
await trigger.ClickAsync();             // then act
var popup = await task;                  // then collect
```

Or use `RunAndWaitFor*` which wraps the pattern in a lambda.

### Dialog rule
Register `Page.Dialog +=` handler **before** the JS call that triggers it. Not handling a dialog hangs the page.

### StorageState scope
Saves cookies + localStorage. Does **not** save sessionStorage.

### Context vs Page scope

| Feature | `IBrowserContext` | `IPage` |
|---|---|---|
| RouteAsync | all tabs | one tab |
| Cookies | all tabs | via JS only |
| Headers | all tabs | - |
| Geolocation | all tabs | - |
| Dialog | - | one tab |
| URL / DOM | - | one tab |

### Parallelism gotcha
Parallel tests must not share `IPage` or `IBrowserContext`. `PageTest` creates a fresh context per test by default.

### Docker gotcha
Headed browser (`Headless=false`) crashes in Docker — no display server. Always set `Headless=true`.

### Visual regression workflow
1. First run (no baseline) → creates `.png` snapshot files.
2. Subsequent runs → compare against baseline.
3. Update baselines: `dotnet test -- Playwright.UpdateSnapshots=true`.

---
