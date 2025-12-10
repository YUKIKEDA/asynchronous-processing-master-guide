# 第1章: CPUとプログラム実行の仕組み

> 🎯 **この章の目標**: コンピュータがプログラムをどのように実行するのかを理解し、非同期処理を学ぶための土台を築く

---

## 1.1 フォン・ノイマンアーキテクチャ

現代のほとんどのコンピュータは**フォン・ノイマンアーキテクチャ**に基づいています。これは1945年に数学者ジョン・フォン・ノイマンが提唱した設計思想です。

### 基本構成要素

```mermaid
flowchart TB
    subgraph CPU["🧠 CPU（中央処理装置）"]
        CU["制御装置\n(Control Unit)"]
        ALU["演算装置\n(ALU)"]
        REG["レジスタ群"]
    end
    
    subgraph MEM["💾 メインメモリ"]
        PROG["プログラム\n（命令列）"]
        DATA["データ"]
    end
    
    subgraph IO["🔌 入出力装置"]
        INPUT["入力装置\n(キーボード等)"]
        OUTPUT["出力装置\n(ディスプレイ等)"]
    end
    
    BUS["📡 システムバス"]
    
    CPU <--> BUS
    MEM <--> BUS
    IO <--> BUS
```

### 主要な特徴

| 特徴 | 説明 |
|------|------|
| **プログラム内蔵方式** | プログラムとデータを同じメモリに格納する |
| **逐次実行** | 命令を1つずつ順番に実行する |
| **二進数表現** | すべての情報を0と1で表現する |

### なぜこれが重要なのか？

フォン・ノイマンアーキテクチャの**逐次実行**という特性が、非同期処理を理解する上で非常に重要です。

```mermaid
sequenceDiagram
    participant CPU as CPU
    participant MEM as メモリ
    
    Note over CPU,MEM: 基本的に1度に1つの命令しか実行できない
    
    CPU->>MEM: 命令1を読み込み
    CPU->>CPU: 命令1を実行
    CPU->>MEM: 命令2を読み込み
    CPU->>CPU: 命令2を実行
    CPU->>MEM: 命令3を読み込み
    CPU->>CPU: 命令3を実行
```

> 💡 **ポイント**: CPUは基本的に「1度に1つのこと」しかできません。これが非同期処理が必要になる根本的な理由の一つです。

---

## 1.2 命令サイクル（フェッチ・デコード・実行）

CPUがプログラムを実行する際、各命令は**命令サイクル**と呼ばれる一連のステップを経ます。

### 命令サイクルの3つのフェーズ

```mermaid
flowchart LR
    subgraph CYCLE["🔄 命令サイクル"]
        F["📥 フェッチ\n(Fetch)"]
        D["🔍 デコード\n(Decode)"]
        E["⚡ 実行\n(Execute)"]
    end
    
    F --> D --> E --> F
    
    style F fill:#e1f5fe
    style D fill:#fff3e0
    style E fill:#e8f5e9
```

### 各フェーズの詳細

#### 1️⃣ フェッチ（Fetch）- 命令の取得

```mermaid
flowchart LR
    subgraph CPU
        PC["プログラムカウンタ\n(PC: 0x1000)"]
        IR["命令レジスタ\n(IR)"]
    end
    
    subgraph Memory["メモリ"]
        M1["0x0FFF: ..."]
        M2["0x1000: ADD R1, R2"]
        M3["0x1001: ..."]
    end
    
    PC -->|"アドレス指定"| M2
    M2 -->|"命令を転送"| IR
```

- **プログラムカウンタ（PC）** が次に実行する命令のアドレスを保持
- そのアドレスからメモリの命令を**命令レジスタ（IR）** に読み込む
- PCをインクリメント（次の命令を指すように更新）

#### 2️⃣ デコード（Decode）- 命令の解読

```mermaid
flowchart TB
    IR["命令レジスタ\nADD R1, R2"]
    
    subgraph DECODE["制御装置による解読"]
        OP["オペコード: ADD\n（加算命令）"]
        OP1["オペランド1: R1\n（第1レジスタ）"]
        OP2["オペランド2: R2\n（第2レジスタ）"]
    end
    
    SIGNAL["制御信号の生成"]
    
    IR --> DECODE
    DECODE --> SIGNAL
```

- 命令を**オペコード**（何をするか）と**オペランド**（何に対して）に分解
- 必要な制御信号を生成

#### 3️⃣ 実行（Execute）- 命令の実行

```mermaid
flowchart LR
    subgraph BEFORE["実行前"]
        R1a["R1 = 5"]
        R2a["R2 = 3"]
    end
    
    subgraph ALU["ALU（演算装置）"]
        ADD["加算処理\n5 + 3"]
    end
    
    subgraph AFTER["実行後"]
        R1b["R1 = 8"]
        R2b["R2 = 3"]
    end
    
    BEFORE --> ALU --> AFTER
```

