# 如何發布 `System.Reactive` 版本

本文件說明如何發布 `System.Reactive` 及其相關 NuGet 套件的新版本。

有兩種程序：一種適用於主要（major）或次要（minor）版本號碼變更時，另一種則是針對現有 major.minor 版本發布新修補（patch）版本的不同流程。

## 新的 Major.Minor 發布版本

如果我們要提高主要或次要版本號碼（例如，當最新版本為 6.0.1，而我們想要發布 6.1 或 7.0 時），我們執行以下操作：

### 建立新的發布分支

從 `main` 建立一個名為 `rel/rx-v<major>.<minor>` 的新分支，例如：

```
git checkout -b rel/rx-v6.0`
```

### 更新 `version.json`

`Rx.NET/Source/version.json` 檔案在 `main` 上始終指定 preview 標籤。我們在發布分支上將其移除。例如，我們將類似這樣的內容：

```json
  "version": "6.1.0-preview.{height}",
```

變更為這樣：

```json
  "version": "6.1",
```

**注意**：儘管 `main` 指定了語意化版本的所有 3 個數字，但 `nbgv` 在發布版本上的運作方式要求我們僅指定兩個數字。它將根據自上次變更版本號碼以來的認可（commit）次數計數來新增第三個數字。如果我們將其設定為 `6.1.0`，它會將版本設定為 `6.1.0.1`。這不會影響 NuGet 套件版本，但這就是我們意外導致 [6.0.0 發布版本](https://github.com/dotnet/reactive/releases/tag/rxnet-v6.0.0.1) 在 GitHub 中被意外錯誤標記為 6.0.0.1 的原因。

認可此變更。您可以透過從 `Rx.NET/Source` 資料夾執行 nbgv 來檢查是否獲得正確的版本：

```
nbgv get-version
```

**注意**：認可對 nbgv 很重要，這意味著：

* 您需要認可對 `version.json` 的變更，`nbgv get-version` 才能回報正確的版本
* 發布分支上的任何後續認可都將導致更高的版本號碼

第二點意味著對 `Rx.NET/Source/version.json` 的更新應該是發布分支上的*最後一個*認可。如果不是，那意味著您尚未完全準備好建立發布分支，但這也意味著您將獲得例如 6.1.1 而不是 6.1.0。（如果出於某種原因您確實無法避免額外的認可，您可以使用稍後顯示的建立新 Patch 發布版本的技術來調低版本號碼，但通常您不需要為新的 Major.Minor 發布版本執行此操作。）

### 將分支發布到 GitHub

將您的新分支推送至 GitHub。這應該會觸發新的 CI 建構。觀察 Azure DevOps 上的 [Rx.NET-CI](https://dev.azure.com/dotnet/Rx.NET/_build?definitionId=9&_a=summary) 管線。

偶爾，這些建構會無端失敗。例如，如果您查看 [Rx.NET 6.0.1 建構](https://dev.azure.com/dotnet/Rx.NET/_build/results?buildId=105869&view=logs&j=d19fa873-01b0-5cbd-94ee-08d1f56df169)，您會看到 `IntegrationTests` 步驟在 Linux 和 Windows 上都嘗試了兩次。在這兩種情況下，第一次嘗試都在建構代理程式上安裝 .NET 8.0 SDK 的步驟中失敗了。這似乎沒有什麼原因，稍後重新執行失敗的步驟就成功了。

您應該檢查 Azure DevOps 回報的組建編號是否符合您的預期。


### 在 Azure DevOps 中建立 Release

建構完成後，前往 [Rx.NET Deploy](https://dev.azure.com/dotnet/Rx.NET/_release?view=all&_a=releases&definitionId=1) 發布，並建立新的發布以發行該組建。

請注意，對於非預覽組建，此發布的標題將增加版本號碼的額外部分，例如 v6.1.0.1。不用擔心這個。它不會出現在此產生的任何成品中。這只是該發布在設定為納入預覽標籤（如果存在）時的副作用。

### 檢查 Azure DevOps 套件摘要源中的 NuGet 套件

該發布將自動將套件發行到 [RxNet Azure DevOps 套件摘要源](https://dev.azure.com/dotnet/Rx.NET/_artifacts/feed/RxNet)（但尚未發行到 NuGet.org）。您應該在 Visual Studio 中建立一個使用該摘要源的專案，以測試新發布的套件。

請注意，在此階段，這些是真正的套件。它們應該具有非預覽版本號碼，並且已經過簽章。當您在 Visual Studio 中使用它們時，它們的外觀應該與您將它們放在 `NuGet.org` 上後向大眾顯示的外觀完全相同。唯一的區別在於它們來自 Azure DevOps 摘要源。

這是我們決定一切是否出錯的最後機會。（請注意，如果您確實決定放棄該發布，您應該認為該版本號碼已被「作廢（burned）」。修復問題後，您將需要一個新號碼，例如 6.1.1。這令人不太滿意，但 Azure DevOps RxNet 摘要源是公開可見的，因此可以想像有人在某處設法取得了您放在該摘要源中的 6.1.0 套件。為具有相同版本號碼的 NuGet 套件建立多個不同的公開可見版本是不好的做法。）

### 發行至 NuGet.org

如果您的套件通過了冒煙測試（smoke test），您可以核准發布的最後階段。它標記為「NuGet.org」，但它也會建立一個 GitHub release。

一旦完成，唯一要做的事情就是在 NuGet 建立套件索引並將其提供給所有人時耐心等待。


## 新的 Patch 發布版本

當某個現有的 Major.Minor 版本（例如 6.1.0）已經存在時，有時適合建立一個修補（patch）發布版本，其中僅變更版本號碼的第三部分。在語意化版本控制中，這意味著預期的公開介面沒有任何變更。可能存在錯誤修復（這可能需要修復與預期公開 API 行為的非預期偏差）或效能增強。但是沒有新的或修改過的 API，並且除了目前行為不符合預期的情況外，行為沒有任何變更。

### 挑選變更（Cherry Pick）

我們使用現有的 `rel/rx-v<major>.<minor>` 分支。變更（例如錯誤修復或效能增強）將已經作為另一個分支（通常是 `main`）上的一個或多個認可存在。挑選（Cherry pick）實現所需變更所需的盡可能多的認可。如有必要，進行所需的任何修改以使此認可在上下文中運作。（相關變更可能取決於在另一個分支上所做的其他變更，而我們不希望將這些變更納入此修補版本中。修補版本應該盡可能少地進行變更。這些發布分支基本上停留在時間中，除了我們選擇引入的任何錯誤修復或其他變更之外，因此 cherry picking 並非總是能在沒有某些額外變更的情況下運作。）

顯然您應該在此時進行測試。

### 考慮調整 `version.json` 以避免跳過版本號碼

`nbgv` 透過計算自上次導致使用特定版本號碼的認可以來發生的認可次數，自動產生新的修補版本號碼。由於我們位於 `rel/rx-v<major>.<minor>` 分支上，因此 `version.json` 會有變更，因為已經遵循了[前面描述的步驟](#更新-versionjson)。如果您正在為此 major.minor 版本製作第一個修補版本，那通常是發布分支上的最後一個認可。因此，如果您 cherry pick 單個認可到發布分支上，`nbgv` 將看到在修改 `version.json` 的認可之後現在有一個認可，因此它將選擇 `.1` 作為修補版本。

但是，如果您想要納入多個變更，或者如果您需要 cherry pick 多個認可，或者您在 cherry pick 之後必須進行進一步變更以使各部分正常運作，您可能會發現版本號碼高於您的預期。您可能會發現 `nbgv` 選擇了 6.1.3 而不是 6.1.0 後面接 6.1.1，因為它看到了在最後一次變更 `version.json` 之後的 3 個認可。

我們可以透過編輯 `version.json` 進行調整來避免版本編號中出現令人困惑的間隔。如果您查看 [`rel/rx-v6.0`](https://github.com/dotnet/reactive/commits/rel/rx-v6.0) 分支，您會看到認可 [`e29c7a5`](https://github.com/dotnet/reactive/commit/e29c7a50db88513a87651c366088da8b7f40b1f0)，其中 preview 標籤已從 `version.json` 中移除。（您還可以看到我們未能移除修補版本號碼，這就是為什麼 Rx.NET 6.0.0 的 GitHub 發布異常地標記為 6.0.0.1 的原因。）這是將語意化版本固定在 6.0.0 的認可。

6.0.1 版本大約在 9 個認可之後。這是因為我們現在支援 .NET 8.0 測試，所以除了我們實際想要的兩個變更之外，我們還需要將一些建構變更引入該分支。為了避免版本號碼出現較大差距，[認可 `1cfc646`](https://github.com/dotnet/reactive/commit/1cfc6465d1c9c6144d5b4e6420240f2767c8f85c) 新增了 `versionHeightOffset`，將其設定為 -8，以依需求讓我們回到版本 6.0.1。

您不必這樣做，而且從歷史上看，有一些 Rx.NET 發布版本跳過了一個版本。但是保持其循序可能較不易引起混淆，特別是在像這樣變更相當大的情況下。

### 發布

您現在可以依照稍早從[在 Azure DevOps 中建立 Release](#在-azure-devops-中建立-release)開始的步驟，以提供這個新版本。
