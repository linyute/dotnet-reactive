# 測試 Rx (Testing Rx)

現代品質保證標準要求全面的自動化測試，這有助於評估和預防錯誤。良好的實務做法是擁有一套驗證正確行為的測試套件，並將其作為建構程序的一部分執行，以儘早偵測迴歸 (regression)。

`System.Reactive` 原始程式碼包含一個全面的測試套件。測試基於 Rx 的程式碼會帶來一些挑戰，特別是涉及對時間敏感的運算子時。Rx.NET 的測試套件包含許多旨在演練棘手邊緣案例的測試，以確保在負載下的可預測行為。這之所以可能，僅僅是因為 Rx.NET 被設計為可測試的。

在本章中，我們將展示如何在自己的程式碼中利用 Rx 的可測試性。

## 虛擬時間 (Virtual Time)

在 Rx 中處理時間點很常見。如你所見，它提供了幾個考慮時間的運算子，這帶來了挑戰。我們不想引入緩慢的測試，因為這會使測試套件花費太長時間執行，但是我們該如何測試一個在提交查詢之前等待使用者停止輸入半秒鐘的應用程式呢？非確定性測試也可能是一個問題：當存在競爭條件時，很難可靠地重新建立這些條件。

[排程與執行緒](11_SchedulingAndThreading.md)一章描述了排程器如何使用虛擬化的時間表示。這對於使測試能夠驗證與時間相關的行為至關重要。它讓我們能夠控制 Rx 對時間進展的感知，使我們能夠編寫在邏輯上需要幾秒鐘但在微秒內執行的測試。

考慮這個範例，我們建立一個在五秒鐘內每秒發布一次值的序列。

```csharp
IObservable<long> interval = Observable
    .Interval(TimeSpan.FromSeconds(1))
    .Take(5);
```

一個確保以一秒為間隔產生五個值的單純測試將需要五秒鐘才能執行。這不好；我們希望在五秒鐘內執行數百個甚至數千個測試。另一個非常普遍的要求是測試逾時。在此，我們嘗試測試一分鐘的逾時。

```csharp
var never = Observable.Never<int>();
var exceptionThrown = false;

never.Timeout(TimeSpan.FromMinutes(1))
     .Subscribe(
        i => Console.WriteLine("This will never run."),
        ex => exceptionThrown = true);

Assert.IsTrue(exceptionThrown);
```

看起來我們別無選擇，只能讓測試在執行該斷言之前等待一分鐘。在實務上，我們會希望等待一分鐘多一點，因為如果執行測試的電腦很忙，它觸發逾時的時間可能會比我們要求的晚一點。這種類型的情境因即使受測程式碼沒有真正的問題也偶爾導致測試失敗而臭名昭著。

沒有人想要緩慢且不一致的測試。因此，讓我們檢驗 Rx 如何幫助我們避免這些問題。

## TestScheduler

[排程與執行緒](11_SchedulingAndThreading.md)一章解釋了排程器如何確定何時以及如何執行程式碼，以及它們如何追蹤時間。我們在該章中檢驗的大多數排程器解決了各種執行緒問題，當涉及時間點時，它們都試圖在請求的時間執行工作。但是 Rx 提供了 `TestScheduler`，它以完全不同的方式處理時間。它利用排程器控制所有與時間相關之行為的事實，允許我們模擬和控制時間。

**注意：**`TestScheduler` 不在主要的 `System.Reactive` 套件中。你將需要新增對 `Microsoft.Reactive.Testing` 的參考才能使用它。

任何排程器都會維護要執行的動作佇列。每個動作都會指派一個應該執行的時間點。（有時該時間是「盡快」，但基於時間的運算子通常會排程工作在未來的某個特定時間執行。）如果我們使用 `TestScheduler`，它將有效地表現為時間靜止，直到我們告訴它我們希望時間前進為止。

在此範例中，我們透過使用最簡單的 `Schedule` 多載來排程立即執行的工作。即使這實際上要求盡快執行工作，`TestScheduler` 也總是等待我們告訴它我們已準備好，然後才處理新排入佇列的工作。我們將虛擬時鐘向前推進一個刻度 (tick)，此時它將執行該排入佇列的工作。（每當我們推進虛擬時間時，它都會執行所有新排入佇列的「盡快」工作。如果我們將時間推進得足夠遠，以至於先前在邏輯上處於未來的工作現在可執行，它也會執行該工作。）

