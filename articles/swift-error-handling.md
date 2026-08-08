---
title: "Swiftのエラー処理を使い分ける（try? / try! / Result / typed throws）"
emoji: "🧯"
type: "tech"
topics: ["swift", "ios", "errorhandling", "swift6"]
published: false
---

`throws` / `try` / `try?` / `try!` / `Result`、そして Swift 6.2 の **typed throws**。似た道具が多くて迷いがちですが、**「今処理する／理由を捨てる／持ち運ぶ／型で縛る」**で選べば一意に決まります。

## 題材

正の整数をパースする関数で考えます。

```swift
enum ParseError: Error { case empty, notANumber, notPositive }

func parsePositive(_ s: String) throws -> Int {
    if s.isEmpty { throw ParseError.empty }
    guard let i = Int(s) else { throw ParseError.notANumber }
    if i <= 0 { throw ParseError.notPositive }
    return i
}
```

:::message
落とし穴：`Int("")` も `Int("abc")` も `nil` です。だから **空判定を先に**やり、そのあと `Int()` の失敗を `.notANumber` にしないと、空文字と「数値でない」を区別できません。「0以下」も `< 0` ではなく **`<= 0`**（`0` を含める）。
:::

## 1. throws + do/catch —「今ここで処理」

```swift
do {
    let n = try parsePositive(input)
    use(n)
} catch ParseError.notPositive {
    // 特定ケースだけ個別対応もできる
} catch {
    show(error)
}
```

`throws` は「失敗しうる」印。呼び出しは `try`、その場で `do/catch` して **今すぐ** 処理します。

## 2. try? —「理由は捨てて、成功値 or nil」

```swift
let n = try? parsePositive(input)              // Int?（失敗なら nil）
let v = (try? parsePositive(input)) ?? 0       // 既定値で代替
```

**なぜ失敗したかは要らない**／nil で代替が効く時に。エラーの中身は捨てられます。

## 3. try! —「成功が確実な時だけ」

```swift
let n = try! parsePositive("42")   // 失敗したらクラッシュ
```

リテラルなど **絶対に成功すると分かっている** 時のみ。ユーザー入力には使いません。

## 4. Result —「成功/失敗を値として持ち運ぶ」

```swift
let r: Result<Int, Error> = Result { try parsePositive(input) }

// 今すぐでなく、あとで／別の場所で処理
switch r {
case .success(let n): use(n)
case .failure(let e): show(e)
}

let n = try r.get()   // Result → throws に戻すこともできる
```

保存する・非同期コールバックで渡す・複数結果をまとめる、など **後で処理したい** 時に。エラーの中身が残ります。

## 5. typed throws（Swift 6.2）—「投げる型を明示」

```swift
func parsePositiveTyped(_ s: String) throws(ParseError) -> Int {
    if s.isEmpty { throw .empty }              // 型が確定するので .empty と書ける
    guard let i = Int(s) else { throw .notANumber }
    if i <= 0 { throw .notPositive }
    return i
}

do { _ = try parsePositiveTyped("abc") }
catch {
    // error は ParseError 型に推論される（as? 不要）
    if error == .notANumber { /* ... */ }
}
```

従来の `throws` は「何を投げるか」が型に出ず、呼び出し側は `any Error` を受け取ります。`throws(ParseError)` は **投げる型を型シグネチャに明示** するので、catch が具体型に推論され、`as?` キャストや `default` が減ります。

**使いどころ**：エラー集合が閉じている（列挙で尽きる）ライブラリ境界など。逆に **将来エラーが増えうる所は型を固定しない**（`any Error` のまま）方が柔軟です。

## もう一歩：実戦での使い方

### Result が活きる典型＝非同期コールバック

`async/await` が使える所は `try await` が第一候補ですが、**コールバックAPI**では `Result` が定番です。成功/失敗を1つの値で運べるので、引数がすっきりします。

```swift
func load(_ id: Int, completion: @escaping (Result<Data, Error>) -> Void) {
    // 非同期処理のあと completion(.success(...)) / completion(.failure(...))
}

load(1) { result in
    switch result {
    case .success(let data): use(data)
    case .failure(let error): show(error)
    }
}
```

### エラーは「握りつぶさず上へ」

中間層でむやみに `catch` せず、関数に `throws` を付けて **扱える所まで伝播**させるのが基本です。

```swift
func loadUser(_ id: Int) throws -> User {
    let data = try fetch(id)     // ここでは処理しない
    return try decode(data)      // throws のまま上位へ委ねる
}
```

握りつぶし（`try?` で握って nil を無視、空 `catch {}`）は、原因が消えてデバッグ困難になりがち。**「今ここで意味のある対処ができるか？」がNoなら伝播**が原則。

### rethrows と throws(Never)

```swift
// rethrows: 渡されたクロージャが throw する時だけ自分も throw（map/filter などが該当）
func transform<T>(_ x: T, _ f: (T) throws -> T) rethrows -> T { try f(x) }

// typed throws で「投げない」は throws(Never)＝非スロー関数と同義
func pure() throws(Never) -> Int { 42 }
```

Swift 6 では **`throws` ＝ `throws(any Error)`／非スロー ＝ `throws(Never)`** と、typed throws で綺麗に統一されています。

### 表示用メッセージは LocalizedError

ユーザーに見せる文言は `LocalizedError` で持たせると、`error.localizedDescription` がその文言になります。

```swift
extension ParseError: LocalizedError {
    var errorDescription: String? {
        switch self {
        case .empty:       "入力が空です"
        case .notANumber:  "数値を入力してください"
        case .notPositive: "正の数を入力してください"
        }
    }
}
```

## 使い分け（結論）

| やりたいこと | 選択 |
|---|---|
| 今ここで処理する | `do / catch` |
| 理由を捨てて成功値 or nil | `try?` |
| 成功が確実（失敗ならバグ） | `try!` |
| 成功/失敗を値として持ち運ぶ・後で処理 | `Result` |
| 投げる型を型で保証したい | `throws(SomeError)`（typed throws） |

合言葉：**今処理＝do/catch ／ 理由を捨てる＝try? ／ 持ち運ぶ＝Result ／ 型で縛る＝typed throws**。

## まとめ

- `throws` は失敗の印、`try` で呼ぶ。
- `try?` は理由を捨てて Optional に、`try!` は成功確実時のみ。
- `Result` は成功/失敗を値として持ち運ぶ（非同期コールバックで活きる）。
- typed throws は投げる型を明示して catch を具体型に。閉じたエラー集合で有効。
- 中間層は握りつぶさず `throws` で**伝播**、表示文言は `LocalizedError` で。
