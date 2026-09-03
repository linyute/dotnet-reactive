# Ix 6.0 發行歷史


## v6.0.1

版本號更新至 v6.0.x 的第一個發行版本。（當時，Rx 與 Ix 嘗試遵循使版本號與 .NET 執行階段函式庫保持一致的政策。）

新增 `MinByWithTies` 與 `MaxByWithTies` 以恢復在 v5.1 中遺失的功能。（當 .NET 6.0 新增自己的 MinBy/MaxBy 時，Ix v5.1 移除了其方法，但其中一些方法能執行 .NET 6.0 版本無法做到的事。）

