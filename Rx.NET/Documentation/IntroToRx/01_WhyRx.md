# 第 1 部分 - 入門

Rx 是一個用於處理事件串流的 .NET 函式庫。為什麼您會需要它？

## 為什麼選擇 Rx？

使用者希望獲得及時的資訊。如果您正在等待包裹送達，即時回報貨車的配送進度會比一個可疑的 2 小時配送時間範圍給您更多自由。金融應用程式依賴持續不斷的最新資料串流。我們期望手機和電腦向我們提供各種重要通知。而某些應用程式根本無法在沒有即時資訊的情況下運作。線上協作工具和多人遊戲完全依賴於資料的快速分發和傳遞。

簡而言之，當有趣的事情發生時，我們的系統需要做出反應。

即時資訊串流是電腦系統中基本且無處不在的元素。儘管如此，它們在程式語言中往往是次等公民。大多數語言透過類似陣列的方式支援資料序列，這假定資料位於記憶體中，隨時可供我們的程式碼隨意讀取。如果您的應用程式處理事件，陣列可能適用於歷史資料，但它們不是表示應用程式執行時所發生事件的好方法。儘管串流資料在計算中是一個相當古老的概念，但它往往顯得笨拙，其抽象通常透過與我們程式語言的型別系統整合不佳的 API 來呈現。

這很糟糕。即時資料對廣泛的應用程式至關重要。處理它應該像處理清單、字典和其他集合一樣容易。

