# 將核心 `IAsyncEnumerable<T>` LINQ 遷移至執行階段函式庫

.NET 10.0 在執行階段類別庫中為 `IAsyncEnumerable<T>` 提供了 LINQ 支援。這實際上使得 `System.Linq.Async` 的大部分內容不再需要。然而，要讓此函式庫的現有使用者順暢過渡到 .NET 10.0 並非完全簡單直接。本文件說明這將如何運作。

## 狀態

已接受。

## 作者

@idg10 ([Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/))


## 背景

作為歷史的巧合，從 2019 年 .NET Core 3 發行到 2025 年底 .NET 10 發行期間，Rx.NET 存放庫最終成為了 `IAsyncEnumerable<T>` 的實質 LINQ 實作。

這是因為 Rx.NET 實際上是最初開發 `IAsyncEnumerable<T>` 的孵化器。在 .NET Core 3.0 之前，.NET 中並沒有內建這樣的介面，但 Rx _確實_ 將此介面定義為其「.NET 互動式擴充」功能的一部分。（早在 2010 年就已經這麼做了。）它還為該介面實作了常見的 LINQ 運算子。

.NET Core 3.0 定義了自己版本的 `IAsyncEnumerable<T>`，但當時 .NET 團隊並未為其實作 LINQ。由於 Rx.NET 存放庫已經針對其原始版本的 `IAsyncEnumerable<T>` 擁有相當完整的 LINQ 實作，因此將其調整以適應內建於 .NET 中的新版 `IAsyncEnumerable<T>` 相當容易。因此 `System.Linq.Async` 誕生了。

在 .NET 10.0 中，.NET 團隊決定接管這項功能。出於各種原因，他們並未直接採用現有程式碼。（原因之一是 .NET 類別庫設計指引隨著時間演進，而 Rx 的 `System.Linq.Async` 中的某些方法與這些指引不符。）因此，.NET 團隊做出決定，不打算維持與原本源自 Rx.NET 的現有 `System.Linq.Async` 函式庫的向下相容性。相反地，出現了一個新的 `System.Linq.AsyncEnumerable` 函式庫，它定義了對等的功能，但是從頭開始實作，並完全符合目前的 .NET 類別庫設計指引。

大多數 API 變更屬於以下類別之一：

1. 在 `System.Linq.Async` 定義接受 `IComparer<T>` 的方法及不帶 `IComparer<T>` 的相關多載之處，`System.Linq.AsyncEnumerable` 僅定義接受 `IComparer<T>` 的多載，將其設為預設值為 `null` 的選擇性參數
2. 對於某些運算子（例如 `Min`、`Max`、`Sum`），`System.Linq.Async` 定義了直接在數值序列上運作的方法，以及在任何型別的序列上運作的方法（接受額外引數以將每個元素投影為數值）；在 `System.Linq.AsyncEnumerable` 中，這些以投影為基礎的變體要麼名稱不同（例如 `MaxByAsync`），要麼根本不存在（例如 `SumAsync`）
3. `System.Linq.Async` 提供了一些配接器（例如 `ToEnumerable`、`ToObservable`），這些配接器以潛在危險的方式（同步包裝非同步）或嵌入了特定做法觀點的方式（例如 `ToObservable` 未向呼叫者提供任何排程選項）來處理非同步作業；`System.Linq.AsyncEnumerable` 選擇完全不實作這些功能
4. 接受回呼的運算子（例如 `Select` 和 `Where`）可以傳入一般的非非同步回呼（例如 `Select` 的 `Func<TElement, TResult>` 或 `Where` 的 `Func<TElement, bool>`），或是傳入 `async` 回呼，在這種情況下回呼會傳回 `Task<T>` 並且可能支援取消。`System.Linq.Async` 對這些方法使用了不同的名稱：它新增了 `Await` 字尾以及 `WithCancellation` 字尾，以區分回呼接受取消語彙基元的形式。`System.Linq.AsyncEnumerable` 要求所有 `async` 回呼都必須接受取消語彙基元（當然它們可以自由忽略），並且對這些形式不使用不同的名稱。例如，`System.Linq.AsyncEnumerable` 只是提供了 `Where` 的額外多載，來取代 `System.Linq.Async` 的 `WhereAwait`。

