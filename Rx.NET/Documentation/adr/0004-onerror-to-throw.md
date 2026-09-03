# OnError 通知何時變為擲回例外狀況的規則

Rx 以稍微不尋常的方式使用 .NET `Exception` 物件：它們通常不會被擲回。相反地，它們作為引數傳遞給觀察者的 `OnError` 方法。在某些情況下，以此種方式回報的錯誤最終會導致擲回例外狀況。例如，如果您 `await` 一個呼叫了 `OnError` 的 `IObservable<T>`，則該 `await` 將會擲回：

```cs
IObservable<int> ts = Observable.Throw<int>(new Exception("Pow!"));

await ts; // Exception thrown here
```

這可能會導致問題。例如，如 [#2187](https://github.com/dotnet/reactive/issues/2187) 中所描述，如果您多次 `await` 此範例中所示的 observable，則每次例外狀況的 `StackTrace` 都會變長。

出現問題的原因是，即使直接使用 `throw`，使用單例例外狀況物件也有點棘手，但是當您開始將一般 .NET 例外狀況處理與 Rx 在 `OnError` 中使用 `Exception`「交叉混用」時，情況會變得更加微妙。

Rx 先前從未提供任何指引讓開發人員能夠理解上面顯示的程式碼可能存在問題。本 ADR 的目的是建立適當的規則。

## 狀態

已提議


## 作者

@idg10（[Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/)）。



## 背景

例外狀況看似是一般的 .NET 物件，但它們在執行階段中會獲得特殊處理。MSIL 有一個用於引發例外狀況的 [`throw`](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.emit.opcodes.throw?view=net-9.0) 指令，並且 CLR 內部有 C++ 程式碼直接操作由 `Exception` 類別定義的欄位。圍繞例外狀況型別使用的某些預期已深深根植於執行階段中。

Rx 通常不會以執行階段預期的方式使用例外狀況。特別是，它通常不使用 MSIL 的 `throw` 指令來引發例外狀況。相反地，當 Rx `IObservable<T>` 想要回報錯誤時，它只會將例外狀況物件作為引數傳遞給 `IObserver<T>.OnError` 方法。

當應用程式完全保留在 Rx 的世界中時，這不會造成任何問題。但是當我們想要轉向更傳統的擲回例外狀況的 .NET 方法時，它提出了一個有趣的問題：例外狀況看起來應該源自何處？

考慮這個範例：

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

如果嘗試開啟檔案擲回了例外狀況，我們期望看到什麼？熟悉 .NET 中例外狀況通常如何與 `async` 搭配運作的開發人員可能合理地預期該例外狀況會回報兩個堆疊追蹤（stack trace）：一個用於最初擲回例外狀況的點，另一個用於從 `await` 重新擲回例外狀況的位置。而這正是我們所看到的：

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

這很直接，因為此處的例外狀況是以傳統的 .NET 方式擲回的。它碰巧被 Rx 攔截——`Observable.Create` 的這個多載將回呼傳回的 `Task` 包裝在一個配接器中，該配接器偵測 `Task` 何時進入出錯（faulted）狀態，在此情況下，它會擷取例外狀況並將其傳遞給訂閱的 `IObserver<T>`。然後，當您 `await` 一個 observable 時 Rx 提供的 awaiter 會重新擲回同一個例外狀況。

但是對於前面的範例（其中例外狀況源自 `Observable.Throw`），情況又如何呢？在該程式碼中，我們建構了一個 `Exception`，但我們從未對其使用 `throw` 關鍵字，也沒有叫用任何可能為我們執行此操作的 API。在這種情況下，您期望呼叫堆疊顯示什麼？在實務上，我們得到這個結果：

```
Unhandled exception. System.Exception: Pow!
   at System.Reactive.PlatformServices.ExceptionServicesImpl.Rethrow(Exception exception)
   at System.Reactive.ExceptionHelpers.Throw(Exception exception)
   at System.Reactive.Subjects.AsyncSubject`1.GetResult()
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\RxThrowExamples\Program.cs:line 20
   at Program.<Main>(String[] args)
```

這一次，我們只獲得了一個單一的堆疊追蹤，有效地顯示了 `await`。這看起來與前一個範例中的第二個追蹤非常相似——此處的差異在於我們沒有額外的追蹤顯示最初擲回例外狀況的原始位置。而且您可以認為這是有道理的：這個特定的例外狀況直到它從 `await` 出現時才被擲回。

到目前為止一切都很好。但是看看如果我們多次使用同一個 observable 來源會發生什麼：

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

由於我們做的是相同的事情三次，您可能期望看到相同的例外狀況報告三次。但實際發生的情況並非如此：

```
System.Exception: Pow!
   at System.Reactive.PlatformServices.ExceptionServicesImpl.Rethrow(Exception exception)
   at System.Reactive.ExceptionHelpers.Throw(Exception exception)
   at System.Reactive.Subjects.AsyncSubject`1.GetResult()
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\RxThrowExamples\Program.cs:line 27


System.Exception: Pow!
   at System.Reactive.PlatformServices.ExceptionServicesImpl.Rethrow(Exception exception)
   at System.Reactive.ExceptionHelpers.Throw(Exception exception)
   at System.Reactive.Subjects.AsyncSubject`1.GetResult()
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\RxThrowExamples\Program.cs:line 27
   at System.Reactive.PlatformServices.ExceptionServicesImpl.Rethrow(Exception exception)
   at System.Reactive.ExceptionHelpers.Throw(Exception exception)
   at System.Reactive.Subjects.AsyncSubject`1.GetResult()
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\RxThrowExamples\Program.cs:line 27


System.Exception: Pow!
   at System.Reactive.PlatformServices.ExceptionServicesImpl.Rethrow(Exception exception)
   at System.Reactive.ExceptionHelpers.Throw(Exception exception)
   at System.Reactive.Subjects.AsyncSubject`1.GetResult()
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\RxThrowExamples\Program.cs:line 27
   at System.Reactive.PlatformServices.ExceptionServicesImpl.Rethrow(Exception exception)
   at System.Reactive.ExceptionHelpers.Throw(Exception exception)
   at System.Reactive.Subjects.AsyncSubject`1.GetResult()
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\RxThrowExamples\Program.cs:line 27
   at System.Reactive.PlatformServices.ExceptionServicesImpl.Rethrow(Exception exception)
   at System.Reactive.ExceptionHelpers.Throw(Exception exception)
   at System.Reactive.Subjects.AsyncSubject`1.GetResult()
   at Program.<Main>$(String[] args) in D:\source\RxThrowExamples\RxThrowExamples\Program.cs:line 27
```

堆疊追蹤每次都變長了！

（這並未準確反映實際的執行階段行為：呼叫堆疊實際上並沒有變得更深。只不過是 `Exception` 用來回報此資訊的 `StackTrace` 字串最終包含了堆疊追蹤的多個複本。）

這毫無道理。

這是 Rx 為了在前一個範例中產生我們期望的堆疊追蹤所採取之步驟的直接結果。它使用 .NET 執行階段函式庫的 `ExceptionDispatchInfo.Throw` 方法從 `await` 重新擲回例外狀況。該方法保留了最初擲回例外狀況的原始內容，並附加了重新擲回例外狀況的內容：這就是我們最終獲得 .NET 開發人員在正常使用 `async` 和 `await` 時所習慣的多個堆疊追蹤的方式。（事實上，Rx 使用的重新擲回機制與在傳統 `async` 程式碼中啟用此行為的機制完全相同。）

這種行為並非 Rx 所特有。它源自於 `ExceptionDispatchInfo.Throw`，而且我們可以在不使用 Rx 的情況下建立此行為的基於 `Task` 的版本：

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

堆疊追蹤較短，但我們看到了相同的重複行為：

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

（準確地說，如果您建立單一 `Task` 並多次 `await` 它，這*不會*發生。是 `ExceptionDispatchInfo.Capture` 和 `ExceptionDispatchInfo.Throw` 的結合導致了這種累積，而這個 `Task` 在我們使用 `Task.FromException` 建立它時就擷取了例外狀況資訊。）

這種附加例外狀況資料是特意設計的：`ExceptionDispatchInfo.Throw` 旨在將目前內容附加到在 `ExceptionDispatchInfo` 中擷取的任何內容之後。.NET 執行階段假設如果您想要表示一個新的例外事件，您將會執行一個 `throw`。Rx 在 `await` 中（或在可以重新擲回透過 `OnError` 傳遞的例外狀況的其他機制中，例如 `ToEnumerable`）不會這樣做，正是因為它保留了接收到例外狀況時存在的任何內容。它不執行 `throw`（或執行任何其他操作來重設例外狀況內容），因為這會阻止在範例（例如前面顯示的 `FileNotFoundException` 處理）中保留完整的內容。

這種行為在其設計的背景下是有意義的——擷取最初擲回例外狀況的內容，並在從不同內容重新擲回時以額外資訊對其進行擴充。但除非您意識到這一點，否則這一點根本不明顯：儘管使用 `Observable.Throw<int>()` 本身沒有任何問題，但它與擁有多個各自會重新擲回該例外狀況的訂閱者是不相容的。


## 決策

Rx.NET 將明確採取這一立場：如果使用 Rx 的開發人員選擇使用一種接收由 `IObservable<T>` 傳遞的例外狀況並將其擲回的機制（例如，如果您 `await` 一個 `IObservable<T>`），則開發人員有責任確保以下兩者之一：

* 每個例外狀況物件僅使用一次

或者

* 在將例外狀況提供給將重新擲回該例外狀況的觀察者之前，重設該例外狀況的派遣狀態（例如，透過執行 `throw`）

由於 Rx 定義了不符合第一個選項的運算子（特別是 `Observable.Throw`，還有 `ReplaySubject` 和相關的 `Observable.Replay`），因此 Rx 6.1 引入了一個新的運算子 `ResetExceptionDispatchState`。這會傳遞所有通知，但在轉發任何 `Exception` 之前有效地對其執行 `throw`。它可以像這樣使用：

```cs
var ts = Observable.Throw<int>(new Exception("Aaargh!")).ResetExceptionDispatchState();
```

當觀察者訂閱它時，`Throw` 會立即呼叫 `OnError`，而 `ResetExceptionDispatchState` 將在將該例外狀況傳遞給訂閱者之前擲回（並立即攔截）該例外狀況。（您*不會*在諸如前面顯示的 `Create` 範例之類的情境中使用它，因為在這種情況下，每個例外狀況都是新擲回的，並且具有有用的內容資訊，因此我們不想重設它。這專門用於否則不會擲回該例外狀況的情況。）


## 後果

透過採取這個立場，我們明確表示類似 [#2187](https://github.com/dotnet/reactive/issues/2187) 中的範例不被預期能正常運作。

更廣泛地說，這澄清了重複產生相同例外狀況物件的 observable 來源（例如 `Observable.Throw` 或 `Observable.Repeat`）與對 `await` 的多次呼叫是不相容的。

新增 `ResetExceptionDispatchState` 運算子提供了一種清晰、簡單的方法來修復遇到此問題的程式碼。
