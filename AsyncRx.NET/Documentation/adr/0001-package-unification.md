# AsyncRx.NET NuGet 套件整合

當 AsyncRx.NET 於 2017 年加入 dotnet/reactive 存放庫時，它反映了 Rx v3 的封裝方式，將所有不同元素拆分開來。本 ADR 描述改為較少套件數量的變更。

## 狀態

提案中。

## 背景

原本的 AsyncRx.NET 程式碼被拆分為許多套件。請注意，這些套件都未曾發行至 NuGet。我們過去有以下套件：

* `System.Reactive.Async.Concurrency`
* `System.Reactive.Async.Core`
* `System.Reactive.Async.Disposables`
* `System.Reactive.Async.Interfaces`
* `System.Reactive.Async.Linq`
* `System.Reactive.Async.Subjects`
* `System.Reactive.Async`

另外還有兩個不包含任何非同步 Rx 特定內容的專案：

* `System.Reactive.Bcl`
* `System.Reactive.Shared`

其中最後一個 `System.Reactive.Shared` 包含以下型別：

* `EventPattern`
* `IEventPattern`
* `IEventPatternSource`
* `IEventSource`
* `Notification`
* `TimeInterval`
* `TimeStamped`
* `Unit`

所有這些型別的定義都已存在於發行至 NuGet 的 `System.Reactive` 元件中。

`System.Reactive.Bcl` 專案包含兩個提供非同步鎖定功能的型別。它們被標記為 `Bcl` 是因為它們完全非 Rx 專屬，並且合理來說最終可能遷移至 BCL（從歷史上看，Rx 中的少數功能確實曾經這麼做過）。它們定義在 `System.Threading` 命名空間中。

了解 `System.Reactive.Async.Interfaces` 的歷史特別有幫助。這是 `System.Reactive.Interfaces` 的非同步對應元件。回到 Rx v2.0 時代，該介面函式庫被構想為預期不會演進之型別的歸宿。當時的計畫是，當 Rx 的其餘部分擴展與演進時，該函式庫將保持不變。透過這種方式，API 可以公開 Rx 介面型別，而不會對 Rx 實作型別的某些特定版本產生相依性。但計畫並未如願進行。新版本的 `System.Reactive.Async.Interfaces` 隨著新的 Rx 發行版本一同發行。接著在 Rx 4.0 中，發生了「大整合」（great unification），相關型別移入了 `System.Reactive`，而 `System.Reactive.Interfaces` 則成為僅包含型別轉發器的向下相容外觀（façade）。

當談到核心介面時，Rx.NET 與 AsyncRx.NET 之間存在一個重大差異。在 Rx.NET 中，兩個最關鍵的介面 `IObservable<T>` 和 `IObserver<T>` 已被新增至 .NET 執行階段函式庫。這代表任何元件都可以公開以 Rx 為基礎的介面範圍，而完全不需要相依於任何 Rx 函式庫。但 AsyncRx.NET 則**並非**如此。

AsyncRx.NET 中兩個最關鍵的介面 `IAsyncObservable<T>` 和 `IAsyncObserver<T>` 尚未在 .NET 執行階段函式庫中定義。可以說這是有需求的——例如，Project Orleans 定義了 (Orleans.Streams.IAsyncObservable<T>)[https://learn.microsoft.com/en-us/dotnet/api/orleans.streams.iasyncobservable-1?view=orleans-7.0]。然而，其定義包含了 Orleans 專屬的相依性，因此雖然該介面的存在證明了對此類功能具有通用需求，但 AsyncRx.NET 無法直接使用該定義。

此外，在 https://github.com/dotnet/reactive/issues/1296 中討論了取消（cancellation）如何融入其中的一些未解決問題，這表明 AsyncRx.NET 這些介面的現有定義可能尚未完全成熟。

## 決策

將會提供單一的 `System.Reactive.Async` 元件。在大多數情況下，我們只需將程式碼從其他專案移入此專案，而不做任何修改。然而，部分專案需要特殊考量。

`System.Reactive.Async.Interfaces` 將會被移除，因為目前尚不清楚即使是最關鍵的兩個介面 `IAsyncObservable<T>` 和 `IAsyncObserver<T>` 是否已完全成熟，因此若暗示這些介面代表某種穩定的型別、且比起任何特定 AsyncRx.NET 實作版本能夠在更長時間內值得信賴，將會造成誤導。（況且無論如何，這個構想在 Rx.NET 上並未完全如願實現。）此外，將這些介面型別合併至 `System.Reactive.Async` 完全符合 Rx.NET 在 v4.0 中所做的整合。（而且這不會引入我們現在看到由於整合過度所導致的任何問題，因為 AsyncRx.NET 中沒有 UI 框架特定的考量。）

`System.Reactive.Shared` 元件將會被移除，AsyncRx.NET 將改為相依於 `System.Reactive`，以便使用該函式庫中這些型別的定義。長期目標是讓 AsyncRx.NET 相依於（即將建立的）Rx.NET 版本，以解決意外相依於 UI 框架的問題，目前計畫為 Rx 7.0。為了標示 AsyncRx.NET 這種「面向未來」的特性，初始預覽版將不會相依於目前發行的 Rx 5.0，而是會相依於最新的可用預覽版，直到非預覽版的 Rx 7.0 發行。

`System.Reactive.Bcl` 元件將會被移除。其包含的型別將移入 `System.Reactive.Async`。我們將把它們移出 `System.Threading` 命名空間，因為目前我們沒有將它們納入 .NET 執行階段函式庫的計畫。我們將檢查執行階段函式庫現在是否實際上已具有類似功能，若是如此，我們將改用該功能並刪除這些型別。但若沒有直接對應的功能，我們將把它們移入適當的命名空間並設為 `internal`，因為它們屬於實作細節，且 AsyncRx.NET 的目的並非提供通用的非同步程式設計公用程式。

## 後果

AsyncRx.NET 的取用者將只會看到單一 NuGet 套件 `System.Reactive.Async`，消除了對於該使用哪個套件的不確定性。

透過移除 `System.Reactive.Shared`，我們避免了其包含之所有型別的重複定義。缺點是現在相依於 AsyncRx.NET 必然代表相依於 Rx.NET，而以前並非如此。（我們的觀點是，大多數需要 AsyncRx.NET 的專案可能也會需要 Rx.NET，因此我們不認為這是一個嚴重的缺點。）

