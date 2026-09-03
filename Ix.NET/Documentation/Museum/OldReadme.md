# 2023 年之前的 README

到 2023 年初，主要 README 已變成資訊的堆放處，其中大部分內容對於首次造訪存放庫以了解其內容的人來說，詳細程度並不合適。

然而，其中的大部分資訊對某些人來說仍有些用處，因此我們將較為冷門的部分移入此檔案，以免它們完全消失。

## 簡短簡介

（**注意：** 這些並非不實，但它著重於細節，而沒有真正說明 Rx 究竟是用來做什麼的。）

Reactive Extensions (Rx) 是一個函式庫，用於使用可觀察序列與 LINQ 樣式的查詢運算子來組合非同步與事件驅動程式。使用 Rx，開發人員可以用 [Observables](https://docs.microsoft.com/dotnet/api/system.iobservable-1) *__表示__* 非同步資料流，使用 [LINQ 運算子](http://msdn.microsoft.com/en-us/library/hh242983.aspx) *__查詢__* 非同步資料流，並使用 [Schedulers](http://msdn.microsoft.com/en-us/library/hh242963.aspx) 將非同步資料流中的並行性 *__參數化__*。簡而言之，Rx = Observables + LINQ + Schedulers。

無論您是在編寫傳統桌面還是以 Web 為基礎的應用程式，您都必須不時處理非同步與事件驅動程式設計。桌面應用程式具有 I/O 作業和計算繁重的任務，這些任務可能需要很長時間才能完成，並可能封鎖其他作用中執行緒。此外，處理例外狀況、取消和同步是困難且容易出錯的。

使用 Rx，您可以表示多個非同步資料流（來自不同來源，例如：股票報價、推文、電腦事件、Web 服務請求等），並使用 `IObserver<T>` 介面訂閱事件流。每當事件發生時，`IObservable<T>` 介面都會通知已訂閱的 `IObserver<T>` 介面。

由於可觀察序列是資料流，您可以使用由 Observable 擴充方法實作的標準 LINQ 查詢運算子來查詢它們。因此，您可以透過使用這些標準 LINQ 運算子，輕鬆對多個事件進行篩選、投影、彙總、組合以及執行以時間為基準的作業。此外，還有許多其他反應式串流專屬的運算子，可編寫功能強大的查詢。使用 Rx 提供的擴充方法也能優雅地處理取消、例外狀況和同步。

Rx 與同步資料流 (`IEnumerable<T>`) 及單一值非同步計算 (`Task<T>`) 均能互補並順暢地互通，如下表所示：


<table>
   <th></th><th>單一傳回值</th><th>多個傳回值</th>
   <tr>
      <td>Pull/Synchronous/Interactive</td>
      <td>T</td>
      <td>IEnumerable&lt;T&gt;</td>
   </tr>
   <tr>
      <td>Push/Asynchronous/Reactive</td>
      <td>Task&lt;T&gt;</td>
      <td>IObservable&lt;T&gt;</td>
   </tr>
</table>

更多文件、影片、教學課程和 HOL 可在 [MSDN](https://docs.microsoft.com/en-us/previous-versions/dotnet/reactive-extensions/hh242985(v=vs.103))、[*Introduction to Rx*](http://introtorx.com/)、[*ReactiveX*](http://reactivex.io/) 以及 [ReactiveUI](https://reactiveui.net/) 上取得。


## Rx 的多種變體

* __Rx.NET__: *(本存放庫)* Reactive Extensions (Rx) 是一個函式庫，用於使用可觀察序列與 LINQ 樣式的查詢運算子來組合非同步與事件驅動程式。
* [RxJS](https://github.com/ReactiveX/rxjs): Reactive Extensions for JavaScript (RxJS) 是一個函式庫，用於在 JavaScript 中使用可觀察序列和 LINQ 樣式的查詢運算子來組合非同步與事件驅動程式，可同時以瀏覽器和 Node.js 為目標。
* [RxJava](https://github.com/ReactiveX/RxJava): Reactive Extensions for the JVM – 適用於 Java VM 的函式庫，用於使用可觀察序列來組合非同步與事件驅動程式。
* [RxScala](https://github.com/ReactiveX/RxScala): Reactive Extensions for Scala – 用於使用可觀察序列來組合非同步與事件驅動程式的函式庫
* [RxCpp](https://github.com/Reactive-Extensions/RxCpp): Reactive Extensions for Native (RxCpp) 是一個函式庫，用於在 C 與 C++ 中使用可觀察序列和 LINQ 樣式的查詢運算子來組合非同步與事件驅動程式。
* [RxPy](https://github.com/ReactiveX/RxPY): Reactive Extensions for Python 3 (Rx.Py) 是一組函式庫，用於在 Python 3 中使用可觀察集合與 LINQ 樣式的查詢運算子來組合非同步與事件驅動程式。


## 應用程式

* [Tx](https://github.com/Reactive-Extensions/Tx): 一組程式碼範例，展示如何將 LINQ 用於事件，例如即時常駐查詢以及從追蹤與記錄檔查詢過去歷史紀錄，目標為 ETW、Windows 事件記錄檔和 SQL Server 擴充事件。
* [LINQ2Charts](http://linq2charts.codeplex.com): Rx 繫結的範例。類似於 LINQ to XML 等現有 API，它允許開發人員以簡易方式使用 LINQ 來建立/變更/更新圖表，並避免處理 XML 或其他底層資料結構。我們樂見更多像這樣的 Rx 繫結。
