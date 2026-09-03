# 篩選

Rx 為我們提供了工具，用以接收潛在的大量事件並對其進行處理以產生更高層次的見解。這通常涉及數量的減少。如果較低數量串流中的個別事件平均而言提供更多資訊，那麼少量的事件可能比大量的事件更有用。實現這一點的最簡單機制就是單純地篩選掉我們不需要的事件。Rx 定義了幾個可以做到這一點的運算子。

在繼續介紹新運算子之前，我們將快速定義一個擴充方法來幫助說明幾個範例。這個 `Dump` 擴充方法會訂閱任何 `IObservable<T>`，並使用處理常式顯示來源產生的每個通知的訊息。此方法接受一個 `name` 引數，該引數將作為每個訊息的一部分顯示，使我們能夠在訂閱多個來源的範例中查看事件來自何處。

```csharp
public static class SampleExtensions
{
    public static void Dump<T>(this IObservable<T> source, string name)
    {
        source.Subscribe(
            value =>Console.WriteLine($"{name}-->{value}"), 
            ex => Console.WriteLine($"{name} failed-->{ex.Message}"),
            () => Console.WriteLine($"{name} completed"));
    }
}
```

## Where

將篩選條件套用至序列是一項極其常見的練習，而 LINQ 中最直接的篩選器就是 `Where` 運算子。就像 LINQ 的一貫作法一樣，Rx 以擴充方法的形式提供其運算子。如果你已經熟悉 LINQ，Rx 的 `Where` 方法簽名將不會讓你感到意外：

```csharp
IObservable<T> Where<T>(this IObservable<T> source, Func<T, bool> predicate)
```

請注意，`source` 參數的元素型別與返回型別相同。這是因為 `Where` 不會修改元素。它可以篩選掉某些元素，但未移除的元素會原封不動地傳遞下去。

此範例使用 `Where` 篩選掉從 `Range` 序列產生的所有奇數值，這意味著只有偶數會出現。

```csharp
IObservable<int> xs = Observable.Range(0, 10); // 數字 0-9
IObservable<int> evenNumbers = xs.Where(i => i % 2 == 0);

evenNumbers.Dump("Where");
```

輸出：

```
Where-->0
Where-->2
Where-->4
Where-->6
Where-->8
Where completed
```

`Where` 運算子是你將在所有 LINQ 提供者中找到的眾多標準 LINQ 運算子之一。例如，`IEnumerable<T>` 實作 LINQ to Objects 提供了一個等價的方法。在大多數情況下，Rx 的運算子行為就像它們在 `IEnumerable<T>` 實作中一樣，儘管正如我們稍後將看到的有一些例外。我們將討論每個實作並隨時解釋任何差異。透過實作這些通用運算子，Rx 還透過 C# 查詢運算式語法免費獲得語言支援。例如，我們可以這樣編寫第一條陳述式，它將編譯為實際上完全相同的程式碼：

```csharp
IObservable<int> evenNumbers =
    from i in xs
    where i % 2 == 0
    select i;
```

本書中的範例大多使用擴充方法而非查詢運算式，部分原因是 Rx 實作了一些沒有對應查詢語法的運算子，部分原因是方法呼叫方式有時可以更容易看清楚正在發生的事情。

與大多數 Rx 運算子一樣，`Where` 不會立即訂閱其來源。（Rx LINQ 運算子非常類似於 LINQ to Objects 中的運算子：`Where` 的 `IEnumerable<T>` 版本在返回時不會嘗試列舉其來源。只有當有項目嘗試列舉 `Where` 返回的 `IEnumerable<T>` 時，它才會接著開始列舉來源。）只有當有項目在 `Where` 返回的 `IObservable<T>` 上呼叫 `Subscribe` 時，它才會在其來源上呼叫 `Subscribe`。並且對於每次這樣的 `Subscribe` 呼叫，它都會執行一次。更廣泛地說，當你將 LINQ 運算子鏈結在一起時，在產生的 `IObservable<T>` 上的每個 `Subscribe` 呼叫都會導致沿著鏈結一路向下進行級聯式的 `Subscribe` 呼叫。

這種級聯 `Subscribe` 的副作用是 `Where`（與大多數其他 LINQ 運算子一樣）本質上既不是_熱_的也不是_冷_的：因為它只是訂閱其來源，因此如果其來源是熱的，它就是熱的，如果其來源是冷的，它就是冷的。

`Where` 運算子傳遞其 `predicate` 回呼返回 `true` 的所有元素。更準確地說，當你訂閱 `Where` 時，它將建立自己的 `IObserver<T>`，並將其作為引數傳遞給 `source.Subscribe`，並且該觀察者會針對每次 `OnNext` 呼叫調用 `predicate`。如果該述詞返回 `true`，那麼且僅當此時，由 `Where` 建立的觀察者才會在你傳遞給 `Where` 的觀察者上呼叫 `OnNext`。

`Where` 始終會傳遞對 `OnComplete` 或 `OnError` 的最終呼叫。這意味著如果你編寫以下程式碼：

```csharp
IObservable<int> dropEverything = xs.Where(_ => false);
```

那麼儘管這將篩選掉所有元素（因為述詞忽略其引數並始終返回 `false`，指示 `Where` 捨棄所有內容），但這不會篩選掉錯誤或完成通知。

事實上，如果這就是你想要的——一個捨棄所有元素並僅在來源完成或失敗時通知你的運算子——有一種更簡單的方法。

## IgnoreElements

