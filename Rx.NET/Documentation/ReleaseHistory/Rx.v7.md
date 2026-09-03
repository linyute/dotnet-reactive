# Rx 發布歷程記錄 v7.0

## 7.0.0

在 .NET 8、.NET 9 和 .NET 10.0 上受支援。

新功能：

* 具有 Windows 專屬 TFM（例如 `net8.0-windows.10.0.19041`）的應用程式現在可以參考 `System.Reactive` 套件，而不會自動取得對 `Microsoft.Desktop.App` 框架（包含 WPF 和 WinForms）的參考。如果應用程式使用獨立式部署或 AoT，這可以修復參考 Rx 會大幅增加可部署應用程式大小的問題。


### 重大變更

* UI 框架專屬的功能現在需要參考相關的平台專屬套件：
  * UWP 請使用 `System.Reactive.Uwp`
  * Windows Forms 請使用 `System.Reactive.Windows.Forms`
  * WPF 請使用 `System.Reactive.Wpf`
  * WinRT（例如 `CoreDispatcher`）支援請使用 `System.Reactive.WindowsRuntime`
* 如果具有 Windows 專屬 TFM 的應用程式一直依賴 `System.Reactive` 來取得 `Microsoft.Desktop.App` 框架相依性，則需要新增 `<UseWPF>true</UseWPF>` 或 `<UseWindowsForms>true</UseWindowsForms>`
* 修復了 `OfType` 的可為 Null 性（請參閱 [#2247](https://github.com/dotnet/reactive/issues/2247)）——這是一項 API 變更，因此雖然這修復了瑕疵，但在技術上是一項重大變更
* 已不再支援超出支援範圍的目標架構（.NET 6.0、.NET 7.0）

有關這些變更原因的深入說明，請參閱 [ADR 0005（將 UI 框架支援移出 `System.Reactive`）](../adr/0005-package-split.md)。

請注意，UI 專屬功能的套件變更構成了原始程式碼層級的重大變更，但不是二進位層級的重大變更。儘管 UI 框架專屬型別已從 `System.Reactive` 的公開 API 中移除，但它們在執行階段仍然存在。（NuGet 套件同時具有 `ref` 和 `lib` 資料夾。.NET 建構工具在編譯階段使用 `ref` 資料夾，而這些型別僅從 `ref` 組件中移除。在執行階段會使用 `lib` 資料夾，並且 `System.Reactive` v6 的完整 API 在 `lib` 中的組件中仍然可用。因此，針對 Rx 6.0 建構且在執行階段使用 Rx 7.0 的現有二進位檔將繼續正常運作。）

`System.Reactive` 有一個分析器，可偵測專案何時因為使用了過去位於 `System.Reactive` 但現在位於專案尚未參考之新套件中的 UI 專屬功能而出現建構錯誤。該分析器會產生診斷訊息，告知開發人員他們需要哪些新參考。


### 棄用 Facade

早在 Rx 4.0（上次對套件進行重大重組時），先前作為 Rx.NET 核心元件的各種 NuGet 套件被降級為相容性 facade。這些套件僅包含型別轉發器（type forwarders），指出它們過去定義的各種型別已移入 `System.Reactive`。

這些套件的存在是為了讓針對舊版 Rx.NET 建構的程式碼在升級到 Rx 4.0 或更新版本時能夠繼續運作。隨著 .NET 的每個後續版本，我們繼續建構這些套件的新版本，但通常變更的只是 TFM 中的確切版本。沒有人應該再使用這些 facade 了，因此沒有理由繼續產生新的 facade。（任何仍在使用舊版本的人都可以繼續使用。）

因此我們不再產生這些套件的新版本。

* [`System.Reactive.Compatibility`](https://www.nuget.org/packages/System.Reactive.Compatibility) 是一個中繼套件（metapackage），旨在支援從 Rx 3 遷移到 Rx 4，這兩者都早已不受支援
* [`System.Reactive.Core`](https://www.nuget.org/packages/System.Reactive.Core) 曾經是各種 Rx.NET 型別的所在地，這些型別現在是核心功能，並且永遠不會移出 `System.Reactive`：匿名 observer 和 observable、基底 observer 和 observable、具體化的通知（reified notifications）、`Unit`、通用排程器、通用 disposable、平台抽象
* [`System.Reactive.Experimental`](https://www.nuget.org/packages/System.Reactive.Experimental) 定義了 `ExperimentalAttribute` 和少數型別，所有這些型別都已經存在於 `System.Reactive` 中很長時間了
* [`System.Reactive.Interfaces`](https://www.nuget.org/packages/System.Reactive.Interfaces)：最初被構想為定義核心介面（例如 `IScheduler`、`IQbservable`）的穩定套件，這些介面極少變更（如果有的話），因此不需要隨 Rx.NET 的每個版本推出新版本。然而，這一點在某些時候被遺忘了，並且它與所有其他 Rx.NET 套件一起獲得了版本更新，從而失去了該套件的初衷。所有介面都已存在於 `System.Reactive` 中多年。
* [`System.Reactive.Linq`](https://www.nuget.org/packages/System.Reactive.Linq) 曾經是各種 Rx.NET 型別的所在地，這些型別現在是核心功能，並且永遠不會移出 `System.Reactive`：LINQ 運算子、subject、join 模式、事件模式、`Task` 整合、時間戳記、虛擬排程器
* [`System.Reactive.PlatformServices`](https://www.nuget.org/packages/System.Reactive.PlatformServices)：曾是「平台啟發」功能的所在地，這些功能可追溯到在 [2021 年被棄用](https://github.com/dotnet/modernize-dotnet/issues/437)的舊版 PCL（可攜式類別庫）系統，並且過去也用於定義依賴於當時並非在所有 .NET 執行階段型別中都可用的多執行緒支援的排程器；這些在很久以前就移到了 `System.Reactive` 中
* [`System.Reactive.Providers`](https://www.nuget.org/packages/System.Reactive.Providers) 曾經是 Rx.NET 的 `IQueryable` 支援所在地，但這早已內建於 `System.Reactive` 中
* [`System.Reactive.Runtime.Remoting`](https://www.nuget.org/packages/System.Reactive.Runtime.Remoting) 定義了 `RemotingObservable`。微軟不再支援 .NET Remoting；我們在 `System.Reactive` 的 .NET Framework 目標中保留了這一點，但現在強烈建議不要使用 remoting。
* [`System.Reactive.Windows.Threading`](https://www.nuget.org/packages/System.Reactive.Windows.Threading) 過去曾嘗試實現撰寫支援多個不同 UI 框架的單一程式碼庫，當時 WPF、Silverlight、Windows 8 ('Metro') 和 Windows Phone 同時可用，並且對「dispatcher」有著非常相似但略有不同的想法。即使在發揮作用時，這個特定套件背後的封裝概念也令人困惑，現在已成為歷史遺跡；開發人員現在應該為其設定目標的任何 UI 框架挑選合適的 UI 整合套件

請注意，這些套件多年來一直是 facade：

* `System.Reactive.Windows.Forms`
* `System.Reactive.WindowsRuntime`

在 Rx 7 中，這些套件已回歸其最初的角色：它們現在是 Rx 中 Windows Forms 和 WinRT 支援的所在地。（我們沒有恢復 `System.Reactive.Windows.Threading` 套件，因為它的名稱在某種程度上是歷史的偶然，並沒有太大幫助。WPF 功能現在位於新的 `System.Reactive.Wpf` 元件中，同樣地，UWP 功能位於新的 `System.Reactive.Uwp` 套件中。）
