# 建立可觀察序列

在上一章中，我們看到了兩個基本的 Rx 介面：`IObservable<T>` 與 `IObserver<T>`。我們也看到了如何透過實作 `IObserver<T>` 來接收事件，以及如何使用 `System.Reactive` 套件所提供的實作。在本章中，我們將探討如何建立 `IObservable<T>` 來源，以表示應用程式中感興趣的來源事件。

我們將從直接實作 `IObservable<T>` 開始。在實務上，這樣做相對少見，因此接著我們將介紹各種讓 `System.Reactive` 提供實作的方法，這些實作能替你完成大部分的工作。

## 一個非常基礎的 `IObservable<T>` 實作

這是一個產生數字序列的 `IObservable<int>` 實作：

```csharp
public class MySequenceOfNumbers : IObservable<int>
{
    public IDisposable Subscribe(IObserver<int> observer)
    {
        observer.OnNext(1);
        observer.OnNext(2);
        observer.OnNext(3);
        observer.OnCompleted();
        return System.Reactive.Disposables.Disposable.Empty; // 便捷且不做任何事的 IDisposable
    }
}
```

我們可以透過建構其實例並訂閱它來進行測試：

```csharp
var numbers = new MySequenceOfNumbers();
numbers.Subscribe(
    number => Console.WriteLine($"Received value: {number}"),
    () => Console.WriteLine("Sequence terminated"));
```

這會產生以下輸出：

```
Received value 1
Received value 2
Received value 3
Sequence terminated
```

雖然 `MySequenceOfNumbers` 在技術上是 `IObservable<int>` 的正確實作，但它過於簡單以至於不太實用。首先，我們通常在有感興趣的事件時才會使用 Rx，但這根本不是真正的反應式——它只是立即產生一組固定的數字。此外，此實作是阻塞式的——在完成產生所有值之前，它甚至不會從 `Subscribe` 返回。這個範例展示了來源如何向訂閱者提供事件的基本概念，但如果我們只是想表示一個預先定義好的數字序列，我們不妨使用 `IEnumerable<T>` 的實作，例如 `List<T>` 或陣列。

## 在 Rx 中表示檔案系統事件

讓我們看一個更貼近真實情境的範例。這是圍繞 .NET 的 `FileSystemWatcher` 所建構的包裝器，將檔案系統變更通知呈現為 `IObservable<FileSystemEventArgs>`。（注意：對於 Rx `FileSystemWatcher` 包裝器來說，這不一定是最佳設計。監控器為幾種不同類型的變更提供事件，其中之一 `Renamed` 以 `RenamedEventArgs` 的形式提供詳細資訊。這衍生自 `FileSystemEventArgs`，因此將所有內容摺疊為單一事件串流確實可行，但對於需要存取重新命名事件詳細資訊的應用程式來說，這會很不方便。更嚴重的設計問題是，它無法回報來自 `FileSystemWatcher.Error` 的多個事件。這類錯誤可能是暫時性且可復原的，在此情況下應用程式可能希望繼續運作，但由於此類別選擇使用單一 `IObservable<T>` 表示所有內容，因此它透過呼叫觀察者的 `OnError` 來回報錯誤，此時 Rx 的規則要求我們必須停止。可以使用 Rx 的 `Retry` 運算子來解決此問題，該運算子可以在發生錯誤後自動重新訂閱，但提供一個單獨的 `IObservable<ErrorEventArgs>` 可能會更好，這樣我們就能以非終止的方式回報錯誤。然而，這種額外的複雜性並不總是必要的。此設計的簡潔性意味著它非常適合某些應用程式。就像軟體設計中常見的那樣，並不存在一體適用的方法。）

```csharp
// 將檔案系統變更表示為 Rx 可觀察序列。
// 注意：這是一個用於說明目的的過度簡化範例。
//       它無法有效率地處理多個訂閱者，也沒有使用
//       IScheduler，並且在遇到第一個錯誤後立即停止。
public class RxFsEvents : IObservable<FileSystemEventArgs>
{
    private readonly string folder;

    public RxFsEvents(string folder)
    {
        this.folder = folder;
    }

    public IDisposable Subscribe(IObserver<FileSystemEventArgs> observer)
    {
        // 如果有多個訂閱者，效率會很低。
        FileSystemWatcher watcher = new(this.folder);

        // FileSystemWatcher 的說明文件完全沒有提到
        // 它在哪個執行緒引發事件（除非你使用它的
        // SynchronizationObject，該物件能與 Windows Forms
        // 良好整合，但在這裡對我們來說不便使用），
        // 也沒有承諾在交付下一個事件之前
        // 會等待我們處理完前一個事件。
        // Mac、Windows 和 Linux 的實作都有顯著差異，
        // 因此依賴說明文件中未保證的任何行為都是不明智的。
        // （碰巧的是，.NET 7 上的 Win32 實作
        // 似乎確實會在交付下一個事件之前
        // 等待每個事件處理常式返回，所以我們暫時
        // 忽略這個問題可能也不會出錯。目前如此。在 Windows 上。
        // 而事實上 Linux 實作專門指派單一執行緒來處理這項工作，
        // 但原始程式碼中有一條註解表示這可能應該改變——
        // 這是僅依賴已記錄行為的另一個原因。）
        // 因此，確保遵循 IObserver<T> 的規則是我們的責任。
        // 首先，我們需要確保一次只對觀察者進行一次呼叫。
        // 更貼近實際的範例會使用 Rx IScheduler，
        // 但由於我們尚未解釋這些是什麼，我們在此物件中使用 lock。
        object sync = new();

        // 更微妙的是，FileSystemWatcher 說明文件
        // 並未明確指出在回報錯誤後，是否可能繼續
        // 收到一些變更事件。由於沒有關於執行緒的承諾，
        // 因此可能存在競態條件，導致我們在 FileSystemWatcher
        // 回報錯誤之後仍嘗試處理來自它的事件。
        // 因此我們需要記住是否已經呼叫過 OnError，
        // 以確保在這種情況下不會違反
        // IObserver<T> 規則。
        bool onErrorAlreadyCalled = false;

        void SendToObserver(object _, FileSystemEventArgs e)
        {
            lock (sync)
            {
                if (!onErrorAlreadyCalled)
                {
                    observer.OnNext(e); 
                }
            }
        }

        watcher.Created += SendToObserver;
        watcher.Changed += SendToObserver;
        watcher.Renamed += SendToObserver;
        watcher.Deleted += SendToObserver;

        watcher.Error += (_, e) =>
        {
            lock (sync)
            {
                // FileSystemWatcher 可能會回報多個錯誤，
                // 但我們只允許向 IObservable<T> 回報一個錯誤。
                if (!onErrorAlreadyCalled)
                {
                    observer.OnError(e.GetException());
                    onErrorAlreadyCalled = true; 
                    watcher.Dispose();
                }
            }
        };

        watcher.EnableRaisingEvents = true;

        return watcher;
    }
}
```

情況很快就變得更複雜了。這說明了 `IObservable<T>` 實作有責任遵循 `IObserver<T>` 規則。這通常是一件好事：它將並行處理的繁雜問題限制在單一地方。我訂閱到此 `RxFsEvents` 的任何 `IObserver<FileSystemEventArgs>` 都不需要擔心並行問題，因為它可以依賴 `IObserver<T>` 規則，保證一次只需要處理一件事。如果沒有要求我在來源中強制執行這些規則，我的 `RxFsEvents` 類別可能會更簡單，但處理重疊事件的所有複雜性就會擴散到處理事件的程式碼中。當並行的影響被局限住時，處理起來就已經夠困難了；一旦開始擴散到多個型別，幾乎就無法進行推導分析了。Rx 的 `IObserver<T>` 規則防止了這種情況的發生。

（注意：這是 Rx 的一個重要特點。這些規則為觀察者保持了簡單性。隨著事件來源或事件處理複雜性的增長，這變得越來越重要。）

這段程式碼有幾個問題（除了前面提到的 API 設計問題之外）。其一是當 `IObservable<T>` 實作產生的事件模擬現實生活中的非同步活動（例如檔案系統變更）時，應用程式通常會希望有某種方法來控制通知在哪個執行緒上送達。例如，UI 框架往往有執行緒關聯性需求。你通常需要位於特定執行緒上才能獲准更新使用者介面。Rx 提供了將通知重導向到不同排程器的機制，因此我們可以解決這個問題，但我們通常希望能夠向此類觀察者提供 `IScheduler`，並讓它透過該排程器交付通知。我們將在後續章節中討論排程器。

另一個問題是，這無法有效率地處理多個訂閱者。你可以多次呼叫 `IObservable<T>.Subscribe`，如果你對此程式碼這樣做，它每次都會建立一個新的 `FileSystemWatcher`。這種情況發生的頻率可能比你想像的要高。假設我們有這個監控器的執行個體，並希望以不同方式處理不同的事件。我們可以使用 `Where` 運算子來定義可觀察來源，依照我們想要的方式分割事件：

```csharp
IObservable<FileSystemEventArgs> configChanges =
    fs.Where(e => Path.GetExtension(e.Name) == ".config");
IObservable<FileSystemEventArgs> deletions =
    fs.Where(e => e.ChangeType == WatcherChangeTypes.Deleted);
```

當你在 `Where` 運算子返回的 `IObservable<T>` 上呼叫 `Subscribe` 時，它會在其輸入上呼叫 `Subscribe`。因此在這種情況下，如果我們在 `configChanges` 和 `deletions` 上都呼叫 `Subscribe`，那將導致對 `fs` 進行_兩次_ `Subscribe` 呼叫。因此，如果 `fs` 是我們上面 `RxFsEvents` 型別的執行個體，則每個呼叫都會建構自己的 `FileSystemEventWatcher`，這是不有效率的。

Rx 提供了幾種處理此問題的方法。它提供了專門設計的運算子，用於接收不支援多個訂閱者的 `IObservable<T>`，並將其包裝在可以支援的配接器中：

```csharp
IObservable<FileSystemEventArgs> fs =
    new RxFsEvents(@"c:\temp")
    .Publish()
    .RefCount();
```

但這有點言之過早了。（這些運算子在[發布運算子章節](15_PublishingOperators.md)中有詳細說明。）如果你想要建構一個本質上對多訂閱者友善的型別，你真正需要做的就是追蹤所有訂閱者，並在迴圈中通知他們每一個人。以下是檔案系統監控器的修改版本：

