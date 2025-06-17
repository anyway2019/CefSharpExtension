# CefSharpExtension

```Csharp
/// <summary>
/// 尝试点击,如果失败则不做任何操作
/// </summary>
/// <param name="browser"></param>
/// <param name="selector"></param>
/// <returns></returns>
public static async Task<JavascriptResponse> TryClickAsync(this ChromiumWebBrowser browser, string selector)
{
    var res = await browser.EvaluateScriptAsync($"var dm = {selector}[0]; dm.scrollIntoView({{block: 'center'}}); dm.getBoundingClientRect();");
    if (res.Success)
    {
        var rect = JsonConvert.DeserializeObject<JsRect>(JsonConvert.SerializeObject(res.Result));
        if (rect == null)
        {
            res.Success = false;
            res.Message = "can not get element rect";
            return res;
        }

        await browser.ClickRectAsync(rect);
    }

    return res;
}

/// <summary>
/// 点击dom节点失败则抛出异常
/// </summary>
/// <param name="browser"></param>
/// <param name="selector"></param>
/// <returns></returns>
/// <exception cref="Exception"></exception>
public static async Task<JavascriptResponse> ClickAsync(this ChromiumWebBrowser browser, string selector)
{
    var res = await browser.EvaluateScriptAsync($"var dm = {selector}[0]; dm.scrollIntoView({{block: 'center',instant: 'instant'}}); dm.getBoundingClientRect();");
    if (!res.Success)
    {
        throw new Exception($"fail to locate element {selector} : {res.Message}");
    }

    var rect = JsonConvert.DeserializeObject<JsRect>(JsonConvert.SerializeObject(res.Result));
    if (rect == null)
    {
        res.Success = false;
        res.Message = "can not get element rect";
        throw new Exception($"can not get element rect {selector} : {res.Message}");
    }

    await browser.ClickRectAsync(rect);

    return res;
}

private static async Task ClickRectAsync(this ChromiumWebBrowser browser, JsRect rect)
{
	 var x = (int)(rect.left + (rect.width / 2));
	 var y = (int)(rect.top + (rect.height / 2));
	 browser.GetBrowser().GetHost().SendMouseClickEvent(x, y, MouseButtonType.Left, false, 1, CefEventFlags.None);
	 await Task.Delay(Random.Shared.Next(200, 500));
	 browser.GetBrowser().GetHost().SendMouseClickEvent(x, y, MouseButtonType.Left, true, 1, CefEventFlags.None);
	 await Task.Delay(Random.Shared.Next(150, 500));
}

/// <summary>
/// 填充input,通过模拟键盘输入后点击回车
/// </summary>
/// <param name="browser"></param>
/// <param name="selector"></param>
/// <returns></returns>
public static async Task<JavascriptResponse> FillAsync(this ChromiumWebBrowser browser, string selector, string text)
{
    var res = await browser.ClickAsync(selector);
    await Task.Delay(Random.Shared.Next(10, 20));
    await browser.SendTextEventAsync(text);
    await browser.SendKeyCodeEventAsync(0xD);
    return res;
}

public static async Task SendKeyCodeEventAsync(this ChromiumWebBrowser browser, int keyCode)
{
    KeyEvent k = new KeyEvent()
    {
        FocusOnEditableField = true,
        WindowsKeyCode = keyCode,
        Modifiers = CefEventFlags.None,
        Type = KeyEventType.KeyDown,
        IsSystemKey = false,
    };

    browser.GetBrowserHost().SendKeyEvent(k);

    await Task.Delay(Random.Shared.Next(10, 50));

    k = new KeyEvent
    {
        WindowsKeyCode = (int)keyCode,
        FocusOnEditableField = false,
        IsSystemKey = false,
        Modifiers = CefEventFlags.None,
        Type = KeyEventType.Char
    };
    browser.GetBrowserHost().SendKeyEvent(k);

    await Task.Delay(Random.Shared.Next(10, 50));

    k = new KeyEvent()
    {
        FocusOnEditableField = true,
        WindowsKeyCode = keyCode,
        Modifiers = CefEventFlags.None,
        Type = KeyEventType.KeyUp,
        IsSystemKey = false,
    };
    browser.GetBrowserHost().SendKeyEvent(k);
}

public static async Task SendTextEventAsync(this ChromiumWebBrowser browser, string text)
{
    var chars = text.ToCharArray();
    foreach (var c in chars)
    {
        var keyCode = (int)c;
        await browser.SendKeyCodeEventAsync(keyCode);
    }
}

public static async Task<JavascriptResponse> PastContentToSelectorAsync(this ChromiumWebBrowser browser, string selector, string content)
{
    var js = $" var customValue = \"{content}\";var dataTransfer = new DataTransfer();dataTransfer.setData('text/plain', customValue);var clipboardEvent = new ClipboardEvent('paste', {{bubbles: true,cancelable: true,clipboardData: dataTransfer}});{selector}[0].dispatchEvent(clipboardEvent);";
    return await browser.GetMainFrame().EvaluateScriptAsync(js);
}

/// <summary>
/// 统计dom节点个数
/// </summary>
/// <param name="browser"></param>
/// <param name="selector"></param>
/// <returns></returns>
public static async Task<int> CountAsync(this ChromiumWebBrowser browser, string selector)
{
    try
    {

        var script = $"{selector}.length;";
        var t = await browser.EvaluateScriptAsync(script);
        return t.Result == null ? 0 : (int)t.Result;
    }
    catch 
    {
    }
    return 0;
}


public static async Task LoadUrlWithJQueryAsync(this ChromiumWebBrowser browser, string url)
{
    await browser.LoadUrlAsync(url);
    await browser.InjectJQueryAsync();
}

/// <summary>
/// 给当前浏览器实例注入jQuery（CDN的方式）
/// </summary>
/// <param name="browser"></param>
/// <returns></returns>
public static async Task InjectJQueryAsync(this ChromiumWebBrowser browser)
{
    try
    {
        string jqueryUrl = "https://apps.bdimg.com/libs/jquery/2.1.4/jquery.min.js"; // 百度jQuery的CDN 链接
        var script = $@"
            var script = document.createElement('script');
            script.src = '{jqueryUrl}';
            script.onload = function() {{
                console.log('jQuery has been loaded');
                window.jQueryLoaded = true; // 设置全局变量表示 jQuery 已加载
            }};
            document.head.appendChild(script);
        ";

        await browser.EvaluateScriptAsync(script);
        // 等待 jQuery 加载完成
        int times = 0;
        while (true)
        {
            var result = await browser.EvaluateScriptAsync("window.jQueryLoaded");
            if (result.Success && result.Result is bool loaded && loaded)
            {
                break;
            }
            await Task.Delay(100); // 每100毫秒检查一次
            times++;
            if (times > 60)
            {
                throw new TimeoutException("注入jQuery超时");
            }
        }
    }
    catch (Exception ex)
    {
        Logger.Error($"注入jQuery失败：{ex.Message} @ {ex.StackTrace}");
    }
}

 /// <summary>
 /// 执行界面操作后拦截网络请求
 /// </summary>
 /// <param name="browser"></param>
 /// <param name="action">界面操作</param>
 /// <param name="urlOrPredicate">筛选指定网络请求</param>
 /// <returns></returns>
 public static async Task<string> RunAndWaitForResponseAsync(this ChromiumWebBrowser browser,Func<Task> action, Func<IRequest, IResponse, bool> urlOrPredicate,int timeout = 6000)
 {
     var tcs = new TaskCompletionSource<string>();
     browser.RequestHandler = new CustomRequestHandler((s) => {
         if (tcs.Task.IsCompleted)
         {
             browser.RequestHandler = null;
             return;
         }
         tcs.SetResult(s); 
     }, urlOrPredicate);

     await action.Invoke();
     var completedTask = await Task.WhenAny(tcs.Task, Task.Delay(timeout));
     if (completedTask == tcs.Task)
     {
         return await tcs.Task;
     }
     else
     {
         browser.RequestHandler = null; 
         throw new TimeoutException("RunAndWaitForResponseAsync操作超时");
     }
 }

/// <summary>
/// 获取cookie
/// </summary>
/// <param name="browser"></param>
/// <returns></returns>
public static async Task<string> GetCookieStringAsync(this ChromiumWebBrowser browser,string domain)
{
    List<string> cookiePairs = new List<string>();
    var cookieManager = Cef.GetGlobalCookieManager();
    var cookies = await cookieManager.VisitAllCookiesAsync();
    foreach (var cookie in cookies)
    {
        if(cookie.Domain.Contains(domain))
            cookiePairs.Add($"{cookie.Name}={cookie.Value}");
    }

    return string.Join("; ", cookiePairs);
}

```