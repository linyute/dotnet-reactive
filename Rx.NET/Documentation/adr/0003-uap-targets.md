# 在不使用 MSBuild.SDK.Extras 的情況下建構 uap10.0 (UWP) 目標

Rx.NET 不能再依賴 [MSBuild.SDK.Extras](https://github.com/novotnyllc/MSBuildSdkExtras)，因為該專案已不再維護。本文件說明了為什麼 Rx.NET 一直在使用它，以及我們如何擺脫對它的依賴。

簡而言之，Rx.NET 使用 `MSBuild.SDK.Extras` 透過提供 `uap10.0` 目標來支援 UWP。現代 .NET 專案不直接支援此 TFM——新的 UWP 專案仍在使用早於 .NET Core 的舊專案格式。截至 .NET 9.0，沒有官方方法可以建構包含 `uap10.0` 目標的多目標 .NET NuGet 套件。（9.0 SDK 現在確實支援為 UWP 進行建構，但這是透過新增在 UWP 中使用 .NET 9.0 的支援來啟用的。仍然沒有辦法針對較舊的 `uap` TFM 進行建構。）多年來，我們使用 [MSBuild.SDK.Extras](https://github.com/novotnyllc/MSBuildSdkExtras) 專案解決了這個問題，該專案由曾經也是 Rx.NET 專案維護者的人編寫和維護。該人員已不再參與 Rx.NET 的工作，而且多年來沒有人對 `MSBuild.SDK.Extras` 進行任何開發。

當我們開始為 .NET 9.0 的到來做準備時，很明顯是時候停止使用 `MSBuild.SDK.Extras` 了。

## 狀態

已接受


## 作者

@idg10（[Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/)）。


## 背景

[UWP](https://en.wikipedia.org/wiki/Universal_Windows_Platform) 是微軟在 Windows 10 中引入的 API，用於支援建構 Windows 傳統型應用程式（以及當時的 Windows Phone 應用程式）。微軟現在不鼓勵使用它，而是鼓勵開發人員改用 [WinUI](https://learn.microsoft.com/en-us/windows/apps/winui/)。然而，儘管 UWP 不再處於積極開發中，它仍然繼續受到支援。Visual Studio 2022（撰寫本文時的 Visual Studio 當前版本）包含 UWP 工具。

自 2016 年的 v3.0.0 以來，Rx.NET 就一直支援 UWP 應用程式。當 UWP 首次推出時，它只能使用提供 `uap10.0` TFM 的 NuGet 套件。此時，Rx 核心功能位於 `System.Reactive.Core` 中，該套件的 [v3.0.0](https://www.nuget.org/packages/System.Reactive.Core/3.0.0) 包含 `uap10.0` TFM。UI 專屬功能（特別是支援 UWP dispatcher 的排程器）位於 `System.Reactive.WindowsRuntime` 中，這是歷史上支援 Windows 8 時代 Windows 市集應用程式以及 Windows Phone 8 應用程式的同一個套件。UWP 專屬功能作為該現有套件中的新目標實作，因為 UWP 中的 dispatcher 型別設計為看起來與其他 API 中的等效型別非常相似。可以撰寫以所有三個平台為目標的原始程式碼。

2017 年，[UWP 獲得了對 .NET Standard 2.0 的支援](https://devblogs.microsoft.com/dotnet/announcing-uwp-support-for-net-standard-2-0/)。這出現在「Windows 10 秋季創作者更新」（也稱為版本 1709）中。為此對應的 SDK 版本為 10.0.16299。此時，UWP 應用程式不再僅限於提供明確 UWP 支援的 NuGet 套件。任何具有 `netstandard2.0` 目標的 NuGet 套件都可以使用。然而，並非每個人都會立即升級到最新版本的 Windows。前一個版本 1703（「創作者更新」）一直受到支援，直到 2019 年 10 月。

（Windows 10 的更早版本 1607 在[長期維護通道（LTSC）](https://learn.microsoft.com/en-us/windows/release-health/release-information)上仍然可用，目前預計將持續受支援直到 2026 年 10 月。因此，即使在我在 2024 年 10 月撰寫本文時，執行*不*支援 .NET Standard 2.0 之 UWP 版本的 Windows 版本仍將獲得另外 2 年的支援！然而，由於 Rx.NET 不是微軟產品，我們不會嘗試提供這種延長生命週期的支援。想要在如此古老的版本上使用 Rx.NET 的人可以自由選擇使用舊版 Rx.NET，或者 fork 自己的版本。）

因此在實務上，我們至少需要在 2019 年 10 月之前繼續為主要 Rx.NET 功能提供 `uap10.0` 目標。而且提供 UWP 專屬功能的 Rx.NET 元件即使在現在也需要透過 `uap10.0` 目標來提供，因為平台專屬功能在 `netstandard2.0` 上不可用。

因此在 2019 年 10 月，為 Rx.NET 核心捨棄 `uap10.0` 支援將變得合理（理由是當時處於主流支援中的所有 Windows 版本都完全能夠在 UWP 中使用 `netstandard2.0` 版本）。到這時，Rx.NET 已經處於 v4.2.0，因此這是在將所有 Rx.NET 功能合併為單一 `System.Reactive` 套件的「大統一」之後的一段時間。儘管這解決了一些問題，但隨後引起了巨大的痛點。

「大統一」造成的問題之一是，雖然 UWP 已經能夠使用 `netstandard2.0`，但已無法從 `System.Reactive` 套件中移除 `uap10.0` 目標。這是因為過去位於 `System.Reactive.WindowsRuntime` 中的 UWP 專屬功能（例如提供 UWP dispatcher 支援的排程器）已移至 `System.Reactive` 中。它們僅編譯到 `uap10.0` 目標中，但如果我們將其移除，維護 UWP 應用程式並升級到最新 `System.Reactive` 的任何人都會發現他們的程式碼不再運作。如果幸運的話，他們會收到編譯時期錯誤，但在複雜的 NuGet 相依性樹中，在應用程式開發人員從未明確選擇使用 Rx.NET 的情況下，依賴多個各自依賴不同版本 Rx.NET 的套件並不罕見。如果升級某些其他套件將他們升級到不再提供 `uap10.0` 目標的 `System.Reactive` 新版本（意味著他們在較新版本中使用 `netstandard2.0` 目標），他們可能會遇到執行階段錯誤，抱怨從使用 Rx 的某個函式庫深處出現找不到類別的錯誤，而應用程式作者先前甚至可能根本不知道他們對 Rx 有這個可轉移相依性。

因此我們繼續為 `System.Reactive` 和其他一些元件提供 `uap10.0` 目標。

[MSBuild.SDK.Extras](https://github.com/novotnyllc/MSBuildSdkExtras) 使這成為可能，但自 2021 年 12 月以來它就沒有再更新過，這意味著依賴它是成問題的。

微軟已宣布 [.NET 9.0 最終將獲得對 UWP 的一些支援](https://devblogs.microsoft.com/ifdef-windows/preview-uwp-support-for-dotnet-9-native-aot/)，但不幸的是這對我們幫助不大。我們仍然需要提供 `uap10.0` 目標，以免破壞尚未升級到 .NET 9.0 的現有應用程式開發人員，並且這個新工具鏈不支援針對該目標進行建構。（相反地，它啟用了在 UWP 上使用 .NET 9.0 執行階段，這意味著以這種新方式建構的應用程式可以直接使用 Rx 的一般 .NET 目標。）

因此，我們需要找到一種方法來做類似 `MSBuild.SDK.Extras` 所達成的事情：即提供說服 .NET SDK 建構合適目標所需的設定。

## 決策

首先，我們不能只設定 `<UseUwpTools>True</UseUwpTools>` 來使用 .NET 9.0 SDK 中的新 UWP 支援，因為如上所述，這不支援為 `uap10.0` TFM 進行建構。

我們將執行以下操作：
* 從所有 `csproj` 檔案和 `global.json` 檔案中移除 `MSBuild.SDK.Extras` 的所有使用
* 根據需要設定 MSBuild 變數，以使 `uap10.0.18362` 目標能夠在編譯時不出錯
* 新增所需的中繼資料參考，以使 `uap10.0.18362` 目標能夠在編譯時不出錯

我們將*不會*執行以下操作：
* 嘗試匯入建構 UWP 應用程式時通常使用的完整 `.props` 和 `.targets` 檔案集

我們的目標*不是*盡可能忠實地重新建立 C# 程式碼在傳統 UWP 專案中所擁有的建構環境。（這*確實*差不多就是 `MSBuild.SDK.Extras` 試圖要做的事情，因為它試圖對遇到此問題的任何專案都普遍有用。）我們的目標是繼續能夠建構像往常一樣在 UWP 系統上運作的二進位檔案。因此，我們只需要做達成該目的所需的最低限度工作。例如，我們不需要任何 XAML 支援。

在實務中實現這一點需要我們解決一些問題。以下各節提供了一些重要的背景資訊，然後繼續說明已經完成的工作及其原因。

### 背景

接下來的幾節將說明理解該設計所需的一些背景資訊。

#### TFM 與平台版本

近期版本針對 UWP 所設定目標的精確 TFM 是 `uap10.0.18362`。版本號碼決定了程式碼可用的 UWP API 版本。我們在 2023 年 3 月將其移至此版本（從 `uap10.0.16299`），作為在 Visual Studio 2022 上建構 Rx.NET 所需工作的一部分。這是一個非常保守的選擇：這是我們可以使用且適用於目前工具鏈的最低版本。在我們這樣做的時候，不支援 18362 的最新 Windows 版本已經停止支援好幾年了。

隨著這次停止使用 `MSBuild.SDK.Extras` 的最新更新，我們認為沒有任何理由進行變更。Visual Studio 2022 仍然是最新版本，並且它繼續支援 10.0.18362 作為目標版本。

請注意，特定版本 TFM 中的 Windows 版本並未嚴格決定作業系統的最低版本。它決定了您針對哪個作業系統 API 版本進行建構，因此實際上它決定了您可以*嘗試*使用的作業系統 API 的新舊程度，但在技術上可以撰寫嘗試使用作業系統 API 但在該 API 不存在時正常降級行為的程式碼。因此，元件可以指定比其 TFM 中的版本更低的 `TargetPlatformMinVersion`。

以下版本值得關注。關於後續版本狀態的說明在 2025/06/05 時是正確的：

* `10.0.16299`：向 UWP 新增 `netstandard2.0` 支援的版本；若不是因為下一個項目，這將是我們的下限
* `10.0.17763`：支援 C#/WinRT 的最舊版本，因此是我們的絕對下限（又稱 1809，此版本仍處於延伸支援中）
* `10.0.18362`：Visual Studio 2022 支援的最舊 SDK 版本
* `10.0.19041`：Rx 的 `windows-` TFM 設定為目標的版本；我從未查明為什麼是這個特定版本，儘管這可能僅僅是因為在向 Rx.NET 新增 .NET 5 支援時，這個 Windows 版本（又稱 2004，又稱 20H1，於 2020 年 5 月推出）是目前的最新版本
* `10.0.19045`：Windows 10 22H2，Windows 10 的最後一個版本（支援於 2025 年 10 月終止）
* `10.0.22621`：仍處於正式發布（GA）支援中的最舊 Windows 11 版本（22H2）（僅限企業版；支援於 2025 年 10 月終止）
* `10.0.22631`：針對家用版、專業版和教育版具備 GA 支援的最舊 Windows 11 版本（23H2）（非企業版服務於 2025 年 11 月終止；企業版服務於 2026 年 11 月終止）
* `10.0.26100`：Windows 的最新版本（24H2），也是安裝在 `windows-2025` Azure DevOps 託管建構映像上的版本

因此碰巧的是，我們在技術上不需要任何比 10.0.17763 更具時效性的內容。因此我們可以將其指定為最低平台版本。然而，沒有令人信服的理由這樣做，而且由於 10.0.18362 是目前工具鏈所能完全理解的最早版本，並且是 Rx 6.0 一直設定為目標的版本，因此繼續使用該版本是有意義的。


#### Azure DevOps 建構代理程式

截至 2024/10/24，Azure DevOps Windows 建構代理程式僅安裝了以下 Windows SDK 版本：

* 10.0.17763.0
* 10.0.19041.0
* 10.0.20348.0
* 10.0.22000.0
* 10.0.22621.0

請注意，我們實際想要設定為目標的特定版本 18362 不在其中。這沒問題，因為 19041 SDK 能夠建構以 18362 為目標的應用程式，但這偶爾會導致建構檔案中出現一些令人困惑的設定。（正如我稍後將描述的，在某些地方針對 `10.0.18362` 時我們必須參考 `10.0.190401.0` 資料夾路徑）


### 實作詳細資訊

以下各節說明我們如何啟用將 `uap10.0.18362` 指定為目標架構，即使工具不支援這一點。

專案包含 `Directory.build.props` 和 `Directory.build.targets` 檔案。建構工具會搜尋這些檔案並自動為方案中的所有專案載入它們。`Directory.build.props` 檔案有一個帶有 `Condition` 的 `<PropertyGroup>`，這意味著它僅在建構 `uap10.0.18362` 目標時執行，並且它設定了許多屬性，如下各節所述。


#### 目標平台版本

我們讓最低平台版本與 TFM 中的版本相符：

```xml   
<TargetPlatformMinVersion>10.0.18362</TargetPlatformMinVersion>
<TargetPlatformVersion>10.0.18362.0</TargetPlatformVersion>
```

請注意，有必要同時指定 `TargetPlatformMinVersion` 屬性以及 `<TargetPlatformVersion>`。`Microsoft.NetCore.UniversalWindowsPlatform` NuGet 套件（我稍後將討論）會使用它，如果我們不設定此屬性，會收到許多令人困惑的編譯器錯誤。

預設情況下，SDK 會將 TFM 拆解為描述其元件的各種其他屬性。但是，如果我們讓它對 UAP TFM 這樣做，建構就會失敗。因此我們手動設定所有相關屬性：

```xml
<TargetFrameworkMoniker>.NETCore,Version=v5.0</TargetFrameworkMoniker>
<TargetFrameworkIdentifier>.NETCore</TargetFrameworkIdentifier>
<TargetFrameworkVersion>v5.0</TargetFrameworkVersion>
<TargetPlatformIdentifier>UAP</TargetPlatformIdentifier>
<NugetTargetMoniker>UAP,Version=v10.0</NugetTargetMoniker>
```

如果沒有這些明確的設定，前兩個值將變成 `UAP,Version=v10.0` 和 `UAP`，但這會進一步導致一連串的錯誤，因為 SDK 的其餘部分不知道這些是什麼。「.NETCore」v5.0 與 .NET 5.0 不是同一回事。我相信它對應於簡短的 TFM `netcore50`（而 .NET 5.0 是 `net5.0`）。我認為這是由於 UWP 應用程式的前身 Windows 8 市集應用程式實際上是第一個推出 .NET Core 版本的產物。它們在 .NET Core 1.0 推出之前就使用了 .NET Core，並且令人困惑地將其稱為 `netcore45`（在 Windows 8.1 中為 `netcore451`）。[已棄用的 TFM](https://learn.microsoft.com/en-us/dotnet/standard/frameworks#deprecated-target-frameworks) 清單顯示 `netcore50` 是 `uap10.0` 的舊稱。但顯然 SDK 的某些部分沒有收到該通知，仍然需要使用舊名稱！

然而，只有*部分*屬性應該使用舊名稱。我們需要設定*所有*這些屬性，因為否則建構系統的其他部分會感到困惑（例如 NuGet 處理）。因此我們在某些地方需要「.NETCore」名稱，在其他地方需要「UAP」名稱。

另請注意，套件驗證工具（我們用來確保 `System.Reactive` 繼續呈現與以往相同的 API）最終被證明無法理解 `.NETCore,Version=v5.0` TFM。因此為了使其正常運作，我們需要在建構流程的後續階段將 TFM 還原為原樣，這就是我們擁有此目標的原因：

```xml
<Target Name="_SetUwpTfmForPackageValidation" BeforeTargets="RunPackageValidation">
<ItemGroup>
    <PackageValidationReferencePath Condition="%(PackageValidationReferencePath.TargetFrameworkMoniker) == '.NETCore,Version=v5.0'" TargetFrameworkMoniker="UAP,Version=10.0.18362.0" TargetPlatformMoniker="Windows,Version=10.0.18362.0" />
</ItemGroup>
</Target>
```

這還設定了目標平台 moniker 以指出這是一個 Windows 專屬的 TFM，否則套件驗證工具似乎無法理解這一點。（但我們確實需要在建構早期將目標平台識別碼設為 `UAP` 以使其他各種功能正常運作，這就是為什麼我們只在套件驗證執行之前切換它的原因。）


#### 編譯器常數

當使用受支援的 UWP 建構工具時（使用舊格式的專案系統，我們無法使用該系統，因為我們還需要建構現代目標），會設定 `WINDOWS_UWP` 定義常數，使得編譯到多個目標中的原始程式碼能夠透過 `#if WINDOWS_UWP` 偵測到它正在針對 UWP 進行建構。因此我們在 `Directory.build.props` 中需要這個：

```xml
<DefineConstants>$(DefineConstants);WINDOWS_UWP</DefineConstants>
```


#### 套件與中繼資料

通常，當您指定 TFM 時，.NET SDK 會計算需要哪些框架函式庫參考並為您新增它們。因此，如果您在專案檔中撰寫 `<TargetFramework>net8.0<TargetFramework>`，您將自動存取所有 .NET 8.0 執行階段函式庫。但由於 .NET SDK 不支援 UWP，這根本行不通。因此我們需要做三件事。

首先，我們需要在 `Directory.build.props` 中設定此屬性：

```xml
<NoStdLib>True</NoStdLib>
```

如果沒有這個，建構工具會嘗試新增對 `mscorlib.dll` 的參考，但它們似乎沒有意識到 a) 這是錯誤的做法，以及 b) 它們實際上沒有該檔案的正確位置，因此參考最終變成了 `\mscorlib.dll`（即它在硬碟的根目錄上尋找）。

其次，我們需要在 `Directory.build.targets` 中加入一個包含以下內容的 `ItemGroup`：

```xml
<PackageReference Include="Microsoft.NETCore.UniversalWindowsPlatform"
                  Version="6.2.12" />
```

這提供了對 .NET 執行階段函式庫元件的參考。（因此這提供了您通常從 SDK 自動獲得的行為，但我們透過設定 `NoStdLib` 屬性停用了該行為。我們需要這樣做，因為 SDK 不知道針對 UWP 專案執行此操作的正確方法。）

因此這使得一般 .NET 程式碼能夠編譯。然而，Rx.NET 還包含使用某些 UWP 專屬 API 的程式碼。（畢竟，我們在此處理的問題在很大程度上是由於支援 UWP dispatcher 的排程器等功能而存在的。）為了使其正常運作，編譯器需要存取包含這些 API 中繼資料的 `.winmd` 檔案。因此我們有以下內容：

```xml
<ReferencePath Include="$(TargetPlatformSdkPath)UnionMetadata\10.0.26100.0\Windows.winmd" />
```

這依賴於設定 `TargetPlatformSdkPath` 建構變數。在本地建構時（無論是在 Visual Studio 中，還是從命令列使用 `dotnet build`），此變數都設定正確，但出於某種原因，它似乎未在建構代理程式上設定。因此我們在 `azure-pipelines.rx.yml` 建構管線定義中將其設定為環境變數。

您可能會對其中的 26100 感到好奇。為什麼那不是與 TFM 一致的 18362？這是因為，如前所述，Azure DevOps Windows 建構代理程式僅安裝了某些 Windows SDK 版本。它們沒有 18362，但 `windows-2025` 映像確實具有 26100 版本，我們可以使用它來針對 `10.0.18362`。


#### 防止過度積極的 WinRT Interop 程式碼產生

.NET SDK 具有一項功能，可以產生 .NET 型別的 WinRT 版本，以啟用 .NET 和 WinRT 程式碼之間的 Interop。不幸的是，我們為了能夠為 `uap10.0.18362.0` 進行建構而設置的方式似乎導致它為實作 `IDisposable` 的任何 .NET 類別產生這些 Interop 型別！這毫無幫助。因此我們在 `Directory.build.targets` 中停用了該功能：

```xml
<CsWinRTAotOptimizerEnabled>false</CsWinRTAotOptimizerEnabled>
```

這應該不會影響使用此函式庫的專案。此設定僅影響建構 Rx.NET 本身時工具的行為，並且據我們所知，我們不需要這個特定的 Interop 功能。


#### 實作限制

首先，請注意任何具有 UWP 的專案都必須使用複數的 `<TargetFrameworks>` MSBuild 屬性。這包括以它為目標且不包含其他目標的 `System.Reactive.Windows.Runtime` 專案。通常在這種情況下您會使用單數的 `<TargetFramework>`，但這會使建構進入不同的模式，並且我們為啟用 UWP 目標所採取的措施僅在多目標模式下有效。


## 後果

我們不再依賴不受支援的建構擴充功能。我們現在可以使用 `dotnet build` 進行建構。過去必須使用 `msbuild` 從命令列進行建構。我們現在適當地支援 .NET 9.0，因為我們在其上執行測試。
