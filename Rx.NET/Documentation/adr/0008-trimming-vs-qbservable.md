# 修剪警告與 `IQbservable<T>`

本 ADR 描述了 Rx.NET 用於處理在使用修剪（trimming）時 `IQbservable<T>` 所產生問題的方法。`IQbservable<T>` 依賴於 .NET 運算式樹狀架構，這是一種大量使用反映的機制，在使用修剪時可能會帶來挑戰。

## 狀態

草案（Draft）。


## 作者

@idg10 ([Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/))。


## 背景

`IQbservable<T>` 是 `IObservable<T>` 的導向運算式版本。這類似於 `IQueryable<T>` 是 `IEnumerable<T>` 的導向運算式版本。就像 `IQueryable<T>` 定義了與 `IEnumerable<T>` 可用的相同 LINQ 運算子，但形式為所有 Lambda 都作為運算式樹狀架構傳遞（使查詢提供者能夠檢查查詢的結構，並可能將其轉換為其他內容），`IQbservable<T>` 在響應式世界中也做了同樣的事情。

`System.Reactive` 僅提供了 `IQbservable<T>` 的一個實作，即 `System.Reactive.ObservableQuery<T>`。這是一個內部型別，但您可以透過呼叫 `Qbservable.Provider` 上的方法來取得它：

```cs
using System.Reactive.Linq;

IQbservable<int> qis = Qbservable.Provider.Range(1, 10);
```

類似 Factory 的運算子（即憑空建立序列的運算子），例如 `Range`，實作為 `IQbservableProvider` 的擴充方法。類似組合子的運算子（即將一個或多個其他序列作為輸入的運算子），例如 `Where`，實作為 `IQbservable<T>` 的擴充方法，這意味著如果我們已經有一個 `IQbservable<T>`，我們可以像對普通 `IObservable<T>`（或就此而言像 `IEnumerable<T>`）一樣直接在其上呼叫運算子，例如：

```cs
qis = qis.Where(i => i % 2 == 0);
```

無論運算子是從頭開始建立全新的來源（如在 `Range` 中），還是附加到現有序列（如在 `Where` 中），這些 `IQbservable<T>` 運算子都以大致相同的方式實作：它們呼叫 `IQbservableProvider.CreateQuery<int>` 方法，並傳遞描述查詢的 `Expression`。（在像 `Where` 這樣的組合子情況下，它們從傳入來源的 `Provider` 屬性取得 `IQbservableProvider`）。

此運算式作為 `IQbservable<T>`（或更準確地說，其基底型別 `IQbservable`）的一部分提供，因此我們可以檢查 `IQbservable<T>` 的運算式：

```cs
Console.WriteLine(qis.Expression);
```

這會產生以下輸出：

```
value(System.Reactive.ObservableQueryProvider).Range(1, 10).Where(i => ((i % 2) == 0))
```

這表明 `qis` 目前參考的查詢始於型別為 `System.Reactive.ObservableQueryProvider` 的特定提供者執行個體。（這是一個內部型別；它碰巧是 `Qbservable.Provider` 傳回的型別。）從那裡，我們呼叫了傳遞引數 `1` 和 `10` 的 `Range` 運算子，然後在產生的 `IQbservable<int>` 上呼叫了傳遞所示 Lambda 運算式的 `Where` 運算子。（出現額外的括弧是因為將 Lambda 的運算式樹狀架構表示形式轉換回文字的程式碼即使在嚴格不需要避免多義性的情況下也會新增括弧。）

理論上，我們可以從 `IQbservableProvider` 的某些其他實作開始。（例如，我們可以撰寫自己的實作。）但 `Expression` 應該仍然相同，因為所有提供者都必須使其 `Expression` 可用，並且該運算式的內容實際上不是由提供者本身決定的，而是由這些運算子方法中的程式碼決定的。例如，`Range` 運算子如下所示：

```cs
public static IQbservable<int> Range(this IQbservableProvider provider, int start, int count)
{
    if (provider == null)
        throw new ArgumentNullException(nameof(provider));

    return provider.CreateQuery<int>(
        Expression.Call(
            null,
            new Func<IQbservableProvider, int, int, IQbservable<int>>(Range).Method,
            Expression.Constant(provider, typeof(IQbservableProvider)),
            Expression.Constant(start, typeof(int)),
            Expression.Constant(count, typeof(int))
        )
    );
}
```

這不在乎它使用什麼提供者；它將以相同的方式為任何提供者建構運算式。提供者在這裡的工作只是建構一個保留其所接收內容的新 `IQbservable<T>`。而 `Where` 也類似：

