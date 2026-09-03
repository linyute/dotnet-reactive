# 排程與執行緒 (Scheduling and Threading)

Rx 主要是用於非同步處理_運動中的資料 (data in motion)_ 的系統。如果我們處理多個資訊來源，它們很可能會並行產生資料。我們可能需要在處理資料時具有某種程度的平行處理原則，以達到我們的延展性目標。我們需要對系統的這些方面進行控制。

到目前為止，我們成功避免了對執行緒或並行性的任何明確使用。我們已經看到一些必須處理時間以執行其工作的方法。（例如，`Buffer`、`Delay` 和 `Sample` 必須安排在特定的排程上進行工作。）然而，我們依賴於預設行為，儘管預設行為通常符合我們的需求，但我們有時需要行使更多控制權。本章將介紹 Rx 的排程系統，該系統提供了一種優雅的方式來管理這些問題。

## Rx、執行緒與並行性 (Rx, Threads and Concurrency)

Rx 不會對我們使用的執行緒施加限制。`IObservable<T>` 可以自由地在任何執行緒上叫用其訂閱者的 `OnNext/Completed/Error` 方法，甚至可能為每次呼叫使用不同的執行緒。儘管如此自由，Rx 的一個方面可以防止混亂：可觀察來源在任何情況下都必須遵守 [Rx 序列的基本規則](02_KeyTypes.md#the-fundamental-rules-of-rx-sequences)。

當我們第一次探索這些規則時，我們重點關注它們如何決定呼叫任何單一觀察者的順序。可以對 `OnNext` 進行任意次數的呼叫，但是一旦叫用了 `OnError` 或 `OnCompleted`，就不得再進行呼叫。但現在我們正在研究並行性，這些規則的另一個面向變得更加重要：對於任何單一訂閱，可觀察來源不得對該訂閱的觀察者進行並行呼叫。因此，如果來源呼叫 `OnNext`，它必須等待該呼叫返回，然後才能再次呼叫 `OnNext`，或呼叫 `OnError` 或 `OnComplete`。

對觀察者來說，結果是只要你的觀察者只參與一個訂閱，它就只需要一次處理一件事。訂閱的來源是否是一個涉及許多不同運算子的冗長且複雜的處理鏈並不重要。即使你透過組合多個輸入（例如使用 [`Merge`](09_CombiningSequences.md#merge)）來建構該來源，基本規則也要求如果你在單一 `IObservable<T>` 上僅呼叫了一次 `Subscribe`，則該來源永遠不允許對你的 `IObserver<T>` 方法進行多次並行呼叫。

因此，儘管每次呼叫可能來自不同的執行緒，但呼叫是嚴格循序的（除非單一觀察者參與了多個訂閱）。

接收傳入通知以及產生通知的 Rx 運算子將在傳入通知碰巧到達的任何執行緒上通知其觀察者。假設你有一個如下所示的運算子序列：

```csharp
source
    .Where(x => x.MessageType == 3)
    .Buffer(10)
    .Take(20)
    .Subscribe(x => Console.WriteLine(x));
```

當對 `Subscribe` 的呼叫發生時，我們最終會得到一個觀察者鏈。將叫用我們回呼函式的 Rx 提供的觀察者被傳遞給 `Take` 返回的可觀察序列，而 `Take` 將轉而建立一個訂閱 `Buffer` 返回的可觀察序列的觀察者，該觀察者又將建立一個訂閱 `Where` 可觀察序列的觀察者，該觀察者又建立了一個訂閱 `source` 的觀察者。

因此，當 `source` 決定產生一個項目時，它將叫用 `Where` 運算子觀察者的 `OnNext`。這將叫用述詞，如果 `MessageType` 確實為 3，`Where` 觀察者將在同一執行緒上呼叫 `Buffer` 觀察者的 `OnNext`。在 `Buffer` 觀察者的 `OnNext` 返回之前，`Where` 觀察者的 `OnNext` 不會返回。現在，如果 `Buffer` 觀察者確定它已完全填滿緩衝區（例如，它剛剛收到第 10 個項目），那麼它也不會返回——它將叫用 `Take` 觀察者的 `OnNext`，並且只要 `Take` 尚未收到 20 個緩衝區，它就會在將叫用我們回呼函式的 Rx 提供觀察者上呼叫 `OnNext`。

因此，對於一路傳遞到傳遞給 subscribe 的回呼函式中的 `Console.WriteLine` 的來源通知，我們最終會在堆疊上產生許多巢狀呼叫：

```
`source` 呼叫：
  `Where` 觀察者，其呼叫：
    `Buffer` 觀察者，其呼叫：
      `Take` 觀察者，其呼叫：
        `Subscribe` 觀察者，其呼叫我們的 Lambda
```

這一切都發生在一個執行緒上。大多數 Rx 運算子都沒有任何特定的執行緒作為其歸屬。它們只是在呼叫進來的任何執行緒上執行工作。這使得 Rx 非常有效率。將資料從一個運算子傳遞到下一個運算子僅涉及方法呼叫，而且這些呼叫非常快。（事實上，通常還有更多層。Rx 往往會新增一些包裝函式來處理錯誤和提早取消訂閱。因此呼叫堆疊看起來會比我剛才展示的稍微複雜一些。但通常仍然只是方法呼叫。）

你偶爾會聽到 Rx 被描述為具有_自由執行緒 (free threaded)_ 模型。這僅意味著運算子通常不在乎它們使用哪個執行緒。正如我們將看到的，存在例外情況，但是一個運算子直接呼叫下一個運算子是常態。

這樣做的結果是，通常是原始來源決定了使用哪個執行緒。下一個範例透過建立一個主體 (subject)，然後在各個執行緒上呼叫 `OnNext` 並回報執行緒 ID 來對此進行說明。

```csharp
Console.WriteLine($"Main thread: {Environment.CurrentManagedThreadId}");
var subject = new Subject<string>();

subject.Subscribe(
    m => Console.WriteLine($"Received {m} on thread: {Environment.CurrentManagedThreadId}"));

object sync = new();
ParameterizedThreadStart notify = arg =>
{
    string message = arg?.ToString() ?? "null";
    Console.WriteLine(
        $"OnNext({message}) on thread: {Environment.CurrentManagedThreadId}");
    lock (sync)
    {
        subject.OnNext(message);
    }
};

notify("Main");
new Thread(notify).Start("First worker thread");
new Thread(notify).Start("Second worker thread");
```

輸出：

```
Main thread: 1
OnNext(Main) on thread: 1
Received Main on thread: 1
OnNext(First worker thread) on thread: 10
Received First worker thread on thread: 10
OnNext(Second worker thread) on thread: 11
Received Second worker thread on thread: 11
```

在每種情況下，傳遞給 `Subscribe` 的處理常式都是在對 `subject.OnNext` 進行呼叫的同一執行緒上被回呼的。這既簡單又有效率。然而，事情並不總是這麼簡單。

## 定時呼叫 (Timed invocation)

某些通知不會是來源提供項目的直接結果。例如，Rx 提供了 [`Delay`](12_Timing.md#delay) 運算子，它對項目的傳遞進行時間平移。下一個範例基於前面的範例，主要區別在於我們不再直接訂閱來源。我們透過 `Delay` 進行訂閱：

```csharp
Console.WriteLine($"Main thread: {Environment.CurrentManagedThreadId}");
var subject = new Subject<string>();

subject
    .Delay(TimeSpan.FromSeconds(0.25))
    .Subscribe(
    m => Console.WriteLine($"Received {m} on thread: {Environment.CurrentManagedThreadId}"));

object sync = new();
ParameterizedThreadStart notify = arg =>
{
    string message = arg?.ToString() ?? "null";
    Console.WriteLine(
        $"OnNext({message}) on thread: {Environment.CurrentManagedThreadId}");
    lock (sync)
    {
        subject.OnNext(message);
    }
};

notify("Main 1");
Thread.Sleep(TimeSpan.FromSeconds(0.1));
notify("Main 2");
Thread.Sleep(TimeSpan.FromSeconds(0.3));
notify("Main 3");
new Thread(notify).Start("First worker thread");
Thread.Sleep(TimeSpan.FromSeconds(0.1));
new Thread(notify).Start("Second worker thread");

Thread.Sleep(TimeSpan.FromSeconds(2));
```

這也會在傳送來源項目之間等待一段時間，因此我們可以看到 `Delay` 的效果。這是輸出：

```
Main thread: 1
OnNext(Main 1) on thread: 1
OnNext(Main 2) on thread: 1
Received Main 1 on thread: 12
Received Main 2 on thread: 12
OnNext(Main 3) on thread: 1
OnNext(First worker thread) on thread: 13
OnNext(Second worker thread) on thread: 14
Received Main 3 on thread: 12
Received First worker thread on thread: 12
Received Second worker thread on thread: 12
```

請注意，在這種情況下，每個 `Received` 訊息都在執行緒 ID 12 上，這與引發通知的三個執行緒中的任何一個都不同。

這並不完全令人驚訝。Rx 在這裡使用原始執行緒的唯一方法是讓 `Delay` 在轉發呼叫之前將執行緒封鎖指定的時間（此處為四分之一秒）。這對於大多數情境都是不可接受的，因此 `Delay` 運算子會安排在適當的延遲後發生回呼函式。正如你可以從輸出中看到的，這些似乎都發生在一個特定的執行緒上。無論哪個執行緒呼叫 `OnNext`，延遲的通知都會到達執行緒 ID 12。但這不是由 `Delay` 運算子建立的執行緒。之所以會發生這種情況，是因為 `Delay` 正在使用_排程器 (scheduler)_。

## 排程器 (Schedulers)

排程器做三件事：

* 確定執行工作的內容（例如，某個執行緒）
* 決定何時執行工作（例如，立即或延後）
* 追蹤時間

這是一個探索前兩項的簡單範例：

```csharp
Console.WriteLine($"Main thread: {Environment.CurrentManagedThreadId}");

Observable
    .Range(1, 5)
    .Subscribe(m => 
      Console.WriteLine(
        $"Received {m} on thread: {Environment.CurrentManagedThreadId}"));

Console.WriteLine("Subscribe returned");
Console.ReadLine();
```

這看起來可能與排程沒有任何關係，但實際上，`Range` 始終使用排程器來執行其工作。我們只是讓它使用其預設排程器。這是輸出：

```
Main thread: 1
Received 1 on thread: 1
Received 2 on thread: 1
Received 3 on thread: 1
Received 4 on thread: 1
Received 5 on thread: 1
Subscribe returned
```

查看排程器所做工作清單中的前兩項，我們可以看到執行工作的內容是我呼叫 `Subscribe` 的執行緒。至於它決定何時執行工作，它決定在 `Subscribe` 返回之前完成所有工作。因此，你可能會認為 `Range` 會立即產生我們要求的所有項目然後返回。然而，事情並沒有那麼簡單。讓我們看看如果同時執行多個 `Range` 執行個體會發生什麼。這引入了一個額外的運算子：一個再次呼叫 `Range` 的 `SelectMany`：

```csharp
Observable
    .Range(1, 5)
    .SelectMany(i => Observable.Range(i * 10, 5))
    .Subscribe(m => 
      Console.WriteLine(
        $"Received {m} on thread: {Environment.CurrentManagedThreadId}"));
```

輸出顯示 `Range` 實際上不一定立即產生其所有項目：

```
Received 10 on thread: 1
Received 11 on thread: 1
Received 20 on thread: 1
Received 12 on thread: 1
Received 21 on thread: 1
Received 30 on thread: 1
Received 13 on thread: 1
Received 22 on thread: 1
Received 31 on thread: 1
Received 40 on thread: 1
Received 14 on thread: 1
Received 23 on thread: 1
Received 32 on thread: 1
Received 41 on thread: 1
Received 50 on thread: 1
Received 24 on thread: 1
Received 33 on thread: 1
Received 42 on thread: 1
Received 51 on thread: 1
Received 34 on thread: 1
Received 43 on thread: 1
Received 52 on thread: 1
Received 44 on thread: 1
Received 53 on thread: 1
Received 54 on thread: 1
Subscribe returned
```

由 `SelectMany` 回呼函式產生的第一個巢狀 `Range` 產生了幾個值（10 和 11），但隨後第二個設法在第一個產生第三個值 (12) 之前輸出其第一個值 (20)。你可以看到這裡有一些進度的交錯。因此，儘管執行工作的內容仍然是我們叫用 `Subscribe` 的執行緒，但排程器必須做出的第二個選擇——何時執行工作——比最初看起來更微妙。這告訴我們 `Range` 並不像這個單純的實作那麼簡單：

```csharp
public static IObservable<int> NaiveRange(int start, int count)
{
    return System.Reactive.Linq.Observable.Create<int>(obs =>
    {
        for (int i = 0; i < count; i++)
        {
            obs.OnNext(start + i);
        }

        return Disposable.Empty;
    });
}
```

如果 `Range` 這樣運作，這段程式碼將在轉移到下一個範圍之前產生由 `SelectMany` 回呼函式返回的第一個範圍中的所有項目。事實上，如果這是我們想要的，Rx 確實提供了一個可以為我們提供該行為的排程器。此範例將 `ImmediateScheduler.Instance` 傳遞給巢狀的 `Observable.Range` 呼叫：

```csharp
Observable
    .Range(1, 5)
    .SelectMany(i => Observable.Range(i * 10, 5, ImmediateScheduler.Instance))
    .Subscribe(
    m => Console.WriteLine($"Received {m} on thread: {Environment.CurrentManagedThreadId}"));
```

這是結果：

```
Received 10 on thread: 1
Received 11 on thread: 1
Received 12 on thread: 1
Received 13 on thread: 1
Received 14 on thread: 1
Received 20 on thread: 1
Received 21 on thread: 1
Received 22 on thread: 1
Received 23 on thread: 1
Received 24 on thread: 1
Received 30 on thread: 1
Received 31 on thread: 1
Received 32 on thread: 1
Received 33 on thread: 1
Received 34 on thread: 1
Received 40 on thread: 1
Received 41 on thread: 1
Received 42 on thread: 1
Received 43 on thread: 1
Received 44 on thread: 1
Received 50 on thread: 1
Received 51 on thread: 1
Received 52 on thread: 1
Received 53 on thread: 1
Received 54 on thread: 1
Subscribe returned
```

透過在對 `Observable.Range` 的最內層呼叫中指定 `ImmediateScheduler.Instance`，我們要求了一種特定原則：這會在呼叫者的執行緒上叫用所有工作，並且始終立即執行。這不是 `Range` 預設原則有幾個原因。（它的預設值是 `Scheduler.CurrentThread`，它總是返回 `CurrentThreadScheduler` 的執行個體。）首先，`ImmediateScheduler.Instance` 最終可能會導致相當深的呼叫堆疊。大多數其他排程器都維護工作佇列，因此如果一個運算子在另一個運算子正在執行某項操作的中途決定它有新工作要做（例如，巢狀 `Range` 運算子決定開始發出其值），而不是立即開始該工作（這將涉及叫用將執行該工作的方法），該工作可以放入佇列中，使已經在進行的工作能夠在開始下一項工作之前完成。當查詢變得複雜時，在所有地方使用 immediate 排程器可能會導致堆疊溢位。`Range` 預設不使用 immediate 排程器的第二個原因是，當多個可觀察序列同時處於作用中狀態時，它們都可以取得一些進展——`Range` 盡快產生其所有項目，因此如果不使用使運算子能夠輪流進行的排程器，它最終可能會使其他運算子缺乏 CPU 時間。

請注意，`Subscribe returned` 訊息在兩個範例中都是最後出現的。因此，儘管 `CurrentThreadScheduler` 不如 immediate 排程器那麼急切，但在完成所有未完成的工作之前它仍然不會返回給其呼叫者。它維護一個工作佇列，實現了稍微更多的公平性，並避免了堆疊溢位，但是一旦有任何東西要求 `CurrentThreadScheduler` 執行某些操作，它在清空佇列之前不會返回。

並非所有排程器都具有此特性。這是先前範例的一個變化形式，其中我們只有對 `Range` 的單一呼叫，沒有任何巢狀可觀察序列。這次我要求它使用 `TaskPoolScheduler`。

```csharp
Observable
    .Range(1, 5, TaskPoolScheduler.Default)
    .Subscribe(
    m => Console.WriteLine($"Received {m} on thread: {Environment.CurrentManagedThreadId}"));
```

與 immediate 和 current thread 排程器相比，這對於執行工作的內容做出了不同的決定，正如我們從其輸出中看到的那樣：

```
Main thread: 1
Subscribe returned
Received 1 on thread: 12
Received 2 on thread: 12
Received 3 on thread: 12
Received 4 on thread: 12
Received 5 on thread: 12
```

請注意，通知都發生在與我們叫用 `Subscribe` 的執行緒（ID 1）不同的執行緒（ID 12）上。這是因為 `TaskPoolScheduler` 的定義特徵是它透過工作平行程式庫 (Task Parallel Library, TPL) 的工作集區叫用所有工作。這就是為什麼我們看到不同的執行緒 ID：工作集區不擁有我們應用程式的主執行緒。在這種情況下，它認為沒有必要啟動多個執行緒。這是合理的，這裡只有一個來源一次提供一個項目。在這種情況下我們沒有得到更多執行緒是一件好事——當單一執行緒循序處理工作項目時，執行緒集區的效率最高，因為它避免了環境定義切換額外負荷，並且由於這裡沒有並行工作的實際空間，如果在這種情況下它建立了多個執行緒，我們將一無所獲。

此排程器還有一個非常顯著的差異：請注意，對 `Subscribe` 的呼叫在_任何_通知到達我們的觀察者之前就已返回。那是因為這是我們看過的第一個將引入真正的平行處理原則的排程器。`ImmediateScheduler` 和 `CurrentThreadScheduler` 本身永遠不會啟動新執行緒，無論執行的運算子多麼希望執行並行操作。儘管 `TaskPoolScheduler` 確定不需要建立多個執行緒，但它確實建立的一個執行緒是與應用程式主執行緒不同的執行緒，這意味著主執行緒可以與此訂閱平行繼續執行。由於 `TaskPoolScheduler` 不會在啟動工作的執行緒上執行任何工作，因此它可以在對工作進行排入佇列後立即返回，從而使 `Subscribe` 方法能夠立即返回。

如果我們在帶有巢狀可觀察序列的範例中使用 `TaskPoolScheduler` 會怎麼樣？這僅在對 `Range` 的內部呼叫中使用它，因此外部呼叫仍將使用預設的 `CurrentThreadScheduler`：

```csharp
Observable
    .Range(1, 5)
    .SelectMany(i => Observable.Range(i * 10, 5, TaskPoolScheduler.Default))
    .Subscribe(
    m => Console.WriteLine($"Received {m} on thread: {Environment.CurrentManagedThreadId}"));
```

現在我們可以看到更多執行緒參與進來：

```
Received 10 on thread: 13
Received 11 on thread: 13
Received 12 on thread: 13
Received 13 on thread: 13
Received 40 on thread: 16
Received 41 on thread: 16
Received 42 on thread: 16
Received 43 on thread: 16
Received 44 on thread: 16
Received 50 on thread: 17
Received 51 on thread: 17
Received 52 on thread: 17
Received 53 on thread: 17
Received 54 on thread: 17
Subscribe returned
Received 14 on thread: 13
Received 20 on thread: 14
Received 21 on thread: 14
Received 22 on thread: 14
Received 23 on thread: 14
Received 24 on thread: 14
Received 30 on thread: 15
Received 31 on thread: 15
Received 32 on thread: 15
Received 33 on thread: 15
Received 34 on thread: 15
```

由於在此範例中我們只有一個觀察者，因此 Rx 規則要求一次給予它一個項目，所以在實務上這裡並沒有真正的平行處理原則空間，但是與前面的範例相比，更複雜的結構最初會導致更多的工作項目進入排程器的佇列，這可能就是這次工作被多個執行緒選取的原因。在實務上，大多數這些執行緒大部分時間都會封鎖在 `SelectMany` 內部的程式碼中，該程式碼確保將項目一次一個傳遞給其目標觀察者。項目沒有更加混亂可能有點令人驚訝。子範圍本身似乎是以隨機順序出現的，但它在每個子範圍內幾乎是循序產生項目的（項目 14 是其中的一個例外）。這是與 `Range` 與 `TaskPoolScheduler` 互動方式相關的古怪之處。

我還沒有談到排程器的第三項工作：追蹤時間。這不會在 `Range` 中出現，因為它試圖盡快產生其所有項目。但對於我在[定時呼叫](#timed-invocation)一節中展示的 `Delay` 運算子，時間顯然是一個關鍵元素。事實上，這將是展示排程器提供的 API 的好時機：

```csharp
public interface IScheduler
{
    DateTimeOffset Now { get; }
    
    IDisposable Schedule<TState>(TState state, 
                                 Func<IScheduler, TState, IDisposable> action);
    
    IDisposable Schedule<TState>(TState state, 
                                 TimeSpan dueTime, 
                                 Func<IScheduler, TState, IDisposable> action);
    
    IDisposable Schedule<TState>(TState state, 
                                 DateTimeOffset dueTime, 
                                 Func<IScheduler, TState, IDisposable> action);
}
```

你可以看到除了其中一個之外，所有這些都與時間有關。只有第一個 `Schedule` 多載不是，當運算子希望在排程器允許的情況下盡快排程工作執行時，它們會呼叫該多載。那是 `Range` 使用的多載。（嚴格來說，`Range` 會詢問排程器以找出它是否支援長時間執行的操作，其中運算子可以在較長時間內暫時控制執行緒。它更喜歡在可能的情況下使用該操作，因為它往往比為其希望產生的每個單一項目向排程器提交工作更有效率。`TaskPoolScheduler` 確實支援長時間執行的操作，這解釋了我們之前看到的稍微令人驚訝的輸出，但是 `CurrentThreadScheduler`，即 `Range` 的預設選擇，不支援。因此預設情況下，`Range` 將為其希望產生的每個項目叫用該第一個 `Schedule` 多載一次。）

`Delay` 使用第二個多載。確切的實作相當複雜（主要是因為當繁忙的來源導致其落後時它如何有效率地追趕），但本質上，每次新項目到達 `Delay` 運算子時，它都會排程一個工作項目在設定的延遲後執行，以便它可以將該項目以預期的時間平移提供給其訂閱者。

排程器必須負責管理時間，因為 .NET 有幾種不同的計時器機制，並且計時器的選擇通常取決於你想要處理計時器回呼函式的環境定義。由於排程器確定了工作執行的內容，這意味著它們也必須選擇計時器類型。例如，UI 架構通常提供在其回呼函式適合對使用者介面進行更新的環境定義中叫用其回呼函式的計時器。Rx 提供了一些使用這些計時器的特定於 UI 架構的排程器，但這些對於其他情境來說是不合適的選擇。因此，每個排程器都使用適合將在其內部執行工作項目的環境定義的計時器。

這樣做有一個有用的結果：因為 `IScheduler` 為與時間相關的細節提供了抽象，所以可以將時間虛擬化。這對於測試非常有用。如果你查看 [Rx 存放庫](https://github.com/dotnet/reactive)中的大量測試套件，你會發現有許多驗證與時間相關行為的測試。如果這些是即時執行的，測試套件將花費太長時間執行，並且還可能產生奇怪的偶發性失敗，因為在與測試相同的電腦上執行的背景工作偶爾會以可能使測試混淆的方式變更執行速度。相反，這些測試使用專門的排程器，提供對時間流逝的完全控制。（如需詳細資訊，請參閱稍後的[測試排程器一節](#test-schedulers)，並且即將推出一整個[測試章節](16_TestingRx.md)。）

請注意，所有三個 `IScheduler.Schedule` 方法都需要回呼函式。排程器將在其選擇的時間和環境定義中叫用此回呼函式。排程器回呼函式採用另一個 `IScheduler` 作為其第一個引數。這用於需要重複叫用的情境，正如我們稍後將看到的。

Rx 提供了幾種排程器。以下各節描述了最廣泛使用的排程器。

### ImmediateScheduler

`ImmediateScheduler` 是 Rx 提供的最簡單的排程器。正如你在前面幾節中所看到的，每當要求它排程某些工作時，它都會立即執行該工作。它在其 `IScheduler.Schedule` 方法內部執行此操作。

這是一個非常簡單的策略，它使 `ImmediateScheduler` 非常有效率。出於這個原因，許多運算子預設使用 `ImmediateScheduler`。但是，對於立即產生多個項目的運算子（特別是當項目數量可能很大時），它可能會有問題。例如，Rx 定義了 [`IEnumerable<T>` 的 `ToObservable` 擴充方法](03_CreatingObservableSequences.md#from-ienumerablet)。當你訂閱由此返回的 `IObservable<T>` 時，它將立即開始逐一查看該集合，如果你告訴它使用 `ImmediateScheduler`，則 `Subscribe` 在到達集合末尾之前不會返回。這對於無限序列顯然是一個問題，這就是為什麼這種類型的運算子預設不使用 `ImmediateScheduler` 的原因。

當你叫用接受 `TimeSpan` 的 `Schedule` 多載時，`ImmediateScheduler` 也具有潛在令人驚訝的行為。這要求排程器在指定的時間長度後執行某些工作。它實現此目的的方法是呼叫 `Thread.Sleep`。對於大多數 Rx 的排程器，此多載將安排某種計時器機制在以後執行程式碼，使當前執行緒能夠繼續進行其業務，但 `ImmediateScheduler` 在這裡忠於其名稱，因為它拒絕參與此類延後執行。它只是封鎖當前執行緒，直到該執行工作為止。這意味著如果你指定了此排程器，則像 `Interval` 返回的那些基於時間的可觀察序列將起作用，但代價是阻止執行緒執行任何其他操作。

接受 `DateTime` 的 `Schedule` 多載略有不同。如果你指定未來少於 10 秒的時間，它將像使用 `TimeSpan` 時一樣封鎖呼叫端執行緒。但是如果你傳遞一個距離未來更遠的 `DateTime`，它會放棄立即執行，並改為使用計時器。

### CurrentThreadScheduler

`CurrentThreadScheduler` 與 `ImmediateScheduler` 非常相似。不同之處在於，當當前執行緒上已在處理現有工作項目時，它如何處理排程工作的請求。如果你將多個使用排程器執行其工作的運算子鏈結在一起，就可能會發生這種情況。

為了理解會發生什麼，了解快速連續產生多個項目的來源（例如 [`IEnumerable<T>` 的 `ToObservable` 擴充方法](03_CreatingObservableSequences.md#from-ienumerablet) 或 [`Observable.Range`](03_CreatingObservableSequences.md#observablerange)）如何使用排程器會很有幫助。這些類型的運算子不使用正常的 `for` 或 `foreach` 迴圈。它們通常為每次反覆運算排程一個新的工作項目（除非排程器碰巧為長時間執行的工作做出了特殊規定）。`ImmediateScheduler` 會立即執行此類工作，而 `CurrentThreadScheduler` 會檢查它是否已在處理工作項目。我們在前面的範例中看到了這一點：

```csharp
Observable
    .Range(1, 5)
    .SelectMany(i => Observable.Range(i * 10, 5))
    .Subscribe(
        m => Console.WriteLine($"Received {m} on thread: {Environment.CurrentManagedThreadId}"));
```

讓我們準確追蹤這裡發生的情況。首先，假設此程式碼只是正常執行而不是在任何不尋常的環境定義中——也許在程式的 `Main` 進入點內。當此程式碼在由 `SelectMany` 返回的 `IObservable<int>` 上呼叫 `Subscribe` 時，將轉而呼叫第一個 `Observable.Range` 返回的 `IObservable<int>` 上的 `Subscribe`，這將反過來為產生範圍中的第一個值 (`1`) 排程一個工作項目。

由於我們沒有向 `Range` 明確傳遞排程器，它將使用其預設選擇 `CurrentThreadScheduler`，並且它會問自己「我是否已在此執行緒上處理某些工作項目的中途？」在這種情況下，答案將是「否」，因此它將立即執行工作項目（在從 `Range` 運算子進行的 `Schedule` 呼叫返回之前）。然後 `Range` 運算子將產生其第一個值，呼叫 `SelectMany` 運算子在訂閱範圍時提供的 `IObserver<int>` 上的 `OnNext`。

`SelectMany` 運算子的 `OnNext` 方法現在將叫用其 Lambda，傳入提供的引數（來自 `Range` 運算子的值 `1`）。你可以從上面的範例中看到此 Lambda 再次呼叫 `Observable.Range`，返回一個新的 `IObservable<int>`。`SelectMany` 將立即訂閱此序列（在其 `OnNext` 返回之前）。這是此程式碼第二次最終呼叫由 `Range` 返回的 `IObservable<int>` 上的 `Subscribe`（但它是與上次不同的執行個體），並且 `Range` 將再次預設使用 `CurrentThreadScheduler`，並將再次排程工作項目以執行第一次反覆運算。

因此，`CurrentThreadScheduler` 將再次問自己「我是否已在此執行緒上處理某些工作項目的中途？」但這一次，答案將是「是」。這就是與 `ImmediateScheduler` 不同的行為所在。`CurrentThreadScheduler` 為其使用的每個執行緒維護一個工作佇列，在這種情況下，它只是將新排程的工作新增到佇列中，並返回到 `SelectMany` 運算子的 `OnNext`。

`SelectMany` 現在已完成對來自第一個 `Range` 的此項目（值 `1`）的處理，因此其 `OnNext` 返回。此時，外部 `Range` 運算子排程另一個工作項目。同樣，`CurrentThreadScheduler` 將偵測到它當前正在執行工作項目，因此它只是將其新增至佇列。

在排程了將產生其第二個值 (`2`) 的工作項目之後，`Range` 運算子返回。請記住，此時正在執行的 `Range` 運算子中的程式碼是第一個排程工作項目的回呼函式，因此它返回到 `CurrentThreadScheduler`——我們回到了其 `Schedule` 方法內部（該方法由範圍運算子的 `Subscribe` 方法叫用）。

此時，`CurrentThreadScheduler` 不會從 `Schedule` 返回，因為它檢查其工作佇列，並會看到現在佇列中有兩個項目。（有巢狀 `Range` 可觀察序列排程以產生其第一個值的工作項目，並且還有最上層 `Range` 可觀察序列剛剛排程以產生其第二個值的工作項目。）`CurrentThreadScheduler` 現在將執行其中的第一個：巢狀 `Range` 運算子現在可以產生其第一個值（將為 `10`），因此它呼叫 `SelectMany` 提供的觀察者上的 `OnNext`，然後將呼叫其觀察者，該觀察者是由於範例中最上層呼叫 `Subscribe` 而提供的。該觀察者只會呼叫我們傳遞給 `Subscribe` 的 Lambda，導致我們的 `Console.WriteLine` 執行。在返回之後，巢狀 `Range` 運算子將排程另一個工作項目以產生其第二個項目。同樣，`CurrentThreadScheduler` 將意識到它已在此執行緒上處理工作項目的中途，因此它只是將其放入佇列中，然後立即從 `Schedule` 返回。巢狀 `Range` 運算子現在已完成本次反覆運算，因此它返回到排程器。排程器現在將選取佇列中的下一個項目，在這種情況下是頂層 `Range` 新增以產生第二個項目的工作項目。

依此類推。當工作已在進行中時對工作項目進行排入佇列，這就是允許多個可觀察來源平行取得進展的原因。

相比之下，`ImmediateScheduler` 會立即執行新的工作項目，這就是為什麼我們看不到這種平行進展。

（嚴格來說，在某些情境下 `ImmediateScheduler` 無法立即執行工作。在這些反覆運算情境中，它實際上提供了一個稍微不同的排程器，運算子使用該排程器在第一個項目之後排程所有工作，並且這會檢查是否要求它同時處理多個工作項目。如果是，它會退回到類似於 `CurrentThreadScheduler` 的佇列策略，除了它是初始工作項目本機的佇列，而不是個別執行緒佇列。這可以防止多執行緒引起的問題，並且它還可以避免在反覆運算運算子在當前工作項目的處理常式內排程新工作項目時發生的堆疊溢位。由於佇列在執行緒中的所有工作之間不共享，這仍然具有確保工作項目排入佇列的任何巢狀工作在對 `Schedule` 的呼叫返回之前完成的效果。因此，即使觸發了此排入佇列機制，我們通常也不會看到像使用 `CurrentThreadScheduler` 那樣來自不同來源的工作交錯。例如，如果我們告訴巢狀 `Range` 使用 `ImmediateScheduler`，則當 `Range` 開始反覆運算時將觸發此排入佇列行為，但因為佇列是該巢狀 `Range` 執行的初始工作項目的本機佇列，所以它最終將在返回之前產生所有巢狀 `Range` 項目。）

### DefaultScheduler

`DefaultScheduler` 適用於可能需要隨時間分散進行的工作，或者你可能需要並行執行的工作。這些功能意味著這不能保證在任何特定執行緒上執行工作，並且在實務上它透過 CLR 的執行緒集區排程工作。這是 Rx 所有基於時間的運算子的預設排程器，也是可以將 .NET 方法包裝為 `IObservable<T>` 的 `Observable.ToAsync` 運算子的預設排程器。

儘管如果你希望工作不在當前執行緒上發生（也許你正在編寫具有使用者介面的應用程式，並且你希望避免在負責更新 UI 和回應使用者輸入的執行緒上執行過多工作），此排程器很有用，但它最終可能在任何執行緒上執行工作的事實可能會使事情變得複雜。如果你希望所有工作都發生在一個執行緒上，只是不是你現在所在的執行緒呢？還有另一個排程器可用於此目的。

### EventLoopScheduler

`EventLoopScheduler` 提供一次一個的排程，將新排程的工作項目排入佇列。這類似於如果你僅從一個執行緒使用 `CurrentThreadScheduler` 時的操作方式。不同之處在於 `EventLoopScheduler` 為此工作建立了一個專用執行緒，而不是使用你碰巧排程該工作的任何執行緒。

與我們到目前為止檢驗的排程器不同，沒有用於取得 `EventLoopScheduler` 的靜態屬性。那是因為每個排程器都有自己的執行緒，所以你需要明確建立一個。它提供了兩個建構函式：

```csharp
public EventLoopScheduler()
public EventLoopScheduler(Func<ThreadStart, Thread> threadFactory)
```

第一個為你建立一個執行緒。第二個讓你控制執行緒建立程序。它會叫用你提供的回呼函式，並會將其自己的回呼函式傳遞給此函式，要求你在新建立的執行緒上執行該回呼函式。

`EventLoopScheduler` 實作了 `IDisposable`，呼叫 Dispose 將允許執行緒終止。這可以與 `Observable.Using` 方法很好地搭配使用。以下範例展示了如何使用 `EventLoopScheduler` 在專用執行緒上反覆查看 `IEnumerable<T>` 的所有內容，確保在完成後執行緒結束：

```csharp
IEnumerable<int> xs = GetNumbers();
Observable
    .Using(
        () => new EventLoopScheduler(),
        scheduler => xs.ToObservable(scheduler))
    .Subscribe(...);
```

### NewThreadScheduler

`NewThreadScheduler` 建立一個新執行緒來執行給予它的每個工作項目。這在大多數情境下不太可能有意義。然而，在你想執行某些長時間執行的工作，並透過 `IObservable<T>` 表示其完成的情況下，它可能會很有用。`Observable.ToAsync` 確實做到了這一點，並且通常會使用 `DefaultScheduler`，這意味著它將在執行緒集區執行緒上執行工作。但是如果工作可能需要超過一兩秒鐘，執行緒集區可能不是一個好的選擇，因為它是針對較短的執行時間進行最佳化的，並且其管理執行緒集區大小的啟發式演算法並非針對長時間執行的操作而設計。在這種情況下，`NewThreadScheduler` 可能是更好的選擇。

儘管對 `Schedule` 的每次呼叫都會建立一個新執行緒，但 `NewThreadScheduler` 會將不同的排程器傳遞到工作項目回呼函式中，這意味著任何嘗試執行反覆運算工作的作業都不會為每次反覆運算建立新執行緒。例如，如果你將 `NewThreadScheduler` 與 `Observable.Range` 一起使用，每次訂閱產生的 `IObservable<int>` 時都會獲得一個新執行緒，但不會為每個項目獲得一個新執行緒，即使 `Range` 確實為其產生的每個值排程了一個新的工作項目。它透過提供給工作項目回呼函式的巢狀排程器來排程這些逐值工作項目，並且 `NewThreadScheduler` 在這些情況下提供的巢狀排程器會在同一執行緒上叫用所有此類巢狀工作項目。

### SynchronizationContextScheduler

這透過 [`SynchronizationContext`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.synchronizationcontext) 叫用所有工作。這在使用者介面情境中很有用。大多數 .NET 用戶端使用者介面架構都提供了一個 `SynchronizationContext`，可用於在適合對 UI 進行變更的環境定義中叫用回呼函式。（通常這涉及在正確的執行緒上叫用它們，但是個別實作可以決定什麼構成適當的環境定義。）

### TaskPoolScheduler

使用 [TPL 工作](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl)透過執行緒集區叫用所有工作。TPL 是在 CLR 執行緒集區之後許多年引入的，現在是透過執行緒集區啟動工作的建議方式。在新增 TPL 時，當你透過工作排程工作時，執行緒集區將使用稍微不同的演算法，而不是依賴較舊的執行緒集區 API。這個較新的演算法使其在某些情境下更有效率。現在的說明文件對此相當含糊，因此尚不清楚現代 .NET 上是否仍然存在這些差異，但工作仍然是使用執行緒集區的推薦機制。出於回溯相容性原因，Rx 的 DefaultScheduler 使用較舊的 CLR 執行緒集區 API。在效能關鍵的程式碼中，在執行緒集區執行緒上執行大量工作的情況下，你可以嘗試改用 `TaskPoolScheduler`，看看它是否能為你的工作負載提供任何效能優勢。

### ThreadPoolScheduler

使用舊的 pre-[TPL](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl) API 透過執行緒集區叫用所有工作。這種類型是一個歷史產物，可以追溯到並非所有平台都提供相同類型執行緒集區的時代。在幾乎所有情況下，如果你想要設計此類型的行為，你應該使用 `DefaultScheduler`（儘管 [`TaskPoolScheduler`](#taskpoolscheduler) 提供了可能適用的不同行為）。使用 `ThreadPoolScheduler` 唯一會產生差異的情境是編寫 UWP 應用程式時。`System.Reactive` v6.0 的 UWP 目標提供了此類別的不同實作，而不是所有其他目標所獲得的實作。它使用 `Windows.System.Threading.ThreadPool`，而所有其他目標都使用 `System.Threading.ThreadPool`。UWP 版本提供了允許你設定 UWP 執行緒集區特定功能的屬性。

在實務上，最好在新程式碼中避免使用此類別。UWP 目標具有不同實作的唯一原因是 UWP 過去不提供 `System.Threading.ThreadPool`。但是當 UWP 在 Windows 版本 10.0.19041 中新增對 .NET Standard 2.0 的支援時，情況發生了變化。不再有任何充分的理由存在特定於 UWP 的 `ThreadPoolScheduler`，並且這種類型在 UWP 目標中截然不同是一個令人困惑的根源，但出於回溯相容性目的它必須保留。（它很可能會被棄用，因為 Rx 7 將解決由於 `System.Reactive` 元件目前直接依賴 UI 架構而產生的一些問題。）如果你使用 `DefaultScheduler`，無論你在哪個平台上執行，都將使用 `System.Threading.ThreadPool`。

### UI 架構排程器：ControlScheduler、DispatcherScheduler 和 CoreDispatcherScheduler

儘管 `SynchronizationContextScheduler` 適用於 .NET 中所有廣泛使用的用戶端 UI 架構，但 Rx 提供了更專門的排程器。`ControlScheduler` 適用於 Windows Forms 應用程式，`DispatcherScheduler` 適用於 WPF，`CoreDispatcherScheduler` 適用於 UWP。

這些更專業的類型提供了兩個好處。首先，你不一定非要在目標 UI 執行緒上取得這些排程器的執行個體。而使用 `SynchronizationContextScheduler`，你通常取得這所需的 `SynchronizationContext` 的唯一方法是在 UI 執行緒上執行時擷取 `SynchronizationContext.Current`。但是這些其他特定於 UI 架構的排程器可以傳遞適當的 `Control`、`Dispatcher` 或 `CoreDispatcher`，這些可以從非 UI 執行緒取得。其次，`DispatcherScheduler` 和 `CoreDispatcherScheduler` 提供了一種使用 `Dispatcher` 和 `CoreDispatcher` 類型支援的優先權機制的方法。

### 測試排程器 (Test Schedulers)

Rx 函式庫定義了幾種虛擬化時間的排程器，包括 `HistoricalScheduler`、`TestScheduler`、`VirtualTimeScheduler` 和 `VirtualTimeSchedulerBase`。我們將在[測試章節](16_TestingRx.md)中探討這種類型的排程器。

## SubscribeOn 與 ObserveOn

到目前為止，我已經討論了為什麼某些 Rx 來源需要存取排程器。這對於與時間相關的行為是必要的，對於盡快產生項目的來源也是必要的。但請記住，排程器控制三件事：

- 確定執行工作的內容（例如，某個執行緒）
- 決定何時執行工作（例如，立即或延後）
- 追蹤時間

到目前為止的討論主要集中在第 2 和第 3 個功能上。當談到我們自己的應用程式程式碼時，我們最有可能使用排程器來控制第一個面向。Rx 為 `IObservable<T>` 定義了兩個擴充方法：`SubscribeOn` 和 `ObserveOn`。這兩種方法都採用 `IScheduler` 並返回 `IObservable<T>`，因此你可以在這些運算子的下游鏈結更多運算子。

這些方法的作用正如其名稱所示。如果你使用 `SubscribeOn`，那麼當你在產生的 `IObservable<T>` 上呼叫 `Subscribe` 時，它會安排透過指定的排程器呼叫原始 `IObservable<T>` 的 `Subscribe` 方法。這是一個範例：

```csharp
Console.WriteLine($"[T:{Environment.CurrentManagedThreadId}] Main thread");

Observable
    .Interval(TimeSpan.FromSeconds(1))
    .SubscribeOn(new EventLoopScheduler((start) =>
    {
        Thread t = new(start) { IsBackground = false };
        Console.WriteLine($"[T:{t.ManagedThreadId}] Created thread for EventLoopScheduler");
        return t;
    }))
    .Subscribe(tick => 
          Console.WriteLine(
            $"[T:{Environment.CurrentManagedThreadId}] {DateTime.Now}: Tick {tick}"));

Console.WriteLine($"[T:{Environment.CurrentManagedThreadId}] {DateTime.Now}: Main thread exiting");
```

這呼叫了 `Observable.Interval`（預設使用 `DefaultScheduler`），但不是直接訂閱此序列，而是首先採用 `Interval` 返回的 `IObservable<T>` 並叫用 `SubscribeOn`。我使用了 `EventLoopScheduler`，並且向其傳遞了將用於確保它是非背景執行緒的執行緒工廠回呼函式。（預設情況下，`EventLoopScheduler` 為自己建立一個背景執行緒，這意味著該執行緒不會強制處理序保持存活。通常這是你想要的，但我在本範例中變更了這一點以顯示正在發生的情況。）

當我在 `SubscribeOn` 返回的 `IObservable<long>` 上呼叫 `Subscribe` 時，它會在我提供的 `EventLoopScheduler` 上呼叫 `Schedule`，並且在該工作項目的回呼函式中，它隨後會在原始 `Interval` 來源上呼叫 `Subscribe`。因此其效果是對底層來源的訂閱不是發生在我的主執行緒上，而是發生在為我的 `EventLoopScheduler` 建立的執行緒上。執行該程式會產生以下輸出：

```
[T:1] Main thread
[T:12] Created thread for EventLoopScheduler
[T:1] 21/07/2023 14:57:21: Main thread exiting
[T:6] 21/07/2023 14:57:22: Tick 0
[T:6] 21/07/2023 14:57:23: Tick 1
[T:6] 21/07/2023 14:57:24: Tick 2
...
```

請注意，我的應用程式主執行緒在來源開始產生通知之前結束。但也要注意新建立的執行緒的執行緒 ID 是 12，而我的通知卻是在 ID 為 6 的不同執行緒上傳入的！發生了什麼事？

這經常讓人困惑。你訂閱可觀察來源的排程器不一定會對來源在啟動並執行後的行為方式產生任何影響。還記得我之前說過 `Observable.Interval` 預設使用 `DefaultScheduler` 嗎？嗯，我們沒有在此處為 `Interval` 指定排程器，因此它將使用該預設值。它不在乎我們從什麼環境定義叫用其 `Subscribe` 方法。因此實際上，此處引入 `EventLoopScheduler` 的唯一效果是即使在其主執行緒結束後也保持處理序存活。該排程器執行緒在對 `Observable.Interval` 返回的 `IObservable<long>` 進行初始 `Subscribe` 呼叫後實際上永遠不會再次使用。它只是耐心地等待永遠不會到來的對 `Schedule` 的進一步呼叫。

不過，並非所有來源都完全不受叫用其 `Subscribe` 之環境定義的影響。如果我要替換這一行：

```csharp
    .Interval(TimeSpan.FromSeconds(1))
```

替換為：

```csharp
    .Range(1, 5)
```

然後我們得到這個輸出：

```
[T:1] Main thread
[T:12] Created thread for EventLoopScheduler
[T:12] 21/07/2023 15:02:09: Tick 1
[T:1] 21/07/2023 15:02:09: Main thread exiting
[T:12] 21/07/2023 15:02:09: Tick 2
[T:12] 21/07/2023 15:02:09: Tick 3
[T:12] 21/07/2023 15:02:09: Tick 4
[T:12] 21/07/2023 15:02:09: Tick 5
```

現在所有通知都來自執行緒 12，即為 `EventLoopScheduler` 建立的執行緒。請注意，即使在此處，`Range` 也沒有使用該排程器。不同之處在於 `Range` 預設為 `CurrentThreadScheduler`，因此它將從你碰巧呼叫它的任何執行緒產生其輸出。因此，即使它實際上沒有使用 `EventLoopScheduler`，它最終也會使用該排程器的執行緒，因為我們使用了該排程器來訂閱 `Range`。

因此，這說明了 `SubscribeOn` 正在兌現它的承諾：它確實決定了叫用 `Subscribe` 的環境定義。只是該環境定義是什麼並不總是有影響。如果 `Subscribe` 執行非簡單工作，這可能會有影響。例如，如果你使用 [`Observable.Create`](03_CreatingObservableSequences.md#observablecreate) 建立自訂序列，則 `SubscribeOn` 決定叫用你傳遞給 `Create` 之回呼函式的環境定義。但是 Rx 沒有「目前」排程器的概念——無法詢問「我是從哪個排程器叫用的？」——因此 Rx 運算子不會僅僅從訂閱它們的環境定義繼承其排程器。

在發出項目方面，Rx 提供的大多數來源分為三類之一。首先，響應來自上游來源的輸入而產生輸出的運算子（例如 `Where`、`Select` 或 `GroupBy`）通常從其自己的 `OnNext` 內部呼叫其觀察者的方法。因此，無論它們的來源可觀察序列在呼叫 `OnNext` 時是在什麼環境定義中執行，這都是運算子在呼叫其觀察者時將使用的環境定義。其次，反覆運算或根據時間產生項目的運算子將使用排程器（明確提供，或未指定時的預設類型）。第三，某些來源只是從它們喜歡的任何環境定義中產生項目。例如，如果 `async` 方法使用 `await` 並指定 `ConfigureAwait(false)`，那麼在 `await` 完成之後它可能處於幾乎任何執行緒和任何環境定義中，然後它可能會繼續在觀察者上叫用 `OnNext`。

只要來源遵循 [Rx 序列的基本規則](02_KeyTypes.md#the-fundamental-rules-of-rx-sequences)，就允許它從其喜歡的任何環境定義叫用其觀察者的方法。它可以選擇接受排程器作為輸入並使用它，但它沒有義務這樣做。如果你有這種想要馴服的桀驁不馴的來源，這就是 `ObserveOn` 擴充方法的用武之地。考慮以下相當愚蠢的範例：

```csharp
Observable
    .Interval(TimeSpan.FromSeconds(1))
    .SelectMany(tick => Observable.Return(tick, NewThreadScheduler.Default))
    .Subscribe(tick => 
      Console.WriteLine($"{DateTime.Now}-{Environment.CurrentManagedThreadId}: Tick {tick}"));
```

這故意導致每個通知到達不同的執行緒，如下輸出所示：

```
Main thread: 1
21/07/2023 15:19:56-12: Tick 0
21/07/2023 15:19:57-13: Tick 1
21/07/2023 15:19:58-14: Tick 2
21/07/2023 15:19:59-15: Tick 3
...
```

（它是透過為從 `Interval` 出現的每一個刻度呼叫 `Observable.Return` 並告訴 `Return` 使用 `NewThreadScheduler` 來實現這一點的。每次對 `Return` 的此類呼叫都將建立一個新執行緒。這是一個糟糕的主意，但它是獲得每次都從不同環境定義呼叫的來源的簡單方法。）如果我想施加一些順序，我可以新增對 `ObserveOn` 的呼叫：

```csharp
Observable
    .Interval(TimeSpan.FromSeconds(1))
    .SelectMany(tick => Observable.Return(tick, NewThreadScheduler.Default))
    .ObserveOn(new EventLoopScheduler())
    .Subscribe(tick => 
      Console.WriteLine($"{DateTime.Now}-{Environment.CurrentManagedThreadId}: Tick {tick}"));
```

我在這裡建立了一個 `EventLoopScheduler`，因為它建立了一個單一執行緒，並在該執行緒上執行每個排程的工作項目。現在輸出每次都顯示相同的執行緒 ID (13)：

```
Main thread: 1
21/07/2023 15:24:23-13: Tick 0
21/07/2023 15:24:24-13: Tick 1
21/07/2023 15:24:25-13: Tick 2
21/07/2023 15:24:26-13: Tick 3
...
```

因此，儘管由 `Observable.Return` 建立的每個新可觀察序列都會建立一個全新的執行緒，但 `ObserveOn` 確保我的觀察者的 `OnNext`（以及在呼叫它們的情況下的 `OnCompleted` 或 `OnError`）是透過指定的排程器叫用的。

### UI 應用程式中的 SubscribeOn 與 ObserveOn

如果你在使用者介面中使用 Rx，當你處理未在 UI 執行緒上提供通知的資訊來源時，`ObserveOn` 非常有用。你可以使用 `ObserveOn` 包裝任何 `IObservable<T>`，傳遞 `SynchronizationContextScheduler`（或特定於架構的類型，例如 `DispatcherScheduler`），以確保你的觀察者在 UI 執行緒上接收通知，從而可以安全地更新 UI。

`SubscribeOn` 在使用者介面中也很有用，可作為確保可觀察來源為啟動而執行的任何初始化工作不會發生在 UI 執行緒上的一種方式。

對於任何一個視窗，大多數 UI 架構都指定一個特定的執行緒用於接收來自使用者的通知以及更新 UI。避免封鎖此 UI 執行緒至關重要，因為這樣做會導致不良的使用者體驗——如果你在 UI 執行緒上執行工作，在該工作完成之前它將無法回應使用者輸入。一般規則是，如果你導致使用者介面失去回應超過 100 毫秒，使用者會感到惱火，因此你不應在 UI 執行緒上執行花費比這更長的工作。當微軟首次推出其應用程式市集（隨 Windows 8 一起推出）時，他們指定了更嚴格的限制：如果你的應用程式封鎖 UI 執行緒超過 50 毫秒，可能不允許進入市集。憑藉現代處理器提供的處理能力，你可以在 50 毫秒內實現大量處理。即使在行動裝置中相對低功耗的處理器上，這也足以執行數百萬條指令。但是，任何涉及 I/O 的操作（讀取或寫入檔案，或等待來自任何類型的網路服務的回應）都不應該在 UI 執行緒上完成。建立具回應性之 UI 應用程式的一般模式是：

- 接收有關某種使用者動作的通知
- 如果需要緩慢的工作，在背景執行緒上執行此工作
- 將結果傳回 UI 執行緒
- 更新 UI

這非常適合 Rx：回應事件、可能組合多個事件、將資料傳遞給鏈結的方法呼叫。透過納入排程，我們甚至有能力離開並返回 UI 執行緒，以獲得使用者要求的回應敏捷的應用程式體驗。

考慮使用 Rx 填入 `ObservableCollection<T>` 的 WPF 應用程式。你可以使用 `SubscribeOn` 確保主要工作不在 UI 執行緒上完成，然後使用 `ObserveOn` 確保在正確的執行緒上收到通知。如果你未能使用 `ObserveOn` 方法，則你的 `OnNext` 處理常式將在引發通知的同一執行緒上叫用。在大多數 UI 架構中，這將導致某種不受支援/跨執行緒的例外狀況。在此範例中，我們訂閱了一系列 `Customers`。我正在使用 `Defer`，以便如果 `GetCustomers` 在返回其 `IObservable<Customer>` 之前執行任何緩慢的初始工作，該工作直到我們訂閱時才會發生。然後我們使用 `SubscribeOn` 呼叫該方法並在工作集區執行緒上執行訂閱。然後我們確保在收到 `Customer` 通知時，將它們新增至 `Dispatcher` 上的 `Customers` 集合中。

```csharp
Observable
    .Defer(() => _customerService.GetCustomers())
    .SubscribeOn(TaskPoolScheduler.Default)
    .ObserveOn(DispatcherScheduler.Instance) 
    .Subscribe(Customers.Add);
```

Rx 還為 `IObservable<T>` 提供了 `SubscribeOnDispatcher()` 和 `ObserveOnDispatcher()` 擴充方法，這些方法自動使用當前執行緒的 `Dispatcher`（以及 `CoreDispatcher` 的等效項）。雖然這些可能稍微更方便，但它們會使測試程式碼變得更加困難。我們在[測試 Rx](16_TestingRx.md) 章節中解釋了原因。

## 並行性的陷阱 (Concurrency pitfalls)

在你的應用程式中引入並行性將會增加其複雜性。如果透過新增並行層並沒有顯著改善你的應用程式，那麼你應該避免這樣做。並行應用程式可能會出現維護問題，其症狀表現在偵錯、測試和重構方面。

並行性引入的常見問題是不可預測的時間點。不可預測的時間點可能是由系統上的可變負載以及系統組態的差異（例如不同的核心時脈速度和可用處理器數量）引起的。這些最終可能導致[死結 (deadlocks)](http://en.wikipedia.org/wiki/Deadlock)、[活結 (livelocks)](http://en.wikipedia.org/wiki/Deadlock#Livelock) 和損毀的狀態。

向應用程式引入並行性的一個特別重大的危險是，你可能會悄悄引入錯誤。由不可預測的時間點引起的錯誤眾所周知難以偵測，使得這類缺陷很容易躲過開發、品管 (QA) 和使用者驗收測試 (UAT)，並且僅在實際執行環境 (Production) 中表現出來。然而，Rx 在簡化可觀察序列的並行處理方面做得非常好，以至於許多此類擔憂都可以減輕。你仍然可能會建立出問題，但如果你遵循指引，你就可以更放心地知道你已大幅減少了發生不必要競爭條件的機會。

在後面的章節[測試 Rx](16_TestingRx.md) 中，我們將探討 Rx 如何提高你測試並行工作流程的能力。

### 鎖死 (Lock-ups)

Rx 可以簡化並行性的處理，但它並不能免於死結。某些呼叫（如 `First`、`Last`、`Single` 和 `ForEach`）是封鎖性的——它們在所等待的事情發生之前不會返回。以下範例顯示這非常容易導致死結發生：

```csharp
var sequence = new Subject<int>();

Console.WriteLine("Next line should lock the system.");

IEnumerable<int> value = sequence.First();
sequence.OnNext(1);

Console.WriteLine("I can never execute....");
```

`First` 方法在其來源發出序列之前不會返回。但是導致此來源發出序列的程式碼在呼叫 `First` _之後_的那一行。因此，在 `First` 返回之前，來源無法發出序列。這種死結形式——雙方都無法繼續進行，直到對方繼續進行為止——通常被稱為_致命擁抱 (deadly embrace)_。如這段程式碼所示，即使在單執行緒程式碼中也完全可能發生致命擁抱。事實上，此程式碼的單執行緒本質正是導致死結的原因：我們有兩個操作（等待第一個通知，以及傳送第一個通知）而只有單一執行緒。這本不必成為問題。如果我們使用了 `FirstAsync` 並附加了一個觀察者，那麼當來源 `Subject<int>` 叫用其 `OnNext` 時，`FirstAsync` 就會執行其邏輯。但這比僅僅呼叫 `First` 並將結果指派給變數更複雜。

這是一個過於簡化的範例來說明該行為，我們永遠不會在正式環境中編寫此類程式碼。（即使我們這樣做了，它失敗得如此迅速和一致，以至於我們會立即意識到問題。）但在真實的應用程式程式碼中，這些類型的問題可能更難被發現。競爭條件通常會在整合點悄悄進入系統，因此問題不一定是任何單一程式碼中的證據：時間問題可能會由於我們如何將多個程式碼組合在一起而出現。

下一個範例可能稍微難以偵測，但與我們第一個不切實際的範例僅有一小步之遙。基本概念是我們有一個代表使用者介面中按鈕點擊的 Subject。代表使用者輸入的事件處理常式由 UI 架構叫用。我們只需向架構提供事件處理常式方法，每當感興趣的事件（例如按鈕被點擊）發生時，它就會為我們呼叫它們。此程式碼在代表點擊的 Subject 上呼叫 `First`，但在這裡可能導致問題並不像在上一個範例中那麼明顯：

```csharp
public Window1()
{
    InitializeComponent();
    DataContext = this;
    Value = "Default value";
    
    // Deadlock! We need the dispatcher to continue to allow me to click the button to produce a value
    Value = _subject.First();
    
    // This will have the intended effect, but because it does not block,
    // we can call this on the UI thread without deadlocking.
    //_subject.FirstAsync(1).Subscribe(value => Value = value);
}

private void MyButton_Click(object sender, RoutedEventArgs e)
{
    _subject.OnNext("New Value");
}

public string Value
{
    get { return _value; }
    set
    {
        _value = value;
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("Value"));
    }
}
```

較早的範例在 `First` 返回後呼叫 Subject 的 `OnNext`，這使得相對容易看出如果 `First` 沒有返回，那麼 Subject 就不會發出通知。但在此處並不明顯。`MyButton_Click` 事件處理常式將在對 `InitializeComponent` 的呼叫內部設定（在 WPF 程式碼中很正常），因此顯然我們已經完成了啟用事件流動的必要設定。當我們到達對 `First` 的呼叫時，UI 架構已經知道如果使用者點擊 `MyButton`，它應該呼叫 `MyButton_Click`，並且該方法將導致 Subject 發出一個值。

使用 `First` 本身並沒有本質上的錯誤。（冒險，是的，但在某些情境下，完全相同的程式碼是絕對沒問題的。）問題在於我們使用它的環境定義。此程式碼位於 UI 元件的建構函式中，並且這些建構函式始終在與該視窗的 UI 元件相關聯的特定執行緒上執行。（這碰巧是一個 WPF 範例，但其他 UI 架構的工作方式相同。）這也是 UI 架構將用於傳遞有關使用者輸入之通知的相同執行緒。如果我們封鎖此 UI 執行緒，我們就會阻止 UI 架構叫用我們的按鈕點擊事件處理常式。因此，此封鎖呼叫正在等待只能從它正在封鎖的同一執行緒引發的事件，從而造成死結。

你可能會開始有一種印象，即我們應該盡量避免在 Rx 中使用封鎖呼叫。這是一個很好的經驗法則。我們可以透過註解掉使用 `First` 的那一行，並取消註解其下方包含此程式碼的那一行來修復上面的程式碼：

```csharp
_subject.FirstAsync(1).Subscribe(value => Value = value);
```

這使用了 `FirstAsync`，它執行相同的工作，但採用了不同的方法。它實作了相同的邏輯，但它返回一個 `IObservable<T>`，如果我們想在第一個值最終出現時接收它，我們必須訂閱該可觀察序列。它比僅將 `First` 的結果指派給 `Value` 屬性更複雜，但它更能適應我們無法知道該來源何時會產生值的事實。

如果你進行大量的 UI 開發，最後一個範例對你來說可能顯得顯而易見是錯誤的：我們在視窗的建構函式中編寫了程式碼，該程式碼在使用者點擊該視窗中的按鈕之前不允許建構函式完成。在建構完成之前視窗甚至不會出現，因此等待使用者點擊按鈕是沒有意義的。在我們的建構函式完成之前，該按鈕甚至在螢幕上都不可見。此外，經驗豐富的 UI 開發人員知道你不能只暫停一切並等待使用者的特定動作。（即使是強制回應對話方塊，在繼續之前確實需要回應，也不會封鎖 UI 執行緒。）但正如接下來的範例所示，問題很容易變得更難以察覺。在此範例中，按鈕的點擊處理常式將嘗試從透過介面公開的可觀察序列中取得第一個值。

```csharp
public partial class Window1 : INotifyPropertyChanged
{
    //Imagine DI here.
    private readonly IMyService _service = new MyService(); 
    private int _value2;

    public Window1()
    {
        InitializeComponent();
        DataContext = this;
    }

    public int Value2
    {
        get { return _value2; }
        set
        {
            _value2 = value;
            var handler = PropertyChanged;
            if (handler != null) handler(this, new PropertyChangedEventArgs(nameof(Value2)));
        }
    }

    #region INotifyPropertyChanged Members
    public event PropertyChangedEventHandler PropertyChanged;
    #endregion

    private void MyButton2_Click(object sender, RoutedEventArgs e)
    {
        Value2 = _service.GetTemperature().First();
    }
}
```

與較早的範例不同，這並未嘗試封鎖建構函式中的進展。對 `First` 的封鎖呼叫發生在按鈕點擊處理常式中（接近結尾處的 `MyButton2_Click` 方法）。這個範例更有趣，因為這種事情不一定是錯誤的。應用程式經常在點擊處理常式中執行封鎖操作：當我們點擊按鈕以儲存文件的複本時，我們期望應用程式執行所有必要的 IO 工作以將我們的資料寫入儲存裝置。使用現代固態硬碟 (SSD)，這通常發生得非常快，以至於看起來是瞬間完成的，但在機械硬碟普及的年代，應用程式在儲存我們的文件時暫時失去回應並不罕見。即使在今天，如果你的儲存裝置是遠端的，並且網路問題導致延遲，這也可能會發生。

因此，即使我們已經學會對像 `First` 這樣的封鎖操作保持警惕，在該範例中它也可能是可以接受的。僅透過檢視此程式碼無法確定這一點。這完全取決於 `GetTemperature` 返回什麼樣的可觀察序列，以及它產生其項目的方式。對 `First` 的呼叫將在 UI 執行緒上封鎖，直到第一個項目可用為止，因此如果產生該第一個項目需要存取 UI 執行緒，這將產生死結。以下是建立該問題的一種稍微刻意的方式：

```csharp
class MyService : IMyService
{
    public IObservable<int> GetTemperature()
    {
        return Observable.Create<int>(
            o =>
            {
                o.OnNext(27);
                o.OnNext(26);
                o.OnNext(24);
                return () => { };
            })
            .SubscribeOnDispatcher();
    }
}
```

這透過進行一系列對 `OnNext` 的呼叫來模擬旨在模擬實際溫度感測器的行為。但它執行了一些奇怪的明確排程：它呼叫了 `SubscribeOnDispatcher`。這是一個擴充方法，實際上呼叫了 `SubscribeOn(DispatcherScheduler.Current.Dispatcher)`。這實際上告訴 Rx，當某些程式碼嘗試訂閱 `GetTemperature` 返回的 `IObservable<int>` 時，該訂閱呼叫應透過在 UI 執行緒上執行其工作項目的 WPF 特定排程器來完成。（嚴格來說，WPF 確實允許多個 UI 執行緒，因此更準確地說，此程式碼僅在你於 UI 執行緒上呼叫它時才有效，如果你這樣做，排程器將確保工作項目被排程到同一個 UI 執行緒上。）

其效果是當我們的點擊處理常式呼叫 `First` 時，將進而訂閱 `GetTemperature` 返回的 `IObservable<int>`，並且因為使用了 `SubscribeOnDispatcher`，所以這不會立即叫用傳遞給 `Observable.Create` 的回呼函式。相反，它排程了一個工作項目，當 UI 執行緒（即我們正在執行的執行緒）變為可用時將執行該回呼函式。它現在不被認為是可用的，因為它正處於處理按鈕點擊的中途。將此工作項目交給排程器後，`Subscribe` 呼叫返回到 `First` 方法。而 `First` 方法現在坐下來等待第一個項目的出現。由於在此發生之前它不會返回，因此在該事件發生之前 UI 執行緒將不被視為可用，這意味著原本應該產生該第一個項目的已排程工作項目永遠無法執行，我們就遇到了死結。

這歸結為與這些與 `First` 相關的死結範例中的第一個相同的基本問題。我們有兩個程序：項目的產生，以及等待項目的發生。這些需要並行進行——我們需要「等待第一個項目」的邏輯在來源發出其第一個項目時處於啟動並執行狀態。這些範例都僅使用單一執行緒，這使得使用單一封鎖呼叫 (`First`) 來設定監看第一個項目的程序並同時等待該情況發生是一個壞主意。但即使在所有三種情況下它都是相同的基本問題，隨著程式碼變得更加複雜，它變得更難被看清。對於真實的應用程式程式碼，通常比這更難看清死結的根本原因。

到目前為止，本章可能透過關注你可能面臨的問題以及它們在實務中通常難以發現的事實，使並行性看起來全是一片陰霾；但這並非本意。
儘管採用 Rx 無法神奇地避免經典的並行性問題，但只要你遵循以下兩條規則，Rx 就可以讓你更容易做對。

- 只有最頂層訂閱者應該做出排程決策
- 避免使用封鎖呼叫：例如 `First`、`Last` 和 `Single`

最後一個範例因一個簡單的問題而陷入困境；`GetTemperature` 服務在主導排程模型，而實際上它根本無權這樣做。代表溫度感測器的程式碼不需要知道我正在使用特定的 UI 架構，當然也不應該單方面決定它將在 WPF 使用者介面執行緒上執行某些工作。

剛開始使用 Rx 時，很容易說服自己將排程決策烘焙到底層中在某種程度上是在「提供幫助」。「看！」你可能會說。「我不僅提供了溫度讀數，而且我還使其自動在 UI 執行緒上通知你，因此你就不必費心使用 `ObserveOn` 了。」出發點可能是好的，但這極易造成執行緒惡夢。

只有設定訂閱並使用其結果的程式碼才能對並行性要求有完整的全局了解，因此這才是選擇使用哪些排程器的正確層級。較低層級的程式碼不應嘗試參與；它們應該只做被告知的事情。（Rx 本身可以說略微打破了這個規則，在需要的地方選擇預設排程器。但它做出了非常保守的選擇，旨在最大限度地減少死結的機會，並且始終允許應用程式透過指定排程器來取得控制權。）

請注意，遵循上述兩條規則中的任何一條就足以防止此範例中的死結。但最好同時遵循這兩條規則。

這確實留下了一個未回答的問題：頂層訂閱者應該_如何_做出排程決策？我已經確定了需要做出決策的程式碼區域，但是該決策應該是什麼？這將取決於你正在編寫的應用程式類型。對於 UI 程式碼，這種模式通常效果很好：「在背景執行緒上訂閱；在 UI 執行緒上觀察」。對於 UI 程式碼，死結的風險之所以產生，是因為 UI 執行緒實際上是一個共享資源，並且對該資源的爭用可能會產生死結。因此，策略是盡可能避免需要該資源：不需要在該執行緒上的工作不應該在該執行緒上，這就是為什麼在工作執行緒上執行訂閱（例如，透過使用 `TaskPoolScheduler`）可以降低死結風險的原因。

由此可見，如果你有決定何時產生事件的可觀察來源（例如計時器，或代表來自外部資訊摘要或裝置之輸入的來源），你也會希望這些來源在工作執行緒上排程工作。只有當我們需要更新使用者介面時，我們才需要我們的程式碼在 UI 執行緒上執行，因此我們透過將 `ObserveOn` 與適當的支援 UI 的排程器（例如 WPF `DispatcherScheduler`）結合使用，將其延後到最後一刻。如果我們有一個由多個運算子組成的複雜 Rx 查詢，則此 `ObserveOn` 應該放在最末尾，就在我們呼叫 `Subscribe` 附加將更新 UI 的處理常式之前。這樣，只有最後一步，即更新 UI，才需要存取 UI 執行緒。當此執行時，所有複雜處理都將完成，因此這應該能夠非常快速地執行，幾乎立即放棄對 UI 執行緒的控制，從而提高應用程式的回應速度，並降低死結風險。

其他情境將需要其他策略，但死結的一般原則始終相同：了解哪些共享資源需要獨佔存取權。例如，如果你有一個感測器函式庫，它可能會建立一個專用執行緒來監視裝置並報告新的測量值，如果它規定必須在該執行緒上完成某些工作，這將與 UI 情境非常相似：有一個你需要避免封鎖的特定執行緒。相同的方法可能適用於此處。但這並不是唯一的情境。

你可以想像一個其中某些資料結構被共享的資料處理應用程式。在這些情況下，允許從任何執行緒存取此類資料結構是很常見的，但需要一次一個執行緒這樣做。通常我們會使用執行緒同步基元來防止並行使用這些關鍵資料結構。在這些情況下，死結的風險並不是由使用特定執行緒引起的。相反，它們是由於以下可能性而產生的：一個執行緒無法取得進展，因為某個其他執行緒正在使用共享資料結構，但該其他執行緒正在等待第一個執行緒執行某個操作，並且在該情況發生之前不會釋放其在該資料結構上的鎖定。在此避免問題的最簡單方法是盡可能避免封鎖。避免像 `First` 這樣的方法，偏好其非封鎖等效項，例如 `FirstAsync`。（如果存在無法避免封鎖的情況，請盡量避免在持有保護對共享資料之存取的鎖定時這樣做。如果你真的也無法避免這種情況，那麼就沒有簡單的答案。你現在必須開始考慮鎖定階層以有系統地避免死結，就像你沒有使用 Rx 一樣。）非封鎖風格是使用 Rx 處理事務的自然方式，這也是 Rx 可以幫助你在這些情況下避免與並行相關的問題的主要方式。

## 排程器的高級功能 (Advanced features of schedulers)

排程器提供了一些主要在編寫需要與排程器互動的可觀察來源時感興趣的功能。使用排程器最常見的方式是在設定訂閱時，要麼在建立可觀察來源時將其作為引數提供，要麼將其傳遞給 `SubscribeOn` 和 `ObserveOn`。但是，如果你需要編寫一個可觀察來源，使其按照自己選擇的排程產生項目（例如，假設你正在編寫一個代表某個外部資料來源的函式庫，並且你想將其呈現為 `IObservable<T>`），你可能需要使用這些更高級的功能中的一部分。

### 傳遞狀態 (Passing state)

`IScheduler` 定義的所有方法都採用 `state` 引數。以下是介面定義：

```csharp
public interface IScheduler
{
    DateTimeOffset Now { get; }

    IDisposable Schedule<TState>(TState state, 
                                 Func<IScheduler, TState, IDisposable> action);
    
    IDisposable Schedule<TState>(TState state, 
                                 TimeSpan dueTime, 
                                 Func<IScheduler, TState, IDisposable> action);
    
    IDisposable Schedule<TState>(TState state, 
                                 DateTimeOffset dueTime, 
                                 Func<IScheduler, TState, IDisposable> action);
}
```

排程器不在乎此 `state` 引數中有什麼。它只是在執行你的工作項目時將其未修改地傳遞到你的回呼函式中。這提供了一種為該回呼函式提供環境定義的方法。這並非嚴格必要：我們作為 `action` 傳遞的委派可以包含我們需要的任何狀態。最簡單的方法是在 Lambda 中擷取變數。但是，如果你檢視 [Rx 原始程式碼](https://github.com/dotnet/reactive/)，你會發現它通常不這樣做。例如，`Range` 運算子的核心是一個名為 `LoopRec` 的方法，如果你檢視 [`LoopRec` 的原始程式碼](https://github.com/dotnet/reactive/blob/95d9ea9d2786f6ec49a051c5cff47dc42591e54f/Rx.NET/Source/src/System.Reactive/Linq/Observable/Range.cs#L55-L73)，你會看到它包含這一行：

```csharp
var next = scheduler.Schedule(this, static (innerScheduler, @this) => @this.LoopRec(innerScheduler));
```

從邏輯上講，`Range` 只是一個針對其產生的每個項目執行一次的迴圈。但為了啟用並行執行並避免堆疊溢位，它透過將迴圈的每次反覆運算排程為個別工作項目來實作這一點。（該方法被稱為 `LoopRec`，因為它在邏輯上是一個遞迴迴圈：它是透過呼叫 `Schedule` 啟動的，並且每次排程器呼叫此方法時，它都會再次呼叫 `Schedule` 以請求執行下一個項目。這實際上不會在任何 Rx 的內建排程器（甚至是 `ImmediateScheduler`）中引起遞迴，因為它們都會偵測到這一點並安排在當前項目返回後執行下一個項目。但是如果你編寫了最簡單樸素的排程器，這實際上會在執行階段以遞迴告終，如果你嘗試建立大型序列，可能會導致堆疊溢位。）

請注意，傳遞給 `Schedule` 的 Lambda 已使用 `static` 進行註解。這告訴 C# 編譯器我們的意圖是_不_擷取任何變數，任何嘗試這樣做的操作都應該導致編譯器錯誤。這樣做的好處是編譯器能夠產生為每次呼叫重複使用相同委派執行個體的程式碼。第一次執行時，它將建立一個委派並將其儲存在隱藏欄位中。在此的每一次後續執行中（無論是在同一範圍的未來反覆運算中，還是對於全新的範圍執行個體），它都可以一次又一次地使用相同的委派。這是可能的，因為委派不擷取任何狀態。這避免了每次繞過迴圈時配置新物件。

Rx 函式庫難道不能使用更直接的方法嗎？我們可以選擇不使用狀態，將 `null` 狀態傳遞給排程器，然後捨棄傳遞給回呼函式的狀態引數：

```csharp
// Less weird, but less efficient:
var next = scheduler.Schedule<object?>(null, (innerScheduler, _) => LoopRec(innerScheduler));
```

這避免了前面範例傳遞我們自己的 `this` 引數的奇怪之處：現在我們只是以普通方式叫用 `LoopRec` 執行個體成員：我們隱含地使用了範圍內的 `this` 參考。因此，這將建立一個擷取該隱含 `this` 參考的委派。這有效，但效率低下：它將迫使編譯器產生配置幾個物件的程式碼。它會建立一個包含保存所擷取之 `this` 欄位的物件，然後它需要建立一個相異的委派執行個體，該執行個體具有對該擷取物件的參考。

實際上在 `Range` 實作中的更複雜程式碼避免了這一點。它透過使用 `static` 註解 Lambda 來停用擷取。這可以防止程式碼依賴隱含的 `this` 參考。因此它必須安排 `this` 參考對回呼函式可用。這正是 `state` 引數存在的目的。它提供了一種傳入某些個別工作項目狀態的方法，以便你可以避免在每次反覆運算中擷取變數的額外負荷。

### 未來排程 (Future scheduling)

我之前談到了基於時間的運算子，以及啟用此功能的 `ISchedule` 的兩個基於時間的成員，但我還沒有展示如何使用它。這些使你能夠排程將在未來執行的動作。（這取決於處理序在需要時繼續執行。如前幾章所述，`System.Reactive` 不支援持久、耐久的訂閱。因此，如果你想排程未來幾天的內容，你可能需要檢視 [Reaqtor](https://reaqtive.net/)。）你可以透過呼叫採用 `DateTimeOffset` 的 `Schedule` 多載來指定叫用動作的確切時間點，或者你可以使用基於 `TimeSpan` 的多載來指定在叫用動作之前等待的時間週期。

你可以像這樣使用 `TimeSpan` 多載：

```csharp
var delay = TimeSpan.FromSeconds(1);
Console.WriteLine("Before schedule at {0:o}", DateTime.Now);

scheduler.Schedule(delay, () => Console.WriteLine("Inside schedule at {0:o}", DateTime.Now));
Console.WriteLine("After schedule at  {0:o}", DateTime.Now);
```

輸出：

```
Before schedule at 2012-01-01T12:00:00.000000+00:00
After schedule at 2012-01-01T12:00:00.058000+00:00
Inside schedule at 2012-01-01T12:00:01.044000+00:00
```

這說明了這裡的排程是非封鎖的，因為「before」和「after」呼叫在時間上非常接近。（對於大多數排程器來說都是這樣，但正如前面所討論的，`ImmediateScheduler` 的運作方式不同。在這種情況下，你會在 Inside 訊息之後看到 After 訊息。這就是為什麼預設情況下沒有任何計時運算子會使用 `ImmediateScheduler` 的原因。）你還可以看到，在排程動作大約一秒鐘後，它被叫用了。

你可以使用 `DateTimeOffset` 多載指定排程工作的特定時間點。如果出於某種原因，你指定的時間點在過去，則會盡快排程該動作。請注意，系統時鐘的變更會使事情複雜化。Rx 的排程器確實做出了一些調整來處理時鐘漂移，但對系統時鐘的突然大幅變更可能會引起一些短期的混亂。

### 取消 (Cancellation)

`Schedule` 的每個多載都會返回一個 `IDisposable`，對此呼叫 `Dispose` 將取消已排程的工作。在上一個範例中，我們排程了在一秒鐘後叫用的工作。我們可以透過處置返回值來取消該工作。

```csharp
var delay = TimeSpan.FromSeconds(1);
Console.WriteLine("Before schedule at {0:o}", DateTime.Now);

var workItem = scheduler.Schedule(delay, 
   () => Console.WriteLine("Inside schedule at {0:o}", DateTime.Now));

Console.WriteLine("After schedule at  {0:o}", DateTime.Now);

workItem.Dispose();
```

輸出：

```
Before schedule at 2012-01-01T12:00:00.000000+00:00
After schedule at 2012-01-01T12:00:00.058000+00:00
```

請注意，已排程的動作從未發生，因為我們幾乎立即取消了它。

當使用者在排程器能夠叫用排程動作方法之前取消該方法時，該動作只會從工作佇列中移除。這就是我們在上面的範例中看到的。可以取消已經在執行的排程工作，這就是為什麼工作項目回呼函式需要返回 `IDisposable` 的原因：如果在你嘗試取消工作項目時工作已經開始，Rx 會在你的工作項目回呼函式返回的 `IDisposable` 上呼叫 `Dispose`。這為使用者提供了一種取消可能已經在執行的工作的方式。此工作可能是某種 I/O、繁重的運算或使用 `Task` 來執行某些工作。

你可能想知道這種機制怎麼可能有用：工作項目回呼函式需要已經返回，Rx 才能叫用它返回的 `IDisposable`。實際上，只有在返回排程器後工作仍在繼續的情況下，才能使用此機制。你可以啟動另一個執行緒，使工作並行發生，儘管我們通常嘗試避免在 Rx 中建立執行緒。另一種可能性是已排程的工作項目叫用了某些非同步 API 並返回而無需等待其完成。如果該 API 提供了取消功能，你可以返回取消該 API 的 `IDisposable`。

為了說明運作中的取消功能，這個稍微不切實際的範例將某些工作作為 `Task` 執行，以使其能夠在我們的回呼函式返回後繼續。它只是透過執行自旋等待並將值新增至 `list` 引數來偽造某些工作。這裡的關鍵是我們建立了一個 `CancellationToken` 以便能夠告訴工作我們希望它停止，並且我們返回一個將此權杖置於已取消狀態的 `IDisposable`。

```csharp
public IDisposable Work(IScheduler scheduler, List<int> list)
{
    CancellationTokenSource tokenSource = new();
    CancellationToken cancelToken = tokenSource.Token;
    Task task = new(() =>
    {
        Console.WriteLine();
   
        for (int i = 0; i < 1000; i++)
        {
            SpinWait sw = new();
   
            for (int j = 0; j < 3000; j++) sw.SpinOnce();
   
            Console.Write(".");
   
            list.Add(i);
   
            if (cancelToken.IsCancellationRequested)
            {
                Console.WriteLine("Cancellation requested");
                
                // cancelToken.ThrowIfCancellationRequested();
                
                return;
            }
        }
    }, cancelToken);
   
    task.Start();
   
    return Disposable.Create(tokenSource.Cancel);
}
```

此程式碼排程了上述程式碼，並允許使用者透過按 Enter 來取消處理工作

```csharp
List<int> list = new();
Console.WriteLine("Enter to quit:");

IDisposable token = scheduler.Schedule(list, Work);
Console.ReadLine();

Console.WriteLine("Cancelling...");

token.Dispose();

Console.WriteLine("Cancelled");
```

輸出：

```
Enter to quit:
........
Cancelling...
Cancelled
Cancellation requested
```

這裡的問題是我們引入了 `Task` 的明確使用，因此我們以排程器控制之外的方式增加了並行性。Rx 函式庫通常透過接受排程器參數來允許控制引入並行性的方式。如果目標是啟用長時間執行的反覆運算工作，我們可以透過使用 Rx 遞迴排程器功能來避免必須啟動新執行緒或工作。我已經在[傳遞狀態](#passing-state)一節中稍微討論過這一點，但是有幾種方法可以解決它。

### 遞迴 (Recursion)

除了 `IScheduler` 方法之外，Rx 還以擴充方法的形式定義了 `Schedule` 的各種多載。其中一些採用一些看起來很奇怪的委派作為參數。特別注意 `Schedule` 擴充方法的每個多載中的最後一個參數。

```csharp
public static IDisposable Schedule(
    this IScheduler scheduler, 
    Action<Action> action)
{...}

public static IDisposable Schedule<TState>(
    this IScheduler scheduler, 
    TState state, 
    Action<TState, Action<TState>> action)
{...}

public static IDisposable Schedule(
    this IScheduler scheduler, 
    TimeSpan dueTime, 
    Action<Action<TimeSpan>> action)
{...}

public static IDisposable Schedule<TState>(
    this IScheduler scheduler, 
    TState state, 
    TimeSpan dueTime, 
    Action<TState, Action<TState, TimeSpan>> action)
{...}

public static IDisposable Schedule(
    this IScheduler scheduler, 
    DateTimeOffset dueTime, 
    Action<Action<DateTimeOffset>> action)
{...}

public static IDisposable Schedule<TState>(
    this IScheduler scheduler, 
    TState state, DateTimeOffset dueTime, 
    Action<TState, Action<TState, DateTimeOffset>> action)
{...}   
```

這些多載中的每一個都採用一個委派「action」，該委派允許你遞迴呼叫「action」。這可能看起來是一個非常奇特的簽章，但它允許我們實現類似的邏輯遞迴反覆運算方法，正如你在[傳遞狀態](#passing-state)一節中看到的那樣，但可能以更簡單的方式。

此範例使用最簡單的遞迴多載。我們有一個可以遞迴呼叫的 `Action`。

```csharp
Action<Action> work = (Action self) =>
{
    Console.WriteLine("Running");
    self();
};

var token = s.Schedule(work);
    
Console.ReadLine();
Console.WriteLine("Cancelling");

token.Dispose();

Console.WriteLine("Cancelled");
```

輸出：

```
Enter to quit:
Running
Running
Running
Running
Cancelling
Cancelled
Running
```

請注意，我們不必在委派中編寫任何取消程式碼。Rx 代表我們處理了迴圈並檢查了取消。由於每個個別反覆運算都被排程為單獨的工作項目，因此沒有長時間執行的工作，因此讓排程器完全處理取消就足夠了。

這些多載與直接使用 `IScheduler` 方法之間的主要區別在於，你不需要將另一個回呼函式直接傳遞給排程器。你只需叫用提供的 `Action`，它就會排程對你的方法的另一次呼叫。如果你對狀態引數沒有任何用處，它們也使你能夠不傳遞狀態引數。

正如前面幾節所提到的，雖然這在邏輯上表示遞迴，但 Rx 保護我們免於堆疊溢位。排程器透過在執行遞迴呼叫之前等待方法返回來實作這種遞迴樣式。
    
這結束了我們的排程與執行緒導覽。接下來，我們將檢驗計時的相關主題。
