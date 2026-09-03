# 附錄 D：Rx 的代數基礎 (Appendix D: Rx's Algebraic Underpinnings)

Rx 運算子可以透過你能想像的幾乎任何方式組合在一起，並且它們通常可以順利組合而不會出現任何問題。這能夠正常運作不僅僅是一個幸運的巧合。一般來說，軟體元件之間的整合通常是軟體開發中最大的痛苦來源之一，因此它能夠運作得如此出色是值得注意的。這在很大程度上歸功於 Rx 依賴於某些底層理論的事實。Rx 的設計使得你不需要知道這些細節就可以使用它，但是好奇的開發人員通常想知道這些事情。

本書前面的章節已經討論了 Rx 的一個正式面向：可觀察來源與其觀察者之間的協定。對於什麼構成對 `IObserver<T>` 的可接受使用，有明確定義的語法規則。這超出了 .NET 型別系統所能強制的範圍，因此我們依賴程式碼做正確的事情。然而，`System.Reactive` 函式庫始終遵守此協定，並且它還設置了一些防護類型，用於偵測應用程式程式碼何時未完全遵守規則，並防止這造成嚴重破壞。

`IObserver<T>` 語法規則很重要。元件依靠它來確保正確運作。以 `Where` 運算子為例。它提供了自己的 `IObserver<T>` 實作，並用它來訂閱底層來源。這會接收來自該來源的項目，然後決定將哪些項目轉發給訂閱由 `Where` 呈現的 `IObservable<T>` 的觀察者。你可以想像它看起來像這樣：

```csharp
public class OverSimplifiedWhereObserver<T> : IObserver<T>
{
    private IObserver<T> downstreamSubscriber;
    private readonly Func<T, bool> predicate;

    public OverSimplifiedWhereObserver(
        IObserver<T> downstreamSubscriber, Func<T, bool> predicate)
    {
        this.downstreamSubscriber = downstreamSubscriber;
        this.predicate = predicate;
    }

    public void OnNext(T value)
    {
        if (this.predicate(value))
        {
            this.downstreamSubscriber.OnNext(value);
        }
    }

    public void OnCompleted()
    {
        this.downstreamSubscriber.OnCompleted();
    }

    public void OnError(Exception x)
    {
        this.downstreamSubscriber.OnError(x);
    }
}
```

這並沒有採取任何明確的步驟來遵循 `IObserver<T>` 語法規則。如果它訂閱的來源也遵守這些規則，則不需要這樣做。由於這只會在它自己的 `OnNext` 中呼叫其訂閱者的 `OnNext`，並且對於 `OnCompleted` 和 `OnError` 也是如此，因此只要此運算子所訂閱的底層來源遵守如何呼叫這三個方法的規則，此類別隨後也會自動遵循這些規則。

事實上，`System.Reactive` 並沒有那麼輕信。它確實有一些偵測某些違反語法規則的程式碼，但即使是這些措施也只是確保一旦執行進入 Rx 就會遵守語法規則。在系統的邊界處有一些檢查，但 Rx 的內部嚴重依賴上游來源會遵守規則的事實。

然而，`IObservable<T>` 的語法規則並不是 Rx 依賴形式主義來確保正確運作的唯一地方。它還取決於一組特定的數學概念：

* 單子 (Monads)
* 降階態射 (Catamorphisms)
* 升階態射 (Anamorphisms)

標準的 LINQ 運算子可以純粹用這三個概念來表達。

