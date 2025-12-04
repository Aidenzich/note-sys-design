# 行程 (Process) 與執行緒 (Thread) 概念筆記


Process 是作業系統分配資源的基本單位，擁有獨立的記憶體空間和資源；而 Thread（執行緒）是 Process 內更小、可實際執行指令的單位，與同一個 Process 中的其他 Thread 共享該 Process 的資源。可以將 Process 比喻為一個工廠，而 Thread 則是工廠內的工人，一個工廠裡可以有多個工人（Thread）來執行任務。 

| 特性 | Process (行程) | Thread (執行緒) |
| :--- | :--- | :--- |
| 定義	| 一個正在執行的程式，是系統進行資源分配和調度的獨立單位。 | Process 內部的更小執行單位，負責實際執行指令流程。 |
| 執行單位 | <span style="color: orange;">作業系統分配資源的最小單位。</span> | <span style="color: cyan;">作業系統進行運算調度的最小單位。</span> |
| 資源 | 擁有獨立的記憶體空間、檔案描述符等系統資源。 | 與同 Process 的其他 Thread 共享 Process 的記憶體空間、檔案等資源。 |
| 獨立性 | 每個 Process 之間是獨立的。 | 在同一個 Process 內，多個 Thread 是互相依存的。 |
| 記憶體 | 擁有獨立的記憶體空間。 | 共享同一個 Process 的記憶體空間，但擁有自己獨立的堆疊(Stack)和程式計數器(PC)。 |
| 類比 | 一間工廠 | 工廠中的員工|


## 底層硬體與軟體抽象的區分: Process 究竟存在於哪裡呢？
| 概念 | 存在於何處 | 核心功能 |
| :--- | :--- | :--- |
| **Process** | 作業系統 (OS) | **不存在**。Process 是 OS 為了資源管理與多工而創造的**軟體抽象概念**。 |
| **CPU** | 硬體 | 僅執行指令，依賴 **Program Counter (PC)** 順序抓取指令。 |

> 📌 **重點：** CPU 執行時是「**認地圖（分頁表）不認人（PID）**」。

### 比較總結

| 概念 | 地址空間 | 資源 (檔案、信號) | 核心是否可調度 | 錯誤影響 (終止行為) |
| :--- | :--- | :--- | :--- | :--- |
| **Process / Sub Process** | **獨立** | 獨立 | 是 | 父行程終止，子行程**可能存活** (被 Init 收養)。 |
| **LWP / Thread** | **共享** | 共享 | **是** (最小調度單元) | 任何一個 Thread/LWP 引起主要錯誤，會導致**整個 Process 崩潰**。 |
| **Kernel Thread** | **共享** (但無 User Space) | 共享 | **是** | 僅限核心模式，錯誤可能導致**核心恐慌 (Kernel Panic)**。 |



## 行程家族 vs. 執行緒家族

#### 1. 行程家族 (Process Family)
| 類型 | 定義與特點 | 核心特徵 |
| :--- | :--- | :--- |
| **廣義 Process** | 重量級 (Heavy Weight)，OS 提供資源管理與保護的基本單位。 | **獨立地址空間** (最高的隔離性)。 |
| **Sub Process (子行程)** | **是**一個 Process，由父行程 (Parent Process) 創建並管理。 | **獨立地址空間**，即使 **父行程終止，子行程仍能獨立存活**。 |

- 子行程的獨立存活機制
    * **機制：** 由於子行程擁有自己的獨立地址空間，當 **父行程意外終止或崩潰時**，作業系統會自動將子行程交由 **Init 行程 (PID 1)** 收養 (Adoption)。
    * **結果：** 子行程成為 **孤兒行程 (Orphan Process)**，它仍可繼續執行直到完成任務。這確保了系統任務的健壯性。
    * **與執行緒的對比：** 如果是執行緒（Thread）所屬的 Process 崩潰，那麼所有執行緒都會一起終止。


#### 2. 執行緒家族 (Thread Family)

| 類型 | 定義與特點 | 核心特徵 |
| :--- | :--- | :--- |
| **User-level Thread** | 由程式庫管理，核心不可見。切換速度快，但阻塞會影響整個 Process。 | 共享地址空間，由**程式碼層級**調度。 |
| **Light Weight Process (LWP)** | 由核心支援和管理的執行流，**核心可調度的最小單位**。 | 共享地址空間，由**核心層級**調度。 |
| **Kernel Thread (核心執行緒)** | 僅在**核心模式 (Kernel Mode)** 執行的執行緒。用於 OS 背景服務與管理。 | 共享地址空間，**無使用者層級地址空間**。 |

