# 附錄 C：使用方針

這是一份快速方針清單，旨在幫助您撰寫 Rx 查詢。

- 傳回序列的成員絕不應傳回 null。這適用於 `IEnumerable<T>` 和 `IObservable<T>` 序列。請改為傳回空序列。
- 僅當您需要提前取消訂閱時才處置訂閱。
- 始終提供 `OnError` 處理常式。
- 避免使用封鎖運算子，例如 `First`、`FirstOrDefault`、`Last`、`LastOrDefault`、`Single`、`SingleOrDefault` 和 `ForEach`；請改用非封鎖替代方案，例如 `FirstAsync`。
- 避免在 `IObservable<T>` 和 `IEnumerable<T>` 之間來回切換
- 偏好延遲評估而非積極評估。
- 將大型查詢拆分為多個部分。大型查詢的關鍵指標：
    1. 巢狀
    2. 超過 10 行的查詢運算式語法
    3. 使用 `into` 關鍵字
- 妥善命名您的可觀察物件，即避免使用類似 `query`、`q`、`xs`、`ys`、`subject` 等變數名稱。
- 避免產生副作用。如果確實無法避免，請不要將副作用掩埋在設計為以函數式使用的運算子（如 `Select` 或 `Where`）的回呼中。透過使用 `Do` 運算子保持明確。
- 在可能的情況下，偏好使用 `Observable.Create` 而不是 Subject 作為定義新 Rx 來源的方法。
- 避免建立您自己的 `IObservable<T>` 介面實作。使用 `Observable.Create`（或者如果您確實需要，使用 Subject）。
- 避免建立您自己的 `IObserver<T>` 介面實作。偏好改用 `Subscribe` 擴充方法多載。
- 應用程式應定義並行存取模型。
    - 如果您需要排程延遲工作，請使用排程器
    - `SubscribeOn` 和 `ObserveOn` 運算子應始終緊鄰 `Subscribe` 方法之前。（因此不要將其夾在中間，例如 `source.SubscribeOn(s).Where(x => x.Foo)`。）