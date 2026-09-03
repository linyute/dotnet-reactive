# Ix 5.0 及更早版本的發行歷史

除了原本的 `System.Interactive` 函式庫，以及為 `IAsyncEnumerable<T>` 新增相同功能的 `System.Interactive.Async` 函式庫之外，Ix.NET 專案還包含 `System.Linq.Async`，它為 `IAsyncEnumerable<T>` 定義了與 .NET 執行階段函式庫中為 `IEnumerable<T>` 內建的相同 LINQ 運算子。

## V5.1

移除了各種 `IEnumerable<T>` 的 min 和 max 擴充方法，因為 .NET 6.0 現在已具有內建的對應項目。

修復了導致 `Delay` 重複發出的錯誤。


## V5.0

`System.Linq.Async` 新增了對 C# 8.0 可為 Null 的參考型別功能的支援。


## v4.0

由於 `IAsyncEnumerable<T>` 被新增至 .NET 執行階段函式庫，Ix Async 4.0 與先前版本相比有重大變更。在較早版本中，Ix Async 定義了自己版本的此介面，但 v4.0 已修改為使用現在內建於執行階段中的定義。這使得 Ix Async 中的 `IAsyncEnumerable` 實作能夠被 C# 8 中新增的[非同步串流](https://github.com/dotnet/roslyn/blob/master/docs/features/async-streams.md)語言功能所使用。這代表對於 .NET Standard 2.1 和 .NET Core 3 目標，我們使用 `IAsyncEnumerable<T>` 及其相關的內建介面。在其他平台上，我們使用來自 `Microsoft.Bcl.AsyncInterfaces` 的 `IAsyncEnumerable` 定義，提供 Ix 長期以來為 `IEnumerable<T>` 定義的類似 Rx 的 LINQ 運算子的完整實作。在平台提供系統型別的地方，這些型別將統一為系統型別。

.NET 執行階段函式庫並未為 `IAsyncEnumerable<T>` 新增完整的 LINQ to Objects 實作。雖然 `IEnumerable<T>` 提供了標準運算子，例如 `Where`、`Single` 和 `GroupBy`，從而能夠使用 LINQ 查詢運算式語法，但 .NET 對於 `IAsyncEnumerable<T>` 並未內建提供此功能。由於該函式庫的早期版本已完成大部分相關工作，在其 v4.0 之前的 `IAsyncEnumerable<T>` 版本上實作了這些運算子，因此 Ix 的 v4.0 也建構了 `System.Linq.Async` 函式庫，使 LINQ to Objects 適用於 `IAsyncEnumerable<T>`。