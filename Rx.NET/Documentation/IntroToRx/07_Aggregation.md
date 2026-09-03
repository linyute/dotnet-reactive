# 彙總 (Aggregation)

資料在其原始形式下並不總是易於處理的。有時我們需要合併、對照、組合或濃縮我們接收到的大量資料。這可能只是將資料量減少到可管理水準的情況。例如，考慮來自儀表、金融、訊號處理和營運智慧等領域的快速變動資料。對於個別來源，此類資料可能以每秒超過十個值的速率變更，而如果我們觀察多個來源，速率會更高。人類真的能消化這些內容嗎？對於人類的消費，平均值、最小值和最大值等彙總值可能更有用。

我們通常可以實現比這更多的目標。我們組合和關聯的方式可能使我們能夠揭示模式，提供任何單一訊息或簡單縮減為單一統計測量所無法獲得的見解。Rx 的可組合性使我們能夠對資料流表達複雜且細緻的運算，使我們不僅能減少使用者必須處理的訊息量，還能增加人類接收到的每則訊息的價值量。

我們將從最簡單的彙總函式開始，它們以某種特定方式將可觀察序列縮減為具有單一值的序列。然後我們轉向更通用的運算子，讓你可以定義自己的彙總機制。

## 簡單數值彙總 (Simple Numeric Aggregation)

Rx 支援各種標準 LINQ 運算子，可將序列中的所有值縮減為單一數值結果。

### Count

`Count` 告訴你序列包含多少個元素。儘管這是標準 LINQ 運算子，但 Rx 的版本偏離了 `IEnumerable<T>` 版本，因為 Rx 將返回一個可觀察序列，而不是純量值。像往常一樣，這是因為 Rx 的推播特性。Rx 的 `Count` 無法要求其來源立即提供所有元素，因此它只需要等待直到來源表示它已完成。`Count` 返回的序列將始終是 `IObservable<int>` 類型，無論來源的元素類型為何。在來源完成之前這不會做任何事情，此時它將發出一個單一值報告來源產生了多少個元素，然後它將立即完成。此範例將 `Count` 與 `Range` 一起使用，因為 `Range` 會盡快產生其所有值然後完成，這意味著我們立即從 `Count` 獲得結果：

```csharp
IObservable<int> numbers = Observable.Range(0,3);
numbers.Count().Dump("count");
```

輸出：

```
count-->3
count Completed
```

如果你預期序列中的值超出 32 位元帶正負號整數的計數範圍，則可以改用 `LongCount` 運算子。這與 `Count` 相同，只不過它返回 `IObservable<long>`。

### Sum

`Sum` 運算子將其來源中的所有值相加，產生總和作為其唯一的輸出。與 `Count` 一樣，Rx 的 `Sum` 與大多數其他 LINQ 提供者不同，因為它不產生純量作為其輸出。它產生一個可觀察序列，在來源完成之前不執行任何操作。當來源完成時，`Sum` 返回的可觀察序列會產生一個單一值，然後立即完成。此範例展示了它的使用方式：

```csharp
IObservable<int> numbers = Observable.Range(1,5);
numbers.Sum().Dump("sum");
```

輸出顯示 `Sum` 產生的單一結果：

```
sum-->15
sum completed
```

