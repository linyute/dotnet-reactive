---
name: 'Ix 的 Issue 回報'
about: '建立關於 Ix.NET 的錯誤、問題或功能請求的 Issue 回報'
title: ''
labels: '[area] Ix'
assignees: ''
---
您好，感謝您使用 dotnet/reactive。請選擇一個類別，並透過回答其中的問題來詳細說明您的問題：

#### 錯誤

儘管我們竭盡全力，錯誤仍可能出現在發行版本中，或是極端情況被遺忘。我們將盡最大努力補救這種情況
並/或提供因應措施。請注意，某些（奇特的）行為是依設計運作的，因此不被視為錯誤。

> 哪一個子元件函式庫（Ix, Async.Ix）？

> 哪一個函式庫版本？

> 平台、環境與相關元件版本為何？

> 使用案例或問題為何？

> 預期結果為何？

> 實際結果為何？

> 例外狀況的堆疊追蹤為何（若有的話）？

> 您是否有重現該問題的程式碼片段或專案？

#### 問題

在向我們提出問題之前，請注意 dotnet/reactive 是由少數熱心人士在業餘時間自願維護的。
您可以先在喜愛的搜尋引擎、我們的 [issue 清單](https://github.com/dotnet/reactive/issues) 或相關的 [stackoverflow.com](https://stackoverflow.com) 關鍵字（例如
[ienumerable](https://stackoverflow.com/questions/tagged/ienumerable) 與
[iasyncenumerable](https://stackoverflow.com/questions/tagged/iasyncenumerable)
）中搜尋與您的問題相關的關鍵字，這將給我們極大的幫助。也請考慮改為在 *StackOverflow* 上的這些標籤下提出問題，例如 **"How do I do X?"** 或 **"Where can I find Y?"**。

如果您尚未找到答案，或者您的問題不太適合發布在 *StackOverflow*，歡迎在此提出。

> 您的問題或困難的背景脈絡為何？

> 您嘗試解決的問題或困難為何？

> 您嘗試滿足的（原始）需求為何？

> 到目前為止您嘗試過什麼，寫了哪些程式碼？

#### 功能請求

dotnet/reactive 託管了 `IObservable`、`IEnumerable` 和 `IAsyncEnumerable` 的基礎元件與運算子，因此，為了成為並
保持為可靠的函式庫家族，我們必須謹慎評估要納入哪些新功能。因此，在請求新的元件、
運算子或功能之前，請先考慮以下情況與解決方案：

a) **新的來源 Factory 方法。** 建立上述型別執行個體的靜態 Factory 方法可以存在於任何函式庫的任何類別中，
而不會帶來太多不便。請考慮將此類 Factory 方法託管於 dotnet/reactive 之外。

b) **新的執行個體方法/運算子。** .NET 世界具有擴充方法，讓您能夠在本地專案或任何第三方函式庫中靈活地進行流暢的 API 擴充。
請考慮也將此類方法託管於 dotnet/reactive 之外。

c) **對其他第 1 方或第 3 方元件的支援或橋接。** 這些會依具體情況個別考量，但一般而言，
請考慮將此類支援/橋接程式碼託管於 dotnet/reactive 之外。

d) **新的 reactive/interactive 基礎型別或概念。** 需要變更或引入新通訊協定（例如流量控制、
項目生命週期、非同步）的元件通常更適合由其自己的第三方函式庫託管，並應透過上述標準
型別在那裡提供互操作性。

e) **現有運算子的行為變更。** 此類變更對現有使用者帶來許多風險，因此通常最好引入
一個全新的元件或運算子，對此請再次參閱 a)..d) 點。

考量上述幾點，請描述您希望 dotnet/reactive 納入的功能或行為：

> 哪一個子元件函式庫（Ix, Async.Ix）？

> 下一個函式庫版本為何（即修補程式、次要或主要版本）？

> 平台、環境與相關元件版本為何？

> 此功能的需求程度為何（單一專案、數個專案、全公司、全域）？

> 請描述該功能。
