# 第11章: JavaScript/TypeScript

> 🎯 **この章の目標**: JavaScriptのシングルスレッドモデル、イベントループ、非同期処理の進化（Callback→Promise→async/await）、そしてWeb Workersを理解する

---

## 11.1 JavaScriptのシングルスレッドモデル

### なぜシングルスレッドなのか

JavaScriptは1995年にNetscape Navigatorのために、わずか10日間で設計されました。当時の主な目的はWebページに簡単なインタラクティブ性を追加することでした。マルチスレッドによる複雑さ（レースコンディション、デッドロック）を避けるため、シングルスレッドモデルが採用されました。

```mermaid
flowchart TB
    subgraph HISTORY["JavaScript誕生の背景"]
        H1["1995年: Netscape Navigator"]
        H2["目的: Webページのインタラクティブ性"]
        H3["設計期間: わずか10日"]
        H4["決定: シングルスレッドモデル"]
    end
    
    H1 --> H2 --> H3 --> H4
```

### シングルスレッドの意味

JavaScriptのメインスレッド（UIスレッド）は1つだけです。すべてのJavaScriptコードは、このスレッドで順番に実行されます。

```mermaid
flowchart LR
    subgraph SINGLE_THREAD["シングルスレッド実行"]
        CODE1["コード1"]
        CODE2["コード2"]
        CODE3["コード3"]
        CODE4["コード4"]
    end
    
    CODE1 --> CODE2 --> CODE3 --> CODE4
    
    NOTE["1つずつ順番に実行<br/>同時に複数のコードは実行されない"]
```

```javascript
// シングルスレッドの証明
console.log("1: 開始");

// 重い処理（メインスレッドをブロック）
for (let i = 0; i < 1000000000; i++) {
    // 何もしない
}

console.log("2: ループ完了");  // 1の後、必ずここが実行される
console.log("3: 終了");        // 2の後、必ずここが実行される
```

### ブロッキングの問題

シングルスレッドでは、1つの処理が完了するまで次の処理に進めません。重い処理がメインスレッドをブロックすると、UIがフリーズします。

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Main as メインスレッド
    participant UI as UI更新
    
    User->>Main: ボタンクリック
    Main->>Main: 重い処理開始<br/>(5秒かかる)
    
    User->>Main: 別のボタンクリック
    Note over Main: ブロック中...<br/>応答できない
    
    User->>Main: スクロール
    Note over Main: まだブロック中...
    
    Main->>Main: 重い処理完了
    Main->>UI: やっと更新
```

```javascript
// ブロッキングの例
document.getElementById('button').addEventListener('click', () => {
    console.log("クリック！");
    
    // この間、UIは完全にフリーズ
    const start = Date.now();
    while (Date.now() - start < 5000) {
        // 5秒間ブロック
    }
    
    console.log("完了");  // 5秒後にやっと実行
});
```

### 非同期処理による解決

JavaScriptは非同期処理を使って、ブロッキングを回避します。I/O操作などの待ち時間が発生する処理は、バックグラウンドで実行され、完了時にコールバックが呼ばれます。

```mermaid
flowchart TB
    subgraph ASYNC_MODEL["非同期モデル"]
        MAIN["メインスレッド<br/>(JavaScript)"]
        BG["バックグラウンド<br/>(ブラウザAPI / Node.js)"]
        QUEUE["コールバックキュー"]
        LOOP["イベントループ"]
    end
    
    MAIN -->|"非同期処理を依頼"| BG
    BG -->|"完了時にコールバックを登録"| QUEUE
    LOOP -->|"キューからコールバックを取り出し"| MAIN
```

---

## 11.2 イベントループ

### イベントループとは

**イベントループ**は、JavaScriptの非同期処理を実現する中核的な仕組みです。コールバックキューを監視し、メインスレッドが空いたときにキューからタスクを取り出して実行します。

```mermaid
flowchart TB
    subgraph EVENT_LOOP["イベントループの構造"]
        STACK["コールスタック<br/>(実行中の関数)"]
        HEAP["ヒープ<br/>(オブジェクト)"]
        
        subgraph QUEUES["キュー"]
            MICRO["マイクロタスクキュー"]
            MACRO["マクロタスクキュー"]
        end
        
        LOOP["イベントループ"]
        
        WEB_API["Web APIs / Node.js APIs"]
    end
    
    STACK --> LOOP
    LOOP --> MICRO
    MICRO --> STACK
    LOOP --> MACRO
    MACRO --> STACK
    
    WEB_API --> MICRO
    WEB_API --> MACRO