```csharp
public class RxFsEventsMultiSubscriber : IObservable<FileSystemEventArgs>
{
    private readonly object sync = new();
    private readonly List<Subscription> subscribers = new();
    private readonly FileSystemWatcher watcher;

    public RxFsEventsMultiSubscriber(string folder)
    {
        this.watcher = new FileSystemWatcher(folder);

        watcher.Created += SendEventToObservers;
        watcher.Changed += SendEventToObservers;
        watcher.Renamed += SendEventToObservers;
        watcher.Deleted += SendEventToObservers;

        watcher.Error += SendErrorToObservers;
    }

    public IDisposable Subscribe(IObserver<FileSystemEventArgs> observer)
    {
        Subscription sub = new(this, observer);
        lock (this.sync)
        {
            this.subscribers.Add(sub); 

            if (this.subscribers.Count == 1)
            {
                // 我們之前沒有訂閱者，但現在有了一個，
                // 所以我們需要啟動 FileSystemWatcher。
                watcher.EnableRaisingEvents = true;
            }
        }

        return sub;
    }

    private void Unsubscribe(Subscription sub)
    {
        lock (this.sync)
        {
            this.subscribers.Remove(sub);

            if (this.subscribers.Count == 0)
            {
                watcher.EnableRaisingEvents = false;
            }
        }
    }

    void SendEventToObservers(object _, FileSystemEventArgs e)
    {
        lock (this.sync)
        {
            foreach (var subscription in this.subscribers)
            {
                subscription.Observer.OnNext(e);
            }
        }
    }

    void SendErrorToObservers(object _, ErrorEventArgs e)
    {
        Exception x = e.GetException();
        lock (this.sync)
        {
            foreach (var subscription in this.subscribers)
            {
                subscription.Observer.OnError(x);
            }

            this.subscribers.Clear();
        }
    }

    private class Subscription : IDisposable
    {
        private RxFsEventsMultiSubscriber? parent;

        public Subscription(
            RxFsEventsMultiSubscriber rxFsEventsMultiSubscriber,
            IObserver<FileSystemEventArgs> observer)
        {
            this.parent = rxFsEventsMultiSubscriber;
            this.Observer = observer;
        }
        
        public IObserver<FileSystemEventArgs> Observer { get; }

        public void Dispose()
        {
            this.parent?.Unsubscribe(this);
            this.parent = null;
        }
    }
}
```

無論呼叫 `Subscribe` 多少次，這都只會建立單一 `FileSystemWatcher` 執行個體。請注意，我必須引入一個巢狀類別來提供 `Subscribe` 所返回的 `IDisposable`。在本章的第一個 `IObservable<T>` 實作中我不需要這樣做，因為它在返回之前就已經完成了序列，所以它能夠返回 Rx 便捷提供的 `Disposable.Empty` 屬性。（當你必須提供 `IDisposable`，但在處置時實際上不需要執行任何操作時，這很方便。）而在我的第一個 `FileSystemWatcher` 包裝器 `RxFsEvents` 中，我只是從 `Dispose` 返回 `FileSystemWatcher` 本身。（這之所以可行，是因為 `FileSystemWatcher.Dispose` 會關閉監控器，且每個訂閱者都被賦予了自己的 `FileSystemWatcher`。）但現在既然單一 `FileSystemWatcher` 支援多個觀察者，當觀察者取消訂閱時，我們需要做更多的工作。

當我們從 `Subscribe` 返回的 `Subscription` 執行個體被處置時，它會將自己從訂閱者清單中移除，確保不會再收到任何通知。如果沒有更多訂閱者，它還會將 `FileSystemWatcher` 的 `EnableRaisingEvents` 設定為 false，確保在目前不需要任何通知時，此來源不會執行不必要的工作。

這看起來比第一個範例更符合實際情況。這確實是一個可能在任何時刻發生事件的來源（這正是非常適合 Rx 的那種情境），而且它現在能智慧地處理多個訂閱者。然而，我們通常不會這樣寫程式碼。我們在這裡自己完成了所有工作——這段程式碼甚至不需要引用 `System.Reactive` 套件，因為它引用的唯一 Rx 型別是 `IObservable<T>` 和 `IObserver<T>`，這兩者都內建於 .NET 執行階段函式庫中。在實務上，我們通常會借助 `System.Reactive` 中的協助程式，因為它們可以替我們完成大量工作。

例如，假設我們只關心 `Changed` 事件。我們可以只寫成這樣：

```csharp
FileSystemWatcher watcher = new (@"c:\temp");
IObservable<FileSystemEventArgs> changes = Observable
    .FromEventPattern<FileSystemEventArgs>(watcher, nameof(watcher.Changed))
    .Select(ep => ep.EventArgs);
watcher.EnableRaisingEvents = true;
```

在這裡，我們使用了來自 `System.Reactive` 函式庫中 `Observable` 類別的 `FromEventPattern` 協助程式，它可以用來從符合標準模式的任何 .NET 事件建立 `IObservable<T>`（在該模式中，事件處理常式接受兩個引數：一個型別為 `object` 的 sender，以及包含事件資訊的某個衍生自 `EventArgs` 的型別）。這不像先前的範例那麼靈活。它僅回報其中一個事件，並且我們必須手動啟動（以及在必要時停止）`FileSystemWatcher`。但對於某些應用程式來說這已經足夠了，而且需要編寫的程式碼少得多。如果我們的目標是編寫一個適用於許多不同情境且功能齊全的 `FileSystemWatcher` 包裝器，那麼編寫如前所述的專用 `IObservable<T>` 實作可能是值得的。（我們可以輕鬆擴充這最後一個範例來監看所有事件。我們只需對每個事件使用一次 `FromEventPattern`，然後使用 `Observable.Merge` 將四個產生的可觀察序列合併為一個。我們從完整自訂實作中獲得的唯一實質好處是，我們可以根據目前是否存在任何觀察者來自動啟動和停止 `FileSystemWatcher`。）但如果我們只需要將某些事件表示為 `IObservable<T>` 以便在應用程式中處理它們，我們只需使用這種更簡單的方法即可。

在實務上，我們幾乎總是讓 `System.Reactive` 替我們實作 `IObservable<T>`。即使我們想控制某些層面（例如在這些範例中自動啟動和關閉 `FileSystemWatcher`），我們幾乎總能找到支援此功能的運算子組合。下列程式碼使用來自 `System.Reactive` 的各種方法來返回一個 `IObservable<FileSystemEventArgs>`，它具有與上述功能齊全的手寫 `RxFsEventsMultiSubscriber` 相同的所有功能，但程式碼量顯著減少。

```csharp
IObservable<FileSystemEventArgs> ObserveFileSystem(string folder)
{
    return 
        // Observable.Defer 使我們能夠在有訂閱者之前
        // 避免執行任何工作。
        Observable.Defer(() =>
            {
                FileSystemWatcher fsw = new(folder);
                fsw.EnableRaisingEvents = true;

                return Observable.Return(fsw);
            })
        // 一旦前一部分發出 FileSystemWatcher
        // （這會在有人首次訂閱時發生），我們
        // 希望將所有事件包裝為 IObservable<T>，為此
        // 我們將使用投影。為了避免最終得到
        // IObservable<IObservable<FileSystemEventArgs>>，我們使用
        // SelectMany，這能有效地將其扁平化一層。
        .SelectMany(fsw =>
            Observable.Merge(new[]
                {
                    Observable.FromEventPattern<FileSystemEventHandler, FileSystemEventArgs>(
                        h => fsw.Created += h, h => fsw.Created -= h),
                    Observable.FromEventPattern<FileSystemEventHandler, FileSystemEventArgs>(
                        h => fsw.Changed += h, h => fsw.Changed -= h),
                    Observable.FromEventPattern<RenamedEventHandler, FileSystemEventArgs>(
                        h => fsw.Renamed += h, h => fsw.Renamed -= h),
                    Observable.FromEventPattern<FileSystemEventHandler, FileSystemEventArgs>(
                        h => fsw.Deleted += h, h => fsw.Deleted -= h)
                })
            // FromEventPattern 同時提供 sender 和事件引數。
            // 僅擷取後者。
            .Select(ep => ep.EventArgs)
            // 這裡的 Finally 確保在沒有訂閱者時
            // 監控器會被關閉。
            .Finally(() => fsw.Dispose()))
        // 這種 Publish 和 RefCount 的組合意味著多個
        // 訂閱者將共享單個 FileSystemWatcher，
        // 但如果所有訂閱者都取消訂閱，它就會被關閉。
        .Publish()
        .RefCount();
}
```

我在那裡使用了許多方法，其中大部分我之前都沒有談過。為了讓該範例有意義，顯然我需要開始介紹 `System.Reactive` 套件為你實作 `IObservable<T>` 的眾多方式。

## 簡單工廠方法

由於可用於建立可觀察序列的方法數量龐大，我們將它們分門別類。我們第一類方法建立的 `IObservable<T>` 序列最多只會產生單一結果。

### Observable.Return

最簡單的工廠方法之一是 `Observable.Return<T>(T value)`，你已經在上一章的 `Quiescent` 範例中看過它了。此方法接受型別為 `T` 的值，並返回一個 `IObservable<T>`，該序列會產生此單一值然後完成。在某種意義上，這是在 `IObservable<T>` 中_包裝_一個值；它在概念上類似於編寫 `new T[] { value }`，因為它是一個僅包含一個元素的序列。你也可以將其視為 Rx 對應於 `Task.FromResult` 的等價物，當你擁有某個型別 `T` 的值，並且需要將其傳遞給需要 `Task<T>` 的項目時可以使用它。

```csharp
IObservable<string> singleValue = Observable.Return<string>("Value");
```

為了清晰起見，我指定了型別參數，但這不是必需的，因為編譯器可以從提供的引數推斷型別：

```csharp
IObservable<string> singleValue = Observable.Return("Value");
```

