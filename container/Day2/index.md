[Day2] 淺談 Container 實現原理, 以 Docker 為例(上)
=============================================

# 前言
前一天中，我們探討了 `kubernetes` 的基本架構，只有稍微粗淺的介紹，並沒有仔細深入到各自的標準介面中。

再我們往下探索各類標準前，我們必須要先瞭解如何打造 `Container`，因為這些 `kubernetes` 與標準介面溝通的過程，其實與 `如何打造一個 Container` 息息相關。

為了避免混淆，接下來的探討都會基於下列環境
1. 作業系統採用 `Linux`
2. `Container` 的實作選擇使用 `Docker`.

# Container
隨者 `Docker` 過去數年的蓬勃發展，就算不曾用過 Docker 也必定聽過所謂的 `容器(Container)`，雖然 `Container` 的概念其實不算新，不論是 `Jail` 或是 `LXC` 都有者相同輕量級虛擬化的作用。

但是真正將其落地並且廣泛被接受的我認為還是歸功於 `Docker`， `Docker` 簡化了整體的複雜性，讓整個使用起來更加簡單，透過 `Image`的方式重複使用各式各樣的環境為整個生態性帶來了廣泛的討論。

接下來我們就要探討 `Docker` 與  `Container` 這兩者的關係，並且理解一個最簡單的 `docker run` 其背後到底是怎麼運作的。

由於 `Docker`  本身版本的演進，其底層實作的方式也在演進，因此我們這主要是針對 Docker 1.11 之後的架構來介紹。

# Open Container Initiative (OCI)
`Container` 與 `Docker` 的關係要特別注意，`Container` 是所謂的概念與目標，而 `Docker` 則是一種實現方式，除了 `Docker` 之外， `CRI-O`, `RKT`  都是其他的選擇。


就如同軟體開發一樣，為了滿足各式各樣的實現，則該介面必定有所謂的標準，而各種實現必須符合該介面才算是一種可用的實作。

在 `Container` 的世界中，該介面就是所謂的 `Open Container Iinitiative (OCI)`
- 隸屬於 Linux Foundation 專案底下
- 2015/01/22 專案啟動`
- 目標希望能夠提供基於作業系統層級的虛擬化介面
- 主要定義兩大標準
    - Runtime Specification (運行標準)
    - Image Specification (容器映像檔案標準)

熟悉 `Docker` 的人應該對於 `docker images` 相關指令不陌生，不論是使用各式各樣包好的套件，甚至是自己透過 `docker build` 建置自己的環境都會與所謂的 `image` 脫不了關係，而所謂的 `Image Spec` 就是用來規範 `Image` 的格式。

而 `Runtime Spec` 則是規範如何控制所謂的 `Container`, 包含了 `Create/Delete/Execute..etc` 等各種操作，此外如何產生 `Container` 所需要的資訊也都在標準內有所設定。

`Docker` 底層用到的相關元件都有滿足 `OCI` 的需求，所以如果今天有其他的解決方案也都遵循 `OCI` 的標準來設計，則理論上他們之間的容器映像檔案 `Container Image` 要可以交替使用而不影響功能。

接下來細看一下所謂的 `Runtime Spec` 以及 `Image Spec` 到底規範了什麼資訊與設定。

## Runtime Spec
`Runetime Spec` 的文件可以在  [Github](https://github.com/opencontainers/runtime-spec/blob/master/spec.md) 上參閱， `Runtime Spec` 規範的範疇有三，分別是
1. 設定檔案格式
    - 該內容基本上會根據不同平台而有不同的規範，同時也明確表示創建該平台上 `container` 所需要的一切參數。
3. 執行環境一致
    - 確保 `container` 運行期間能夠有一致的運行環境。
5. `Container`的生命週期
    - 必須支持下列的操作行為，這邊不再贅述，可以到此[閱覽](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md#operations)
        - Query State
        - Create
        - Start
        - Delete
        - Kill
        - Hooks

#### Configuration
講到設定檔案的部分，我們可以稍微看一下到底在 `Linux` 上創建一個 `Container` 會需要哪些[設定](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)。

首先是資源隔離，這部分是仰賴 `Linux Kernel` 內的 `namespace` 來完成的，藉由資源隔離可以達到讓 `namespace` 內部與 `host` 本身互相使用該資源卻不衝突的優點。

根據上述的規範，會需要用到的 `namespace` 如下。
- pid
    - process 相關的隔離。不知道大家是否有觀察到在 `docker container` 的世界內，用 `ps auxw` 能夠看到的 `process` 比想像中的少?
- network
    - 精準來說是 `Network Stack` 的隔離，體現出來簡單的範例就是 `網卡`,`iptables 規則` 等諸多與網路有關的資源。
- ipc
    - `Inter Process Communication` 意味者 `namespace` 內的 `process` 只能跟同 `namespace` 內的其他 `process` 透過 `IPC` 的機制溝通
- mount
    - 常常使用 `mount` 指令的人就可以發現到 `docker container` 內與外面看到的結果不同也是因為該 `namespace` 將資源區隔了
- uts
    - `UNIX Timesharing System` 主要負責的就是 `hostname`, 包含給 `NIS` 使用的 `domain name`
- user
    - 使用者與群組清單，透過 `id` 指令等對應到的 `UID/GID` 都是獨立的
- cgroup
    - `Control Group`，主要是跟資源存取限制有關，譬如 `CPU`，`Memory` 等資源的存取上限。再 `docker container` 的使用中就可以透過該機制設定一些 `soft, hard/limit memory` 等設定來避免 `OOM` 或是減少 `CPU` 存取。

每個 `namespace` 都有自己的設定檔案，因此 `Configuration` 在設定的時候則必須要仔細的設定每個 `namespace` 的資訊。 譬如

```json
        {
            "type": "pid",
            "path": "/proc/1234/ns/pid"
        },
        {
            "type": "network",
            "path": "/var/run/netns/neta"
        },
        {
            "type": "mount"
        },
        {
            "type": "ipc"
        },
        {
            "type": "uts"
        },
        {
            "type": "user"
        },
        {
            "type": "cgroup"
        }
