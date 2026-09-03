# 錯誤處理運算子 (Error Handling Operators)

例外狀況時有發生。某些例外狀況本質上是可以避免的，僅僅因為我們程式碼中的錯誤而發生。例如，如果我們將 CLR 置於必須引發 `DivideByZeroException` 的情況，我們就做錯了某些事情。但是有許多例外狀況是無法透過防禦性程式碼來預防的。例如，與 I/O 或網路故障相關的例外狀況（例如 `FileNotFoundException` 或 `TimeoutException`）可能是由我們程式碼控制之外的環境因素引起的。在這些情況下，我們需要優雅地處理例外狀況。處理的類型將取決於環境定義。向使用者提供某種錯誤訊息可能是適當的；在某些情境中，記錄錯誤可能是更合適的回應。如果失敗可能是暫時的，我們可以嘗試透過重試失敗的操作來復原。

`IObserver<T>` 介面定義了 `OnError` 方法以便來源可以報告錯誤，但由於這會終止序列，因此它沒有提供直接的方法來確定下一步該做什麼。但是，Rx 提供了提供各種錯誤處理機制的運算子。

## Catch

Rx 定義了一個 `Catch` 運算子。該名稱故意讓人聯想到 C# 的 `try`/`catch` 語法，因為它允許你以類似於處理正常程式碼執行中出現的例外狀況的方式來處理來自 Rx 來源的錯誤。它可以透過兩種不同的方式運作。你可以只提供一個 Rx 將錯誤傳遞給它的函式，此函式可以返回一個 `IObservable<T>`，並且 `Catch` 現在將轉發來自該序列的項目而不是原始來源。或者，除了傳遞函式之外，你還可以僅提供一個或多個附加序列，並且 catch 將在每次當前序列失敗時移至下一個序列。

### 檢驗例外狀況 (Examining the exception)

`Catch` 有一個多載，使你能夠提供在來源產生錯誤時要叫用的處理常式：

```csharp
public static IObservable<TSource> Catch<TSource, TException>(
    this IObservable<TSource> source, 
    Func<TException, IObservable<TSource>> handler) 
    where TException : Exception
```

這在概念上與 C# `catch` 區塊非常相似：我們可以編寫檢驗例外狀況然後決定如何進行的程式碼。就像使用 `catch` 區塊一樣，我們可以決定對哪種類型的例外狀況感興趣。例如，我們可能知道來源有時會產生 `TimeoutException`，並且在這種情況下我們可能只想返回一個空序列，而不是錯誤：

```csharp
IObservable<int> result = source.Catch<int, TimeoutException>(_ => Observable.Empty<int>());
```

只有在例外狀況屬於指定類型（或衍生自該類型的類型）時，`Catch` 才會叫用我們的函式。如果序列要以無法轉型為 `TimeoutException` 的 `Exception` 終止，則不會攔截該錯誤，並且該錯誤將流向訂閱者。

此範例返回 `Observable.Empty<int>()`。這在概念上類似於在 C# 中「吞下 (swallowing)」例外狀況，即選擇不採取任何動作。這可能是對你預期的例外狀況的合理回應，但對於基底 `Exception` 類型這樣做通常是個壞主意。

最後一個範例忽略了其輸入，因為它只對例外狀況類型感興趣。但是，我們可以自由檢驗例外狀況，並對應該從 `Catch` 中出現的內容做出更細緻的決策：

```csharp
IObservable<string> result = source.Catch(
    (FileNotFoundException x) => x.FileName == "settings.txt"
        ? Observable.Return(DefaultSettings) : Observable.Throw<string>(x));
```

這為一個特定檔案提供了特殊處理，但在其他情況下會重新擲回例外狀況。在此處返回 `Observable.Throw<T>(x)`（其中 `x` 是原始例外狀況）在概念上類似於在 catch 區塊中編寫 `throw`。（在 C# 中，`throw;` 和 `throw x;` 之間存在重要區別，因為它會變更例外狀況環境定義的擷取方式，但在 Rx 中，`OnError` 不會擷取堆疊追蹤，因此沒有等效的區別。）

當然，你也可以自由擲回完全不同的例外狀況。你可以返回任何你喜歡的 `IObservable<T>`，只要其元素類型與來源的元素類型相同即可。

### 後備 (Fallback)

`Catch` 的其他多載提供了較少區別的行為：你可以提供一個或多個附加序列，只要當前來源失敗，例外狀況就會被忽略，並且 `Catch` 將簡單地移至下一個序列。由於你永遠不會知道例外狀況是什麼，因此此機制無法讓你知道發生的例外狀況是你預期的還是完全意外的，因此你通常會希望避免使用這種形式。但為了完整起見，以下是如何使用它：

```csharp
IObservable<string> settings = settingsSource1.Catch(settingsSource2);
```

該形式僅提供單一後備。還有一個採用 `params` 陣列的靜態 `Observable.Catch` 方法，因此你可以傳遞任意數量的來源。這與前面的範例完全等效：

```csharp
IObservable<string> settings = Observable.Catch(settingsSource1, settingsSource2);
```