```

### コールスタック

**コールスタック**は、現在実行中の関数を追跡するデータ構造です。関数が呼ばれるとスタックにプッシュされ、関数が完了するとポップされます。

```javascript
function first() {
    console.log("first 開始");
    second();
    console.log("first 終了");
}

function second() {
    console.log("second 開始");
    third();
    console.log("second 終了");
}

function third() {
    console.log("third");
}

first();
```

```mermaid
sequenceDiagram
    participant Stack as コールスタック
    
    Note over Stack: []
    Stack->>Stack: first() をプッシュ
    Note over Stack: [first]
    Stack->>Stack: second() をプッシュ
    Note over Stack: [first, second]
    Stack->>Stack: third() をプッシュ
    Note over Stack: [first, second, third]
    Stack->>Stack: third() をポップ
    Note over Stack: [first, second]
    Stack->>Stack: second() をポップ
    Note over Stack: [first]
    Stack->>Stack: first() をポップ
    Note over Stack: []
```

### Web APIs / Node.js APIs

ブラウザやNode.jsは、非同期操作を処理するためのAPIを提供します。これらは**メインスレッドとは別**で動作します。

```mermaid
flowchart TB
    subgraph BROWSER["ブラウザ環境"]
        B1["setTimeout / setInterval"]
        B2["fetch / XMLHttpRequest"]
        B3["DOM Events"]
        B4["Web Workers"]
    end
    
    subgraph NODEJS["Node.js環境"]
        N1["setTimeout / setInterval"]
        N2["fs (ファイルシステム)"]
        N3["http / net"]
        N4["child_process"]
    end
```

```javascript
console.log("1: 開始");

// setTimeout はブラウザ/Node.js のAPIで処理される
setTimeout(() => {
    console.log("3: タイマー完了");
}, 0);

console.log("2: 終了");

// 出力順序:
// 1: 開始
// 2: 終了
// 3: タイマー完了  ← 0msでも後から実行される
```

### イベントループの動作

```mermaid
flowchart TB
    START["開始"]
    CHECK_STACK{"コールスタックは<br/>空か？"}
    EXECUTE_STACK["スタックのコードを実行"]
    CHECK_MICRO{"マイクロタスク<br/>キューは空か？"}
    EXECUTE_MICRO["マイクロタスクを実行"]
    CHECK_MACRO{"マクロタスク<br/>キューは空か？"}
    EXECUTE_MACRO["マクロタスクを1つ実行"]
    RENDER["レンダリング<br/>(必要に応じて)"]
    
    START --> CHECK_STACK
    CHECK_STACK -->|"No"| EXECUTE_STACK
    EXECUTE_STACK --> CHECK_STACK
    CHECK_STACK -->|"Yes"| CHECK_MICRO
    CHECK_MICRO -->|"No"| EXECUTE_MICRO
    EXECUTE_MICRO --> CHECK_MICRO
    CHECK_MICRO -->|"Yes"| CHECK_MACRO
    CHECK_MACRO -->|"No"| EXECUTE_MACRO
    EXECUTE_MACRO --> RENDER
    RENDER --> CHECK_STACK
    CHECK_MACRO -->|"Yes"| RENDER
```

---

## 11.3 マイクロタスクとマクロタスク

### 2種類のタスクキュー

JavaScriptには2種類のタスクキューがあります：

```mermaid
flowchart TB
    subgraph TASK_TYPES["タスクの種類"]
        subgraph MICRO["マイクロタスク（優先度高）"]
            M1["Promise.then()"]
            M2["Promise.catch()"]
            M3["Promise.finally()"]
            M4["queueMicrotask()"]
            M5["MutationObserver"]
        end
        
        subgraph MACRO["マクロタスク（優先度低）"]
            MA1["setTimeout()"]
            MA2["setInterval()"]
            MA3["setImmediate() (Node.js)"]
            MA4["I/O操作"]
            MA5["UIレンダリング"]
            MA6["requestAnimationFrame()"]
        end
    end
