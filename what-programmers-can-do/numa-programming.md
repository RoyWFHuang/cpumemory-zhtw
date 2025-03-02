# 6.5. NUMA 程式設計

以 NUMA 程式設計而言，目前為止說過的有關快取最佳化的所有東西也都適用。差異僅在這個層級以下才開始。NUMA 在存取定址空間的不同部分時引入了不同的成本。使用均勻記憶體存取的話，我們能夠最佳化以最小化分頁錯誤（見 7.5 節），但這是對它而言。所有建立的分頁都是均等的。

NUMA 改變了這點。存取成本可能取決於被存取的分頁。存取成本的差異也增加了針對記憶體分頁的局部性進行最佳化的重要性。NUMA 對於大多 SMP 機器而言都是無可避免的，因為有著 CSI 的 Intel（for x86, x86-64, and IA-64）與 AMD（for Opteron）都會使用它。隨著每個處理器的核心數量增加，我們很可能會看到被使用的 SMP 系統急遽減少（至少除了資料中心與有著非常高 CPU 使用率需求的人們的辦公室之外）。大多家用機器僅有一個處理器就很好了，因此沒有 NUMA 的問題。但這 a) 不代表程式開發者能夠忽略 NUMA，以及 b) 不代表沒有相關的問題。

假如理解 NUMA 的一般化的話，也能快速意識到拓展至處理器快取的概念。在使用相同快取的核心上的兩條執行緒，會合作得比不共享快取的核心上的執行緒還快。這不是個杜撰的狀況：

* 早期的雙核處理器沒有 L2 共享。
* 舉例來說，Intel 的 Core 2 QX 6700 與 QX 6800 四核晶片擁有兩個獨立的 L2 快取。
* 正如早先猜測的，由於一片晶片上的更多核心、以及統一快取的渴望，我們將會有更多層的快取。

快取形成它們自己的階層結構；執行緒在核心上的擺放，對於許多快取的共享（或者沒有）來說變得很重要。這與 NUMA 面對的問題並沒有很大的不同，因此能夠統一這兩個概念。即使是只對非 SMP 機器感興趣的人也該讀一讀本節。

在 5.3 節中，我們已經看到 Linux 系統核心提供了許多在 NUMA 程式設計中有用––而且需要––的資訊。不過，收集這些資訊並沒有這麼簡單。以這個目的而言，當前在 Linux 上可用的 NUMA 函式庫是完全不足的。一個更為合適的版本正由本作者建造中。

現有的 NUMA 函式庫，`libnuma`––numactl 套件（package）的一部分––並不提供對系統架構資訊的存取。它僅是一個對可用系統呼叫的包裝（wrapper）、與針對常用操作的一些方便的介面。現今在 Linux 上可用的系統呼叫為：

<dl>
    <dt><code>mbind</code></dt>
    <dd>選擇指定記憶體分頁的連結（binding）。</dd>

    <dt><code>set_mempolicy</code></dt>
    <dd>設定預設的記憶體連結策略。</dd>

    <dt><code>get_mempolicy</code></dt>
    <dd>取得預設的記憶體連結策略。</dd>

    <dt><code>migrate_pages</code></dt>
    <dd>將一組給定節點上的一個行程的所有分頁遷移到一組不同的節點上。</dd>

    <dt><code>move_pages</code></dt>
    <dd>將選擇的分頁移到給定的節點、或是請求關於分頁的節點資訊。</dd>
</dl>

這些介面被宣告在與 `libnuma` 函式庫一起出現的 `<numaif.h>` 標頭檔中。在我們深入更多細節之前，我們必須理解記憶體策略的概念。

