# `System.Interactive` NuGet 套件中 lib 與 ref 之間顯而易見的不相符

`System.Interactive` NuGet 套件包含一般的 `lib` 資料夾，以及用於參考組件的標準資料夾 `ref`。但這兩個資料夾中的目標架構集卻相當不尋常地互不相同。本文件將說明其原因。

## 狀態

既成事實。

這至少可以追溯至 2018 年。本文件（撰寫於 2024 年）的主要目的是解釋為什麼會是現在這個樣子。原本的想法早已淹沒在歲月之中，花費了相當大的心力才理清究竟為何會是如此。本 ADR 旨在避免其他人進行同樣繁重的事後考證。而且這項發現非常重要：如果我們沒有成功逆向工程出此設計選擇背後顯而易見的思維，我們可能就會把它當作錯誤而忽視。（事實上，其目前實作的一部分_確實_是個錯誤。）但事實證明，它以一種不明顯的方式發揮了重要作用。


## 作者

@idg10 ([Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/)) 撰寫了本文件，但他想澄清的是，他並沒有做出該項原始設計決策。他只是將其記錄下來。


## 背景

在撰寫本文時，`System.Interactive` 的目前版本為 6.0.1。如果您檢視該 NuGet 套件，將會發現 lib 與 ref 資料夾具有不同的架構組合：

* `lib`
    * `net6.0`
    * `net48`
    * `netstandard2.0`
* `ref`
    * `net4.8`
    * `net6.0`
    * `netstandard2.1`

  
