using Microsoft.Playwright;
using System.Threading.Tasks;

namespace PlaywrightTests.PageObjects
{
    public class TextInputPage
    {
        private readonly IPage _page;
        private readonly string _url = "https://www.uitestingplayground.com/textinput";
        private readonly ILocator _inputField;
        private readonly ILocator _button;

        public TextInputPage(IPage page)
        {
            _page = page;
            _inputField = _page.Locator("input#newButtonName");
            _button = _page.Locator("button#updatingButton");
        }

        public async Task NavigateAsync()
        {
            await _page.GotoAsync(_url);
        }

        public async Task EnterTextAsync(string text)
        {
            await _inputField.FillAsync(text);
        }

        public async Task ClickButtonAsync()
        {
            await _button.ClickAsync();
        }

        public async Task<string> GetButtonTextAsync()
        {
            var text = await _button.TextContentAsync();
            return text ?? string.Empty;
        }
    }
}