```

### 実行順序

マイクロタスクは、**現在のマクロタスクが完了した直後**、**次のマクロタスクの前**に、すべて実行されます。

```javascript
console.log("1: 同期コード開始");

setTimeout(() => {
    console.log("4: マクロタスク1");
}, 0);

Promise.resolve().then(() => {
    console.log("3: マイクロタスク1");
});

setTimeout(() => {
    console.log("5: マクロタスク2");
}, 0);

Promise.resolve().then(() => {
    console.log("3.5: マイクロタスク2");
});

console.log("2: 同期コード終了");

// 出力順序:
// 1: 同期コード開始
// 2: 同期コード終了
// 3: マイクロタスク1
// 3.5: マイクロタスク2
// 4: マクロタスク1
// 5: マクロタスク2
```

```mermaid
sequenceDiagram
    participant Sync as 同期コード
    participant Micro as マイクロタスク
    participant Macro as マクロタスク
    
    Sync->>Sync: console.log("1")
    Sync->>Macro: setTimeout 登録
    Sync->>Micro: Promise.then 登録
    Sync->>Macro: setTimeout 登録
    Sync->>Micro: Promise.then 登録
    Sync->>Sync: console.log("2")
    
    Note over Sync: 同期コード完了
    
    Micro->>Micro: console.log("3")
    Micro->>Micro: console.log("3.5")
    
    Note over Micro: マイクロタスク完了
    
    Macro->>Macro: console.log("4")
    Macro->>Macro: console.log("5")
```

### 複雑な例

```javascript
console.log("script start");

setTimeout(() => {
    console.log("setTimeout");
}, 0);

Promise.resolve()
    .then(() => {
        console.log("promise1");
    })
    .then(() => {
        console.log("promise2");
    });

Promise.resolve().then(() => {
    console.log("promise3");
});

console.log("script end");

// 出力順序:
// script start
// script end
// promise1
// promise3
// promise2
// setTimeout
```

```mermaid
flowchart LR
    subgraph EXECUTION["実行順序の詳細"]
        S1["1. script start"]
        S2["2. setTimeout をキューに登録"]
        S3["3. Promise1 をキューに登録"]
        S4["4. Promise3 をキューに登録"]
        S5["5. script end"]
        S6["6. promise1 実行"]
        S7["7. promise3 実行"]
        S8["8. promise2 実行"]
        S9["9. setTimeout 実行"]
    end
    
    S1 --> S2 --> S3 --> S4 --> S5
    S5 -->|"マイクロタスク"| S6 --> S7 --> S8
    S8 -->|"マクロタスク"| S9
```

### queueMicrotask

`queueMicrotask()`を使って、明示的にマイクロタスクを登録できます。

```javascript
console.log("start");

queueMicrotask(() => {
    console.log("microtask 1");
});

Promise.resolve().then(() => {
    console.log("promise microtask");
});

queueMicrotask(() => {
    console.log("microtask 2");
});

console.log("end");

// 出力順序:
// start
// end
// microtask 1
// promise microtask
// microtask 2
```

---

## 11.4 Callback → Promise → async/await

### コールバック時代

初期のJavaScript非同期処理は、コールバック関数を使って行われました。

```javascript
// コールバックスタイル
function fetchData(url, callback) {
    const xhr = new XMLHttpRequest();
    xhr.open('GET', url);
    xhr.onload = function() {
        if (xhr.status === 200) {
            callback(null, xhr.responseText);
        } else {
            callback(new Error('Request failed'));
        }
    };
    xhr.onerror = function() {
        callback(new Error('Network error'));
    };
    xhr.send();
}