還有一個接受 `IEnumerable<IObservable<T>>` 的多載。

如果任何來源在沒有報告例外狀況的情況下到達其末尾，`Catch` 也會立即報告完成，並且不會訂閱任何後續來源。如果最後一個來源報告了例外狀況，`Catch` 將沒有進一步的來源可以後備，因此在這種情況下它不會攔截該例外狀況。它將向其訂閱者轉發該最終例外狀況。

## Finally

與 C# 中的 `finally` 區塊類似，Rx 使我們能夠在序列完成時執行某些程式碼，無論它是自然執行完成還是失敗。Rx 新增了第三種完成模式，在 `catch`/`finally` 中沒有完全等效的模式：訂閱者可能會在來源有機會完成之前取消訂閱。（這在概念上類似於使用 `break` 提前終止 `foreach`。）`Finally` 擴充方法接受一個 `Action` 作為參數。當序列終止時，無論是否呼叫了 `OnCompleted` 或 `OnError`，都將叫用此 `Action`。如果在完成之前處置了訂閱，它也將叫用該動作。

```csharp
public static IObservable<TSource> Finally<TSource>(
    this IObservable<TSource> source, 
    Action finallyAction)
{
    ...
}
```

在此範例中，我們有一個完成的序列。我們提供了一個動作，並看到它在我們的 `OnCompleted` 處理常式之後被呼叫。

```csharp
var source = new Subject<int>();
IObservable<int> result = source.Finally(() => Console.WriteLine("Finally action ran"));
result.Dump("Finally");
source.OnNext(1);
source.OnNext(2);
source.OnNext(3);
source.OnCompleted();
```

輸出：

```
Finally-->1
Finally-->2
Finally-->3
Finally completed
Finally action ran
```

來源序列也可能因例外狀況而終止。在這種情況下，例外狀況將傳送到訂閱者的 `OnError`（我們將在主控台輸出中看到該例外狀況），然後將執行我們提供給 `Finally` 的委派。

或者，我們本可以處置我們的訂閱。在下一個範例中，我們看到即使序列沒有完成，也會叫用 `Finally` 動作。

```csharp
var source = new Subject<int>();
var result = source.Finally(() => Console.WriteLine("Finally"));
var subscription = result.Subscribe(
    Console.WriteLine,
    Console.WriteLine,
    () => Console.WriteLine("Completed"));
source.OnNext(1);
source.OnNext(2);
source.OnNext(3);
subscription.Dispose();
```

輸出：

```
1
2
3
Finally
```

請注意，如果訂閱者的 `OnError` 擲回例外狀況，並且如果來源在沒有 `try`/`catch` 區塊的情況下呼叫 `OnNext`，則 CLR 的未處理例外狀況報告機制將啟動，並且在某些情況下，這可能會導致應用程式在 `Finally` 運算子有機會叫用回呼函式之前關閉。我們可以使用以下程式碼建立這種情境：

```csharp
var source = new Subject<int>();
var result = source.Finally(() => Console.WriteLine("Finally"));
result.Subscribe(
    Console.WriteLine,
    // Console.WriteLine,
    () => Console.WriteLine("Completed"));
source.OnNext(1);
source.OnNext(2);
source.OnNext(3);

// Brings the app down. Finally action might not be called.
source.OnError(new Exception("Fail"));
```

如果你直接從程式的進入點執行此操作而未將其包裝在 `try`/`catch` 中，你可能會看到也可能不會看到顯示的 `Finally` 訊息，因為在例外狀況一路到達堆疊頂部而未被攔截的情況下，例外狀況處理的運作方式會有些微不同。（奇怪的是，它通常確實會執行，但是如果你附加了偵錯工具，程式通常會在不執行 `Finally` 回呼函式的情況下結束。）

這主要只是一個好奇點：像 ASP.NET Core 或 WPF 這樣的應用程式架構通常會安裝它們自己的堆疊頂部例外狀況處理常式，而且在任何情況下，你都不應該在未提供錯誤回呼函式的情況下訂閱你知道會呼叫 `OnError` 的來源。出現此問題僅僅是因為此處使用的基於委派的 `Subscribe` 多載提供了一個在其 `OnError` 中擲回例外狀況的 `IObserver<T>` 實作。但是，如果你正在建構主控台應用程式來試驗 Rx 的行為，你很可能會遇到這種情況。在實務上，`Finally` 在更正常的情況下會做正確的事情。（但在任何情況下，你都不應該從 `OnError` 處理常式擲回例外狀況。）

## Using

`Using` Factory 方法允許你將資源的存留期繫結至可觀察序列的存留期。該方法採用兩個回呼函式：一個用於建立可處置資源，另一個用於提供序列。這允許對所有內容進行延遲求值。當程式碼在此方法返回的 `IObservable<T>` 上呼叫 `Subscribe` 時，將叫用這些回呼函式。

```csharp
public static IObservable<TSource> Using<TSource, TResource>(
    Func<TResource> resourceFactory, 
    Func<TResource, IObservable<TSource>> observableFactory) 
    where TResource : IDisposable
{
    ...
}
```

