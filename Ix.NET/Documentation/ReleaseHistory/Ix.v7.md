# Ix 7.0 發行歷史

## v7.0.1

由於 `System.Memory` v10.0.0 存在已知弱點，且部分 Ix.NET 元件透過 `System.Linq.AsyncEnumerable` 套件間接相依於此版本，因此有必要發行使用 `System.Linq.AsyncEnumerable` v10.0.6 的 Ix.NET 更新，以避免相依於 `System.Linq.Async` 時導致相依於這個存在弱點的 `System.Memory` 版本。

## v7.0.0

Ix.NET 的 `System.Linq.Async` 即將被淘汰。相反地，您應該使用 .NET 執行階段的 [`System.Linq.AsyncEnumerable`](https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/10.0/asyncenumerable) 函式庫（在 .NET 10.0 中新增，但可用於較舊的執行階段）。如果您正在使用 `System.Linq.Async` 中尚未包含在 .NET 的 `System.Linq.AsyncEnumerable` 中的功能，您可以新增對 `System.Interactive.Async` 的參考，該處是此功能的新歸宿。

請注意，`System.Linq.Async` 將繼續運作。LINQ 實作已從 `System.Linq.Async` 的公開 API 中移除，因此當專案升級至 v7 時，先前針對 `IAsyncEnumerable<T>` 使用 `System.Linq.Async` 之 LINQ 實作的任何程式碼，現在都將使用由 .NET 10 提供的 `System.Linq.AsyncEnumerable`。（即使您的專案以較舊版本的 .NET 為目標也能運作，因為新的 `System.Linq.AsyncEnumerable` 函式庫支援這些較低版本的目標，且 `System.Linq.Async` 為較舊的目標提供了對它的遞移參考。）

透過繼續在執行階段函式庫中提供完整的舊版 LINQ 實作，維持了二進位相容性；我們僅從參考組件中移除了此實作。這使得 `System.Linq.Async` 已淘汰的 LINQ 實作對編譯器隱形，但它在執行階段仍然可用。（這確保了如果專案升級至 v7，但仍使用針對較舊版本建構的元件，這些元件不會因為 LINQ 實作不在它們預期的元件中而在執行階段遇到例外狀況。）

請注意，如果專案使用的是 `System.Linq.Async` v6，並將其目標架構升級至 .NET 10，將會遇到建構錯誤。編譯器將能存取 `IAsyncEnumerable<T>` 的兩種 LINQ 實作，並會回報模稜兩可錯誤。快速修復方法是升級至此函式庫的 v7 版本。長期修復方法是移除對此函式庫的所有參考。（以較舊版本 .NET 為目標的專案也可能發生這種情況。如果使用 `System.Linq.Async` 的專案最終也取得了對 `System.Linq.AsyncEnumerable` 的相依性，編譯器同樣會在您嘗試對 `IAsyncEnumerable<T>` 使用 LINQ 的任何地方回報模稜兩可錯誤。同樣地，短期修復方法是升級至 `System.Linq.Async` 的 v7 版本，而長期修復方法是停止使用它。）我們最終將在 NuGet 上將 `System.Linq.Async` 套件標記為已淘汰。

### 為什麼要遷移？

簡短的答案是，多年來 Ix.NET 一直在填補 .NET 執行階段函式庫中的缺口。.NET 執行階段終於填補了該缺口，因此現在是 Ix.NET 退居幕後的時候了。

從命名上不太明顯的一點是，`System.Linq.Async` 從未獲得微軟的支援。下表可以澄清此情況：

| 函式庫 | 上層專案 | 支援者 |
|---|---|---|
| `System.Linq.Async` | Ix.NET | 無資金贊助的社群成果 |
| `System.Linq.AsyncEnumerable` | .NET 執行階段 | Microsoft |


當 `IAsyncEnumerable<T>` 首次移入 .NET 執行階段類別庫（在 .NET Core 3.0 中）時，並沒有官方支援的 LINQ 實作。`IAsyncEnumerable<T>` 介面源自於 Ix.NET 專案，該專案一直提供 LINQ 實作，因此當 `IAsyncEnumerable<T>` 移入 .NET 執行階段函式庫時，Ix.NET 只是調整其現有的 `IAsyncEnumerable<T>` LINQ 以配合該介面的新歸宿。因此，`System.Linq.Async` 成為了 `IAsyncEnumerable<T>` 的非官方實質 LINQ。然而，微軟並未提供官方支援，也沒有維護預算。最終，微軟認為 .NET 執行階段函式庫確實應該具備內建的 `IAsyncEnumerable<T>` LINQ，於是 `System.Linq.AsyncEnumerable` 誕生了。

（.NET 執行階段類別庫團隊決定定義新的函式庫，而不是接管 Ix.NET 現有的 `System.Linq.Async`，因為類別庫設計指引自 `System.Linq.Async` 建立以來已經改變，且它與這些指引不一致。）