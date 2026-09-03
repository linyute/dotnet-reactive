# 脫離 Rx 的世界 (Leaving Rx's World)

可觀察序列是一種有用的建構，特別是當我們擁有 LINQ 的強大能力在其上組合複雜的查詢時。即使我們體認到可觀察序列的好處，有時我們也必須脫離 `IObservable<T>` 範式。當我們需要與現有的非基於 Rx 的 API（例如使用事件或 `Task<T>` 的 API）進行整合時，這是必要的。如果你發現脫離可觀察範式更容易進行測試，或者透過在可觀察範式和更熟悉的範式之間切換更容易學習 Rx，你可能會脫離該範式。

Rx 的組合本質是其強大力量的關鍵，但是當你需要與不理解 Rx 的元件整合時，這可能看起來像是一個問題。到目前為止我們檢驗的大多數 Rx 函式庫功能都將其輸入和輸出表示為可觀察序列。你應該如何取得某些真實世界的事件來源並將其轉換為可觀察序列？你應該如何使用可觀察序列的輸出做一些有意義的事情？

你已經看到了這些問題的一些答案。[建立可觀察序列章節](03_CreatingObservableSequences.md)展示了建立可觀察來源的各種方法。但是當談到處理從 `IObservable<T>` 出現的項目時，我們真正看到的只是如何實作 [`IObserver<T>`](02_KeyTypes.md#iobserver)，以及[如何使用基於回呼函式的 `Subscribe` 擴充方法訂閱 `IObservable<T>`](02_KeyTypes.md#iobservable)。

在本章中，我們將檢驗 Rx 中允許你脫離 `IObservable<T>` 世界的方法，以便你可以根據從 Rx 來源出現的通知採取動作。但首先，我們需要檢視這為錯誤處理帶來的挑戰。

## 例外狀況狀態 (Exception state)

當例外狀況脫離 Rx 的世界時，你必須遵守一條規則：如果使用 Rx 的開發人員選擇使用一種機制，該機制採用 `IObservable<T>` 傳遞的例外狀況並將其擲回（例如，如果你 `await` 一個 `IObservable<T>`），那麼開發人員有責任確保滿足以下任一條件：

* 每個例外狀況物件僅使用一次

或

* 在將例外狀況提供給將重新擲回該例外狀況的觀察者之前，重設該例外狀況的發派狀態（例如，透過執行 `throw`）

例如，如果你使用 [`Throw`](03_CreatingObservableSequences.md#observablethrow) 運算子，如果你計劃 `await` 產生的序列，這會產生問題：按照設計，`Throw` 完全不會對你提供的例外狀況的狀態做任何事情。這是故意的，因為你可能給了它一個已經包含重要狀態的例外狀況，並且它不想破壞該資訊。但是，在例外狀況實際上從未被擲回的情況下，該狀態永遠不會被設定（因此也永遠不會被重設）。沒有好的方法來偵測是否存在例外狀況狀態，因此 `Throw` 必須保守地假設它可能存在。這意味著如果你要兩次 `await` 這樣的序列，你將違反這些規則。Rx 6.1 引入了一個新的運算子 [`ResetExceptionDispatchState`](#resetexceptiondispatchstate) 來處理這種情境。如果你編寫類似這樣的程式碼：

```cs
IObservable<int> t = Observable
    .Throw<int>(new Exception("Boom!"))
    .ResetExceptionDispatchState();
```

`ResetExceptionDispatchState` 的存在確保了當 `Throw` 透過呼叫 `OnError` 報告錯誤時，例外狀況狀態會在該瞬間被重設。

但是為什麼存在這些規則呢？

完全生活在 Rx 世界中的程式碼透過讓 `IObservable<T>` 叫用觀察者的 `OnError` 方法來報告錯誤，並傳遞代表該錯誤的 `Exception`。報告、發現和處理例外狀況時完全不使用 `throw` 或 `catch`。這使得 Rx 對例外狀況的使用有些不同尋常，並且帶來了一個後果：如果你建構但未擲回例外狀況，則其_例外狀況狀態 (exception state)_永遠不會被設定。

例外狀況狀態是描述擲回例外狀況之環境定義的資訊。它包括呼叫堆疊的文字描述，這可以幫助開發人員診斷問題。它還包括對機器更友善的資訊，使用可以幫助自動化系統分析錯誤的技術識別碼來描述當機位置。[Windows 錯誤報告 (Windows Error Reporting)](https://learn.microsoft.com/en-us/windows/win32/wer/windows-error-reporting) 可以使用它來區分不同類型的應用程式失敗。對於桌面應用程式，這些資訊可以在獲得使用者同意的情況下集中收集，使開發人員能夠發現使用者在實際使用中最常遇到的當機。失敗由技術識別碼的特定組合來識別（這些識別碼有效地識別了失敗的程式碼中的特定位置，以及發生的失敗類型）。每個唯一的組合在非正式場合被稱為_錯誤儲存貯體 (fault bucket)_。你可以在 Windows 事件檢視器中看到此資訊。如果 .NET 應用程式擲回未處理的例外狀況，你通常會在事件記錄檔中看到三個事件：

* 一個 **.NET Runtime** 項目（通常具有事件識別碼 1026），報告應用程式可執行檔名稱、.NET 版本和 .NET 堆疊追蹤文字
* 一個 **Application Crashing Events** 項目（通常具有事件識別碼 1000），包含通用（非 .NET 專屬）詳細資訊，其格式與任何應用程式當機時顯示的格式相同，例如可執行檔路徑、Win32 SEH 例外狀況類型、例外狀況發生的來源 DLL 以及該 DLL 內的位移
* 一個 **Windows Error Reporting** 項目（通常具有事件識別碼 1001），包含_錯誤儲存貯體 (fault bucket)_詳細資訊

以下是來自發生當機之 .NET 應用程式的第三種項目的範例：

```
Fault bucket 1162564721043373785, type 4
Event Name: APPCRASH
Response: Not available
Cab Id: 0

Problem signature:
P1: TestResetExceptionDispatchState.exe
P2: 1.0.0.0
P3: 68a40000
P4: KERNELBASE.dll
P5: 10.0.26100.6584
P6: 0a9b38fe
P7: e0434352
P8: 00000000000c66ca
P9: 
P10: 
```

這比堆疊追蹤更難理解，但這些數值識別碼使得自動判斷兩次當機在某種意義上是否「相同」變得更加容易。這裡的想法是使其能夠發現何時頻繁發生同種類型的錯誤，使開發人員能夠將精力集中在修復導致最多麻煩的當機類型上。

對於 .NET 應用程式，我們希望這些資訊能夠反映失敗的根源。但是例外狀況被攔截並重新擲回是很常見的。我們_不_希望的是，僅僅因為我們編寫了一些常見的錯誤處理程式碼，每個錯誤都被分類為相同的類型。.NET 本身必須應對這一點：當我們使用 `await` 等待失敗的非同步操作時，該失敗例外狀況將被放入 `Task` 或 `ValueTask` 中，並且 .NET 將其重新擲回到呼叫 `await` 的程式碼中。如果向 Windows 錯誤處理報告每一個失敗的非同步操作都發生在重新擲回 `Task` 擷取之例外狀況的 .NET 執行階段函式庫程式碼內部，那將是非常無效的！

_例外狀況狀態_的目的是保留例外狀況的來源資訊，即使例外狀況被重新擲回也是如此。如果你透過在 `catch` 區塊中使用 `throw;` 來重新擲回例外狀況，則大部分例外狀況狀態都會被保留。（之所以說是「大部分」，是因為堆疊資訊得到了增強：堆疊追蹤將報告原始擲回位置和重新擲回位置。）如果你嘗試在 `catch` 區塊內使用 `throw ex;` 重新擲回例外狀況，這將_無法_正常運作，這就是為什麼在重新擲回時使用無引數的 `throw;` 形式很重要的原因。至於最終在一個地方被攔截並在完全不同的地方重新擲回的例外狀況（例如，在不同的執行緒上，因為正在使用 `await`），.NET 執行階段函式庫提供了一個名為 [`ExceptionDispatchInfo`](https://learn.microsoft.com/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo) 的協助程式，可以幫助管理此例外狀況狀態，並確保在與擲回例外狀況完全不同的環境定義中重新擲回例外狀況時使用它。

Rx 使用它來確保以下範例表現符合預期：

```cs
IObservable<string> fileLines = Observable.Create<string>(async obs =>
{
    using var reader = new StreamReader(@"c:\temp\test.txt");

    while ((await reader.ReadLineAsync()) is string line)
    {
        obs.OnNext(line);
    }
});

string firstNonEmptyLine = await fileLines
    .FirstAsync(line => line.Length > 0);
Console.WriteLine(firstNonEmptyLine);
```

如果嘗試開啟檔案時擲回例外狀況，我們期望看到什麼？熟悉例外狀況在 .NET 中通常如何與 `async`/`await` 一起運作的開發人員可能會合理地預期該例外狀況會報告兩個堆疊追蹤：一個用於最初擲回例外狀況的位置，另一個用於從 `await` 重新擲回的位置。這正是我們所看到的：

```
Unhandled exception. System.IO.FileNotFoundException: Could not find file 'c:\temp\test.txt'.
File name: 'c:\temp\test.txt'
   at Microsoft.Win32.SafeHandles.SafeFileHandle.CreateFile(String fullPath, FileMode mode, FileAccess access, FileShare share, FileOptions options)
   at Microsoft.Win32.SafeHandles.SafeFileHandle.Open(String fullPath, FileMode mode, FileAccess access, FileShare share, FileOptions options, Int64 preallocationSize, Nullable`1 unixCreateMode)
   at System.IO.Strategies.OSFileStreamStrategy..ctor(String path, FileMode mode, FileAccess access, FileShare share, FileOptions options, Int64 preallocationSize, Nullable`1 unixCreateMode)
   at System.IO.Strategies.FileStreamHelpers.ChooseStrategyCore(String path, FileMode mode, FileAccess access, FileShare share, FileOptions options, Int64 preallocationSize, Nullable`1 unixCreateMode)
   at System.IO.StreamReader.ValidateArgsAndOpenPath(String path, Encoding encoding, Int32 bufferSize)
   at System.IO.StreamReader..ctor(String path)
   at Program.<>c.<<<Main>$>b__0_0>d.MoveNext() in D:\source\RxThrowExamples\RxThrowExamples\Program.cs:line 5
--- End of stack trace from previous location ---
   at System.Reactive.PlatformServices.ExceptionServicesImpl.Rethrow(Exception exception)
   at System.Reactive.ExceptionHelpers.Throw(Exception exception)
   at System.Reactive.Subjects.AsyncSubject`1.GetResult()
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\RxThrowExamples\Program.cs:line 13
   at Program.<Main>(String[] args)
```

這符合上述規則，因為此處的例外狀況是以傳統的 .NET 方式擲回的。然後 Rx 攔截了它——`Observable.Create` 的這個多載將回呼函式返回的 `Task` 包裝在一個配接器中，該配接器偵測 `Task` 何時進入出錯狀態，在這種情況下它會擷取例外狀況並將其傳遞給訂閱的 `IObserver<T>`。然後當你 `await` 一個可觀察序列時 Rx 提供的 awaiter 會重新擲回同一個例外狀況。Rx 使用 `ExceptionDispatchInfo` 機制來確保在最初擲回例外狀況的位置擷取的例外狀況狀態在最終將例外狀況重新擲回給在可觀察序列上使用 `await` 的程式碼時仍然存在。

當例外狀況以傳統方式擲回而起源時，這很好，在最後一個範例中確實如此。它可能出錯的地方在於例外狀況根本沒有被擲回：在這種情況下，`ExceptionDispatchInfo` 提供的重新擲回機制會有點混亂。順便說一句，這個問題並不是 Rx 特有的，如下面的程式碼所示：

```cs
Exception ox = new("Kaboom!");

for (int i = 0; i < 3; ++i)
{
    Console.WriteLine();
    Console.WriteLine();

    try
    {
        await Task.FromException(ox); // Exception thrown here

    }
    catch (Exception x)
    {
        Console.WriteLine(x);
    }
}
```

這會建立一個單一的 `Exception` 物件，然後將其包裝在一個 `Task` 中（使用 `Task.FromException`），並立即對其進行 `await`。這裡沒有 Rx 程式碼。執行此程式碼，我們得到以下稍微令人驚訝的行為：

```
System.Exception: Kaboom!
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\ThrowStackDupWithoutRx\Program.cs:line 10


System.Exception: Kaboom!
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\ThrowStackDupWithoutRx\Program.cs:line 10
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\ThrowStackDupWithoutRx\Program.cs:line 10


System.Exception: Kaboom!
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\ThrowStackDupWithoutRx\Program.cs:line 10
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\ThrowStackDupWithoutRx\Program.cs:line 10
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\ThrowStackDupWithoutRx\Program.cs:line 10
```

請注意，每次繞過迴圈時，堆疊追蹤都會變長。這裡的問題是沒有任何東西重設例外狀況發派狀態。通常這會在發生原始 `throw` 時發生，但在這個範例中我們從未使用過 `throw`，因此該重設永遠不會發生。當你 `await` 一個 `Task` 時，.NET 執行階段使用 `ExceptionDispatchInfo` 來重新擲回例外狀況，該機制的要點在於它保留了原始例外狀況狀態，並附加了當前堆疊，以便你獲得例外狀況歷史記錄的完整記錄。它並非設計用於應對在沒有相應的多個 `throw` 的情況下多次重新擲回同一個例外狀況。

由於當例外狀況脫離 Rx 的世界時 Rx 使用相同的機制，因此該 `Task` 範例的 Rx 等效項具有完全相同的問題：

```cs
IObservable<int> ts = Observable.Throw<int>(new Exception("Pow!"));

for (int i = 0; i < 3; ++i)
{
    Console.WriteLine();
    Console.WriteLine();

	try
	{
		await ts; // Exception thrown here

	}
	catch (Exception x)
	{
        Console.WriteLine(x);
	}
}
```

這也會在每次繞過迴圈時產生更長的堆疊追蹤，因為我們再次多次重新擲回相同的例外狀況物件而從未重設其狀態。

請記住，Rx 故意保留該狀態，因為據它所知，該狀態很重要。（這就是上面帶有 `FileNotFoundException` 的範例報告正確資訊的方式）。這就是為什麼 Rx 6.1 引入了 [`ResetExceptionDispatchState`](#resetexceptiondispatchstate) 運算子。它使我們能夠告訴 Rx，每當可觀察序列出現錯誤時，我們確實希望它重設例外狀況狀態。如果我們將上面範例的第一個陳述式變更為：

```cs
IObservable<int> ts = Observable
   .Throw<int>(new Exception("Pow!"))
   .ResetExceptionDispatchState();
```

這現在符合本節開頭描述的規則，從而防止了堆疊追蹤不斷增長的問題。


## 與 `async` 和 `await` 的整合 (Integration with `async` and `await`)

你可以將 C# 的 `await` 關鍵字與任何 `IObservable<T>` 一起使用。我們之前在 [`FirstAsync`](05_Filtering.md#blocking-versions-of-firstlastsingleordefault) 中看到了這一點：

```csharp
long v = await Observable.Timer(TimeSpan.FromSeconds(2)).FirstAsync();
Console.WriteLine(v);
```

儘管 `await` 最常用於 `Task`、`Task<T>` 或 `ValueTask<T>`，但它實際上是一種可延伸的語言功能。透過提供一個名為 `GetAwaiter` 的方法（通常作為擴充方法），以及一個供 `GetAwaiter` 返回的合適類型，為 C# 提供 `await` 所需的功能，就可以使 `await` 適用於幾乎任何類型。這正是 Rx 所做的。如果你的原始程式檔包含 `using System.Reactive.Linq;` 指示詞，則將提供適當的擴充方法，因此你可以 `await` 任何工作。

其實際運作方式是相關的 `GetAwaiter` 擴充方法將 `IObservable<T>` 包裝在 `AsyncSubject<T>` 中，後者提供了 C# 支援 `await` 所需的一切。這些包裝函式的工作方式是在每次針對 `IObservable<T>` 執行 `await` 時都會呼叫一次 `Subscribe`。

如果來源透過叫用其觀察者的 `OnError` 來報告錯誤，Rx 的 `await` 整合將透過使工作進入出錯狀態來處理此問題，以便 `await` 將重新擲回該例外狀況。

序列可以是空的。它們可能會呼叫 `OnCompleted` 而從未呼叫過 `OnNext`。然而，由於無法從來源的類型判斷它是否為空，這與 `await` 範式並不是特別契合。對於工作，你可以透過檢視正在等待的是 `Task` 還是 `Task<T>` 在編譯時知道是否會獲得結果，因此編譯器能夠知道特定 `await` 運算式是否會產生值。但是當你 `await` 一個 `IObservable<T>` 時，沒有編譯時區別，因此當你 `await` 時 Rx 報告序列為空的唯一方法是擲回一個 `InvalidOperationException`，報告序列不包含任何元素。

正如你可能從[第 3 章的 `AsyncSubject<T>` 一節](03_CreatingObservableSequences.md#asyncsubjectt)中回想起來的那樣，`AsyncSubject<T>` 僅報告從其來源出現的最後一個值。因此，如果你 `await` 一個報告多個項目的序列，除最後一個項目外的所有項目都將被忽略。如果你想檢視所有項目，但仍想使用 `await` 來處理完成和錯誤，該怎麼辦？

## ForEachAsync

`ForEachAsync` 方法支援 `await`，但它提供了一種處理每個元素的方法。你可以將其視為前面章節中描述的 `await` 行為與基於回呼函式的 `Subscribe` 的混合體。我們仍然可以使用 `await` 來偵測完成和錯誤，但我們提供了一個回呼函式使我們能夠處理每個項目：

```csharp
IObservable<long> source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(5);
await source.ForEachAsync(i => Console.WriteLine($"received {i} @ {DateTime.Now}"));
Console.WriteLine($"finished @ {DateTime.Now}");
```

輸出：

```
received 0 @ 02/08/2023 07:53:46
received 1 @ 02/08/2023 07:53:47
received 2 @ 02/08/2023 07:53:48
received 3 @ 02/08/2023 07:53:49
received 4 @ 02/08/2023 07:53:50
finished @ 02/08/2023 07:53:50
```

請注意，如你所料，`finished` 行是最後一行。讓我們將其與 `Subscribe` 擴充方法進行比較，後者也允許我們提供用於處理項目的單一回呼函式：

```csharp
IObservable<long> source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(5);
source.Subscribe(i => Console.WriteLine($"received {i} @ {DateTime.Now}"));
Console.WriteLine($"finished @ {DateTime.Now}");
```

如輸出所示，`Subscribe` 立即返回。我們的個別項目回呼函式就像以前一樣被叫用，但這都是在稍後發生的：

```
finished @ 02/08/2023 07:55:42
received 0 @ 02/08/2023 07:55:43
received 1 @ 02/08/2023 07:55:44
received 2 @ 02/08/2023 07:55:45
received 3 @ 02/08/2023 07:55:46
received 4 @ 02/08/2023 07:55:47
```

這在執行某些工作然後結束的批次樣式程式中很有用。在該情境中使用 `Subscribe` 的問題在於，我們的程式很容易在未完成其開始的工作的情況下就結束。使用 `ForEachAsync` 很容易避免這種情況，因為我們只需使用 `await` 來確保我們的方法在工作完成之前不會結束。

當我們直接針對 `IObservable<T>` 或透過 `ForEachAsync` 使用 `await` 時，我們本質上是選擇以傳統方式而非反應式方式處理序列完成。錯誤和完成處理不再由回呼函式驅動——Rx 為我們提供 `OnCompleted` 和 `OnError` 處理常式，而是透過 C# 的 awaiter 機制來表示這些處理常式。（具體來說，當我們直接 `await` 來源時，Rx 提供自訂 awaiter，而當我們使用 `ForEachAsync` 時，它只返回一個 `Task`。）

請注意，在某些情況下 `Subscribe` 會封鎖，直到其來源完成為止。[`Observable.Return`](03_CreatingObservableSequences.md#observablereturn) 預設會這樣做，[`Observable.Range`](03_CreatingObservableSequences.md#observablerange) 也是如此。我們可以嘗試透過指定不同的排程器讓最後一個範例這樣做：

```csharp
// Don't do this!
IObservable<long> source = 
   Observable.Interval(TimeSpan.FromSeconds(1), ImmediateScheduler.Instance)
             .Take(5);
source.Subscribe(i => Console.WriteLine($"received {i} @ {DateTime.Now}"));
Console.WriteLine($"finished @ {DateTime.Now}");
```

然而，這凸顯了非非同步封鎖呼叫的危險：儘管這看起來應該有效，但實際上它在當前版本的 Rx 中會發生死結。Rx 認為 `ImmediateScheduler` 不適合基於計時器的操作，這就是為什麼它不是預設值的原因，而這個情境就是為什麼會這樣的一個很好的範例。（根本問題是取消排程工作項目的唯一方法是在呼叫 `Schedule` 返回的物件上呼叫 `Dispose`。根據定義，`ImmediateScheduler` 在完成工作之後才會返回，這意味著它實際上無法支援取消。因此對 `Interval` 的呼叫實際上建立了一個無法取消的定期排程工作項目，因此注定要永遠執行。）

這就是為什麼我們需要 `ForEachAsync`。看起來我們可能可以透過巧妙地使用排程器來獲得相同的效果，但實際上如果你需要等待非同步事件發生，使用 `await` 總是比使用需要封鎖呼叫端執行緒的方法更好。

## ToEnumerable

到目前為止，我們檢驗的兩種機制將完成和錯誤處理從 Rx 的回呼函式機制轉換為由 `await` 啟用的更傳統的方法，但我們仍然必須提供回呼函式才能處理每個個別項目。但是 `ToEnumerable` 擴充方法更進一步：它使整個序列能夠透過傳統的 `foreach` 迴圈來取用：

```csharp
var period = TimeSpan.FromMilliseconds(200);
IObservable<long> source = Observable.Timer(TimeSpan.Zero, period).Take(5);
IEnumerable<long> result = source.ToEnumerable();

foreach (long value in result)
{
    Console.WriteLine(value);
}

Console.WriteLine("done");
```

輸出：

```
0
1
2
3
4
done
```

當你開始列舉序列時（即延遲），將訂閱來源可觀察序列。
如果尚無可用元素，或者你已取用了到目前為止產生的所有元素，則 `foreach` 對列舉程式的 `MoveNext` 進行的呼叫將封鎖，直到來源產生元素為止。因此，這種方法依賴來源能夠從某個其他執行緒產生元素。（在此範例中，`Timer` 預設為 [`DefaultScheduler`](11_SchedulingAndThreading.md#defaultscheduler)，該排程器在執行緒集區上執行計時工作。）如果序列產生值的速度快於你取用它們的速度，它們將為你排入佇列。（這意味著使用 `ToEnumerable` 時在技術上可以在同一執行緒上取用和產生項目，但這將依賴生產者始終保持領先。這將是一種危險的方法，因為如果 `foreach` 迴圈追趕上，它將隨後發生死結。）

與 `await` 和 `ForEachAsync` 一樣，如果來源報告錯誤，這將被擲回，因此你可以使用普通的 C# 例外狀況處理，如下例所示：

```csharp
try 
{ 
    foreach (long value in result)
    { 
        Console.WriteLine(value); 
    } 
} 
catch (Exception e) 
{ 
    Console.WriteLine(e.Message);
} 
```

## 轉換為單一集合 (To a single collection)

有時你會希望將來源產生的所有項目作為單一清單。例如，也許你不能只單獨處理元素，因為你係時需要回頭參考較早收到的元素。以下幾節中描述的四個操作將所有項目收集到單一集合中。它們都仍然產生 `IObservable<T>`（例如 `IObservable<int[]>` 或 `IObservable<Dictionary<string, long>>`），但這些都是單一元素可觀察序列，並且如你所見，你可以使用 `await` 關鍵字來取得此單一輸出。

### ToArray 與 ToList

`ToArray` 和 `ToList` 分別採用可觀察序列並將其打包為陣列或 `List<T>` 的執行個體。與所有單一集合操作一樣，這些操作返回一個可觀察來源，該來源等待其輸入序列完成，然後將陣列或清單作為單一值產生，之後它們立即完成。此範例使用 `ToArray` 將來自來源序列的所有 5 個元素收集到一個陣列中，並使用 `await` 從 `ToArray` 返回的序列中擷取該陣列：

```csharp
TimeSpan period = TimeSpan.FromMilliseconds(200);
IObservable<long> source = Observable.Timer(TimeSpan.Zero, period).Take(5);
IObservable<long[]> resultSource = source.ToArray();

long[] result = await resultSource;
foreach (long value in result)
{
    Console.WriteLine(value);
}
```

輸出：

```
0
1
2
3
4
```

由於這些方法仍然返回可觀察序列，因此你也可以使用常規 Rx `Subscribe` 機制，或將這些用作其他運算子的輸入。

如果來源產生值然後發生錯誤，你將不會收到任何這些值。到那時為止接收到的所有值都將被捨棄，並且運算子將叫用其觀察者的 `OnError`（在上面的範例中，這將導致從 `await` 擲回例外狀況）。所有四個運算子（`ToArray`、`ToList`、`ToDictionary` 和 `ToLookup`）都以這種方式處理錯誤。

### ToDictionary 與 ToLookup

Rx 可以使用 `ToDictionary` 和 `ToLookup` 方法將可觀察序列封裝到字典或查詢查閱中。這兩種方法採用與 `ToArray` 和 `ToList` 方法相同的基本方法：它們返回一個單元素序列，一旦輸入來源完成，該序列就會產生集合。

`ToDictionary` 提供了四個多載，這些多載直接對應於 LINQ to Objects 定義的 `IEnumerable<T>` 的 `ToDictionary` 擴充方法：

```csharp
// Creates a dictionary from an observable sequence according to a specified 
// key selector function, a comparer, and an element selector function.
public static IObservable<IDictionary<TKey, TElement>> ToDictionary<TSource, TKey, TElement>(
    this IObservable<TSource> source, 
    Func<TSource, TKey> keySelector, 
    Func<TSource, TElement> elementSelector, 
    IEqualityComparer<TKey> comparer) 
{...} 

// Creates a dictionary from an observable sequence according to a specified
// key selector function, and an element selector function. 
public static IObservable<IDictionary<TKey, TElement>> ToDictionary<TSource, TKey, TElement>( 
    this IObservable<TSource> source, 
    Func<TSource, TKey> keySelector, 
    Func<TSource, TElement> elementSelector) 
{...} 

// Creates a dictionary from an observable sequence according to a specified
// key selector function, and a comparer. 
public static IObservable<IDictionary<TKey, TSource>> ToDictionary<TSource, TKey>( 
    this IObservable<TSource> source, 
    Func<TSource, TKey> keySelector,
    IEqualityComparer<TKey> comparer) 
{...} 

// Creates a dictionary from an observable sequence according to a specified 
// key selector function. 
public static IObservable<IDictionary<TKey, TSource>> ToDictionary<TSource, TKey>( 
    this IObservable<TSource> source, 
    Func<TSource, TKey> keySelector) 
{...} 
```

`ToLookup` 擴充方法提供了外觀幾乎相同的多載，差別在於傳回型別（顯然還有名稱）。它們都傳回 `IObservable<ILookup<TKey, TElement>>`。與 LINQ to Objects 一樣，字典與查詢查閱 (lookup) 之間的區別在於 `ILookup<TKey, TElement>>` 介面允許每個鍵具有任意數量的值，而字典將每個鍵對應到一個值。

## ResetExceptionDispatchState

`ResetExceptionDispatchState` 指示 Rx 在將傳遞給 `OnError` 的任何例外狀況傳遞給其觀察者之前，重設該例外狀況的_例外狀況發派狀態 (exception dispatch state)_。除此之外，此運算子只是未修改地傳遞所有通知。

此運算子的目的是使其更容易符合上述[例外狀況狀態](#exception-state)一節中描述的規則。通常只有在例外狀況源自 Rx 可觀察序列而從未執行過 `throw`，且你隨後導致該例外狀況脫離 Rx 的世界並以正常的 .NET 方式重新擲回時，才需要使用它。

例如，[`Observable.Throw`](03_CreatingObservableSequences.md#observablethrow) 不會重設你提供給它的例外狀況的狀態（以防該狀態中有任何重要的內容），但是如果你可能正在使用像 `await` 這樣會重新擲回例外狀況的跨界機制，並且如果你可能對同一個例外狀況物件多次這樣做，你將需要使用此運算子來確保每次都重設例外狀況狀態。

這在配合使用 [`Replay`](15_PublishingOperators.md#replay) 運算子時也很有用，因為即使底層來源使用的是 `throw`（從而重設了例外狀況狀態），`Replay` 也會保留該例外狀況，並將相同的物件傳遞給任何後續的訂閱者。使用 `Replay` 有效地阻止了本來會發生的重設。因此，如果你隨後繼續使用 `await`（或本章中任何其他會導致傳遞給 `OnError` 的例外狀況被重新擲回的機制），你將不再遵循[例外狀況狀態](#exception-state)中描述的規則，這可能會導致例如堆疊追蹤不斷增長等問題。

如果你完全留在 Rx 的世界中，則不需要使用 `ResetExceptionDispatchState`。此運算子的存在僅是為了解決從 Rx 的世界跨越到傳統例外狀況擲回的世界時可能發生的問題。

## ToTask

儘管 Rx 提供了將 `await` 與 `IObservable<T>` 一起使用的直接支援，但有時取得代表 `IObservable<T>` 的 `Task<T>` 可能很有用。這很有用，因為某些 API 需要 `Task<T>`。你可以在任何 `IObservable<T>` 上呼叫 `ToTask()`，這將訂閱該可觀察序列，返回一個在工作完成時完成的 `Task<T>`，產生序列的最終輸出作為該工作的結果。如果來源在未產生元素的情況下完成，則該工作將進入出錯狀態，並帶有抱怨輸入序列不包含任何元素的 `InvalidOperation` 例外狀況。

你可以選擇性地傳遞取消權杖。如果你在可觀察序列完成之前取消此權杖，Rx 將取消訂閱來源，並將工作置於已取消狀態。

這是如何使用 `ToTask` 運算子的簡單範例。
請注意，`ToTask` 方法位於 `System.Reactive.Threading.Tasks` 命名空間中。

```csharp
IObservable<long> source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(5);
Task<long> resultTask = source.ToTask();
long result = await resultTask; // Will take 5 seconds. 
Console.WriteLine(result);
```

輸出：

```
4
```

如果來源序列呼叫 `OnError`，Rx 將使用提供的例外狀況使工作處於出錯狀態。

一旦擁有了工作，你當然可以使用 TPL 的所有功能，例如接續 (continuations)。

## ToEvent

就像你可以使用 [`FromEventPattern`](03_CreatingObservableSequences.md#from-events) 將事件用作可觀察序列的來源一樣，你也可以使用 `ToEvent` 擴充方法使你的可觀察序列看起來像標準的 .NET 事件。

```csharp
// Exposes an observable sequence as an object with a .NET event. 
public static IEventSource<unit> ToEvent(this IObservable<Unit> source)
{...}

// Exposes an observable sequence as an object with a .NET event. 
public static IEventSource<TSource> ToEvent<TSource>(this IObservable<TSource> source) 
{...}
```

`ToEvent` 方法返回一個 `IEventSource<T>`，它具有單一成員：一個 `OnNext` 事件。

```csharp
public interface IEventSource<T> 
{ 
    event Action<T> OnNext; 
} 
```

當我們使用 `ToEvent` 方法轉換可觀察序列時，我們只需提供一個 `Action<T>` 即可進行訂閱，我們在此處使用 Lambda 來執行此操作。

```csharp
var source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(5); 
var result = source.ToEvent(); 
result.OnNext += val => Console.WriteLine(val);
```

輸出：

```
0
1
2
3
4
```

儘管這是將 Rx 通知轉換為事件的最簡單方法，但它不遵循標準的 .NET 事件模式。如果我們想要標準模式，我們可以使用稍微不同的方法。

### ToEventPattern

通常，.NET 事件會向其處理常式提供 `sender` 和 `EventArgs` 引數。在上面的範例中，我們只獲取值。如果你想將序列公開為遵循標準模式的事件，則需要使用 `ToEventPattern`。

```csharp
// Exposes an observable sequence as an object with a .NET event. 
public static IEventPatternSource<TEventArgs> ToEventPattern<TEventArgs>(
    this IObservable<EventPattern<TEventArgs>> source) 
    where TEventArgs : EventArgs 
```

`ToEventPattern` 將採用 `IObservable<EventPattern<TEventArgs>>` 並將其轉換為 `IEventPatternSource<TEventArgs>`。這些類型的公用介面非常簡單。

```csharp
public class EventPattern<TEventArgs> : IEquatable<EventPattern<TEventArgs>>
    where TEventArgs : EventArgs 
{ 
    public EventPattern(object sender, TEventArgs e)
    { 
        this.Sender = sender; 
        this.EventArgs = e; 
    } 
    public object Sender { get; private set; } 
    public TEventArgs EventArgs { get; private set; } 
    //...equality overloads
} 

public interface IEventPatternSource<TEventArgs> where TEventArgs : EventArgs
{ 
    event EventHandler<TEventArgs> OnNext; 
} 
```

要使用它，我們將需要一個合適的 `EventArgs` 類型。你可能可以使用 .NET 執行階段函式庫提供的類型，但如果沒有，你可以輕鬆編寫自己的類型：

`EventArgs` 類型：

```csharp
public class MyEventArgs : EventArgs 
{ 
    private readonly long _value; 
    
    public MyEventArgs(long value) 
    { 
        _value = value; 
    } 

    public long Value 
    { 
        get { return _value; } 
    } 
} 
```

然後，我們可以透過使用 `Select` 套用簡單的轉換來在 Rx 中使用它：

```csharp
IObservable<EventPattern<MyEventArgs>> source = 
   Observable.Interval(TimeSpan.FromSeconds(1))
             .Select(i => new EventPattern<MyEventArgs>(this, new MyEventArgs(i)));
```

現在我們有了一個相容的序列，我們可以使用 `ToEventPattern`，進而使用標準的事件處理常式。

```csharp
IEventPatternSource<MyEventArgs> result = source.ToEventPattern(); 
result.OnNext += (sender, eventArgs) => Console.WriteLine(eventArgs.Value);
```

現在我們知道如何返回 .NET 事件，讓我們暫停一下，記住為什麼 Rx 是一個更好的模型。

- 事件難以組合
- 事件不能作為引數傳遞或儲存在欄位中
- 事件無法輕鬆地隨著時間推移進行查詢
- 事件沒有報告錯誤的標準模式
- 事件沒有指示值序列結束的標準模式
- 事件對於管理並行性或多執行緒應用程式幾乎沒有任何幫助

## Do

正式環境系統的非功能性需求通常要求高可用性、品質監視功能以及低缺陷解決前置時間。記錄、偵錯、檢測和日誌記錄是實作非功能性需求的常見實作選擇。為了實現這些功能，「切入 (tap into)」你的 Rx 查詢通常很有用，使其在正常運作的副作用下傳遞監視和診斷資訊。

`Do` 擴充方法允許你插入副作用行為。從 Rx 的角度來看，`Do` 似乎什麼也沒做：你可以將它套用於任何 `IObservable<T>`，它會返回另一個 `IObservable<T>`，該可觀察序列報告與其來源完全相同的元素以及錯誤或完成。但是，它的各種多載採用看起來與 `Subscribe` 的回呼引數完全相同的引數：你可以為個別項目、完成和錯誤提供回呼函式。與 `Subscribe` 不同，`Do` 不是最終目的地——`Do` 回呼函式看到的一切也將轉發給 `Do` 的訂閱者。這使得它對於記錄和類似的檢測非常有用，因為你可以使用它來報告資訊如何在 Rx 查詢中流動，而無需變更該查詢的行為。

當然，你必須小心一點。使用 `Do` 會對效能產生影響。如果你提供給 `Do` 的回呼函式執行了可能變更其所屬 Rx 查詢之輸入的任何操作，你將建立一個回饋迴圈，使行為完全難以理解。

首先讓我們定義一些記錄方法，以便我們可以在範例中繼續使用：

```csharp
private static void Log(object onNextValue)
{
    Console.WriteLine($"Logging OnNext({onNextValue}) @ {DateTime.Now}");
}

private static void Log(Exception error)
{
    Console.WriteLine($"Logging OnError({error}) @ {DateTime.Now}");
}

private static void Log()
{
    Console.WriteLine($"Logging OnCompleted()@ {DateTime.Now}");
}
```

此程式碼使用 `Do` 並利用上述方法引入了一些記錄。

```csharp
IObservable<long> source = Observable.Interval(TimeSpan.FromSeconds(1)).Take(3);
IObservable<long> loggedSource = source.Do(
    i => Log(i),
    ex => Log(ex),
    () => Log());

loggedSource.Subscribe(
    Console.WriteLine,
    () => Console.WriteLine("completed"));
```

輸出：

```
Logging OnNext(0) @ 01/01/2012 12:00:00
0
Logging OnNext(1) @ 01/01/2012 12:00:01
1
Logging OnNext(2) @ 01/01/2012 12:00:02
2
Logging OnCompleted() @ 01/01/2012 12:00:02
completed
```

請注意，因為 `Do` 是查詢的一部分，所以它必然比作為鏈結中最後一環的 `Subscribe` 更早看到值。這就是記錄訊息出現在 `Subscribe` 回呼函式產生的行之前的原因。我喜歡將 `Do` 方法視為序列的[竊聽器 (wire tap)](http://en.wikipedia.org/wiki/Telephone_tapping)。它使你能夠在不修改序列的情況下傾聽序列。

與 `Subscribe` 一樣，除了傳遞回呼函式之外，還有一些多載可讓你為所需的 OnNext、OnError 和 OnCompleted 通知提供回呼函式，或者你可以向 `Do` 傳遞 `IObserver<T>`。

## 使用 AsObservable 進行封裝 (Encapsulating with AsObservable)

不良的封裝是開發人員為錯誤敞開大門的一種方式。以下是少數因粗心而導致抽象洩漏的情境。我們第一個範例乍看之下可能無害，但存在許多問題。

```csharp
public class UltraLeakyLetterRepo
{
    public ReplaySubject<string> Letters { get; }

    public UltraLeakyLetterRepo()
    {
        Letters = new ReplaySubject<string>();
        Letters.OnNext("A");
        Letters.OnNext("B");
        Letters.OnNext("C");
    }
}
```

在此範例中，我們將可觀察序列公開為屬性。我們使用了 `ReplaySubject<string>`，以便每個訂閱者在訂閱時都會收到所有值。然而，在 `Letters` 屬性的公用類型中揭示此實作選擇是不良的封裝，因為取用端可以呼叫 `OnNext`/`OnError`/`OnCompleted`。為了消除該漏洞，我們可以簡單地將公開可見的屬性類型設為 `IObservable<string>`。

```csharp
public class ObscuredLeakinessLetterRepo
{
    public IObservable<string> Letters { get; }

    public ObscuredLeakinessLetterRepo()
    {
        var letters = new ReplaySubject<string>();
        letters.OnNext("A");
        letters.OnNext("B");
        letters.OnNext("C");
        this.Letters = letters;
    }
}
```

這是一個顯著的改進：編譯器不會讓使用此來源執行個體的人編寫 `source.Letters.OnNext("1")`。因此 API 介面區域適當地封裝了實作詳細資訊，但如果我們偏執一點，我們注意到沒有任何東西可以阻止取用端將結果轉型回 `ISubject<string>`，然後呼叫他們喜歡的任何方法。在此範例中，我們看到外部程式碼將其值推入序列中。

```csharp
var repo = new ObscuredLeakinessLetterRepo();
IObservable<string> good = repo.GetLetters();
    
good.Subscribe(Console.WriteLine);

// Be naughty
if (good is ISubject<string> evil)
{
    // So naughty, 1 is not a letter!
    evil.OnNext("1");
}
else
{
    Console.WriteLine("could not sabotage");
}
```

輸出：

```
A
B
C
1
```

可以說，做這類事情的程式碼是在自討苦吃，但如果我們想主動預防它，這個問題的修復非常簡單。透過套用 `AsObservable` 擴充方法，我們可以修改建構函式中設定 `this.Letters` 的那一行，將 Subject 包裝在僅實作 `IObservable<T>` 的類型中。

```csharp
this.Letters = letters.AsObservable();
```

輸出：

```
A
B
C
could not sabotage
```

雖然我在這些範例中使用了「邪惡 (evil)」和「破壞 (sabotage)」等詞彙，但[通常是由於疏忽而非惡意](https://en.wikipedia.org/wiki/Hanlon%27s_razor)導致問題。失敗首先落在設計了存在洩漏之類別的程式設計師身上。設計介面很困難，但我們應該盡力幫助程式碼的取用端落入[成功之坑 (the pit of success)](https://learn.microsoft.com/en-gb/archive/blogs/brada/the-pit-of-success)，為他們提供可發現且一致的類型。如果我們縮小類型的介面區域以僅公開我們打算讓取用端使用的功能，則類型會變得更易於發現。在此範例中，我們減少了該類型的介面區域。我們透過為屬性選擇合適的面向大眾的類型，然後使用 `AsObservable` 方法防止存取底層類型來實現這一點。

我們在本章中檢視的這組方法完成了在[建立序列章節](03_CreatingObservableSequences.md)中開始的循環。我們現在擁有了進出 Rx 世界的方法。在選擇加入和離開 `IObservable<T>` 時要小心。最好不要來回轉換——進行一些基於 Rx 的處理，然後是一些更傳統的程式碼，然後將其結果重新導入 Rx 中，這很快就會使你的程式碼庫變得混亂，並可能表示存在設計缺陷。通常最好將所有 Rx 邏輯保持在一起，因此你只需要與外界整合兩次：一次用於輸入，一次用於輸出。