也有少數功能根本未被重現的情況。例如，`System.Linq.Async` 提供了 `AsAsyncEnumerable` 以啟用刻意的型別抹除。

`System.Linq.Async` 還定義了一些未在 `System.Linq.AsyncEnumerable` 中複製的介面。`System.Linq.Async` 定義了 `IAsyncGrouping` 作為 `GroupBy` 的傳回型別。`System.Linq.AsyncEnumerable` 僅使用 `IAsyncEnumerable<IGrouping<TKey, TElement>>`，這並不完全相同：這允許對群組序列進行非同步反覆運算，但每個個別群組的內容並非可非同步列舉的。`IAsyncGrouping` 允許對兩者進行非同步列舉。在實務上，`System.Linq.Async` 並未利用這一點：它在傳回第一個群組之前完整列舉了整個來源清單以將項目拆分為各群組，因此儘管它強制您在兩個層級都進行列舉（例如，使用巢狀 `await foreach` 迴圈），但在現實中實際上只有外層是非同步的。因此這個介面增加了複雜性卻沒有帶來實質好處。另外還有 `IAsyncIListProvider<T>`，這個介面可以說一開始就不應該是公開的，它僅用於啟用某些內部最佳化。（顯然它在 `System.Linq.Async` 中是公開的，因為它也在 Ix.NET 的其他部分中使用。） 

更複雜的是，`System.Interactive.Async` 中的某些方法與 `System.Linq.AsyncEnumerable` 中的方法發生衝突。例如 `MaxByAsync` 和 `MinByAsync`。最初 `MinBy` 和 `MaxBy` 是 Rx.NET 和 Ix.NET 所獨有的。但 .NET 6.0 在 LINQ to Objects 中新增了具有這些名稱的運算子。令人困惑的是，它們略有不同：Rx.NET 和 Ix.NET 版本認識到可能不會只有單一的最小值或最大值，因此提供處於最大值之所有項目的集合，但 .NET 執行階段類別庫版本僅任意挑選一個勝出者。因此在當時，`System.Interactive` 將其版本重新命名為 `MinByWithTies` 和 `MaxByWithTies`。不幸的是，`System.Interactive.Async` 中並未進行相同的變更，因此我們現在在 `System.Linq.AsyncEnumerable` 上遇到了相同的情況：.NET 執行階段類別庫現在為 `IAsyncEnumerable<T>` 定義了 `MinByAsync` 和 `MaxByAsync` 擴充方法，這些方法接受與 `System.Interactive.Async` 中的方法相同的引數，但具有不同的傳回型別，並且具有不同的行為！

另一個需要考量的重要點是，儘管 `IAsyncEnumerable<T>` 的 LINQ _主要_ 由擴充方法組成，但仍有一些靜態方法。（例如，.NET 函式庫實作了 `AsyncEnumerable.Range`，而沒有實作 `AsyncEnumerable.Create`。）對於擴充方法，只要個別方法不衝突，編譯器對於不同組件中定義擴充方法的多個同名型別不會有問題。然而，非擴充方法則會產生問題。如果 `System.Linq.Async` 繼續定義公開的 `AsyncEnumerable` 型別，那麼對 `AsyncEnumerable.Range` 的呼叫將無法編譯：即使只有單一的 `Range` 方法（由新的 `System.Linq.AsyncEnumerable` 提供），這也將無法編譯，因為 `AsyncEnumerable` 本身是一個模稜兩可的類別名稱。因此，`System.Linq.Async` v7 的公開 API 必須不定義 `AsyncEnumerable` 型別。這對我們在原始碼層級相容性方面能做到的程度施加了某些限制。（二進位相容性不是問題，因為執行階段組件可以繼續定義此型別。）