`IgnoreElements` 擴充方法允許你僅接收 `OnCompleted` 或 `OnError` 通知。它等效於使用述詞始終返回 `false` 的 `Where` 運算子，如下例所示：

```csharp
IObservable<int> xs = Observable.Range(1, 3);
IObservable<int> dropEverything = xs.IgnoreElements();

xs.Dump("Unfiltered");
dropEverything.Dump("IgnoreElements");
```

如輸出所示，`xs` 來源產生數字 1 到 3 然後完成，但如果我們透過 `IgnoreElements` 執行它，我們看到的只有 `OnCompleted`。

```
Unfiltered-->1
Unfiltered-->2
Unfiltered-->3
Unfiltered completed
IgnoreElements completed
```

## OfType

一些可觀察序列產生各種類型的項目。例如，考慮一個想要追蹤船舶移動的應用程式。這可以透過 AIS 接收器實現。AIS 是自動識別系統（Automatic Identification System），大多數遠洋船舶使用它來回報其位置、航向、航速和其他資訊。AIS 訊息有許多種。有些回報船隻的位置和航速，但其名稱是在不同類型的訊息中回報的。（這是因為大多數船隻移動的頻率高於其更改名稱的頻率，因此它們以截然不同的時間間隔廣播這兩種類型的資訊。）