這些概念來自[範疇論 (category theory)](https://en.wikipedia.org/wiki/Category_theory)，這是一門關注數學結構的相當抽象的數學分支。在 1980 年代後期，一些電腦科學家正在探索這一數學領域，旨在用它們來為程式的行為建模。[Eugenio Moggi](https://en.wikipedia.org/wiki/Eugenio_Moggi)（一位當時在愛丁堡大學工作的義大利電腦科學家）通常因意識到單子特別適合描述運算而受到讚譽，正如他在 1991 年的論文[運算與單子的概念 (Notions of computations and monads)](https://person.dibris.unige.it/moggi-eugenio/ftp/ic91.pdf) 中所解釋的那樣。這些理論概念被納入 Haskell 程式語言中，主要由 Philip Wadler 和 Simon Peyton Jones 完成，他們在 1992 年發表了關於 [IO 單子處理 (monadic handling of IO)](https://www.microsoft.com/en-us/research/wp-content/uploads/1993/01/imperative.pdf) 的提案。到 1996 年，這項技術已完全納入 Haskell 的 v1.3 版本中，以使程式對輸入和輸出的處理（例如處理使用者輸入或將資料寫入檔案）能夠以強大的數學基礎為後盾的方式運作。這已被廣泛認為是 Haskell 在純函數式語言中為混亂的 IO 現實建模的早期嘗試的重大改進。

為什麼這一切都很重要？這些數學基礎正是 LINQ 運算子可以自由組合的原因。

範疇論的數學學科對各種數學結構有了非常深刻的理解，對於程式設計師來說最有用的結果之一是它提供了某些規則，如果遵循這些規則，可以確保軟體元素在組合在一起時表現良好。誠然，這是一個相當籠統的解釋。如果你想詳細了解範疇論究竟如何應用於程式設計，以及為什麼這樣做很有用，我強烈推薦 [Bartosz Milewski 的《給程式設計師的範疇論》 (Category Theory for Programmers)](https://bartoszmilewski.com/2014/10/28/category-theory-for-programmers-the-preface/)。那裡提供的大量資訊應該可以說明為什麼我不打算在本附錄中嘗試完整的解釋。相反，我的目標只是概述基本概念，並解釋它們如何對應到 Rx 的功能。

## 單子 (Monads)

單子是支撐 LINQ（以及 Rx）設計的最重要的數學概念。完全沒有必要知道單子是什麼才能使用 Rx。最重要的事實是它們的數學特性（特別是對組合的支援）使 Rx 運算子能夠自由地組合在一起。從實用的角度來看，真正重要的只是它能夠正常運作，但是如果你讀到這裡，這可能無法讓你滿意。

通常很難準確描述數學物件究竟是什麼，因為它們本質上是抽象的。因此在討論單子的定義之前，了解 LINQ 如何使用這個概念可能會有幫助。LINQ 將單子視為項目容器的通用表示。作為開發人員，我們知道有很多種東西可以包含項目。有陣列，以及其他集合類型（例如 `IList<T>`）。還有資料庫，儘管資料庫資料表在許多方面與陣列截然不同，但在某些方面它們也很相似。支撐 LINQ 的基本見解是，存在一種數學抽象，它擷取了容器共同特徵的本質。如果我們確定某個 .NET 類型表示一個單子，那麼數學家多年來為理解單子的特性和行為所做的所有工作都將適用於該 .NET 類型。

例如，`IEnumerable<T>` 是一個單子，`IQueryable<T>` 也是。至關重要的是，對於 Rx 來說，`IObservable<T>` 也是如此。LINQ 的設計依賴於單子的屬性，因此如果你能確定某個 .NET 類型是單子，那麼它就是 LINQ 實作的候選者。（反之，如果你嘗試為非單子的類型建立 LINQ 提供者，你可能會遇到問題。）

那麼 LINQ 依賴的這些特性是什麼呢？第一個與包含性直接相關：必須能夠獲取某個值並將其放入你的單子中。你會注意到我到目前為止給出的所有範例都是泛型類型，這並非巧合：單子本質上是類型建構函式，類型引數指示你希望單子包含的內容種類。因此給定某個類型 `T` 的值，必須能夠將其包裝在該類型的單子中。給定一個 `int`，我們可以獲得一個 `IEnumerable<int>`，如果我們做不到這一點，`IEnumerable<T>` 就不是單子。第二個特性如果不在高度抽象中迷失就稍微難以確定，但它本質上歸結為這樣一個想法：如果我們有可以套用於個別包含項目的函式，並且如果這些函式以有用的方式組合，我們可以建立不對個別值操作而是對容器操作的新函式，至關重要的是，這些函式也可以以相同的方式進行組合。

這使我們能夠像處理個別值一樣自由地處理整個容器。

### 單子運算：return 與 bind (The monadic operations: return and bind)

我們剛剛看到單子不僅僅是一種型別。它們需要提供某些操作。這第一個操作，即在單子中包裝一個值的能力，在數學文本中有時被稱為 _unit_，但在計算環境定義中它通常被稱為 _return_。這就是 [`Observable.Return`](03_CreatingObservableSequences.md#observablereturn) 獲取其名稱的方式。

在技術上不需要存在實際的函式。只要有某種機制可以將值放入單子中，就可以滿足單子法則。例如，與 `Observable` 不同，`Enumerable` 類型_沒有_定義 `Return` 方法，但這並不重要。你只需編寫 `new[] { value }`，這就足夠了。

單子只需要提供另一個操作。數學文獻稱之為 _bind_，某些程式系統稱之為 `flatMap`，LINQ 將其稱為 `SelectMany`。這是往往引起最多思考的一個，因為儘管它有明確的正式定義，但要說出它真正做什麼比 _return_ 更難。然而，我們透過單子表示容器的能力來檢驗單子，這提供了一種相當直接的方式來理解 bind/`SelectMany`：它讓我們先取一個每個項目都是巢狀容器的容器（例如，陣列的陣列，或 `IEnumerable<IEnumerable<T>>`），然後將其扁平化。例如，清單的清單將變成一個清單，包含來自每個清單的每個項目。正如我們很快將看到的，這與 bind 的形式數學定義沒有明顯的關係，後者完全更加抽象，但它與之相容，這就是我們享受數學家勞動成果所需要的一切。

至關重要的是，為了符合單子的條件，剛才描述的兩個操作（return 和 bind）必須符合某些規則，或者在文獻中通常被稱為_法則 (laws)_。共有三條法則。所有這些法則都支配著 bind 操作的工作方式，其中兩條法則關注 return 和 bind 如何相互作用。這些法則是基於單子的操作的可組合性的基礎。這些法則有些抽象，因此它們為何能夠啟用此功能並非顯而易見，但它們是不可協商的。如果你的類型和操作不遵循這些法則，那麼你擁有的就不是單子，因此你不能依賴單子保證的特性。

那麼 bind 實際上是什麼樣子的呢？以下是它在 `IEnumerable<T>` 中的形式：

```csharp
public static IEnumerable<TResult> SelectMany<TSource, TResult> (
    this IEnumerable<TSource> source,
    Func<TSource,IEnumerable<TResult>> selector);
```

因此它是一個接受兩個輸入的函式。第一個是 `IEnumerable<TSource>`。第二個輸入本身是一個函式，當提供 `TSource` 時，它會產生一個 `IEnumerable<TResult>`。當你使用這兩個引數叫用 `SelectMany`（又稱 _bind_）時，你會獲得一個 `IEnumerable<TResult>`。儘管 bind 的正式定義要求它具有這種形狀，但它並沒有規定任何特定的行為——任何符合法則的事物都是可以接受的。但在 LINQ 的環境定義中，我們確實期望一種特定的行為：這將為來源可列舉物件（第一個引數）中的每個 `TSource` 叫用一次函式（第二個引數），然後收集由該函式的所有叫用返回的所有 `IEnumerable<TResult>` 集合所產生的所有 `TResult` 值，將它們包裝為一個大的 `IEnumerable<TResult>`。在 `IEnumerable<T>` 的這個特定案例中，我們可以將 `SelectMany` 描述為為每個輸入值獲取一個輸出集合，然後串連所有這些輸出集合。

但我們現在有點太具體了。即使我們專門研究 LINQ 使用單子來表示通用容器，`SelectMany` 也不一定需要串連。它僅要求由 `SelectMany` 返回的容器包含該函式產生的所有項目。串連是一種策略，但 Rx 採取了不同的做法。由於可觀察序列往往會在它們想要的時候產生值，因此 `Observable.SelectMany` 返回的 `IObservable<TResult>` 只會在該函式產生的任何個別個別 `TSource` 的 `IObservable<TResult>` 產生值時產生值。（它執行一些同步以確保其遵循 Rx 對 `IObserver<T>` 呼叫的規則，因此如果這些可觀察序列之一在對訂閱者的 `OnNext` 的呼叫進行中產生值，它將在推送下一個值之前等待該呼叫返回。但除此之外，它只是直接推送所有值。）因此來源值在這裡本質上是交錯的，而不是被串連的。但更廣泛的原則——結果是一個包含回呼函式為個別輸入產生的每個值的容器——適用。

單子 bind 的數學定義具有相同的基本形狀，它只是沒有規定特定的行為。因此任何單子都將具有一個接受兩個輸入的 bind 操作：為某個輸入值類型 (`TSource`) 建構的單子類型的執行個體，以及一個將 `TSource` 作為其輸入並產生為某個輸出值類型 (`TResult`) 建構的單子類型的執行個體的函式。當你使用這兩個輸入叫用 bind 時，結果是為輸出值類型建構的單子類型的執行個體。我們無法在 C# 的類型系統中精確地表示這個通用概念，但這大致給出了寬泛的輪廓：

```csharp
// An impressionistic sketch of the general form of a monadic bind
public static M<TResult> SelectMany<TSource, TResult> (
    this M<TSource> source,
    Func<TSource, M<TResult>> selector);
```

將你選擇的單子類型（`IObservable<T>`、`IEnumerable<T>`、`IQueryable<T>` 或其他）替換為 `M<T>`，這就告訴了你該特定類型的 bind 應該是什麼樣子的。

但是僅提供兩個函式 return 和 bind 是不夠的。它們不僅必須具有正確的形狀，而且還必須遵守法則。

### 單子法則 (The monadic laws)

因此，單子由一個類型建構函式（例如 `IObservable<T>`）和兩個函式 `Return` 與 `SelectMany` 組成。（從現在開始，我將僅使用這些帶有 LINQ 風格的名稱。）但要符合單子的條件，這些功能必須遵守三條「法則」（此處以非常簡潔的形式給出，我將在以下幾節中進行解釋）：

1. `Return` 是 `SelectMany` 的「左單位元 (left-identity)」
2. `Return` 是 `SelectMany` 的「右單位元 (right-identity)」
3. `SelectMany` 實質上應該是結合性的 (associative)

讓我們更詳細地檢驗其中的每一個

#### 單子法則 1：`Return` 是 `SelectMany` 的「左單位元」

這條法則意味著如果你將某個值 `x` 傳入 `Return`，然後將結果作為輸入之一傳遞給 `SelectMany`（其中另一個輸入是函式 `SomeFunc`），那麼結果應該與直接將 `x` 傳入 `SomeFunc` 完全相同。例如：

```csharp
// Given a function like this:
//  IObservable<bool> SomeFunc(int)
// then these two should be identical.
IObservable<bool> o1 = Observable.Return(42).SelectMany(SomeFunc);
IObservable<bool> o2 = SomeFunc(42);
```

這是一種非正式的理解方式。`SelectMany` 將其輸入容器中的每個項目推入 `SomeFunc`，並且每個此類呼叫都會產生一個類型為 `IObservable<bool>` 的容器，並將所有這些容器收集在一起形成一個包含來自所有個別 `IObservable<bool>` 容器之項目的大 `IObservable<bool>`。但在這個範例中，我們提供給 `SelectMany` 的輸入僅包含單個項目，這意味著沒有收集工作要做。`SelectMany` 將使用該唯一的輸入僅叫用我們的函式一次，這將僅產生一個輸出 `IObservable<bool>`。`SelectMany` 必須返回一個 `IObservable<bool>`，其中包含它從對 `SomeFunc` 的單次呼叫中獲得的單個 `IObservable<bool>` 中的所有內容。在這種情況下，它沒有實際的進一步處理要做。由於對 `SomeFunc` 只有一次呼叫，因此在這種情況下它不需要組合來自多個容器的項目：由對 `SomeFunc` 的單次呼叫產生的單個輸出包含了應該在 `SelectMany` 將返回的容器中的所有內容。因此，我們可以直接使用單個輸入項目叫用 `SomeFunc`。

如果 `SelectMany` 做了任何其他事情，那將是很奇怪的。如果 `o1` 在某種程度上有所不同，那將意味著以下三件事之一：

* `o1` 將包含不在 `o2` 中的項目（意味著它以某種方式包含了_非_由 `SomeFunc` 產生的項目）
* `o2` 將包含不在 `o1` 中的項目（意味著 `SelectMany` 遺漏了由 `SomeFunc` 產生的一些項目）
* `o1` 和 `o2` 包含相同的項目，但在使用中的單子類型特有的某種可偵測意義上有所不同（例如，項目的產出順序不同）

因此，這條法則本質上形式化了這樣一種思想：`SelectMany` 不應新增或移除項目，或未能保留使用中的單子通常會保留的特徵（如順序）。（請注意，在 .NET LINQ 提供者中，這通常不要求它們是完全相同的物件。它們通常不會是。它只是意味著它們必須代表完全相同的東西。例如，在這種情況下，`o1` 和 `o2` 都是 `IEnumerable<bool>`，因此這意味著它們應該各自產生完全相同的 `bool` 值序列。）

#### 單子法則 2：`Return` 是 `SelectMany` 的「右單位元」

這條法則意味著如果你將 `Return` 作為函式輸入傳遞給 `SelectMany`，然後將所建構單子類型的某個值作為另一個引數傳入，你應該獲得相同的輸出值。例如：

```csharp
// These two should be identical.
IObservable<int> o1 = GetAnySource();
IObservable<int> o2 = o1.SelectMany(Observable.Return);
```

透過使用 `Return` 作為 `SelectMany` 的函式，我們本質上是要求取得輸入容器中的每個項目並將其包裝在其自己的容器中（`Return` 包裝單個項目），然後將所有這些容器扁平化回單個容器。我們正在新增一層包裝然後再次將其移除，因此這應該沒有任何效果是有道理的。

#### 單子法則 3：`SelectMany` 實質上應該是結合性的 (associative)

假設我們有兩個函式 `Tx1` 和 `Tx2`，每個函式的形式都適合傳遞為 `SelectMany` 的引數。有兩種方法可以套用這些函式：

```csharp
// These two should be identical.
IObservable<int> o1 = source.SelectMany(x => Tx1(x).SelectMany(Tx2));
IObservable<int> o2 = source.SelectMany(x => Tx1(x)).SelectMany(Tx2);
```

這裡的差異只是括號位置的輕微變更：改變的只是右側對 `SelectMany` 的呼叫是在傳遞給另一個 `SelectMany` 的函式內部叫用，還是對另一個 `SelectMany` 的結果叫用。接下來的這個範例調整了版面配置，並且還用完全等效的 `Tx1` 替換了 Lambda `x => Tx1(x)`，這可能使結構上的差異更容易看出：

```csharp
IObservable<int> o1 = source
    .SelectMany(x => Tx1(x).SelectMany(Tx2));
IObservable<int> o2 = source
    .SelectMany(Tx1)
    .SelectMany(Tx2);
```

第三條法則表示其中任何一種都應該具有相同的效果。第二個 `SelectMany` 呼叫（對於 `Tx2`）發生在第一個 `SelectMany` 呼叫「內部」還是之後應該沒有關係。

一種非正式的思考方式是 `SelectMany` 實際上套用了兩個操作：轉換和解包 (unwrap)。轉換由你傳遞給 `SelectMany` 的任何函式定義，但因為該函式返回單子類型（用 LINQ 的術語來說，它返回一個可能包含任意數量項目的容器），所以 `SelectMany` 在將項目傳遞給函式時解包返回的每個容器，以便將所有項目收集在一起放入它最終返回的單個容器中。當你嵌套這種操作時，解包發生的順序並不重要。例如，考慮以下函式：

```csharp
IObservable<int> Tx1(int i) => Observable.Range(1, i);
IObservable<string> Tx2(int i) => Observable.Return(i.ToString());
```

第一個將數字轉換為相同長度的數字範圍。`1` 變為 `[1]`，`3` 變為 `[1,2,3]` 等等。在我們討論 `SelectMany` 之前，想像一下如果我們在產生數字範圍的可觀察來源上將其與 `Select` 一起使用會發生什麼：

```csharp
IObservable<int> input = Observable.Range(1, 3); // [1,2,3]
IObservable<IObservable<int>> expandTx1 = input.Select(Tx1);
```

我們得到一個序列的序列。`expand2` 實際上是這樣的：

```
[
    [1],
    [1,2],
    [1,2,3],
]
```

如果我們改用 `SelectMany`：

```csharp
IObservable<int> expandTx1Collect = input.SelectMany(Tx1);
```

它將套用相同的轉換，但隨後將結果扁平化回單一清單：

```
[
    1,
    1,2,
    1,2,3,
]
```

我保留了換行符號以強調這與先前輸出之間的聯繫，但我也可以只寫 `[1,1,2,1,2,3]`。

如果我們隨後想套用第二個轉換，我們可以使用 `Select`：

```csharp
IObservable<IObservable<string>> expandTx1CollectExpandTx2 = expandTx1Collect
    .SelectMany(Tx1)
    .Select(Tx2);
```

這會將 `expandTx1Collect` 中的每個數字傳遞給 `Tx2`，後者將其轉換為包含單個字串的序列：

```
[
    ["1"],
    ["1"],["2"],
    ["1"],["2"],["3"]
]
```

但如果我們在該最終位置也使用 `SelectMany`：

```csharp
IObservable<string> expandTx1CollectExpandTx2Collect = expandTx1Collect
    .SelectMany(Tx1)
    .SelectMany(Tx2);
```

它將這些重新扁平化為字串：

```
[
    "1",
    "1","2",
    "1","2","3"
]
```

類似結合律的要求表示，如果我們在傳遞給第一個 `SelectMany` 的函式內部套用 `Tx1`，而不是將其套用於該第一個 `SelectMany` 的結果，應該沒有關係。因此，與其從這個開始：

```csharp
IObservable<IObservable<int>> expandTx1 = input.Select(Tx1);
```

我們可能會這樣寫：

```csharp
IObservable<IObservable<IObservable<string>>> expandTx1ExpandTx2 =
    input.Select(x => Tx1(x).Select(Tx2));
```

這將產生此結果：

```
[
    [["1"]],
    [["1"],["2"]],
    [["1"],["2"],["3"]]
]
```

如果我們將其變更為對巢狀呼叫使用 `SelectMany`：

```csharp
IObservable<IObservable<string>> expandTx1ExpandTx2Collect =
    input.Select(x => Tx1(x).SelectMany(Tx2));
```

這將扁平化內部項目（但我們在外部仍在使用 `Select`，因此我們仍然得到清單的清單），產生以下結果：

```
[
    ["1"],
    ["1","2"],
    ["1","2","3"]
]
```

然後如果我們將第一個 `Select` 變更為 `SelectMany`：

```csharp
IObservable<string> expandTx1ExpandTx2CollectCollect =
    input.SelectMany(x => Tx1(x).SelectMany(Tx2));
```

它將扁平化外層清單，給我們：

```
[
    "1",
    "1","2",
    "1","2","3"
]
```

正如第 3 條單子法則所要求的，這與我們之前獲得的最終結果相同。

總結一下，這裡的兩個過程分別是：

* 展開並轉換 Tx1，扁平化，展開並轉換 Tx2，扁平化
* 展開並轉換 Tx1，展開並轉換 Tx2，扁平化，扁平化

這兩者都套用了兩種轉換，並扁平化了這些轉換所新增的額外包含層級，因此儘管中間步驟看起來不同，但我們最終獲得了相同的結果，因為在每次轉換後解包，還是在解包前執行兩種轉換，結果都是一樣的。

#### 為什麼這些法則很重要 (Why these laws matter)

這三條法則直接反映了對數字進行簡單函式組合時適用的法則。如果我們有兩個函式 $f$ 和 $g$，我們可以寫一個新函式 $h$，定義為 $g(f(x))$。這種組合函式的方式稱為_複合 (composition)_，通常寫作 $g \circ f$。如果單位元函式稱為 $id$，則以下陳述為真：

* $id \circ f$ 等價於 $f$
* $f \circ id$ 等價於 $f$
* $(f \circ g) \circ s$ 等價於 $f \circ (g \circ s)$

這些直接對應於三條單子法則。非正式地說，這反映了單子 bind 操作 (`SelectMany`) 與函式複合具有深刻的結構相似性。這就是為什麼我們可以自由地將 LINQ 運算子組合在一起。

### 使用 `SelectMany` 重建其他運算子 (Recreating other operators with `SelectMany`)

請記住，LINQ 的核心有三個數學概念：單子 (monads)、升階態射 (anamorphisms) 和降階態射 (catamorphisms)。因此，儘管前面的討論重點是 `SelectMany`，但其意義要廣泛得多，因為我們可以用這些基本元素來表達其他標準 LINQ 運算子。例如，這顯示了我們如何僅使用 `Return` 和 `SelectMany` 實作 [`Where`](05_Filtering.md#where)：

```csharp
public static IObservable<T> Where<T>(this IObservable<T> source, Func<T, bool> predicate)
{
    return source.SelectMany(item =>
        predicate(item)
            ? Observable.Return(item)
            : Observable.Empty<T>());
}
```

這實作了 `Select`：

```csharp
public static IObservable<TResult> Select<TSource, TResult>(
    this IObservable<TSource> source, Func<TSource, TResult> f)
{
    return source.SelectMany(item => Observable.Return(f(item)));
}
```

某些運算子需要升階態射或降階態射，所以現在讓我們來檢視它們。

## 降階態射 (Catamorphisms)

降階態射本質上是將容器中每個項目都納入考慮的任何處理的泛化。在 LINQ 的實務中，這通常意味著檢查所有值並產生單個值作為結果的程序，例如 [Observable.Sum](07_Aggregation.md#sum)。更廣泛地說，任何類型的彙總都構成降階態射。降階態射的數學定義比這更通用——例如，它不一定需要將事物一路縮減為單個值——但出於理解 LINQ 的目的，這種面向容器的觀點是思考此建構最直接的方式。

降階態射是 LINQ 的基本建構塊之一，因為你無法從其他元素建構出降階態射。但是有許多 LINQ 運算子可以由 LINQ 最基本的降階態射——[`Aggregate`](07_Aggregation.md#aggregate) 運算子建構出來。例如，這是一種根據 `Aggregate` 實作 `Count` 的方法：

```csharp
public static IObservable<int> MyCount<T>(this IObservable<T> items)
    => items.Aggregate(0, (total, _) => total + 1);
```

我們可以這樣實作 `Sum`：

```csharp
public static IObservable<T> MySum<T>(this IObservable<T> items)
    where T : INumber<T>
    => items.Aggregate(T.Zero, (total, x) => x + total);
```

這比我在[彙總章節](07_Aggregation.md)中展示的類似總和範例更靈活，因為該範例僅適用於 `IObservable<int>`。這裡我使用了 C# 11.0 和 .NET 7.0 中新增的_泛型數學 (generic math)_功能，使 `MySum` 能夠在任何類似數字的型別上運作。但是基本的操作原理是相同的。

如果你是為了理論而來到這裡，僅僅看到各種彙總運算子都是 `Aggregate` 的特例可能對你來說還不夠。到底什麼是降階態射？一種定義是「從初始代數到某個其他代數的唯一同態 (unique homomorphism from an initial algebra into some other algebra)」，但正如範疇論的典型情況一樣，這是那些如果你已經理解它試圖描述的概念就最容易理解的解釋之一。如果你試圖從學校數學形式的代數（我們編寫其中一些值由字母表示的方程式）來理解這種描述，很難理解這個定義。那是因為降階態射對什麼構成「代數」採取了更為一般的看法，本質上意味著可以藉以建構和求值某種運算式的某個系統。

更準確地說，降階態射是相對於稱為 F-代數 (F-algebra) 的事物來描述的。那是三件事物的組合：

1. 一個函子 (Functor) _F_，定義了某個範疇 _C_ 上的某種結構
2. 範疇 _C_ 中的某個物件 _A_
3. 從 _F A_ 到 _A_ 的態射 (morphism)，有效地評估該結構

但這引出的問題多於答案。所以讓我們從顯而易見的問題開始：什麼是函子 (Functor)？從 LINQ 的角度來看，它本質上是實作 `Select` 的任何事物。（某些程式設計系統稱之為 `fmap`。）從我們面向容器的觀點來看，它是兩件事物：1) 類似容器的型別建構函式（例如類似 `IEnumerable<T>` 或 `IObservable<T>` 的事物）和 2) 將函式套用於容器中所有項目的某種方式。因此，如果你有一個從 `string` 轉換為 `int` 的函式，函子可以讓你在單一步驟中將其套用於其包含的所有內容。

`IEnumerable<T>` 及其 `Select` 擴充方法的組合是一個函子。你可以使用 `Select` 將 `IEnumerable<string>` 轉換為 `IEnumerable<int>`。`IObservable<T>` 及其 `Select` 形成了另一個函子，我們可以使用它們從 `IObservable<string>` 獲得 `IObservable<int>`。那麼「在某個範疇 _C_ 上」的部分呢？這暗示了函子的數學描述相當廣泛的事實。當開發人員使用範疇論時，我們通常堅持使用代表型別（如程式語言型別，如 `int`）和函式的範疇。（嚴格來說，函子是從一個範疇映射到另一個範疇，因此在最一般的形勢下，函子將某個範疇 _C_ 中的物件和態射映射到某個範疇 _D_ 中的物件和態射。但出於程式設計目的，我們始終使用代表型別的範疇，因此對於我們使用的函子，_C_ 和 _D_ 將是同一件事。嚴格來說，這意味著我們應該稱它們為自函子 (Endofunctors)，但似乎沒有人會在意。在實務中，我們使用更一般形式的名稱 Functor，並且理所當然地認為我們指的是型別和函式範疇上的自函子。）

所以，這就是函子部分。讓我們繼續看第 2 點，「範疇 _C_ 中的某個物件 _A_。」那麼 _C_ 是函子的範疇，而我們剛剛確定該範疇中的物件是型別，因此這裡的 _A_ 可能是 `string` 型別。如果我們選擇的函子是 `IObservable<T>` 及其 `Select` 方法的組合，則 _F A_ 將是 `IObservable<string>`。

那麼第 3 點中的「態射 (morphisms)」呢？同樣，出於我們的目的，我們只是在型別和函式上使用自函子，因此在這種情況下，態射只是函式。因此，我們可以用更熟悉的術語將 F-代數的定義重塑為：

1. 某個類似容器的泛型型別，例如 `IObservable<T>`
2. 項目型別 `A`（例如 `string` 或 `int`）
3. 接受 `IObservable<A>` 並傳回型別為 `A` 之值的函式（例如 `Observable.Aggregate<A>`）

這更加具體了。範疇論通常關注擷取關於數學結構的最普遍真理，而這種重新表述拋棄了這種普遍性。然而，從希望依靠數學理論的程式設計師的角度來看，這很好。只要我們所做的事情符合 F-代數的模型，數學家推導出的所有一般結果都將適用於我們更專門的理論應用。

儘管如此，為了讓你了解 F-代數的通用概念可以實現哪些類型的事物，函子有可能是表示程式語言中運算式的型別，並且你可以建立一個評估這些運算式的 F-代數。這與 LINQ 的 `Aggregate` 概念類似，因為它遍歷由函子表示的整個結構（如果它是 `IEnumerable<T>`，則為清單中的每個元素；如果你表示運算式，則為每個子運算式）並將整個事物縮減為單個值，但我們的函子不是表示事物的序列，而是具有更複雜的結構：某種程式語言中的運算式。

這就是 F-代數。從理論的角度來看，第三部分不一定必須縮減事物，這一點很重要。理論上，型別可以是遞迴的，項目型別 _A_ 是 _F A_。（這對於本質上遞迴的結構（如運算式）很重要。）通常存在一個最大程度通用的 F-代數，其中第 3 點中的函式（或態射）僅處理結構，並且實際上根本不執行任何縮減。（例如，給定某個運算式語法，你可以想像包含遍歷運算式的每個單一子運算式所需的所有知識的程式碼，但它對要套用什麼處理沒有特定意見。）降階態射的概念是，對於相同的函子，可用的其他 F-代數較少，且通用性較差。

例如，對於 `IObservable<T>`，通用概念是來源產生的每個項目都可以透過重複套用具有兩個引數的某個函式來處理，其中一個是來自容器的型別為 `T` 的值，另一個是某種累加器，代表到目前為止彙總的所有資訊。並且此函式將傳回更新後的累加器，準備好與下一個 `T` 一起再次傳入該函式。然後有更具體的形式，其中套用了特定的累加邏輯（例如求和或確定最大值）。從技術上講，這裡的降階態射是從一般形式到更專用形式的連接。但在實務中，通常將特定的專用形式（例如 [`Sum`](07_Aggregation.md#sum) 或 [`Average`](07_Aggregation.md#average)）稱為降階態射。

### 留在容器內部 (Remaining inside the container)

儘管一般來說降階態射可以剝離容器（例如 `IEnumerable<int>` 的 `Sum` 會產生一個 `int`），但這並不是絕對必要的，並且在 Rx 中大多數降階態射不會這樣做。如執行緒與排程章節的[鎖定 (Lock-ups)](11_SchedulingAndThreading.md#lock-ups) 一節中所述，阻塞某個執行緒同時等待僅在 `IObservable<T>` 執行了特定操作後才會出現的結果（例如，如果你想計算項目的總和，你必須等待直到看到所有項目），在實務中是死結的根源。

出於這個原因，大多數降階態射都會執行某種縮減，但繼續產生包裝在 `IObservable<T>` 中的結果。

## 升階態射 (Anamorphisms)

粗略地說，升階態射是降階態射的反面。降階態射本質上將某種結構摺疊為更簡單的事物，而升階態射則將某些輸入展開為更複雜的結構。例如，給定某個數字（例如 5），我們可以想像一種將其轉換為具有指定元素數量的序列的機制（例如 [0,1,2,3,4]）。

事實上我們不必想像這樣的事情：這就是 [`Observable.Range`](03_CreatingObservableSequences.md#observablerange) 所做的事。

我們可以將單子 `Return` 操作視為非常簡單的升階態射。給定某個型別為 `T` 的值，[`Observable.Return`](03_CreatingObservableSequences.md#observablereturn) 會將其展開為 `IObservable<T>`。升階態射本質上是這種想法的推廣。

升階態射的數學定義是「將餘代數 (coalgebra) 指派給其對自函子的終餘代數 (final coalgebra) 的唯一態射」。這是降階態射定義的「對偶 (dual)」，從範疇論的角度來看，這本質上意味著你反轉所有態射的方向。在我們對範疇論非完全通用的應用中，這裡討論的態射是在降階態射中將項目縮減為某個輸出，因此在升階態射中，這變成了將某個值展開為容器型別的某個執行個體（例如，從 `int` 到 `IObservable<int>`）。

我不打算像降階態射那樣詳細介紹。相反，我將指出其核心的關鍵部分：函子的最通用 F-代數體現了對函子基本結構的某種理解，降階態射利用該結構來定義各種縮減。類似地，函子的最通用餘代數也體現了對函子基本結構的某種理解，升階態射利用該結構來定義各種展開。

[`Observable.Generate`](03_CreatingObservableSequences.md#observablegenerate) 代表了這種最通用的功能：它具有產生 `IObservable<T>` 的能力，但需要提供某些專門的展開函式來產生任何特定的可觀察序列。

## 理論就講到這裡 (So much for theory)

現在我們已經回顧了 LINQ 背後的理論概念，讓我們退一步看看我們如何使用它們。我們有三種操作：

* 升階態射進入序列：`T1 --> IObservable<T2>`
* Bind 修改序列：`IObservable<T1> --> IObservable<T2>`
* 降階態射離開序列。邏輯上為 `IObservable<T1> --> T2`，但在實務中通常是 `IObservable<T1> --> IObservable<T2>`，其中輸出可觀察序列僅產生單個值

順便提一下，bind 和 catamorphism 因 Google 的 [MapReduce](http://en.wikipedia.org/wiki/MapReduce) 架構而聞名。在這裡，Google 用某些函數式語言中更常用的名稱 Map 和 Reduce 來指代 Bind 和 Catamorphism。

大多數 Rx 運算子實際上是高階函數式概念的特化。舉幾個範例：

- 升階態射 (Anamorphisms)：
  - [`Generate`](03_CreatingObservableSequences.md#observablegenerate)
  - [`Range`](03_CreatingObservableSequences.md#observablerange)
  - [`Return`](03_CreatingObservableSequences.md#observablereturn)
- Bind：
  - [`SelectMany`](06_Transformation.md#selectmany)
  - [`Select`](06_Transformation.md#select)
  - [`Where`](05_Filtering.md)
- 降階態射 (Catamorphism)：
  - [`Aggregate`](07_Aggregation.md#aggregate)
  - [`Sum`](07_Aggregation.md#sum)
  - [`Min` 和 `Max`](07_Aggregation.md#min-and-max)

## Amb

當我開始使用 Rx 時，`Amb` 方法對我來說是一個新概念。此函式最初由 [John McCarthy](https://en.wikipedia.org/wiki/John_McCarthy_(computer_scientist)) 在其 1961 年發表於西部聯合電腦會議論文集的論文[「計算數學理論基礎」('A Basis for a Mathematical Theory of Computation')](https://www.cambridge.org/core/journals/journal-of-symbolic-logic/article/abs/john-mccarthy-a-basis-for-a-mathematical-theory-of-computation-preliminary-report-proceedings-of-the-western-joint-computer-conference-papers-presented-at-the-joint-ireaieeacm-computer-conference-los-angeles-calif-may-911-1961-western-joint-computer-conference-1961-pp-225238-john-mccarthy-a-basis-for-a-mathematical-theory-of-computation-computer-programming-and-formal-systems-edited-by-p-braffort-and-d-hirschberg-studies-in-logic-and-the-foundations-of-mathematics-northholland-publishing-company-amsterdam1963-pp-3370/D1AD4E0CDB7FBE099B04BB4DAF24AFFA) 中提出。（這份文件的數位副本很難找到，但後來的版本於 [1963](http://www-formal.stanford.edu/jmc/basis1.pdf) 年發表在「電腦程式設計與格式系統」中。）它是單字 _Ambiguous_ 的縮寫。Rx 在使用此縮寫時略微偏離了正常的 .NET 類別庫命名約定，部分原因是 `amb` 是此運算子的既定名稱，但也為了向 McCarthy 致敬，他的工作是 Rx 設計的靈感來源。

但是 `Amb` 是做什麼的呢？[_模稜兩可函式 (ambiguous function)_](http://www-formal.stanford.edu/jmc/basis1/node7.html) 的基本思想是允許我們定義產生結果的多種方式，並且在實務中證明這些方式中的某些或全部可能無法產生結果。假設我們定義了某個名為 `equivocate` 的模稜兩可函式，也許對於某個特定的輸入值，`equivocate` 的所有組成部分（我們為其提供的計算結果的所有不同方式）都無法處理該值。（也許它們中的每一個都用輸入來除以某個數字。如果我們提供 `0` 的輸入，那麼這些元件都無法為該輸入產生值，因為它們都會嘗試除以 0。）在這種 `equivocate` 的組成部分都無法產生結果的情況下，`equivocate` 本身也無法產生結果。但是假設我們提供了某個輸入，其中恰好其組成部分之一能夠產生結果。在那種情況下，該結果將成為該輸入的 `equivocate` 的結果。

因此實質上，我們提供了許多處理輸入的不同方式，如果其中恰好有一種能夠產生結果，我們就選擇該結果。如果處理輸入的方式都沒有產生任何結果，那麼我們的模稜兩可函式也不會產生任何結果。

稍微更奇怪的地方（以及 Rx 偏離 `amb` 原始定義的地方）在於模稜兩可函式有多個組成部分產生結果的時候。在 McCarthy 的理論公式中，模稜兩可函式有效地將所有結果作為可能的輸出產生。（這在技術上稱為_非確定性 (nondeterministic)_計算，儘管該名稱可能具有誤導性：它聽起來像結果是不可預測的。但這不是我們在談論計算時所說的_非確定性_的意思。這就好像評估模稜兩可函式的電腦複製了自己，為每個可能的結果產生一個副本，繼續執行每一個副本。你可以想像這樣一個系統的多執行緒實作，每當模稜兩可函式產生多個可能的結果時，我們都會建立那麼多新執行緒，以便能夠評估所有可能的結果。這是非確定性計算的合理心智模型，但這並不是 Rx 的 `Amb` 運算子實際發生的情況。）在引入模稜兩可函式的各類理論工作中，歧義最終往往會消失。運算進行的方式可能有極多種，但它們最終可能都會產生相同的結果。然而，此類理論問題使我們脫離了 Rx 的 `Amb` 所做的事情，以及我們如何在實務中使用它。

[Rx 的 `Amb`](09_CombiningSequences.md#amb) 提供了在輸入都沒有產生任何內容或恰好其中一個產生內容的情況下所描述的行為。然而，它沒有嘗試支援非確定性計算，因此它對多個組成部分能夠產生值的情況的處理被過度簡化了，但 McCarthy 的 `amb` 首先是一種分析建構，因此它的任何實際實作總是有所欠缺。

## 留在單子內部 (Staying inside the monad)

使用 Rx 時，在程式設計風格之間切換可能很誘人。對於容易看出 Rx 如何套用的部分，我們自然會使用 Rx。但是當事情變得很棘手時，變更路線似乎是最容易的。最簡單的事情似乎是 `await` 一個可觀察序列，然後繼續使用普通的循序程式碼。或者，讓傳遞給像 `Select` 或 `Where` 這樣的運算子的回呼在主要工作之外執行其他操作——產生具有實用功能的副作用——似乎是最簡單的。

儘管這有時可行，但在典範之間切換應謹慎進行，因為這是並行問題（如死結和延展性問題）的常見根本原因。其基本原因在於，只要你留在 Rx 的做事方式中，你就會受益於數學基礎的基本穩健性。但要使其發揮作用，你需要使用函數式風格。函式應該處理其輸入並根據這些輸入以確定性方式產生輸出，並且它們既不應依賴外部狀態，也不應變更外部狀態。這可能是一個很高的要求，而且並不總是可能的，但如果你違反這些規則，很多理論就會崩潰。組合無法如預期般可靠地運作。因此，使用函數式風格並將程式碼保持在 Rx 的慣用語法內往往會提高可靠性。

## 副作用的問題 (Issues with side effects)

如果程式要做任何有用的事情，它們總是必須有一些副作用——如果程式執行的結果與世界沒有什麼不同，那麼你還不如不安裝執行它——因此探索副作用的問題可能很有用，這樣我們就可以知道在必要時如何最好地處理它們。因此，我們現在將討論在使用可觀察序列時引入副作用的後果。如果函式除了任何傳回值之外還具有某些其他可觀察到的效果，則該函式被認為具有副作用。通常「可觀察到的效果」是狀態的修改。這種可觀察到的效果可能是：

* 修改比函式具有更廣泛範圍的變數（即全域、靜態或可能是引數）
* I/O，例如讀取或修改檔案、發送或接收網路訊息或更新顯示
* 引起物理活動，例如自動販賣機分配物品，或將硬幣引導到其錢箱中

一般的函數式程式設計試圖避免產生任何副作用。具有副作用的函式，特別是那些修改狀態的函式，要求程式設計師理解的不僅僅是函式的輸入和輸出。完全理解該函式的操作可能需要了解正在修改的狀態的完整歷史記錄和上下文。這會大大增加函式的複雜性，並使其更難以正確理解和維護。

副作用並不總是有意的。減少意外副作用的簡單方法是減少變更的表面積。以下是程式設計人員可以採取的兩個簡單動作：減少狀態的可見性或範圍，並儘可能使其不可變。你可以透過將變數的範圍限制在像方法這樣的程式碼區塊（而不是欄位或屬性）來減少變數的可見性。你可以透過將類別成員設為私有 (private) 或受保護 (protected) 來減少其可見性。根據定義，不可變資料無法修改，因此它不會表現出副作用。這些是明智的封裝規則，將顯著提高你的 Rx 程式碼的可維護性。

為了提供具有副作用的查詢的簡單範例，我們將嘗試透過更新變數（閉包）來輸出訂閱接收到的元素的索引和值。

```csharp
IObservable<char> letters = Observable
    .Range(0, 3)
    .Select(i => (char)(i + 65));

int index = -1;
IObservable<char> result = letters.Select(
    c =>
    {
        index++;
        return c;
    });

result.Subscribe(
    c => Console.WriteLine("Received {0} at index {1}", c, index),
    () => Console.WriteLine("completed"));
```

輸出：

```
Received A at index 0
Received B at index 1
Received C at index 2
completed
```

雖然這看起來足夠無害，但想像一下，如果另一個人看到這段程式碼並將其理解為團隊正在使用的模式。他們反過來自己採用這種風格。為了範例的緣故，我們將在前面的範例中新增一個重複的訂閱。

```csharp
var letters = Observable.Range(0, 3)
                        .Select(i => (char)(i + 65));

var index = -1;
var result = letters.Select(
    c =>
    {
        index++;
        return c;
    });

result.Subscribe(
    c => Console.WriteLine("Received {0} at index {1}", c, index),
    () => Console.WriteLine("completed"));

result.Subscribe(
    c => Console.WriteLine("Also received {0} at index {1}", c, index),
    () => Console.WriteLine("2nd completed"));
```

輸出

```
Received A at index 0
Received B at index 1
Received C at index 2
completed
Also received A at index 3
Also received B at index 4
Also received C at index 5
2nd completed
```

現在第二個人的輸出顯然是胡說八道。他們期望索引值為 0、1 和 2，但得到的卻是 3、4 和 5。我在程式碼庫中看過更險惡的副作用版本。討厭的那些通常會修改為布林值的狀態，例如 `hasValues`、`isStreaming` 等。

除了在現有軟體中產生潛在不可預測的結果之外，具有副作用的程式更難以測試和維護。未來對具有副作用的程式進行重構、增強或其他維護更有可能是脆弱的。在非同步或並行軟體中尤其如此。

## 在管線中組合資料 (Composing data in a pipeline)

擷取狀態的偏好方式是將其作為流經構成訂閱的 Rx 運算子管線的資訊的一部分。理想情況下，我們希望管線的每個部分都是獨立且具確定性的。也就是說，組成管線的每個函式都應該將其輸入和輸出作為其唯一的狀態。為了修正我們的範例，我們可以豐富管線中的資料，以便沒有共享狀態。這將是一個很好的範例，我們可以在其中使用公開索引的 `Select` 多載。

```csharp
IObservable<int> source = Observable.Range(0, 3);
IObservable<(int Index, char Letter)> result = source.Select(
    (idx, value) => (Index: idx, Letter: (char) (value + 65)));

result.Subscribe(
    x => Console.WriteLine($"Received {x.Letter} at index {x.Index}"),
    () => Console.WriteLine("completed"));

result.Subscribe(
    x => Console.WriteLine($"Also received {x.Letter} at index {x.Index}"),
    () => Console.WriteLine("2nd completed"));
```

輸出：

```
Received A at index 0
Received B at index 1
Received C at index 2
completed
Also received A at index 0
Also received B at index 1
Also received C at index 2
2nd completed
```

跳脫傳統思維，我們還可以使用像 `Scan` 這樣的其他功能來獲得類似的結果。這是一個範例。

```csharp
var result = source.Scan(
                new
                {
                    Index = -1,
                    Letter = new char()
                },
                (acc, value) => new
                {
                    Index = acc.Index + 1,
                    Letter = (char)(value + 65)
                });
```

這裡的關鍵是隔離狀態，並減少或消除任何諸如突變狀態之類的副作用。
