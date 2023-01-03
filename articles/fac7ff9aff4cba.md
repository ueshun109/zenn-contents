---
title: "SwiftUIでハーフモーダルを表示してみる(iOS15~)"
emoji: "📰"
type: "tech"
topics:
  - "ios"
  - "swiftui"
published: true
published_at: "2021-09-11 16:03"
---

# 概要
iOS15からUIKitに[UISheetPresentationController](https://developer.apple.com/documentation/uikit/uisheetpresentationcontroller)というクラスが追加されますが、SwiftUIにはそのようなAPIは追加されませんでした。
今回は`UISheetPresentationController`を使用しハーフモーダルを実現し、それをSwiftUIのコードで使用できるようにしていきます。

コードの全容は[github](https://github.com/ueshun109/SwiftUIStylebook/blob/main/SwiftUIStylebook/UIComponents/HalfModalView.swift)にアップしているので良ければ参考にしてみてください。

# 完成イメージ
以下のようにボタンをタップした際、ハーフモーダルが表示されるような形を目指します。

![](https://storage.googleapis.com/zenn-user-upload/0863e1dfefa81b38f9dc8678.gif)

今回はハーフシート内のViewはSwiftUIで書くことができるようにしていきます。
つまりハーフシート部分のみ`UIViewControllerRepresentable`で表現していきます。

# 実装

## インターフェース定義
`UIViewControllerRepresentable`で実装したハーフシート上に、SwiftUIでViewを定義できるようにするため、Viewのextensionに以下を追加します。
```swift
extension View {
  func halfModal<Sheet: View>(
    isShow: Binding<Bool>,
    @ViewBuilder sheet: @escaping () -> Sheet,
    onEnd: @escaping () -> ()
  ) -> some View {
    return self
      .background(
        HalfModalSheetViewController(
	 sheet: sheet(),
	 isShow: isShow,
	 onClose: onEnd
        )
      )
  }
}
```
引数について少し説明します。
|引数名|内容|
|---|---|
|isShow|ハーフモーダルの表示状態を持ちます。`Binding`としているのは、`View`と`UIViewControllerRepresentable`の両方から表示状態を操作したいためです。|
|sheet|`View`でハーフモーダル上に表示するUIを描画するためのクロージャです。`@ViewBuilder`をつけているのは複数の`View`で構成されるのを想定するためです。|
|onEnd|ハーフモーダルが閉じたことを通知するイベントリスナーです。|

このインターフェースを定義することで、以下のように扱うことを想定しています。
```swift
@State var isShowHalfModal = false
// 省略...
ZStack {
  Button("Show half modal") {
    isShowHalfModal.toggle()
  }
}
.halfModal(isShow: $isShowHalfModal) {
  // ここにハーフモーダルシートに表示したいViewを定義する
  Text("Test")
} onEnd: {
  print("Dismiss half modal")
}
```

## ハーフモーダルシートを実装
後は`UISheetPresentationController`を使用してハーフモーダルを表示するだけです。

まず`UIViewControllerRepresentable`を使用してハーフモーダルのViewを実装します。
```swift
struct HalfModalSheetViewController<Sheet: View>: UIViewControllerRepresentable {
  var sheet: Sheet
  @Binding var isShow: Bool
  var onClose: () -> Void
  
  func makeUIViewController(context: Context) -> UIViewController {
    UIViewController()
  }
  
  func updateUIViewController(
    _ viewController: UIViewController,
    context: Context
  ) {
  if isShow {
    let sheetController = CustomHostingController(rootView: sheet)
    sheetController.presentationController!.delegate = context.coordinator
    viewController.present(sheetController, animated: true)
  } else {
    viewController.dismiss(animated: true) { onClose() }
  }
  
  func makeCoordinator() -> Coordinator {
    Coordinator(parent: self)
  }
  
  class Coordinator: NSObject, UISheetPresentationControllerDelegate {
    var parent: HalfModalSheetViewController

    init(parent: HalfModalSheetViewController) {
      self.parent = parent
    }

    func presentationControllerDidDismiss(_ presentationController: UIPresentationController) {
      parent.isShow = false
    }
  }
  
  class CustomHostingController<Content: View>: UIHostingController<Content> {
    override func viewDidLoad() {
      super.viewDidLoad()
      if let sheet = self.sheetPresentationController {
        sheet.detents = [.medium(),]
        sheet.prefersGrabberVisible = true
      }
    }
  }
}
```
`present`の引数に`CustomHostingController`という`UIHostingController`を継承したクラスのオブジェクトを渡しているのは、以下２つの理由のためです。
1. `present`の引数は`UIViewController`でなければならないため
2. SwiftUIで定義した`View`をモーダルシートに描画したいため

2について、`UIHostingController`のイニシャライザは以下のような定義であり、
```swift
init(rootView: Content)
```
`Content`は`View`の制約があり且つ`UIKit`の世界から`SwiftUI`の世界に`View`を渡すことができるためです。

## モーダルシートを表示
あとは以下の様な感じでモーダルシートを表示するだけです。

```swift
import SwiftUI

struct ContentView: View {
  @State var isShowHalfModal = false

  var body: some View {
    Button("show half modal") {
      isShowHalfModal.toggle()
    }
    .halfModal(isShow: $isShowHalfModal) {
      VStack {
        Text("Shown half modal!")
          .font(.title.bold())
          .foregroundColor(.black)
        Button("Close") {
          isShowHalfModal.toggle()
        }
      }
    } onEnd: {
      print("Dismiss half modal")
    }
  }
}
```

# まとめ
今まではサードパーティー製のライブラリを使うなどしてハーフモーダルを実現していましたが、iOS15からはサポートしてくれるので嬉しいです。

# 参考URL
- https://developer.apple.com/videos/play/wwdc2021/10063/