// 使用例
fetchData('/api/user', function(error, data) {
    if (error) {
        console.error(error);
        return;
    }
    console.log(data);
});
```

### コールバック地獄（Callback Hell）

複数の非同期処理を順次実行すると、ネストが深くなり読みにくくなります。

```javascript
// コールバック地獄の例
getUser(userId, function(error, user) {
    if (error) {
        handleError(error);
        return;
    }
    getOrders(user.id, function(error, orders) {
        if (error) {
            handleError(error);
            return;
        }
        getOrderDetails(orders[0].id, function(error, details) {
            if (error) {
                handleError(error);
                return;
            }
            getShippingInfo(details.shippingId, function(error, shipping) {
                if (error) {
                    handleError(error);
                    return;
                }
                // やっと処理できる...
                displayResult(user, orders, details, shipping);
            });
        });
    });
});
```

```mermaid
flowchart TB
    subgraph CALLBACK_HELL["コールバック地獄"]
        C1["getUser"]
        C2["getOrders"]
        C3["getOrderDetails"]
        C4["getShippingInfo"]
        C5["displayResult"]
        
        C1 --> C2 --> C3 --> C4 --> C5
    end
    
    PROBLEMS["問題点:<br/>・深いネスト<br/>・エラー処理が分散<br/>・可読性が低い<br/>・保守が困難"]
```

### Promise の登場

ES2015（ES6）でPromiseが導入され、非同期処理がより扱いやすくなりました。

```javascript
// Promise を返す関数
function fetchData(url) {
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        xhr.open('GET', url);
        xhr.onload = function() {
            if (xhr.status === 200) {
                resolve(xhr.responseText);
            } else {
                reject(new Error('Request failed'));
            }
        };
        xhr.onerror = function() {
            reject(new Error('Network error'));
        };
        xhr.send();
    });
}

// Promiseチェーン
fetchData('/api/user')
    .then(data => {
        console.log(data);
        return fetchData('/api/orders');
    })
    .then(orders => {
        console.log(orders);
    })
    .catch(error => {
        console.error(error);
    });
```

### Promise の状態

```mermaid
stateDiagram-v2
    [*] --> Pending: new Promise()
    Pending --> Fulfilled: resolve(value)
    Pending --> Rejected: reject(error)
    Fulfilled --> [*]
    Rejected --> [*]
    
    note right of Pending: 待機中
    note right of Fulfilled: 成功
    note right of Rejected: 失敗
```

### Promise チェーンによる改善

```javascript
// コールバック地獄がPromiseチェーンで改善
getUser(userId)
    .then(user => getOrders(user.id))
    .then(orders => getOrderDetails(orders[0].id))
    .then(details => getShippingInfo(details.shippingId))
    .then(shipping => {
        displayResult(shipping);
    })
    .catch(error => {
        handleError(error);  // エラー処理が1箇所に
    });
```

```mermaid
flowchart LR
    subgraph PROMISE_CHAIN["Promiseチェーン"]
        P1["getUser"]
        P2["getOrders"]
        P3["getOrderDetails"]
        P4["getShippingInfo"]
        P5["displayResult"]
        
        P1 -->|".then"| P2 -->|".then"| P3 -->|".then"| P4 -->|".then"| P5
    end
    
    CATCH["catch で<br/>一括エラー処理"]
    
    P1 -.->|"error"| CATCH
    P2 -.->|"error"| CATCH
    P3 -.->|"error"| CATCH
    P4 -.->|"error"| CATCH
```

### Promise のユーティリティメソッド

```javascript
// Promise.all: すべてが成功したら成功
const promises = [
    fetch('/api/user'),
    fetch('/api/orders'),
    fetch('/api/products')
];

Promise.all(promises)
    .then(([user, orders, products]) => {
        console.log('すべて成功:', user, orders, products);
    })
    .catch(error => {
        console.error('どれかが失敗:', error);
    });

// Promise.race: 最初に完了したものを採用
Promise.race([
    fetch('/api/data'),
    new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Timeout')), 5000)
    )
])
    .then(data => console.log(data))
    .catch(error => console.error(error));

// Promise.allSettled: すべての結果を取得（成功/失敗問わず）
Promise.allSettled(promises)
    .then(results => {
        results.forEach((result, i) => {
            if (result.status === 'fulfilled') {
                console.log(`${i}: 成功`, result.value);
            } else {
                console.log(`${i}: 失敗`, result.reason);
            }
        });
    });

