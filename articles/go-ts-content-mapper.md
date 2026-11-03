---
title: "Goの関数をTypeScriptからimportする【Content Mapper実用編】"
emoji: "🐹"
type: "tech"
topics: ["typescript", "go", "webassembly", "wasm"]
published: true
---

[前回の記事](https://zenn.dev/p_jzd/articles/mermaid-machine-content-mapper)では、Content Mapper で Mermaid の図を型にしました。作ってみて楽しかったものの、実務で使うならコードで読めれば十分だな、というのが正直な感想でした。

今回はもう少し役に立ちそうなものとして、**Go のコードをフロントエンドの TypeScript からそのまま import** してみます。

```ts
import { Price, type Order } from "../shared/checkout/checkout.go";

const quote = Price(order); // 型は Go の宣言から、実行はブラウザの Wasm で
```

コード生成したファイルをコミットする必要はありません。Go を変えると、TypeScript 側がそのまま型エラーになります。

:::message
執筆時点の TypeScript は `7.1.0-beta` です。API や設定は正式版までに変わる可能性があります。
:::

動くものは [super-himetan/go-ts-shared-logic](https://github.com/super-himetan/go-ts-shared-logic) に置いています。

## 解決したいこと

金額計算や入力チェックのようなロジックは、フロントエンドとバックエンドの両方に必要になります。画面ではすぐに結果を出したいですし、サーバーではクライアントの値を信用せずに計算し直す必要があるからです。

バックエンドが Go だと、次のどちらかになりがちです。

- 同じロジックを TypeScript でも書く。端数処理などがいつかずれます。
- 毎回 API を呼ぶ。入力のたびに通信が発生します。

型だけなら OpenAPI などから生成できますが、ロジックは共有できません。そこで、Go のコードを1か所に書いて、3つの経路で使います。

```text
checkout.go ─┬ tsc / エディタ → Content Mapper → 型
             ├ Vite プラグイン → GOOS=js GOARCH=wasm go build → Wasm
             └ server/main.go → 普通の import
```

Content Mapper が作るのは型だけで、実行時には使われません。これは前回と同じ構成で、ブラウザで動くものは Vite プラグインが用意します。

## Go を書くだけで型が付く

題材はコーヒーの注文です。Go 側には特別な記法がなく、ただのパッケージです。

```go:shared/checkout/checkout.go
// Size is a cup size.
type Size string

const (
	SizeS Size = "S"
	SizeM Size = "M"
	SizeL Size = "L"
)

// Order is what a customer is about to buy.
type Order struct {
	Items  []LineItem `json:"items"`
	Coupon string     `json:"coupon,omitempty"`
}

// Price computes the amount to pay. The discount and the tax both round down.
func Price(order Order) (Quote, error) {
	// ...
}
```

mapper はこのファイルを、次のような仮想的な TypeScript として TypeScript に見せます。

```ts
/** Size is a cup size. */
export type Size = "S" | "M" | "L";

/** Order is what a customer is about to buy. */
export interface Order {
  items: LineItem[] | null;
  coupon?: string;
}

/**
 * Price computes the amount to pay. The discount and the tax both round down.
 * @throws {Error} The Go function returned an error.
 */
export declare function Price(order: Order): Quote;
```

Go の宣言がそのまま型になっていて、次のように変換されます。

| Go | TypeScript |
| --- | --- |
| `` `json:"items"` `` | プロパティ名 `items` |
| `` `json:",omitempty"` `` | 省略可能なプロパティ |
| `type Size string` と定数 | 文字列の union |
| `(Quote, error)` | `Quote` を返し、`error` は例外になる |
| `[]LineItem` | `LineItem[] \| null` |

最後の `null` が地味に大事です。Go の nil スライスは `encoding/json` で `null` になるので、TypeScript の型に `null` を残しています。忘れて触ると、ちゃんとコンパイルエラーになります。

```ts
Validate(order).length;
// Object is possibly 'null'.

const order: Order = { items: [{ drink: "tea", size: "M", quantity: 1 }] };
// Type '"tea"' is not assignable to type 'Drink'.
```

## Go を変えると TypeScript が落ちる

`Quote.Total` のタグを `json:"grandTotal"` に変えて型チェックすると、古い名前を使っている TypeScript がエラーになります。

```text
web/app.ts(51,36): error TS2339: Property 'total' does not exist on type 'Quote'.
```

逆に、JSON で正しく渡せない Go の書き方は、**Go のファイルの上で**エラーになります。`int64` は JavaScript の数値に収まらないことがありますし、`chan` は JSON にできません。

```go
type Result struct {
	Big int64 `json:"big"`
	//  ^^^^^ 'int64' can exceed what a JavaScript number holds exactly. Use int, or a string.
}

func Compute(events chan int) Result {
	//                  ^^^^^^^^ This type has no JSON form that TypeScript can describe.
}
```

エディタの機能も Go まで届きます。

- `Price` にホバーすると、Go のドキュメントコメントが表示されます。
- 定義ジャンプで `checkout.go` の `func Price` に移動します。
- `quote.subtotal` の定義ジャンプは、Go の `json:"subtotal"` の中の `subtotal` に移動します。
- `quote.subtotal` をリネームすると、Go のタグも一緒に変わります。

Go の関数名のリネームだけは、あえて止めています。TypeScript からは Go 側の呼び出し元を直せないためです。こういう制御を位置情報ごとに指定できるのが、Content Mapper の面白いところだと思います。

## ブラウザでは Go がそのまま動く

実行時は Vite プラグインが `.go` を受け取り、パッケージを Wasm にビルドします。`Price` の中身は Go のままで、JavaScript に変換しているわけではありません。

![数量とクーポンを入れると金額が計算される注文フォーム](/images/go-ts-content-mapper/checkout.png)

この画面の金額は、数量やクーポンを変えるたびに通信なしで計算されています。割引も税も切り捨てで、同じ Go のコードなので端数の処理がずれる余地がありません。

サーバーは注文を受けたら同じパッケージで計算し直し、画面に出ていた金額と違えば拒否します。

```go:server/main.go
quote, err := checkout.Price(req.Order)
// ...
if quote.Total != req.ExpectedTotal {
	writeJSON(w, http.StatusConflict, checkout.ErrorResponse{
		Message: fmt.Sprintf("金額が変わりました: 表示 %s / 実際 %s",
			checkout.FormatYen(req.ExpectedTotal), checkout.FormatYen(quote.Total)),
	})
	return
}
```

ソースは1つですが、ビルドはネイティブと Wasm で別です。そこで同じテストケースの JSON を Go のテストと Wasm のテストの両方で読み、同じ結果になることを確かめています。

## 実装

全体でも mapper が73行、Vite プラグインが106行です。Go ファイルの解析だけは、Go の標準パッケージ `go/parser` を使う小さなツールにしています（テストを除いて約600行）。

:::details Content Mapper 本体

mapper は TypeScript から呼ばれる外部プログラムです。実装言語は自由なので、Go の解析は Go に任せて、結果を [ts-content-mapper](https://github.com/sxzz/ts-content-mapper) で TypeScript に組み立てています。

```js:packages/go-mapper/server.mjs（抜粋）
runContentMapper({
  diagnosticSource: "go",
  transform(params, context) {
    const b = new MappedCodeBuilder(params.content, { positionEncoding: context.positionEncoding });
    const schema = extractSchema(params.content); // tools/goschema の結果
    writeTypes(schema, b);
    return { ...b.build(".ts"), diagnostics: schema.diagnostics };
  },
});
```

`appendVerbatim` は Go のソースの範囲をそのまま写す命令で、元の位置との対応が残ります。関数名ではリネームだけ外しています。

```js
const NO_RENAME = SpanMapFeature.All & ~SpanMapFeature.Rename;

b.append("export declare function ");
b.appendVerbatim(fn.span.start, fn.span.end, NO_RENAME);
b.append(`(${params}): ${fn.result};\n`);
```

位置は UTF-16 に変換してから渡しています。Go のオフセットはバイト単位なので、変換しないと日本語のコメントがあるだけでエラーの位置がずれます。

```json:tsconfig.json（抜粋）
{
  "contentMappers": [{ "package": "go-mapper", "extensions": [".go"] }]
}
```

```json:packages/go-mapper/package.json（抜粋）
{
  "typescript": {
    "contentMapper": {
      "exec": ["node", "server.mjs"],
      "compilerOptions": [],
      "dynamicConfig": false
    }
  }
}
```

:::

:::details JavaScript と Go の橋渡し

Vite プラグインは、パッケージの公開関数を並べた `main.go` を生成して Wasm にビルドします。

```go
func main() {
	bridge.Serve("__go_mapper_checkout_...", map[string]any{
		"Menu":     pkg.Menu,
		"Validate": pkg.Validate,
		"Price":    pkg.Price,
		// ...
	})
}
```

受け取った関数は `reflect` で呼びます。引数と戻り値はどちらも JSON です。これで型ごとのラッパーを生成せずに済みます。

```go:bridge/bridge.go（抜粋）
func Invoke(fn any, args []string) (result string, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("panic: %v", r)
		}
	}()

	v := reflect.ValueOf(fn)
	t := v.Type()
	if len(args) != t.NumIn() {
		return "", fmt.Errorf("expected %d argument(s), got %d", t.NumIn(), len(args))
	}
	in := make([]reflect.Value, len(args))
	for i, arg := range args {
		p := reflect.New(t.In(i))
		if err := json.Unmarshal([]byte(arg), p.Interface()); err != nil {
			return "", fmt.Errorf("argument %d: %w", i, err)
		}
		in[i] = p.Elem()
	}
	// 呼び出して、error を分けて、残りを JSON にして返す
}
```

`recover` しているのは、panic が Wasm のランタイムまで届くと、以降の呼び出しがすべて失敗するためです。

`checkout.go` の import は、Vite プラグインによって次のようなモジュールになります。

```js
import { lookup } from "\0go-mapper:package:...";
export const SizeS = "S";
export const Price = lookup("Price");
```

Wasm の読み込みはパッケージごとに1回だけです。同じパッケージのファイルは、ネイティブと同じように1つの Go ランタイムと状態を共有します。

:::

## 実用編と言いつつの注意点

良いことばかりではないので、試して分かった制限も書いておきます。

- **Wasm が大きい**: 通常の Go で作るとランタイムごと入るので、このデモで約 4.5MB あります。読み込むまで動かないので、読み込み中の表示も必要です。
- **解析はファイル単位**: Content Mapper は1ファイルずつ変換するので、TypeScript に公開する型は同じファイルに置く必要があります。別のファイルに書かれた `MarshalJSON` も検出できません。
- **数値は `number` にまとまる**: `int` の引数に `1.5` を渡してもコンパイルは通り、実行時に例外になります。
- **union は閉じた型ではない**: Go では `Size("XL")` も `""` も作れますが、型は定数の一覧です。バックエンドから来た値を信用するための型ではありません。
- **mapper のエラーが先に止まる**: `tsc` は mapper のエラーがあると、その先の型エラーを報告しません。

それでも、小さなルールの塊を共有するだけなら、二重に実装するより安全で、毎回 API を呼ぶより速いと感じました。

## まとめ

Content Mapper の面白いところは、変換処理が TypeScript の外にあることです。今回は Go のパーサーをそのまま使えたので、型の変換を自分で頑張る必要があまりありませんでした。

Go との変換には、他にもいろいろな遊び方がありそうです。特に試してみたいのが、gRPC の `.proto` や OpenAPI の定義のような DSL です。こうしたものは今でもコード生成で型を作っていることが多いと思いますが、その変換処理を Content Mapper に任せると、次のように書ける可能性があります。

```ts
// どちらもまだ作っていない想像の例です
import type { GetUserRequest } from "./user.proto";
import type { paths } from "./openapi.yaml";
```

生成物の置き場所や再生成の手順を考えなくてよくなり、定義の間違いは `.proto` や `.yaml` の上で赤線になり、定義ジャンプで DSL まで戻れます。既存のパーサーや生成ツールをそのまま mapper から呼べるのも相性が良さそうです。

コード生成を置き換えるとまでは言えませんが、スキーマと実装の間の継ぎ目をもっと減らし、よりシームレスな実装ができるようになると考えられます。
大AI時代だからこそ、こんな想像力が刺激されるようなAPIが生えると色々手早く試せてワクワクしますね！
