# Rx 發布歷程記錄 v5.0 及更早版本

## v5.0 變更

CSWinRT 的 .NET 5 支援需要 Rx.NET v5；舊版的 Rx 將無法正常運作。若要取得 Windows API，您需要使用最新的 Windows SDK (19041)，不過您可以將目標設定為較早期的 Windows 版本。

**重大變更**
在 Windows 中，由於 `net5.0-windows10.0.19041` 目標現在支援所有 Windows 桌面物件模型，因此在 UWP 上，`DispatcherObservable` 及相關的 SubscribeOn/ObserveOn 已重新命名為 `CoreDispatcherObservable`、`SubscribeOnCoreDispatcher` 和 `ObserveOnCoreDispatcher`。這反映了作業系統的型別名稱。`Dispatcher` 僅指 WPF 型別。

### 支援的平台
Rx 5.x 支援以下平台

- .NET 5
- 包含 Windows 10 19041 SDK 的 .NET 5（能夠以較早期的 Windows 版本為目標）
- .NET Framework 4.7
- UWP 10.0.16299
- .NET Standard 2.0

已移除對舊版 UWP 10.0 的支援。

## v4.0 變更

由於修復 [#205 - 實作組件版本策略](https://github.com/dotnet/reactive/issues/205) 帶來了[極大的](https://github.com/dotnet/reactive/issues/299) [痛苦](https://github.com/dotnet/reactive/issues/305)，我們已將這些函式庫重構為單一函式庫 `System.Reactive`。為防止破壞參考 v3 函式庫的現有程式碼，我們提供了帶有指向新組件之 TypeForwarder 的 facade。如果您擁有針對 v3.0 建構之二進位檔的參考，請使用新的 `System.Reactive.Compatibility` 套件。

#### 支援的平台
Rx 4.1 支援以下平台

- .NET Framework 4.6+
- .NET Standard 2.0+（包含 .NET Core、Xamarin 等）
- UWP

值得注意的是，已不再支援 Windows 8、Windows Phone 8 和舊版 PCL 函式庫。

## v3.0 重大變更
在從 v2.x.x 移至 v3.0.0 的過程中，NuGet 套件變更了其套件命名
 * ~~`Rx-Main`~~ 現在是 [`System.Reactive`](https://www.nuget.org/packages/System.Reactive/)
 * ~~`Rx-Core`~~ 現在是 [`System.Reactive.Core`](https://www.nuget.org/packages/System.Reactive.Core/)
 * ~~`Rx-Interfaces`~~ 現在是 [`System.Reactive.Interfaces`](https://www.nuget.org/packages/System.Reactive.Interfaces/)
 * ~~`Rx-Linq`~~ 現在是 [`System.Reactive.Linq`](https://www.nuget.org/packages/System.Reactive.Linq/)
 * ~~`Rx-PlatformServices`~~ 現在是 [`System.Reactive.PlatformServices`](https://www.nuget.org/packages/System.Reactive.PlatformServices/)
 * ~~`Rx-Testing`~~ 現在是 [`Microsoft.Reactive.Testing`](https://www.nuget.org/packages/Microsoft.Reactive.Testing/)

這使得 NuGet 套件命名符合 NuGet 指引以及每個套件中的主要命名空間。
強式名稱金鑰也已變更，這被視為一項重大變更。
然而，預期不會有 API 變更，因此，一旦您進行了 NuGet 變更，就無需變更任何程式碼。