- ALUや他のユニットが実際の処理を実行
- 結果をレジスタやメモリに書き戻す

### 命令サイクルの実行例

```mermaid
sequenceDiagram
    participant PC as プログラムカウンタ
    participant MEM as メモリ
    participant IR as 命令レジスタ
    participant CU as 制御装置
    participant ALU as 演算装置
    participant REG as レジスタ
    
    Note over PC,REG: 命令: ADD R1, R2 の実行
    
    rect rgb(225, 245, 254)
        Note over PC,MEM: フェッチ
        PC->>MEM: アドレス 0x1000 を指定
        MEM->>IR: "ADD R1, R2" を転送
        PC->>PC: PC = PC + 1
    end
    
    rect rgb(255, 243, 224)
        Note over IR,CU: デコード
        IR->>CU: 命令を解析
        CU->>CU: ADD命令と判定
        CU->>CU: R1, R2が対象と判定
    end
    
    rect rgb(232, 245, 233)
        Note over REG,ALU: 実行
        REG->>ALU: R1(5), R2(3) を転送
        ALU->>ALU: 5 + 3 = 8
        ALU->>REG: 結果(8)をR1に格納
    end
```

---

## 1.3 レジスタとメモリ

### メモリ階層

コンピュータには複数種類の記憶装置があり、階層構造を形成しています。

```mermaid
flowchart TB
    subgraph HIERARCHY["📊 メモリ階層"]
        REG["🔴 レジスタ\n容量: 数百バイト\n速度: 1サイクル"]
        L1["🟠 L1キャッシュ\n容量: 数十KB\n速度: 数サイクル"]
        L2["🟡 L2キャッシュ\n容量: 数百KB\n速度: 十数サイクル"]
        L3["🟢 L3キャッシュ\n容量: 数MB\n速度: 数十サイクル"]
        RAM["🔵 メインメモリ(RAM)\n容量: 数GB〜数十GB\n速度: 数百サイクル"]
        SSD["🟣 SSD/HDD\n容量: 数百GB〜数TB\n速度: 数万〜数百万サイクル"]
    end
    
    REG --- L1 --- L2 --- L3 --- RAM --- SSD
    
    FAST["⚡ 高速・小容量・高価"] -.-> REG
    SLOW["🐢 低速・大容量・安価"] -.-> SSD
```

### レジスタの種類

```mermaid
flowchart TB
    subgraph REGISTERS["🔧 主要なレジスタ"]
        subgraph GENERAL["汎用レジスタ"]
            R0["R0 / RAX"]
            R1["R1 / RBX"]
            R2["R2 / RCX"]
            RN["..."]
        end
        
        subgraph SPECIAL["特殊レジスタ"]
            PC["PC\nプログラムカウンタ\n次の命令のアドレス"]
            SP["SP\nスタックポインタ\nスタックの先頭アドレス"]
            BP["BP\nベースポインタ\nスタックフレームの基準"]
            FLAGS["FLAGS\nステータスレジスタ\n演算結果のフラグ"]
        end
    end
```

### なぜレジスタが重要か？

```mermaid
flowchart LR
    subgraph COMPARISON["⏱️ アクセス速度の比較"]
        REG_TIME["レジスタ\n≈ 1ナノ秒"]
        RAM_TIME["メインメモリ\n≈ 100ナノ秒"]
        SSD_TIME["SSD\n≈ 100,000ナノ秒"]
    end
    
    REG_TIME -->|"100倍遅い"| RAM_TIME
    RAM_TIME -->|"1000倍遅い"| SSD_TIME
```

> 💡 **ポイント**: メモリアクセスには時間がかかります。特にストレージ（SSD/HDD）へのアクセスは非常に遅く、これが**I/O待ち**の原因となり、非同期処理の必要性につながります。

---

## 1.4 スタックとヒープ

プログラムが使用するメモリは、主に**スタック**と**ヒープ**という2つの領域に分けられます。

### メモリレイアウト

```mermaid
flowchart TB
    subgraph MEMORY["🗄️ プログラムのメモリ空間"]
        direction TB
        
        HIGH["高いアドレス ↑"]
        
        subgraph STACK["📚 スタック領域"]
            S1["ローカル変数"]
            S2["関数の引数"]
            S3["戻りアドレス"]
        end
        
        FREE["⬇️ 空き領域 ⬆️"]
        
        subgraph HEAP["🗃️ ヒープ領域"]
            H1["動的に確保された"]
            H2["オブジェクト"]
        end
        
        subgraph STATIC["📋 静的領域"]
            ST1["グローバル変数"]
            ST2["静的変数"]
        end
        
        subgraph CODE["💻 コード領域"]
            C1["プログラムの命令"]
        end
        
        LOW["低いアドレス ↓"]
    end
    
    HIGH ~~~ STACK
    STACK ~~~ FREE
    FREE ~~~ HEAP
    HEAP ~~~ STATIC
    STATIC ~~~ CODE
    CODE ~~~ LOW
```