```csharp
var scheduler = new TestScheduler();
var wasExecuted = false;
scheduler.Schedule(() => wasExecuted = true);
Assert.IsFalse(wasExecuted);
scheduler.AdvanceBy(1); // execute 1 tick of queued actions
Assert.IsTrue(wasExecuted);
```

`TestScheduler` 實作了 `IScheduler` 介面，並定義了允許我們控制和監視虛擬時間的方法。這顯示了這些附加方法：

```csharp
public class TestScheduler : // ...
{
    public bool IsEnabled { get; private set; }
    public TAbsolute Clock { get; protected set; }
    public void Start()
    public void Stop()
    public void AdvanceTo(long time)
    public void AdvanceBy(long time)
    
    ...
}
```

`TestScheduler` 的運作單位與 [`TimeSpan.Ticks`](https://learn.microsoft.com/en-us/dotnet/api/system.timespan.ticks) 相同。如果你想將時間向前推進 1 秒，可以呼叫 `scheduler.AdvanceBy(TimeSpan.FromSeconds(1).Ticks)`。一個刻度對應於 100 奈秒，因此 1 秒是 10,000,000 個刻度。

### AdvanceTo

`AdvanceTo(long)` 方法將虛擬時間設定為指定的刻度數。這將執行直到指定的絕對時間為止已排程的所有動作。`TestScheduler` 使用刻度作為其時間測量。在此範例中，我們排程立即、在 10 個刻度後以及在 20 個刻度後叫用動作（分別為 1 微秒和 2 微秒）。

```csharp
var scheduler = new TestScheduler();
scheduler.Schedule(() => Console.WriteLine("A")); // Schedule immediately
scheduler.Schedule(TimeSpan.FromTicks(10), () => Console.WriteLine("B"));
scheduler.Schedule(TimeSpan.FromTicks(20), () => Console.WriteLine("C"));

Console.WriteLine("scheduler.AdvanceTo(1);");
scheduler.AdvanceTo(1);

Console.WriteLine("scheduler.AdvanceTo(10);");
scheduler.AdvanceTo(10);

Console.WriteLine("scheduler.AdvanceTo(15);");
scheduler.AdvanceTo(15);

Console.WriteLine("scheduler.AdvanceTo(20);");
scheduler.AdvanceTo(20);
```

輸出：

```
scheduler.AdvanceTo(1);
A
scheduler.AdvanceTo(10);
B
scheduler.AdvanceTo(15);
scheduler.AdvanceTo(20);
C
```

請注意，當我們前進到 15 個刻度時什麼也沒發生。在 15 個刻度之前排程的所有工作都已執行，並且我們尚未前進足夠長的時間來到達下一個排程的動作。

### AdvanceBy
    
`AdvanceBy(long)` 方法允許我們將時鐘向前推進一定的時間量。與 `AdvanceTo` 不同，此處的引數是相對於當前虛擬時間的。同樣，測量單位是刻度。我們可以採用上一個範例並對其進行修改以使用 `AdvanceBy(long)`。

```csharp
var scheduler = new TestScheduler();
scheduler.Schedule(() => Console.WriteLine("A")); // Schedule immediately
scheduler.Schedule(TimeSpan.FromTicks(10), () => Console.WriteLine("B"));
scheduler.Schedule(TimeSpan.FromTicks(20), () => Console.WriteLine("C"));

Console.WriteLine("scheduler.AdvanceBy(1);");
scheduler.AdvanceBy(1);

Console.WriteLine("scheduler.AdvanceBy(9);");
scheduler.AdvanceBy(9);

Console.WriteLine("scheduler.AdvanceBy(5);");
scheduler.AdvanceBy(5);

Console.WriteLine("scheduler.AdvanceBy(5);");
scheduler.AdvanceBy(5);
```

輸出：

```
scheduler.AdvanceBy(1);
A
scheduler.AdvanceBy(9);
B
scheduler.AdvanceBy(5);
scheduler.AdvanceBy(5);
C
```

### Start
    
`TestScheduler` 的 `Start()` 方法執行已排程的所有內容，並根據需要為排入佇列以在特定時間執行的工作項目推進虛擬時間。我們再次採用相同的範例，並將 `AdvanceBy(long)` 呼叫替換為單一 `Start()` 呼叫。

```csharp
var scheduler = new TestScheduler();
scheduler.Schedule(() => Console.WriteLine("A")); // Schedule immediately
scheduler.Schedule(TimeSpan.FromTicks(10), () => Console.WriteLine("B"));
scheduler.Schedule(TimeSpan.FromTicks(20), () => Console.WriteLine("C"));

Console.WriteLine("scheduler.Start();");
scheduler.Start();

Console.WriteLine("scheduler.Clock:{0}", scheduler.Clock);
```

輸出：

```
scheduler.Start();
A
B
C
scheduler.Clock:20
```

請注意，一旦執行了所有已排程的動作，虛擬時鐘就會與我們最後一個排程的項目（20 個刻度）相符。

我們透過排程在呼叫 `Start()` 之後發生的新動作來進一步延伸我們的範例。

```csharp
var scheduler = new TestScheduler();
scheduler.Schedule(() => Console.WriteLine("A"));
scheduler.Schedule(TimeSpan.FromTicks(10), () => Console.WriteLine("B"));
scheduler.Schedule(TimeSpan.FromTicks(20), () => Console.WriteLine("C"));

Console.WriteLine("scheduler.Start();");
scheduler.Start();

Console.WriteLine("scheduler.Clock:{0}", scheduler.Clock);

scheduler.Schedule(() => Console.WriteLine("D"));
```

輸出：

```
scheduler.Start();
A
B
C
scheduler.Clock:20
```

請注意，輸出完全相同；如果我們希望執行第四個動作，我們將必須再次呼叫 `Start()`（或 `AdvanceTo` 或 `AdvanceBy`）。

### Stop

有一個名為 `Stop()` 的方法，其名稱似乎暗示與 `Start()` 具有某種對稱性。這將排程器的 `IsEnabled` 屬性設定為 false，如果 `Start` 當前正在執行，這意味著它將停止檢查佇列以尋找進一步的工作，並將在當前正在處理的工作項目完成後立即返回。

在此範例中，我們展示如何使用 `Stop()` 暫停對已排程動作的處理。

```csharp
var scheduler = new TestScheduler();
scheduler.Schedule(() => Console.WriteLine("A"));
scheduler.Schedule(TimeSpan.FromTicks(10), () => Console.WriteLine("B"));
scheduler.Schedule(TimeSpan.FromTicks(15), scheduler.Stop);
scheduler.Schedule(TimeSpan.FromTicks(20), () => Console.WriteLine("C"));

Console.WriteLine("scheduler.Start();");
scheduler.Start();
Console.WriteLine("scheduler.Clock:{0}", scheduler.Clock);
```

輸出：

```
scheduler.Start();
A
B
scheduler.Clock:15
```

請注意，「C」從未被輸出，因為我們在 15 個刻度時停止了時鐘。

由於 `Start` 在排空工作佇列時會自動停止，因此你沒有義務呼叫 `Stop`。它僅在你想呼叫 `Start` 但在測試過程中暫停處理時才存在。

### 排程衝突 (Schedule collision)

在排程動作時，可能甚至很可能會為同一個時間點排程許多動作。這最常發生在為_現在_排程多個動作時。也可能發生為未來的同一時間點排程多個動作的情況。`TestScheduler` 有一種簡單的方法來處理這個問題。排程動作時，會使用排程的時鐘時間對其進行標記。如果為同一時間點排程了多個項目，則它們將按照排程的順序排入佇列；當時鐘前進時，該時間點的所有項目將按其排程的順序執行。

```csharp
var scheduler = new TestScheduler();
scheduler.Schedule(TimeSpan.FromTicks(10), () => Console.WriteLine("A"));
scheduler.Schedule(TimeSpan.FromTicks(10), () => Console.WriteLine("B"));
scheduler.Schedule(TimeSpan.FromTicks(10), () => Console.WriteLine("C"));

Console.WriteLine("scheduler.Start();");
scheduler.Start();
Console.WriteLine("scheduler.Clock:{0}", scheduler.Clock);
```

輸出：

```
scheduler.AdvanceTo(10);
A
B
C
scheduler.Clock:10
```

請注意，虛擬時鐘位於 10 個刻度，這是我們推進到的時間。

## 測試 Rx 程式碼 (Testing Rx code)

現在我們對 `TestScheduler` 有了一些了解，讓我們檢驗如何使用它來測試我們最初使用 `Interval` 和 `Timeout` 的兩個程式碼片段。我們希望盡快執行測試，但仍保持時間的語意。在此範例中，我們相隔一秒產生五個值，但將我們的 `TestScheduler` 傳入 `Interval` 方法以取代預設排程器。

```csharp
[TestMethod]
public void Testing_with_test_scheduler()
{
    var expectedValues = new long[] {0, 1, 2, 3, 4};
    var actualValues = new List<long>();
    var scheduler = new TestScheduler();

    var interval = Observable.Interval(TimeSpan.FromSeconds(1), scheduler).Take(5);
    
    interval.Subscribe(actualValues.Add);

    scheduler.Start();
    CollectionAssert.AreEqual(expectedValues, actualValues);
    // Executes in less than 0.01s "on my machine"
}
```

雖然這稍微有點意思，但我認為更重要的是我們將如何測試一段真實的程式碼。你可以想像一個訂閱價格串流的 ViewModel。當發布價格時，它會將它們新增到集合中。假設這是一個 WPF 實作，我們冒昧地強制在 `ThreadPool` 上完成訂閱，並在 `Dispatcher` 上執行觀察。

```csharp
public class MyViewModel : IMyViewModel
{
    private readonly IMyModel _myModel;
    private readonly ObservableCollection<decimal> _prices;

    public MyViewModel(IMyModel myModel)
    {
        _myModel = myModel;
        _prices = new ObservableCollection<decimal>();
    }

    public void Show(string symbol)
    {
        // TODO: resource mgt, exception handling etc...
        _myModel.PriceStream(symbol)
                .SubscribeOn(Scheduler.ThreadPool)
                .ObserveOn(Scheduler.Dispatcher)
                .Timeout(TimeSpan.FromSeconds(10), Scheduler.ThreadPool)
                .Subscribe(
                    Prices.Add,
                    ex=>
                        {
                            if(ex is TimeoutException)
                                IsConnected = false;
                        });
        IsConnected = true;
    }

    public ObservableCollection<decimal> Prices
    {
        get { return _prices; }
    }

    public bool IsConnected { get; private set; }
}
```

### 插入排程器相依性 (Injecting scheduler dependencies)

雖然上面的程式碼片段可能可以滿足我們的需求，但它將難以測試，因為它是透過靜態屬性存取排程器的。你將需要某種方法使測試能夠在測試期間提供不同的排程器。在此範例中，我們將為此目的定義一個介面：

```csharp
public interface ISchedulerProvider
{
    IScheduler CurrentThread { get; }
    IScheduler Dispatcher { get; }
    IScheduler Immediate { get; }
    IScheduler NewThread { get; }
    IScheduler ThreadPool { get; }
    IScheduler TaskPool { get; } 
}
```

我們將在正式環境中執行的預設實作如下：

```csharp
public sealed class SchedulerProvider : ISchedulerProvider
{
    public IScheduler CurrentThread => Scheduler.CurrentThread;
    public IScheduler Dispatcher => DispatcherScheduler.Instance;
    public IScheduler Immediate => Scheduler.Immediate;
    public IScheduler NewThread => Scheduler.NewThread;
    public IScheduler ThreadPool => Scheduler.ThreadPool;
    public IScheduler TaskPool => Scheduler.TaskPool;
}
```

我們可以替換 `ISchedulerProvider` 的實作來協助測試。例如：

```csharp
public sealed class TestSchedulers : ISchedulerProvider
{
    // Schedulers available as TestScheduler type
    public TestScheduler CurrentThread { get; }  = new TestScheduler();
    public TestScheduler Dispatcher { get; }  = new TestScheduler();
    public TestScheduler Immediate { get; }  = new TestScheduler();
    public TestScheduler NewThread { get; }  = new TestScheduler();
    public TestScheduler ThreadPool { get; }  = new TestScheduler();
    
    // ISchedulerService needs us to return IScheduler, but we want the properties
    // to return TestScheduler for the convenience of test code, so we provide
    // explicit implementations of all the properties to match ISchedulerService.
    IScheduler ISchedulerProvider.CurrentThread => CurrentThread;
    IScheduler ISchedulerProvider.Dispatcher => Dispatcher;
    IScheduler ISchedulerProvider.Immediate => Immediate;
    IScheduler ISchedulerProvider.NewThread => NewThread;
    IScheduler ISchedulerProvider.ThreadPool => ThreadPool;
}
```

請注意，`ISchedulerProvider` 是明確實作的，因為該介面要求每個屬性都返回 `IScheduler`，但我們的測試將需要直接存取 `TestScheduler` 執行個體。我現在可以為我的 ViewModel 編寫一些測試。下面，我們測試了 `MyViewModel` 類別的修改版本，該類別接受一個 `ISchedulerProvider` 並使用它來代替來自 `Scheduler` 類別的靜態排程器。我們還使用流行的 [Moq](https://github.com/Moq) 架構來提供模型的合適虛擬實作。

```csharp
[TestInitialize]
public void SetUp()
{
    _myModelMock = new Mock<IMyModel>();
    _schedulerProvider = new TestSchedulers();
    _viewModel = new MyViewModel(_myModelMock.Object, _schedulerProvider);
}

[TestMethod]
public void Should_add_to_Prices_when_Model_publishes_price()
{
    decimal expected = 1.23m;
    var priceStream = new Subject<decimal>();
    _myModelMock.Setup(svc => svc.PriceStream(It.IsAny<string>())).Returns(priceStream);

    _viewModel.Show("SomeSymbol");
    
    // Schedule the OnNext
    _schedulerProvider.ThreadPool.Schedule(() => priceStream.OnNext(expected));  

    Assert.AreEqual(0, _viewModel.Prices.Count);

    // Execute the OnNext action
    _schedulerProvider.ThreadPool.AdvanceBy(1);  
    Assert.AreEqual(0, _viewModel.Prices.Count);
    
    // Execute the OnNext handler
    _schedulerProvider.Dispatcher.AdvanceBy(1);  
    Assert.AreEqual(1, _viewModel.Prices.Count);
    Assert.AreEqual(expected, _viewModel.Prices.First());
}

[TestMethod]
public void Should_disconnect_if_no_prices_for_10_seconds()
{
    var timeoutPeriod = TimeSpan.FromSeconds(10);
    var priceStream = Observable.Never<decimal>();
    _myModelMock.Setup(svc => svc.PriceStream(It.IsAny<string>())).Returns(priceStream);

    _viewModel.Show("SomeSymbol");

    _schedulerProvider.ThreadPool.AdvanceBy(timeoutPeriod.Ticks - 1);
    Assert.IsTrue(_viewModel.IsConnected);
    _schedulerProvider.ThreadPool.AdvanceBy(timeoutPeriod.Ticks);
    Assert.IsFalse(_viewModel.IsConnected);
}
```

輸出：

```
2 passed, 0 failed, 0 skipped, took 0.41 seconds (MSTest 10.0).
```

這兩個測試確保了五件事：

* 當模型產生價格時，`Price` 屬性會向其新增價格
* 在 ThreadPool 上訂閱該序列
* `Price` 屬性在 Dispatcher 上更新，即在 Dispatcher 上觀察該序列
* 價格之間 10 秒的逾時會將 ViewModel 設定為中斷連線
* 測試執行快速。
  
雖然執行測試的時間並沒有那麼令人印象深刻，但該時間的大部分似乎都花在了預熱我的測試控管上。此外，將測試數量增加到 10 個僅增加 0.03 秒。一般來說，現代 CPU 應該能夠每秒執行數千個單元測試。

在第一個測試中，我們可以看到只有在 `ThreadPool` 和 `Dispatcher` 排程器都執行之後，我們才會得到結果。在第二個測試中，它有助於驗證逾時不少於 10 秒。

在某些情境下，你對排程器不感興趣，並且希望將測試重點放在其他功能上。如果是這種情況，那麼你可能希望建立 `ISchedulerProvider` 的另一個測試實作，該實作對其所有成員返回 `ImmediateScheduler`。這有助於減少測試中的雜訊。

```csharp
public sealed class ImmediateSchedulers : ISchedulerService
{
    public IScheduler CurrentThread => Scheduler.Immediate;
    public IScheduler Dispatcher => Scheduler.Immediate;
    public IScheduler Immediate => Scheduler.Immediate;
    public IScheduler NewThread => Scheduler.Immediate;
    public IScheduler ThreadPool => Scheduler.Immediate;
}
```

## 高級功能 - ITestableObserver (Advanced features - ITestableObserver)

`TestScheduler` 提供了進一步的高級功能。當測試設定的某些部分需要在特定的虛擬時間執行時，這些功能可能會很有用。

### `Start(Func<IObservable<T>>)`

`Start` 有三個多載，用於在給定時間啟動可觀察序列，記錄其進行的通知，並在給定時間處置訂閱。起初這可能會令人困惑，因為 `Start` 的無參數多載是完全無關的。這三個多載返回一個 `ITestableObserver<T>`，它允許你記錄來自可觀察序列的通知，就像我們在[轉換章節](06_Transformation.md#materialize-and-dematerialize)中看到的 `Materialize` 方法一樣。

```csharp
public interface ITestableObserver<T> : IObserver<T>
{
    // Gets recorded notifications received by the observer.
    IList<Recorded<Notification<T>>> Messages { get; }
}
```

雖然有三個多載，但我們將首先檢驗最特定的多載。此多載採用四個參數：

* 可觀察序列 Factory 委派
* 叫用 Factory 的時間點
* 訂閱從 Factory 返回之可觀察序列的時間點
* 處置訂閱的時間點

最後三個參數的_時間_以刻度為單位進行測量，與 `TestScheduler` 的其餘成員一樣。

```csharp
public ITestableObserver<T> Start<T>(
    Func<IObservable<T>> create, 
    long created, 
    long subscribed, 
    long disposed)
{...}
```

我們可以使用此方法來測試 `Observable.Interval` Factory 方法。在此，我們建立了一個每秒產生一個值並持續 4 秒的可觀察序列。我們使用 `TestScheduler.Start` 方法立即建立並訂閱它（透過為第二個和第三個參數傳遞 0）。我們在 5 秒後處置我們的訂閱。一旦 `Start` 方法執行完畢，我們就會輸出我們所記錄的內容。

```csharp
var scheduler = new TestScheduler();
var source = Observable.Interval(TimeSpan.FromSeconds(1), scheduler)
    .Take(4);

var testObserver = scheduler.Start(
    () => source, 
    0, 
    0, 
    TimeSpan.FromSeconds(5).Ticks);

Console.WriteLine("Time is {0} ticks", scheduler.Clock);
Console.WriteLine("Received {0} notifications", testObserver.Messages.Count);

foreach (Recorded<Notification<long>> message in testObserver.Messages)
{
    Console.WriteLine("{0} @ {1}", message.Value, message.Time);
}
```

輸出：

```
Time is 50000000 ticks
Received 5 notifications
OnNext(0) @ 10000001
OnNext(1) @ 20000001
OnNext(2) @ 30000001
OnNext(3) @ 40000001
OnCompleted() @ 40000001
```

請注意，`ITestObserver<T>` 記錄 `OnNext` 和 `OnCompleted` 通知。如果序列以錯誤終止，`ITestObserver<T>` 將改為記錄 `OnError` 通知。

我們可以嘗試調整輸入變數以查看其產生的影響。我們知道 `Observable.Interval` 方法是一個冷可觀察序列 (Cold Observable)，因此建立的虛擬時間無關緊要。變更訂閱的虛擬時間可以變更我們的結果。如果我們將其變更為 2 秒，我們將注意到如果我們將處置時間保留為 5 秒，我們將錯過一些訊息。

```csharp
var testObserver = scheduler.Start(
    () => Observable.Interval(TimeSpan.FromSeconds(1), scheduler).Take(4), 
    0,
    TimeSpan.FromSeconds(2).Ticks,
    TimeSpan.FromSeconds(5).Ticks);
```

輸出：

```
Time is 50000000 ticks
Received 2 notifications
OnNext(0) @ 30000000
OnNext(1) @ 40000000
```

我們在 2 秒時開始訂閱；`Interval` 在每秒後產生值（即第 3 秒和第 4 秒），我們在第 5 秒處置。因此我們錯過了其他兩個 `OnNext` 訊息以及 `OnCompleted` 訊息。

此 `TestScheduler.Start` 方法還有另外兩個多載。

```csharp
public ITestableObserver<T> Start<T>(Func<IObservable<T>> create, long disposed)
{
    if (create == null)
    {
        throw new ArgumentNullException("create");
    }
    else
    {
        return this.Start<T>(create, 100L, 200L, disposed);
    }
}

public ITestableObserver<T> Start<T>(Func<IObservable<T>> create)
{
    if (create == null)
    {
        throw new ArgumentNullException("create");
    }
    else
    {
        return this.Start<T>(create, 100L, 200L, 1000L);
    }
}
```

如你所見，這些多載只是呼叫我們一直在檢驗的變體，但傳遞了一些預設值。這些預設值在建立之前以及在建立與訂閱之間提供了簡短的間隙，從而留出足夠的空間來設定在它們之間發生的其他事情。然後處置發生在稍後，允許事物執行更長時間。這些預設值並沒有特別神奇之處，但是如果你重視沒有雜亂而不是完全清楚何時發生什麼，並且樂於依賴慣例的無形效果，那麼你可能更喜歡這個。Rx 原始程式碼本身包含數千個測試，其中非常多的測試使用最簡單的 `Start` 多載，在程式碼庫中日復一日工作的開發人員很快就習慣了建立發生在時間 100、訂閱發生在時間 200、並且你在 1000 之前測試所需的一切的想法。

### CreateColdObservable

就像我們可以記錄可觀察序列一樣，我們也可以使用 `CreateColdObservable` 來重播一組 `Recorded<Notification<int>>`。`CreateColdObservable` 的簽章僅採用記錄通知的 `params` 陣列。

```csharp
// Creates a cold observable from an array of notifications.
// Returns a cold observable exhibiting the specified message behavior.
public ITestableObservable<T> CreateColdObservable<T>(
    params Recorded<Notification<T>>[] messages)
{...}
```

`CreateColdObservable` 返回一個 `ITestableObservable<T>`。此介面透過公開「訂閱」清單和它將產生的訊息清單來延伸 `IObservable<T>`。

```csharp
public interface ITestableObservable<T> : IObservable<T>
{
    // Gets the subscriptions to the observable.
    IList<Subscription> Subscriptions { get; }

    // Gets the recorded notifications sent by the observable.
    IList<Recorded<Notification<T>>> Messages { get; }
}
```

使用 `CreateColdObservable`，我們可以模擬我們之前的 `Observable.Interval` 測試。

```csharp
var scheduler = new TestScheduler();
var source = scheduler.CreateColdObservable(
    new Recorded<Notification<long>>(10000000, Notification.CreateOnNext(0L)),
    new Recorded<Notification<long>>(20000000, Notification.CreateOnNext(1L)),
    new Recorded<Notification<long>>(30000000, Notification.CreateOnNext(2L)),
    new Recorded<Notification<long>>(40000000, Notification.CreateOnNext(3L)),
    new Recorded<Notification<long>>(40000000, Notification.CreateOnCompleted<long>())
    );

var testObserver = scheduler.Start(
    () => source,
    0,
    0,
    TimeSpan.FromSeconds(5).Ticks);

Console.WriteLine("Time is {0} ticks", scheduler.Clock);
Console.WriteLine("Received {0} notifications", testObserver.Messages.Count);

foreach (Recorded<Notification<long>> message in testObserver.Messages)
{
    Console.WriteLine("  {0} @ {1}", message.Value, message.Time);
}
```

輸出：

```
Time is 50000000 ticks
Received 5 notifications
OnNext(0) @ 10000001
OnNext(1) @ 20000001
OnNext(2) @ 30000001
OnNext(3) @ 40000001
OnCompleted() @ 40000001
```

請注意，我們的輸出與前面帶有 `Observable.Interval` 的範例完全相同。

### CreateHotObservable

我們還可以使用 `CreateHotObservable` 方法建立熱測試可觀察序列。它具有與 `CreateColdObservable` 相同的參數和傳回值；不同之處在於，現在為每個訊息指定的虛擬時間是相對於建立可觀察序列的時間，而不是像 `CreateColdObservable` 方法那樣相對於訂閱該序列的時間。

此範例就是最後一個「冷」範例，但改為建立一個熱可觀察序列。

```csharp
var scheduler = new TestScheduler();
var source = scheduler.CreateHotObservable(
    new Recorded<Notification<long>>(10000000, Notification.CreateOnNext(0L)),
// ...    
```

輸出：

```
Time is 50000000 ticks
Received 5 notifications
OnNext(0) @ 10000000
OnNext(1) @ 20000000
OnNext(2) @ 30000000
OnNext(3) @ 40000000
OnCompleted() @ 40000000
```

請注意，輸出幾乎相同。建立和訂閱的排程不會影響熱可觀察序列，因此通知發生時間比冷對應項早 1 個刻度。

我們可以透過將虛擬建立時間和虛擬訂閱時間變更為不同值來查看熱可觀察序列所具有的主要差異。對於冷可觀察序列，虛擬建立時間沒有實際影響，因為訂閱是啟動任何動作的關鍵。這意味著我們不會錯過冷可觀察序列上的任何早期訊息。對於熱可觀察序列，如果我們訂閱得太晚，我們可能會錯過訊息。在此，我們立即建立熱可觀察序列，但僅在 1 秒後訂閱它（從而錯過了第一條訊息）。

```csharp
var scheduler = new TestScheduler();
var source = scheduler.CreateHotObservable(
    new Recorded<Notification<long>>(10000000, Notification.CreateOnNext(0L)),
    new Recorded<Notification<long>>(20000000, Notification.CreateOnNext(1L)),
    new Recorded<Notification<long>>(30000000, Notification.CreateOnNext(2L)),
    new Recorded<Notification<long>>(40000000, Notification.CreateOnNext(3L)),
    new Recorded<Notification<long>>(40000000, Notification.CreateOnCompleted<long>())
    );

var testObserver = scheduler.Start(
    () => source,
    0,
    TimeSpan.FromSeconds(1).Ticks,
    TimeSpan.FromSeconds(5).Ticks);

Console.WriteLine("Time is {0} ticks", scheduler.Clock);
Console.WriteLine("Received {0} notifications", testObserver.Messages.Count);

foreach (Recorded<Notification<long>> message in testObserver.Messages)
{
    Console.WriteLine("  {0} @ {1}", message.Value, message.Time);
}
```

輸出：

```
Time is 50000000 ticks
Received 4 notifications
OnNext(1) @ 20000000
OnNext(2) @ 30000000
OnNext(3) @ 40000000
OnCompleted() @ 40000000
```

### CreateObserver 

最後，如果你不想使用 `TestScheduler.Start` 方法，並且需要對觀察者進行更細緻的控制，則可以使用 `TestScheduler.CreateObserver()`。這將返回一個 `ITestObserver`，你可以使用它來管理對可觀察序列的訂閱。此外，你仍然可以接觸到所記錄的訊息和任何訂閱者。

當前的產業標準要求廣泛覆蓋自動化單元測試，以滿足品質保證標準。然而，並行程式設計通常是一個難以妥善測試的領域。Rx 提供了經過深思熟慮的測試功能實作，允許進行具確定性且高輸送量的測試。`TestScheduler` 提供了控制虛擬時間和產生用於測試的可觀察序列的方法。這種輕鬆且可靠地測試並行系統的能力使 Rx 脫穎而出。