```cs
public static IQbservable<TSource> Where<TSource>(this IQbservable<TSource> source, Expression<Func<TSource, bool>> predicate)
{
    if (source == null)
        throw new ArgumentNullException(nameof(source));
    if (predicate == null)
        throw new ArgumentNullException(nameof(predicate));

    return source.Provider.CreateQuery<TSource>(
        Expression.Call(
            null,
            new Func<IQbservable<TSource>, Expression<Func<TSource, bool>>, IQbservable<TSource>>(Where<TSource>).Method,
            source.Expression,
            predicate
        )
    );
}
```

區分一個提供者與另一個提供者的是當您實際請求查詢所描述的項目時會發生什麼。（對於 `IQueryable<T>` 也是如此：當您使用像 EF Core 這樣的提供者時，它也會以完全相同的方式建構運算式，只有在您實際嘗試評估查詢時才會發生任何有趣的事情。）而我們透過呼叫 `Subscribe` 來做到這一點，就像我們對任何 `IObservable<T>` 所做的一樣。（`IQbservable<T>` 繼承自 `IObservable<T>`。）

`System.Reactive` 提供的唯一提供者（我們從 `Qbservable.Provider` 獲得的提供者）在您呼叫其 `Subscribe` 方法時會執行此操作：它在執行階段產生程式碼，該程式碼執行的程式碼與您從 `Observable` 而非 `Qbservable.Provider` 開始時獲得的程式碼完全相同。因此，運算式的起點和後續的 Factory 方法（表示為 `value(System.Reactive.ObservableQueryProvider).Range(1, 10)` 的運算式部分）會轉換為呼叫 `Observable.Range(1, 10)` 的程式碼，然後對 `Qbservable.Where` 擴充方法的呼叫（由上面的 `.Where(i => ((i % 2) == 0))` 文字表示）會轉換為對 `Observable.Where` 的呼叫。它將編譯該 Lambda 運算式，因為 `Observable.Where` 需要 `Func<T, bool>`，而不是存在於 `IQbservable.Expression` 中的 `Expression<Func<T, bool>>`。

這類似於如果您開始列舉其中一個時所獲得的 `IQueryable<T>` 實作。（就像 `IQbservable<T>` 繼承自 `IObservable<T>` 一樣，`IQueryable<T>` 也繼承自 `IEnumerable<T>`，因此從 `IEnumerable<T>` 取得項目的任何方式也適用於 `IQueryable<T>`。例如，您可以撰寫 `foreach` 迴圈。）

考慮到這一點，現在讓我們來看看 Rx 6 中存在的一些問題。

### 問題：IL2060 和 IL2026 診斷

.NET 10 SDK 會產生與修剪相關的編譯器診斷。事實上，這在早期的 SDK 中就發生過，我們已經在其中放置了 pragma 來抑制 IL2060。這是因為在 Rx 7 之前，我們使用與上面顯示的程式碼不同的方式來取得 `Expression.Call` 方法的 `MethodInfo` 引數。我們沒有建構 `Func`，而是使用了類似以下內容：

```cs
((MethodInfo)MethodInfo.GetCurrentMethod()!).MakeGenericMethod(typeof(TFirst), typeof(TSecond)),
```
即使在較舊的 .NET SDK 中，這也會產生 IL2060 診斷。這是由於 `MakeGenericMethod` 造成的：分析器回報「無法保證泛型方法的需求可用性」。在較新的 SDK 中，由於 `GetCurrentMethod` 現在標記有 `RequiresUnreferencedCodeAttribute`，此程式碼還將產生 IL2026 診斷。顯然這是隨 .NET 7.0 引入以解決 https://github.com/dotnet/runtime/issues/53242 的，在我看來這似乎有點過頭了：該問題指出這可能「允許反映程式中的基本上所有內容」。我不清楚為什麼僅請求呼叫方法的 `MethodInfo` 就意味著「程式中的所有內容」。當然，您可以從此 `MethodInfo` 逐步存取其他事物，但這對幾乎任何反映型別都是如此。

.NET 執行階段本身的 `Queryable` 類別也需要擷取目前方法的 Metadata，它透過委派避免了這個問題：

```cs
new Func<IQbservable<TFirst>, IObservable<TSecond>, IQbservable<(TFirst First, TSecond Second)>>(SomeMethod).Method
```

這會建構一個委派並純粹使用它來擷取其 `MethodInfo`。由於這擷取了與 `MethodInfo.GetCurrentMethod` 完全相同的 Metadata，因此完全不清楚為什麼在兩種方式都能獲得完全相同的 `MethodInfo` 的情況下，這種基於委派的方法被認為是可行的。該問題中的描述暗示，當您使用 `GetCurrentMethod` 擷取此資訊時，工具丟棄的 Metadata（例如參數名稱）比您透過其他方式取得相同 `MethodInfo` 時丟棄的更多。