// Promise.any: 最初に成功したものを採用
Promise.any(promises)
    .then(firstSuccess => {
        console.log('最初の成功:', firstSuccess);
    })
    .catch(error => {
        console.error('すべて失敗:', error);
    });
```

```mermaid
flowchart TB
    subgraph PROMISE_METHODS["Promiseユーティリティメソッド"]
        subgraph ALL["Promise.all"]
            ALL_DESC["すべて成功 → 成功<br/>1つでも失敗 → 失敗"]
        end
        
        subgraph RACE["Promise.race"]
            RACE_DESC["最初に完了したもの<br/>（成功/失敗問わず）"]
        end
        
        subgraph ALL_SETTLED["Promise.allSettled"]
            ALS_DESC["すべての結果を取得<br/>（成功/失敗を区別）"]
        end
        
        subgraph ANY["Promise.any"]
            ANY_DESC["最初に成功したもの<br/>すべて失敗なら失敗"]
        end
    end
```

### async/await の登場

ES2017でasync/awaitが導入され、非同期コードを同期的なスタイルで書けるようになりました。

```javascript
// async/await スタイル
async function processOrder(userId) {
    try {
        const user = await getUser(userId);
        const orders = await getOrders(user.id);
        const details = await getOrderDetails(orders[0].id);
        const shipping = await getShippingInfo(details.shippingId);
        
        displayResult(shipping);
    } catch (error) {
        handleError(error);
    }
}

processOrder(123);
```

```mermaid
flowchart LR
    subgraph EVOLUTION["非同期処理の進化"]
        CALLBACK["Callback<br/>(ES5以前)"]
        PROMISE["Promise<br/>(ES2015)"]
        ASYNC["async/await<br/>(ES2017)"]
        
        CALLBACK -->|"改善"| PROMISE -->|"糖衣構文"| ASYNC
    end
```

### async/await の仕組み

`async`関数は常にPromiseを返します。`await`はPromiseの解決を待ちます。

```javascript
// async 関数は常に Promise を返す
async function hello() {
    return "Hello";
}

console.log(hello());  // Promise { "Hello" }

hello().then(msg => console.log(msg));  // "Hello"

// await は Promise を解決する
async function example() {
    const result = await Promise.resolve(42);
    console.log(result);  // 42
    
    // await なしだと Promise オブジェクトが返る
    const promise = Promise.resolve(100);
    console.log(promise);  // Promise { 100 }
}
```

### 並列実行 vs 順次実行

```javascript
// 順次実行（遅い）
async function sequential() {
    const start = Date.now();
    
    const a = await fetchData('/api/a');  // 1秒
    const b = await fetchData('/api/b');  // 1秒
    const c = await fetchData('/api/c');  // 1秒
    
    console.log(`完了: ${Date.now() - start}ms`);  // 約3000ms
}

// 並列実行（速い）
async function parallel() {
    const start = Date.now();
    
    const [a, b, c] = await Promise.all([
        fetchData('/api/a'),  // 1秒
        fetchData('/api/b'),  // 1秒
        fetchData('/api/c'),  // 1秒
    ]);
    
    console.log(`完了: ${Date.now() - start}ms`);  // 約1000ms
}
```

```mermaid
flowchart TB
    subgraph SEQUENTIAL["順次実行"]
        S_A["fetchData A<br/>(1秒)"]
        S_B["fetchData B<br/>(1秒)"]
        S_C["fetchData C<br/>(1秒)"]
        
        S_A --> S_B --> S_C
        
        S_TOTAL["合計: 3秒"]
    end
    
    subgraph PARALLEL["並列実行"]
        P_A["fetchData A (1秒)"]
        P_B["fetchData B (1秒)"]
        P_C["fetchData C (1秒)"]
        
        P_TOTAL["合計: 1秒"]
    end
