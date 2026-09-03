# 附錄 B：可處置物件（Disposables）
    
Rx 使用現有的 `IDisposable` 介面來表示訂閱。此設計選擇意味著我們可以使用知道如何處理此介面的現有語言功能。Rx 還提供了 `IDisposable` 的數個公用實作。這些可以在 `System.Reactive.Disposables` 命名空間中找到。本附錄簡要介紹了其中的每一個。

除 [`ScheduledDisposable`](#scheduleddisposable) 外，這些與 Rx 沒有特定的聯繫，並且在需要處理 `IDisposable` 的任何程式碼中都很有用。（不過，這些程式碼都位於 `System.Reactive` 中，因此儘管您可以在基於 Rx 的程式碼之外完全使用這些功能，但如果這樣做，您仍然會依賴 Rx.NET。）

## `Disposable.Empty`
此靜態屬性公開了 `IDisposable` 的實作，該實作在呼叫 `Dispose` 方法時不執行任何動作。當您必須提供 `IDisposable`（如果您使用 `Observable.Create` 則可能會發生這種情況）但在處置時無需執行任何操作時，這很有用。

## `Disposable.Create(Action)`

此靜態方法公開了 `IDisposable` 的實作，該實作在呼叫 `Dispose` 方法時呼叫提供的方法。由於該實作遵循具有等冪性的指引，因此僅在第一次呼叫 `Dispose` 方法時才會呼叫該動作。

## `BooleanDisposable`

此類別實作了 `IDisposable.Dispose` 方法，並且還定義了唯讀屬性 `IsDisposed`。在建構該類別時，`IsDisposed` 為 <code>false</code>，當呼叫 `Dispose` 方法時，將其設定為 <code>true</code>。

## `CancellationDisposable`

`CancellationDisposable` 類別在 .NET [取消範式](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/task-cancellation)（`CancellationTokenSource`）與資源管理範式（`IDisposable`）之間提供了一個整合點。您可以透過向建構函式提供 `CancellationTokenSource`，或者讓無參數建構函式為您建立一個執行個體來建立 `CancellationDisposable` 類別的執行個體。呼叫 `Dispose` 將呼叫 `CancellationTokenSource` 上的 `Cancel` 方法。`CancellationDisposable` 公開了兩個屬性（`Token` 和 `IsDisposed`）；它們分別是 `CancellationTokenSource` 屬性 `Token` 和 `IsCancellationRequested` 的包裝函式。

## `CompositeDisposable`

`CompositeDisposable` 型別允許您將多個可處置資源視為一個。您可以透過傳入可處置資源的 <code>params</code> 陣列來建立 `CompositeDisposable` 的執行個體。您還可以透過呼叫其 `Add` 方法將可處置資源新增至現有的 `CompositeDisposable` 執行個體。在 `CompositeDisposable` 上呼叫 `Dispose` 將依照提供這些資源的順序對每個資源呼叫 dispose。此外，`CompositeDisposable` 類別實作了 `ICollection<IDisposable>`；這允許您從集合中新增和移除資源。在 `CompositeDisposable` 被處置後，新增至該集合的任何後續資源都將立即被處置。無論集合本身是否已被處置，從集合中移除的任何項目也將被處置。這包括使用 `Remove` 和 `Clear` 方法。

`System.Reactive.Disposables.Fluent` 命名空間定義了一個名為 `DisposeWith` 的擴充方法，可用於任何 `IDisposable`，與 `CompositeDisposable` 搭配使用，提供了另一種新增可處置資源的方法。對此的典型用法是新增訂閱，例如：

```cs
CompositeDisposable d = new();

someObservable1.Subscribe(myObserver1)
    .DisposeWith(d);
someObservable2.Subscribe(myObserver2)
    .DisposeWith(d);
```

這與我們使用 `Add` 具有完全相同的效果，但它支援「流暢」（fluent）的開發風格，在該風格中，我們透過將多個方法呼叫鏈接在一起來建構行為。`DisposeWith` 傳回其引數，使您能夠根據需要鏈接進一步的呼叫。

## `ContextDisposable`
`ContextDisposable` 允許您強制在給定的 `SynchronizationContext` 上執行資源的處置。建構函式需要 `SynchronizationContext` 和 `IDisposable` 資源。當在 `ContextDisposable` 上呼叫 `Dispose` 方法時，提供的資源將在指定的內容中處置。

## `MultipleAssignmentDisposable`

`MultipleAssignmentDisposable` 公開了唯讀的 `IsDisposed` 屬性和讀/寫屬性 `Disposable`。在 `MultipleAssignmentDisposable` 上呼叫 `Dispose` 方法將處置 `Disposable` 屬性所持有的目前值。然後它將該值設定為 null。只要 `MultipleAssignmentDisposable` 尚未被處置，您就可以像預期的那樣將 `Disposable` 屬性設定為 `IDisposable` 值。一旦 `MultipleAssignmentDisposable` 被處置，嘗試設定 `Disposable` 屬性將導致該值立即被處置；同時，`Disposable` 將保持為 null。

## `RefCountDisposable`

`RefCountDisposable` 提供了防止處置底層資源的能力，直到所有相依資源都已處置為止。您需要一個底層的 `IDisposable` 值來建構 `RefCountDisposable`。然後，您可以呼叫 `RefCountDisposable` 執行個體上的 `GetDisposable` 方法以擷取相依資源。每次呼叫 `GetDisposable` 時，內部計數器都會遞增。每次從 `GetDisposable` 獲得的相依可處置物件之一被處置時，計數器都會遞減。只有當計數器達到零時，底層資源才會被處置。這允許您在計數為零之前或之後在 `RefCountDisposable` 本身呼叫 `Dispose`。

## `ScheduledDisposable`

與 `ContextDisposable` 類似，`ScheduledDisposable` 型別允許您指定一個排程器，底層資源將在該排程器上進行處置。您需要將 `IScheduler` 執行個體和 `IDisposable` 執行個體都傳遞給建構函式。當 `ScheduledDisposable` 執行個體被處置時，底層資源的處置將透過提供的排程器執行。

## `SerialDisposable`

`SerialDisposable` 與 `MultipleAssignmentDisposable` 非常相似，因為它們都公開了讀/寫 `Disposable` 屬性。它們之間的對比在於，每當在 `SerialDisposable` 上設定 `Disposable` 屬性時，先前的值都會被處置。與 `MultipleAssignmentDisposable` 一樣，一旦 `SerialDisposable` 被處置，`Disposable` 屬性將設定為 null，任何進一步設定它的嘗試都將處置該值。該值將保持為 null。

## `SingleAssignmentDisposable`

`SingleAssignmentDisposable` 類別還公開了 `IsDisposed` 和 `Disposable` 屬性。與 `MultipleAssignmentDisposable` 和 `SerialDisposable` 一樣，當 `SingleAssignmentDisposable` 被處置時，`Disposable` 值將設定為 null。這裡實作上的區別在於，如果在值不為 null 且 `SingleAssignmentDisposable` 尚未被處置的情況下嘗試設定 `Disposable` 屬性，`SingleAssignmentDisposable` 將擲回 `InvalidOperationException`。

<!-- 
TODO: we recently made SingleAssignmentDisposableValue public after a request to do so. This also doesn't mention MultipleAssignmentDisposableValue, which has been around for a while.

TODO: ICancelable?

TODO: StableCompositeDisposable?

TODO: fit this in?

```csharp
namespace System.Reactive.Disposables
{
    public static class Disposable
    {
    // 取得在處置時不執行任何動作的可處置物件。
    public static IDisposable Empty { get {...} }

    // 建立在處置時呼叫指定動作的可處置物件。
    public static IDisposable Create(Action dispose)
    {...}
    }
}
```

如您所見，它公開了兩個成員：`Empty` 和 `Create`。`Empty` 方法允許您取得一個 `IDisposable` 的虛設常式執行個體，在呼叫 `Dispose()` 時不執行任何操作。這在您需要滿足傳回 `IDisposable` 的介面要求但沒有相關的具體實作時非常有用。

另一個多載是 `Create` Factory 方法，允許您傳遞在處置執行個體時要呼叫的 `Action`。`Create` 方法將確保標準的 Dispose 語意，因此多次呼叫 `Dispose()` 只會呼叫您提供的委派一次：

```csharp
var disposable = Disposable.Create(() => Console.WriteLine("Being disposed."));
Console.WriteLine("Calling dispose...");
disposable.Dispose();
Console.WriteLine("Calling again...");
disposable.Dispose();
```

輸出：

```
Calling dispose...
Being disposed.
Calling again...
```

請注意，「Being disposed.」只列印一次。 -->