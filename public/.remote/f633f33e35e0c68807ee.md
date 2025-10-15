---
title: JVMのGarbage Collectionを理解する
tags:
  - Java
  - Datadog
  - SRE
private: false
updated_at: '2025-08-19T10:17:49+09:00'
id: f633f33e35e0c68807ee
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

こんにちは、Javaのアプリを運用しているゾイです！
最近、アプリのパフォーマンスに問題があり、調査したところGarbage Collection（GC）が原因ではないかと疑っていました🤔
そこで、GCがどのように動作しているかを正しく理解し、最適化する必要がありました。
この記事では、JVMのGarbage Collectionの基本から、実際に観察した問題とその対策について詳しく解説したいと思います。

## ⚠️ TL;DR – Major GCに注意！

- **Major GC（別名 Full GC）が発生するとアプリ全体が停止します。しかもガチで止まります😵**
- Major GC自体が悪いわけではありませんが、発生する時間が長いとユーザーに影響が出ます。
- なので、**Major GCの頻度を下げることが重要**です
- ユーザーに遅延が伝わる前に、Major GCのメトリクスを常に監視しましょう！


## Javaアプリのメモリの使い方

まず、Javaアプリがどのようにメモリを使用しているかを理解することが重要です。
Javaアプリは、以下のようなメモリ領域を使用します。

### 1. Heapメモリ（本記事では主にこれを扱います）

- 格納：オブジェクトと配列
- 管理：Garbage Collectorによって管理される
- 構成：
  - **Young Gen**：EdenとSurvivor
  - **Old Gen**：長寿命オブジェクトが昇格される場所

🔸 例：`new User()` はHeapにメモリが確保される

### 2. Stackメモリ

- 格納：ローカル変数、メソッド呼び出しの情報
- スレッドごとのスタック
- メソッド終了時に自動解放

🔸 例：`int count = 5` はStackに入る

### 3. Metaspace（Java 8以降）

- 格納：クラスのメタデータ
- ネイティブメモリを使用（Heapではない）
- 自動拡張されるが、OOMの可能性あり

🔸 例：`class Product` のようなクラス定義

### 4. Code Cache

- 格納：JITコンパイルされたバイトコード

### 5. Native Memory

- 含まれるもの：スレッドスタック、DirectByteBuffer、JNIなど
- `-Xmx`の制御外

## Heapについて

GCが行われるHeapについて詳しく見ていきましょう。

### 🧱 Heapの構造

| 世代 | 説明 |
|------|------|
| Young Generation | 新しいオブジェクトが入る（EdenとSurvivor） |
| Old Generation | 長く生きたオブジェクトが昇格される |
| Metaspace | （Heapではないが）クラスのメタデータが入る |

### Heapサイズはどう決まるの？

Heapはシステムメモリに比例して自動拡張されないので、以下の設定で指定できます：

- `-Xms`：初期Heapサイズ
- `-Xmx`：最大Heapサイズ

指定しない場合、JVMはデフォルトのサイズを使用します。

| メモリ上限 | デフォルトの最大Heap |
|------------|------------------|
| 2 GB | 約512 MB〜1 GB |
| 8 GB | 約2〜4 GB |
| 16 GB | 約4〜8 GB |



### 💥 Heapの管理Tips

- 小さすぎる → `OutOfMemoryError`
- 大きすぎる → GCが遅延する（メモリーが大きければ大きいほど、探しに行く時間が長くなる）
- `-Xms`, `-Xmx`でチューニングしたら良き！

## 📦 Garbage Collection（GC）とは？

Javaのメモリ管理について理解できたので、いよいよ本題のGarbage Collection（GC）に入りたいと思います。
Garbage Collectionは、メモリを自動で管理する仕組みです。
不要になったオブジェクトを特定して破棄し、新しいオブジェクトのためのメモリを確保します。

Javaでは、GCは以下のように動作します：

1. `User user = new User();` → Eden
2. GCを生き延びる → Survivor → Old Genへ
3. 参照がなくなる → GCによって破棄される

## Minor GCとMajor GCの違い

JavaのGCには主に2つのタイプがあります。
主な違いは、対象となるメモリ領域と処理の重さです。

| 種類 | 対象 | 頻度 | 負荷 | 停止時間 | 目的 |
|------|------|------|------|----------|------|
| Minor GC | Young Gen | 頻繁（数秒単位） | 低い | 短い | 短命オブジェクトの掃除 |
| Major GC | Old Gen | 稀（数分〜） | 高い | 長い | 長寿命オブジェクトの掃除 |
| Full GC  | Heap全体＋Metaspace | 非常時 | 非常に高い | 最長 | 全メモリの大掃除 |

## GCがスパイクするとどうなるの？

### 📈 Minor GCスパイク

- オブジェクトの生成と破棄が激しい
- Edenがすぐにいっぱいになる
- 頻度が高いとCPU負荷やレイテンシ増加


✅ 対処（インフラ視点）：
- `-Xmn`でYoung Genを増やす
- オブジェクトプールの活用

✅ 対処（アプリ視点）：
- 一時的なオブジェクトを過剰に生成していないか確認（例: `new String()` の多用）
- キャッシュやバッファの設計が適切か見直す
- 不要なオブジェクトを再利用するようコードを改善


### 📈 Major GCスパイク ← 要注意！

- Old Genの圧迫
- メモリリークの可能性
- **長時間アプリが止まる**

✅ 対処（インフラ視点）：
- ヒープダンプでメモリの状態を分析
- Heapサイズの増加（`-Xmx`を調整）
- DatadogなどでOld Genの使用量を継続監視

✅ 対処（アプリ視点）：
- 開発チームと連携し、どの処理やクラスがメモリを大量に保持しているか確認
- メモリリークや、想定以上に寿命が長くなっているオブジェクトの特定
- 保持オブジェクトのライフサイクルを見直し、不要な参照を解放
- `finalize()`やキャッシュの使い方が適切か確認


## GC関連メトリクスのDatadogクエリ

DatadogでGCのメトリクスを監視するために役たつクエリを紹介します。

### Minor GC Count

```hcl
exclude_null(avg:jvm.gc.minor_collection_count{...})
```

### Minor GC Time

```hcl
exclude_null(avg:jvm.gc.minor_collection_time{...})
```

### Major GC Count

```hcl
exclude_null(avg:jvm.gc.major_collection_count{...})
```

### Major GC Time

```hcl
exclude_null(avg:jvm.gc.major_collection_time{...})
```

### Old Gen Size

```hcl
exclude_null(avg:jvm.gc.old_gen_size{...})
```

## 実際私が経験したGCの問題

### 観察したこと

1. リリース直後にMinor GCが急増
2. 数日後にMajor GCが発生
3. Old Genが徐々に増加
4. メモリ使用量が40%跳ね上がった🤯

### 原因

新しくリリースされたサービスが原因で、メモリ使用量が急増。
開発者に依頼して、ヒープを圧迫しているオブジェクトを特定してもらいました。
おそらくメモリリーク、または不要な参照が原因でした。

## TL;DR

- GCはJavaのメモリを自動で管理してくれる
- Minor GC = Young Gen → 高頻度・低負荷
- Major GC = Old Gen → 低頻度・高負荷
- Heapの構造（Eden → Survivor → Old）がGC動作に影響
- GCスパイクは注意：Minorは負荷増、Majorはリークサイン
- **Major GC自体は悪くないが、発生時間が長いとユーザーに影響が出る**
- `-Xms`, `-Xmx`で調整し、Datadogなどでメトリクス監視を徹底
