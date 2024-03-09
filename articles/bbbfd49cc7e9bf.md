---
title: "Privacy Manifests対応〜アプリ開発者がすべきこと〜"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [iOS, Apple]
published: true
---

# 概要

WWDC23 で Privacy Manifests の対応がアナウンスされて以降、いつかは対応しなければならないと考えていましたが、2024 年２月 29 日に、対応期日は 4 月末までと正式に決定しました[^1]。Privacy Manifests の対応をしていないアプリはリジェクト対象となってしまいます。

# Privacy Manifests とは

アプリ開発者はアプリに含まれる全てのコードに責任を持たなければなりません。
しかしアプリ開発者が、自身のアプリ（サードパーティ SDK を含む）がどのようなデータを収集・トラッキングしているかなど、すべてを把握するのが困難な場合があります。
そこで Apple は Privacy Manifests という新しい仕組みを提供しました。
アプリやサードパーティ SDK が Privacy Manifests 対応することで、プライバシーに関わる事柄に関して把握しやすくなります。

## 対応すべきこと

アプリ開発者がすべき大まかなことは以下の２つです。

- Privacy manifest files(`PrivacyInfo.xcprivacy`)を作成する
- サードパーティ SDK を Privacy Manifests 対応されたバージョンに上げる

# Privacy manifest files(`PrivacyInfo.xcprivacy`)を作成する

Xcode の、`File` -> `New` -> `File`から`App Privacy`を選択し、`PrivacyInfo.xcprivacy`を作成します。

:::message
このとき、Target を選択していないとビルドに失敗するみたいです。
:::

## Privacy manifest files の構成

Privacy manifest files はプロパティリスト形式（`.plist`）のファイルであり、以下４つのセクションで構成します。

1. Privacy Accessed API Types
1. Privacy Nutrition Label Types
1. Privacy Tracking Enabled
1. Privacy Tracking Domains

開発者はこれらのセクションを埋めていく必要があります。

### Privacy Accessed API Types

特定の API を使用している場合、利用している理由を明記する必要があります。
この特定の API を**required reason API**と呼んでいます。
以下のいずれかの API を利用している場合は、その API の「種類」と「理由」を入力していきます。

- File Timestamp
- System Boot Time
- Disk Space
- Active Keyboards
- User Defaults

「種類」と「理由」は[こちら](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api/)から該当するものを選択します。
例えば UserDefaults のみ利用している場合、私は以下の設定になりました。

- Privacy Accessed API Type: `User Defaults`
- Privacy Accessed API Reasons: `CA92.1: Access info from same app, per documentation`

![](/images/bbbfd49cc7e9bf/required_reason_api.png)

[^1]: ４月末までに対応必須なことは、`Requeired API`とサードパーティ SDK を Privacy Manifests 対応されたバージョンへ更新することです。

### Privacy Nutrition Label Types

Privacy Nutrition Label は、アプリがデータをどのように扱うかをユーザーが理解できるようにするために、iOS14.3 から導入されました。すでに iOS アプリをリリース済みの開発者は、App Store Connect で Privacy Nutrition Label を作成したことがあると思います。これと同じ内容を Privacy manifests files にも記載する必要があります。
既存の Privacy Nutrition Label と見比べて同様の内容になるように選択していきます。

![](/images/bbbfd49cc7e9bf/privacy_nutrition_labe.png)

参考 URL: https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_data_use_in_privacy_manifests

### Privacy Tracking Enabled

こちらは ATT フレームワークを使っている場合は`true`、そうでない場合は`false`を選択します

![](/images/bbbfd49cc7e9bf/privacy_tracking_enabled.png)

### Privacy Tracking Domains

アプリまたはサードパーティ SDK が接続する、トラッキングを行うドメインの一覧を記入していきます。ユーザーが ATT フレームワークを通じてトラッキングを許可していない場合、これらのドメインへのリクエストは失敗します。
トラッキングを行うドメインの一覧は、Instruments の Point of Interests を使って調べることができます。
具体的な使い方は、[WWDC23 の動画](https://developer.apple.com/videos/play/wwdc2023/10060/?time=399)で説明されています。

# サードパーティ SDK を Privacy Manifests 対応されたバージョンに上げる

使用してるサードパーティ SDK を Privacy Manifests に対応されたバージョンに上げる必要があります。
[Upcoming third-party SDK requirements](https://developer.apple.com/support/third-party-SDK-requirements/)に記載されている SDK は対応必須なため、いずれかの SDK を利用している場合は対応状況をウォッチする必要があります。

またその他の SDK については、「データを収集している」または「Required Reason API」を使用しているものに関しても対応状況を確認する必要があります。

# 参考 URL

- https://developer.apple.com/documentation/bundleresources/privacy_manifest_files
- https://creators-note.chatwork.com/entry/2024/02/26/175346