```

### エラーハンドリング

```javascript
// try-catch でエラーハンドリング
async function fetchWithErrorHandling() {
    try {
        const response = await fetch('/api/data');
        if (!response.ok) {
            throw new Error(`HTTP error: ${response.status}`);
        }
        const data = await response.json();
        return data;
    } catch (error) {
        console.error('Error:', error.message);
        throw error;  // 再スロー
    } finally {
        console.log('完了（成功/失敗問わず）');
    }
}

// 個別のエラーハンドリング
async function fetchMultiple() {
    const results = await Promise.allSettled([
        fetch('/api/a').then(r => r.json()),
        fetch('/api/b').then(r => r.json()),
        fetch('/api/c').then(r => r.json()),
    ]);
    
    const successful = results
        .filter(r => r.status === 'fulfilled')
        .map(r => r.value);
    
    const failed = results
        .filter(r => r.status === 'rejected')
        .map(r => r.reason);
    
    return { successful, failed };
}
```

### Top-level await

ES2022から、モジュールのトップレベルで`await`が使えるようになりました。

```javascript
// ES2022: Top-level await (ESモジュールで使用可能)
// config.mjs
const response = await fetch('/api/config');
export const config = await response.json();

// main.mjs
import { config } from './config.mjs';
console.log(config);  // 設定がロード済み
```

---

## 11.5 Web Workers と Worker Threads

### Web Workers（ブラウザ）

**Web Workers**は、メインスレッドとは別のスレッドでJavaScriptを実行する仕組みです。重い計算処理をオフロードして、UIの応答性を維持できます。

```mermaid
flowchart TB
    subgraph BROWSER["ブラウザ"]
        MAIN["メインスレッド<br/>(UI, DOM)"]
        WORKER1["Web Worker 1"]
        WORKER2["Web Worker 2"]
    end
    
    MAIN <-->|"postMessage"| WORKER1
    MAIN <-->|"postMessage"| WORKER2
    
    NOTE["Workers はDOMにアクセス不可<br/>メッセージパッシングで通信"]
```

```javascript
// main.js
const worker = new Worker('worker.js');

// Workerにメッセージを送信
worker.postMessage({ type: 'calculate', data: [1, 2, 3, 4, 5] });

// Workerからのメッセージを受信
worker.onmessage = (event) => {
    console.log('Result:', event.data);
};

worker.onerror = (error) => {
    console.error('Worker error:', error);
};

// worker.js
self.onmessage = (event) => {
    const { type, data } = event.data;
    
    if (type === 'calculate') {
        // 重い計算処理
        const result = data.reduce((sum, n) => sum + n, 0);
        
        // 結果を送信
        self.postMessage(result);
    }
};
```

### Shared Workers

複数のタブ/ウィンドウ間で共有できるWorkerです。

```javascript
// main.js (複数のページで使用)
const sharedWorker = new SharedWorker('shared-worker.js');

sharedWorker.port.onmessage = (event) => {
    console.log('Received:', event.data);
};

sharedWorker.port.postMessage('Hello from page');

// shared-worker.js
const connections = [];

self.onconnect = (event) => {
    const port = event.ports[0];
    connections.push(port);
    
    port.onmessage = (e) => {
        // すべての接続に通知
        connections.forEach(p => {
            p.postMessage(`Broadcast: ${e.data}`);
        });
    };
    
    port.start();
};
```

```mermaid
flowchart TB
    subgraph SHARED_WORKER["Shared Worker"]
        SW["Shared Worker"]
    end
    
    TAB1["タブ1"]
    TAB2["タブ2"]
    TAB3["タブ3"]
    
    TAB1 <-->|"port"| SW
    TAB2 <-->|"port"| SW
    TAB3 <-->|"port"| SW
```

### Service Workers

Service Workersは、ネットワークリクエストをインターセプトし、オフライン対応やキャッシュ戦略を実装できます。

```javascript
// sw.js (Service Worker)
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open('v1').then((cache) => {
            return cache.addAll([
                '/',
                '/index.html',
                '/styles.css',
                '/app.js'
            ]);
        })
    );
});

self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request).then((response) => {
            // キャッシュがあればキャッシュを返す
            if (response) {
                return response;
            }
            // なければネットワークから取得
            return fetch(event.request);
        })
    );
});

