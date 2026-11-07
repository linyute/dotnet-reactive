# `IAsyncScheduler` 狀態引數

`IAsyncScheduler.ScheduleAsync` 接受呼叫端提供的狀態引數，並將其傳給排程的動作，與 Rx.NET 中的 `IScheduler.Schedule` 相同。

## 狀態

已提議。

## 背景

Rx.NET 的 `IScheduler`（自 Rx 2.0 起）在每個 `Schedule` 方法上都接受 `TState` 引數：

```csharp
IDisposable Schedule<TState>(TState state, Func<IScheduler, TState, IDisposable> action);
```

運算子以靜態（不擷取變數的）lambda 將其接收器（sink）作為狀態傳入，因此排程一個工作單元既不會配置閉包，也不會配置委派。AsyncRx.NET 的 `IAsyncScheduler` 於 2017 年 9 月撰寫，原本只是佔位用途，並於隔天補完，它沒有這樣的引數：

```csharp
ValueTask<IAsyncDisposable> ScheduleAsync(Func<CancellationToken, ValueTask> action);
```

沒有任何提交、註解或討論記錄原因。原作者早已為 Rx.NET 設計過帶有狀態引數的非同步回呼排程 API（2012 年 `Scheduler.Async.cs` 中的 `Scheduler.ScheduleAsync<TState>`），因此並非認為非同步回呼與狀態不能並用。最可能的解釋出於結構面：`IAsyncScheduler` 同時也從回呼中移除了排程器引數，因為遞迴排程（Rx.NET 中另一個貫穿 `TState` 的用途）在 AsyncRx.NET 中被取代，改為在單一長時間執行的非同步動作內搭配 `CancellationToken` 使用迴圈。兩者都消失後，簽章就單純成了 `Task.Run` 的形式，配置成本的角度從未被衡量。此問題曾在 dotnet/reactive#896（2020 年）提出，但未獲回應。

反對新增該引數的理由有兩點，已納入考量：

1. 真正會暫停的非同步動作無論如何都會配置其裝箱的狀態機（在 .NET 11 執行階段非同步下則是延續物件），而該物件已經持有被提升的區域變數。因此狀態省下的是三次配置中的兩次，而非兩次中的兩次。這是邊際效益遞減，而非徒勞無功：許多排程動作會同步完成（同步完成的 `OnNextAsync`、取消檢查），對這些動作而言，節省是全面的。執行階段非同步不會改變此分析，因為閉包與委派是它從不觸及的一般物件；它只是縮減了無法避免的部分，使可避免的部分占比更大。
2. 目前的 `AsyncSchedulerBase`/`TaskPoolAsyncScheduler` 自身就會為每個工作項目配置多個物件，因此呼叫端的節省將無從察覺。這並非關於介面的論點。實作可以隨時重寫而不會破壞任何人；介面是公開 API，決定了任何實作所能達成的上限，因此正是應該優先做對的地方。

## 決定

`IAsyncScheduler` 上的每個方法都接受 `TState` 引數，參數順序與 Rx.NET 相同（先狀態，再到期時間，最後是動作，使 lambda 位於尾端），而動作會在取消權杖之前先接收狀態，與 `CancellationToken.Register(Action<object?, CancellationToken>, object?)` 一致：

```csharp
ValueTask<IAsyncDisposable> ScheduleAsync<TState>(TState state, Func<TState, CancellationToken, ValueTask> action);
ValueTask<IAsyncDisposable> ScheduleAsync<TState>(TState state, TimeSpan dueTime, Func<TState, CancellationToken, ValueTask> action);
ValueTask<IAsyncDisposable> ScheduleAsync<TState>(TState state, DateTimeOffset dueTime, Func<TState, CancellationToken, ValueTask> action);
```

先前無狀態的簽章成為 `AsyncScheduler` 上的擴充方法，其實作方式是將呼叫端的委派作為狀態，搭配靜態 lambda 傳入，因此本身不會增加任何配置，且既有的呼叫程式碼無須修改即可編譯。`AsyncScheduler.ExecuteAsync` 新增對應的 `TState` 多載。`AsyncSchedulerBase` 的 `ScheduleAsyncCore` 改為泛型，因此衍生自它的排程器（本儲存庫中的全部排程器，包括 `TestAsyncScheduler`）只需覆寫一個泛型方法。

函式庫中的呼叫位置會在動作所需的一切皆不可變時使用狀態引數：接收器（`this`），或由觀察者、排程器與參數組成的實值 tuple，並在 `static async` lambda 的第一行解構。在排程動作與觀察者之間共用可變區域變數的位置（`Throttle`、`Timeout`、`Buffer`、`Window`、`Skip`、`SkipUntil`、`Delay` 與 `ObserveOn` 的排空邏輯）暫時維持為擷取變數的 lambda；它們需要重構為接收器類別才能受益，這屬於另一項變更。T4 產生的 `ToAsync` 多載也暫不處理。

## 影響

* 對 `IAsyncScheduler` 的實作者與 `AsyncSchedulerBase` 的子類別而言是重大變更。AsyncRx.NET 尚未發行，因此本儲存庫之外沒有任何實作者。
* 對呼叫端而言原始碼相容：`scheduler.ScheduleAsync(async ct => ...)` 與 `scheduler.ScheduleAsync(async ct => ..., dueTime)` 仍可編譯，且行為完全相同。
* `TaskPoolAsyncScheduler` 與 `SynchronizationContextAsyncScheduler` 現在改為向 `StartNew`/`Post` 傳入單一裝箱的 tuple 並搭配靜態回呼，而非閉包與委派；`AsyncSchedulerBase` 的計時多載不再擷取變數。會合等待器（`TaskAwaitable`、`ValueTaskAwaitable`）以單一 `ScheduledContinuation` 物件取代每次 await 的閉包，該物件本身即為其回呼狀態。
* 排程器實作對每個工作項目配置的物件仍多於必要（取消來源、`Unwrap`、`ContinueWith`）。這現在屬於實作層面的問題，介面不再限制其上限。