```

除了 `namespace` 之外，`container` 內部也還有許多的資源需要設定，譬如 `Devices`, `Sysctl`, `Seccomp(SECure COMPuting)`，有興趣的請記得參閱[官方文件](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)。

到這邊對於 `Runtime Spec` 已經有一些基本的認知，用來定義不同平台上如何管理 `Container`，為了管理這些 `Container` 必須要有一些相關的設定來描述該 `Container` 的資訊，而這些設定本身並非跨平台，需要針對每個平台去獨立設計來描述如何產生一個符合需求的 `Container`.

## Image Spec
接下來來看一下到底所謂的 `Image` 是什麼，我們常用的 `Docker pull/push/build/images` 所產生的 `image` 檔案是怎麼被定義的。

從[官方GitHub](https://github.com/opencontainers/image-spec/blob/master/spec.md) 參考而來的圖片清楚明瞭的說明到底 `Image` 要處理什麼事情
![](https://i.imgur.com/acGSy8O.png)

一個簡單的 `Java` 應用程式包裝成 `Container` 後，整個 `Image Layer` 處理了下列事情
1. `Layer`: 相關的檔案系統配置，檔案的位置/內容/權限
2. `Image Index`: 用來描述該 `Image` 的檔案
3. `Configuration`: 應用程式相關的設定檔案，包含了使用的參數，用到的環境變數等

如果仔細去研究的話，其實總共會有[7大相關的類別](https://github.com/opencontainers/image-spec/blob/master/spec.md#understanding-the-specification)被標準化，分別是

- Image Manifest - a document describing the components that make up a container image
- Image Index - an annotated index of image manifests
- Image Layout - a filesystem layout representing the contents of an image
- Filesystem Layer - a changeset that describes a container's filesystem
- Image Configuration - a document determining layer ordering and configuration of the image suitable for translation into a runtime bundle
- Conversion - a document describing how this translation should occur
- Descriptor - a reference that describes the type, metadata and content address of referenced content

如果以前有稍微留意過 `Docker Images` 的話就會觀察到有很多個 `Layer` 的產生，每個 `Layer` 都有獨立的 `Image ID`，這邊有興趣的可以參閱 [Filesystem Layer](https://github.com/opencontainers/image-spec/blob/master/layer.md) 來了解更多，到底這些 `Layer` 代表甚麼，以及底層到底怎麼實現。

每個類別在官方文件中都由獨立的檔案去描述，有興趣的可以自行參閱來理解每個部分實際上的關係。

## Summary

`Open Container Initiative (OCI)` 定義了 `Runtime` 以及 `Image` 兩大標準來規範 `Container` 的介面，本文中跟各位探討了 `Runtime/Image` 這兩標準大致上想要完成什麼，以及要如何完成。

下篇文章將會跟大家分享如果今天想要開發一個符合 `OCI` 標準的 `Container` 解決方案，到底有什麼工具可以直接使用而避免重造輪子，同時也能夠透過這些工具來理解到底 `Docker` 目前是怎麼與 `OCI` 標準銜接的。


# Reference
- https://www.slideshare.net/jeevachelladhurai/runc-open-container-initiative
- https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md
- https://github.com/opencontainers/image-spec/blob/master/spec.md