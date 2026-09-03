# 發布運算子 (Publishing operators)

熱 (Hot) 來源需要能夠將事件傳遞給多個訂閱者。雖然我們可以自己實作訂閱者追蹤，但編寫一個僅適用於單一訂閱者的過度簡化的來源可能會更容易。儘管這不是 `IObservable<T>` 的完整實作，但如果我們隨後使用 Rx 的_多播 (multicast)_運算子之一將其發布為多訂閱者熱來源，那也沒關係。在[「在 Rx 中表示檔案系統事件」](03_CreatingObservableSequences.md#representing-filesystem-events-in-rx)中的範例使用了這個技巧，但正如你在本章中將看到的，這個主題有幾種變化。

## Multicast

Rx 提供了三個運算子，使我們能夠僅使用對某個底層來源的單一訂閱來支援多個訂閱者：[`Publish`](#publish)、[`PublishLast`](#publishlast) 和 [`Replay`](#replay)。這三者都是 Rx 的 `Multicast` 運算子的包裝函式，該運算子提供了所有這些運算子核心的通用機制。

`Multicast` 將任何 `IObservable<T>` 轉換為 `IConnectableObservable<T>`，如你所見，它只是新增了一個 `Connect` 方法：

```csharp
public interface IConnectableObservable<out T> : IObservable<T>
{
    IDisposable Connect();
}
```

由於它衍生自 `IObservable<T>`，你可以在 `IConnectableObservable<T>` 上呼叫 `Subscribe`，但是當你這樣做時，由 `Multicast` 返回的實作不會在底層來源上呼叫 `Subscribe`。它僅在你呼叫 `Connect` 時才在底層來源上呼叫 `Subscribe`。為了讓我們看到這一點的實際運作，讓我們定義一個在每次呼叫 `Subscribe` 時輸出訊息的來源：

```csharp
IObservable<int> src = Observable.Create<int>(obs =>
{
    Console.WriteLine("Create callback called");
    obs.OnNext(1);
    obs.OnNext(2);
    obs.OnCompleted();
    return Disposable.Empty;
});
```

由於無論有多少觀察者訂閱，這都只會被叫用一次，因此 `Multicast` 無法傳遞傳遞給其自身 `Subscribe` 方法的 `IObserver<T>`，因為可能有任意數量的觀察者。它使用一個 [Subject](03_CreatingObservableSequences.md#subjectt) 作為傳遞到底層來源的單一 `IObserver<T>`，並且此 Subject 還負責追蹤所有訂閱者。如果我們直接呼叫 `Multicast`，我們需要傳入我們想要使用的 Subject：

```csharp
IConnectableObservable<int> m = src.Multicast(new Subject<int>());
```

我們現在可以對此訂閱幾次：

```csharp
m.Subscribe(x => Console.WriteLine($"Sub1: {x}"));
m.Subscribe(x => Console.WriteLine($"Sub2: {x}"));
m.Subscribe(x => Console.WriteLine($"Sub3: {x}"));
```

除非我們呼叫 `Connect`，否則這些訂閱者都不會收到任何內容：

```csharp
m.Connect();
```

**注意**：`Connect` 返回一個 `IDisposable`。對其呼叫 `Dispose` 會取消對底層來源的訂閱。

對 `Connect` 的此呼叫會產生以下輸出：

```csharp
Create callback called
Sub1: 1
Sub2: 1
Sub3: 1
Sub1: 2
Sub2: 2
Sub3: 2
```

如你所見，我們傳遞給 `Create` 的方法僅執行一次，證實儘管我們呼叫了三次 `Subscribe`，但 `Multicast` 確實只訂閱了一次。但是每個項目都發送到了所有三個訂閱。

`Multicast` 的運作方式相當簡單：它讓 Subject 完成大部分工作。每當你在 `Multicast` 返回的可觀察序列上呼叫 `Subscribe` 時，它只會在 Subject 上呼叫 `Subscribe`。當你呼叫 `Connect` 時，它只會將 Subject 傳遞到底層來源的 `Subscribe` 中。因此這段程式碼具有相同的效果：

```csharp
var s = new Subject<int>();

s.Subscribe(x => Console.WriteLine($"Sub1: {x}"));
s.Subscribe(x => Console.WriteLine($"Sub2: {x}"));
s.Subscribe(x => Console.WriteLine($"Sub3: {x}"));

src.Subscribe(s);
```

然而，`Multicast` 的一個優點是它返回 `IConnectableObservable<T>`，並且正如我們稍後將看到的，Rx 的其他一些部分知道如何使用此介面。

`Multicast` 提供了一個以截然不同方式運作的多載：它適用於你想要編寫兩次使用其來源可觀察序列的查詢的情境。例如，我們可能想要使用 `Zip` 取得相鄰的項目組：

```csharp
IObservable<(int, int)> ps = src.Zip(src.Skip(1));
ps.Subscribe(ps => Console.WriteLine(ps));
```

（雖然 [`Buffer`](08_Partitioning.md#buffer) 似乎是執行此操作的更明顯方式，但此 `Zip` 方法的一個優點是它永遠不會給我們半組。當我們向 `Buffer` 請求配對時，當我們到達末尾時它會給我們一個單項目緩衝區，這可能需要額外的程式碼來解決。）

這種方法的問題在於來源將看到兩個訂閱：一個直接來自 `Zip`，然後第二個透過 `Skip`。如果我們要執行上面的程式碼，我們將看到此輸出：

```
Create callback called
Create callback called
(1, 2)
```

我們的 `Create` 回呼函式執行了兩次。第二個 `Multicast` 多載使我們能夠避免這種情況：

```csharp
IObservable<(int, int)> ps = src.Multicast(() => new Subject<int>(), s => s.Zip(s.Skip(1)));
ps.Subscribe(ps => Console.WriteLine(ps));
```

如輸出所示，這避免了多個訂閱：

```csharp
Create callback called
(1, 2)
```

`Multicast` 的這個多載返回一個正常的 `IObservable<T>`。這意味著我們不需要呼叫 `Connect`。但這也意味著對產生的 `IObservable<T>` 的每次訂閱都會導致對底層來源的訂閱。但對於它所設計的情境來說，這很好：我們只是試圖避免對底層來源產生兩倍的訂閱。

本節中定義的其餘運算子 `Publish`、`PublishLast` 和 `Replay` 都是圍繞 `Multicast` 的包裝函式，每個運算子都為你提供特定類型的 Subject。

### Publish

`Publish` 運算子使用 [`Subject<T>`](03_CreatingObservableSequences.md#subjectt) 呼叫 `Multicast`。其效果是，一旦你在結果上呼叫了 `Connect`，來源產生的任何項目都將傳遞給所有訂閱者。這使我能夠替換此前的範例：

```csharp
IConnectableObservable<int> m = src.Multicast(new Subject<int>());
```

改為：

```csharp
IConnectableObservable<int> m = src.Publish();
```

這些是完全等效的。

因為 `Subject<T>` 會立即將所有傳入的 `OnNext` 呼叫轉發給其每個訂閱者，並且因為它不儲存任何先前進行的呼叫，所以結果是一個熱 (hot) 來源。如果你在呼叫 `Connect` 之前附加了一些訂閱者，然後在呼叫 `Connect` 之後附加了更多訂閱者，那麼後面的訂閱者將只會收到在他們訂閱之後發生的事件。此範例展示了這一點：

```csharp
IConnectableObservable<long> publishedTicks = Observable
    .Interval(TimeSpan.FromSeconds(1))
    .Take(4)
    .Publish();

publishedTicks.Subscribe(x => Console.WriteLine($"Sub1: {x} ({DateTime.Now})"));
publishedTicks.Subscribe(x => Console.WriteLine($"Sub2: {x} ({DateTime.Now})"));

publishedTicks.Connect();
Thread.Sleep(2500);
Console.WriteLine();
Console.WriteLine("Adding more subscribers");
Console.WriteLine();

publishedTicks.Subscribe(x => Console.WriteLine($"Sub3: {x} ({DateTime.Now})"));
publishedTicks.Subscribe(x => Console.WriteLine($"Sub4: {x} ({DateTime.Now})"));
```

以下輸出顯示我們僅針對最後 2 個事件看到 Sub3 和 Sub4 訂閱的輸出：

```
Sub1: 0 (10/08/2023 16:04:02)
Sub2: 0 (10/08/2023 16:04:02)
Sub1: 1 (10/08/2023 16:04:03)
Sub2: 1 (10/08/2023 16:04:03)

Adding more subscribers

Sub1: 2 (10/08/2023 16:04:04)
Sub2: 2 (10/08/2023 16:04:04)
Sub3: 2 (10/08/2023 16:04:04)
Sub4: 2 (10/08/2023 16:04:04)
Sub1: 3 (10/08/2023 16:04:05)
Sub2: 3 (10/08/2023 16:04:05)
Sub3: 3 (10/08/2023 16:04:05)
Sub4: 3 (10/08/2023 16:04:05)
```

與 [`Multicast`](#multicast) 一樣，`Publish` 提供了一個多載，提供每個頂層訂閱多播。這使我們能夠將該節末尾的範例從：

```csharp
IObservable<(int, int)> ps = src.Multicast(() => new Subject<int>(), s => s.Zip(s.Skip(1)));
ps.Subscribe(ps => Console.WriteLine(ps));
```

簡化為：

```csharp
IObservable<(int, int)> ps = src.Publish(s => s.Zip(s.Skip(1)));
ps.Subscribe(ps => Console.WriteLine(ps));
```

`Publish` 提供了允許你指定初始值的多載。這些多載使用 [`BehaviorSubject<T>`](03_CreatingObservableSequences.md#behaviorsubjectt) 而非 `Subject<T>`。這裡的差異在於所有訂閱者在訂閱後將立即收到一個值。如果底層來源尚未產生項目（或者尚未呼叫 `Connect`，意味著我們甚至尚未訂閱來源），他們將收到初始值。如果已從來源接收到至少一個項目，任何新的訂閱者將立即收到來源產生的最新值，然後將繼續接收任何進一步的新值。

### PublishLast

`PublishLast` 運算子使用 [`AsyncSubject<T>`](03_CreatingObservableSequences.md#asyncsubjectt) 呼叫 `Multicast`。其效果是來源產生的最終項目將傳遞給所有訂閱者。你仍然需要呼叫 `Connect`。這決定了何時對底層來源進行訂閱。但無論何時訂閱，所有訂閱者都將收到最終事件，因為 `AsyncSubject<T>` 會記住最終結果。我們可以透過以下範例看到這一點的實際運作：

```csharp
IConnectableObservable<long> pticks = Observable
    .Interval(TimeSpan.FromSeconds(0.1))
    .Take(4)
    .PublishLast();

pticks.Subscribe(x => Console.WriteLine($"Sub1: {x} ({DateTime.Now})"));
pticks.Subscribe(x => Console.WriteLine($"Sub2: {x} ({DateTime.Now})"));

pticks.Connect();
Thread.Sleep(3000);
Console.WriteLine();
Console.WriteLine("Adding more subscribers");
Console.WriteLine();

pticks.Subscribe(x => Console.WriteLine($"Sub3: {x} ({DateTime.Now})"));
pticks.Subscribe(x => Console.WriteLine($"Sub4: {x} ({DateTime.Now})"));
```

這會建立一個在 0.4 秒內產生 4 個值的來源。它將幾個訂閱者附加到 `PublishLast` 返回的 `IConnectableObservable<T>`，然後立即呼叫 `Connect`。然後它休眠 1 秒鐘，這給了來源完成的時間。這意味著這前兩個訂閱者將在呼叫 `Thread.Sleep` 返回之前收到他們將接收到的唯一值（序列中的最後一個值）。但我們隨後繼續附加另外兩個訂閱者。如輸出所示，這些訂閱者也收到了完全相同的最終事件：

```
Sub1: 3 (11/14/2023 9:15:46 AM)
Sub2: 3 (11/14/2023 9:15:46 AM)

Adding more subscribers

Sub3: 3 (11/14/2023 9:15:49 AM)
Sub4: 3 (11/14/2023 9:15:49 AM)
```

最後這兩個訂閱者之所以較晚收到該值，是因為他們訂閱得較晚，但 `PublishLast` 建立的 `AsyncSubject<T>` 只是將其接收到的最終值重播給這些較晚的訂閱者。

### Replay

`Replay` 運算子使用 [`ReplaySubject<T>`](03_CreatingObservableSequences.md#replaysubjectt) 呼叫 `Multicast`。其效果是在呼叫 `Connect` 之前附加的任何訂閱者只會在底層來源產生事件時接收所有事件，但是稍後附加的任何訂閱者實際上都可以「追趕 (catch up)」，因為 `ReplaySubject<T>` 會記住它已經看到的事件，並將它們重播給新的訂閱者。

此範例與用於 `Publish` 的範例非常相似：

```csharp
IConnectableObservable<long> pticks = Observable
    .Interval(TimeSpan.FromSeconds(1))
    .Take(4)
    .Replay();

pticks.Subscribe(x => Console.WriteLine($"Sub1: {x} ({DateTime.Now})"));
pticks.Subscribe(x => Console.WriteLine($"Sub2: {x} ({DateTime.Now})"));

pticks.Connect();
Thread.Sleep(2500);
Console.WriteLine();
Console.WriteLine("Adding more subscribers");
Console.WriteLine();

pticks.Subscribe(x => Console.WriteLine($"Sub3: {x} ({DateTime.Now})"));
pticks.Subscribe(x => Console.WriteLine($"Sub4: {x} ({DateTime.Now})"));
```

這會建立一個將定期產生項目達 4 秒的來源。它在呼叫 `Connect` 之前附加兩個訂閱者。然後它等待足夠長的時間以使前兩個事件出現，然後附加另外兩個訂閱者。但與 `Publish` 不同的是，那些遲到的訂閱者將看到在他們訂閱之前發生的事件：

```
Sub1: 0 (10/08/2023 16:18:22)
Sub2: 0 (10/08/2023 16:18:22)
Sub1: 1 (10/08/2023 16:18:23)
Sub2: 1 (10/08/2023 16:18:23)

Adding more subscribers

Sub3: 0 (10/08/2023 16:18:24)
Sub3: 1 (10/08/2023 16:18:24)
Sub4: 0 (10/08/2023 16:18:24)
Sub4: 1 (10/08/2023 16:18:24)
Sub1: 2 (10/08/2023 16:18:24)
Sub2: 2 (10/08/2023 16:18:24)
Sub3: 2 (10/08/2023 16:18:24)
Sub4: 2 (10/08/2023 16:18:24)
Sub1: 3 (10/08/2023 16:18:25)
Sub2: 3 (10/08/2023 16:18:25)
Sub3: 3 (10/08/2023 16:18:25)
Sub4: 3 (10/08/2023 16:18:25)
```

他們當然較晚收到它們，因為他們訂閱得較晚。因此我們看到隨著 `Sub3` 和 `Sub4` 追趕進度而報告的一連串快速事件，但一旦它們追趕上，它們隨後就會立即收到所有進一步的事件。

啟用此行為的 `ReplaySubject<T>` 將耗用記憶體來儲存事件。你可能還記得，此 Subject 類型可以設定為僅儲存有限數量的事件，或者不保留早於某些指定時間限制的事件。`Replay` 運算子提供了使你能夠設定此類限制的多載。

`Replay` 還支援我為本節中其他基於 `Multicast` 的運算子展示的每個訂閱多播模型。

請注意，如果來源報告錯誤（透過呼叫 `OnError`），`Replay` 將保留該 `Exception`，並將其提供給所有當前訂閱者以及任何後續訂閱者。這不足為奇——`Replay` 的工作就是重播來源所做的事情——但是請注意，如果你使用[脫離 Rx 的世界](13_LeavingIObservable.md)一章中描述的任何機制，這可能會導致問題。例如，如果你 `await` 一個由 `Replay` 產生的可觀察序列，且底層來源報告了錯誤，你將不再遵循[例外狀況狀態](13_LeavingIObservable.md#exception-state)中描述的規則。如果你需要使用 `await`（或將對 `OnError` 的呼叫轉換為重新擲回的類似機制），你可能需要使用 [`ResetExceptionDispatchState`](13_LeavingIObservable.md#resetexceptiondispatchstate) 運算子來確保每次重新擲回例外狀況都會適當地重設例外狀況狀態。

## RefCount

我們在前面的部分中看到 `Multicast`（以及它的各種包裝函式）支援兩種使用模式：

* 返回 `IConnectableObservable<T>` 以允許對何時對底層來源進行訂閱進行頂層控制
* 返回普通的 `IObservable<T>`，使我們能夠在多處使用來源的查詢（例如 `s.Zip(s.Take(1))`）時避免對來源進行不必要的多個訂閱，但仍然為每個最上層的 `Subscribe` 對底層來源進行一次 `Subscribe` 呼叫

`RefCount` 提供了一種稍微不同的模式。它使對底層來源的訂閱能夠由普通的 `Subscribe` 觸發，但仍然可以僅對底層來源進行單次呼叫。這在整本書中使用的 AIS 範例中可能很有用。你可能想要將多個訂閱者附加到報告輪船和其他船隻廣播的位置訊息的可觀察來源，但你通常希望為此呈現基於 Rx 之 API 的函式庫僅連接到提供這些訊息的任何底層服務一次。並且你很可能希望它僅在至少有一個訂閱者在接聽時才進行連接。`RefCount` 對於此非常理想，因為它使單個來源能夠支援多個訂閱者，並使底層來源能夠知道我們何時在「無訂閱者」和「至少有一個訂閱者」狀態之間轉換。

為了能夠觀察 `RefCount` 的運作方式，我將使用來源的修改版本，該版本在發生訂閱時進行報告：

```csharp
IObservable<int> src = Observable.Create<int>(async obs =>
{
    Console.WriteLine("Create callback called");
    obs.OnNext(1);
    await Task.Delay(250).ConfigureAwait(false);
    obs.OnNext(2);
    await Task.Delay(250).ConfigureAwait(false);
    obs.OnNext(3);
    await Task.Delay(250).ConfigureAwait(false);
    obs.OnNext(4);
    await Task.Delay(100).ConfigureAwait(false);
    obs.OnCompleted();
});
```

與較早的範例不同，這在每個 `OnNext` 之間使用了 `async` 和延遲，以確保主執行緒有時間在產生所有項目之前設定多個訂閱。然後我們可以使用 `RefCount` 包裝它：

```csharp
IObservable<int> rc = src
    .Publish()
    .RefCount();
```

請注意，我必須先呼叫 `Publish`。這是因為 `RefCount` 需要 `IConnectableObservable<T>`。它希望僅在第一次訂閱時才啟動來源。一旦有至少一個訂閱者，它就會呼叫 `Connect`。讓我們試試看：

```csharp
rc.Subscribe(x => Console.WriteLine($"Sub1: {x} ({DateTime.Now})"));
rc.Subscribe(x => Console.WriteLine($"Sub2: {x} ({DateTime.Now})"));
Thread.Sleep(600);
Console.WriteLine();
Console.WriteLine("Adding more subscribers");
Console.WriteLine();
rc.Subscribe(x => Console.WriteLine($"Sub3: {x} ({DateTime.Now})"));
rc.Subscribe(x => Console.WriteLine($"Sub4: {x} ({DateTime.Now})"));
```

以下是輸出：

```
Create callback called
Sub1: 1 (10/08/2023 16:36:44)
Sub1: 2 (10/08/2023 16:36:45)
Sub2: 2 (10/08/2023 16:36:45)
Sub1: 3 (10/08/2023 16:36:45)
Sub2: 3 (10/08/2023 16:36:45)

Adding more subscribers

Sub1: 4 (10/08/2023 16:36:45)
Sub2: 4 (10/08/2023 16:36:45)
Sub3: 4 (10/08/2023 16:36:45)
Sub4: 4 (10/08/2023 16:36:45)
```

請注意，只有 `Sub1` 接收到了第一個事件。那是因為傳遞給 `Create` 的回呼函式立即產生了該事件。只有當它叫用其第一個 `await` 時，它才會返回給呼叫端，使我們能夠附加第二個訂閱者。該訂閱者已經錯過了第一個事件，但正如你所見，它收到了第二個和第三個事件。程式碼等待足夠長的時間以使前三個事件發生，然後附加另外兩個訂閱者，並且你可以看到所有四個訂閱者都收到了最終事件。

顧名思義，`RefCount` 會計算活動訂閱者的數量。如果此數字降為 0，它將在 `Connect` 返回的物件上呼叫 `Dispose`，從而關閉訂閱。如果附加了進一步的訂閱者，它將重新啟動。此範例展示了這一點：

```csharp
IDisposable s1 = rc.Subscribe(x => Console.WriteLine($"Sub1: {x} ({DateTime.Now})"));
IDisposable s2 = rc.Subscribe(x => Console.WriteLine($"Sub2: {x} ({DateTime.Now})"));
Thread.Sleep(600);

Console.WriteLine();
Console.WriteLine("Removing subscribers");
s1.Dispose();
s2.Dispose();
Thread.Sleep(600);
Console.WriteLine();

Console.WriteLine();
Console.WriteLine("Adding more subscribers");
Console.WriteLine();
rc.Subscribe(x => Console.WriteLine($"Sub3: {x} ({DateTime.Now})"));
rc.Subscribe(x => Console.WriteLine($"Sub4: {x} ({DateTime.Now})"));
```

我們得到此輸出：

```
Create callback called
Sub1: 1 (10/08/2023 16:40:39)
Sub1: 2 (10/08/2023 16:40:39)
Sub2: 2 (10/08/2023 16:40:39)
Sub1: 3 (10/08/2023 16:40:39)
Sub2: 3 (10/08/2023 16:40:39)

Removing subscribers


Adding more subscribers

Create callback called
Sub3: 1 (10/08/2023 16:40:40)
Sub3: 2 (10/08/2023 16:40:40)
Sub4: 2 (10/08/2023 16:40:40)
Sub3: 3 (10/08/2023 16:40:41)
Sub4: 3 (10/08/2023 16:40:41)
Sub3: 4 (10/08/2023 16:40:41)
Sub4: 4 (10/08/2023 16:40:41)
```

這一次，`Create` 回呼函式執行了兩次。那是因為活動訂閱者的數量降為 0，因此 `RefCount` 呼叫了 `Dispose` 來關閉事務。當新的訂閱者到來時，它再次呼叫 `Connect` 來重新啟動。有一些多載使你能夠指定 `disconnectDelay`。這告訴它在訂閱者數量降為零之後等待指定的時間再斷開連接，以檢視是否有任何新的訂閱者到來。但如果指定的時間過去了，它仍然會斷開連接。如果這不是你想要的，下一個運算子可能適合你。

## AutoConnect

`AutoConnect` 運算子的行為方式與 `RefCount` 大致相同，因為它在第一個訂閱者訂閱時在其底層 `IConnectableObservable<T>` 上呼叫 `Connect`。不同之處在於它不會嘗試偵測活動訂閱者的數量何時降為零：一旦連接，即使它沒有訂閱者，它也會無限期地保持連接狀態。

儘管 `AutoConnect` 很方便，但你需要稍微小心一點，因為它可能會導致流失：它永遠不會自動斷開連接。仍然可以拆除它建立的連接：`AutoConnect` 接受一個類型為 `Action<IDisposable>` 的選擇性引數。它在首次連接到來源時叫用此委派，並向你傳遞由來源的 `Connect` 方法返回的 `IDisposable`。你可以透過呼叫 `Dispose` 來將其關閉。

每當你的來源不太適合處理多個訂閱者時，本章中的運算子都會很有用。它提供了多種附加多個訂閱者的方法，同時僅對底層來源觸發單個 `Subscribe`。