在 `ref` 中使用 `net4.8` 似乎是一個錯誤：那應該是 `net48`。（我確信這是一個錯誤而不是我們尚未理解的巧妙晦澀技巧，主要原因在於[新增此項目的 2021/12/06 提交](https://github.com/dotnet/reactive/commit/a2410b2267abe193191f3894d243771ae4b126fd)在[另一個套件的參考組件中使用了 `net48`](https://github.com/dotnet/reactive/commit/a2410b2267abe193191f3894d243771ae4b126fd#diff-3b568c93a468dab1b1a619a450bf1c4d88d3ec9539737d09fa6fb7659bc0ae5fR7)，因此這似乎只是一個疏失。）

另一個差異是我們在 `lib` 資料夾中有 `netstandard2.0`，但在 `ref` 資料夾中卻有 `netstandard2.1`。乍看之下，這也非常像是一個錯誤，尤其是當您檢視歷史紀錄時。以下是發行歷史中 `ref` 資料夾首次開始出現 `netstandard2.1` 資料夾的時間點：

* v3.1.1:
    * no reference assemblies
    * folder supports .NET FX 4.5 and .NET Standard 1.0
* v3.2.0
    * reference assemblies for netcoreapp2.0, netstandard1.0, and netstandard2.1
    * lib folder supports .NET FX 4.5 and .NET Standard 1.0, and .NET Standard 2.1

當 v3.2.0 於 2018 年 7 月發行時，最新的 .NET Core 是於 2018 年 5 月發行的 2.1。.NET Standard 2.1 當時還不存在，因為它直到 2018 年 11 月支援它的最舊 .NET (.NET Core 3.0) 發行時才出現。因此初步檢視下，Ix.NET 為根本不存在的目標架構宣告了參考組件！

然而，仔細檢視後，這似乎是刻意為之。檢視此提交：

  https://github.com/dotnet/reactive/commit/0252fb537c9d335b9bc863b65291f152c07ba385

  我們看到 [Ix.NET/Source/refs/Directory.build.props 中的註解](https://github.com/dotnet/reactive/commit/0252fb537c9d335b9bc863b65291f152c07ba385#diff-909504334cbab5c432709c95ae78c24fb2910d850958af2ef6de444b18e5c8ecR6)寫道：
  
> This is here so we can create a fake .NET Standard 2.1 facade

我只能猜測他們知道 .NET Standard 2.1 即將推出，並希望確保 `System.Interactive` 在其發行時已準備就緒。
  
所以這是刻意的。但是為某個平台提供參考組件，卻沒有該平台的任何對應實作，這是一個奇怪的選擇。（雖然當時這只是即將推出的 .NET Standard 版本的佔位符，但在 .NET Standard 2.1 發行後，它仍然保持這種狀態。所有後續的 Ix.NET 發行版本都繼續在 `ref` 資料夾中提供 `netstandard2.1`，而在 `lib` 中沒有相符的資料夾。因此這不只是一項暫時性措施。）這有什麼作用？

Ix 提供的部分功能最終在 .NET Core 中提供，例如 `EnumerableEx.SkipLast`。此方法存在於 Ix.NET 每個 TFM 的實作組件中，但 `netstandard2.1` 和 `net6.0` 參考組件省略了它。這產生的效果是，如果您的目標是任何新到足以將這些方法內建於 .NET 執行階段函式庫中的 .NET 版本，Ix.NET 的對應項目將會：

* 在建構時期看似不存在
* 在執行階段仍然可用

建構時期不可用是很重要的，因為這些是擴充方法。如果相同命名空間中存在相同名稱且適用於相同型別的多個擴充方法，當您嘗試以正常方式使用它們時，就會發生編譯器錯誤。（例如，.NET 定義了 `System.Linq.Enumerable.SkipLast`，而 Ix.NET 定義了 `System.Linq.EnumerableEx.SkipLast`，兩者都被定義為 `IEnumerable<T>` 的擴充方法。若非透過這種 NuGet 套件技巧，任何包含 `using System.Linq;` 的檔案都會將 Ix 與 .NET 版本同時帶入範圍中，而且沒有直接的方法可以指出您需要哪一個。您可以透過使用相當晦澀的 `extern` 別名來解決此問題，但方法完全不明顯。如果任何使用 `System.Interactive` 的人在升級至 .NET Core 3.1 的那一刻遇到這種情況，那將會是非常糟糕的體驗。）

透過安排 `netstandard2.1` 和 `net6.0` 參考組件省略這些方法，Ix.NET 確保 C# 編譯器根本不知道這些方法的存在，從而避免了該問題。但為什麼我們需要為此使用參考組件？為什麼不直接從主要組件中省略這些方法？那是因為您可能相依於針對 `netstandard2.0` 建構的某個函式庫 `OldLib`，例如在該函式庫中 `SkipLast` 是不可用的。`OldLib` 可能使用了 Ix.NET 的 `EnumerableEx.SkipLast`，因此即使您在例如 .NET 8.0 上執行，該方法在執行階段也確實必須存在。.NET 8.0 提供了 `Enumerable.SkipLast`，但如果 `OldLib` 僅以 `netstandard2.0` 形式提供，它將無法存取該方法。它只能使用 `Ix.NET` 的版本。因此該方法在執行階段必須存在。

因此這裡的基本技巧在於，Ix.NET 出於向下相容性考量提供了一個 API 介面範圍，並向以最新 Ix.NET 為目標的新程式碼公佈一個稍小的 API 介面範圍。`lib` 資料夾包含提供前者的完整組件，而 `ref` 資料夾則包含提供後者的參考組件。

這仍然無法明顯解釋為何在 `lib` 不包含 `netstandard2.1` 的情況下，`ref` 包含 `netstandard2.1` 是有用的。如果我理解了這裡的原始設計，其思維是在任何 `netstandard2.1` 可用的執行階段上，像 `Enumerable.SkipLast` 這樣的方法在執行階段函式庫中都是可用的，因此針對 `netstandard2.1` 建構的函式庫應該使用該內建的 `Enumerable.SkipLast`，而不是 Ix.NET 的 `EnumerableEx.SkipLast`。在執行階段，相依於 `System.Interactive` 的 `netstandard2.1` 函式庫很可能會發現自己使用的是 `lib\netstandard2.0` 版本（因為沒有 `lib\netstandard2.1` 版本）。例如：

| `MyLib` TFM | Ix Ref Assembly TFM (for `MyLib` build) | Runtime | Ix Lib Assembly TFM | `SkipLast` used by `MyLib`
|--|--|--|--|--|
| `netstandard2.0` | `netstandard2.0` | .NET Core 2.1 | `netstandard2.0` | Ix |
| `netstandard2.0` | `netstandard2.0` | .NET 6.0 | `net6.0` | Ix |
| `netstandard2.0` | `netstandard2.0` | .NET Framework | `net48` | Ix |
| `netstandard2.1` | `netstandard2.1` | Mono | `netstandard2.0` | .NET Runtime Libraries |
| `netstandard2.1` | `netstandard2.1` | Unity | `netstandard2.0` | .NET Runtime Libraries |
| `netstandard2.1` | `netstandard2.1` | .NET Core 3.1 | `netstandard2.0` (Ix.NET 6.0.1 has no .NET Core 3.1 version so this is the only available option) | .NET Runtime Libraries |
| `netstandard2.1` | `netstandard2.1` | .NET 6.0 | `net6.0` | .NET Runtime Libraries |
| `netcore3.1` | `netstandard2.1` | .NET Core 3.1 | `netstandard2.0` | .NET Runtime Libraries |
| `netcore3.1` | `netstandard2.1` | .NET 6.0 | `net6.0` | .NET Runtime Libraries |
| `net48` | `net48`* | .NET 4.8 | `net48` | Ix |
| `net6.0` | `net6.0` | .NET 6.0 | `net6.0` | .NET Runtime Libraries |

\* although this table represents the design intent, the fact that the reference assemblies were accidentally given the non-existent TFM `net4.8` instead of `net48` means that things didn't work out as described in that row in practice. That mistake meant that for that row, it instead looked like this:

| `MyLib` TFM | Ix Ref Assembly TFM (for `MyLib` build) | Runtime | Ix Lib Assembly TFM | `SkipLast` used by `MyLib`
|--|--|--|--|--|
| `net48` | `netstandard2.0` (oops) | .NET 4.8 | `net48` | Ix |

這之所以未被發現，是因為它碰巧產生了所需的結果：該函式庫使用了 Ix 的 `SkipLast`，而這正是我們在此處需要的，因為 .NET 4.8 沒有該方法。

第一個表格中最有趣的一列是第二列：這是即使執行階段 (.NET 6.0) 確實提供了 `SkipLast`，卻仍使用 Ix 實作的唯一情況。之所以會這樣，是因為該函式庫是針對 `netstandard2.0` 建構的，因此該執行階段提供的 `SkipLast` 根本不是可用介面範圍的一部分（而在函式庫針對 `netstandard2.0` 建構的其他兩列中，它不僅在這些列的編譯時期介面範圍中不存在，在執行階段也不存在，這正是為什麼 `netstandard2.0` 函式庫無法使用它的原因）。這必須能夠運作：在執行階段不提供 `Enumerable.SkipLast` 的環境中編譯、並選擇使用 Ix 的 `EnumerableEx.SkipLast` 的任何函式庫，即使載入到確實提供 `Enumerable.SkipLast` 的較新執行階段中，也必須繼續正常運作。.NET 沒有提供任何方法層級的型別轉發對等機制，因此在這些情況下無法安排讓函式庫取得執行階段的 `Enumerable.SkipLast`，所以它只能使用 Ix.NET 的版本。這就是為什麼當 Ix.NET 載入到提供 `Enumerable.SkipLast` 的執行階段時，它仍然有必要提供 `EnumerableEx.SkipLast`，以防有函式庫需要它。但它也需要確保對於針對將提供 `Enumerable.SkipLast` 之目標編譯的任何程式碼，此 API 看似不存在。


## 決策

我們將繼續使用相同的技術，但有兩項變更：

* 我們將修復 `net4.8`/`net48` 錯誤
* 我們將不再使用 `MSBuild.Extras.SDK`

這種有些不尋常的不相符 `lib` 與 `ref` 資料夾設定先前是由 `MSBuild.Extras.SDK` 處理的，但該套件已不再維護。（自從 .NET 5.0 為最新執行階段以來，它就沒有再更新過。）當我們遷移至 8.0 SDK 時，參考組件的產生停止運作，因此我們必須透過受支援的方式重新建立相同的結果。（這正是為何必須進行 git 歷史考證並導致撰寫本文件的原因。）


## 後果

透過繼續沿用既定做法，以 `netstandard2.0` 為目標的函式庫作者將能繼續使用 Ix.NET 的 `SkipLast`，且它的存在在使用較新的 TFM 時將繼續不會導致難以解決的模稜兩可方法錯誤。透過捨棄 `MSBuild.Extras.SDK`，我們將能夠在 .NET SDK 8.0 上重新建構專案。未來的維護者在初次遇到不相符的 `lib` 和 `ref` 資料夾時，可能仍然會感到困惑，但我們希望本 ADR 的存在能有所助益。