### スタック（Stack）

スタックは**LIFO（Last In, First Out）** 構造で、関数呼び出しに使われます。

```mermaid
sequenceDiagram
    participant MAIN as main()
    participant STACK as スタック
    participant FUNC_A as funcA()
    participant FUNC_B as funcB()
    
    Note over STACK: 空の状態
    
    MAIN->>STACK: main()のフレームをプッシュ
    Note over STACK: [main]
    
    MAIN->>FUNC_A: funcA()を呼び出し
    FUNC_A->>STACK: funcA()のフレームをプッシュ
    Note over STACK: [main][funcA]
    
    FUNC_A->>FUNC_B: funcB()を呼び出し
    FUNC_B->>STACK: funcB()のフレームをプッシュ
    Note over STACK: [main][funcA][funcB]
    
    FUNC_B->>STACK: funcB()のフレームをポップ
    Note over STACK: [main][funcA]
    FUNC_B->>FUNC_A: 戻る
    
    FUNC_A->>STACK: funcA()のフレームをポップ
    Note over STACK: [main]
    FUNC_A->>MAIN: 戻る
```

#### スタックフレームの構造

```mermaid
flowchart TB
    subgraph FRAME["📦 スタックフレーム（関数1つ分）"]
        direction TB
        RET["戻りアドレス\n（呼び出し元に戻るため）"]
        OLD_BP["前のベースポインタ"]
        LOCAL["ローカル変数群\nint x = 10;\nchar c = 'A';"]
        ARGS["引数\narg1, arg2, ..."]
    end
    
    SP["SP（スタックポインタ）"] --> RET
    BP["BP（ベースポインタ）"] --> OLD_BP
```

### ヒープ（Heap）

ヒープは**動的メモリ確保**に使われます。

```mermaid
flowchart LR
    subgraph CODE["プログラムコード"]
        A["int* p = malloc(100);"]
        B["// pを使用"]
        C["free(p);"]
    end
    
    subgraph HEAP["ヒープメモリ"]
        H1["空き"]
        H2["確保された\n100バイト"]
        H3["空き"]
    end
    
    A -->|"確保"| H2
    C -->|"解放"| H1
```

### スタック vs ヒープ

```mermaid
flowchart TB
    subgraph COMPARISON["🔄 スタック vs ヒープ"]
        subgraph STACK_CHAR["スタックの特徴"]
            SC1["✅ 高速なアクセス"]
            SC2["✅ 自動的なメモリ管理"]
            SC3["❌ サイズが固定的"]
            SC4["❌ 関数終了時に消える"]
        end
        
        subgraph HEAP_CHAR["ヒープの特徴"]
            HC1["✅ 柔軟なサイズ"]
            HC2["✅ 長期間保持可能"]
            HC3["❌ アクセスが遅い"]
            HC4["❌ 手動管理が必要\n（またはGC）"]
        end
    end
```

| 特徴 | スタック | ヒープ |
|------|----------|--------|
| 確保速度 | ⚡ 非常に高速 | 🐢 比較的遅い |
| サイズ | 📏 コンパイル時に決定 | 📐 実行時に決定 |
| 管理 | 🤖 自動（スコープベース） | 👨‍💻 手動 or GC |
| 断片化 | ❌ 発生しない | ⚠️ 発生しうる |
| 用途 | ローカル変数、関数呼び出し | 動的データ構造、大きなオブジェクト |

---

## 1.5 まとめ

```mermaid
mindmap
    root((第1章のまとめ))
        フォン・ノイマン
            プログラム内蔵方式
            逐次実行
            CPU・メモリ・I/O
        命令サイクル
            フェッチ
            デコード
            実行
        レジスタとメモリ
            メモリ階層
            速度の違い
            I/O待ちの原因
        スタックとヒープ
            スタック: 関数呼び出し
            ヒープ: 動的確保
            メモリ管理
```

### 重要なポイント

1. **CPUは基本的に1度に1つの命令しか実行できない**
   - これが並行処理・非同期処理が必要になる根本的な理由

2. **メモリアクセスには時間がかかる**
   - 特にストレージへのアクセスは非常に遅い
   - この「待ち時間」を有効活用するのが非同期処理

3. **関数呼び出しはスタックで管理される**
   - この仕組みを理解することで、コールバックやコルーチンの動作が理解できる

---

## 📝 練習問題

1. フォン・ノイマンアーキテクチャの「逐次実行」という特性は、現代のマルチコアCPUでどのように拡張されていますか？

2. レジスタとメインメモリのアクセス速度の違いは約何倍ですか？この違いが非同期処理とどう関係しますか？

3. 再帰関数を呼び出すと、スタックにはどのような影響がありますか？「スタックオーバーフロー」とは何ですか？

---

## 🔗 次の章へ

[第2章: プロセスとスレッド](./02-process-thread.md) では、OSがプログラムをどのように管理・実行するのかを学びます。

---

[← 目次に戻る](../index.md)

