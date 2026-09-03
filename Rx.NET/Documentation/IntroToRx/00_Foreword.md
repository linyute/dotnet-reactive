# Rx 簡介
作者：Ian Griffiths 與 Lee Campbell
   
---

響應式程式設計（Reactive programming）並不是一個新概念。任何形式的使用者介面開發都必然涉及回應事件的程式碼。[Smalltalk](https://en.wikipedia.org/wiki/Smalltalk)、[Delphi](https://en.wikipedia.org/wiki/Delphi_(software)) 和 .NET 語言等語言普及了響應式或事件驅動的程式設計範式。諸如 [CEP（複合事件處理）](https://en.wikipedia.org/wiki/Complex_event_processing) 和 [CQRS（命令查詢職責分離）](https://en.wikipedia.org/wiki/Command_Query_Responsibility_Segregation) 等架構模式都將事件作為其組成的基本部分。在任何必須處理事件發生的程式中，響應式程式設計都是一個有用的概念。

> 在任何必須處理事件發生的程式中，響應式程式設計都是一個有用的概念。

事件驅動範式允許呼叫程式碼，而無需打破封裝或套用昂貴的輪詢技術。有許多常見的實作方法，包括[觀察者模式](https://en.wikipedia.org/wiki/Observer_pattern)、[語言中直接公開的事件（例如 C#）](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/events/) 或透過委派註冊的其他形式的回呼。Reactive Extensions 使用 LINQ 擴充了回呼隱喻，以實現查詢事件序列和管理並行存取。

十年來，.NET 執行階段函式庫一直包含代表響應式程式設計核心概念的 `IObservable<T>` 和 `IObserver<T>` 介面。.NET 的 Reactive Extensions（Rx.NET）實際上是這些介面的實作函式庫。Rx.NET 最早出現在 2010 年，但從那時起，Rx 函式庫已可用於其他語言，並且這種程式設計方式在 JavaScript 中特別流行。

本書將透過 C# 介紹 Rx。這些概念是通用的，因此其他 .NET 語言（如 VB.NET 和 F#）的使用者將能夠提取這些概念並將其轉換為其特定的語言。

Rx.NET 只是一個函式庫，最初由 Microsoft 建立，但現在是一個完全透過社群努力支援的開源專案。（Rx 目前的首席維護者 [Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/) 也是本書最新修訂版的作者，事實上也是這句話的作者。）

如果您以前從未使用過 Rx，它*將*改變您設計和建構軟體的方式。它為計算中一個基本重要的概念提供了一個經過深思熟慮的抽象：事件序列。這些與清單或陣列一樣重要，但在 Rx 之前，函式庫或語言中很少有直接支援，而且僅有的支援往往是相當臨時的，並且建立在薄弱的理論基礎之上。Rx 改變了這一點。這項 Microsoft 發明被一些傳統上對 Microsoft 特別不友好的開發者社群全心全意接受的程度，證明了其基本設計的品質。

本書旨在教您：

  * 關於 Rx 定義的型別
  * 關於 Rx 提供的擴充方法，以及如何使用它們
  * 如何管理對事件來源的訂閱
  * 如何視覺化資料的「序列」並在撰寫程式碼之前繪製解決方案草圖
  * 如何善用並行存取並避免常見陷阱
  * 如何組合、聚合和轉換串流
  * 如何測試您的 Rx 程式碼
  * 使用 Rx 時的一些常見最佳實踐
    
學習 Rx 的最佳方法是使用它。閱讀本書中的理論只會幫助您熟悉 Rx，但要完全理解它，您應該用它來建構事物。因此我們熱烈鼓勵您根據本書中的範例進行建構。

# 致謝

首先，我（[Ian Griffiths](https://endjin.com/who-we-are/our-people/ian-griffiths/)）應該明確表示，這個修訂版是建立在原作者 [Lee Campbell](https://github.com/LeeCampbell) 的出色工作之上的。我非常感謝他慷慨地允許 Rx.NET 專案使用他的內容，使這個新版本得以問世。

我還要感謝促成這本書的人們。

感謝 [endjin](https://endjin.com) 的每個人，特別是 [Howard van Rooijen](https://endjin.com/who-we-are/our-people/howard-van-rooijen/) 和 [Matthew Adams](https://endjin.com/who-we-are/our-people/matthew-adams/)，
感謝他們不僅資助了本書的更新，還資助了 Rx.NET 本身的持續開發。（也感謝僱用我！）。也感謝 [Felix Corke](https://www.blackspike.com/) 在[本書網路版](https://introtorx.com)的設計元素上所做的工作。

除了作者 [Lee Campbell](https://leecampbell.com/) 之外，對本書第一版至關重要的還有：James Miles、Matt Barrett、[John Marks](https://johnhmarks.wordpress.com/)、Duncan Mole、Cathal Golden、Keith Woods、Ray Booysen、Olivier DeHeurles、[Matt Davey](https://mdavey.wordpress.com)、[Joe Albahari](https://www.albahari.com/) 和 Gregory Andrien。

特別感謝在 Microsoft 努力工作並為我們帶來 Rx 的團隊；[Jeffrey Van Gogh](https://www.linkedin.com/in/jeffrey-van-gogh-145673/)、[Wes Dyer](https://www.linkedin.com/in/wesdyer/)、[Erik Meijer](https://en.wikipedia.org/wiki/Erik_Meijer_%28computer_scientist%29) 和 [Bart De Smet](https://www.linkedin.com/in/bartdesmet/)。

也感謝那些在 Rx.NET 不再由 Microsoft 直接支援並成為基於社群的開源專案後繼續致力於 Rx.NET 的人。涉及許多人，在這裡列出每一位貢獻者是不切實際的，但我特別想對 [Bart De Smet](https://github.com/bartdesmet)（再次感謝，因為他在轉向 Microsoft 內部的其他事務很久之後仍繼續致力於開源 Rx）以及 [Claire Novotny](https://github.com/clairernovotny)、[Daniel Weber](https://github.com/danielcweber)、[David Karnok](https://github.com/akarnokd)、[Brendan Forster](https://github.com/shiftkey)、[Ani Betts](https://github.com/anaisbetts) 和 [Chris Pulman](https://www.linkedin.com/in/chrispulman/) 表示由衷的感謝。我們也感謝 [Richard Lander](https://www.linkedin.com/in/richardlander/) 和 [.NET Foundation](https://dotnetfoundation.org/) 幫助我們 [endjin](https://endjin.com) 成為 [Rx.NET 專案](https://github.com/dotnet/reactive) 的新管理者，使其能夠繼續蓬勃發展。

如果您有興趣了解更多關於 Rx 起源的資訊，您可能會發現 [A Little History of Reaqtor](https://reaqtive.net/) 電子書富有啟發性。

本書撰寫所針對的版本是 `System.Reactive` 6.1 版。本書的原始碼可以在 [https://github.com/dotnet/reactive/tree/main/Rx.NET/Documentation/IntroToRx](https://github.com/dotnet/reactive/tree/main/Rx.NET/Documentation/IntroToRx) 找到。如果您在本書中發現任何錯誤或其他問題，請在 https://github.com/dotnet/reactive/ [建立 issue](https://github.com/dotnet/reactive/issues)。如果您開始認真使用 Rx.NET，您可能會發現 [Reactive X slack](reactivex.slack.com) 是一個有用的資源。

因此，啟動 Visual Studio，讓我們開始吧。

# 版本歷史

## 第 1 版

由 Lee Campbell 撰寫的原版書籍。

## 第 2 版

由 Ian Griffiths 更新和修訂，以配合 Rx v6.0 版本。

## 第 3 版

針對 Rx v6.1 的更新：

* 記錄了新的 `TakeUntil(CancellationToken)` 多載
* 記錄了新的 `IDisposable` 的 `DiposeWith` 擴充方法
* 記錄了新的 `ResetExceptionDispatchState` 運算子
* 新增了指引以闡明在 Rx 將重新擲回例外狀況的情境中重複使用例外狀況物件的規則

---
