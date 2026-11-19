# cf2router 設計書

CloudFront Functions 上で動く最軽量ルーターの設計。2026-11-19 時点の AWS 公式ドキュメントと [yusukebe/pico](https://github.com/yusukebe/pico) をもとにする。

## 決定事項

| 項目 | 決定 |
| --- | --- |
| ルート定義の API | pico 風の `app.get(path, handler)` チェーン |
| パスの変換 | ビルド時に正規表現とキー配列へ変換する |
| ランタイムの対象 | `cloudfront-js-2.0`(viewer-request) |

エッジで動くルーター本体は試作で minify 後 455 B。10 KB のコード上限に対して約 4.5% に収まる。

## 1. 前提: CloudFront Functions の API と制限

最新ランタイムは `cloudfront-js-2.0`。API の有効値は `cloudfront-js-1.0 | cloudfront-js-2.0` の 2 つのみ。言語は ES5.1 準拠に ES6〜12 の一部を足したもので、**ドキュメントに記載のない機能は使えない前提で設計する**。

### 1.1 制限(クォータ)

| 項目 | 値 |
| --- | --- |
| コードサイズ | 10 KB(引き上げ不可) |
| メモリ | 2 MB |
| 実行時間 | サブミリ秒。数値は非公開で、上限に対する割合「コンピュート使用率(0〜100)」で計測 |
| 関数数 | 1 アカウント 100(引き上げ可) |
| URI + クエリ文字列 | 8,192 文字未満 |
| ログ | 10 KB で切り捨て。`console.log` は引数 1 つのみ |
| KeyValueStore | 関数あたり 1 つ、キー 512 B、値 1 KB、ストア 5 MB |

イベントは viewer-request と viewer-response(ほかに mTLS 用の Connection Function)。ネットワーク、リクエストボディ、ファイルシステム、環境変数、タイマー、`eval` / `Function` は使えない。`Date` は実行中ずっと開始時刻を返す。

### 1.2 言語機能(runtime 2.0)

- **使える**: `let` / `const`、アロー関数、rest 引数、テンプレートリテラル、`**`、数値セパレータ、`async` / `await`、正規表現の名前付きキャプチャ、常時 strict mode
- **制約あり**: `await` は `async` 関数内のみ。**async な引数とクロージャは非対応**。`Promise.all` や `.then` はメモリを食うため `await` を順に書くことが推奨
- **記載なし(使わない)**: class、分割代入、スプレッド、`for...of`、`?.`、`??`、引数なし catch、ジェネレータ、`Map` / `Set`、`Proxy` / `Reflect`、`URL` / `URLSearchParams` / `URLPattern`

### 1.3 組み込みオブジェクトとモジュール

- String は `startsWith` / `endsWith` / `includes` / `replaceAll` / `padStart` など。Array は ES5 一式と `find` / `findIndex` / `includes` / `fill` まで(`flat` / `at` は記載なし)
- Object は `assign` / `entries` / `values` / `freeze` など。Promise、TypedArray、DataView、TextEncoder / TextDecoder、`atob` / `btoa`、`Symbol.for` もある
- `require()` で読むモジュール: `Buffer`、`querystring`、`crypto`(md5 / sha1 / sha256 の hash と HMAC のみ)

### 1.4 CloudFront 固有のヘルパー(`import cf from 'cloudfront'`)

| API | 用途 | 追加時期 |
| --- | --- | --- |
| `cf.updateRequestOrigin` / `cf.selectRequestOriginById` / `cf.createRequestOriginGroup` | viewer-request でオリジンを切り替える。`hostHeader` / `sni` / `allowedCertificateNames` も指定可 | 2024-11 〜 2025-11 |
| `cf.kvs().get(key, {format})` / `exists` / `meta` | KVS 読み取り。Promise を返し、存在しないキーの `get` は例外 | 2023-11 |
| `request.rawQueryString()` | 未加工のクエリ文字列(import 不要) | 2025-11 |
| `cf.edgeLocation` | エッジの IATA コード、サーバー IP、想定 REC | 2025-11 |
| `cf.logCustomData(str)` | アクセスログへ 800 B まで記録 | 2025-11 |
| `cf.cwt.validateToken` / `generateToken` | CBOR Web Token(HS256)の検証と生成 | 2025-11 |
| `cf.distributionTenant` | SaaS Manager のテナント ID とパラメータ | 2025-05 |

### 1.5 イベント構造でルーターに効く点

- `event.request.method` は読み取り専用。`uri` はクエリを含まない相対パス。`uri` を書き換えてもキャッシュ動作やオリジンは変わらない
- `querystring` / `headers` / `cookies` は解析済みのオブジェクトで、重複は `multiValue` に入る。ヘッダー名は小文字
- viewer-request で `{statusCode, statusDescription?, headers?, cookies?, body?}` を返すと、オリジンに行かずその場で応答する。`request` を返すとそのままオリジンへ流れる

## 2. pico から引き継ぐもの、捨てるもの

pico(`@picojs/pico` 0.3.4)の本体は約 50 行、minify 後 405 B(gzip 309 B)。仕組みは 3 点。

1. `routes` 配列に `{m: メソッド, p: new URLPattern({pathname}), h: ハンドラ}` を積む
2. `fetch` で配列を先頭から舐め、メソッドが違えば `continue`、`p.exec(url)` が当たったら即 return。一致なしは `undefined`
3. `Proxy` で `app.get` / `app.post` / 任意メソッド名を `on(method, ...)` に転送する

| 要素 | 扱い | 理由 |
| --- | --- | --- |
| 線形探索・先勝ち・メソッドを先に比較 | 引き継ぐ | コードが最小。ルート数は 10 KB 上限で自然に数十までになり、ツリー構造の利点が出ない |
| `app.get()` チェーンの書き心地 | 引き継ぐ | 決定事項。ただし Proxy ではなく、メソッド名の配列からループで生成する |
| `ALL` メソッドと「一致なし」の扱い | 引き継ぐ(`all()` / `'*'`) | 一致なしは `request` をそのまま返してオリジンへ流す。CloudFront での自然な既定動作 |
| プロパティ名を省いた短い内部表現 | 引き継ぐ | `[method, regex, handler, keys]` の配列にする |
| `URLPattern` | 捨てる | CloudFront Functions に存在しない。ビルド時に正規表現へ変換 |
| `Proxy` | 捨てる | runtime 2.0 の対応表にない |
| `Request` / `Response` / `env` / `executionContext` | 捨てる | CloudFront はプレーンな `event` オブジェクトを渡し、応答もオブジェクトで返す |

## 3. ルーター設計

### 3.1 設計原則

1. **ドキュメントにある構文だけを使う**: ルーター本体は `var` / `function` / 正規表現 / 配列 / オブジェクトのみで書く
2. **パス解析はエッジに持ち込まない**: `'/users/:id'` の正規表現化はビルド時に済ませ、エッジではループと `exec` だけを回す
3. **async を強制しない**: ルーターは同期。KVS を使うハンドラだけを名前付きの `async function` で宣言する(async クロージャ非対応のため `app.get('/x', async () => ...)` は書けない)
4. **未一致はパススルー**: どのルートにも一致しなければ `event.request` を返し、オリジンへ流す

### 3.2 利用者が書くコード

```javascript
import cf from 'cloudfront'
import { Router } from 'cf2router'

function home(e) { return { statusCode: 200, body: 'hello' } }
function api(e, p) { cf.selectRequestOriginById('api'); return e.request }
async function short(e, p) {           // KVS を使うものだけ async
  try {
    var to = await cf.kvs().get('r:' + p.code)
    return { statusCode: 302, headers: { location: { value: to } } }
  } catch (_) { return { statusCode: 404 } }
}

const app = Router()
app.get('/', home)
   .all('/api/*', api)
   .get('/s/:code', short)

async function handler(event) {        // async ハンドラが 1 つもなければ普通の function でよい
  return app.fetch(event)
}
```

ハンドラの引数は `(event, params)`。戻り値は CloudFront の request か response オブジェクトをそのまま返す。

### 3.3 パスの記法とビルド時変換

ビルド時に `app.<method>('<path>', ...)` の第 1 引数の文字列リテラルを `[正規表現, キー配列]` に置き換える。

| 記法 | 変換後 | params |
| --- | --- | --- |
| `/users/:id` | `[/^\/users\/([^/]+)$/, ['id']]` | `{id}` |
| `/posts/:y/:slug` | `[/^\/posts\/([^/]+)\/([^/]+)$/, ['y', 'slug']]` | `{y, slug}` |
| `/assets/*`(末尾のみ) | `[/^\/assets(?:\/(.*))?$/, ['*']]` | `{'*'}` |
| `/a.b` などの記号 | 正規表現のメタ文字はエスケープ | — |

- パラメータは名前付きキャプチャの `.groups` を使わず、位置キャプチャとキー配列で組み立てる。名前付きキャプチャ自体は対応と書かれているが、`.groups` の挙動までは明記がないため
- params の値はデコードしない生の文字列
- パスは文字列リテラルに限る。変数や式で渡されたパスはビルドエラーにする(エッジに変換器を持ち込まないため)

### 3.4 エッジで動くルーター本体(試作、minify 後 455 B)

```javascript
export function Router() {
  var t = [], a = {
    on: function (m, p, h) { t.push([m, p[0], h, p[1]]); return a },
    fetch: function (e) {
      var q = e.request, i = 0, r, m, p, j
      for (; i < t.length; i++) {
        r = t[i]
        if ((r[0] === q.method || r[0] === '*') && (m = r[1].exec(q.uri))) {
          for (p = {}, j = 0; j < r[3].length; j++) p[r[3][j]] = m[j + 1]
          return r[2](e, p)
        }
      }
      return q
    }
  }
  'GET POST PUT DELETE PATCH HEAD OPTIONS *'.split(' ').forEach(function (m) {
    a[m === '*' ? 'all' : m.toLowerCase()] = function (p, h) { return a.on(m, p, h) }
  })
  return a
}
```

`get` / `post` / `put` / `delete` / `patch` / `head` / `options` / `all` と、任意メソッド用の `on(method, path, handler)` を持つ。各メソッドは `app` を返すのでチェーンできる。

参考: `app` オブジェクトを持たず、ビルド時にルート表と照合ループだけを生成する方式なら 194 B まで縮む。DX を優先して `app.get` を採用したが、将来の最適化候補として残す。

### 3.5 ビルドと検証

1. **変換**: esbuild プラグインが `app.<method>('<path>', ...)` のパス文字列を `[正規表現, キー配列]` に置き換える
2. **束ねる**: `--bundle --format=esm --external:cloudfront --target=es2017`。`es2017` にしないと、esbuild が未記載の `catch {`(引数なし catch)を出力することを試作で確認した
3. **後処理**: 識別子の minify はしない(トップレベルの `handler` 名を残すため)。末尾の `export {...}` を削除する。`import cf from 'cloudfront'` は先頭行に残る
4. **方言チェック**: acorn で出力を構文解析し、class・分割代入・スプレッド・`for...of`・`?.`・`??`・引数なし catch・async クロージャ・`Proxy` / `Map` / `URL` などを検出したら失敗させる。10 KB 超過と `handler` 不在も検出する。試作では正常系が OK、`async () =>` と分割代入を含むコードが NG になることを確認済み
5. **単体テスト**: マッチングは Node でテストする。実ランタイムでの確認は `aws cloudfront test-function`(DEVELOPMENT ステージ)で行い、コンピュート使用率もそこで計測する

### 3.6 入れないもの(将来の拡張候補)

- 応答ヘルパー(`redirect()` / `text()`): 別モジュールにして使うときだけ束ねる
- 全ルートを 1 本の正規表現にまとめる方式(Hono の RegExpRouter の発想): 数十ルートでは差が出ない可能性が高いので計測してから
- 静的パスのオブジェクト引き、ホスト名での振り分け、KVS に置いたリダイレクト表

## 4. 未確認事項

ドキュメントだけでは決まらず、実機の TestFunction で確かめる点。設計はどれが「使えない」結果でも成り立つようにしてある。

- [ ] `async function handler` が同期の戻り値だけを返す場合のコンピュート使用率とメモリ(同期 handler との比較)
- [ ] 同期 handler から Promise を返しても受け付けられるか
- [ ] トップレベルで作ったオブジェクトが呼び出し間で再利用されるか
- [ ] class、分割代入、`Map` など未記載機能の実際の可否(使えても公式に保証されないので、方言チェックでは引き続き禁止する想定)

## 情報源(すべて 2026-11-19 に参照)

- [JavaScript runtime 2.0 features](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-javascript-runtime-20.html)
- [JavaScript runtime features(1.0 と 2.0 の差分)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-javascript-runtime-features.html)
- [Event structure](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-event-structure.html)
- [Use async and await](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/async-await-syntax.html)
- [Helper methods for key value stores](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-custom-methods.html)
- [Helper methods for origin modification](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/helper-functions-origin-modification.html)
- [General helper methods](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/general-helper-methods.html)
- [CWT support](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cwt-support-cloudfront-functions.html)
- [SaaS Manager helper methods](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/saas-specific-logic-function-code.html)
- [Restrictions on CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-function-restrictions.html)
- [Restrictions on all edge functions](https://docs.amazonaws.cn/en_us/AmazonCloudFront/latest/DeveloperGuide/edge-function-restrictions-all.html)
- [Quotas](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html)
- [Differences between CloudFront Functions and Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-choosing.html)
- [FunctionConfig(API リファレンス)](https://docs.aws.amazon.com/cloudfront/latest/APIReference/API_FunctionConfig.html)
- [Document history](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/WhatsNew.html)
- [yusukebe/pico](https://github.com/yusukebe/pico)