## Single-Thread vs Multi-Thread vs Asynchronous I/O vs Concurrency vs Parallelism 
| 特性 | Single-Thread (單執行緒) | Multi-Thread (多執行緒) | Multi-Process (多行程) | Asynchronous I/O (非同步 I/O) | Concurrency (並行性) | Parallelism (平行性) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **類型** | Unit of execution | <span style="color: cyan;">Mechanism</span> | <span style="color: cyan;">Mechanism</span> | <span style="color: cyan;">Mechanism</span> | <span style="color: orange;">Concept</span> | <span style="color: orange;">Concept</span> |
| **核心要求** | 1 個核心 | 1 個核心以上 | 1 個核心以上 | 1 個主核心 (Event Loop) + 可選的背景 Worker Pool(可多核心) | 單核即可 | **需要多核** |
| **執行方式** | 嚴格依序 | 單核：交錯執行 (如: Python GIL) <br> 多核：同時執行 <br> (如 C++, Java, Go, C#) | 單核：交錯執行 <br> 多核：同時執行 | **單核調度，非阻塞 I/O** | 邏輯上同時推進 | **物理上同時執行** |
| **記憶體共享** | - | **共享** (在 Process 內) | **不共享** (獨立地址空間) | **主循環單線程** (內部通訊) | - | - |
| **資源開銷** | 最低 | 中等 (切換開銷小) | 最高 (切換開銷大) | **極低** (主循環開銷極小) | - | - |
| **故障隔離** | - | 差 (一個崩潰，全 Process 崩潰) | **最佳** (Process 間完全獨立) | **極差** (主循環崩潰，全 Process 崩潰) | - | - |
| **典型用途** | 簡單、非 I/O 任務 | CPU 密集型 和 I/O 密集型 皆可受益 | CPU 密集型 任務，可充分利用多核實現平行性 | **超高 I/O 密集型** (高頻率網路連線) | - | - |





## Appendix
### Linux 的統一實作
- 單一抽象： Linux 核心將所有執行單元 (Process, User Thread, Kernel Thread) 統一為 Task (任務)，由 task_struct 結構體表示。
- 調度單位： LWP (Light Weight Process) 是 OS 理論中核心可調度的最小單位。在 Linux 實作中，Task (task_struct) 就是這個調度單位。
- 綜合： User Thread、LWP、Kernel Thread 在本質上都是核心的 Task，它們之間的區別在於 資源共享的程度：
    - User Thread/LWP： 是具備使用者地址空間的 Task，用於執行應用程式碼。
    - Kernel Thread： 是不具備使用者地址空間的 Task，專門在核心模式中執行系統背景服務。

### Multi-Thread vs Python GIL
GIL 是一個 Mutex (互斥鎖)。它的作用是保護 Python 直譯器的內部狀態和記憶體管理，確保同一時間只有一個執行緒可以執行 Python 位元組碼 (Bytecode)。

對多核執行的影響:
- CPU 密集型任務： 由於 GIL 的存在，即使您的電腦有 8 個核心，您的 Python Multi-Thread 程式也只會有 一個執行緒 在任何時間點能夠執行 Python 程式碼。
- 結果： 這些執行緒被迫在單一核心上不斷地進行快速的交錯執行 (Interleaving)，而不是在多個核心上平行執行 (Parallelism)。這增加了上下文切換的開銷，反而可能導致速度變慢。 
- GIL 對 Asynchronous I/O 的影響： 在單個主核心的Event Loop 不會有影響(因為仍是在一個 thread 內)，但是多個核心的背景 Worker Pool 仍會受到 GIL 的限制。


### Multi-Thread vs Goroutine
Goroutine 其實算是基於 Multi-Thread 建立的一種更高階（或更輕量級）的概念。 它同時融合了 Multi-Thread 與 Asynchronous I/O 的特點。 

| 特性 | **Goroutine** (Go 協程) | **OS Thread** (作業系統執行緒) |
| :--- | :--- | :--- |
| **管理者** | **Go Runtime** (Go 程式的運行時) | **OS Kernel** (作業系統核心) |
| **開銷** | **極低** (幾 KB 堆疊) | **高** (幾 MB 固定堆疊) |
| **切換速度** | **極快** (在使用者空間切換，無需陷入核心) | **慢** (需由 OS 核心介入切換) |
| **視野** | 運行在 M:N 模型中，對**程式設計師可見**且易於使用。 | 運行在 1:1 模型中，是**硬體執行**的最小單位。 |
| **概念層級** | **高階抽象**：專門為高並行性設計，對 I/O 阻塞有智慧處理。 | **底層基礎**：是 Goroutine 最終得以執行的**物理載體**。 |


> Go Scheduler 本身不是運行在某一個特定的 OS Thread 上，它是 Go Runtime 的一部分，它的邏輯和排程決策是分散在所有正在運行的 OS Thread (N) 和邏輯核心 (P) 上的。