### 問題：Qbservable 方法暗示對 Observable 方法的相依性

如果您使用 `IQbservable<T>` 運算子，這很可能會導致對對應 `IObservable<T>` 運算子的執行階段相依性。這是因為如果您 `Subscribe` Rx 建立的任何 `IQbservable<T>`（實際上將是 `ObservableQuery<T>` 的執行個體），它將使用 `ObservableQuery<T>.ObservableRewriter` 將 `IQbservable<T>` 的運算式轉換為基於 `IObservable<T>` 的等效實作，然後在執行階段進行編譯。

例如，假設您使用了 `Where` 運算子。該運算子的 `IQbservable<T>` 實作作為 `Qbservable.Where` 擴充方法提供，編譯器將能夠看到您使用了它，因此知道不要修剪它。但是，當您呼叫 `Subscribe` 時，運算式樹狀架構重寫器會將其轉換為對 `Observable.Where` 的呼叫，作為 `Expression<Func<IObservable<T>>>` 的一部分。然後它對此呼叫 `Compile`。

其結果是使用 `Qbservable.Where` 可能暗示您也在使用 `Observable.Where`，但是以 IL 修剪器可能無法直接看到的方式間接使用的。

`IQueryable<T>` 和 LINQ to Objects 也存在完全相同的問題：您可以將 `IQueryable<T>` 轉換為 `IEnumerable<T>`，這會導致執行階段程式碼產生，該程式碼使用所使用的每個 `Queryable` 運算子的 `Enumerable` 等效項。.NET 執行階段函式庫原始程式碼透過新增 `DynamicDependency` 屬性來告知修剪工具，例如：

```cs
[DynamicDependency("Where`1", typeof(Enumerable))]
public static IQueryable<TSource> Where<TSource>(this IQueryable<TSource> source, Expression<Func<TSource, bool>> predicate)
```

這是給 IL 修剪器的一個提示，即任何使用 `IQueryable<T>` 版本 `Where` 的程式可能也使用 `Enumerable.Where` 方法，因此修剪器應在任何使用前者的應用程式中包含後者。


### 問題：產生的查詢空間檔案免於編譯器診斷

似乎只要在檔案名稱中包含文字 `.Generated.` 就會抑制某些編譯器診斷。我們不希望產生的查詢空間程式碼出現這種情況，因為我們確實希望知道是否存在問題。通常需要在產生的程式碼中抑制診斷的原因是無法修復它們。但是在我們的情況下，我們控制程式碼產生。（它由該儲存庫中的 HomoIcon 工具完成。）因此我們可以且應該修復任何問題，而不是抑制它們。


## 決策

### 替換 `GetCurrentMethod`

由於 IL 修剪工具似乎不完全支援 `GetCurrentMethod`，我們已將所有使用該方法的程式碼替換為建立委派並傳回其 `Method` 的程式碼。這是 .NET 執行階段函式庫在 `IQueryable<T>` 運算子中使用的方法，因此對我們來說採用相同的方法似乎是最安全的。

### 暗示的動態相依性

目前我們不會新增 `DynamicDependency` 屬性，因為這暗示了我們測試未支援的動態程式碼產生支援層級。我們不想誤導開發人員認為這是一個完全受支援的情境。

如果我們收到對有助於 `DynamicDependency` 的情境之更好支援請求，我們將重新審視這一點。（我們很可能還需要在某些地方新增 `RequiresDynamicCode`。）目前，我們懷疑沒有人嘗試將 `IQbservable<T>` 與修剪結合使用。

### 變更 Q 空間運算子的產生檔案名稱

由於檔案名稱中「.Generated.」的存在似乎會阻止修剪警告的出現，HomoIcon 工具產生的檔案現在具有類似 `Qbservable.Homoicon.cs` 而非 `Qbservable.Generated.cs` 的名稱。


## 後果

透過移除 `GetCurrentMethod`，我們不再收到 IL2026 警告，並且我們還能夠移除一直抑制 IL2060 警告的 `#pragma` 行。此外，由於我們現在使用了一種似乎被修剪工具更好地理解的方法，因此這可能會提高圍繞這些方法的靜態分析的有效性。

由於我們目前不新增 `DynamicDependency` 屬性，因此在正在使用修剪的情境中，我們繼續不支援使用來自 `IQbservable<T>` 的執行階段程式碼產生。

透過變更 HomoIcon 工具產生的檔案命名慣例，我們現在將更早地發現它產生的程式碼何時出現可由 .NET SDK 內建分析器偵測到的問題。