`Sum` 僅能處理 `int`、`long`、`float`、`double`、`decimal` 類型的值，或這些類型的可為 Null (nullable) 版本。這意味著有些類型你可能期望能夠 `Sum` 但實際上不能。例如 `System.Numerics` 命名空間中的 `BigInteger` 類型表示其大小僅受可用記憶體以及你準備等待其執行計算的時間限制的整數值。（即使是基本的算術在具有數百萬位數的數字上也變得非常緩慢。）你可以使用 `+` 將它們加在一起，因為該類型為該運算子定義了多載。但 `Sum` 在歷史上一直無法找到它。[C# 11.0 中泛型數學](https://learn.microsoft.com/en-us/dotnet/standard/generics/math#operator-interfaces)的引入意味著技術上可以引入一個適用於任何實作 [`IAdditionOperators<T, T, T>`](https://learn.microsoft.com/en-us/dotnet/api/system.numerics.iadditionoperators-3) 的類型 `T` 的 `Sum` 版本。然而，這意味著依賴 .NET 7.0 或更新版本（因為泛型數學在較舊版本中不可用），在撰寫本文時，Rx 透過其 `net6.0` 目標支援 .NET 7.0。它可以引入單獨的 `net7.0` 和/或 `net8.0` 目標以啟用此功能，但尚未這樣做。（平心而論，[LINQ to Objects 中的 `Sum` 也尚未支援此功能](https://github.com/dotnet/runtime/issues/64031)。）

如果你向 `Sum` 提供這些類型的可為 Null 版本（例如，你的來源是 `IObservable<int?>`），則 `Sum` 也將返回具有可為 Null 項目類型的序列，並且如果任何輸入值為 `null`，它將產生 `null`。

儘管 `Sum` 只能處理小且固定的數值類型清單，但你的來源不一定必須產生這些類型的值。`Sum` 提供了接受 Lambda 的多載，該 Lambda 從每個輸入元素中擷取合適的數值。例如，假設你想回答以下不太可能的問題：如果接下來透過 AIS 廣播自身描述的 10 艘船並排擺放，它們是否能全部容納在某個特定寬度的水道中？我們可以透過將 AIS 訊息篩選為提供船舶尺寸資訊的訊息，使用 `Take` 收集接下來的 10 則此類訊息，然後使用 `Sum` 來做到這一點。Ais.NET 函式庫的 `IVesselDimensions` 介面未實作加法（即使實作了，我們剛才也看到 Rx 無法利用這一點），但沒關係：我們只需要提供一個可以接受 `IVesselDimensions` 並返回 `Sum` 可以處理的某種數值類型的值的 Lambda：

```csharp
IObservable<IVesselDimensions> vesselDimensions = receiverHost.Messages
    .OfType<IVesselDimensions>();

IObservable<int> totalVesselWidths = vesselDimensions
    .Take(10)
    .Sum(dimensions => 
            checked((int)(dimensions.DimensionToPort + dimensions.DimensionToStarboard)));
```

（如果你想知道這裡的型別轉換和 `checked` 關鍵字是怎麼回事，AIS 將這些值定義為不帶正負號的整數，因此 Ais.NET 函式庫將它們回報為 `uint`，這不是 Rx 的 `Sum` 所支援的類型。在實務上，船舶寬度不太可能溢位 32 位元帶正負號整數，因此我們只需將其轉換為 `int`，而在極少數情況下如果我們遇到寬度超過 21 億公尺的船舶，`checked` 關鍵字將擲回例外狀況。）

### Average

標準 LINQ 運算子 `Average` 有效地計算 `Sum` 將計算的值，然後將其除以 `Count` 將計算的值。再次強調，雖然大多數 LINQ 實作會返回純量，但 Rx 的 `Average` 會產生一個可觀察序列。

雖然 `Average` 可以處理與 `Sum` 相同數值類型的值，但在某些情況下輸出類型會有所不同。如果來源是 `IObservable<int>`，或者如果你使用接受從來源擷取值的 Lambda 的多載之一，且該 Lambda 返回 `int`，則結果將是 `double`。這是因為一組整數的平均值不一定是整數。同樣地，平均 `long` 值會產生 `double`。但是，`decimal` 類型的輸入會產生 `decimal` 類型的輸出，同樣地 `float` 輸入會產生 `float` 輸出。

與 `Sum` 一樣，如果 `Average` 的輸入是可為 Null 的，則輸出也會是可為 Null 的。

### Min and Max

Rx 實作了標準 LINQ `Min` 和 `Max` 運算子，用以尋找具有最高或最低值的元素。與本節中的所有其他運算子一樣，這些不返回純量，而是返回產生單一值的 `IObservable<T>`。

Rx 為 `Sum` 和 `Average` 支援的相同數值類型定義了專門的實作。然而，與這些運算子不同的是，它還定義了一個接受任何類型的來源項目的多載。當你在 Rx 未定義專門實作的來源類型上使用 `Min` 或 `Max` 時，它會使用 [`Comparer<T>.Default`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.comparer-1.default) 來比較項目。還有一個多載可讓您傳遞比較子。

與 `Sum` 和 `Average` 一樣，存在接受回呼函式的多載。如果你使用這些多載，`Min` 和 `Max` 將為每個來源項目呼叫此回呼函式，並尋找回呼函式返回的最低或最高值。請注意，它們最終產生的單一輸出將是回呼函式返回的值，而不是衍生該值的原始來源項目。要了解這是什麼意思，請看這個範例：

```csharp
IObservable<int> widthOfWidestVessel = vesselDimensions
    .Take(10)
    .Max(dimensions => 
            checked((int)(dimensions.DimensionToPort + dimensions.DimensionToStarboard)));
```

`Max` 在這裡返回一個 `IObservable<int>`，這將是回報船舶尺寸的接下來 10 則訊息中最寬船舶的寬度。但如果你不只是想看到寬度呢？如果你想要整則訊息呢？

## MinBy and MaxBy

Rx 在 `Min` 和 `Max` 上提供了兩個微妙的變化：`MinBy` 和 `MaxBy`。這些類似於我們剛才看到的基於回呼函式的 `Min` 和 `Max`，使我們能夠處理非數值但可能具有數值屬性的元素序列。不同之處在於，`MinBy` 和 `MaxBy` 不返回最小值或最大值，而是告訴你哪個來源元素產生了該值。例如，假設我們不僅僅是發現最寬的船的寬度，而是想知道那究竟是哪艘船：

```csharp
IObservable<IVesselDimensions> widthOfWidestVessel = vesselDimensions
    .Take(10)
    .MaxBy(dimensions => 
              checked((int)(dimensions.DimensionToPort + dimensions.DimensionToStarboard)));
```

這與前一節中的範例非常相似。我們正在處理元素類型為 `IVesselDimensions` 的序列，因此我們提供了一個回呼函式來擷取我們要用於比較目的的值。（事實上與上次相同的回呼函式。）就像 `Max` 一樣，`MaxBy` 試圖找出傳遞給此回呼函式時哪個元素產生最高值。在來源完成之前它無法知道那是哪個元素。如果來源尚未完成，它所能知道的只是_迄今為止_最高的，但未來的某個值可能會超過它。因此，就像我們在本章中看過的所有其他運算子一樣，在來源完成之前這不會產生任何內容，這就是為什麼我在那裡放了一個 `Take(10)`。

然而，我們得到的序列類型是不同的。`Max` 返回了一個 `IObservable<int>`，因為它為來源中的每個項目呼叫回呼函式，然後產生我們的回呼函式返回的值中的最高值。但是使用 `MaxBy`，我們會得到一個 `IObservable<IVesselDimensions>`，因為 `MaxBy` 告訴我們哪個來源元素產生了該值。

當然，可能有多個項目具有最大寬度——例如，可能存在三艘同樣巨大的船隻。對於 `Max` 來說這無所謂，因為它只是試圖返回實際值：有多少來源項目具有最大值並不重要，因為在所有情況下它都是相同的值。但對於 `MaxBy`，我們會取回
產生最大值的原始項目，如果恰好有三個項目都是如此，我們不希望 Rx 隨機挑選其中一個。

因此，與我們到目前為止看到的其他彙總運算子不同，由 `MinBy` 或 `MaxBy` 返回的可觀察序列不一定只產生單一值。它可能會產生好幾個。你可能會問它是否真的是一個彙總運算子，因為它沒有將輸入序列縮減為一個輸出。但它確實將其縮減為單一值：回呼函式返回的最小值（或最大值）。只不過它呈現結果的方式略有不同。它根據彙總處理的結果產生一個序列。你可以將其視為彙總和篩選的結合：它執行彙總以確定最小值或最大值，然後將來源序列篩選為僅包含回呼函式產生該值的元素。

**注意**：LINQ to Objects 也定義了 [`MinBy`](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.minby) 和 [`MaxBy`](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.maxby) 方法，但它們略有不同。這些 LINQ to Objects 版本實際上是隨機挑選單一來源元素——如果有多個來源值都產生最小（或最大）結果，LINQ to Objects 只給你一個，而 Rx 則給你全部。Rx 在 .NET 6.0 加入同名 LINQ to Objects 之前數年就定義了其運算子版本，因此如果你想知道為什麼 Rx 做法不同，真正的問題是為什麼 LINQ to Objects 沒有遵循 Rx 的先例。

## 簡單布林彙總 (Simple Boolean Aggregation)

LINQ 定義了幾個標準運算子，可將整個序列縮減為單一布林值。

### Any

`Any` 運算子有兩種形式。無參數多載實際上提出了這個問題：「這個序列中有任何元素嗎？」它返回一個可觀察序列，如果來源在未發出任何值的情況下完成，它將產生單一值 `false`。然而，如果來源確實產生了值，則在產生第一個值時，結果序列將立即產生 `true` 然後完成。如果它收到的第一個通知是錯誤，那麼它將傳遞該錯誤。

```csharp
var subject = new Subject<int>();
subject.Subscribe(Console.WriteLine, () => Console.WriteLine("Subject completed"));
var any = subject.Any();

any.Subscribe(b => Console.WriteLine("The subject has any values? {0}", b));

subject.OnNext(1);
subject.OnCompleted();
```

輸出：

```
1
The subject has any values? True
subject completed
```

如果我們現在移除 OnNext(1)，輸出將變為以下內容：

```
subject completed
The subject has any values? False
```

在來源確實產生值的情況下，`Any` 會立即取消訂閱它。因此，如果來源想要回報錯誤，`Any` 只有在那是它產生的第一個通知時才會看到此錯誤。

```csharp
var subject = new Subject<int>();
subject.Subscribe(Console.WriteLine,
    ex => Console.WriteLine("subject OnError : {0}", ex),
    () => Console.WriteLine("Subject completed"));

IObservable<bool> any = subject.Any();

any.Subscribe(b => Console.WriteLine("The subject has any values? {0}", b),
    ex => Console.WriteLine(".Any() OnError : {0}", ex),
    () => Console.WriteLine(".Any() completed"));

subject.OnError(new Exception());
```

輸出：

```
subject OnError : System.Exception: Exception of type 'System.Exception' was thrown.
.Any() OnError : System.Exception: Exception of type 'System.Exception' was thrown.
```

但如果來源在發生例外狀況之前產生了一個值，例如：

```csharp
subject.OnNext(42);
subject.OnError(new Exception());
```

我們會看到以下輸出：

```
42
The subject has any values? True
.Any() completed
subject OnError : System.Exception: Exception of type 'System.Exception' was thrown.
```

雖然直接訂閱來源主體 (subject) 的處理常式仍然會看到錯誤，但我們的 `any` 可觀察序列回報了 `True` 值然後完成，這意味著它沒有回報隨後的錯誤。

`Any` 方法還有一個接受述詞 (predicate) 的多載。這實際上提出了稍微不同的問題：「此序列中是否有任何符合這些條件的元素？」其效果類似於使用 `Where` 後接無引數形式的 `Any`。

```csharp
IObservable<bool> any = subject.Any(i => i > 2);
// 功能上等同於 
IObservable<bool> longWindedAny = subject.Where(i => i > 2).Any();
```

### All

`All` 運算子類似於接受述詞的 `Any` 方法，除了所有值都必須滿足該述詞。一旦述詞拒絕某個值，`All` 返回的可觀察序列就會產生 `false` 值然後完成。如果來源達到結尾而沒有產生任何不滿足述詞的元素，則 `All` 將推播 `true` 作為其值。（這樣做的結果是，如果你在空序列上使用 `All`，結果將是一個產生 `true` 的序列。這與 `All` 在其他 LINQ 提供者中的運作方式一致，但對於任何不熟悉稱為[空泛真 (vacuous truth)](https://en.wikipedia.org/wiki/Vacuous_truth) 的形式邏輯約定的人來說，這可能會令人驚訝。）

一旦 `All` 決定產生 `false` 值，它就會立即取消訂閱來源（就像 `Any` 在確定可以產生 `true` 時所做的那樣）。如果在此之前來源產生錯誤，該錯誤將傳遞給 `All` 方法的訂閱者。

```csharp
var subject = new Subject<int>();
subject.Subscribe(Console.WriteLine, () => Console.WriteLine("Subject completed"));
IEnumerable<bool> all = subject.All(i => i < 5);
all.Subscribe(b => Console.WriteLine($"All values less than 5? {b}"));

subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(6);
subject.OnNext(2);
subject.OnNext(1);
subject.OnCompleted();
```

輸出：

```
1
2
6
All values less than 5? False
all completed
2
1
subject completed
```

### IsEmpty

LINQ `IsEmpty` 運算子在邏輯上與無引數的 `Any` 方法相反。若且唯若來源在不產生任何元素的情況下完成時，它返回 `true`。如果來源產生項目，`IsEmpty` 會產生 `false` 並立即取消訂閱。如果來源產生錯誤，這將轉發該錯誤。

### Contains

`Contains` 運算子判斷特定元素是否存在於序列中。你可以使用 `Any` 來實作它，只需提供一個將每個項目與你正在尋找的值進行比較的回呼函式。然而，寫成 `Contains` 通常會稍微簡潔一些，並且可能是更直接表達意圖的方式。

```csharp
var subject = new Subject<int>();
subject.Subscribe(
    Console.WriteLine, 
    () => Console.WriteLine("Subject completed"));

IEnumerable<bool> contains = subject.Contains(2);

contains.Subscribe(
    b => Console.WriteLine("Contains the value 2? {0}", b),
    () => Console.WriteLine("contains completed"));

subject.OnNext(1);
subject.OnNext(2);
subject.OnNext(3);
    
subject.OnCompleted();
```

輸出：

```
1
2
Contains the value 2? True
contains completed
3
Subject completed
```

`Contains` 還有一個多載，允許你指定該類型的預設值以外的 `IEqualityComparer<T>` 實作。如果你有一個自訂類型的序列，且根據使用案例可能有某些特殊的相等性規則，這會很有用。

## 建立你自己的彙總 (Build your own aggregations)

如果前面幾節中描述的內建彙總無法滿足你的需求，你可以建立自己的彙總。Rx 提供了兩種不同的方法來做到這一點。

### Aggregate

`Aggregate` 方法非常具有彈性：可以使用它來建構本章到目前為止所展示的任何運算子。你提供一個函式給它，它會為每個元素呼叫該函式一次。但它不僅僅是將元素傳遞到你的函式中：它還為你的函式提供了一種_彙總_資訊的方式。除了當前元素之外，它還會傳入一個_累加器 (accumulator)_。累加器可以是任何你喜歡的類型——這取決於你想要累積哪種資訊。無論你的函式返回什麼值，都會成為新的累加器值，並且它會將該值與來自來源的下一個元素一起傳遞給函式。這有一些變化形式，但最簡單的多載看起來像這樣：

```csharp
IObservable<TSource> Aggregate<TSource>(
    this IObservable<TSource> source, 
    Func<TSource, TSource, TSource> accumulator)
```

如果你想為 `int` 值產生自己版本的 `Count`，你可以透過提供一個只為來源產生的每個值加 1 的函式來做到這一點：

```csharp
IObservable<int> sum = source.Aggregate((acc, element) => acc + 1);
```

為了確切了解這在做什麼，讓我們看看 `Aggregate` 將如何呼叫此 Lambda。為了更容易看清楚，假設我們將該 Lambda 放在它自己的變數中：

```csharp
Func<int, int, int> c = (acc, element) => acc + 1;
```

現在假設來源產生了一個值為 100 的項目。`Aggregate` 將呼叫我們的函式：

```csharp
c(0, 100) // returns 1
```

第一個引數是目前的累加器。`Aggregate` 已使用 `default(int)` 作為初始累加器值，即 0。我們的函式返回 1，這成為新的累加器值。因此，如果來源產生第二個值，例如 200，`Aggregate` 將傳遞新的累加器以及來自來源的第二個值：

```csharp
c(1, 200) // returns 2
```

這個特定的函式完全忽略了它的第二個引數（來自來源的元素）。它每次只對累加器加一。因此，累加器只不過是記錄了我們的函式被呼叫的次數。

現在讓我們看看如何使用 `Aggregate` 實作 `Sum`：

```csharp
Func<int, int, int> s = (acc, element) => acc + element
IObservable<int> sum = source.Aggregate(s);
```

對於第一個元素，`Aggregate` 將再次傳遞我們選擇的累加器類型 `int` 的預設值：0。並且它將傳遞第一個元素的值。因此，如果第一個元素是 100，它會執行以下操作：

```csharp
s(0, 100) // returns 100
```

然後如果第二個元素是 200，`Aggregate` 將進行此呼叫：

```csharp
s(100, 200) // returns 300
```

請注意，這一次第一個引數是 100，因為那是前一次呼叫 `s` 返回的值。因此在這種情況下，在看到元素 100 和 200 之後，累加器的值為 300，即所有元素的總和。

如果我們希望初始累加器值不是 `default(TAccumulator)` 該怎麼辦？有一個多載可用於此目的。例如，以下是我們如何使用 `Aggregate` 實作類似 `All` 的功能：

```csharp
IObservable<bool> all = source.Aggregate(true, (acc, element) => acc && element);
```

順便說一句，這並不完全等同於真正的 `All`：它處理錯誤的方式不同。如果 `All` 看到單一元素為 `false`，它會立即取消訂閱其來源，因為它知道來源產生的任何其他內容都不可能改變結果。這意味著如果來源即將產生錯誤，它將不再有機會這樣做，因為 `All` 已取消訂閱。但是 `Aggregate` 無法知道累加器已進入永遠無法離開的狀態，因此它將保持訂閱來源，直到來源完成（或者直到訂閱由 `Aggregate` 返回的 `IObservable<T>` 的任何程式碼取消訂閱）。這意味著如果來源產生 `true`，然後產生 `false`，`Aggregate` 與 `All` 不同，仍將保持訂閱來源，因此如果來源繼續呼叫 `OnError`，`Aggregate` 將接收該錯誤，並將其傳遞給其訂閱者。

有些人覺得以下思考 `Aggregate` 的方式很有幫助。如果你的來源產生值 1 到 5，並且如果我們傳遞給 `Aggregate` 的函式稱為 `f`，則一旦來源完成，`Aggregate` 產生的值將是：

```csharp
T result = f(f(f(f(f(default(T), 1), 2), 3), 4), 5);
```

因此在我們重建 `Count` 的情況下，累加器類型是 `int`，所以變成：

```csharp
int sum = s(s(s(s(s(0, 1), 2), 3), 4), 5);
// Note: Aggregate doesn't return this directly -
// it returns an IObservable<int> that produces this value.
```

Rx 的 `Aggregate` 不會一次執行所有這些呼叫：每次來源產生元素時它都會呼叫該函式，因此計算將隨著時間分散進行。如果你的回呼函式是一個_純函式 (pure function)_——不受全域變數和其他環境因素影響，並且對於任何特定輸入始終返回相同結果的函式——這並不重要。`Aggregate` 的結果將與前面的範例一樣，彷彿全都發生在一個大運算式中。但是如果你的回呼函式行為受到例如全域變數或檔案系統當前內容的影響，那麼在來源產生每個值時呼叫它的事實可能更為重要。

順便說一句，`Aggregate` 在某些程式設計系統中有其他名稱。某些系統將其稱為 `reduce`。它也常被稱為 _fold_。（具體來說是 _left fold_。Right fold 則以相反方向進行。依照慣例，其函式以相反順序接受引數，因此看起來像 `s(1, s(2, s(3, s(4, s(5, 0)))))`。Rx 不提供內建的 right fold。這不是很自然的契合，因為它必須等到收到最後一個元素才能開始，這意味著它需要保留整個序列中的每個元素，然後在序列完成時一次評估整個 fold。）

你可能已經注意到，在重新實作一些內建彙總運算子的過程中，我直接從 `Sum` 跳到了 `Any`。那麼 `Average` 呢？事實證明，使用我目前為止向你展示的多載無法做到這一點。那是因為 `Average` 需要累積兩條資訊——累加總計和計數——並且它還需要在最後執行最後一個步驟：它需要將總計除以計數。使用目前展示的多載，我們只能做到一部分：

```csharp
IObservable<int> nums = Observable.Range(1, 5);

IObservable<(int Count, int Sum)> avgAcc = nums.Aggregate(
    (Count: 0, Sum: 0),
    (acc, element) => (Count: acc.Count + 1, Sum: acc.Sum + element));
```

這使用元組 (tuple) 作為累加器，使其能夠累積兩個值：計數和總和。但是最終的累加器值成為結果，而這不是我們想要的。我們缺少計算平均值（將總和除以計數）的最後一步。幸運的是，`Aggregate` 提供了第 3 個多載，使我們能夠提供這最後一步。我們傳遞第二個回呼函式，該函式在來源完成時僅被呼叫一次。`Aggregate` 將最終的累加器值傳入此 Lambda，然後無論它返回什麼，都會成為由 `Aggregate` 返回的可觀察序列所產生的單一項目。

```csharp
IObservable<double> avg = nums.Aggregate(
    (Count: 0, Sum: 0),
    (acc, element) => (Count: acc.Count + 1, Sum: acc.Sum + element),
    acc => ((double) acc.Sum) / acc.Count);
```

我一直在展示 `Aggregate` 如何重新實作某些內建彙總運算子，以說明它是一個強大且非常通用的運算子。然而，這不是我們使用它的目的。`Aggregate` 之所以有用，正是因為它讓我們能夠定義自訂彙總。

例如，假設我想建立透過 AIS 廣播其詳細資料的所有船舶名稱的清單。這是一種做法：

```csharp
IObservable<IReadOnlySet<string>> allNames = vesselNames
    .Take(10)
    .Aggregate(
        ImmutableHashSet<string>.Empty,
        (set, name) => set.Add(name.VesselName));
```

我在此處使用了 `ImmutableHashSet<string>`，因為它的使用模式剛好非常適合 `Aggregate`。一般的 `HashSet<string>` 也可以運作，但稍微沒那麼方便，因為其 `Add` 方法不返回集合，因此我們的函式需要額外的陳述式來返回累加的集合：

```csharp
IObservable<IReadOnlySet<string>> allNames = vesselNames
    .Take(10)
    .Aggregate(
        new HashSet<string>(),
        (set, name) =>
        {
            set.Add(name.VesselName);
            return set;
        });
```

使用這兩種實作中的任何一種，`vesselNames` 都將產生一個單一值，該值是一個 `IReadOnlySet<string>`，包含在回報名稱的前 10 則訊息中看到的每個船舶名稱。

在最後這兩個範例中我不得不迴避一個問題。我讓它們僅適用於出現的前 10 則合適的訊息。想想如果我沒有在其中放入 `Take(10)` 會發生什麼。程式碼將會編譯，但我們會遇到問題。我在各種範例中使用的 AIS 訊息來源是一個無止境的來源。在可預見的未來，船隻將繼續在海洋中移動。Ais.NET 不包含任何旨在偵測文明終結或發明將使船隻使用過時的技術的程式碼，因此它永遠不會在其訂閱者上呼叫 `OnCompleted`。由 `Aggregate` 返回的可觀察序列在其來源完成或失敗之前不會回報任何內容。因此，如果我們移除該 `Take(10)`，其行為將等同於 `Observable.Never<IReadOnlySet<string>>`。我必須強制 `Aggregate` 的輸入結束才能讓它產生某些內容。但還有另一種方式。

### Scan

雖然 `Aggregate` 允許我們將完整序列縮減為單一最終值，但有時這不是我們需要的。如果我們處理的是無止境的來源，我們可能需要更像一個執行中累加總計的東西，每次我們收到一個值時都會更新。`Scan` 運算子正是針對此需求而設計的。`Scan` 和 `Aggregate` 的簽章相同；不同之處在於 `Scan` 不等待其輸入結束。它在每個項目之後產生彙總值。

我們可以使用它來建立如前一節所述的船舶名稱集合，但是使用 `Scan` 我們不必等到結尾。每次收到包含名稱的訊息時，這都會回報目前的清單：

```csharp
IObservable<IReadOnlySet<string>> allNames = vesselNames
    .Scan(
        ImmutableHashSet<string>.Empty,
        (set, name) => set.Add(name.VesselName));
```

請注意，即使沒有任何改變，這個 `allNames` 可觀察序列也會產生元素。如果累加的名稱集合中已包含剛從 `vesselNames` 出現的名稱，則對 `set.Add` 的呼叫將不執行任何操作，因為該名稱已經在集合中。但 `Scan` 會為每個輸入產生一個輸出，並且不在乎累加器是否沒有改變。這是否有影響取決於你計劃對此 `allNames` 可觀察序列做什麼，但如果需要，你可以使用[第 5 章中展示的 `DistinctUntilChanged` 運算子](05_Filtering.md#distinct-and-distinctuntilchanged)輕鬆解決此問題。

你可以將 `Scan` 視為展示其運算過程的 `Aggregate` 版本。如果我們想查看計算平均值的過程如何彙總計數和總和，我們可以這樣寫：

```csharp
IObservable<int> nums = Observable.Range(1, 5);

IObservable<(int Count, int Sum)> avgAcc = nums.Scan(
    (Count: 0, Sum: 0),
    (acc, element) => (Count: acc.Count + 1, Sum: acc.Sum + element));

avgAcc.Dump("acc");
```

這會產生以下輸出：

```
acc-->(1, 1)
acc-->(2, 3)
acc-->(3, 6)
acc-->(4, 10)
acc-->(5, 15)
acc completed
```

你可以在這裡清楚地看到，每次來源產生值時，`Scan` 都會發出當前累加的值。

與 `Aggregate` 不同，`Scan` 不提供接受第二個函式將累加器轉換為結果的多載。因此我們可以在這裡看到包含計數和總和的元組，但看不到我們想要的實際平均值。但我們可以透過使用[轉換章節](06_Transformation.md)中描述的 [`Select`](06_Transformation.md#select) 運算子來實現這一點：

```csharp
IObservable<double> avg = nums.Scan(
    (Count: 0, Sum: 0),
    (acc, element) => (Count: acc.Count + 1, Sum: acc.Sum + element))
    .Select(acc => ((double) acc.Sum) / acc.Count);

avg.Dump("avg");
```

現在我們得到這個輸出：

```
avg-->1
avg-->1.5
avg-->2
avg-->2.5
avg-->3
avg completed
```

`Scan` 是一個比 `Aggregate` 更通用的運算子。你可以透過將 `Scan` 與[篩選章節中描述的 `TakeLast()` 運算子](05_Filtering.md#takelast)結合來實作 `Aggregate`。

```csharp
source.Aggregate(0, (acc, current) => acc + current);
// is equivalent to 
source.Scan(0, (acc, current) => acc + current).TakeLast();
```

彙總對於縮減資料量或組合多個元素以產生平均值或其他結合來自多個元素之資訊的量測非常有用。但是為了執行某些類型的分析，我們還需要在計算彙總值之前對資料進行切片或重組。因此在下一章中，我們將探討 Rx 提供的用於分割資料的各種機制。