# 使用 .NET SDK 套件驗證以確保回溯相容性

 從 PublicApiGenerator 遷移至 .NET SDK 套件驗證


## 狀態

未決定（Undecided）


## 作者

@idg10 ([Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/))。


## 背景

主要 `System.Reactive` 元件必須保持與舊版的回溯相容性。出於 [ADR-0005](0005-package-split.md) 中描述的原因，有必要從該套件的公開 API 中移除特定於 UI 框架的元件，但我們保留了相同的執行階段 API。

因此我們有兩個略有不同的相容性層級：

* 完整的二進位回溯相容性
* 除特定於 UI 框架的 API 功能外所有內容的原始碼相容性


### 現有 API 相容性測試的問題

多年來，Rx.NET 一直執行旨在驗證公開 API 不會被意外變更的測試。這些測試使用了 `[PublicApiGenerator`](https://github.com/PublicApiGenerator/PublicApiGenerator) 來產生包含組件對外公開 API 的 C# 原始程式檔。例如，這是它產生的輸出片段：

```cs
namespace System.Reactive
{
    public sealed class AnonymousObservable<T> : System.Reactive.ObservableBase<T>
    {
        public AnonymousObservable(System.Func<System.IObserver<T>, System.IDisposable> subscribe) { }
        protected override System.IDisposable SubscribeCore(System.IObserver<T> observer) { }
    }
```

如此處所示，這在技術上不是合法的 C#：編譯器會拒絕該 `SubscribeCore` 方法，因為該方法具有傳回型別但主體沒有 `return` 陳述式。但是，此工具的輸出並非用於編譯：它的工作只是擷取組件定義的對外公開型別和方法，因此它只使用足夠的 C# 語法來執行此操作。

Rx 測試套件包含一個 `Tests.System.Reactive.ApiApprovals` 專案，該專案為各種 Rx.NET 組件產生此形式的檔案，並將其與包含預期結果的檔案進行比較。這防止了我們意外變更公開 API。

這運作得相當好，但它有一些顯著的缺點：

* 僅驗證了 .NET Framework 目標
* 舊版外觀套件未經測試（除非您將略顯奇特的 `System.Reactive.Observable.Aliases` 算作外觀套件，可以說它是，但它提供包裝函式型別而不是型別轉發器）
* 直到最近，`PublicApiGenerator` 函式庫仍無法理解型別轉發器

第一個問題只要做一些工作就可以克服。第二個問題是第三個問題的產物：舊版外觀套件使用 `TypeForwardedToAttribute` 組件層級屬性公開實際定義在其他套件中的公用型別。這些在低階 Metadata 格式中具有完全不同的表示形式，並且透過反映使用時也略顯棘手。旨在描述或比較公開 API 的工具需要採取特殊步驟來正確處理型別轉發器，而直到最近，`PublicApiGenerator` 都沒有這樣做。

儘管 `PublicApiGenerator` 現在確實支援型別轉發器，但我們現在遇到了一個新問題：`System.Reactive` 實際上對每個目標都有兩個 API：

* 執行階段 API（由套件的 `lib` 資料夾中的組件提供），其繼續包含特定於 UI 框架的型別
* 建構階段 API（由套件的 `ref` 資料夾中的組件提供），其排除了特定於 UI 框架的型別

此外，在驗證所有 TFM 的 API 時，我們不僅希望確保每個 TFM 內部從 Rx.NET 的一個版本到下一個版本的一致性，而且還希望盡可能確保各種 TFM 彼此之間保持一致。

### 可能的替代方案

選項：

1. 繼續使用 `PublicApiGenerator`
2. 使用 Microsoft 使用的參考組件產生工具
3. 使用 .NET SDK 內建的套件驗證
4. 使用 `Microsoft.CodeAnalysis.PublicApiAnalyzers`，我們將在其中定義所有已知 API 的明確清單。（例如，Aspire 和 Orleans 使用此方法。）

目前我們尚未決定。我們認為我們將無法使用選項 4，因為 `PublicApiAnalyzers` 似乎也不理解型別轉發器，而當我們認為 `System.Reactive` 將成為舊版外觀套件時，這將是一個大問題。（我們尚未 100% 確定我們使 `System.Reactive` 保持為主要套件的方法是否絕對適用於所有情境，並且我們可能仍不得不退回到將其降級為舊版外觀套件的先前計劃，屆時這將再次變得重要。）

選項 3 有利於確保公開 API 符合語意化版本控制：它將阻止我們意外移除或變更現有 API。但它有一個缺點：它沒有提供刻意新增新項目的好方法。預設情況下，基準檢查會驗證沒有變更或移除任何現有功能，但如果主要或次要版本號碼已增加，則將允許新功能。

這裡的問題在於它不會告訴我們是否*意外*新增了新的公開 API 功能。

您可以啟用回報任何類型變更的嚴格模式，這將標記新增以及移除。這可以防止意外新增，但是當您打算新增項目時應該怎麼做？我們可以新增抑制項目以防止在新增項目時發出警告，但當意圖是新增公用成員時，這是一種笨拙的機制。舊的 `PublicApiGenerator` 方法在這裡有一個優勢：存在一個精確反映我們預期 API 應該是什麼的特定產物。當新增新項目時，我們始終會編輯該檔案，這意味著新增了新的 API 功能始終是非常明確且有意為之的。

選項 4 可能是更好的選擇，但在我們確信其缺乏型別轉發器支援不會成為問題之前，我們無法承諾這一點。

選項 2 有其吸引人之處。Microsoft 擁有可產生用於產生參考組件的可編譯原始程式碼的工具。由於它用於建構 .NET 執行階段類別函式庫，我們知道它是一個全面且維護良好的工具。但是，目前似乎只有當您以與 Microsoft 相同的方式建構事物時，它才可用。例如，我們可能需要使用「arcade」建構系統，因為所涉及的一些 NuGet 套件（例如 `Microsoft.DotNet.ApiCompat.Task` 和 `Microsoft.DotNet.GenAPI.Task`）似乎僅在這些建構系統中使用的特殊 NuGet 摘要中可用。此外，檢視這些套件，它們似乎處於永久預覽狀態。如果您是在 .NET 執行階段函式庫原始碼樹中工作的開發人員，及時了解該建構工具是工作的一部分，但對於 Rx.NET 來說踏上那條特定的跑步機可能不是一個好主意。


### 像往常一樣，舊版 UWP 使事情變得棘手

目前我們正在這樣做：

```xml
  <Target Name="_SetUwpTfmForPackageValidation" BeforeTargets="RunPackageValidation">
    <ItemGroup>
      <PackageValidationReferencePath Condition="%(PackageValidationReferencePath.TargetFrameworkMoniker) == '.NETCore,Version=v5.0'" TargetFrameworkMoniker="UAP,Version=10.0.18362.0" TargetPlatformMoniker="Windows,Version=10.0.18362.0" />
    </ItemGroup>
  </Target>
```

我們認為這對於使套件驗證在針對 `uap10.0.18362` 目標時不出錯是必要的。

這可能僅僅是因為我讓 UWP 專案在現代 SDK 中建構的技巧有誤。我可以嘗試從一開始就指定這些 moniker。但是，我依稀記得建構的某些部分實際上依賴於設定錯誤的 TFM。因此，在套件驗證之前修改它的這種技術可能是唯一的方法。


## 決策

* `System.Reactive` 專案使用 `<EnablePackageValidation>` 和 `<PackageValidationBaselineVersion>` 確保與 Rx v6.1 的相容性
* 目前我們繼續使用 `PublicApiGenerator` 工具來防止意外新增新的公開 API 功能

我們需要使用 `<ApiCompatSuppressionFile>`，因為該工具不僅會偵測與基準版本的不一致，還會偵測套件內各目標之間的不一致。由於 Rx 6.1 為 UWP 提供的 `ThreadPoolScheduler` 與為所有其他平台提供的同名型別略有不同，因此 Rx 6.1 已經具有一些內部一致性問題。並且該工具會偵測到我們刻意隱藏特定於 UI 框架的 API 功能，因此我們需要抑制項目來告訴它我們是故意這樣做的。


## 後果

內建的套件驗證意味著我們現在終於驗證了*所有* TFM，這是一項顯著的改進。此外，該工具現在執行內部一致性檢查，因此如果 TFM 之間或 `ref` 與 `lib` 組件之間存在意外差異，我們現在會收到通知，而之前這將不會被偵測到。

由於我們目前繼續使用基於 `PublicApiGenerator` 的測試來防止對 API 的意外新增，因此我們需要為此承擔一些持續的開發負擔。我們更願意轉向 Microsoft 支援的工具，但在確定 `Microsoft.CodeAnalysis.PublicApiAnalyzers` 是否絕對適用於我們的情境之前，我們只能接受現狀。
