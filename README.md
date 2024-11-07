
在使用 `HttpClient` 发起 HTTP 请求时，可能会遇到请求头丢失的问题，尤其是像 `Accept-Language` 这样的请求头丢失。这个问题可能会导致请求的内容错误，甚至影响整个系统的稳定性和功能。本文将深入分析这一问题的根源，并介绍如何通过 `HttpRequestMessage` 来解决这一问题。


#### **1\. 问题的背景：HttpClient的设计与共享机制**


`HttpClient` 是 .NET 中用于发送 HTTP 请求的核心类，它是一个设计为可复用的类，其目的是为了提高性能，减少在高并发情况下频繁创建和销毁 HTTP 连接的开销。`HttpClient` 的复用能够利用操作系统底层的连接池机制，避免了每次请求都要建立新连接的性能损失。


但是，`HttpClient` 复用的机制也可能导致一些问题，尤其是在多线程并发请求时。例如，如果我们在共享的 `HttpClient` 实例上频繁地修改请求头，可能会导致这些修改在不同的请求之间意外地“传递”或丢失。


#### **2\. 常见问题：丢失请求头**


假设我们有如下的代码，其中我们希望在每次请求时设置 `Accept-Language` 头：



[?](https://github.com)

| 123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384 | `using` `System.Net.Http;``using` `System.Text;``using` `Newtonsoft.Json;``using` `Newtonsoft.Json.Serialization;` `namespace` `ConsoleApp9``{``internal` `class` `Program``{``private` `static` `readonly` `JsonSerializerSettings serializerSettings =` `new` `JsonSerializerSettings``{``ContractResolver =` `new` `CamelCasePropertyNamesContractResolver(),``NullValueHandling = NullValueHandling.Ignore``};` `private` `static` `readonly` `HttpClient httpClient =` `new` `HttpClient();` `// 复用HttpClient实例``private` `static` `readonly` `SemaphoreSlim semaphore =` `new` `SemaphoreSlim(100);` `// 限制并发请求数量为100` `static` `async Task Main(``string``[] args)``{``List tasks =` `new` `List();``int` `taskNoCounter = 1;` `// 用于跟踪 taskno``// 只使用一个HttpClient对象（全局共享）``for` `(``int` `i = 0; i < 50; i++)``{``tasks.Add(Task.Run(async () =>``{``// 等待信号量，控制最大并发数``await semaphore.WaitAsync();` `try``{``var` `postData =` `new``{``taskno = taskNoCounter++,``content =` `"等待翻译的内容"``};``var` `json = JsonConvert.SerializeObject(postData, serializerSettings);``var` `reqdata =` `new` `StringContent(json, Encoding.UTF8,` `"application/json"``);` `// 设置请求头语言``httpClient.DefaultRequestHeaders.Add(``"Accept-Language"``,` `"en-US"``);` `// 发送请求``var` `result = await httpClient.PostAsync(``"http://localhost:5000/translate"``, reqdata);` `// 读取并反序列化 JSON 数据``var` `content = await result.Content.ReadAsStringAsync();``var` `jsonResponse = JsonConvert.DeserializeObject(content);``var` `response = jsonResponse.Data.Content;` `// 反序列化后，直接输出解码后的文本``Console.WriteLine($``"结果为：{response}"``);``}``catch` `(Exception ex)``{``Console.WriteLine($``"请求失败: {ex.Message}"``);``}``finally``{``// 释放信号量``semaphore.Release();``}``}));``}` `await Task.WhenAll(tasks);``}``}` `// 定义与响应结构匹配的类``public` `class` `Response``{``public` `int` `Code {` `get``;` `set``; }``public` `ResponseData Data {` `get``;` `set``; }``public` `string` `Msg {` `get``;` `set``; }``}` `public` `class` `ResponseData``{``public` `string` `Content {` `get``;` `set``; }``public` `string` `Lang {` `get``;` `set``; }``public` `int` `Taskno {` `get``;` `set``; }``}``}` |
| --- | --- |



　　


接收代码如下：



[?](https://github.com):[楚门加速器](https://shexiangshi.org)

| 1234567891011121314151617181920212223242526272829303132333435363738394041424344 | `from` `flask import Flask, request, jsonify``from` `google.cloud import translate_v2` `as` `translate` `app = Flask(__name__)` `# 初始化 Google Cloud Translate 客户端``translator = translate.Client()` `@app.route(``'/translate'``, methods=[``'POST'``])``def translate_text():``try``:``# 从请求中获取 JSON 数据``data = request.get_json()` `# 获取请求的文本内容``text = data.``get``(``'content'``)``taskno = data.``get``(``'taskno'``, 1)` `# 获取请求头中的 Accept-Language 信息，默认为 'zh-CN'``accept_language = request.headers.``get``(``'Accept-Language'``,` `'zh-CN'``)` `# 调用 Google Translate API 进行翻译``result = translator.translate(text, target_language=accept_language)` `# 构造响应数据``response_data = {``"code"``: 200,``"msg"``:` `"OK"``,``"data"``: {``"taskno"``: taskno,``"content"``: result[``'translatedText'``],``"lang"``: accept_language``}``}` `# 返回 JSON 响应``return` `jsonify(response_data), 200` `except Exception` `as` `e:``return` `jsonify({``"code"``: 500,` `"msg"``: str(e)}), 500`  `if` `__name__ ==` `"__main__"``:``app.run(debug=True, host=``"0.0.0.0"``, port=5000)` |
| --- | --- |



　


`Accept-Language` 请求头是通过 `httpClient.DefaultRequestHeaders.Add("Accept-Language", language)` 来设置的。这是一个常见的做法，目的是为每个请求指定特定的语言。然而，在实际应用中，尤其是当 `HttpClient` 被复用并发发送多个请求时，这种方法可能会引发请求头丢失或错误的情况。


测试结果：每20个请求就会有一个接收拿不到语言，会使用默认的zh\-CN，这条请求就不会翻译。在上面的代码中，


#### **3\. 为什么会丢失请求头？**


丢失请求头的问题通常出现在以下两种情况：


* **并发请求之间共享 `HttpClient` 实例**：当多个线程或任务共享同一个 `HttpClient` 实例时，它们可能会修改 `DefaultRequestHeaders`，导致请求头在不同请求之间互相干扰。例如，如果一个请求修改了 `Accept-Language`，它会影响到后续所有的请求，而不是每个请求都独立使用自己的请求头。
* **头部缓存问题**：`HttpClient` 实例可能会缓存头部信息。如果请求头未正确设置，缓存可能会导致丢失之前设置的头部。


在这种情况下，丢失请求头或请求头不一致的现象就会发生，从而影响请求的正确性和响应的准确性。


#### **4\. 解决方案：使用 `HttpRequestMessage`**


为了解决这个问题，我们可以使用 `HttpRequestMessage` 来替代直接修改 `HttpClient.DefaultRequestHeaders`。`HttpRequestMessage` 允许我们为每个请求独立地设置请求头，从而避免了多个请求之间共享头部的风险。


以下是改进后的代码：





[?](https://github.com)

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667686970717273747576777879808182838485868788899091 | `using` `System.Net.Http;``using` `System.Text;``using` `Newtonsoft.Json;``using` `Newtonsoft.Json.Serialization;` `namespace` `ConsoleApp9``{``internal` `class` `Program``{``private` `static` `readonly` `JsonSerializerSettings serializerSettings =` `new` `JsonSerializerSettings``{``ContractResolver =` `new` `CamelCasePropertyNamesContractResolver(),``NullValueHandling = NullValueHandling.Ignore``};` `private` `static` `readonly` `HttpClient httpClient =` `new` `HttpClient();` `// 复用HttpClient实例``private` `static` `readonly` `SemaphoreSlim semaphore =` `new` `SemaphoreSlim(100);` `// 限制并发请求数量为100` `static` `async Task Main(``string``[] args)``{``List tasks =` `new` `List();``int` `taskNoCounter = 1;` `// 用于跟踪 taskno``// 只使用一个HttpClient对象（全局共享）``for` `(``int` `i = 0; i < 50; i++)``{``tasks.Add(Task.Run(async () =>``{``// 等待信号量，控制最大并发数``await semaphore.WaitAsync();` `try``{``var` `postData =` `new``{``taskno = taskNoCounter++,``content =` `"等待翻译的内容"``};``var` `json = JsonConvert.SerializeObject(postData, serializerSettings);``var` `reqdata =` `new` `StringContent(json, Encoding.UTF8,` `"application/json"``);` `// 使用HttpRequestMessage确保每个请求都可以单独设置头``var` `requestMessage =` `new` `HttpRequestMessage(HttpMethod.Post,` `"http://localhost:5000/translate"``)``{``Content = reqdata``};` `// 设置请求头``requestMessage.Headers.Add(``"Accept-Language"``,` `"en-US"``);` `// 发起POST请求``var` `result = await httpClient.SendAsync(requestMessage);` `// 读取并反序列化 JSON 数据``var` `content = await result.Content.ReadAsStringAsync();``var` `jsonResponse = JsonConvert.DeserializeObject(content);``var` `response = jsonResponse.Data.Content;` `// 反序列化后，直接输出解码后的文本``Console.WriteLine($``"结果为：{response}"``);``}``catch` `(Exception ex)``{``Console.WriteLine($``"请求失败: {ex.Message}"``);``}``finally``{``// 释放信号量``semaphore.Release();``}``}));``}` `await Task.WhenAll(tasks);``}``}` `// 定义与响应结构匹配的类``public` `class` `Response``{``public` `int` `Code {` `get``;` `set``; }``public` `ResponseData Data {` `get``;` `set``; }``public` `string` `Msg {` `get``;` `set``; }``}` `public` `class` `ResponseData``{``public` `string` `Content {` `get``;` `set``; }``public` `string` `Lang {` `get``;` `set``; }``public` `int` `Taskno {` `get``;` `set``; }``}``}` |
| --- | --- |



　　


#### **5\. 解析解决方案：为何 `HttpRequestMessage` 更加可靠**


* **独立请求头**：`HttpRequestMessage` 是一个每个请求都可以独立设置头部的类，它允许我们为每个 HTTP 请求单独配置请求头，而不会被其他请求所干扰。通过这种方式，我们可以确保每个请求都使用准确的请求头。
* **高并发控制**：当 `HttpClient` 实例被多个请求共享时，`HttpRequestMessage` 确保每个请求都能够独立处理头部。即使在高并发环境下，每个请求的头部设置都是独立的，不会相互影响。
* **请求灵活性**：`HttpRequestMessage` 不仅可以设置请求头，还可以设置请求方法、请求体、请求的 URI 等，这使得它比直接使用 `DefaultRequestHeaders` 更加灵活和可控。


#### **6\. 小结：优化 `HttpClient` 请求头管理**


总结来说，当使用 `HttpClient` 时，若多个请求共用一个实例，直接修改 `DefaultRequestHeaders` 会导致请求头丢失或不一致的问题。通过使用 `HttpRequestMessage` 来管理每个请求的头部，可以避免这个问题，确保请求头的独立性和一致性。


* 使用 `HttpRequestMessage` 来独立设置请求头，是确保请求头正确性的最佳实践。
* 复用 `HttpClient` 实例是提升性能的好方法，但要注意并发请求时请求头可能会丢失或错误，`HttpRequestMessage` 是解决这一问题的有效工具。


通过这种方式，我们不仅避免了请求头丢失的问题，还提升了请求的可靠性和可控性，使得整个 HTTP 请求管理更加高效和精确。




---


### 总结


以上从 `HttpClient` 设计和并发请求的角度，详细探讨了请求头丢失的问题，并通过实例代码展示了如何通过 `HttpRequestMessage` 来优化请求头管理。通过这种方式，能够确保在高并发或多线程环境中每个请求的请求头都能够独立设置，从而避免了请求头丢失或错误的问题。




