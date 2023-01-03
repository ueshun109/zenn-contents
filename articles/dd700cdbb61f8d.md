---
title: "初めてのHealthKit"
emoji: "💚"
type: "tech"
topics:
  - "ios"
  - "swift"
  - "healthkit"
published: true
published_at: "2021-05-03 13:04"
---

# HealthKitの概要
HealthKitはiPhoneとAppleWatchによって収集されたヘルスデータ（心拍数や睡眠 etc.）とフィットネスデータ（ランニングや水泳 etc.）の読み書きを行うためのAPIを提供しています。

ヘルスケアデータやフィットネスデータはAppleWatchに保存されますが、AppleWatchとペアリングしているiPhoneが近くにある場合は、そのiPhoneにデータが自動的にバックアップされます。
またiCloudへのバックアップを有効にしている場合は、iCloudにもデータがバックアップされます。
以下のようなイメージです。
![](https://storage.googleapis.com/zenn-user-upload/d4x62hxg0tv2j54it9i5i3oi6v6x =200x)

# HealthKitで扱うデータについて
HeakthKitで扱うデータは主に`HKSample`というオブジェクトで表しています。
`HKSample`は以下のような要素で構成されています。
- データタイプ(`HKSampleType`)
- データの値(`HKSample`のサブクラスによってプロパティは異なる)
- 記録した時間(`startDate`, `endDate`)
- データを作成したアプリやデバイスの情報(`device`)
- メタデータ（`metadata`）

:::message
メタデータには他に保存したいデータが入っています。
e.g. 運動が屋外か屋内か、天気、データを記録したアプリケーション etc.
:::

HealthKitで扱うデータタイプは多種多様であり、それらは`HKSample`のサブクラスとして表されています。次にその一例として`HKQuantitySample`について見ていきます。

# データを保存する
HealthKitで扱うデータについて知るために、試しに歩行距離を保存してみます。流れとしては以下のとおりです。
1. 書き込み権限のリクエスト
2. サンプルデータの作成
3. HealthKitにデータを保存する

## 書き込み権限のリクエスト
```swift
// 1. アクセスしたいデータタイプを指定します。
let distanceType = HKObjectType.quantityType(forIdentifier: .distanceWalkingRunning)

// 2. toShareには書き込みしたいデータタイプを指定しています。
healthStore.requestAuthorization(toShare: [distanceType], read: nil) { success, error in
    if success {
    } else {
    }
}
```
1. `HKObjectType`に特定のデータタイプを表すオブジェクトを返すクラスメソッドが用意されています。　歩行距離は量を表すデータであるため、量を表すデータを書き込めるように`quantityType(forIdentifier:)`を使用して`HKQuantityType`を取得します。。`forIdentifier`にはどのような量のデータがほしいかを指定します。今回は歩行距離のデータを書き込みたいため`distanceWalkingRunning`を指定しています。
2. `healthStore.requestAuthorization`でヘルスデータへのアクセス権限をリクエストします。今回は歩行距離のデータのみを書き込みたいので`toShare`に`distanceType`を指定します。

:::message
個人情報であるヘルスデータはプライバシー情報なので取り扱いが重要になります。
ですので許可を求めるのは、特定のデータが必要なときに毎回申請するようにするほうが良いでしょう。
(e.g. 別のタイミングで`heartRate`のデータにアクセスしたいタイミングで上記の権限リクエストする。)
:::

## 歩行/走行距離のデータの作成と保存
次に歩行/走行の距離を保存してみます。
```swift
// 1. データのタイプを指定
let distanceType = HKObjectType.quantityType(forIdentifier: .distanceWalkingRunning)!

// 2. データを記録した日時
let startDate = Calendar.current.date(bySettingHour: 14, minute: 35, second: 0, of: Date())!
let endDate = Calendar.current.date(bySettingHour: 15, minute: 0, second: 0, of: Date())!

// 3. 値
// HKQuantityは、値と単位を持っています。
let distanceQuantity = HKQuantity(unit: .meter(), doubleValue: 628.0)

// 4. 上記の要素を一つのデータとしてまとめる
let sample = HKQuantitySample(
    type: distanceType,
    quantity: distanceQuantity,
    start: startDate,
    end: endDate
)

// 5. データを保存
healthStore.save(sample) { success, error in
    if success {
    } else {
    }
}
```

1. 保存したいデータのタイプを指定します。<br> `distanceWalkingRunning`は歩行距離と走行距離の`HKQuantityType`です。
2. データを記録した日時を指定します。
3. `HKQuantity`は量に関するデータを表すオブジェクトです。
4. `HKQuantitySample`でこれまで用意したコンポーネントをまとめます。
5. `HKHealthStore`でクエリを実行し、データを保存します。クロージャには保存の成否がコールバックされます。
これでサンプルデータの保存が完了しました。

今回は距離に関するデータを扱いたかったため`HKQuantitySample`を使用しましたが、
当然`HKQuantitySample`が向かないデータもあります。睡眠状態やワークアウトなどのデータです。
睡眠状態のデータは量で表すことができません。起きている、ベッドにいる、睡眠中、など定量的なデータではなく定質的なデータで表されます。
試しに睡眠状態も保存してみます。

## 睡眠状態のデータの作成と保存
定量的なデータタイプは`HKQuantityType`で表していましたが、定質的なデータは`HKCategoryTtpe`で表します。
また睡眠状態のデータは表すには`sleepAnalysis`を用います。これらを用いると以下のような感じのデータを作成します。
|時間|状態|
|:--:|:--:|
|00:00 ~ 01:00|ベッドいる|
|01:00 ~ 07:00|睡眠中|
|07:00 ~ 07:05|起床|

以下は01:00~07:00まで睡眠中状態であることを保存する処理です。

```swift
// 1. データのタイプを指定
let sleepType = HKObjectType.categoryType(forIdentifier: .sleepAnalysis)!

// 2. 開始/終了時刻
let startDate = Calendar.current.date(bySettingHour: 01:00, minute: 00, second: 0, of: Date())!
let endDate = Calendar.current.date(bySettingHour: 07:00, minute: 00, second: 0, of: Date())!

// 3. 上記の要素を一つのデータとしてまとめる
let sample = HKCategorySample(
    type: sleepType,
    value: HKCategoryValueSleepAnalysis.inBed.rawValue,
    start: startDate,
    end: endDate
)

// 4. 上記の要素を一つのデータとしてまとめる
self.healthStore.save(sample) { success, error in
    if success {
        print("success")
    } else {
        print("failed")
    }
}
```
基本的な流れは、歩行/走行距離を保存した時と同じですが、データを一つにまとめる際のクラスが異なります。睡眠状態は定質的なデータであるため、`HKCategorySample`を使用し、`value`引数は`Int`を取り、ベッドにいる、睡眠中、起床のそれぞれの状態に対応しています。

このように扱うデータのタイプに応じて、Healthkitで用意されているものの中から適当なオブジェクトを選択するようにしてください。

## ここまでのまとめ
ここまでHealthKitでデータを扱うためにまずはデータ構造について見てきました。
HealthKitで扱うデータで重要なことはデータのタイプです。具体的には、
- 定量的なデータのタイプは`HKQuantityType`、定質的なデータのタイプは`HKCategoryType`
- それに対応するデータは`HKQuantitySample`、`HKCategorySample`

のように表され、クラスの継承関係は以下のようになっています。

![](https://storage.googleapis.com/zenn-user-upload/y2f1bnjeghhrzqj2uv5qls29sbw0)

![](https://storage.googleapis.com/zenn-user-upload/9nhvqekf33m6uy6me2fubxtty4xk)

`HKObjectType`/`HKObject`がルートクラスであり、データを作成したアプリやデバイスの情報やメタデータなどを持っています。
そしてそのサブクラスに`HKSampleType`/`HKSample`があり、これはデータを記録した日時などを持っています。
そして最後に具体的なデータを表す`HKQuantityType`/`HKQuantitySample`、`HKCategoryType`/`HKCategorySample`などがあり、それぞれに適したデータを持っている、というような階層になっています。

# データの取得
データの作成/保存方法について見たので、次にデータの取得方法についても見ていきます。
HealthKitでヘルスデータを取得するには、クエリを使用します。
様々なクエリが用意されており、用途によって使い分けます。
ここでは、`HKStatisticsQuery`と`HKStatisticsCollectionQuery`、`HKSampleQuery`について見ていきます。

まずは`HKStatisticsQuery`について見ていきます。

## 1週間の歩行/走行距離の取得
名前の通り`HKQuantitySample`の統計値を計算するクエリです。(統計値を計算するとは、平均値や最大値、最小値などを求めるということです。)
ではどのような統計値を計算できるか確認するために、例として1週間の歩数を取得してみます。

```swift
let distanceType = HKObjectType.quantityType(forIdentifier: .stepCount)!
let startDate = DateComponents(year: 2021, month: 6, day: 15)
let endDate = DateComponents(year: 2021, month: 6, day: 22)
let predicate = HKQuery.predicateForSamples(
    withStart: Calendar.current.date(from: startDate)!,
    end: Calendar.current.date(from: endDate)!
)
let query = HKStatisticsQuery(
    quantityType: distanceType,
    quantitySamplePredicate: predicate,
    options: [.cumulativeSum]) { query, statistics, error in
        print(statistics!.sumQuantity()!) // 6902 count
    }
healthStore.execute(query)
```

:::message
今回取得した歩数はすべてAppleWatchで記録したものでした。
しかしiPhoneを持って歩いた場合iPhoneにも歩数を記録してしまいます。
ですので`HKSampleQuery`で先ほどと同じように歩数を合計した場合、同じデータも加算してしまいます。ただ`HKStatisticsQuery`は重複を排除してくれます
:::

## 日毎の歩数を取得
先程は1週間の歩数の合計を取得しましたが、
日毎の歩数パターンを知りたい場合、日毎の歩数を取得しなければなりません。
まずは1週間の毎日の歩数を取得してみます。前述のコードの日付を変えるだけです。
```swift
let startDate = DateComponents(year: 2021, month: 6, day: 15)
let endDate = DateComponents(year: 2021, month: 6, day: 16)
```
ただ1週間なら良いですが、1年間の毎日の歩数を取得するとなると300個以上の`HKStatisticsQuery`を生成することとなり、対応しきれません。
解決方法は`HKStatisticsCollectionQuery`使用することです。
これは指定した期間の統計値を問い合わせるクエリです。

```swift
var calendar = Calendar.current
let stepCountType = HKObjectType.quantityType(forIdentifier: .stepCount)!

let startDate = DateComponents(year: 2021, month: 4, day: 23, hour: 0, minute: 0, second: 0)
let endDate = DateComponents(year: 2021, month: 4, day: 29, hour: 23, minute: 59, second: 59)

let predicate = HKQuery.predicateForSamples(
    withStart: calendar.date(from: startDate),
    end: calendar.date(from: endDate)
)

let query = HKStatisticsCollectionQuery(
    quantityType: stepCountType,
    quantitySamplePredicate: predicate,
    options: .cumulativeSum,
    anchorDate: Calendar.current.date(from: anchorDate)!,
    intervalComponents: DateComponents(day: 1)
)

// クエリの実行結果のコールバックハンドラーです
query.initialResultsHandler = { query, collection, error in
    collection?.enumerateStatistics(
        from: calendar.date(from: startDate)!,
	to: calendar.date(from: endDate)!
    ) { statistics, stop in
           print(statistics.sumQuantity() ?? "nil")
    }
}
    
healthStore.execute(query)
```
こうすることで2021/4/23〜2021/4/29までの各日の歩数を取得できます。
また`HKStatisticsCollectionQuery`では`statisticsUpdateHandler`を用いることで歩数の変更を監視することができます。

# 参考URL
- https://developer.apple.com/videos/play/wwdc2020/10664/
- https://developer.apple.com/documentation/healthkit