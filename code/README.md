# Creating a Simple Test

```C#
using System.Text.RegularExpressions;
using Microsoft.Playwright;
using Microsoft.Playwright.MSTest;

namespace PlaywrightTests;

[TestClass]
public class ExampleTest : PageTest
{
    [TestMethod]
    public async Task HasTitle()
    {
        await Page.GotoAsync("https://playwright.dev");

        // Expect a title "to contain" a substring.
        await Expect(Page).ToHaveTitleAsync(new Regex("Playwright"));
    }

    [TestMethod]
    public async Task GetStartedLink()
    {
        await Page.GotoAsync("https://playwright.dev");

        // Click the get started link.
        await Page.GetByRole(AriaRole.Link, new() { Name = "Get started" }).ClickAsync();

        // Expect the URL to contain "intro"
        await Expect(Page).ToHaveURLAsync(new Regex(".*intro"));

    }
}
```

# Test Structure
```C#
using System.Text.RegularExpressions;
using Microsoft.Playwright;
using Microsoft.Playwright.MSTest;

namespace PlaywrightTests;

[TestClass]
public class LoginTest: PageTest
{
    [TestMethod]
    public async Task SuccessfulTitle()
    {
        // Navigate to the login page
        await Page.GotoAsync("https://the-internet.herokuapp.com/login");

        // Enter username
        await Page.FillAsync("input#username", "tomsmith");

        // Enter password
        await Page.FillAsync("input#password", "SuperSecretPassword!");

        // Click the login button
        await Page.ClickAsync("button[type='submit']");

        // Verify successful login message
        var flashMessage = Page.Locator("#flash");
        await Expect(flashMessage).ToContainTextAsync(new Regex("You logged into a secure area!"));

        // Verify that the URL has changed to the secure area
        await Expect(Page).ToHaveURLAsync("https://the-internet.herokuapp.com/secure");

    }
}
```

# Page Object Mode

```C#
using Microsoft.Playwright;

namespace PlaywrightTests.PageObjects;

public class LoginPage
{
    private readonly IPage _page;
    private readonly string _url = "https://the-internet.herokuapp.com/login";

    public ILocator FlashMessage => _page.Locator("#flash");

    public LoginPage(IPage page)
    {
        _page = page;
    }

    public async Task NavigateAsync()
    {
        await _page.GotoAsync(_url);
    }

    public async Task LoginAsync(string username, string password)
    {
        await _page.FillAsync("input#username", username);
        await _page.FillAsync("input#password", password);
        await _page.ClickAsync("button[type='submit']");
    }
}
```

```C#
using Microsoft.Playwright;
using Microsoft.Playwright.MSTest;
using PlaywrightTests.PageObjects;

namespace PlaywrightTests;

[TestClass]
public class LoginTests : PageTest
{
    [TestMethod]
    public async Task SuccessfulLoginUsingPOM()
    {
        // Initialize the Page Object Model
        var loginPage = new LoginPage(Page);

        // Navigate to the login page
        await loginPage.NavigateAsync();

        // Perform login
        await loginPage.LoginAsync("tomsmith", "SuperSecretPassword!");

        // Verify successful login message
        await Expect(loginPage.FlashMessage).ToContainTextAsync("You logged into a secure area!");

        // Verify that the URL has changed to the secure area
        await Expect(Page).ToHaveURLAsync("https://the-internet.herokuapp.com/secure");
    }

    [TestMethod]
    public async Task UnsuccessfulLoginUsingPOM()
    {
        // Initialize the Page Object Model
        var loginPage = new LoginPage(Page);

        // Navigate to the login page
        await loginPage.NavigateAsync();

        // Perform login with invalid credentials
        await loginPage.LoginAsync("invalidUser", "invalidPass");

        // Verify error message
        await Expect(loginPage.FlashMessage).ToContainTextAsync("Your username is invalid!");
    }
}
```

# Handling Dynamic Content
```C#
using Microsoft.Playwright;
using Microsoft.Playwright.MSTest;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace PlaywrightTests;

[TestClass]
public class DynamicContentTests : PageTest
{
    [TestMethod]
    public async Task DynamicContentLoadsCorrectly()
    {
        // Navigate to the dynamic content page with changing images
        await Page.GotoAsync("https://the-internet.herokuapp.com/dynamic_content");

        // Locate the images
        var images = Page.Locator(".large-2 img");

        // Verify that three images are loaded
        await Expect(images).ToHaveCountAsync(3);

        // Capture the sources of the images, replacing null with an empty string in JavaScript
        var imageSources = await images.EvaluateAllAsync<string[]>("imgs => imgs.map(img => img.getAttribute('src') || '')");

        // Reload the page and verify that images have changed
        await Page.ReloadAsync();

        // Re-locate the images after reload
        images = Page.Locator(".large-2 img");

        // Capture the new sources of the images
        var newImageSources = await images.EvaluateAllAsync<string[]>("imgs => imgs.map(img => img.getAttribute('src') || '')");

        // Assert that the new image sources are not the same as the original ones
        Assert.IsFalse(imageSources.SequenceEqual(newImageSources), "Image sources should change after reload.");
    }
}
```

