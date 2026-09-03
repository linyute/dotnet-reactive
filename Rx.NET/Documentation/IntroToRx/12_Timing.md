# 基於時間的序列 (Time-based sequences)

對於事件來源，時間點通常很重要。在某些情況下，關於某個事件唯一感興趣的資訊可能是它發生的時間。核心的 `IObservable<T>` 和 `IObserver<T>` 介面在其方法簽章中完全沒有提及時間，但它們不需要提及，因為來源可以決定何時呼叫觀察者的 `OnNext` 方法。訂閱者知道事件何時發生，因為它正在當下發生。這並不總是以時間進行操作的最便利方式，因此 Rx 函式庫提供了一些與時間相關的運算子。我們已經看過幾個提供可選的基於時間之操作的運算子：[`Buffer`](08_Partitioning.md#buffer) 和 [`Window`](08_Partitioning.md#window)。本章檢驗完全關於時間的各種運算子。

## Timestamp 與 TimeInterval

由於可觀察序列是非同步的，因此知道何時接收到元素會很方便。顯然，訂閱者總是可以直接使用 `DateTimeOffset.Now`，但是如果你想將到達時間作為較大查詢的一部分來參考，`Timestamp` 擴充方法是一個方便的便利方法，它可以為每個元素附加時間戳記。它將來自其來源序列的元素包裝在輕量級 `Timestamped<T>` 結構中。`Timestamped<T>` 類型是一個結構 (struct)，它公開了其包裝的元素的值，並且還公開了一個 `DateTimeOffset`，指示 `Timestamp` 運算子何時接收到它。

在此範例中，我們建立了相隔一秒的三個值的序列，然後將其轉換為帶有時間戳記的序列。

```csharp
Observable.Interval(TimeSpan.FromSeconds(1))
          .Take(3)
          .Timestamp()
          .Dump("Timestamp");
```

如你所見，`Timestamped<T>` 對 `ToString()` 的實作給了我們可讀的輸出。

```
Timestamp-->0@07/08/2023 10:03:58 +00:00
Timestamp-->1@07/08/2023 10:03:59 +00:00
Timestamp-->2@07/08/2023 10:04:00 +00:00
TimeStamp completed
```

我們可以看到值 0、1 和 2 各自相隔一秒產生。

Rx 還提供了 `TimeInterval`。它不報告項目到達的時間，而是報告項目之間的時間間隔（或者在第一個元素的情況下，報告在訂閱後需要多長時間才會出現）。與 `Timestamp` 方法類似，元素被包裝在輕量級結構中。但是，`Timestamped<T>` 為每個項目添加了到達時間，而 `TimeInterval` 則使用新增了 `TimeSpan` 的 `TimeInterval<T>` 類型包裝每個元素。我們可以修改前面的範例以使用 `TimeInterval`：

```csharp
Observable.Interval(TimeSpan.FromSeconds(1))
          .Take(3)
          .TimeInterval()
          .Dump("TimeInterval");
```

如你所見，輸出現在報告元素之間的時間，而不是接收它們的當天時間：

```
Timestamp-->0@00:00:01.0183771
Timestamp-->1@00:00:00.9965679
Timestamp-->2@00:00:00.9908958
Timestamp completed
```

正如你從輸出中看到的，時間點並不完全是一秒，但非常接近。其中一些將是 `TimeInterval` 運算子中的測量雜訊，但這種變化的大部分可能來自 `Observable.Interval` 類別。排程器能夠滿足其時間請求的精度總是有極限的。某些排程器比其他排程器引入更多的變化。透過 UI 執行緒傳遞工作的排程器最終受到該執行緒訊息迴圈回應速度的限制。但即使在最有利的條件下，排程器也受到 .NET 並非為即時系統而建構的事實（Rx 可以使用的大多數作業系統也不是為此而建構的）的限制。因此，對於本節中的所有運算子，你應該意識到在 Rx 中計時始終是一件_盡力而為 (best effort)_ 的事情。

事實上，時間點固有的變化可以使 `Timestamp` 特別有用。單純查看 `DateTimeOffset.Now` 的問題在於處理事件需要非零的時間量，因此在處理一個事件期間每次嘗試讀取當前時間時，你可能會看到稍微不同的時間。透過附加一次時間戳記，我們可以擷取觀察到事件的時間，然後下游處理增加多少延遲都無關緊要了。該事件將標註一個單一且固定的時間，指示其通過 `Timestamp` 的時間。

## Delay

`Delay` 擴充方法會對整個序列進行時間平移。`Delay` 嘗試保留值之間的相對時間間隔。它能做到這一點的精度不可避免地存在極限——它不會精確到最接近的奈秒來重新建立時間點。確切的精度取決於你使用的排程器，並且在重負載下通常會變差，但它通常會在幾毫秒內重現時間點。

`Delay` 的多載提供了各種指定時間平移的不同方法。（在所有選項中，你都可以選擇傳遞排程器，但是如果你呼叫不接受排程器的多載，它會預設為 [`DefaultScheduler`](11_SchedulingAndThreading.md#defaultscheduler)。）最直接的方法是傳遞 `TimeSpan`，這會將序列延遲指定的量。還有一些接受 `DateTimeOffset` 的延遲，它們將等待直到指定的時間發生，然後開始重播輸入。（這第二種基於絕對時間的方法基本上等同於 `TimeSpan` 多載。你可以透過從目標時間中減去當前時間以獲取 `TimeSpan` 來獲得大致相同的效果，只不過 `DateTimeOffset` 版本嘗試處理在呼叫 `Delay` 到指定時間到達之間發生的系統時鐘變更。）

為了展示 `Delay` 方法的實際運作，此範例建立了相隔一秒的值序列並對其進行時間標記。這將表明被延遲的不是訂閱，而是向我們的最終訂閱者實際轉發通知的動作。

```csharp
IObservable<Timestamped<long>> source = Observable
    .Interval(TimeSpan.FromSeconds(1))
    .Take(5)
    .Timestamp();

IObservable<Timestamped<long>> delay = source.Delay(TimeSpan.FromSeconds(2));

delay.Subscribe(value => 
   Console.WriteLine(
     $"Item {value.Value} with timestamp {value.Timestamp} received at {DateTimeOffset.Now}"),
   () => Console.WriteLine("delay Completed"));
```

如果你檢視輸出中的時間戳記，可以看到 `Timestamp` 擷取的時間都比訂閱報告的時間早兩秒：

```
Item 0 with timestamp 09/11/2023 17:32:20 +00:00 received at 09/11/2023 17:32:22 +00:00
Item 1 with timestamp 09/11/2023 17:32:21 +00:00 received at 09/11/2023 17:32:23 +00:00
Item 2 with timestamp 09/11/2023 17:32:22 +00:00 received at 09/11/2023 17:32:24 +00:00
Item 3 with timestamp 09/11/2023 17:32:23 +00:00 received at 09/11/2023 17:32:25 +00:00
Item 4 with timestamp 09/11/2023 17:32:24 +00:00 received at 09/11/2023 17:32:26 +00:00
delay Completed
```

請注意，`Delay` 不會對 `OnError` 通知進行時間平移。這些將立即傳播。

## Sample

`Sample` 方法以你要求的任何間隔產生項目。每次產生值時，它都會報告來源中出現的最後一個值。如果你的來源產生資料的速率高於你的需求（例如，假設你有一個每秒報告 100 次測量值的加速度計，但你只需要每秒讀取 10 次），`Sample` 提供了一種降低資料速率的簡單方法。此範例展示了 `Sample` 的實際運作。

```csharp
IObservable<long> interval = Observable.Interval(TimeSpan.FromMilliseconds(150));
interval.Sample(TimeSpan.FromSeconds(1)).Subscribe(Console.WriteLine);
```

輸出：

```
5
12
18
```

如果你仔細觀察這些數字，可能會注意到值之間的間隔每次都不相同。我選擇了 150 毫秒的來源間隔和 1 秒的取樣間隔，以強調取樣的一個可能需要仔細處理的面向：如果來源產生項目的速率與取樣速率沒有精確對齊，這可能意味著 `Sample` 引入了來源中不存在的不規則性。如果我們列出底層序列產生值的時間，以及 `Sample` 獲取每個值的時間，我們可以看到在這些特定時間點下，取樣間隔僅每 3 秒與來源時間點對齊一次。

| 相對時間 (ms) | 來源值 | 取樣值 |
| :----------------- | :----------- | :------------ |
| 0                  |              |               |
| 50                 |              |               |
| 100                |              |               |
| 150                | 0            |               |
| 200                |              |               |
| 250                |              |               |
| 300                | 1            |               |
| 350                |              |               |
| 400                |              |               |
| 450                | 2            |               |
| 500                |              |               |
| 550                |              |               |
| 600                | 3            |               |
| 650                |              |               |
| 700                |              |               |
| 750                | 4            |               |
| 800                |              |               |
| 850                |              |               |
| 900                | 5            |               |
| 950                |              |               |
| 1000               |              | 5             |
| 1050               | 6            |               |
| 1100               |              |               |
| 1150               |              |               |
| 1200               | 7            |               |
| 1250               |              |               |
| 1300               |              |               |
| 1350               | 8            |               |
| 1400               |              |               |
| 1450               |              |               |
| 1500               | 9            |               |
| 1550               |              |               |
| 1600               |              |               |
| 1650               | 10           |               |
| 1700               |              |               |
| 1750               |              |               |
| 1800               | 11           |               |
| 1850               |              |               |
| 1900               |              |               |
| 1950               | 12           |               |
| 2000               |              | 12            |
| 2050               |              |               |
| 2100               | 13           |               |
| 2150               |              |               |
| 2200               |              |               |
| 2250               | 14           |               |
| 2300               |              |               |
| 2350               |              |               |
| 2400               | 15           |               |
| 2450               |              |               |
| 2500               |              |               |
| 2550               | 16           |               |
| 2600               |              |               |
| 2650               |              |               |
| 2700               | 17           |               |
| 2750               |              |               |
| 2800               |              |               |
| 2850               | 18           |               |
| 2900               |              |               |
| 2950               |              |               |
| 3000               | 19           | 19            |

由於第一次取樣是在來源發出 5 之後、且在產生 6 之前的間隙的三分之二處進行的，因此在某種意義上「正確」的當前值大約是 5.67，但 `Sample` 不會嘗試任何此類插值。它只會報告來源中出現的最後一個值。一個相關的結果是，如果取樣間隔足夠短，以至於你要求 `Sample` 報告值的速度快於它們從來源中出現的速度，它只會重複輸出值。

## Throttle

`Throttle` 擴充方法針對以可變速率產生值且有時過快的序列提供了一種保護。與 `Sample` 方法類似，`Throttle` 將返回一段時間內的最後一個取樣值。然而，與 `Sample` 不同的是，`Throttle` 的週期是一個滑動視窗。每次 `Throttle` 接收到一個值時，視窗都會重設。只有在該時間段過去之後，最後一個值才會被傳播。這意味著 `Throttle` 方法僅對以可變速率產生值的序列有用。以恆定速率產生值的序列（例如 `Interval` 或 `Timer`），如果它們產生值的速度快於節流週期，則其所有值都將被抑制；而如果它們產生值的速度慢於節流週期，則其所有值都將被傳播。

```csharp
// Ignores values from an observable sequence which 
// are followed by another value before dueTime.
public static IObservable<TSource> Throttle<TSource>(
    this IObservable<TSource> source, 
    TimeSpan dueTime)
{...}
public static IObservable<TSource> Throttle<TSource>(
    this IObservable<TSource> source, 
    TimeSpan dueTime, 
    IScheduler scheduler)
{...}
```

我們可以套用 `Throttle` 來使用即時搜尋功能，該功能可在你輸入時提供建議。我們通常希望等到使用者暫停輸入片刻後再搜尋建議，因為否則，我們可能會連續啟動多個搜尋，並在使用者每次按下另一個按鍵時取消上一個搜尋。只有在出現暫停時，我們才可以使用他們到目前為止鍵入的內容執行搜尋。`Throttle` 非常適合這種情境，因為如果來源產生值的速度快於指定的速率，它根本不允許任何事件通過。

請注意，RxJS 函式庫決定讓其版本的 throttle 以不同的方式運作，因此如果你發現自己同時使用 Rx.NET 和 RxJS，請注意它們的工作方式並不相同。在 RxJS 中，當來源超過指定速率時，throttle 並不會完全關閉：它只會捨棄足夠的項目，以使輸出永遠不會超過指定速率。因此 RxJS 的 throttle 實作是一種速率限制器 (rate limiter)，而 Rx.NET 的 `Throttle` 更像是一個自動重設的斷路器 (circuit breaker)，在過載期間完全關閉。

## Timeout

`Timeout` 運算子方法允許我們在來源在給定時間段內未產生任何通知時以錯誤終止序列。我們可以透過 `TimeSpan` 將時間段指定為滑動視窗，或者透過提供 `DateTimeOffset` 將時間段指定為序列必須完成的絕對時間。

```csharp
// Returns either the observable sequence or a TimeoutException
// if the maximum duration between values elapses.
public static IObservable<TSource> Timeout<TSource>(
    this IObservable<TSource> source, 
    TimeSpan dueTime)
{...}
public static IObservable<TSource> Timeout<TSource>(
    this IObservable<TSource> source, 
    TimeSpan dueTime, 
    IScheduler scheduler)
{...}

// Returns either the observable sequence or a  
// TimeoutException if dueTime elapses.
public static IObservable<TSource> Timeout<TSource>(
    this IObservable<TSource> source, 
    DateTimeOffset dueTime)
{...}
public static IObservable<TSource> Timeout<TSource>(
    this IObservable<TSource> source, 
    DateTimeOffset dueTime, 
    IScheduler scheduler)
{...}
```

如果我們提供 `TimeSpan` 並且在該時間跨度內沒有產生任何值，則序列會失敗並擲回 `TimeoutException`。

```csharp
var source = Observable.Interval(TimeSpan.FromMilliseconds(100))
                       .Take(5)
                       .Concat(Observable.Interval(TimeSpan.FromSeconds(2)));

var timeout = source.Timeout(TimeSpan.FromSeconds(1));
timeout.Subscribe(
    Console.WriteLine, 
    Console.WriteLine, 
    () => Console.WriteLine("Completed"));
```

最初這會以足夠頻繁的頻率產生值以滿足 `Timeout`，因此由 `Timeout` 返回的可觀察序列只會轉發來自來源的項目。但是一旦來源停止產生項目，我們就會得到一個 OnError：

```
0
1
2
3
4
System.TimeoutException: The operation has timed out.
```

或者，我們可以向 `Timeout` 傳遞一個絕對時間；如果序列未在該時間之前完成，則會產生錯誤。

```csharp
var dueDate = DateTimeOffset.UtcNow.AddSeconds(4);
var source = Observable.Interval(TimeSpan.FromSeconds(1));
var timeout = source.Timeout(dueDate);
timeout.Subscribe(
    Console.WriteLine, 
    Console.WriteLine, 
    () => Console.WriteLine("Completed"));
```

輸出：

```
0
1
2
System.TimeoutException: The operation has timed out.
```

還有其他 `Timeout` 多載使我們能夠在發生逾時時替換為替代序列。

```csharp
// Returns the source observable sequence or the other observable 
// sequence if the maximum duration between values elapses.
public static IObservable<TSource> Timeout<TSource>(
    this IObservable<TSource> source, 
    TimeSpan dueTime, 
    IObservable<TSource> other)
{...}

public static IObservable<TSource> Timeout<TSource>(
    this IObservable<TSource> source, 
    TimeSpan dueTime, 
    IObservable<TSource> other, 
    IScheduler scheduler)
{...}

// Returns the source observable sequence or the 
// other observable sequence if dueTime elapses.
public static IObservable<TSource> Timeout<TSource>(
    this IObservable<TSource> source, 
    DateTimeOffset dueTime, 
    IObservable<TSource> other)
{...}  

public static IObservable<TSource> Timeout<TSource>(
    this IObservable<TSource> source, 
    DateTimeOffset dueTime, 
    IObservable<TSource> other, 
    IScheduler scheduler)
{...}
```

正如我們現在所見，Rx 提供了在反應式範式中管理時間點的功能。可以根據你的需求對資料進行計時、節流或取樣。可以使用延遲功能在時間上平移整個序列，並且可以使用 `Timeout` 運算子來斷言資料的及時性。

接下來我們將探討 Rx 與外界之間的邊界。
