# UWP 專屬的 `ThreadPoolScheduler` 功能在 `uap10.0.18362` 目標中保持可見

作為主要 Rx.NET 套件的 [`System.Reactive` NuGet 套件](https://www.nuget.org/packages/System.Reactive/)，現在在其對外公開的 API 中隱藏了特定於 UI 框架的型別。它在執行階段二進位檔案中保留了它們以實現回溯相容性，但這些型別不存在於 `ref` 組件中，這意味著編譯器看不到它們。然而，`uap10.0.18362` 目標（針對*未*使用 .NET 9 中新增的 .NET 執行階段對 UWP 支援之 UWP 應用程式的目標）提供了一個具有某些 UWP 專屬功能的 `ThreadPoolScheduler` 類別，並且這些功能即使在 `ref` 組件中也是可見的。本文件解釋了原因。

## 狀態

提議（Proposed）


## 作者

@idg10 ([Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/))。


## 背景

如 [ADR 0005](0005-package-split.md) 中所述，主要 [`System.Reactive` NuGet 套件](https://www.nuget.org/packages/System.Reactive/)現在在建構階段隱藏特定於 UI 框架的型別。這意味著應用程式僅在明確要求時才會獲得對 UI 框架的支援。這修復了一個長期存在的問題，即使用 Rx 的獨立式應用程式即使兩者都未使用，也會獲得 WPF 和 Windows Forms 框架的完整複本。

在執行階段，`System.Reactive` 需要保留與先前版本相同的 API 介面，以便當使用為例如 Rx 6.0 建構之元件的應用程式最終使用較新版本的 Rx 時，那些較舊的元件仍能正常執行。如果某個函式庫參考了 `System.Reactive` 並使用該元件中的 `System.Reactive.Linq.ControlObservable` 類別，則不會有任何問題，因為執行階段 `System.Reactive` 組件（來自 NuGet 套件的 `lib` 資料夾）仍然提供此型別。但是 `ref` 資料夾中的組件省略了這些型別，這意味著它們在編譯階段不可用。

如果開發人員想要使用特定於 UI 框架的程式碼，該專案將需要適當的套件參考。

這其中有一個波折：UWP 的特化 `ThreadPoolScheduler`。

`ThreadPoolScheduler` 應該是一個獨立於 UI 框架的型別。它在所有 Rx.NET 目標中都可用，包括 `netstandard2.0` 和沒有 UI 框架可用的 `netX.0` 目標。（例如，Rx 6.0 中的 `net6.0` 目標。）因此它屬於公開可見的 API，即它需要存在於 `System.Reactive` 套件的 `ref` 組件中。問題在於 `System.Reactive` UWP 目標（`uap10.0.18362` TFM）包含與所有其他目標略有不同的該型別版本。它具有：

* 三個公用建構函式
  * 一個預設建構函式
  * 一個接受 `WorkItemPriority` 引數的建構函式
  * 一個接受 `WorkItemPriority` 和 `WorkItemOptions` 引數的建構函式
* 唯讀的 `Priority` 和 `Options` 屬性，用於回報建構時提供的 `WorkItemPriority` 和 `WorkItemOptions`

它之所以提供這些功能，是因為它是實作在 Windows 執行階段 [`Windows.System.Threading.ThreadPool`](https://learn.microsoft.com/en-us/uwp/api/windows.system.threading.threadpool?view=winrt-18362) 之上的。所有其他目標都使用 .NET 執行階段函式庫的 [`System.Threading.ThreadPool`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.threadpool)。這在早期版本的 UWP 中不可用，因此需要在該平台上採用不同的 `ThreadPoolScheduler` 實作。自 2017 年發布的 Windows 10.0.16299（又稱 1709，又稱「Windows 10 秋季創作者更新」）以來，UWP 就已支援 `netstandard2.0`，因此不再絕對需要 UWP 專屬的 `ThreadPoolScheduler`：`netstandard2.0` 的 Rx.NET 實作現在運作良好。

然而，當 UWP 確實獲得對 .NET `ThreadPool` 的支援時，已無法修改 UWP 實作來使用它。這是因為上述那些額外的公用成員只有在使用 `Windows.System.Threading.ThreadPool` 時才能提供：`WorkItemPriority` 和 `WorkItemOptions` 型別是特定於該特定執行緒集區的。

使用 Rx 6.0 或更舊版本為 UWP 撰寫的舊版程式碼可能期望 `ThreadPoolScheduler` 提供這些成員。因此，由 `System.Reactive` 套件的執行階段 UAP 組件（其 `lib/uap10.0.18362` 資料夾中的組件）提供的 `ThreadPoolScheduler` 絕對有必要提供這些 UWP 專屬成員。

我們可以在 `ref` 組件中隱藏這些成員。但這意味著與我們支援的所有其他 UI 框架不同，開發人員無法僅透過新增新的套件參考來解決升級到 Rx 7 所引起的建構錯誤。對於所有其他特定於 UI 框架的功能，我們已將整個*型別*從 `System.Reactive` 遷移到新套件。但我們無法在此情況下這樣做，因為 `ThreadPoolScheduler` 無法移出 `System.Reactive`——它是所有平台上可用的核心型別——它只是少數的

.NET 中沒有任何機制可以使我們在 `System.Reactive` 中定義核心 `ThreadPoolScheduler`，然後在單獨的套件中使用額外的 UWP 專屬成員對其進行擴充。（即使 C# 14 中的新 `extensions` 功能也沒有幫助，因為它僅支援屬性、運算子和方法。目前您無法定義擴充建構函式。）因此，當開發人員正在建構 UAP 樣式的 UWP 應用程式時，即使他們新增了對 `System.Reactive.WindowsRuntime` 套件的參考，`ThreadPoolScheduler` 型別也將繼續是來自 `System.Reactive` 的型別。

我們有兩種處理此問題的方式：

1. 從 `System.Reactive` 公開 API 的 `ThreadPoolScheduler` 中移除 UWP 專屬成員，這將迫使開發人員不僅要新增對 `System.Reactive.WindowsRuntime` 的套件參考，還要重寫其程式碼以使用替代型別
2. 僅針對舊版樣式的 UWP 應用程式（具有 `uap10` 目標的應用程式），繼續在 `System.Reactive` 公開 API 中使這個特定的 UWP 專屬功能保持可見。


## 決策

我們選擇了選項 2。然而，我們正在棄用這些成員，並且我們引入了一個新的 `WindowsRuntimeThreadPoolScheduler` 型別作為使用此功能的新方法，可在 `System.Reactive.WindowsRuntime` 中使用。（請注意，建議新的 UWP 應用程式以 .NET（例如 `net10.0`）為目標，而不是以舊的 `uap10` TFM 為目標，在這種情況下，此新型別是獲得此 UWP 專屬排程器行為的*唯一*方法。）

我們的觀點是，將 UWP 專屬成員新增到 `ThreadPoolScheduler` 是一個錯誤。我們不希望這成為正常使用下 Rx.NET 的特性。然而，在沒有警告的情況下將它們從公開 API 中完全移除將是無法接受的。（我們贊同從公開 API 中移除其他特定於 UI 框架的型別，因為可以使用適當的套件參考輕鬆地恢復它們，而無需進行其他程式碼變更。）因此我們拒絕了選項 1。

由於我們認為將 UWP 專屬成員納入 `ThreadPoolScheduler` 是一個錯誤，我們希望棄用它們的使用，因此在 `System.Reactive.WindowsRuntime` 套件中定義了新的 `WindowsRuntimeThreadPoolScheduler` 型別。並且我們已棄用 `ThreadPoolScheduler` 的 UWP 專屬成員，以鼓勵需要此功能的開發人員轉移到這個新型別。


## 後果

* 針對 `System.Reactive` 建構的舊版程式碼繼續執行且行為沒有變更（因為執行階段組件都繼續提供與以前相同的 API，包括特定於 UI 框架的功能）
* 主要 Rx.NET 元件 `System.Reactive` 的公開 API 並未完全脫離特定於 UI 框架的程式碼：在 `uap10.0.18362` 目標的這一個特定情況下，它仍然定義了一些 UWP 專屬成員
* 任何在 `ThreadPoolScheduler` 中使用這些 UWP 專屬成員並升級到新版本 Rx.NET 的開發人員都將看到警告（因為這些成員已使用 `[Obsolete]` 屬性宣告為已棄用），建議他們使用新的 `WindowsRuntimeThreadPoolScheduler`