由於該限制要求我們定義一個新型別來容納所有已廢棄的擴充方法（我們將其稱為 `AsyncEnumerableDeprecated`），這產生了一個新問題：執行階段 API 需要繼續提供所有這些方法作為 `AsyncEnumerable` 的成員（以提供二進位相容性），但針對 Ix.NET v7 新編譯且繼續使用這些已淘汰方法的任何程式碼（因此容忍或隱藏了淘汰警告），現在最終將建構出預期這些方法位於 `AsyncEnumerableDeprecated` 類別中的程式碼。因此，我們需要在執行階段組件中提供所有這些方法_兩次_：一次出於二進位相容性作為 `AsyncEnumerable` 的成員（我們在建構時期完全隱藏該類別），另一次作為 `AsyncEnumerableDeprecated` 的成員（我們新增此類別是為了以不會導致模稜兩可型別名稱錯誤的方式，為使用已淘汰方法的程式碼提供原始碼層級的向下相容性）。


## 決策

下一個 Ix.NET 發行版本將會：

1. 在 `System.Linq.Async` 中新增對 `System.Linq.AsyncEnumerable` 和 `System.Interactive.Async` 的參考
2. 從 `System.Linq.Async` 與 `System.Interactive.Async` 的公開可見 API（ref 組件）中移除存在直接替代方案的所有 `IAsyncEnumerable<T>` 擴充方法（對於新的 .NET 執行階段函式庫方法實際上具有略微不同功能的情況，新增 `MinByWithTiesAsync` 和 `MaxByWithTiesAsync`）
3. 在公開 API（參考組件）中將 `AsyncEnumerable` 重新命名為 `AsyncEnumerableDeprecated`（舊名稱將保留在執行階段組件中以實現二進位相容性），以避免因同一命名空間中存在兩個 `AsyncEnumerable` 定義而產生錯誤
4. 為 `AsyncEnumerableDeprecated` 的成員新增 [Obsolete] 屬性，這些成員在 `System.Linq.AsyncEnumerable` 中提供了需要修改程式碼才能使用的替代方案（例如，由 `Where` 的多載所取代的 `WhereAwait`）
5. 屬於糟糕構想且可能根本不應該存在的 `AsyncEnumerable.ToEnumerable` 方法已被標記為 `Obsolete` 且不會被取代；請注意，雖然 `ToObservable` 存在導致 .NET 團隊決定不複製它的問題，但主要問題在於它嵌入了特定觀點，而非存在根本性損壞，因此我們並未將 `ToObservable` 納入此類別
6. `AsyncEnumerable` 的其餘方法（`System.Linq.AsyncEnumerable` 未提供對等方法之處）將從 `System.Linq.Async` 的公開可見 API 中移除，並由 `System.Interactive.Async` 中的 `AsyncEnumerableEx` 定義完全相同的替代方法
7. 將 `IAsyncGrouping` 標記為已廢棄
8. 將公開的 `IAsyncIListProvider` 標記為已廢棄，並定義非公開版本以供 `System.Interactive.Linq` 內部繼續使用
9. 繼續在 `lib` 組件中提供完整的 `System.Linq.Async` API 以提供二進位相容性
10. 在執行階段 `System.Linq.Async` 組件中提供一個外觀，在 `AsyncEnumerableDeprecated` 型別上複製舊版 `AsyncEnumerable` 方法，以便針對 `System.Linq.Async` v7 建構並選擇繼續使用標記為 `[Obsolete]` 之方法的程式碼可以在執行階段找到這些方法
11. 將 `System.Linq.Async` NuGet 套件標記為已廢棄，並建議改為使用 `System.Linq.AsyncEnumerable` 和/或 `System.Interactive.Async`

其主要效果是，針對 `IAsyncEnumerable<T>` 使用 `System.Linq.Async` 之 LINQ 實作的程式碼，在針對此新版 `System.Linq.Async` 重新建構時，在大多數情況下現在都將使用 .NET 執行階段函式庫實作。

如果使用 .NET 10，開發人員可能會發現他們只需移除對 `System.Linq.Async` 的參考。（如果使用較早版本的 .NET 或 .NET FX，可以將其替換為對 `System.Linq.AsyncEnumerable` 的參考。）如果他們先前使用了任何 `XxxAwaitAsync` 和 `XxxAwaitWithCancellationAsync` 方法，則必須變更這些呼叫以使用新的對等多載。

