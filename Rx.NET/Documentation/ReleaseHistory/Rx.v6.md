# Rx 發布歷程記錄 v6.0

## 6.1.0

此版本新增：

* 適用於 `IDisposable` 的 `DisposeWith` 擴充方法，以簡化與 `CompositeDisposable` 結合時的處置（請參閱 [#2178](https://github.com/dotnet/reactive/pull/2178)，感謝 [Chris Pulman](https://github.com/ChrisPulman)）
* 接受 `CancellationToken` 的 `TakeUntil` 新多載（請參閱 [#2181](https://github.com/dotnet/reactive/issues/2181)，感謝 [Nils Aufschläger](https://github.com/nilsauf)）
* 新的 `ResetExceptionDispatchState` 運算子，用於當重複提供相同 `Exception` 執行個體的來源（例如 `Throw` 或 `Repeat` 運算子）將與把 `OnError` 通知轉換為實際例外狀況的機制（例如 `await` 支援）結合使用時，以避免 [#2187](https://github.com/dotnet/reactive/issues/2187) 中描述的問題，即每次重新擲回例外狀況時，例外狀況的 `StackTrace` 都會變得越來越長

## v6.0.2

此版本修復：

* [議題 #2173：「MinObservers > 1 的 RefCount 行為異常」](https://github.com/dotnet/reactive/issues/2173)
* [議題 #2214：「具有斷開連線延遲和 minObservers 的 RefCount 在重新訂閱同步完成的來源時擲回 InvalidOperationException」](https://github.com/dotnet/reactive/issues/2214)
* [議題 #2215：「如果觀察者計數降至零然後又回升，具有斷開連線延遲和 minObservers 的 RefCount 會過早斷開連線」](https://github.com/dotnet/reactive/issues/2215)

注意：測試套件現在會在 .NET 9.0 上進行測試。因此無需進行任何變更。

## v6.0.1

此版本修復：

* [議題 #1942：「AOT 編譯產生單一修剪分析警告」](https://github.com/dotnet/reactive/issues/1942)
* [議題 #2005：「在群組數量上，GroupBy+[SelectMany/Merge] 中的 Observable 完成展現出 O(n^2) 行為」](https://github.com/dotnet/reactive/issues/2005)

注意：測試套件現在會在 .NET 8.0 上進行測試。因此無需進行任何變更。

## v6.0.0

### 重大變更

* 移除了已不受支援的目標架構（.NET Core 3.1、.NET 5）
* UWP 應用程式的最低目標平台從 10.0.16299.0 提高到 10.0.18362.0

### 新功能

* 針對 .NET 6 和 .NET 7 進行了測試
* 當過去來自 `Task` 的未處理例外狀況會導致 `TaskScheduler.UnobservedExceptions` 時，應用程式可以選擇無訊息地忽略失敗（以與 Rx 在等效的非 Task 導向情境中處理未處理例外狀況的方式保持一致；這僅適用於 Rx 無法回報例外狀況的情況，通常是因為相關 observable 不再具有我們可以呼叫 `OnError` 的訂閱者）
* `SingleAssignmentDisposableValue` 型別現在是 public
* 現在包含修剪註釋
* 偵錯符號現在作為獨立的 `snupkg` 提供，而不是內建在組件中，從而減小了組件大小