---
title: "Swiftのclassを整理する（継承・override・多態・final・init）"
emoji: "🐶"
type: "tech"
topics: ["swift", "ios", "oop", "class"]
published: true
---

Swift は「まず struct、必要な時だけ class」。
その **class ならではの仕組み**——継承 / `override` / **多態（動的ディスパッチ）** / `final` / `init`（designated・convenience）——を最小例で整理します。

## 題材

```swift
open class Animal {
    let name: String
    init(name: String) { self.name = name }   // designated init
    func sound() -> String { "..." }           // override 対象
}

final class Dog: Animal {
    convenience init() { self.init(name: "Dog") }   // convenience → designated へ委譲
    override func sound() -> String { "Wan" }       // override
}
```

## 継承・override・多態

```swift
let a: Animal = Dog(name: "d")   // 変数の「型」は Animal、「実体」は Dog
a.sound()                        // → "Wan"（Dog の実装が動く）
```

ポイントは **「変数の型」ではなく「実体の型」でメソッドが選ばれる** こと。
これが多態（polymorphism）で、仕組みは **動的ディスパッチ**（実行時に実体の型で解決）です。

| ディスパッチ | いつ | 特徴 |
|---|---|---|
| 動的（dynamic） | `override` 可能なメソッド（class） | 実行時に実体で解決。多態が効く。わずかにコスト |
| 静的（static） | `final` / `static` / struct のメソッド | コンパイル時に確定。インライン化など最適化しやすい |

## final —「これ以上いじらせない」

`final` はクラス/メソッドの **継承・`override` を禁止** します。
うれしさは2つ：

- **意図の明示・安全**：想定外のサブクラス化で壊されない。
- **最適化**：`override` されない＝**静的ディスパッチ**にでき、インライン化などが効く。

継承させる前提のないクラスは `final` を既定にする、という設計指針もよく使われます。

## designated と convenience init

| 種類 | 役割 | ルール |
|---|---|---|
| designated（指定） | 全 stored property を初期化する**主**イニシャライザ | スーパークラスの designated を呼ぶ（階層を**上へ**） |
| convenience（簡易） | 引数を補って designated を呼ぶ**補助** | `self.init(...)` で**同じクラスの**別 init を呼ぶ（**横へ**） |

```swift
convenience init() { self.init(name: "Dog") }   // 横：同クラスの designated へ
init(name: String) { self.name = name }          // 主：ここで stored property を確定
```

:::message
**init 継承のルール**：サブクラスが**独自の designated init を持たない**と、スーパークラスの designated init を**継承**します。だから `Dog` は `Dog(name:)` でも作れます（Dog は `override` と `convenience` しか足していない）。
:::

## もう一歩

### override で親の実装を使う（super）

```swift
final class LoudDog: Dog {
    override func sound() -> String { super.sound() + "!!" }  // 親の "Wan" を土台に "Wan!!"
}
```

override 内で `super.method()` を呼べば、親の実装を**完全に置き換えず“足す”**拡張ができます。

### required init と失敗可能 init（init?）

```swift
class Base {
    required init() {}                              // サブクラスにも実装を強制
    init?(_ ok: Bool) { if !ok { return nil } }     // 失敗時に nil を返す
}
```

- `required`：全サブクラスがその init を（継承 or 実装で）**必ず持つ**ことを強制。
- `init?`（failable）：不正な引数など **初期化に失敗しうる**時に `nil` を返せる。

### static と class（型メソッドのディスパッチ）

```swift
class Service {
    static func a() {}   // 型メソッド。override 不可（静的ディスパッチ）
    class func b() {}    // 型メソッドだが override 可（動的ディスパッチ）
}
```

インスタンスメソッドの `final` vs 通常と同じ「静的 vs 動的」の対比が、型メソッドにも `static`／`class` として現れます。

## struct と class の使い分け（前提）

| | struct（値型） | class（参照型） |
|---|---|---|
| コピー | 代入で独立コピー | 参照を共有（同一インスタンス） |
| 継承 | なし（プロトコルで合成） | あり（単一継承）＋多態 |
| 既定 | **まず struct** | 同一性・共有・継承が要る時 |

多態・参照共有・Obj-C 連携が要る時に class、それ以外は struct が基本です。

## 検証

最小の Swift Package でテスト済み。

```swift
func testPolymorphism() {
    let a: Animal = Dog(name: "d")
    XCTAssertEqual(a.sound(), "Wan")                  // 動的ディスパッチで Dog の実装
}
func testConvenienceInit() {
    XCTAssertEqual(Dog().name, "Dog")                 // convenience → designated 委譲
}
func testInheritedDesignatedInit() {
    XCTAssertEqual(Dog(name: "Pochi").name, "Pochi")  // designated init を継承
}
```

## まとめ

- 多態＝**実体の型でメソッドが選ばれる**（動的ディスパッチ）。変数の型ではない。
- `final`＝継承/override 禁止。安全＋静的ディスパッチで最適化。
- designated＝主（stored 初期化・上へ）／convenience＝補助（`self.init`・横へ）。
- 独自 designated を足さなければスーパークラスの init を継承。
- override 内は `super` で親実装を再利用。型メソッドは `static`＝静的／`class`＝動的。