想像一下這在 Rx 中會是什麼樣子。實際上你不需要想像。開源的 [Ais.Net 專案](https://github.com/ais-dotnet)包含一個 [`ReceiverHost` 類別](https://github.com/ais-dotnet/Ais.Net.Receiver/blob/15de7b2908c3bd67cf421545578cfca59b24ed2c/Solutions/Ais.Net.Receiver/Ais/Net/Receiver/Receiver/ReceiverHost.cs)，它透過 Rx 提供 AIS 訊息。`ReceiverHost` 定義了一個型別為 `IObservable<IAisMessage>` 的 `Messages` 屬性。由於 AIS 定義了多種訊息類型，因此這個可觀察來源可以產生許多不同種類的物件。它發出的所有內容都將實作 [`IAisMessage` 介面](https://github.com/ais-dotnet/Ais.Net.Receiver/blob/15de7b2908c3bd67cf421545578cfca59b24ed2c/Solutions/Ais.Net.Models/Ais/Net/Models/Abstractions/IAisMessage.cs)，該介面回報船隻的唯一識別碼，但沒有太多其他內容。但是 [`Ais.Net.Models` 函式庫](https://www.nuget.org/packages/Ais.Net.Models/)定義了許多其他介面，包括回報位置、航速和航向的 [`IVesselNavigation`](https://github.com/ais-dotnet/Ais.Net.Receiver/blob/15de7b2908c3bd67cf421545578cfca59b24ed2c/Solutions/Ais.Net.Models/Ais/Net/Models/Abstractions/IVesselNavigation.cs)，以及告訴你船隻名稱的 [`IVesselName`](https://github.com/ais-dotnet/Ais.Net.Receiver/blob/15de7b2908c3bd67cf421545578cfca59b24ed2c/Solutions/Ais.Net.Models/Ais/Net/Models/Abstractions/IVesselName.cs)。

假設你只對水中船隻的位置感興趣，而不在乎船隻的名稱。你將希望看到實作 `IVesselNavigation` 介面的所有訊息，並忽略所有未實作的訊息。你可以嘗試使用 `Where` 運算子來實現這一點：

```csharp
// 無法編譯！
IObservable<IVesselNavigation> vesselMovements = 
   receiverHost.Messages.Where(m => m is IVesselNavigation);
```

但是，這將無法編譯。你將收到以下錯誤：

```
Cannot implicitly convert type 
'System.IObservable<Ais.Net.Models.Abstractions.IAisMessage>' 
to 
'System.IObservable<Ais.Net.Models.Abstractions.IVesselNavigation>'
```

請記住，`Where` 的返回型別始終與其輸入相同。由於 `receiverHost.Messages` 的型別為 `IObservable<IAisMessage>`，這也是 `Where` 將返回的型別。碰巧的是我們的述詞確保只有實作 `IVesselNavigation` 的訊息才能通過，但是 C# 編譯器無法理解述詞和輸出之間的關係。（就編譯器所知，`Where` 可能會執行完全相反的操作，僅包含述詞返回 `false` 的那些元素。事實上，編譯器無法猜測 `Where` 可能如何使用其述詞。）

幸運的是，Rx 提供了專門針對這種情況的運算子。`OfType` 將項目篩選為僅屬於特定型別的項目。項目必須是指定的確切型別，或者繼承自該型別，或者如果它是介面，則它們必須實作它。這使我們能夠修正上一個範例：

```csharp
IObservable<IVesselNavigation> vesselMovements = 
   receiverHost.Messages.OfType<IVesselNavigation>();
```

## 位置篩選

有時，我們關心的不是元素是什麼，而是它在序列中的位置。Rx 定義了幾個可以幫助我們解決此問題的運算子。

### FirstAsync 與 FirstOrDefaultAsync

LINQ 提供者通常實作提供序列第一個元素的 `First` 運算子。Rx 也不例外，但 Rx 的本質意味著我們通常需要它以略微不同的方式運作。對於靜態資料提供者（例如 LINQ to Objects 或 Entity Framework Core），來源元素已經存在，因此擷取第一個項目只是讀取它的問題。但是對於 Rx，來源在它們選擇的時間產生資料，因此無法知道第一個項目何時可用。

因此在 Rx 中，我們通常使用 `FirstAsync`。這將返回一個 `IObservable<T>`，該物件將產生從來源序列中出現的第一個值，然後完成。（Rx 也確實提供了更傳統的 `First` 方法，但它可能會產生問題。詳情請參閱後續的[**First/Last/Single[OrDefault] 的阻塞版本**小節](#blocking-versions-of-firstlastsingleordefault)。）

例如，此程式碼使用前面介紹的 AIS.NET 來源回報特定船隻（碰巧名為 HMS Example）第一次回報它正在移動：

```csharp
uint exampleMmsi = 235009890;
IObservable<IVesselNavigation> moving = 
   receiverHost.Messages
    .Where(v => v.Mmsi == exampleMmsi)
    .OfType<IVesselNavigation>()
    .Where(vn => vn.SpeedOverGround > 1f)
    .FirstAsync();
```

除了使用 `FirstAsync` 之外，這還使用了前面描述的其他幾個篩選元素。它從一個 [`Where`](#where) 步驟開始，將訊息篩選為來自我們碰巧感興趣的一艘船隻。（具體來說，我們根據該船隻的[水上行動業務識別碼，即 MMSI](https://en.wikipedia.org/wiki/Maritime_Mobile_Service_Identity) 進行篩選。）然後我們使用 [`OfType`](#oftype)，以便我們只檢視那些回報船隻如何/是否正在移動的訊息。然後我們使用另一個 `Where` 子句，以便我們可以忽略表示船隻實際上沒有移動的訊息，最後，我們使用 `FirstAsync`，以便我們僅獲得指示移動的第一條訊息。一旦船隻移動，這個 `moving` 來源將發出單個 `IVesselNavigation` 事件，然後立即完成。

我們可以稍微簡化該查詢，因為 `FirstAsync` 可以選擇性地接受述詞。這使我們能夠將最終的 `Where` 和 `FirstAsync` 摺疊為單個運算子：

```csharp
IObservable<IVesselNavigation> moving = 
   receiverHost.Messages
    .Where(v => v.Mmsi == exampleMmsi)
    .OfType<IVesselNavigation>()
    .FirstAsync(vn => vn.SpeedOverGround > 1f);
```

如果 `FirstAsync` 的輸入為空會怎麼樣？如果它在沒有產生任何項目的情況下完成，`FirstAsync` 會呼叫其訂閱者的 `OnError`，並傳遞一個 `InvalidOperationException`，其錯誤訊息回報該序列不包含任何元素。如果我們使用的是接受述詞的形式（如第二個範例中所示），並且沒有出現與述詞相符的元素，情況也是如此。這與 LINQ to Objects 的 `First` 運算子一致。（請注意，在剛顯示的範例中，我們預期不會發生這種情況，因為只要應用程式正在執行，來源就會繼續回報 AIS 訊息，這意味著它沒有任何理由完成。）

有時，我們可能希望容忍這種沒有事件的情況。大多數 LINQ 提供者不僅提供 `First`，還提供 `FirstOrDefault`。我們可以透過修改前面的範例來使用它。這使用 [`TakeUntil` 運算子](#skipuntil-and-takeuntil)引入截止時間：此範例準備等待 5 分鐘，但在那之後放棄。（因此，儘管 AIS 接收器可以無休止地產生訊息，但此範例已決定不會永遠等待。）既然這意味著我們可能會在沒有看到船隻移動的情況下完成，我們已將 `FirstAsync` 替換為 `FirstOrDefaultAsync`：

```csharp
IObservable<IVesselNavigation?> moving = 
   receiverHost.Messages
    .Where(v => v.Mmsi == exampleMmsi)
    .OfType<IVesselNavigation>()
    .TakeUntil(DateTimeOffset.Now.AddMinutes(5))
    .FirstOrDefaultAsync(vn => vn.SpeedOverGround > 1f);
```

如果在 5 分鐘後，我們沒有看到來自船隻指示其以 1 節或更快的速度移動的訊息，`TakeUntil` 將從其上游來源取消訂閱，並將在由 `FirstOrDefaultAsync` 提供的觀察者上呼叫 `OnCompleted`。`FirstAsync` 會將此視為錯誤，而 `FirstOrDefaultAsync` 將產生其元素型別的預設值（在此範例中為 `IVesselNavigation`；介面型別的預設值為 `null`），將其傳遞給其訂閱者的 `OnNext`，然後呼叫 `OnCompleted`。

簡而言之，這個 `moving` 可觀察物件將始終產生恰好一個項目。它要麼產生一個表示船隻已移動的 `IVesselNavigation`，要麼產生 `null` 以表示在此程式碼允許的 5 分鐘內沒有發生這種情況。

產生 `null` 可能是指示某事沒有發生的一種不錯的方式，但它有一些笨拙之處：任何使用此 `moving` 來源的程式碼現在都必須弄清楚通知是表示感興趣的事件，還是表示沒有此類事件。如果這對你的程式碼來說很方便，那就太好了，但 Rx 提供了一種更直接的方式來表示事件的缺失：空序列。

你可以想像一個以這種方式運作的 _first or empty_ 運算子。這對於返回實際值的 LINQ 提供者來說沒有意義。例如，因為 LINQ to Objects 的 `First` 返回 `T` 而不是 `IEnumerable<T>`，所以它無法返回空序列。但是因為 Rx 提供返回 `IObservable<T>` 的類似 `First` 的運算子，所以在技術上可以擁有一個返回第一個項目或根本不返回任何項目的運算子。Rx 中沒有內建這樣的運算子，但我們可以透過使用更通用的運算子 `Take` 來獲得完全相同的效果。

### Take

`Take` 是一個標準的 LINQ 運算子，它從序列中獲取前幾個項目，然後捨棄其餘項目。

在某種意義上，`Take` 是 `First` 的泛化：`Take(1)` 僅返回第一個項目，因此你可以將 LINQ 的 `First` 視為 `Take` 的特例。這並不完全正確，因為這些運算子對缺失元素的反應不同：正如我們剛才所見，`First`（以及 Rx 的 `FirstAsync`）堅持接收至少一個元素，如果你向其提供空序列，則會產生 `InvalidOperationException`。即使在存在性上更為寬鬆的 `FirstOrDefault` 仍然堅持產生某些東西。`Take` 的運作方式略有不同。

如果 `Take` 的輸入在產生與指定數量相同的元素之前完成，`Take` 不會發出抱怨——它只是轉發來源提供的任何內容。如果來源除了呼叫 `OnCompleted` 之外什麼也沒做，那麼 `Take` 就只是在其觀察者上呼叫 `OnCompleted`。如果我們使用 `Take(5)`，但來源產生了三個項目然後完成，`Take(5)` 將把這三個項目轉發給其訂閱者，然後完成。這意味著我們可以使用 `Take` 來實作前一節中討論的假設的 `FirstOrEmpty`：

```csharp
public static IObservable<T> FirstOrEmpty<T>(this IObservable<T> src) => src.Take(1);
```

現在是提醒你的好時機：大多數 Rx 運算子（以及本章中的所有運算子）本質上既不是熱的也不是冷的。它們依賴其來源。給定某個熱 `source`，`source.Take(1)` 也是熱的。我在這些範例中使用的 AIS.NET `receiverHost.Messages` 來源是熱的（因為它回報來自船隻的即時訊息廣播），因此衍生自它的可觀察序列也是熱的。為什麼現在是討論這個的好時機？因為這使我能夠講出以下絕對糟糕的雙關語（雙關語：hot take）：

```csharp
IObservable<IAisMessage> hotTake = receiverHost.Messages.Take(1);
```

謝謝大家。我整個星期都在這裡。

`FirstAsync` 和 `Take` 運算子從序列的開頭運作。如果我們只對結尾感興趣呢？

### LastAsync、LastOrDefaultAsync 與 PublishLast

LINQ 提供者通常提供 `Last` 和 `LastOrDefault`。除了顧名思義它們返回最後一個元素而不是第一個元素之外，它們執行的操作與 `First` 或 `FirstOrDefault` 幾乎完全相同。與 `First` 一樣，Rx 的本質意味著與處理待用資料的 LINQ 提供者不同，最後一個元素可能不會只是放在那裡等待被提取。因此，就像 Rx 提供 `FirstAsync` 和 `FirstOrDefault` 一樣，它也提供 `LastAsync` 和 `LastOrDefaultAsync`。（它確實也提供了 `Last`，但同樣地，正如 [First/Last/Single[OrDefault] 的阻塞版本](#blocking-versions-of-firstlastsingleordefault)小節所討論的，這可能會產生問題。）

還有 [`PublishLast`](15_PublishingOperators.md#publishlast)。這具有與 `LastAsync` 類似的語意，但它處理多個訂閱的方式不同。每次你訂閱 `LastAsync` 返回的 `IObservable<T>` 時，它都會訂閱底層來源，但 `PublishLast` 僅對底層來源進行單一 `Subscribe` 呼叫。（為了精確控制何時發生這種情況，`PublishLast` 返回一個 `IConnectableObservable<T>`。正如[第 2 章的冷熱來源小節](02_KeyTypes.md#hot-and-cold-sources)所描述的，這提供了一個 `Connect` 方法，並且當你呼叫此方法時，由 `PublishLast` 返回的可連接可觀察物件會訂閱其底層來源。）一旦此單一訂閱從來源收到 `OnComplete` 通知，它將向所有訂閱者交付最終值。（它還會記住最終值，因此如果在產生最終值後有任何新的觀察者訂閱，他們將在訂閱時立即收到該值。）最終值之後會立即跟隨一個 `OnCompleted` 通知。這是一組基於 [`Multicast`](15_PublishingOperators.md#multicast) 運算子的系列運算子之一，將在後續章節中詳細說明。

`LastAsync` 和 `LastOrDefaultAsync` 之間的區別與 `FirstAsync` 和 `FirstOrDefaultAsync` 相同。如果來源在沒有產生任何內容的情況下完成，`LastAsync` 會回報錯誤，而 `LastOrDefaultAsync` 會發出其元素型別的預設值然後完成。`PublishLast` 處理空來源的方式再次不同：如果來源完成而不產生任何元素，則 `PublishLast` 返回的可觀察物件將執行相同的操作：在此情境下它既不產生錯誤也不產生預設值。

回報序列的最後一個元素帶來了 `First` 所沒有面臨的挑戰。知道何時從來源收到了第一個元素是非常容易的：如果來源產生了一個元素，並且它先前沒有產生過元素，那麼那就是第一個元素。這意味著像 `FirstAsync` 這樣的運算子可以立即回報第一個元素。但 `LastAsync` 和 `LastOrDefaultAsync` 沒有這種優勢。

如果你從來源收到一個元素，你怎麼知道它是最後一個元素？一般來說，你在收到它的那一刻無法知道這一點。只有當來源繼續呼叫你的 `OnCompleted` 方法時，你才會知道你已經收到了最後一個元素。這不一定會立即發生。前面的範例使用 `TakeUntil(DateTimeOffset.Now.AddMinutes(5))` 在 5 分鐘後結束序列，如果你這樣做，完全有可能在發出最後一個元素與 `TakeUntil` 關閉序列之間流逝了大量的時間。在 AIS 情境中，船隻可能每隔幾分鐘才發出一條訊息，因此完全有可能發生 `TakeUntil` 轉發了一條訊息，然後在幾分鐘後發現已達到截止時間而沒有任何後續訊息進入的情況。在最後一個 `OnNext` 和 `OnComplete` 之間可能已經過去了幾分鐘。

因此，`LastAsync` 和 `LastOrDefaultAsync` 在其來源完成之前完全不發出任何內容。**這具有一個重要的後果：** 在 `LastAsync` 從來源接收最後一個元素與將該元素轉發給其訂閱者之間可能存在顯著的延遲。

### TakeLast

前面我們看到 Rx 實作了標準的 `Take` 運算子，該運算子從序列的開頭轉發最多指定數量的元素，然後停止。`TakeLast` 轉發序列末尾的元素。例如，`TakeLast(3)` 要求取得來源序列的最後 3 個元素。與 `Take` 一樣，`TakeLast` 對產生過少項目的來源具有容錯性。如果來源產生的項目少於 3 個，`TakeLast(3)` 將只轉發整個序列。

`TakeLast` 面臨與 `Last` 相同的挑戰：它不知道自己何時接近序列的末尾。因此，它必須保留最近看到的值的複本。它需要記憶體來保留你指定的任意數量的值。如果你編寫 `TakeLast(1_000_000)`，它將需要配置足夠大的緩衝區來儲存 1,000,000 個值。它不知道它收到的第一個元素是否會是最後一百萬個元素之一。直到來源完成或來源發出了超過 1,000,000 個項目之前，它無法知道這一點。當來源最終完成時，`TakeLast` 現在將知道最後一百萬個元素是什麼，並且需要將它們一個接一個地全部傳遞給其訂閱者的 `OnNext` 方法。

### Skip 與 SkipLast

如果我們想要與 `Take` 或 `TakeLast` 運算子完全相反的操作呢？如果我不想從來源取得前 5 個項目，而是想捨棄前 5 個項目呢？也許我有某個從感測器獲取讀數的 `IObservable<float>`，並且我發現該感測器在前幾次讀數中會產生垃圾值，因此我想忽略這些值，並僅在它穩定後才開始接聽。我可以透過 `Skip(5)` 來實現這一點。

`SkipLast` 在序列末尾執行相同的操作：它省略末尾指定數量的元素。與我們剛剛討論過的一些其他運算子一樣，這必須處理它無法判斷自己何時接近序列末尾的問題。只有在來源發出了所有元素並接著發出 `OnComplete` 之後，它才能發現哪些是最後（例如）4 個元素。因此 `SkipLast` 將引入延遲。如果你使用 `SkipLast(4)`，在來源產生第 5 個元素之前，它不會轉發來源產生的第一個元素。因此它不需要等待 `OnCompleted` 或 `OnError` 才能開始執行操作，它只需要等待直到確定某個元素不是我們想要捨棄的元素之一即可。

其他關鍵的篩選方法非常相似，我認為我們可以將它們視為一個大組。首先我們將看 `Skip` 和 `Take`。這些方法的行為就像它們在 `IEnumerable<T>` 實作中的行為一樣。這些是 Skip/Take 方法中最簡單且可能最常用的方法。這兩個方法都只有一個參數：要略過或取得的值的數量。

### SingleAsync 與 SingleOrDefaultAsync

LINQ 運算子通常提供 `Single` 運算子，用於當來源應該提供恰好一個項目，且包含更多項目或為空都屬於錯誤的情況。此處適用的 Rx 考量與 `First` 和 `Last` 相同，因此得知 Rx 提供了一個返回 `IObservable<T>` 的 `SingleAsync` 方法，該方法要麼恰好呼叫一次其觀察者的 `OnNext`，要麼呼叫其 `OnError` 以指示來源回報了錯誤或來源未恰好產生一個項目，你可能不會感到驚訝。

使用 `SingleAsync`，如果來源為空，你將獲得錯誤，就像 `FirstAsync` 和 `LastAsync` 一樣，但如果來源包含多個項目，你也會獲得錯誤。有一個 `SingleOrDefault`，就像它的 first/last 對應項目一樣，容許空的輸入序列，在這種情況下產生一個帶有元素型別預設值的單一元素。

`Single` 和 `SingleAsync` 與 `Last` 和 `LastAsync` 具有相同的特性：當它們從來源收到一個項目時，它們一開始並不知道它是否應該是輸出。這看起來可能很奇怪：既然 `Single` 要求來源串流僅提供一個項目，它當然必須知道它將交付給其訂閱者的項目將是它收到的第一個項目。這是事實，但當它收到第一個項目時它還不知道的是來源是否會產生第二個項目。除非且直到來源完成，否則它無法轉發第一個項目。我們可以說 `SingleAsync` 的工作是首先驗證來源是否恰好包含一個項目，如果包含則轉發該項目，如果不包含則回報錯誤。在錯誤情況下，如果 `SingleAsync` 收到第二個項目，它就會知道出錯了，因此它可以在此時立即對其訂閱者呼叫 `OnError`。但在成功情境下，直到來源透過完成確認不再有任何內容之前，它無法知道一切正常。只有那時 `SingleAsync` 才會發出結果。

### First/Last/Single[OrDefault] 的阻塞版本

前面小節中描述的幾個運算子都以 `Async` 結尾。這有點奇怪，因為通常以 `Async` 結尾的 .NET 方法會返回 `Task` 或 `Task<T>`，然而這些都返回 `IObservable<T>`。此外，正如已經討論過的，這些方法中的每一個都對應於通常不以 `Async` 結尾的標準 LINQ 運算子。（為了進一步增加混淆，某些 LINQ 提供者（如 Entity Framework Core）確實包含其中一些運算子的 `Async` 版本，但它們是不同的。與 Rx 不同，這些版本實際上確實返回 `Task<T>`，因此它們仍然產生單一值，而不是 `IQueryable<T>` 或 `IEnumerable<T>`。）這種命名源自於 Rx 設計早期的不幸選擇。

如果今天從頭開始設計 Rx，前面小節中的相關運算子將只會具有一般名稱：`First`、`FirstOrDefault` 等等。它們都以 `Async` 結尾的原因是它們是在 Rx 2.0 中新增的，而 Rx 1.0 已經定義了具有這些名稱的運算子。此範例使用 `First` 運算子：

```csharp
int v = Observable.Range(1, 10).First();
Console.WriteLine(v);
```

這會印出值 `1`，這是此處由 `Range` 返回的第一個項目。但是看看該變數 `v` 的型別。它不是 `IObservable<int>`，它只是一個 `int`。如果我們在訂閱時不會立即產生值的 Rx 運算子中使用它會發生什麼？這是一個範例：

```csharp
long v = Observable.Timer(TimeSpan.FromSeconds(2)).First();
Console.WriteLine(v);
```

如果你執行此程式碼，你會發現對 `First` 的呼叫直到產生值後才會返回。它是一個_阻塞式_（blocking）運算子。我們通常在 Rx 中避免使用阻塞運算子，因為使用它們很容易造成死結。Rx 的重點在於我們可以建立對事件做出反應的程式碼，因此只是坐著等待直到特定可觀察來源產生值並不符合其精神。如果你發現自己想要這樣做，通常有更好的方法來實現你所尋求的結果。（或者也許 Rx 不適合你正在做的事情的模型。）

如果你確實需要等待這樣的值，最好將 `Async` 形式與 Rx 對 C# 的 `async`/`await` 的整合支援結合使用：

```csharp
long v = await Observable.Timer(TimeSpan.FromSeconds(2)).FirstAsync();
Console.WriteLine(v);
```

這在邏輯上具有相同的效果，但因為我們使用的是 `await`，所以在等待可觀察來源產生值時這不會阻塞呼叫執行緒。這可以減少死結的機會。

我們能夠使用 `await` 的事實使得這些方法以 `Async` 結尾有些道理，但你可能想知道這裡發生了什麼事。我們已經看到這些方法都返回 `IObservable<T>` 而不是 `Task<T>`，那麼我們如何能夠使用 `await` 呢？[離開 Rx 世界章節中有完整解釋](13_LeavingIObservable.md#integration-with-async-and-await)，但簡短的回答是 Rx 提供了啟用此功能的擴充方法。當你 `await` 一個可觀察序列時，一旦來源完成，`await` 就會完成，並且它將返回從來源中出現的最終值。這對於產生恰好一個項目的運算子（例如 `FirstAsync` 和 `LastAsync`）非常有效。

請注意，偶爾會出現值立即可用的情況。例如，[第 3 章中的 `BehaviorSubject<T>` 小節](03_CreatingObservableSequences.md#behaviorsubjectt)指出 `BehaviorSubject<T>` 的定義特徵是它始終具有目前值。這意味著 Rx 的 `First` 方法實際上不會阻塞——它將訂閱 `BehaviorSubject<T>`，並且 `BehaviorSubject<T>.Subscribe` 在返回之前會在其訂閱者的可觀察物件上呼叫 `OnNext`。這使得 `First` 能夠立即返回值而不會阻塞。（當然，如果你使用接受述詞的 `First` 多載，且如果 `BehaviorSubject<T>` 的值不滿足述詞，則 `First` 將會阻塞。）

### ElementAt

還有另一個用於從來源選取特定元素的標準 LINQ 運算子：`ElementAt`。你向其提供一個數字，指出你所需的元素在序列中的位置。在待用資料 LINQ 提供者中，這在邏輯上等同於依索引存取陣列元素。Rx 實作了此運算子，但大多數 LINQ 提供者的 `ElementAt<T>` 實作返回 `T`，而 Rx 的實作返回 `IObservable<T>`。與 `First`、`Last` 和 `Single` 不同，Rx 不提供 `ElementAt<T>` 的阻塞形式。但由於你可以 await 任何 `IObservable<T>`，你始終可以這樣做：

```csharp
IAisMessage fourth = await receiverHost.Message.ElementAt(4);
```

如果你的來源序列僅產生五個值而我們請求 `ElementAt(5)`，則 `ElementAt` 返回的序列將在來源完成時向其訂閱者回報 `ArgumentOutOfRangeException` 錯誤。我們可以用三種方法來處理這個問題：

- 妥善處理 OnError
- 使用 `.Skip(5).Take(1);` 這將忽略前 5 個值並且只獲取第 6 個值。
如果序列少於 6 個元素，我們只會得到一個空序列，而不會出現錯誤。
- 使用 `ElementAtOrDefault`

`ElementAtOrDefault` 擴充方法會在索引超出範圍的情況下透過推送 `default(T)` 值來保護我們。目前無法提供你自己的預設值。

## 時間篩選

`Take` 和 `TakeLast` 運算子讓我們篩選掉除了最開頭或最末尾的元素之外的所有內容（而 `Skip` 和 `SkipLast` 讓我們看到除這些之外的所有內容），但這些都需要我們知道確切的元素數量。如果我們不想根據元素數量而是根據特定時間點來指定截止點呢？

事實上你已經看過一個範例：先前我使用 `TakeUntil` 將無止境的 `IObservable<T>` 轉換為在五分鐘後完成的序列。這是一個系列運算子之一。

### SkipWhile 與 TakeWhile

在 [`Skip` 與 `SkipLast` 小節](#skip-and-skiplast)中，我描述了一個在其前幾次讀數中產生垃圾值的感測器。這很常見。例如，氣體監控感測器通常需要將某些元件提升到正確的運作溫度，然後才能產生準確的讀數。在該小節的範例中，我使用了 `Skip(5)` 來忽略前幾次讀數，但這是一個粗糙的解決方案。我們怎麼知道 5 個就足夠了？或者它可能更快準備好，在這種情況下 5 個又太少了。

我們真正想做的是捨棄讀數，直到我們知道讀數將有效為止。這正是 `SkipWhile` 可以派上用場的情境。假設我們有一個回報某種特定氣體濃度的氣體感測器，但該感測器也回報正在執行偵測的感測器板的溫度。我們可以表達實際需求，而不是寄望於略過 5 個讀數是一個合理的數字：

```csharp
const int MinimumSensorTemperature = 74;
IObservable<SensorReading> readings = sensor.RawReadings
    .SkipUntil(r => r.SensorTemperature >= MinimumSensorTemperature);
```

這直接表達了我們需要的邏輯：這將捨棄讀數，直到設備達到其最低運作溫度為止。

下一組方法允許你在述詞評估為 true 時從序列中略過或獲取值。對於 `SkipWhile` 作業，這將篩選掉所有值，直到某個值未通過述詞為止，然後可以返回剩餘的序列。

```csharp
var subject = new Subject<int>();
subject
    .SkipWhile(i => i < 4)
    .Subscribe(Console.WriteLine, () => Console.WriteLine("Completed"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnNext(4);
subject.OnNext(3);
subject.OnNext(2);
subject.OnNext(1);
subject.OnNext(0);

subject.OnCompleted();
```

輸出：

```
4
3
2
1
0
Completed

```

`TakeWhile` 將在述詞通過時返回所有值，當第一個值失敗時序列將完成。

```csharp
var subject = new Subject<int>();
subject
    .TakeWhile(i => i < 4)
    .Subscribe(Console.WriteLine, () => Console.WriteLine("Completed"));
subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
subject.OnNext(4);
subject.OnNext(3);
subject.OnNext(2);
subject.OnNext(1);
subject.OnNext(0);

subject.OnCompleted();
```

輸出：

```
1
2
3
Completed

```

### SkipUntil 與 TakeUntil

除了 `SkipWhile` 和 `TakeWhile` 之外，Rx 還定義了 `SkipUntil` 和 `TakeUntil`。這些聽起來可能只不過是相同想法的另一種表達方式：你可能會期望 `SkipUntil` 執行的操作與 `SkipWhile` 幾乎完全相同，唯一的區別是 `SkipWhile` 只要其述詞返回 `true` 就會執行，而 `SkipUntil` 只要其述詞返回 `false` 就會執行。並且確實有一個 `SkipUntil` 多載可以做到這一點（以及 `TakeUntil` 的對應多載）。如果僅此而已，它們就不會那麼有趣了。然而，`SkipUntil` 和 `TakeUntil` 的多載使我們能夠完成 `SkipWhile` 和 `TakeWhile` 無法完成的事情。

你已經看過一個範例。[`FirstAsync` 與 `FirstOrDefaultAsync`](#firstasync-and-firstordefaultasync) 包含了一個使用接受 `DateTimeOffset` 的 `TakeUntil` 多載的範例。這會包裝任何 `IObservable<T>`，返回一個將轉發來源的所有內容直到指定時間的 `IObservable<T>`，在該時間點它將立即完成（並將從底層來源取消訂閱）。

我們無法使用 `TakeWhile` 實現這一點，因為它僅在來源產生項目時才查詢其述詞。如果我們希望來源在特定時間完成，我們可以使用 `TakeWhile` 實現該目標的唯一方法是其來源碰巧在我們希望結束的確切時刻產生了一個項目。`TakeWhile` 僅會作為其來源產生項目的結果而完成。`TakeUntil` 可以非同步完成。如果我們指定了未來 5 分鐘的時間，那麼當該時間到達時來源是否完全閒置都沒有關係。`TakeUntil` 無論如何都會完成。（它依賴[排程器](11_SchedulingAndThreading.md#schedulers)才能執行此操作。）

我們不必使用時間，`TakeUntil` 提供了一個接受第二個 `IObservable<T>` 的多載。這使我們能夠告訴它在發生有趣的事情時停止，而不需要事先確切知道何時會發生。`TakeUntil` 的這個多載會轉發來自來源的項目，直到第二個 `IObservable<T>` 產生值為止。`SkipUntil` 提供了一個類似的多載，其中第二個 `IObservable<T>` 決定何時應開始轉發來自來源的項目。

**注意**：這些多載需要第二個可觀察物件產生一個值才能觸發開始或結束。如果第二個可觀察物件在沒有產生單一通知的情況下完成，則它沒有任何效果——`TakeUntil` 將無限期地繼續獲取項目；`SkipUntil` 將永遠不會產生任何內容。換句話說，這些運算子將把 `Observable.Empty<T>()` 視為實際上等效於 `Observable.Never<T>()`。

還有一個接受 `CancellationToken` 的 `TakeUntil` 多載。這將轉發來自來源的通知，直到來源本身完成，或者語彙基元發出取消信號為止，此時 `TakeUntil` 將完成。

### Distinct 與 DistinctUntilChanged

`Distinct` 是另一個標準的 LINQ 運算子。它從序列中移除重複項。為此，它需要記住其來源產生的所有值，以便它可以篩選掉之前看過的任何項目。Rx 包含 `Distinct` 的實作，此範例使用它來顯示產生 AIS 訊息的船隻的唯一識別碼，但確保我們僅在第一次看到每個此類識別碼時才顯示它：

```csharp
IObservable<uint> newIds = receiverHost.Messages
    .Select(m => m.Mmsi)
    .Distinct();

newIds.Subscribe(id => Console.WriteLine($"New vessel: {id}"));
```

（這有點言之過早——它使用了 `Select`，我們將在[序列轉換章節](06_Transformation.md)中介紹它。但是，這是一個使用非常廣泛的 LINQ 運算子，因此你可能已經熟悉它了。我在這裡使用它從訊息中僅擷取 MMSI——船隻識別碼。）

如果我們只對船隻的識別碼感興趣，這個範例很好。但是如果我們想檢查這些訊息的細節呢？我們如何才能保留僅查看以前從未聽說過的船隻訊息的能力，但仍然能夠查看這些訊息中的資訊？使用 `Select` 擷取 ID 會阻止我們這樣做。幸運的是，`Distinct` 提供了一個多載，使我們能夠變更它確定唯一性的方式。我們可以向其提供一個函式，讓我們挑選任何我們喜歡的特徵，而不是讓 `Distinct` 檢視它正在處理的值。因此，我們不是將串流篩選為以前從未見過的值，而是可以將串流篩選為具有我們以前從未見過的某個特定屬性或屬性組合的值。這是一個簡單的範例：

```csharp
IObservable<IAisMessage> newVesselMessages = 
   receiverHost.Messages.Distinct(m => m.Mmsi);
```

在這裡，`Distinct` 的輸入現在是一個 `IObservable<IAisMessage>`。（在前面的範例中它實際上是 `IObservable<uint>`，因為 `Select` 子句僅挑選出 MMSI。）因此，每次來源發出訊息時，`Distinct` 現在都會接收整個 `IAisMessage`。但是因為我們提供了一個回呼，它不會嘗試將整個 `IAisMessage` 訊息相互比較。相反，每次它收到一個訊息時，都會將其傳遞給我們的回呼，然後查看我們的回呼返回的值，並將其與回呼為所有先前看到的訊息返回的值進行比較，並且僅當它是新值時才讓該訊息通過。

所以效果類似於以前。訊息僅在具有以前未見過的 MMSI 時才會被允許通過。但不同之處在於，此處的 `Distinct` 運算子輸出是 `IObservable<IAisMessage>`，因此當 `Distinct` 讓一個項目通過時，整個原始訊息仍然可用。

除了標準的 LINQ `Distinct` 運算子之外，Rx 還提供了 `DistinctUntilChanged`。這僅在發生變更時才讓通知通過，它是透過僅篩選掉相鄰的重複項來實現的。例如，給定序列 `1,2,2,3,4,4,5,4,3,3,2,1,1`，它將產生 `1,2,3,4,5,4,3,2,1`。`Distinct` 記住曾經產生的每個值，而 `DistinctUntilChanged` 僅記住最近發出的值，並且僅當新值與該最新值相符時才篩選掉新值。

此範例使用 `DistinctUntilChanged` 偵測特定船隻何時回報 `NavigationStatus` 的變更。

```csharp
uint exampleMmsi = 235009890;
IObservable<IAisMessageType1to3> statusChanges = receiverHost.Messages
    .Where(v => v.Mmsi == exampleMmsi)
    .OfType<IAisMessageType1to3>()
    .DistinctUntilChanged(m => m.NavigationStatus)
    .Skip(1);
```

例如，如果船隻反覆回報 `AtAnchor` 狀態，`DistinctUntilChanged` 將捨棄每條此類訊息，因為狀態與之前的狀態相同。但如果狀態變更為 `UnderwayUsingEngine`，這將導致 `DistinctUntilChanged` 允許回報該狀態的第一條訊息通過。然後它將不允許任何進一步的訊息通過，直到值再次發生變更，要麼變回 `AtAnchor`，要麼變為其他值（例如 `Moored`）。（末尾的 `Skip(1)` 存在是因為 `DistinctUntilChanged` 始終允許它看到的第一條訊息通過。我們無法知道這是否實際代表狀態的變更，但很可能不是：船隻每隔幾分鐘回報一次狀態，但它們變更該狀態的頻率要低得多，因此我們第一次收到船隻狀態的報告時，它可能並不代表狀態的變更。透過捨棄第一個項目，我們確保 `statusChanges` 僅在我們可以確定狀態已變更時才提供通知。）

這就是我們對 Rx 中可用篩選方法的快速瀏覽。雖然它們相對簡單，但正如我們已經開始看到的那樣，Rx 的強大之處在於其運算子的可組合性。

篩選運算子是你在這個資訊豐富的時代管理可能面臨的大量資料的第一站。我們現在知道如何套用各種準則來移除資料。接下來，我們將討論可以轉換資料的運算子。