// main.js (登録)
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js')
        .then(registration => {
            console.log('SW registered:', registration);
        })
        .catch(error => {
            console.error('SW registration failed:', error);
        });
}
```

### Worker Threads（Node.js）

Node.jsでは、`worker_threads`モジュールを使ってマルチスレッド処理ができます。

```javascript
// main.js
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
    // メインスレッド
    const worker = new Worker(__filename, {
        workerData: { numbers: [1, 2, 3, 4, 5] }
    });
    
    worker.on('message', (result) => {
        console.log('Result:', result);
    });
    
    worker.on('error', (error) => {
        console.error('Error:', error);
    });
    
    worker.on('exit', (code) => {
        console.log('Worker exited with code:', code);
    });
} else {
    // ワーカースレッド
    const { numbers } = workerData;
    const sum = numbers.reduce((a, b) => a + b, 0);
    
    parentPort.postMessage(sum);
}
```

### SharedArrayBuffer

スレッド間でメモリを共有する方法です。

```javascript
const { Worker } = require('worker_threads');

// 共有メモリを作成
const sharedBuffer = new SharedArrayBuffer(4);  // 4バイト
const sharedArray = new Int32Array(sharedBuffer);

sharedArray[0] = 0;

const worker = new Worker(`
    const { parentPort, workerData } = require('worker_threads');
    const sharedArray = new Int32Array(workerData.sharedBuffer);
    
    // Atomics で安全にインクリメント
    for (let i = 0; i < 1000; i++) {
        Atomics.add(sharedArray, 0, 1);
    }
    
    parentPort.postMessage('done');
`, {
    eval: true,
    workerData: { sharedBuffer }
});

worker.on('message', () => {
    console.log('Final value:', sharedArray[0]);  // 1000
});
```

```mermaid
flowchart TB
    subgraph SHARED_MEMORY["SharedArrayBuffer"]
        MAIN["メインスレッド"]
        WORKER["ワーカースレッド"]
        BUFFER["共有メモリ<br/>(SharedArrayBuffer)"]
        
        MAIN <-->|"Atomics"| BUFFER
        WORKER <-->|"Atomics"| BUFFER
    end
```

### Worker の使い分け

```mermaid
flowchart TB
    subgraph WORKER_TYPES["Workerの種類と用途"]
        subgraph WW["Web Worker"]
            WW_USE["・重い計算処理<br/>・データ処理<br/>・画像処理"]
        end
        
        subgraph SW["Shared Worker"]
            SW_USE["・タブ間の状態共有<br/>・WebSocket接続の共有"]
        end
        
        subgraph SVC["Service Worker"]
            SVC_USE["・オフライン対応<br/>・キャッシュ戦略<br/>・プッシュ通知"]
        end
        
        subgraph WT["Worker Threads"]
            WT_USE["・Node.js の<br/>  CPU集約処理<br/>・並列データ処理"]
        end
    end
```

---

## 11.6 TypeScriptでの非同期処理

### 型付きPromise

TypeScriptでは、Promiseに型パラメータを指定できます。

```typescript
// 型付き Promise
function fetchUser(id: number): Promise<User> {
    return fetch(`/api/users/${id}`)
        .then(response => response.json());
}

interface User {
    id: number;
    name: string;
    email: string;
}

// async/await との組み合わせ
async function getUser(id: number): Promise<User> {
    const response = await fetch(`/api/users/${id}`);
    const user: User = await response.json();
    return user;
}
```

### ジェネリクスを使った非同期関数

```typescript
// ジェネリックな非同期関数
async function fetchData<T>(url: string): Promise<T> {
    const response = await fetch(url);
    if (!response.ok) {
        throw new Error(`HTTP error: ${response.status}`);
    }
    return response.json();
}

// 使用例
interface Product {
    id: number;
    name: string;
    price: number;
}

const product = await fetchData<Product>('/api/products/1');
console.log(product.name);  // 型安全
```

### 型ガードとエラーハンドリング

```typescript
// Result型パターン
type Result<T, E = Error> = 
    | { success: true; data: T }
    | { success: false; error: E };