# File Uploads and Downloads

```C#
using Microsoft.Playwright;
using Microsoft.Playwright.MSTest;
using System.IO;
using System.Threading.Tasks;

namespace PlaywrightTests;

[TestClass]
public class FileUploadTests : PageTest
{
    [TestMethod]
    public async Task FileUploadWorksCorrectly()
    {
        // Navigate to the file upload page
        await Page.GotoAsync("https://the-internet.herokuapp.com/upload");

        // Define the file path relative to the project directory
        // (tests are run from C:\Users\USER\source\repos\PlaywrightTests\PlaywrightTests\bin\Debug\net8.0\)
        var filePath = Path.Combine(Directory.GetParent(Directory.GetCurrentDirectory())!.Parent!.Parent!.FullName, "Fixtures", "test-file.txt");

        // Upload the file
        await Page.SetInputFilesAsync("input#file-upload", filePath);

        // Click the upload button
        await Page.ClickAsync("input#file-submit");

        // Verify the file upload success message
        await Expect(Page.Locator("h3")).ToHaveTextAsync("File Uploaded!");
        await Expect(Page.Locator("#uploaded-files")).ToHaveTextAsync("test-file.txt");
    }
}
```

```C#
using Microsoft.Playwright;
using Microsoft.Playwright.MSTest;
using System.IO;
using System.Threading.Tasks;

namespace PlaywrightTests;

[TestClass]
public class FileDownloadTests : PageTest
{
    [TestMethod]
    public async Task FileDownloadWorksCorrectly()
    {
        // Navigate to the file download page
        await Page.GotoAsync("https://the-internet.herokuapp.com/download");

        // Start waiting for the download
        var download = await Page.RunAndWaitForDownloadAsync(async () =>
        {
            await Page.ClickAsync("a[href='download/some-file.txt']"); // Adjust the selector as needed
        });

        // Save the downloaded file (absolute path)
        var downloadPath = Path.GetFullPath("downloads/some-file.txt");
        await download.SaveAsAsync(downloadPath);

        // Verify that the file exists
        Assert.IsTrue(File.Exists(downloadPath), $"Downloaded file not found at: {downloadPath}");
    }

}
```


# API Mocking
```C#
using Microsoft.Playwright;
using Microsoft.Playwright.MSTest;
using System.Text.Json;
using System.Threading.Tasks;

namespace PlaywrightTests;

[TestClass]
public class ApiMockingTests : PageTest
{
    [TestMethod]
    public async Task ApiMockingExample()
    {
        // Intercept the API request and provide a mock response
        await Page.RouteAsync("https://reqres.in/api/users/2", async route =>
        {
            var mockResponse = new
            {
                data = new
                {
                    id = 2,
                    email = "janet.weaver@reqres.in",
                    first_name = "Janet",
                    last_name = "Weaver",
                    avatar = "https://reqres.in/img/faces/2-image.jpg"
                }
            };

            var json = JsonSerializer.Serialize(mockResponse);

            await route.FulfillAsync(new RouteFulfillOptions
            {
                ContentType = "application/json",
                Body = json
            });
        });

        // Make a fetch request on the page
        await Page.GotoAsync("about:blank"); // Blank page for demonstration
        var response = await Page.EvaluateAsync<JsonElement>(@"async () => {
            const res = await fetch('https://reqres.in/api/users/2');
            return await res.json();
        }");

        // Verify the mocked response
        Assert.AreEqual("Janet", response.GetProperty("data").GetProperty("first_name").GetString());
    }
}
```

# CI / CD Integration
```yaml
name: Playwright Tests (MSTest C#)

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'

    - name: Install dependencies
      run: dotnet restore

    - name: Build the project
      run: dotnet build --configuration Release

    - name: Install Playwright Browsers
      run: pwsh bin/Release/net8.0/playwright.ps1 install

    - name: Run Playwright tests
      run: dotnet test --configuration Release --no-build --verbosity normal
```