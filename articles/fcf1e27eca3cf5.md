---
title: "副作用を持つクラスの良さげな書き方の提案(swift)"
emoji: "💊"
type: "tech"
topics:
  - "ios"
  - "swift"
published: true
published_at: "2021-07-19 22:05"
---

# 概要
[isowords](https://github.com/pointfreeco/isowords)というPoint-Freeさんが作成しているアプリがOSSとして公開されているのですが、
そのコードにて副作用を持つクラスの書き方が良いなと感じたので紹介したいと思います。

# TL;DR
```swift
Main(httpClient: .live)
```
このように副作用を持つクラスを`.live`のような省略した形式でDIすることができます。

# 副作用とは
まず副作用についてざっくり説明します。(私も正しく理解できていない可能性があるので、間違いがある場合はご指摘ください。)
プログラミングにおける式の評価による作用には以下の２つがあります。
- 主たる作用
- それ以外の副作用(side effect)

主たる作用とは、以下のようなコードのように引数を受け取り値を返すことです。
```swift
/// 引数を2乗した結果を返す
func double(x: Int) -> Int {
  x * x
}
```
それ以外の作用とは、
- 状態の変更
- APIリクエストなどのI/O実行

などが挙げられます。
状態の変更とは、以下のように関数外のスコープの変数の値を書き換えてしまっているということです。
```swift
var total: Int = 0
func add(x: Int, y: Int) -> Int {
  let result = x + y
  // ある足し算の結果を返すという主たる作用以外に、変数の値を変更している。
  total = result
  return result
}
print(total) // 0
_ = add(x: 5, y: 3)
// addメソッドの実行前と結果がかわってしまっている
print(total) // 8
```
またAPIリクエストなどのI/O実行は、同じ内容でAPIリクエストを投げても常に同じ結果が返ってくるとは限りません。
つまり**副作用のあるコード**とは、
- 同じ条件を与えても必ず同じ結果になるとは限らない
- 他の機能の結果に影響を与えてしまう

言い換えると**副作用のないコード**とは、
- 同じ条件を与えれば必ず同じ結果になる
- 他のいかなる機能の結果にも影響を与えない

ということです。
このような性質を**参照透過性**と言います。
つまり副作用のないコードは参照透過性という性質を持っているということになります。

# 副作用を持つAPIClientクラスを実装してみる
今回は副作用を持つコードの特徴のひとつである、「同じ条件を与えても必ず同じ結果になるとは限らない」について考えていきます。
「同じ条件を与えても必ず同じ結果になるとは限らない」を再現するクラスを実装するために、APIクライアントを想定したクラスを実装してみます。

```swift
import Combine

struct  HttpClient {
  var random: Int { Int.random(in: 0..<10) }
  func get() -> AnyPublisher<Int, HttpError> {
    Future { completion in
      completion(.success(random))
    }
    .eraseToAnyPublisher()
  }
}

final class Main {
  let client = HttpClient()
  func main() {
    client.get()
      .sink { completion in
        switch completion {
        case let .failure(error):
          print(error)
        case .finished:
          print("finished")
        }
      } receiveValue: { value in
        print(value) // 実行するたびに結果が異なる
      }
  }
}
Main().main()
```
上記コードの`client.get()`を実行するたびに結果が異なっているので、
HttpClientは副作用を持つクラスになります。

この状態でも動作するコードですが、副作用を持つコードに付きまとうよくある問題は、テストや動作確認時です。
副作用を持つクラスに依存しているクラスの動作確認を行う際、上記コードの状態だと毎回結果が変わってしまうので、特定の状態のときの動作確認を行うことができません（よくあるのがエラー発生時の動作確認）。
またテストについても、結果が同じではないので正しいテスト結果を得ることができません。

この問題を解決するためにswiftではプロトコルが用いられることが多いと思います。

# protocolで実装する
前述のコードにて`Main`クラスは`HttpClient`という具象クラスに依存しており、モックへの差し替えが容易ではない状態になっていました。
そこで`HttpClient`を`protocol`にし、`Main`クラスは抽象に依存するように変更します。
また`HttpClient`に適合したインスタンスを`Main`クラスにDIできるようにも変更します。
```swift
import Combine
import Foundation

struct HttpError: Error {
  var message = "failed"
}

protocol HttpClient {
  func get() -> AnyPublisher<Int, HttpError>
}

/// 常に3を返却する
struct HttpClientMock: HttpClient {
  func get() -> AnyPublisher<Int, HttpError> {
    Future { completion in
      completion(.success(3))
    }
    .eraseToAnyPublisher()
  }
}

/// 常にエラーを返却する
struct HttpClientFailure: HttpClient {
  func get() -> AnyPublisher<Int, HttpError> {
    Future { completion in
      let error = HttpError()
      completion(.failure(error))
    }
    .eraseToAnyPublisher()
  }
}

/// production用のクラス
struct HttpClientImpl: HttpClient {
  var random: Int { Int.random(in: 0..<10) }
  func get() -> AnyPublisher<Int, HttpError> {
    Future { completion in
      completion(.success(random))
    }
    .eraseToAnyPublisher()
  }
}

final class Main {
  private let client: HttpClient
  init(httpClient: HttpClient) {
    self.client = httpClient
  }

  func main() {
    client.get()
      .sink { completion in
        switch completion {
        case let .failure(error):
          print(error.message)
        case .finished:
          print("finished")
        }
      } receiveValue: { value in
        print(value)
      }
  }
}
Main(httpClient: HttpClientImpl()).main()
Main(httpClient: HttpClientMock()).main()
Main(httpClient: HttpClientFailure()).main()
```
こうすることで`Main`クラスをテストする際の依存クラスの動作をコントロールできるようになり、テストが書きやすくなります。
ただprotocolを使って実装することについて個人的には
- protocolを用意するのが面倒
- protocolを具象クラスに適合していくのが面倒

と感じました。

そして上記のような実装よりも、以下で紹介するpointfreeさんのコードの書き方の方が良いなと感じました。

# staticな定数で実装する
pointfreeさんのコードでは、副作用を持つクラスのインスタンスを`protocol`を使わずに`static`な定数やメソッドで提供しています。以下のような感じです。
```swift
struct HttpClient {
  // 1. クロージャーでインターフェースのようにプロパティを定義
  var get: () -> AnyPublisher<Int, HttpError>
}

extension HttpClient {
  // 本番用のインスタンスを提供する定数
  static let live = Self(
    get: {
      var random: Int { Int.random(in: 0..<10) }
      return Future<Int, HttpError> { completion in
        completion(.success(random))
      }
      .eraseToAnyPublisher()
    }
  )

  // モックオブジェクトを提供する定数
  static let mock = Self(
    get: {
      Future<Int, HttpError> { completion in
        completion(.success(3))
      }
      .eraseToAnyPublisher()
    }
  )

  // 常にエラーを返すオブジェクトを提供する定数
  static let failed = Self(
    get: {
      Future<Int, HttpError> { completion in
        completion(.failure(HttpError()))
      }
      .eraseToAnyPublisher()
    }
  )
}
・・・

// 2. .liveや.mockのような型省略でインスタンスを生成できる
Main(httpClient: .live).main()
Main(httpClient: .mock).main()
Main(httpClient: .failed).main()
```
上記のコードでは、
- protocolに適合した構造体を作らなくて良いので、面倒くささが減る
- `Main`クラスのイニシャライザに型省略した記法(`.live`や`.mock`)で渡せるので、見やすくなりどのようなインスタンスを渡しているのか(本番用なのかモックようなのか)わかりやすくなる（この点が私にとって最も良いなと思いました）

のようなメリットを感じました。

# 参考URL
- https://github.com/pointfreeco/isowords