[.NET 的 Reactive Extensions](https://github.com/dotnet/reactive)（簡稱 Rx.NET 或 Rx，可作為 [`System.Reactive` NuGet 套件](https://www.nuget.org/packages/System.Reactive/) 取得）將即時資料來源提升為頭等公民。Rx 不需要任何特殊的程式語言支援。它利用 .NET 的型別系統來表示資料串流，其方式讓 C#、F# 和 VB.NET 等 .NET 語言都能像使用集合型別一樣自然地使用它們。

（簡短的語法旁白：雖然「Reactive Extensions」這個詞是複數，但當我們將其縮寫為 Rx.NET 或 Rx 時，我們將其視為單數名詞。這是不一致的，但說「Rx are...」聽起來實在太奇怪了。）

例如，C# 提供了整合查詢功能，我們可以使用它來尋找清單中符合某些條件的所有項目。如果我們有一個 `List<Trade> trades` 變數，我們可能會這樣撰寫：

```csharp
var bigTrades =
    from trade in trades
    where trade.Volume > 1_000_000;
```

使用 Rx，我們可以將完全相同的程式碼套用於即時資料。`trades` 變數可以是一個 `IObservable<Trade>`，而不是 `List<Trade>`。`IObservable<T>` 是 Rx 中的基本抽象。它本質上是 `IEnumerable<T>` 的即時版本。在這種情況下，`bigTrades` 也將是一個 `IObservable<Trade>`，這是一個即時資料來源，能夠將所有 `Volume` 超過一百萬的交易通知我們。至關重要的是，它可以立即回報每筆此類交易——這就是我們所說的「即時」資料來源。

Rx 是一個極具生產力的開發工具。它使開發人員能夠使用所有 .NET 開發人員熟悉的語言功能來處理即時事件串流。它支援宣告式方法，通常使我們能夠比沒有 Rx 時更優雅、使用更少的程式碼來表達複雜的行為。

Rx 建立在 LINQ（語言整合查詢）之上。這使我們能夠使用上面顯示的查詢語法（或者您可以使用某些 .NET 開發人員偏好的明確函式呼叫方法）。LINQ 在 .NET 中廣泛用於資料存取（例如在 Entity Framework Core 中），但也用於處理記憶體中集合（使用 LINQ to Objects），這意味著經驗豐富的 .NET 開發人員往往會對 Rx 感到得心應手。至關重要的是，LINQ 是一種高度可組合的設計：您可以將運算子以任何您喜歡的組合連接在一起，以直接的方式表達潛在複雜的處理。這種可組合性源於其設計的數學基礎，但是儘管您可以根據需要學習 LINQ 的這一方面，但這不是先決條件：對其背後的數學不感興趣的開發人員只需享受 LINQ 提供者（如 Rx）提供了一組可以以無數不同方式插入在一起的建構區塊，並且一切都能正常運作。

LINQ 在處理海量資料方面有著良好的記錄。Microsoft 在其某些系統的內部實作中廣泛使用了它，包括支援數千萬活躍使用者的服務。

## Rx 何時適用？

Rx 專為處理事件序列而設計，這意味著它比其他情境更適合某些情境。接下來的幾節將描述其中的一些情境，以及它雖然不是顯而易見的配對但仍然值得考慮的情況。最後，我們描述了可以使用 Rx 但替代方案可能更好的某些情況。

### 適合 Rx

Rx 非常適合表示源自程式碼外部且應用程式需要對其做出回應的事件，例如：

- 整合事件，例如來自訊息匯流排的廣播，或來自 WebSockets API 的推送事件，或透過 MQTT 或其他低延遲中介軟體（如 [Azure Event Grid](https://azure.microsoft.com/en-gb/products/event-grid/)、[Azure Event Hubs](https://azure.microsoft.com/en-gb/products/event-hubs/) 和 [Azure Service Bus](https://azure.microsoft.com/en-gb/products/service-bus/)）接收的訊息，或非特定廠商的表示形式（如 [cloudevents](https://cloudevents.io/)）
- 來自監視裝置的遙測資料，例如水務公司基礎設施中的流量感測器，或寬頻提供商網路設備中的監視和診斷功能
- 來自行動系統的位置資料，例如來自船舶的 [AIS](https://github.com/ais-dotnet/) 訊息，或汽車遙測資料
- 作業系統事件，例如檔案系統活動或 WMI 事件
- 道路交通資訊，例如事故通知或平均速度變化
- 與[複合事件處理（CEP）](https://en.wikipedia.org/wiki/Complex_event_processing)引擎的整合
- UI 事件，例如滑鼠移動或按鈕點擊

Rx 也是為領域事件建模的好方法。這些事件可能會在剛剛描述的某些事件之後發生，但在處理它們之後會產生更直接代表應用程式概念的事件。這些可能包括：

- 領域物件上的屬性或狀態變更，例如「訂單狀態已更新」或「註冊已接受」
- 領域物件集合的變更，例如「建立新註冊」

事件也可能代表從傳入事件（或稍後分析的歷史資料）中衍生的見解，例如：

- 寬頻客戶可能在不知不覺中成為 DDoS 攻擊的參與者
- 兩艘遠洋船隻參與了一種通常與非法活動相關的移動模式（例如，在遠離海岸的海上長時間並排行駛，時間長到足以轉移貨物或人員）
- [CNC](https://en.wikipedia.org/wiki/Numerical_control)[銑床](https://en.wikipedia.org/wiki/Milling_(machining)) MFZH12 的 4 號軸承所表現出的磨損率明顯高於標準曲線
- 如果使用者希望準時到達城市另一端的會議地點，目前的交通狀況建議他們應該在接下來的 10 分鐘內出發

這三組範例展示了應用程式如何在處理事件時逐步提高資訊的價值。我們從原始事件開始，然後對其進行增強以產生特定於領域的事件，然後進行分析以產生應用程式使用者真正關心的通知。每個處理階段都會提高出現的訊息的價值。每個階段通常也會減少訊息量。如果我們將第一類別中的原始事件直接呈現給使用者，他們可能會被訊息量淹沒，從而無法發現重要事件。但是，如果我們僅在處理偵測到重要事項時才向他們呈現通知，這將使他們能夠更有效率、更準確地工作，因為我們顯著提高了訊噪比。

[`System.Reactive` 函式庫](https://www.nuget.org/packages/System.Reactive) 提供了用於建構此類增值流程的工具，在該流程中，我們馴服大量原始事件來源以產生高價值、即時、可採取動作的見解。它提供了一套運算子，使我們的程式碼能夠以宣告方式表達此類處理，正如您在後續章節中將看到的。

Rx 也非常適合出於*卸載*（offloading）目的引入和管理並行存取。
也就是說，並行執行一組給定的工作，以便偵測到事件的執行緒不一定也必須是處理該事件的執行緒。
這的一個非常流行的用途是維持快速回應的 UI。（UI 事件處理已成為 Rx 的一個*非常*受歡迎的用途——無論是在 .NET 中，還是在最初作為 Rx.NET 分支的 [RxJS](https://rxjs.dev/) 中——以至於很容易認為這就是它的用途。但它在那裡的成功不應讓我們忽視其更廣泛的適用性。）

如果您有一個嘗試為即時事件建模的現有 `IEnumerable<T>`，則應考慮使用 Rx。
雖然 `IEnumerable<T>` *可以*為動態資料建模（透過使用延遲評估，如 `yield return`），但存在一個問題。如果取用集合的程式碼已達到需要下一個項目的時間點（例如，因為 `foreach` 迴圈剛完成一次反覆運算），但尚無可用項目，則 `IEnumerable<T>` 實作別無選擇，只能在其 `MoveNext` 中封鎖呼叫端執行緒，直到資料可用為止，這可能會在某些應用程式中導致延展性問題。即使在執行緒封鎖可接受的情況下（或者如果您使用較新的 `IAsyncEnumerable<T>`，它可以利用 C# 的 `await foreach` 功能來避免在這些情況下封鎖執行緒），`IEnumerable<T>` 和 `IAsyncEnumerable<T>` 對於表示即時資訊來源也是具有誤導性的型別。這些介面代表了一種「拉取」（pull）程式設計模型：程式碼請求序列中的下一個項目。對於為自然按照自己的排程產生資訊的資訊來源進行建模，Rx 是一個更自然的選擇。

### 可能適合 Rx

Rx 可用於表示非同步作業。.NET 的 `Task` 或 `Task<T>` 實際上代表單個事件，而 `IObservable<T>` 可以被視為將其推廣到事件序列。（例如 `Task<int>` 和 `IObservable<int>` 之間的關係類似於 `int` 和 `IEnumerable<int>` 之間的關係。）

這意味著有一些情境可以使用工作和 `async` 關鍵字或透過 Rx 來處理。如果在處理過程中的任何時間點您需要同時處理多個值和單個值，Rx 可以兩者兼顧；工作不能很好地處理多個項目。您可以擁有一個 `Task<IEnumerable<int>>`，它使您能夠 `await` 一個集合，如果該集合中的所有項目都可以在單個步驟中收集，那就沒問題。這樣做的局限性在於，一旦工作產生了其 `IEnumerable<int>` 結果，您的 `await` 就完成了，您將回到對該 `IEnumerable<int>` 的非非同步反覆運算。如果資料無法在單個步驟中提取——也許 `IEnumerable<int>` 代表來自 API 的資料，其中結果是以每次 100 個項目的批次提取的——其 `MoveNext` 每次需要等待時都必須封鎖您的執行緒。

在其存在的前 5 年中，Rx 可以說是表示不一定能立即獲得所有項目的集合的最佳方式。然而，.NET Core 3.0 和 C# 8 中引入的 [`IAsyncEnumerable<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1) 提供了一種在保持在 `async`/`await` 世界中的同時處理序列的方法（並且 [`Microsoft.Bcl.AsyncInterfaces` NuGet 套件](https://www.nuget.org/packages/Microsoft.Bcl.AsyncInterfaces/) 使其在 .NET Framework 和 .NET Standard 2.0 上可用）。因此，現在選擇使用 Rx 往往歸結為「拉取」模型（以 `foreach` 或 `await foreach` 為代表）還是「推送」模型（其中程式碼提供在項目可用時由事件來源呼叫的回呼）更適合正在建模的概念。

自 Rx 首次出現以來新增到 .NET 的另一個相關功能是[通道](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels)（channels）。這些允許來源產生由取用端處理的物件，因此與 Rx 有著明顯的表面相似性。然而，Rx 的一個顯著特徵是它透過廣泛的運算子集合支援組合，這在通道中沒有直接的等效項。另一方面，通道為適應生產和消費速率的變化提供了更多選項。

前面，我提到了*卸載*：使用 Rx 將工作推送到其他執行緒。儘管此技術可以使 Rx 出於*擴充*或執行*平行*計算的目的引入和管理並行存取，但其他專用框架（如 [TPL（工作平行程式庫）資料流程](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/dataflow-task-parallel-library) 或 [PLINQ](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/introduction-to-plinq)）更適合執行平行計算密集型工作。然而，TPL 資料流程透過其 [`AsObserver`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.dataflow.dataflowblock.asobserver) 和 [`AsObservable`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.dataflow.dataflowblock.asobservable) 擴充方法提供了與 Rx 的某些整合。因此，通常使用 Rx 將 TPL 資料流程與應用程式的其餘部分進行整合。

### 不適合 Rx

Rx 的 `IObservable<T>` 不是 `IEnumerable<T>` 或 `IAsyncEnumerable<T>` 的替代品。將本質上基於拉取的內容強制設定為基於推送將是一個錯誤。

此外，在某些情況下，Rx 程式設計模型的簡單性可能會對您不利。例如，某些訊息佇列技術（如 MSMQ）根據定義是循序的，因此可能看起來很適合 Rx。但是，它們通常因其交易處理支援而被選中。Rx 沒有任何直接表達交易語意的方法，因此在需要此功能的情境中，您最好直接使用相關技術的 API。（話雖如此，[Reaqtor](https://reaqtive.net/) 為 Rx 新增了持久性和持續性，因此您可以使用它將這些類型的佇列系統與 Rx 進行整合。）

透過為工作選擇最佳工具，您的程式碼應該更容易維護，它可能會提供更好的效能，並且您可能會獲得更好的支援。

## Rx 實戰

您可以非常快速地啟動並執行一個簡單的 Rx 範例。如果您已安裝 .NET SDK，則可以在命令列中執行以下命令：

```ps1
mkdir TryRx
cd TryRx
dotnet new console
dotnet add package System.Reactive
```

或者，如果您安裝了 Visual Studio，請建立一個新的 .NET 主控台專案，然後使用 NuGet 套件管理員新增對 `System.Reactive` 的參考。

此程式碼建立一個可觀察來源（`ticks`），每秒產生一個事件。該程式碼還向該來源傳遞一個處理常式，該處理常式為每個事件向主控台寫入一則訊息：

```csharp
using System.Reactive.Linq;

IObservable<long> ticks = Observable.Timer(
    dueTime: TimeSpan.Zero,
    period: TimeSpan.FromSeconds(1));

ticks.Subscribe(
    tick => Console.WriteLine($"Tick {tick}"));

Console.ReadLine();
```

如果這看起來不是很令人興奮，那是因為它是可能建立的最基本的範例，並且在其核心，Rx 具有非常簡單的程式設計模型。強大功能來自於組合——我們可以使用 `System.Reactive` 函式庫中的建構區塊來描述將我們從原始、低階事件引導至高價值見解的處理。但要做到這一點，我們必須先了解 [Rx 的關鍵型別 `IObservable<T>` 和 `IObserver<T>`](02_KeyTypes.md)。
