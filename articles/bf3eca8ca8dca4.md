---
title: "[SwiftUI] ForEachから学ぶイニシャライザ"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [iOS, SwiftUI]
published: false
---

# 概要

SwiftUI には、識別可能なデータ群をループし View を計算する[ForEach](https://developer.apple.com/documentation/swiftui/foreach)という API が存在し、`ForEach`には以下のように大きく２つの I/F が存在します。

- [init(\_:content:)](<https://developer.apple.com/documentation/swiftui/foreach/init(_:content:)-2n9gc>)
- [init(\_:id:content:)](<https://developer.apple.com/documentation/swiftui/foreach/init(_:id:content:)-4hb52>)

そしてこの２つの違いは以下になります。

- ジェネリックパラメータの Data が[Identifiable](https://developer.apple.com/documentation/swift/identifiable)に準拠しているか
- 準拠していない場合は、`KeyPath<Data.Element, ID>`を引数に取るかどうか

本記事では、どのようにして `KeyPath` の有無に左右されない実装になっているのか考察していきます。

# 事前情報

考察するにあたって以下のような横スクロール可能な View コンポーネントを例に見ていきます。

![](/images/bf3eca8ca8dca4/demo.gif)

# パラメータを考える

まず View 側でどのようなパラメータを扱うかを決定します。

```swift
struct HorizontalScrollView<Data, ID, Content>: View
where
  Data: RandomAccessCollection, // ForEachで取り扱えるようにするために、RandomAccessCollectionに準拠しておく
  ID: Hashable, // ForEachのKeyPathに渡せるようにHashableに準拠しておく
  Content: View
{
  private let data: Data
  private let id: KeyPath<Data.Element, ID>
  private let content: (Data.Element) -> Content

  var body: some View {
    // あとで実装します
  }
}
```

基本的には、[ForEach](https://developer.apple.com/documentation/swiftui/foreach)を使用するときに必要なデータだけを渡しておきます。

# Identifiable に準拠しないイニシャライザ

次は[init(\_:id:content:)](<https://developer.apple.com/documentation/swiftui/foreach/init(_:id:content:)-4hb52>)相当の、データが`Identifiable`に準拠せず、`KeyPath<Data.Element, ID>`を引数に取るイニシャライザを実装します。

```swift
init(
  data: Data,
  id: KeyPath<Data.Element, ID>,
  @ViewBuilder content: @escaping (Data.Element) -> Content
) {
  self.data = data
  self.id = id
  self.content = content
}
```

# Identifiable に準拠するイニシャライザ

次は[init(\_:content:)](<https://developer.apple.com/documentation/swiftui/foreach/init(_:content:)-2n9gc>)相当の、データが Identifiable に準拠するイニシャライザを実装します。
struct に convinience initializer を追加したいので、extension を生やします。

```swift
extension HorizontalScrollView {
  init(
    data: Data,
    @ViewBuilder content: @escaping (Data.Element) -> Content
  ) where Data.Element: Identifiable, Data.Element.ID == ID {
    self.data = data
    self.id = \.id
    self.content = content
  }
}
```

ここでの注目ポイントは、`Data.Element: Identifiable`の制約と、`Data.Element.ID == ID`の同型制約をつけている点です。
まず`Data.Element`に`Identifiable`の制約をつけることで、`ID`という関連型にアクセスできるようになります。
そしてその`ID`は`Hashable`に適合しているので、HorizontalScrollView の`ID`パラメータの制約にも適合します。
しかし以下のエラーが発生し、`KeyPath<Data.Element, ID>`のパラメータを初期化することができません。

```
Cannot assign value of type 'KeyPath<Data.Element, Data.Element.ID>' to type 'KeyPath<Data.Element, ID>'
```

そこで`Data.Element.ID == ID`の同型制約をつけることで、`KeyPath<Data.Element, ID>`を満たすことができるようになります。

# 横スクロールを実装

あとは、`KeyPath<Data.Element, ID>`の ForEach のイニシャライザを使って横スクロールを実装するだけです。

```swift
var body: some View {
  ScrollView(.horizontal, showsIndicators: false) {
    LazyHStack(spacing: 16) {
      ForEach(data, id: id) { content($0) }
    }
    .padding(.horizontal, 16)
  }
}
```

# まとめ

`Identifiable`に準拠しない・するイニシャライザをそれぞれ用意したい場合、`KeyPath<Data.Element, ID>`を用いた実装をベースとし、`Identifiable`に適合したデータから`KeyPath<Data.Element, ID>`を生成してやれば、両方を満たすことができるようになりました。

今回実装した内容は、[GitHub](https://github.com/ueshun109/SwiftUIStylebook/blob/main/SwiftUIStylebook/UIComponents/HorizontalScrollView.swift) にアップロードしています。