當序列透過 `OnCompleted` 或 `OnError` 終止時，或者當訂閱被處置時，該資源將被處置。

## OnErrorResumeNext

僅僅是本節的標題就會讓老 VB 開發人員感到不寒而慄！（對於不熟悉此晦澀語言功能的人來說，VB 語言允許你指示它忽略執行期間發生的任何錯誤，並在發生任何失敗後直接繼續執行下一陳述式。）在 Rx 中，有一個名為 `OnErrorResumeNext` 的擴充方法，其語意與共享相同名稱的 VB 關鍵字/陳述式相似。此擴充方法允許序列與另一個序列接續，無論第一個序列是正常完成還是由於錯誤而完成。

這與第二種形式的 `Catch` 非常相似（如[後備](#fallback)中所述）。區別在於對於 `Catch`，如果任何來源序列在沒有報告錯誤的情況下到達其末尾，`Catch` 將不會移至下一個序列。`OnErrorResumeNext` 將轉發其所有輸入產生的所有元素，因此它類似於 [`Concat`](09_CombiningSequences.md#concat)，只不過它忽略所有錯誤。

正如在 VB 中除了拋棄式程式碼之外最好避免使用 `OnErrorResumeNext` 關鍵字一樣，在 Rx 中也應謹慎使用它。它會悄悄吞下例外狀況，並可能使你的程式處於未知狀態。通常，這會使你的程式碼更難以維護和偵錯。（這同樣適用於 `Catch` 的後備形式。）

## Retry

如果你預期你的序列會遇到可預測的失敗，你可能只想重試。例如，如果你在雲端環境中執行，操作偶爾會無緣無故失敗是非常常見的。基於維運原因，雲端平台通常會相當定期地重新安置服務，這意味著操作失敗並不罕見——你可能在雲端提供者決定將該服務移至不同計算節點之前向該服務發出請求——但是如果你立即重試，完全相同的操作可能會成功（因為重試的請求會路由到新節點）。Rx 的 `Retry` 擴充方法提供了在失敗時重試指定次數或直到成功的功​​能。其運作方式是在來源報告錯誤時重新訂閱來源。

此範例使用簡單的多載，它將始終在發生任何例外狀況時重試。

```csharp
public static void RetrySample<T>(IObservable<T> source)
{
    source.Retry().Subscribe(t => Console.WriteLine(t)); // Will always retry
    Console.ReadKey();
}
```

給定一個產生值 0、1 和 2 然後呼叫 `OnError` 的來源，輸出將是無休止地重複數字 0、1、2。此輸出將永遠持續下去，因為此範例從不取消訂閱，如果你沒有另行指定，`Retry` 將永遠重試。

我們可以指定最大重試次數。在接下來的這個範例中，我們只重試一次，因此在第二次訂閱時發布的錯誤將傳遞給最終訂閱。請注意，我們告訴 `Retry` 最大嘗試次數，因此如果我們希望它重試一次，你傳遞值 2——即初始嘗試加上一次重試。

```csharp
source.Retry(2).Dump("Retry(2)"); 
```

輸出：

```
Retry(2)-->0
Retry(2)-->1
Retry(2)-->2
Retry(2)-->0
Retry(2)-->1
Retry(2)-->2
Retry(2) failed-->Test Exception
```

使用無限重複多載時應當小心。顯然，如果你的底層序列存在持續性問題，你可能會發現自己陷入無限迴圈。另外請注意，沒有允許你指定重試的例外狀況類型的多載。

Rx 還提供了一個 `RetryWhen` 方法。這類似於我們檢驗的第一個 `Catch` 多載：它不是不加區別地處理所有例外狀況，而是允許你提供可以決定要做什麼的程式碼。它的工作方式略有不同：它不是為每個錯誤叫用一次此回呼函式，而是叫用一次傳入一個 `IObservable<Exception>`（透過該序列提供所有例外狀況），並且回呼函式返回一個稱為_訊號 (signal)_可觀察序列的 `IObservable<T>`。`T` 可以是任何類型，因為此可觀察序列可能返回的值將被忽略：重要的只是叫用了三個 `IObserver<T>` 方法中的哪一個。

如果在接收到例外狀況時，訊號可觀察序列呼叫 `OnError`，則 `RetryWhen` 將不會重試，並將向其訂閱者報告相同的錯誤。另一方面，如果訊號可觀察序列呼叫 `OnCompleted`，`RetryWhen` 也將不會重試，並將在不報告錯誤的情況下完成。但是如果訊號可觀察序列呼叫 `OnNext`，這會導致 `RetryWhen` 透過重新訂閱來源進行重試。

<!--TODO: Build BackOffRetry with the reader-->

應用程式通常需要超越簡單 `OnError` 處理常式的例外狀況管理邏輯。Rx 提供了類似於我們在 C# 中習慣的例外狀況處理運算子，你可以使用它們來組合複雜且穩健的查詢。在本章中，我們介紹了 Rx 的進階錯誤處理和更多資源管理功能。