如果開發人員使用的是 `System.Linq.AsyncEnumerable` 中未提供的 `System.Linq.Async` 功能，他們仍然應該移除 `System.Linq.Async` 參考（因為我們將淘汰該套件），但他們將新增對 `System.Interactive.Async` 的參考。例如，雖然 `System.Linq.AsyncEnumerable` 定義了 `AverageAsync`，但它並未提供與先前 `System.Linq.Async` 相同範圍的功能：接受選取器（同步與非同步）的多載。這些方法在 `System.Linq.Async` 中變為隱藏（僅用於二進位相容性），並且它們已移至 `System.Interactive.Async` 中的 `AsyncEnumerableEx`。`System.Linq.Async` 現在新增了對 `System.Interactive.Async` 的遞移參考，以確保在人們更新其 NuGet 參考之前持續維持原始碼相容性。

使用我們可能根本不應該提供之方法（同步包裝非同步的方法，例如 `ToEnumerable`）的開發人員，將只能透過保留對已淘汰之 `System.Linq.Async` 套件的參考並忽略或隱藏廢棄警告來使用這些方法。我們的立場是，這些開發人員應該尋求其他做法。或者，如果他們堅持進行同步包裝非同步但希望從程式碼中消除廢棄/淘汰警告，他們將必須編寫自己的這些方法版本。

總結來說，先前由 `System.Linq.Async` 提供的每項功能將屬於以下類別之一：

* 在 `ref` 組件中隱藏的方法，可在 `System.Linq.AsyncEnumerable` 中取得
* 在 `ref` 組件中隱藏的方法，可在 `System.Interactive.Async` 中取得
* 可見但標記為 `Obsolete` 的方法，在 `System.Linq.AsyncEnumerable` 中有全新但略有不同的對等項目可供使用

### TFM

我們希望在此版本中保持所有 Ix.NET 套件的 TFM 完全相同，因為 Ix.NET v7 存在的唯一原因就是為了解決 `System.Linq.AsyncEnumerable` 的新存在。

這其中存在一個問題。如果專案具有 `net6.0` 目標並嘗試使用 `System.Linq.AsyncEnumerable`，它會產生建構警告，指出在該執行階段上不受支援。儘管我們不希望出現此建構警告，但我們目前打算對此不做任何處理，因為我們認為更動 TFM 可能會產生非預期的後果。

如果事實證明這確實會導致問題，我們將重新檢視並進行新版本的發行。

## 後果

維持了二進位相容性：針對 `System.Linq.Async` v6 建構但在執行階段對應至 v7 執行的任何程式碼，應繼續如先前一樣正常運作。

編寫為使用 `System.Linq.Async` v6 且升級至 .NET 10 的程式碼，在 .NET 10 實作與 `System.Linq.Async` 具有原始碼相容性的情況下，將自動移至 .NET 執行階段函式庫實作而無需任何程式碼變更。使用 .NET 10 已變更之方法（為了符合目前的類別庫設計規則）的程式碼將繼續正確建構與執行，但編譯器將警告開發人員他們現在正在使用已廢棄的方法，且這些警告將指出建議的替代方案。使用 `System.Linq.Async` 中 .NET 10 選擇不提供對等項目之方法的程式碼，將自動改用 `System.Interactive.Async` 實作而無需任何程式碼變更。由於 `System.Linq.Async` NuGet 套件將被標記為已廢棄，開發人員將知道他們應該停止使用它。如果他們沒有使用任何 `Obsolete` 方法，他們將能夠移除該方法，並且可能需要新增對 `System.Interactive.Async` 的參考。

對於編寫為使用 `System.Linq.Async` v6 但_未_升級至 .NET 10（例如停留在 .NET 8 或 9，或以 .NET Framework 或 .NET Standard 為目標），但由於開發人員主動新增或因為更新了某個元件的新版本而作為新的遞移相依性取得對 `System.Linq.AsyncEnumerable` 相依性的程式碼，情況非常類似。

編寫為使用 `System.Linq.Async` v6 完全不作任何變更，但在 `System.Linq.Async` v7 發行後重新建構的程式碼，將會看到該套件現已淘汰的警告。開發人員可以透過移除該套件並根據需要新增對 `System.Linq.AsyncEnumerable` 或 `System.Interactive.Async`（或兩者皆新增）的參考來修復此警告。