`Return` 會產生一個冷可觀察序列：每個訂閱者在訂閱時都會立即收到該值。（[冷熱可觀察序列](02_KeyTypes.md#hot-and-cold-sources)已在上一章中介紹過。）

### Observable.Empty

有時擁有一個空序列是很有用的。.NET 的 `Enumerable.Empty<T>()` 為 `IEnumerable<T>` 提供了此功能，而 Rx 以 `Observable.Empty<T>()` 的形式提供了直接的等價物，它返回一個空的 `IObservable<T>`。我們需要提供型別引數，因為沒有值可供編譯器推斷型別。

```csharp
IObservable<string> empty = Observable.Empty<string>();
```

在實務上，空序列是指立即對任何訂閱者呼叫 `OnCompleted` 的序列。

與 `IEnumerable<T>` 相比，這只是空清單的 Rx 等價物，但還有另一種看待它的方式。Rx 是建構非同步程序模型的一種強大方式，因此你可以將其視為類似於立即完成而不產生任何結果的工作（Task）——因此它在概念上類似於 `Task.CompletedTask`。（這不像 `Observable.Return` 和 `Task.FromResult` 之間的類比那麼精確，因為在那種情況下我們是在比較 `IObservable<T>` 與 `Task<T>`，而在這裡我們是在比較 `IObservable<T>` 與 `Task`——工作在不產生任何內容的情況下完成的唯一方法是如果我們使用非泛型版本的 `Task`。）

### Observable.Never

`Observable.Never<T>()` 方法返回一個序列，該序列與 `Empty` 一樣不產生任何值，但與 `Empty` 不同的是，它永遠不會結束。在實務上，這意味著它永遠不會對訂閱者呼叫任何方法（既不呼叫 `OnNext`、`OnCompleted`，也不呼叫 `OnError`）。`Observable.Empty<T>()` 會立即完成，而 `Observable.Never<T>` 則具有無限的持續時間。

```csharp
IObservable<string> never = Observable.Never<string>();
```

這為什麼有用可能看起來並不明顯。我在上一章給出了一個可能的用途：你可以在測試中使用它來模擬一個不產生任何值的來源，或許是為了讓你的測試驗證逾時邏輯。

它也可以用於我們使用可觀察物件來表示基於時間的資訊的地方。有時我們實際上並不在乎可觀察物件發出什麼；我們可能只在乎某事（任何事）_何時_發生。（我們在上一章中看過這個「純粹用於計時目的的可觀察序列」概念的範例，儘管 `Never` 在該特定情境中沒有意義。`Quiescent` 範例使用了 `Buffer` 運算子，它在兩個可觀察序列上運作：第一個包含感興趣的項目，第二個純粹用於確定如何將第一個切成區塊。`Buffer` 不會對第二個可觀察物件產生的值執行任何操作：它只關注值_何時_出現，每次第二個可觀察物件產生值時完成前一個區塊。如果我們表示時間資訊，有時擁有一種方法來表示某個事件永遠不會發生的概念會很有用。）

舉一個你可能想將 `Never` 用於計時目的的範例：假設你正在使用某個提供逾時機制的 Rx 函式庫，當發生某個逾時時作業將被取消，且逾時本身被模型化為可觀察序列。如果由於某種原因你不想設定逾時，只想無限期等待，你可以將逾時指定為 `Observable.Never`。

### Observable.Throw

`Observable.Throw<T>(Exception)` 返回一個立即向任何訂閱者回報錯誤的序列。與 `Empty` 和 `Never` 一樣，我們不會向此方法提供值（僅提供例外狀況），因此我們需要提供型別參數，以便它知道在其返回的 `IObservable<T>` 中使用什麼 `T`。（它實際上永遠不會產生 `T`，但如果不為 `T` 選擇某個特定型別，你就無法擁有 `IObservable<T>` 的執行個體。）

```csharp
IObservable<string> throws = Observable.Throw<string>(new Exception()); 
```

請注意，如果你將此運算子與[離開 Rx 世界](13_LeavingIObservable.md)章節中描述的任何機制結合使用，你可能會違反[例外狀況狀態](13_LeavingIObservable.md#exception-state)中描述的規則。如果你需要使用 `await`（或類似的會將 `OnError` 呼叫轉換為重新擲回的機制），你可能需要使用 [`ResetExceptionDispatchState`](13_LeavingIObservable.md#resetexceptiondispatchstate) 運算子來確保每次重新擲回例外狀況時都能獲得適當重設的例外狀況狀態。（只有當你導致同一個例外狀況被多次重新擲回時，才會出現此問題。）


### Observable.Create

`Create` 工廠方法比其他建立方法更強大，因為它可以用來建立任何類型的序列。你可以使用 `Observable.Create` 來實作前面四個方法中的任何一個。
方法簽名本身起初可能看起來比必要的更複雜，但一旦習慣它就會變得非常自然。

```csharp
// 根據指定的 Subscribe 方法實作建立可觀察序列。
public static IObservable<TSource> Create<TSource>(
    Func<IObserver<TSource>, IDisposable> subscribe)
{...}
public static IObservable<TSource> Create<TSource>(
    Func<IObserver<TSource>, Action> subscribe)
{...}
```

你向其提供一個委派，該委派將在每次建立訂閱時執行。你的委派將接收一個 `IObserver<T>`。從邏輯上講，這表示傳遞給 `Subscribe` 方法的觀察者，儘管在實務上 Rx 出於各種原因在其周圍放置了一個包裝器。你可以根據需要呼叫 `OnNext`/`OnError`/`OnCompleted` 方法。這是你直接使用 `IObserver<T>` 介面的少數情境之一。這是一個產生三個項目的簡單範例：

```csharp
private IObservable<int> SomeNumbers()
{
    return Observable.Create<int>(
        (IObserver<int> observer) =>
        {
            observer.OnNext(1);
            observer.OnNext(2);
            observer.OnNext(3);
            observer.OnCompleted();

            return Disposable.Empty;
        });
}
```

你的委派必須返回 `IDisposable` 或 `Action` 以啟用取消訂閱。當訂閱者處置其訂閱以取消訂閱時，Rx 將呼叫你返回的 `IDisposable` 上的 `Dispose()`，或者在你返回 `Action` 的情況下，它將呼叫該委派。

這個範例讓人想起本章開頭的 `MySequenceOfNumbers` 範例，因為它立即產生一些固定值。在這種情況下的主要區別是 Rx 加入了一些可以處理棘手情況（例如可重新進入性）的包裝器。Rx 有時會自動延遲工作以防止死結，因此使用此方法返回的 `IObservable<int>` 的程式碼可能會在上述程式碼中的回呼執行之前看到對 `Subscribe` 的呼叫返回，在這種情況下，他們可以在其 `OnNext` 處理常式內取消訂閱。

以下循序圖顯示了這在實務中如何發生。假設 `SomeNumbers` 返回的 `IObservable<int>` 已被 Rx 包裝，以確保訂閱發生在某些不同的執行內容中。我們通常透過使用適當的[排程器](11_SchedulingAndThreading.md#schedulers)來確定上下文。（[`SubscribeOn`](11_SchedulingAndThreading.md#subscribeon-and-observeon) 運算子會建立這樣的包裝器。）我們可以使用 [`TaskPoolScheduler`](11_SchedulingAndThreading.md#taskpoolscheduler) 來確保訂閱發生在某個工作集區執行緒上。因此，當我們的應用程式程式碼呼叫 `Subscribe` 時，包裝器 `IObservable<int>` 不會立即訂閱底層的可觀察物件。相反，它會在排程器中將一個工作項目排入佇列來執行該操作，然後立即返回而不等待該工作執行。這就是為什麼我們的訂閱者可以在 `Observable.Create` 呼叫我們的回呼之前擁有表示訂閱的 `IDisposable`。該圖顯示了訂閱者隨後將其提供給觀察者。

![一個包含 6 個參與者的循序圖：Subscriber、Rx IObservable Wrapper、Scheduler、Observable.Create、Rx IObserver Wrapper 和 Observer。它顯示了以下訊息。Subscriber 向 Rx IObservable Wrapper 發送 "Subscribe()"。Rx IObservable Wrapper 向 Scheduler 發送 "Schedule Subscribe()"。Rx IObservable Wrapper 向 Subscriber 返回 "IDisposable (subscription)"。Subscriber 向 Observer 發送 "Set subscription IDisposable"。Scheduler 向 Observable.Create 發送 "Subscribe()"。Observable.Create 向 Rx IObserver Wrapper 發送 "OnNext(1)"。Rx IObserver Wrapper 向 Observer 發送 "OnNext(1)"。Observable.Create 向 Rx IObserver Wrapper 發送 "OnNext(2)"。Rx IObserver Wrapper 向 Observer 發送 "OnNext(2)"。Observer 向 Rx IObservable Wrapper 發送 "subscription.Dispose()"。Observable.Create 向 Rx IObserver Wrapper 發送 "OnNext(3)"。Observable.Create 向 Rx IObserver Wrapper 發送 "OnCompleted()"。](GraphicsIntro/Ch03-Sequence-CreateWrappers.svg)

該圖顯示排程器在此之後呼叫底層可觀察物件上的 `Subscribe`，這意味著我們傳遞給 `Observable.Create<int>` 的回呼現在將執行。我們的回呼呼叫 `OnNext`，但傳遞給它的不是真正的觀察者：相反，傳遞給它的是另一個 Rx 產生的包裝器。該包裝器最初將呼叫直接轉發給真正的觀察者，但我們的圖表顯示，當真正的觀察者（位於最右側）收到其第二個呼叫（`OnNext(2)`）時，它透過對訂閱 Rx `IObservable` 包裝器時返回的 `IDisposable` 呼叫 `Dispose` 來取消訂閱。這裡的兩個包裝器——`IObservable` 和 `IObserver` 包裝器——是相連的，因此當我們從 `IObservable` 包裝器取消訂閱時，它會告訴 `IObserver` 包裝器該訂閱正在關閉。這意味著當我們的 `Observable.Create<int>` 回呼在 `IObserver` 包裝器上呼叫 `OnNext(3)` 時，該包裝器_不會_將其轉發給真正的觀察者，因為它知道該觀察者已經取消訂閱。（出於同樣的原因，它也不會轉發 `OnCompleted`。）

你可能會想知道我們返回給 `Observable.Create` 的 `IDisposable` 究竟如何能發揮任何作用。它是回呼的返回值，因此我們只能在回呼執行的最後一步將其返回給 Rx。在我們返回時，我們難道不會總是已經完成了工作，意味著沒有什麼可以取消的嗎？不一定——我們可能會啟動一些在我們返回後繼續執行的工作。下一個範例就是這樣做的，這意味著它返回的取消訂閱動作能夠執行有用的操作：它設定了一個取消語彙基元，該語彙基元正由產生我們可觀察物件輸出的迴圈所觀察。（這返回一個回呼而不是 `IDisposable`——`Observable.Create` 提供的多載允許你執行這兩者中的任一操作。在這種情況下，當訂閱提前終止時，Rx 將呼叫我們的回呼。）

```csharp
IObservable<char> KeyPresses() =>
    Observable.Create<char>(observer =>
    {
        CancellationTokenSource cts = new();
        Task.Run(() =>
        {
            while (!cts.IsCancellationRequested)
            {
                ConsoleKeyInfo ki = Console.ReadKey();
                observer.OnNext(ki.KeyChar);
            }
        });

        return () => cts.Cancel();
    });
```

這說明了取消不一定會立即生效。`Console.ReadKey` API 沒有提供接受 `CancellationToken` 的多載，因此在使用者下次按下按鍵導致 `ReadKey` 返回之前，此可觀察物件將無法偵測到已請求取消。

考慮到在我們等待 `ReadKey` 返回時可能已經請求了取消，你可能會認為我們應該在 `ReadKey` 返回之後且在呼叫 `OnNext` 之前對此進行檢查。事實上，如果我們不這樣做也沒關係。Rx 有一條規則：可觀察來源_絕對不能_在該觀察者訂閱上的 `Dispose` 呼叫返回_之後_呼叫觀察者。為了強制執行該規則，如果你傳遞給 `Observable.Create` 的回呼在請求取消訂閱後繼續在其 `IObserver<T>` 上呼叫方法，Rx 只會忽略該呼叫。這就是為什麼它傳遞給你的 `IObserver<T>` 是一個包裝器的原因之一：它可以在呼叫傳遞給底層觀察者之前攔截它們。然而，這種便利性意味著有兩件重要的事情需要注意：
1. 如果你_確實_忽略了取消訂閱的嘗試並繼續執行工作以產生項目，你只是在浪費時間，因為沒有任何東西會接收這些項目
2. 如果你呼叫 `OnError`，可能沒有任何東西在接聽，且該錯誤將被完全忽略。

`Create` 有一些旨在支援 `async` 方法的多載。下一個方法利用了這一點，以便能夠使用非同步 `ReadLineAsync` 方法將檔案中的文字行呈現為可觀察來源。

```csharp
IObservable<string> ReadFileLines(string path) =>
    Observable.Create<string>(async (observer, cancellationToken) =>
    {
        using (StreamReader reader = File.OpenText(path))
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                string? line = await reader.ReadLineAsync(cancellationToken).ConfigureAwait(false);
                if (line is null)
                {
                    break;
                }

                observer.OnNext(line);
            }

            observer.OnCompleted();
        }
    });
```

從儲存裝置讀取資料通常不會瞬間發生（除非它碰巧已經在檔案系統快取中），因此該來源將以能夠從儲存裝置讀取的速度盡快提供資料。

請注意，因為這是一個 `async` 方法，所以它通常會在完成之前返回給其呼叫者。（第一個真正必須等待的 `await` 會返回，並且該方法的其餘部分會在工作完成時透過回呼執行。）這意味著訂閱者通常會在該方法完成之前擁有表示其訂閱的 `IDisposable`，因此我們在這裡使用不同的機制來處理取消訂閱。`Create` 的這個特定多載傳遞給其回呼的不僅是 `IObserver<T>`，還有一個 `CancellationToken`，當發生取消訂閱時，它將透過該語彙基元請求取消。

檔案 IO 可能會遇到錯誤。我們尋找的檔案可能不存在，或者我們可能由於安全性限制或因為其他應用程式正在使用它而無法開啟它。該檔案可能位於遠端儲存伺服器上，我們可能會失去網路連線。因此，我們必須預期此類程式碼會出現例外狀況。這個範例沒有採取任何措施來偵測例外狀況，但此 `ReadFileLines` 方法返回的 `IObservable<string>` 實際上會回報發生的任何例外狀況。這是因為 `Create` 方法將攔截從我們的回呼中出現的任何例外狀況，並使用 `OnError` 回報它。（如果我們的程式碼已經在觀察者上呼叫了 `OnComplete`，Rx 將不會呼叫 `OnError`，因為那樣會違反規則。相反，它會以無訊息方式捨棄該例外狀況，因此最好不要在呼叫 `OnCompleted` 後嘗試執行任何工作。）

這種自動例外狀況交付是為什麼 `Create` 工廠方法是實作自訂可觀察序列的首選方式的另一個範例。它幾乎總是比建立實作 `IObservable<T>` 介面的自訂型別更好的選擇。這不僅僅是因為它為你節省了一些時間。也是因為 Rx 處理了你可能沒有想到的錯綜複雜之處，例如通知的執行緒安全和訂閱的處置。

`Create` 方法需要延遲求值（lazy evaluation），這是 Rx 非常重要的一部分。它為我們稍後將看到的其他強大功能（例如排程和序列組合）開啟了大門。委派僅在建立訂閱時才會被呼叫。因此，在 `ReadFileLines` 範例中，在訂閱返回的 `IObservable<string>` 之前，它不會嘗試開啟檔案。如果你多次訂閱，它將在每次訂閱時執行回呼。（因此，如果檔案已變更，你可以透過再次呼叫 `Subscribe` 來擷取最新內容。）

作為一項練習，請嘗試使用 `Create` 方法自行建構 `Empty`、`Return`、`Never` 與 `Throw` 擴充方法。如果你現在可以使用 Visual Studio 或 [LINQPad](http://www.linqpad.net/)，請盡快編寫程式碼；或者如果你有 Visual Studio Code，你可以建立一個新的 [Polyglot Notebook](https://code.visualstudio.com/docs/languages/polyglot)。（Polyglot Notebooks 會自動提供 Rx，因此你只需編寫一個帶有適當 `using` 指示詞的 C# 儲存格即可開始執行。）如果你沒有這些工具（也許你在上班途中的火車上），請嘗試在腦海中構想你將如何解決這個問題。

在進入下一段之前，你已經完成了上一步，對吧？因為你現在可以將你的版本與使用 `Observable.Create` 重新建立的 `Empty`、`Return`、`Never` 和 `Throw` 的這些範例進行比較：

```csharp
public static IObservable<T> Empty<T>()
{
    return Observable.Create<T>(o =>
    {
        o.OnCompleted();
        return Disposable.Empty;
    });
}

public static IObservable<T> Return<T>(T value)
{
    return Observable.Create<T>(o =>
    {
        o.OnNext(value);
        o.OnCompleted();
        return Disposable.Empty;
    });
}

public static IObservable<T> Never<T>()
{
    return Observable.Create<T>(o =>
    {
        return Disposable.Empty;
    });
}

public static IObservable<T> Throw<T>(Exception exception)
{
    return Observable.Create<T>(o =>
    {
        o.OnError(exception);
        return Disposable.Empty;
    });
}
```

你可以看到 `Observable.Create` 提供了根據我們的意願建構自訂工廠方法的能力。

### Observable.Defer

`Observable.Create` 的一個非常有用的面向是，它提供了一個放置僅應在發生訂閱時執行的程式碼的位置。通常，函式庫會提供不一定會被所有應用程式使用的 `IObservable<T>` 屬性，因此將涉及的工作延遲到你確定確實需要它為止可能很有用。這種延遲初始化是 `Observable.Create` 運作方式所固有的，但如果我們來源的本質意味著 `Observable.Create` 不合適該怎麼辦？在這種情況下我們如何執行延遲初始化？Rx 為此目的提供了 `Observable.Defer`。

我已經使用過一次 `Defer` 了。`ObserveFileSystem` 方法返回了一個回報資料夾中變更的 `IObservable<FileSystemEventArgs>`。它不適合使用 `Observable.Create`，因為它以 .NET 事件的形式提供了我們想要的所有通知，因此使用 Rx 的事件配接功能是有意義的。但我們仍然希望將 `FileSystemWatcher` 的建立延遲到訂閱的那一刻，這就是為什麼該範例使用 `Observable.Defer` 的原因。

`Observable.Defer` 接受一個返回 `IObservable<T>` 的回呼，並且 `Defer` 將其包裝為一個在訂閱時呼叫該回呼的 `IObservable<T>`。為了展示其效果，我首先將展示一個不使用 `Defer` 的範例：

```csharp
static IObservable<int> WithoutDeferal()
{
    Console.WriteLine("Doing some startup work...");
    return Observable.Range(1, 3);
}

Console.WriteLine("Calling factory method");
IObservable<int> s = WithoutDeferal();

Console.WriteLine("First subscription");
s.Subscribe(Console.WriteLine);

Console.WriteLine("Second subscription");
s.Subscribe(Console.WriteLine);
```

這會產生以下輸出：

```
Calling factory method
Doing some startup work...
First subscription
1
2
3
Second subscription
1
2
3
```

如你所見，當我們呼叫工廠方法時，以及在我們訂閱之前，會出現 `"Doing some startup work...` 訊息。因此，如果從未有任何東西訂閱該方法返回的 `IObservable<int>`，該工作無論如何都會被執行，從而浪費時間和精力。以下是 `Defer` 版本：

```csharp
static IObservable<int> WithDeferal()
{
    return Observable.Defer(() =>
    {
        Console.WriteLine("Doing some startup work...");
        return Observable.Range(1, 3);
    });
}
```

如果我們將此版本與類似於第一個範例的程式碼一起使用，我們將看到以下輸出：

```
Calling factory method
First subscription
Doing some startup work...
1
2
3
Second subscription
Doing some startup work...
1
2
3
```

有兩個重要的區別。首先，`"Doing some startup work..."` 訊息直到我們首次訂閱時才會出現，這說明 `Defer` 實現了我們想要的效果。然而，請注意該訊息現在出現了兩次：每次我們訂閱時它都會執行此工作。如果你想要這種延遲初始化，但同時希望僅執行一次，你應該查看[發布運算子章節](15_PublishingOperators.md)中的運算子，它們提供了各種方法以允許多個訂閱者共用對底層來源的單一訂閱。

## 序列產生器

我們目前為止看到的建立方法都很直接，因為它們要麼產生非常簡單的序列（例如單元素或空序列），要麼依賴我們的程式碼明確告訴它們要產生什麼。現在我們將看一些可以產生較長序列的方法。

### Observable.Range

`Observable.Range(int, int)` 返回一個產生整數範圍的 `IObservable<int>`。第一個整數是起始值，第二個是要產生的值數量。此範例將寫入從 '10' 到 '24' 的值然後完成。

```csharp
IObservable<int> range = Observable.Range(10, 15);
range.Subscribe(Console.WriteLine, () => Console.WriteLine("Completed"));
```

### Observable.Generate

假設你想使用 `Observable.Create` 模擬 `Range` 工廠方法。你可能會嘗試這樣做：

```csharp
// 這不是最好的做法！
IObservable<int> Range(int start, int count) =>
    Observable.Create<int>(observer =>
        {
            for (int i = 0; i < count; ++i)
            {
                observer.OnNext(start + i);
            }

            return Disposable.Empty;
        });
```

這可以運作，但它不遵循取消訂閱的請求。這不會造成直接損害，因為 Rx 會偵測取消訂閱，並直接忽略我們產生的任何後續值。然而，在沒有人接聽時繼續產生數字是浪費 CPU 時間（從而浪費能源，並對電池壽命和/或環境造成影響）。其嚴重程度取決於所請求的範圍有多長。但想像一下你想要一個無限序列？也許擁有一個產生費氏數列（Fibonacci sequence）值或質數的 `IObservable<BigInteger>` 對你很有用。你會如何使用 `Create` 編寫它？在這種情況下，你肯定希望有某種處理取消訂閱的方法。如果要收到取消訂閱的通知，我們需要回呼返回（或者我們可以提供一個 `async` 方法，但在這裡似乎並不合適）。

這裡有另一種效果更好的方法：`Observable.Generate`。`Observable.Generate` 的簡單版本接受以下參數：

- 初始狀態
- 定義序列何時應終止的述詞（predicate）
- 套用於目前狀態以產生下一個狀態的函式
- 將狀態轉換為所需輸出的函式

```csharp
public static IObservable<TResult> Generate<TState, TResult>(
    TState initialState, 
    Func<TState, bool> condition, 
    Func<TState, TState> iterate, 
    Func<TState, TResult> resultSelector)
```

這展示了你如何使用 `Observable.Generate` 來建構 `Range` 方法：

```csharp
// 僅供範例參考的程式碼
public static IObservable<int> Range(int start, int count)
{
    int max = start + count;
    return Observable.Generate(
        start, 
        value => value < max, 
        value => value + 1, 
        value => value);
}
```

`Generate` 方法會反覆呼叫我們，直到我們的 `condition` 回呼表示已完成，或者觀察者取消訂閱為止。我們可以透過從不表示完成來定義無限序列：

```csharp
IObservable<BigInteger> Fibonacci()
{
    return Observable.Generate(
        (v1: new BigInteger(1), v2: new BigInteger(1)),
        value => true, // 永不結束！
        value => (value.v2, value.v1 + value.v2),
        value => value.v1);
}
```

## 定時序列產生器

我們目前為止介紹的大多數方法返回的序列都會立即產生其所有值。（唯一的例外是我們呼叫 `Observable.Create` 並在準備就緒時才產生值的情況。）然而，Rx 能夠依據排程產生序列。

正如我們將看到的，依據排程執行工作的運算子是透過稱為_排程器_（scheduler）的抽象來進行的。如果你未指定排程器，它們將選擇預設排程器，但有時計時器機制非常重要。例如，有些計時器能與 UI 框架整合，在傳遞滑鼠點擊和其他輸入的相同執行緒上傳遞通知，我們可能希望 Rx 的時間架構運算子使用這些計時器。出於測試目的，將時間虛擬化可能很有用，這樣我們就可以驗證時間敏感程式碼中發生的情況，而不必等待測試即時執行。

排程器是一個複雜的主題，超出了本章的討論範圍，但在後續關於[排程與執行緒](11_SchedulingAndThreading.md)的章節中將詳細介紹。

產生定時事件的方法有三種。

### Observable.Interval

第一種是 `Observable.Interval(TimeSpan)`，它將根據你選擇的頻率發布從零開始的遞增值。

此範例每 250 毫秒發布一次值。

```csharp
IObservable<long> interval = Observable.Interval(TimeSpan.FromMilliseconds(250));
interval.Subscribe(
    Console.WriteLine, 
    () => Console.WriteLine("completed"));
```

輸出：

```
0
1
2
3
4
5
```

一旦訂閱，你必須處置你的訂閱以停止序列，因為 `Interval` 返回一個無限序列。Rx 假定你可能具有相當大的耐心，因為 `Interval` 返回的序列型別為 `IObservable<long>`（是 `long`，而不是 `int`），這意味著即使你產生超過微不足道的 21.475 億個事件（即大於 `int.MaxValue`），也不會遇到問題。

### Observable.Timer

產生基於固定時間序列的第二個工廠方法是 `Observable.Timer`。它有幾個多載。最基本的一個就像 `Observable.Interval` 一樣只接受一個 `TimeSpan`。但與 `Observable.Interval` 不同的是，`Observable.Timer` 將在經過該段時間後發布恰好一個值（數字 0），然後它將完成。

```csharp
var timer = Observable.Timer(TimeSpan.FromSeconds(1));
timer.Subscribe(
    Console.WriteLine, 
    () => Console.WriteLine("completed"));
```

輸出：

```
0
completed
```

或者，你可以為 `dueTime` 參數提供一個 `DateTimeOffset`。這將在指定的時間產生值 0 並完成。

另一組多載加入了一個 `TimeSpan`，指出產生後續值的週期。這允許我們產生無限序列。它還展示了 `Observable.Interval` 實際上只是 `Observable.Timer` 的特例。`Interval` 可以像這樣實作：

```csharp
public static IObservable<long> Interval(TimeSpan period)
{
    return Observable.Timer(period, period);
}
```

雖然 `Observable.Interval` 在產生第一個值之前總是會等待指定的週期，但這個 `Observable.Timer` 多載使你能夠在選擇的時間啟動序列。使用 `Observable.Timer`，你可以編寫以下程式碼以擁有立即啟動的間隔序列。

```csharp
Observable.Timer(TimeSpan.Zero, period);
```

這將我們帶到了產生計時器相關序列的第三種也是最通用的方式，回到 `Observable.Generate`。

### 定時 Observable.Generate

 `Observable.Generate` 有一個更複雜的多載，允許你提供一個函式來指定下一個值的到期時間（due time）。

```csharp
public static IObservable<TResult> Generate<TState, TResult>(
    TState initialState, 
    Func<TState, bool> condition, 
    Func<TState, TState> iterate, 
    Func<TState, TResult> resultSelector, 
    Func<TState, TimeSpan> timeSelector)
```

額外的 `timeSelector` 引數讓我們告訴 `Generate` 何時產生下一個項目。我們可以使用它來編寫我們自己的 `Observable.Timer` 實作（正如你已經看到的，這反過來使我們能夠編寫自己的 `Observable.Interval`）。

```csharp
public static IObservable<long> Timer(TimeSpan dueTime)
{
    return Observable.Generate(
        0l,
        i => i < 1,
        i => i + 1,
        i => i,
        i => dueTime);
}

public static IObservable<long> Timer(TimeSpan dueTime, TimeSpan period)
{
    return Observable.Generate(
        0l,
        i => true,
        i => i + 1,
        i => i,
        i => i == 0 ? dueTime : period);
}

public static IObservable<long> Interval(TimeSpan period)
{
    return Observable.Generate(
        0l,
        i => true,
        i => i + 1,
        i => i,
        i => period);
}
```

這展示了你如何使用 `Observable.Generate` 來產生無限序列。我將留給身為讀者的你一個使用 `Observable.Generate` 以不同速率產生值的練習。

## 可觀察序列與狀態

正如 `Observable.Generate` 特別清楚展現的那樣，可觀察序列可能需要維護狀態。在該運算子中，它是顯式的——我們傳入初始狀態，並提供一個回呼在每次反覆運算中更新它。許多其他運算子也維護內部狀態。`Timer` 會記住其刻度計數，而且更微妙的是，它必須以某種方式記錄何時引發了最後一個事件以及下一個事件何時到期。正如你將在後續章節中看到的，許多其他運算子需要記住有關它們已經看到的內容的資訊。

這引出了一個有趣的問題：如果處理序關閉會發生什麼？有沒有辦法保存該狀態，並在新的處理序中重構它。

對於一般的 Rx.NET，答案是否定的：所有此類狀態都完全保存在記憶體中，無法獲取該狀態，也無法要求執行中的訂閱序列化其目前狀態。這意味著如果你正在處理特別長時間執行的作業，你需要自行解決如何重新啟動的問題，並且不能依賴 `System.Reactive` 來協助你。然而，有一個相關的基於 Rx 的函式庫集合，通稱為 [Reaqtive 函式庫](https://reaqtive.net/)。它們提供了與 `System.Reactive` 大多數相同運算子的實作，但採用的形式使你可以收集目前狀態，並從先前保存的狀態重新建立新的訂閱。這些函式庫還包含一個名為 Reaqtor 的元件，它是一種託管技術，可以管理自動檢查點建立（checkpointing）和崩潰後復原，透過使訂閱具有持續性和可靠性，從而支援執行時間非常長的 Rx 邏輯。請注意，這目前尚未形成任何產品化形式，因此你需要做相當多的工作才能使用它，但如果你需要可持續保存的 Rx 版本，請知悉它的存在。

## 將常見型別配接至 `IObservable<T>`

雖然我們現在已經看到了兩種產生任意序列的非常通用的方法——`Create` 與 `Generate`——但如果你已經有了某種其他形式的現有資訊來源，並希望將其作為 `IObservable<T>` 提供，該怎麼辦？Rx 為常見來源型別提供了一些配接器。

### 來自委派

`Observable.Start` 方法允許你將長時間執行的 `Func<T>` 或 `Action` 轉換為單值可觀察序列。該動作是透過[排程器](11_SchedulingAndThreading.md#schedulers)呼叫的。如果你沒有明確傳遞排程器，這將使用 [`DefaultScheduler`](11_SchedulingAndThreading.md#defaultscheduler)，它透過執行緒集區呼叫回呼。如果你使用的多載是 `Func<T>`，則返回型別將是 `IObservable<T>`。當函式返回其值時，`IObservable<T>` 將向訂閱者提供該值，並在提供該值後立即完成。（`Start` 返回的 `IObservable<T>` 基於 [`AsyncSubject`](#asyncsubjectt)，因此如果你在回呼完成後訂閱它，它將立即提供該值然後完成。）如果你使用接受 `Action` 的多載，則返回的序列型別將是 `IObservable<Unit>`。`Unit` 型別表示沒有資訊，因此它在某種程度上類似於 `void`，不同之處在於你可以擁有 `Unit` 型別的執行個體。它在 Rx 中特別有用，因為我們通常只關心某件事何時發生，而且除了時機之外可能沒有任何資訊。在這些情況下，我們通常使用 `IObservable<Unit>`，以便即使其中沒有有意義的資料也可以產生明確的事件。（該名稱來自函數式程式設計領域，該領域經常使用此類結構。）在這種情況下，`Unit` 用於發布 `Action` 已完成的確認，因為 `Action` 不返回任何資訊。`Unit` 型別本身沒有值；它僅作為 `OnNext` 通知的空負載（payload）。以下是使用這兩種多載的範例。

```csharp
static void StartAction()
{
    var start = Observable.Start(() =>
        {
            Console.Write("Working away");
            for (int i = 0; i < 10; i++)
            {
                Thread.Sleep(100);
                Console.Write(".");
            }
        });

    start.Subscribe(
        unit => Console.WriteLine("Unit published"), 
        () => Console.WriteLine("Action completed"));
}

static void StartFunc()
{
    var start = Observable.Start(() =>
    {
        Console.Write("Working away");
        for (int i = 0; i < 10; i++)
        {
            Thread.Sleep(100);
            Console.Write(".");
        }
        return "Published value";
    });

    start.Subscribe(
        Console.WriteLine, 
        () => Console.WriteLine("Action completed"));
}
```

請注意 `Observable.Start` 和 `Observable.Return` 之間的區別。`Return` 要求我們預先提供該值，而 `Start` 立即返回一個可觀察序列，而不需要該值立即可用。（雖然 `Start` 不等待回呼完成，但它確實會立即呼叫它。因此這不是延遲求值——如果你想提供一個僅在有人訂閱來源時才被呼叫的回呼，請使用 [`Defer`](#observabledefer)。）

由 `Start` 返回的可觀察物件可以被視為表示與 `Task` 或 `Task<T>` 相同的基本概念（取決於你使用的是 `Action` 還是 `Func<T>` 多載）。每個都表示可能需要一些時間才能最終完成的工作，也許會產生結果。因此，如果你想要該基本概念，但希望將其表示為 `IObservable<T>` 而不是 `Task` 或 `Task<T>`，則 `Start` 非常有用。

### 來自事件

正如我們在本書前面所討論的，.NET 有一個融入其型別系統的事件模型。這早於 Rx 出現（尤其是因為直到 .NET 在 .NET 2.0 中獲得泛型之前，Rx 是不可行的），因此型別支援事件但不支援 Rx 是很常見的。為了能夠與現有的事件模型整合，Rx 提供了將事件轉換為可觀察序列的方法。我在前面檔案系統監控器範例中簡要展示了這一點，但讓我們更詳細地探討一下。你可以使用幾種不同的變體。這展示了最簡潔的形式：

```csharp
FileSystemWatcher watcher = new (@"c:\incoming");
IObservable<EventPattern<FileSystemEventArgs>> changeEvents = Observable
    .FromEventPattern<FileSystemEventArgs>(watcher, nameof(watcher.Changed));
```

如果你有一個提供事件的物件，你可以使用 `FromEventPattern` 的這個多載，傳入該物件以及你想與 Rx 一起使用的事件名稱。雖然這是將事件配接到 Rx 世界的最簡單方法，但它存在一些問題。

首先，為什麼我需要將事件名稱傳遞為字串？用字串識別成員是一種容易出錯的技術。如果第一個引數和第二個引數之間不符，編譯器不會注意到（例如，如果我錯誤地傳遞了引數 `(somethingElse, nameof(watcher.Changed))`）。我難道不能直接傳遞 `watcher.Changed` 本身嗎？不幸的是不行——這是我在第一章中提到的問題的一個範例：.NET 事件不是一等公民。我們無法像使用其他物件或值那樣使用它們。例如，我們不能將事件作為引數傳遞給方法。事實上，你唯一能對 .NET 事件做的事情就是附加和移除事件處理常式。如果我想讓某個其他方法將處理常式附加到我選擇的事件（例如，這裡我希望 Rx 處理事件），那麼唯一的方法就是指定事件的名稱，以便該方法（`FromEventPattern`）隨後可以使用反射來附加它自己的處理常式。

對於某些部署情境，這是一個問題。在 .NET 中，在建構時執行額外工作以最佳化執行階段行為變得越來越普遍，而對反射的依賴可能會損害這些技術。例如，我們可能使用預先編譯（AOT）機制，而不是依賴程式碼的即時（JIT）編譯。.NET 的 Ready to Run (R2R) 系統使你能夠在一般 IL 旁包含針對特定 CPU 型別的預先編譯程式碼，從而避免必須等待 .NET 將 IL 編譯為可執行程式碼。這對啟動時間有顯著影響。在用戶端應用程式中，它可以解決應用程式首次啟動時遲鈍的問題。它在伺服器端應用程式中也很重要，尤其是在程式碼可能相當頻繁地從一個運算節點移到另一個運算節點的環境中，使得最小化冷啟動成本變得很重要。在某些情境下甚至無法選擇 JIT 編譯，在這種情況下 AOT 編譯不僅僅是一種最佳化：它是程式碼能夠執行的唯一手段。

反射的問題在於它使建構工具難以弄清楚執行階段將執行什麼程式碼。當它們檢查對 `FromEventPattern` 的呼叫時，它們只會看到型別為 `object` 和 `string` 的引數。這在執行階段將導致對 `FileSystemWatcher.Changed` 的 `add` 和 `remove` 方法進行反射驅動呼叫並不是顯而易見的。有一些屬性（attributes）可用於提供提示，但這些提示的效果有限。有時建構工具將無法確定需要 AOT 編譯哪些程式碼才能使該方法在不依賴執行階段 JIT 的情況下執行。

還有另一個相關的問題。.NET 建構工具支援一項稱為「剪裁」（trimming）的功能，在該功能中它們會移除未使用的程式碼。`System.Reactive.dll` 檔案大小約為 1.3MB，但在該元件中使用每個型別的每個成員將是非常不尋常的應用程式。Rx 的基本使用可能只需要幾十 KB。剪裁的想法是找出實際使用的部分，並產生僅包含該程式碼的 DLL 複本。這可以顯著減少執行檔執行時需要部署的程式碼量。這在用戶端 Blazor 應用程式中尤其重要，因為 .NET 元件最終會由瀏覽器下載。必須下載整個 1.3MB 的元件可能會讓你對使用它三思而後行。但如果剪裁意味著基本使用只需要幾十 KB，並且只有在你更廣泛地使用該元件時大小才會增加，這可以使得使用該元件變得合理，否則如果沒有剪裁，該元件會帶來過大的代價而無法證明引入它的合理性。但與 AOT 編譯一樣，剪裁只有在工具能夠確定哪些程式碼正在使用時才能運作。如果它們做不到這一點，那就不僅僅是退回較慢的路徑、等待相關程式碼進行 JIT 編譯的問題。如果程式碼已被剪裁，它在執行階段將不可用，並且你的應用程式可能會因 `MissingMethodException` 而崩潰。

因此，如果你使用這些技術中的任何一種，基於反射的 API 可能會產生問題。幸運的是，有一種替代方案。我們可以使用接受一對委派的多載，Rx 將在想要為事件新增或移除處理常式時呼叫這些委派：

```csharp
IObservable<EventPattern<FileSystemEventArgs>> changeEvents = Observable
    .FromEventPattern<FileSystemEventHandler, FileSystemEventArgs>(
        h => watcher.Changed += h,
        h => watcher.Changed -= h);
```

這是 AOT 和剪裁工具可以輕鬆理解的程式碼。我們編寫了明確為 `FileSystemWatcher.Changed` 事件新增和移除處理常式的方法，因此 AOT 工具可以預先編譯這兩個方法，而剪裁工具知道它們不能移除這些事件的新增和移除處理常式。

缺點是這是一段相當繁瑣的程式碼。如果你還沒有接受使用 Rx 的想法，這很可能足以讓你想「謝謝，我還是用一般的 .NET 事件就好。」但是繁瑣的本質是 .NET 事件缺陷的一種表徵。如果事件從一開始就是一等公民，我們就不必編寫如此醜陋的程式碼。

二等公民的地位不僅意味著我們不能僅將事件本身作為引數傳遞，還意味著我們必須明確聲明型別引數。一般來說，事件的委派型別（在此範例中為 `FileSystemEventHandler`）與其事件引數型別（此處為 `FileSystemEventArgs`）之間的關聯不是 C# 的型別推斷可以自動確定的，這就是為什麼我們必須明確指定這兩種型別的原因。（使用泛型 `EventHandler<T>` 型別的事件更容易進行型別推斷，並且可以使用稍微簡短一點的 `FromEventPattern` 版本。不幸的是，相對較少的事件實際使用它。有些事件除了某事剛剛發生之外不提供任何資訊，並使用基礎 `EventHandler` 型別，對於這種類型的事件，你實際上可以完全省略型別引數，使程式碼稍微不那麼醜陋。不過你仍然需要提供新增和移除回呼。）

請注意，此範例中 `FromEventPattern` 的返回型別是：

`IObservable<EventPattern<FileSystemEventArgs>>`。

`EventPattern<T>` 型別封裝了事件傳遞給處理常式的資訊。大多數 .NET 事件遵循通用模式，其中處理常式方法接受兩個引數：一個 `object sender`，它僅告訴你哪個物件引發了事件（如果你將一個事件處理常式附加到多個物件，這很有用），然後是衍生自 `EventArgs` 的某個型別的第二個引數，該引數提供有關事件的資訊。`EventPattern<T>` 只是將這兩個引數打包成提供 `Sender` 和 `EventArgs` 屬性的單一物件。在實際上不需要將一個處理常式附加到多個來源的情況下，你實際上只需要該 `EventArgs` 屬性，這就是為什麼前面的 `FileSystemWatcher` 範例繼續僅擷取該屬性，以獲得 `IObservable<FileSystemEventArgs>` 型別的更簡單結果。它是透過 `Select` 運算子完成的，我們稍後將更詳細地介紹它：

```csharp
IObservable<FileSystemEventArgs> changes = changeEvents.Select(ep => ep.EventArgs);
```

將屬性變更事件公開為可觀察序列是非常普遍的需求。.NET 執行階段函式庫定義了基於 .NET 事件的介面 `INotifyPropertyChanged` 以通告屬性變更，並且一些使用者介面框架為此提供了更專門的系統，例如 WPF 的 `DependencyProperty`。如果你正在考慮編寫自己的包裝器來執行此類操作，我強烈建議先查看 [Reactive UI 函式庫](https://github.com/reactiveui/ReactiveUI/)。它有一組[用於將屬性包裝為 `IObservable<T>` 的功能](https://www.reactiveui.net/docs/handbook/when-any/)。

### 來自 Task

`Task` 和 `Task<T>` 型別在 .NET 中被廣泛使用。主流 .NET 語言具有使用它們的內建支援（例如 C# 的 `async` 和 `await` 關鍵字）。Task 和 `IObservable<T>` 之間存在一些概念上的重疊：兩者都表示可能需要一段時間才能完成的某種工作。在某種意義上，`IObservable<T>` 是 `Task<T>` 的泛化：兩者都表示可能長時間執行的工作，但 `IObservable<T>` 可以產生多個結果，而 `Task<T>` 只能產生一個結果。

由於 `IObservable<T>` 是更通用的抽象，我們應該能夠將 `Task<T>` 表示為 `IObservable<T>`。Rx 為 `Task` 和 `Task<T>` 定義了各種擴充方法來執行此操作。這些方法都稱為 `ToObservable()`，它提供了各種多載，在需要時提供對詳細資訊的控制，並為最常見的情境提供簡潔性。

儘管它們在概念上相似，但 `Task<T>` 在細節上有一些不同之處。例如，你可以擷取其 [`Status` 屬性](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.status)，該屬性可能會回報它處於已取消或錯誤狀態。`IObservable<T>` 不提供向來源詢問其狀態的方法；它只是告訴你事情。因此，`ToObservable` 在如何以在 Rx 世界中有意義的方式呈現狀態方面做出了一些決策：

- 如果工作為 [Cancelled](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskstatus#system-threading-tasks-taskstatus-canceled)，`IObservable<T>` 會呼叫訂閱者的 `OnError` 並傳入 `TaskCanceledException`
- 如果工作為 [Faulted](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskstatus#system-threading-tasks-taskstatus-faulted)，`IObservable<T>` 會呼叫訂閱者的 `OnError` 並傳入工作的內部例外狀況
- 如果工作尚未處於最終狀態（既非 [Cancelled](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskstatus#system-threading-tasks-taskstatus-canceled)、[Faulted](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskstatus#system-threading-tasks-taskstatus-faulted)，也非 [RanToCompletion](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskstatus#system-threading-tasks-taskstatus-rantocompletion)），則 `IObservable<T>` 在該工作進入這些最終狀態之一前不會產生任何通知

在呼叫 `ToObservable` 的那一刻，工作是否已處於最終狀態並不重要。如果它已完成，`ToObservable` 只會返回表示該狀態的序列。（事實上，它使用了你之前看到的 `Return` 或 `Throw` 建立方法。）如果工作尚未完成，`ToObservable` 將在工作上附加一個延續作業（continuation），以便在工作完成時偵測結果。

工作有兩種形式：產生結果的 `Task<T>` 和不產生結果的 `Task`。但在 Rx 中，只有 `IObservable<T>`——沒有無結果的形式。我們之前已經看過一次這個問題，當時 `Observable.Start` 方法需要能夠[將委派配接為 `IObservable<T>`](#from-delegates)，即使該委派是未產生結果的 `Action`。解決方案是返回 `IObservable<Unit>`，這也是當你在普通 `Task` 上呼叫 `ToObservable` 時所獲得的結果。

該擴充方法使用起來很簡單：

```csharp
Task<string> t = Task.Run(() =>
{
    Console.WriteLine("Task running...");
    return "Test";
});
IObservable<string> source = t.ToObservable();
source.Subscribe(
    Console.WriteLine,
    () => Console.WriteLine("completed"));
source.Subscribe(
    Console.WriteLine,
    () => Console.WriteLine("completed"));
```

這是輸出。

```
Task running...
Test
completed
Test
completed
```

請注意，即使有兩個訂閱者，工作也只執行一次。這並不奇怪，因為我們只建立了一個工作。如果工作尚未完成，則所有訂閱者將在工作完成時收到結果。如果工作已完成，`IObservable<T>` 實際上會成為單值冷可觀察序列。

#### 每次訂閱一個 Task

有一種為來源取得 `IObservable<T>` 的不同方法。我可以將前面範例中的第一條語句替換為：

```csharp
IObservable<string> source = Observable.FromAsync(() => Task.Run(() =>
{
    Console.WriteLine("Task running...");
    return "Test";
}));
```

對此進行兩次訂閱會產生略有不同的輸出：

```
Task running...
Task running...
Test
Test
completed
completed
```

請注意，這會執行工作兩次，每次呼叫 `Subscribe` 執行一次。`FromAsync` 可以做到這一點，因為我們傳遞的不是 `Task<T>`，而是返回 `Task<T>` 的回呼。當我們呼叫 `Subscribe` 時它會呼叫該回呼，因此每個訂閱者本質上都會獲得自己的工作。

如果我想使用 `async` 和 `await` 來定義我的工作，那麼我不需要理會 `Task.Run`，因為 `async` Lambda 運算式會建立一個 `Func<Task<T>>`，這正是 `FromAsync` 所需要的型別：

```csharp
IObservable<string> source = Observable.FromAsync(async () =>
{
    Console.WriteLine("Task running...");
    await Task.Delay(50);
    return "Test";
});
```

這會產生與之前完全相同的輸出。不過這有一個微妙的差異。當我使用 `Task.Run` 時，Lambda 運算式從一開始就在工作集區執行緒上執行。但是當我這樣寫時，Lambda 運算式將開始在呼叫 `Subscribe` 的任何執行緒上執行。只有當它遇到第一個 `await` 時它才會返回（然後對 `Subscribe` 的呼叫將返回），該方法的其餘部分將在執行緒集區上執行。

### 來自 `IEnumerable<T>`

Rx 定義了另一個名為 `ToObservable` 的擴充方法，這次是針對 `IEnumerable<T>`。在前面的章節中，我描述了 `IObservable<T>` 如何被設計為表示與 `IEnumerable<T>` 相同的基本抽象，唯一的區別是我們用來取得序列中元素的機制：對於 `IEnumerable<T>`，我們編寫從集合中_提取_（pull）值的程式碼（例如 `foreach` 迴圈），而 `IObservable<T>` 透過在我們的 `IObserver<T>` 上呼叫 `OnNext` 將值_推送_（push）給我們。

我們可以編寫從_提取_到_推送_的橋接程式碼：

```csharp
// 僅供範例參考的程式碼 - 請勿使用！
public static IObservable<T> ToObservableOversimplified<T>(this IEnumerable<T> source)
{
    return Observable.Create<T>(o =>
    {
        foreach (var item in source)
        {
            o.OnNext(item);
        }

        o.OnCompleted();

        // 錯誤地忽略了取消訂閱。
        return Disposable.Empty;
    });
}
```

這個粗糙的實作傳達了基本概念，但它過於天真。它沒有嘗試處理取消訂閱，而且在此特定情境中使用 `Observable.Create` 時要解決該問題並不容易。正如我們稍後將在本書中看到的，可能嘗試快速連續交付大量事件的 Rx 來源應該與 Rx 的並行模型整合。Rx 提供的實作當然考慮了所有這些棘手的細節。這使得它變得相當複雜，但這是 Rx 的問題；你可以將其視為在邏輯上等同於上面顯示的程式碼，但沒有那些缺點。

事實上，這是貫穿 Rx.NET 的一個反覆出現的主題。許多內建運算子之所以有用，並不是因為它們做了特別複雜的事情，而是因為它們替你處理了許多微妙而棘手的問題。在考慮開發自己的解決方案之前，你應該始終嘗試在 Rx.NET 內建的功能中尋找能滿足你需求的項目。

從 `IEnumerable<T>` 轉換為 `IObservable<T>` 時，你應該仔細考慮你真正想要實現的目標。請考慮到 `IEnumerable<T>` 的阻塞同步（拉取）特性並不總是能與 `IObservable<T>` 的非同步（推送）特性良好融合。一旦有東西訂閱了以這種方式建立的 `IObservable<T>`，它實際上是在要求反覆運算 `IEnumerable<T>`，立即產生所有值。對 `Subscribe` 的呼叫可能直到到達 `IEnumerable<T>` 的末尾才會返回，這使得它類似於[本章開頭](#a-very-basic-iobservablet-implementation)所示的非常簡單的範例。（我說「可能」是因為正如我們在討論排程器時將看到的那樣，確切的行為取決於上下文。）`ToObservable` 無法施展魔法——必須在某處執行相當於 `foreach` 迴圈的操作。

因此，雖然這可能是將資料序列引入 Rx 世界的一種便捷方式，但你應該仔細測試並衡量效能影響。

### 來自 APM

Rx 提供對古老的 [.NET 非同步程式設計模型 (APM)](https://learn.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/asynchronous-programming-model-apm) 的支援。回到 .NET 1.0 時期，這是表示非同步作業的唯一模式。它在 2010 年 .NET 4.0 引入[基於工作（Task）的非同步模式 (TAP)](https://learn.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap) 時被取代。舊的 APM 與 TAP 相比沒有任何優勢。此外，C# 的 `async` 和 `await` 關鍵字（以及其他 .NET 語言中的等價物）僅支援 TAP，這意味著最好避免使用 APM。然而，在 2011 年 Rx 1.0 發布時，TAP 相當新，因此它提供了將 APM 實作呈現為 `IObservable<T>` 的配接器。

現今沒有人應該使用 APM，但為了完整性（以及以防你必須使用僅提供 APM 的古老函式庫），我將對 Rx 對它的支援提供非常簡要的說明。

對 `Observable.FromAsyncPattern` 的呼叫結果_不會_返回可觀察序列。它返回一個返回可觀察序列的委派。（因此它本質上是一個工廠的工廠。）此委派的簽名將與對 `FromAsyncPattern` 呼叫的泛型引數相符，不同之處在於返回型別將包裝在可觀察序列中。以下範例包裝了 `Stream` 類別的 `BeginRead`/`EndRead` 方法（它們是 APM 的實作）。

**注意**：這純粹是為了說明如何包裝 APM。你在實務上絕不會這樣做，因為 `Stream` 多年來一直支援 TAP。

```csharp
Stream stream = GetStreamFromSomewhere();
var fileLength = (int) stream.Length;

Func<byte[], int, int, IObservable<int>> read = 
            Observable.FromAsyncPattern<byte[], int, int, int>(
              stream.BeginRead, 
              stream.EndRead);
var buffer = new byte[fileLength];
IObservable<int> bytesReadStream = read(buffer, 0, fileLength);
bytesReadStream.Subscribe(byteCount =>
{
    Console.WriteLine(
        "Number of bytes read={0}, buffer should be populated with data now.", 
        byteCount);
});
```

## 主體（Subjects）

到目前為止，本章探討了返回 `IObservable<T>` 實作的各種工廠方法。不過還有另一種方式：`System.Reactive` 定義了實作 `IObservable<T>` 的各種型別，我們可以將其實例化。但是我們如何確定這些型別產生什麼值？我們能夠做到這一點是因為它們也實作了 `IObserver<T>`，使我們能夠將值推入其中，而我們推入的這些相同值將會是被觀察者看到的值。

在 Rx 中，同時實作 `IObservable<T>` 和 `IObserver<T>` 的型別稱為_主體_（subjects）。有一個 `ISubject<T>` 來表示這一點。（這位於 `System.Reactive` NuGet 套件中，不像 `IObservable<T>` 和 `IObserver<T>` 兩者都內建於 .NET 執行階段函式庫中。）`ISubject<T>` 看起來像這樣：

```csharp
public interface ISubject<T> : ISubject<T, T>
{
}
```

事實證明，還有一個帶有兩個引數的 `ISubject<TSource, TResult>`，以適應同時作為觀察者和可觀察物件的事物可能會以某種方式轉換流經它的資料，這意味著輸入和輸出型別不一定相同。以下是雙型別引數的定義：

```csharp
public interface ISubject<in TSource, out TResult> : IObserver<TSource>, IObservable<TResult>
{
}
```

如你所見，`ISubject` 介面沒有定義自己的任何成員。它們只是繼承自 `IObserver<T>` 和 `IObservable<T>`——這些介面只不過是主體既是觀察者又是可觀察物件這一事實的直接表達。

但這是為了什麼？你可以分別將 `IObserver<T>` 和 `IObservable<T>` 視為「取用者」（consumer）和「發布者」（publisher）介面。那麼，主體既是取用者又是發布者。資料既流入主體又流出主體。

Rx 提供了一些主體實作，這些實作偶爾在希望提供 `IObservable<T>` 的程式碼中很有用。儘管 `Observable.Create` 通常是執行此操作的首選方式，但有一種重要情況下主體可能更有意義：如果你有一些發現感興趣事件的程式碼（例如透過使用某些傳訊技術的用戶端 API）並希望透過 `IObservable<T>` 提供它們，主體有時可以提供比 `Observable.Create` 或自訂實作更方便的方式來執行此操作。

Rx 提供了幾種主體型別。我們將從最容易理解的一種開始。

### `Subject<T>`

`Subject<T>` 型別會立即將對其 `IObserver<T>` 方法進行的任何呼叫轉發給當前訂閱它的所有觀察者。此範例展示了其基本操作：

```csharp
Subject<int> s = new();
s.Subscribe(x => Console.WriteLine($"Sub1: {x}"));
s.Subscribe(x => Console.WriteLine($"Sub2: {x}"));

s.OnNext(1);
s.OnNext(2);
s.OnNext(3);
```

我建立了一個 `Subject<int>`。我對它進行了兩次訂閱，然後反覆呼叫其 `OnNext` 方法。這會產生以下輸出，說明 `Subject<int>` 將每個 `OnNext` 呼叫轉發給這兩個訂閱者：

```
Sub1: 1
Sub2: 1
Sub1: 2
Sub2: 2
Sub1: 3
Sub2: 3
```

我們可以將其用作在我們接收資料的某些 API 與 Rx 世界之間建立橋樑的一種方式。你可以想像編寫類似以下內容的程式碼：

```csharp
public class MessageQueueToRx : IDisposable
{
    private readonly Subject<string> messages = new();

    public IObservable<string> Messages => messages;

    public void Run()
    {
        while (true)
        {
            // 從某個假設的訊息佇列服務接收訊息
            string message = MqLibrary.ReceiveMessage();
            messages.OnNext(message);
        }
    }

    public void Dispose()
    {
        message.Dispose();
    }
}
```

將其修改為改用 `Observable.Create` 並不會太難。但是，如果你需要提供多個不同的 `IObservable<T>` 來源，這種方法就會變得更容易。想像一下，我們根據內容區分不同的訊息類型，並透過不同的可觀察物件發布它們。如果我們仍然想要一個從佇列中提取訊息的單一迴圈，那麼使用 `Observable.Create` 很難安排。

`Subject<T>` 還會將對 `OnCompleted` 或 `OnError` 的呼叫分發給所有訂閱者。當然，Rx 的規則要求，一旦你在 `IObserver<T>` 上呼叫了這兩種方法中的任何一種（並且任何 `ISubject<T>` 都是 `IObserver<T>`，因此該規則適用於 `Subject<T>`），你絕不能再次在該觀察者上呼叫 `OnNext`、`OnError` 或 `OnComplete`。事實上，`Subject<T>` 將容忍違反此規則的呼叫——它只是忽略它們，因此即使你的程式碼在內部沒有完全遵循這些規則，你呈現給外部世界的 `IObservable<T>` 也會正確運作，因為 Rx 會強制執行這一點。

`Subject<T>` 實作了 `IDisposable`。處置 `Subject<T>` 會將其置於一種狀態，如果你呼叫其任何方法，它將擲回例外狀況。說明文件還將其描述為取消訂閱所有觀察者，但由於已處置的 `Subject<T>` 無論如何都無法產生任何進一步的通知，因此這並沒有太大意義。（請注意，當你 `Dispose` 它時，它_不會_在其觀察者上呼叫 `OnCompleted`。）一個實際的效果是，它追蹤觀察者的內部欄位會重設為指示其已被處置的特殊 sentinel 值，這意味著「取消訂閱」觀察者的一個外部可觀察到的效果是，如果出於某種原因，你的程式碼在處置 `Subject<T>` 之後保留了對它的參考，那將不再使所有訂閱者保持可由 GC 到達。如果 `Subject<T>` 在不再使用後無限期保持可到達狀態，這本身實際上就是記憶體流失，但處置至少會限制其影響：只有 `Subject<T>` 本身保持可到達狀態，而不是其所有訂閱者。

`Subject<T>` 是最直接的主體，但還有其他更專門的主體。

## `ReplaySubject<T>`

`Subject<T>` 不會記住任何內容：它會立即將傳入的值分發給訂閱者。如果有新的訂閱者加入，他們將只會看到在他們訂閱之後發生的事件。另一方面，`ReplaySubject<T>` 可以記住它所見過的每個值。如果新的主體（訂閱者）加入，它將收到迄今為止的完整事件歷程記錄。

這是前面的 [`Subject<T>` 章節](#subjectt)中第一個範例的變體。它建立了一個 `ReplaySubject<int>` 而不是 `Subject<int>`。它不是立即訂閱兩次，而是建立一個初始訂閱，然後僅在發出幾個值之後才建立第二個訂閱。

```csharp
ReplaySubject<int> s = new();
s.Subscribe(x => Console.WriteLine($"Sub1: {x}"));

s.OnNext(1);
s.OnNext(2);

s.Subscribe(x => Console.WriteLine($"Sub2: {x}"));

s.OnNext(3);
```

這會產生以下輸出：

```
Sub1: 1
Sub1: 2
Sub2: 1
Sub2: 2
Sub1: 3
Sub2: 3
```

如你所預期的，我們最初只看到來自 `Sub1` 的輸出。但是當我們進行第二次訂閱呼叫時，我們可以看到 `Sub2` 也收到了前兩個值。然後當我們回報第三個值時，兩者都看到了它。如果這個範例改用 `Subject<int>`，我們將只看到以下輸出：

```
Sub1: 1
Sub1: 2
Sub1: 3
Sub2: 3
```

這裡有一個明顯的潛在問題：如果 `ReplaySubject<T>` 記住發布給它的每個值，我們絕不能將其與無止境的事件來源一起使用，因為它最終會導致我們耗盡記憶體。

`ReplaySubject<T>` 提供了接受簡單快取到期設定的建構函式，可以限制記憶體消耗。一種選擇是指定要記住的最大項目數。下一個範例建立了一個緩衝區大小為 2 的 `ReplaySubject<T>`：

```csharp
ReplaySubject<int> s = new(2);
s.Subscribe(x => Console.WriteLine($"Sub1: {x}"));

s.OnNext(1);
s.OnNext(2);
s.OnNext(3);

s.Subscribe(x => Console.WriteLine($"Sub2: {x}"));

s.OnNext(4);
```

由於第二個訂閱僅在我們已經產生 3 個值之後才加入，因此它不再看到所有值。它只接收訂閱之前發布的最後兩個值（但第一個訂閱當然會繼續看到所有內容）：

```
Sub1: 1
Sub1: 2
Sub1: 3
Sub2: 2
Sub2: 3
Sub1: 4
Sub2: 4
```

或者，你可以透過將 `TimeSpan` 傳遞給 `ReplaySubject<T>` 建構函式來指定基於時間的限制。

請注意，如果來源回報錯誤（透過呼叫 `OnError`），`ReplaySubject<T>` 將保留該 `Exception`，並將其提供給所有目前的訂閱者以及任何後續的訂閱者。這不應該令人意外——該主體的工作就是重播來源所做的事情——但請注意，如果你使用[離開 Rx 世界](13_LeavingIObservable.md)章節中描述的任何機制，這可能會導致問題。例如，如果你 `await` 一個使用 `ReplaySubject<T>` 的可觀察物件，並且底層來源回報了錯誤，你將不再符合[例外狀況狀態](13_LeavingIObservable.md#exception-state)中描述的規則。如果你需要使用 `await`（或類似的會將 `OnError` 呼叫轉換為重新擲回的機制），你可能需要使用 [`ResetExceptionDispatchState`](13_LeavingIObservable.md#resetexceptiondispatchstate) 運算子來確保每次重新擲回例外狀況時都能獲得適當重設的例外狀況狀態。


## `BehaviorSubject<T>`

與 `ReplaySubject<T>` 一樣，`BehaviorSubject<T>` 也有記憶，但它恰好記住一個值。然而，它與緩衝區大小為 1 的 `ReplaySubject<T>` 並不完全相同。`ReplaySubject<T>` 最初處於其記憶體中沒有任何內容的狀態，而 `BehaviorSubject<T>` 始終_恰好_記住一個項目。在我們第一次呼叫 `OnNext` 之前這怎麼可能運作？`BehaviorSubject<T>` 透過要求我們在建構它時提供初始值來強制執行這一點。

因此，你可以將 `BehaviorSubject<T>` 視為_始終_有可用值的主體。如果你訂閱 `BehaviorSubject<T>`，它將立即產生單一值。（它隨後可能會繼續產生更多值，但它總是立即產生一個值。）碰巧的是，它還透過名為 `Value` 的屬性提供該值，因此你不需要向其訂閱 `IObserver<T>` 來擷取該值。

`BehaviorSubject<T>` 可以被視為可觀察屬性。就像一般的屬性一樣，每當你請求時它都可以立即提供一個值。不同之處在於，它隨後可以在每次其值變更時繼續通知你。如果你使用的是 [ReactiveUI 框架](https://www.reactiveui.net/)（一個用於建構使用者介面的基於 Rx 的框架），`BehaviourSubject<T>` 作為檢視模型（在底層領域模型和使用者介面之間進行協調的型別）中屬性的實作型別是有意義的。它具有類似屬性的行為，使你能夠隨時擷取值，但它也提供變更通知，ReactiveUI 可以處理這些通知以使 UI 保持最新。

當涉及到完成時，這個類比略有不足。如果你呼叫 `OnCompleted`，它會立即在其所有觀察者上呼叫 `OnCompleted`，如果有任何新的觀察者訂閱，它們也會立即完成——它不會先提供最後一個值。（因此這是它與緩衝區大小為 1 的 `ReplaySubject<T>` 不同的另一種方式。）

同樣地，如果你呼叫 `OnError`，所有目前的觀察者都將收到 `OnError` 呼叫，並且任何後續訂閱者也只會收到 `OnError` 呼叫。

## `AsyncSubject<T>`

`AsyncSubject<T>` 為所有觀察者提供它接收到的最終值。由於在呼叫 `OnCompleted` 之前它無法知道哪個是最終值，因此在其 `OnCompleted` 或 `OnError` 方法被呼叫之前，它不會在其任何訂閱者上呼叫任何方法。（如果呼叫了 `OnError`，它只會將其轉發給所有目前和未來的訂閱者。）你經常會間接使用此主體，因為它是 [Rx 與 `await` 關鍵字整合](13_LeavingIObservable.md#integration-with-async-and-await)的基礎。（當你 `await` 一個可觀察序列時，`await` 會返回來源發出的最終值。）

如果在 `OnCompleted` 之前沒有對 `OnNext` 進行任何呼叫，則沒有最終值，因此它只會完成任何觀察者而不提供值。

在此範例中，由於序列永不完成，因此不會發布任何值。
不會有任何值寫入主控台。

```csharp
AsyncSubject<string> subject = new();
subject.OnNext("a");
subject.Subscribe(x => Console.WriteLine($"Sub1: {x}"));
subject.OnNext("b");
subject.OnNext("c");
```

在此範例中，我們呼叫了 `OnCompleted` 方法，因此主體將產生一個最終值（'c'）：

```csharp
AsyncSubject<string> subject = new();

subject.OnNext("a");
subject.Subscribe(x => Console.WriteLine($"Sub1: {x}"));
subject.OnNext("b");
subject.OnNext("c");
subject.OnCompleted();
subject.Subscribe(x => Console.WriteLine($"Sub2: {x}"));
```

這會產生以下輸出：

```
Sub1: c
Sub2: c
```

如果在應用程式啟動時需要執行一些潛在緩慢的工作，並且該工作只需要執行一次，你可以選擇 `AsyncSubject<T>` 來提供該工作的結果。需要這些結果的程式碼可以訂閱該主體。如果工作尚未完成，他們將在結果可用時立即收到結果。如果工作已經完成，他們將立即收到它。

## 主體工廠

最後值得讓你了解的是，你也可以透過工廠方法建立主體。考慮到主體結合了 `IObservable<T>` 和 `IObserver<T>` 介面，有一個允許你自行組合它們的工廠似乎是合理的。`Subject.Create(IObserver<TSource>, IObservable<TResult>)` 工廠方法剛好提供了這一點。

```csharp
// 根據指定的觀察者（用於向主體發布訊息）
// 和可觀察物件（用於訂閱從主體發送的訊息）建立主體
public static ISubject<TSource, TResult> Create<TSource, TResult>(
    IObserver<TSource> observer, 
    IObservable<TResult> observable)
{...}
```

請注意，與剛討論的所有其他主體不同，這建立了一個在輸入和輸出之間沒有固有關係的主體。這只是接受你提供的任何 `IObserver<TSource>` 和 `IObserver<TResult>` 實作，並將它們包裝在單一物件中。對主體的 `IObserver<TSource>` 方法進行的所有呼叫將直接傳遞給你提供的觀察者。如果你希望值出現在相應 `IObservable<TResult>` 的訂閱者中，則由你來實現這一點。這實際上是使用最少的黏合劑將你提供的兩個物件結合在一起。

主體提供了一種探索 Rx 的便捷方式，偶爾在生產情境中很有用，但在大多數情況下不建議使用它們。說明請參閱[使用方針附錄](C_UsageGuidelines.md)。不要使用主體，而應優先使用本章前面顯示的工廠方法。

## 摘要

我們介紹了建立序列的各種積極（eager）和延遲（lazy）方式。我們了解了如何使用各種工廠方法產生基於計時器的序列。我們還探索了從其他同步和非同步表示形式進行轉換的方法。

快速回顧：

- 工廠方法
  - Observable.Return
  - Observable.Empty
  - Observable.Never
  - Observable.Throw
  - Observable.Create
  - Observable.Defer

- 產生性方法
  - Observable.Range
  - Observable.Generate
  - Observable.Interval
  - Observable.Timer

- 配接
  - Observable.Start
  - Observable.FromEventPattern
  - Task.ToObservable
  - Task&lt;T&gt;.ToObservable
  - IEnumerable&lt;T&gt;.ToObservable
  - Observable.FromAsyncPattern

建立可觀察序列是我們實際應用 Rx 的第一步：建立序列然後將其公開以供取用。既然我們已經牢牢掌握了如何建立可觀察序列，我們就可以更詳細地了解允許我們描述要套用的處理之運算子，以建構更複雜的可觀察序列。