async function safeFetch<T>(url: string): Promise<Result<T>> {
    try {
        const response = await fetch(url);
        if (!response.ok) {
            return { 
                success: false, 
                error: new Error(`HTTP ${response.status}`) 
            };
        }
        const data: T = await response.json();
        return { success: true, data };
    } catch (error) {
        return { 
            success: false, 
            error: error instanceof Error ? error : new Error(String(error))
        };
    }
}

// 使用例
const result = await safeFetch<User>('/api/user/1');
if (result.success) {
    console.log(result.data.name);  // 型安全
} else {
    console.error(result.error.message);
}
```

### Promiseのユーティリティ型

```typescript
// Awaited<T>: Promise の解決値の型を取得
type A = Awaited<Promise<string>>;  // string
type B = Awaited<Promise<Promise<number>>>;  // number

// 関数の戻り値の型からPromiseの中身を取得
async function getUsers(): Promise<User[]> {
    return fetch('/api/users').then(r => r.json());
}

type UsersResult = Awaited<ReturnType<typeof getUsers>>;  // User[]
```

---

## 11.7 まとめ

この章では、JavaScriptの非同期処理について詳しく学びました。

```mermaid
mindmap
    root((第11章のまとめ))
        シングルスレッド
            メインスレッド1つ
            ブロッキングの問題
            非同期で解決
        イベントループ
            コールスタック
            タスクキュー
            マイクロタスク優先
        非同期の進化
            Callback
            Promise
            async/await
        Workers
            Web Worker
            Shared Worker
            Service Worker
            Worker Threads
```

### 重要なポイント

#### 1. JavaScriptはシングルスレッドだが、非同期処理で並行性を実現

JavaScriptのメインスレッドは1つですが、イベントループと非同期APIにより、ブロッキングを回避しながら複数の処理を効率的に行えます。重い計算はWorkerにオフロードできます。

#### 2. イベントループがマイクロタスクとマクロタスクを管理

マイクロタスク（Promise.then等）はマクロタスク（setTimeout等）より優先されます。この順序を理解することで、コードの実行順序を正確に予測できます。

#### 3. Callback → Promise → async/await への進化

非同期処理の書き方は進化してきました。async/awaitにより、非同期コードを同期的なスタイルで書けるようになり、可読性と保守性が大幅に向上しました。

#### 4. Workersでマルチスレッド処理が可能

Web Workers、Shared Workers、Service Workers、Worker Threads（Node.js）を使えば、メインスレッドをブロックせずに重い処理を実行できます。

---

## 📝 練習問題

1. **以下のコードの出力順序を予測し、なぜそうなるか説明してください。**

   ```javascript
   console.log('1');
   
   setTimeout(() => console.log('2'), 0);
   
   Promise.resolve()
       .then(() => console.log('3'))
       .then(() => console.log('4'));
   
   console.log('5');
   ```
   
   ヒント：マイクロタスクとマクロタスクの優先順位を考えてください。

2. **以下のコードを、コールバックスタイル、Promiseチェーン、async/awaitの3つの方法で書いてください。**
   
   処理内容：ユーザー情報を取得 → そのユーザーの注文一覧を取得 → 最初の注文の詳細を表示

3. **Promise.all、Promise.race、Promise.allSettled、Promise.anyの違いを説明し、それぞれが適したユースケースを挙げてください。**

4. **以下の非同期処理を、順次実行と並列実行の両方で実装してください。**

   ```javascript
   async function fetchA() { /* 1秒かかる */ }
   async function fetchB() { /* 2秒かかる */ }
   async function fetchC() { /* 1秒かかる */ }
   ```
   
   それぞれの実行時間の違いを説明してください。

5. **Web Workerを使って、メインスレッドをブロックせずに1から1億までの合計を計算するコードを書いてください。**
   
   ヒント：main.jsとworker.jsの2つのファイルが必要です。

---

## 🔗 次の章へ

[第12章: Python](./12-python.md) では、PythonのGIL、threading、multiprocessing、asyncioについて詳しく学びます。

---

[← 目次に戻る](../index.md) | [← 前章: 高度な並行処理モデル](./10-advanced-models.md)

