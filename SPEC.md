# Tailwind CSS v4 コア + CLI 移植仕様 (SPEC)

本書は、Tailwind CSS v4 (このリポジトリの `packages/tailwindcss` と `packages/@tailwindcss-cli`、`crates/oxide`) の
機能のうち **コア API (`compile` / `build`) と CLI** を別言語で再実装するための仕様である。
参照実装の挙動を正とし、仕様の記述と参照実装が食い違う場合は参照実装 (ファイル名と行を併記) を優先する。

対象バージョン: `tailwindcss@4.3.3` (このリポジトリの `main`)。

---

## 0. スコープ

### 0.1 実装するもの

| 領域 | 内容 | 参照実装 |
| --- | --- | --- |
| コア API | `compile(css, options) -> { build(candidates) -> css, sources, root, features }` | `src/index.ts` |
| CSS 入力処理 | CSS パーサ、`@import` / `@reference`、`@theme`、`@source`、`@utility`、`@custom-variant`、`@variant`、`@apply`、`@tailwind utilities`、テーマ関数 (`--theme()`, `--spacing()`, `--alpha()`, `theme()`) | `src/css-parser.ts`, `src/at-import.ts`, `src/apply.ts`, `src/css-functions.ts` |
| テーマ | CSS 変数ベースのデザイントークンと名前空間解決 | `src/theme.ts` |
| 候補クラス名 | 文法・パース・印字 | `src/candidate.ts` |
| バリアント | 組み込みバリアント、複合バリアント、順序づけ | `src/variants.ts` |
| ユーティリティ | 組み込みユーティリティ (§7 で必須セットを定義)、`@utility` によるユーザー定義 | `src/utilities.ts` |
| コンパイル | 候補 → AST、ソート、`!important`、プレフィックス | `src/compile.ts`, `src/property-order.ts` |
| 出力 | AST 最適化 (重複除去、未使用テーマ変数の削除、ネスト展開)、CSS 文字列化 | `src/ast.ts` |
| ソース走査 | ファイル走査 (自動検出 + `@source`)、候補抽出、増分走査 | `crates/oxide` |
| CLI | `tailwindcss [build] -i -o -w --poll -m --optimize --cwd --map --silent` | `packages/@tailwindcss-cli` |

### 0.2 実装しないもの

- v3 互換レイヤー (`@config`、`@plugin`、JS 設定ファイル、JS プラグイン API、`theme()` のドット記法解決)。`src/compat/` 全体。
- IntelliSense 向け API (`getClassList`、`getVariants`、`candidatesToCss`)、クラスソート API (`getClassOrder`)、正規化 API (`canonicalizeCandidates`) と `canonicalize` サブコマンド。
- ソースマップ (`--map`)。CLI のフラグは受け付けてもよいが、未対応と明示してよい。
- Lightning CSS による最適化・最小化。`--minify` / `--optimize` は任意の CSS 最小化器で代替してよい (§12.5)。
- ブラウザ版、Vite / PostCSS / webpack / Turbopack 連携、アップグレードツール。
- 言語別プリプロセッサ (Vue / Svelte / Ruby / Pug など) による抽出精度の向上。§11.4 の共通抽出器のみ実装する。
- ポリフィル (`@property` フォールバック、`color-mix()` フォールバック) は任意 (§10.4)。

### 0.3 用語

| 用語 | 意味 |
| --- | --- |
| 候補 (candidate) | ソースから抽出した、ユーティリティクラスかもしれない文字列。例: `md:hover:bg-red-500/50` |
| ユーティリティ (utility) | 候補のうちバリアントを除いた本体。例: `bg-red-500/50` |
| ルート (root) | ユーティリティ名の固定部分。例: `bg` |
| 値 (value) | 名前つき値 (`red-500`)、任意値 (`[#fff]`)、または変数省略形 (`(--x)`) |
| modifier | `/` の後ろ。例: `50`、`[50%]`、`(--x)` |
| バリアント (variant) | `:` で区切られた条件。例: `md`、`hover`、`group-hover/name`、`[&_p]` |
| テーマキー | `@theme` 内の CSS 変数名。例: `--color-red-500` |
| 名前空間 | テーマキーの接頭辞。例: `--color` |
| デザインシステム | テーマ + ユーティリティ表 + バリアント表 + キャッシュをまとめたもの |

---

## 1. 全体アーキテクチャ

```
入力 CSS ──► CSS パース ──► @import 解決 ──► at-rule 収集 ──► デザインシステム構築
                                                               │
     (テーマ, @utility, @custom-variant, @source, important...) │
                                                               ▼
候補文字列 ──► parseCandidate ──► compileAstNodes ──► ソート ──► @tailwind utilities の位置に挿入
                                                               │
                                                               ▼
                      optimizeAst (重複除去 / 未使用変数削除 / ネスト展開) ──► toCss ──► 出力 CSS
```

処理は 2 段階に分かれる。

1. **`compile(css)`** (1 回): 入力 CSS を解析してデザインシステムを構築し、`build` 関数を返す。
2. **`build(candidates)`** (何度でも): 候補の集合からユーティリティ CSS を生成し、入力 CSS 内の `@tailwind utilities` の位置に差し込んだ完全な CSS を返す。候補は **追加のみ** され、一度有効だった候補は以後も出力され続ける。

dev (watch) と prod (build) は同じパイプラインである。違いは、watch では変更ファイルだけを再走査して `build` に差分候補を渡す点と、prod では最後に最小化を通す点だけである (§12)。

---

## 2. 公開 API

### 2.1 `compile`

```ts
compile(css: string, options?: CompileOptions): Promise<Compiler>

type CompileOptions = {
  base?: string          // 相対パス解決の基準ディレクトリ (既定: '')
  from?: string          // 入力ファイルパス (ソースマップ用。未対応でよい)
  polyfills?: Polyfills  // ビットフラグ。既定は All (§10.4)。未対応なら None 相当
  loadStylesheet?: (id: string, base: string) => Promise<{ path: string; base: string; content: string }>
  loadModule?: ...       // @plugin / @config 用。本仕様では常にエラーを投げる
}

type Compiler = {
  sources: { base: string; pattern: string; negated: boolean }[]  // @source 由来
  root: null | 'none' | { base: string; pattern: string }          // @tailwind utilities source(…) 由来
  features: Features                                                // 使用された機能のビットフラグ
  build(candidates: string[]): string
}
```

`Features` ビット: `AtApply=1`, `AtImport=2`, `JsPluginCompat=4`, `ThemeFunction=8`, `Utilities=16`, `Variants=32`, `AtTheme=64`。

`loadStylesheet` が未指定のまま `@import` に到達したらエラー。`loadModule` が未指定のまま `@plugin` / `@config` に到達したらエラー。

### 2.2 `build`

- `features === None` (Tailwind 固有の構文が一切ない) なら、入力 CSS をそのまま返す。
- `@tailwind utilities` が無ければ、最適化済みの入力 CSS を返す (候補は無視)。
- 新しい候補を有効候補集合に追加する。`--` で始まる候補は **CSS 変数の使用申告** として扱い、テーマ変数の "使用済み" マークだけ行う (§6.6)。
- 有効候補集合が変化しなかった場合、前回の出力を返す (参照等価でよい)。
- そうでなければ、全有効候補を `compileCandidates` (§8) で AST にし、`@tailwind utilities` ノードの子として差し替え、`optimizeAst` (§10) を通して `toCss` で文字列化する。
- 一度無効と判定した候補は `invalidCandidates` に記録し、以後はパースを省略する。

生成される CSS の先頭には `/*! tailwindcss v<version> | MIT License | https://tailwindcss.com */` を付ける (テスト環境では省略可)。

### 2.3 エラー

構文エラーおよび §4 で「エラー」とした事象は例外として `compile` (または `build`) から送出する。CLI は例外をメッセージ表示して終了コード 1 (watch 中はメッセージ表示のみで継続) とする。

---

## 3. CSS パーサと AST

### 3.1 AST ノード

```
StyleRule   { kind: 'rule',        selector: string, nodes: AstNode[] }
AtRule      { kind: 'at-rule',     name: string /* '@' を含む */, params: string, nodes: AstNode[] }
Declaration { kind: 'declaration', property: string, value: string | undefined, important: boolean }
Comment     { kind: 'comment',     value: string }
Context     { kind: 'context',     context: Record<string, string|boolean>, nodes: AstNode[] }  // 印字されない。子にメタ情報を渡す
AtRoot      { kind: 'at-root',     nodes: AstNode[] }  // 印字時にドキュメント末尾へ巻き上げる
```

`rule(selector, nodes)` ヘルパーは、`selector` が `@` で始まれば `AtRule` (名前と params に分割)、そうでなければ `StyleRule` を作る。

### 3.2 パーサの要件

- 標準的な CSS をネスト構文込みでパースする (`&`、ネストした at-rule)。
- コメントは通常捨てるが、`/*!` で始まるライセンスコメントは `Comment` として保持する。
- `!important` を宣言から分離して `important: true` にする。
- 値の末尾 `;` の省略 (ブロック末尾) を許容する。
- 引用符と括弧の内側にある `;` `{` `}` を区切りとして扱わない。
- 不正な入力 (閉じ括弧の不足など) は位置つきの構文エラーを投げる。
- `@import` などボディを持たない at-rule は `nodes: []` とする。

### 3.3 文字列化 (`toCss`)

```
<indent>property: value !important;      ← 宣言 (important は " !important")
<indent>selector {                        ← ルール
<indent>  ...
<indent>}
<indent>@name params {                    ← at-rule (params が空なら "@name {")
<indent>@name params;                     ← 子を持たない at-rule
<indent>/*value*/                         ← コメント
```

インデントは半角スペース 2 個、各行末は `\n`。`Context` は子だけを同じ深さで印字する。`AtRoot` はこの段階では現れない (§10 で解消済み)。値が `undefined` の宣言は印字しない。

---

## 4. 入力 CSS の処理 (`parseCss`)

入力 AST を `Context { base }` で包み、以下の順で処理する。

1. `@import` / `@reference` の解決 (§4.1)
2. at-rule の収集 (§4.2〜4.9)。AST を 1 回走査し、該当 at-rule を登録・除去する
3. デザインシステム構築 (`buildDesignSystem`)、`important` の設定、`@source not inline(…)` の候補を `invalidCandidates` へ
4. カスタムバリアントの登録 (§4.7)、カスタムユーティリティの登録 (§4.6)
5. 最初の `@theme` の位置に `:root, :host { …全テーマ変数… }` を出力 (§6.5)
6. ネストされた `@variant` の展開 (§4.8)
7. テーマ関数の置換 (§4.10)
8. `@apply` の展開 (§4.9)
9. `@tailwind utilities` ノードを `Context {}` に変換 (子は `build` 時に差し替える)
10. 残った `@utility` ノードを削除

### 4.1 `@import` と `@reference`

`@import "<uri>" [layer(<name>)] [supports(<cond>)] [<media-query>...];`

- `uri` は引用符必須。`url(…)` 形式、`data:`、`http://`、`https://` は解決せずそのまま残す。
- `loadStylesheet(uri, base)` で読み込み、再帰的に `@import` を解決する (深さ 100 を超えたらエラー)。
- 読み込んだ AST を `Context { base: <読み込んだファイルのディレクトリ> }` で包み、`layer(x)` があれば `@layer x { … }`、メディアクエリがあれば `@media <mq> { … }`、`supports(x)` があれば `@supports (x) { … }` の順で内側から包む。
- `layer(…)` は他の条件より前に書かれていなければエラー。
- `@reference "x";` は `@import "x" reference;` と同じ。
- `@import "tailwindcss";` はパッケージの `index.css` を指す。CLI の `loadStylesheet` は Node のモジュール解決 (`node_modules/tailwindcss/index.css`、`exports` の `style` 条件) を行う。移植版では「`tailwindcss` という id は同梱の `index.css` を返す」実装でよい。

同梱すべき CSS ファイル (このリポジトリからそのままコピーする):

| ファイル | 内容 |
| --- | --- |
| `index.css` | `@layer theme, base, components, utilities;` + 3 つの `@import` |
| `theme.css` | `@theme default { … }` (色パレット、spacing、breakpoint、container、text、font、radius、shadow、ease、animate、blur、default-*、`@keyframes`) |
| `preflight.css` | ブラウザリセット。`@layer base` に入る |
| `utilities.css` | `@tailwind utilities;` のみ |

`index.css` は `@import './theme.css' layer(theme); @import './preflight.css' layer(base); @import './utilities.css' layer(utilities);` である。

#### `@import` のメディア位置に書ける Tailwind 固有パラメータ

`@import` 解決後、`@media <params>` として現れたものを後処理する。パラメータは空白区切りで解釈し、Tailwind 固有のものを消費して残りを `@media` に残す (全部消費されたら `@media` を外して子を展開する)。

| パラメータ | 効果 |
| --- | --- |
| `reference` | 子を `Context { reference: true }` で包む。参照モード (§6.4) |
| `theme(<opts>)` | 子の `@theme` の params に `<opts>` を追記する。`reference` を含むとき、子に `@theme` 以外のルールがあればエラー |
| `prefix(<ident>)` | 子の `@theme` に `prefix(<ident>)` を追記する |
| `important` | デザインシステム全体を `important = true` にする |
| `source(<path>)` | 子の `@tailwind utilities` を `@tailwind utilities source(<path>)` に書き換え、`Context { sourceBase }` で包む |

例: `@import "tailwindcss" important;`、`@import "tailwindcss" prefix(tw);`、`@import "tailwindcss" source("../src");`、`@import "./theme.css" theme(reference);`。

### 4.2 `@tailwind utilities [source(…)]`

- 最初の 1 つだけを記憶し、2 つ目以降は削除する。
- `Context { reference: true }` の内側にあれば無視して削除する。
- `source(none)` → `root = 'none'` (自動検出無効)。`source("<path>")` → `root = { base: <sourceBase または base>, pattern: <path> }`。パスは引用符必須 (無ければエラー)。

### 4.3 `@theme [options] { … }`

- params は空白区切りのオプション: `reference`、`inline`、`default`、`static`、`prefix(<ident>)`。
- `prefix` は `/^[a-z]+$/` を満たさなければエラー。`theme.prefix` に設定する。
- `Context { reference }` の中にある `@theme` は自動的に `reference` 扱い。
- 子は **カスタムプロパティ宣言 (`--` で始まる) と `@keyframes` とコメント** のみ許可。それ以外はエラー。
- 宣言は `theme.add(property, value, options)` (§6.2) で登録。`@keyframes` は `theme.addKeyframes()` に保持。
- 最初の `@theme` は `:root, :host {}` に置き換え (中身は後で埋める)、2 つ目以降は削除する。

### 4.4 `@source`

```
@source "<glob>";                 走査対象を追加
@source not "<glob>";             走査対象から除外
@source inline("<pattern>");      候補を直接追加 (ブレース展開あり)
@source not inline("<pattern>");  候補を明示的に無効化
```

- ボディがあればエラー。ネストされていればエラー。パスは引用符必須。
- 通常形は `sources.push({ base: <Context の base>, pattern, negated })`。
- `inline(…)` は引用符内を空白で分割し、各要素をブレース展開 (§4.4.1) して候補リストに加える。`not` つきは `invalidCandidates` に入る。

#### 4.4.1 ブレース展開

`{a,b,c}` は列挙、`{1..5}` / `{10..0}` / `{0..20..5}` は数値範囲 (負数可、step 可、step 0 はエラー)。ネスト可。括弧が釣り合わなければエラー。
例: `bg-{red,blue}-{100..300..100}` → `bg-red-100 bg-red-200 bg-red-300 bg-blue-100 …`。

### 4.5 `@custom-variant`

2 形式ある。どちらもトップレベル限定 (ネストはエラー)。名前は `/^@?[a-z0-9][a-zA-Z0-9_-]*(?<![_-])$/` を満たすこと。

**(a) セレクタ短縮形** `@custom-variant <name> (<sel>[, <sel>...]);`

- 括弧内をカンマで分割 (括弧・引用符を考慮した分割 §14.1)。空要素があればエラー。
- `@` で始まる要素は at-rule、それ以外はスタイルルールのセレクタ。
- 適用時: スタイルセレクタ群を `, ` で結合した 1 つの `StyleRule` と、at-rule ごとの `AtRule` を生成し、それぞれの子に元のノードを入れる (§8.3 の static バリアントと同じ)。
- 複合可能性 (`compounds`) は `compoundsForSelectors` (§7.4) で決める。

**(b) ボディ形式** `@custom-variant <name> { … @slot; … }`

- ボディ内の `@slot;` の位置にユーティリティの宣言が挿入される。
- ボディ内で `@variant <other>` を使える。依存関係をトポロジカルソートして登録順を決める (循環はエラー)。
- ボディ内の `@keyframes` / `@property` は `AtRoot` で包む (ルートへ巻き上げ)。

セレクタと ボディの両方があればエラー。どちらも無ければエラー。

`@custom-variant` の登録は **まず名前だけを CSS 中の出現順で予約** し (順序を確定するため)、その後トポロジカル順に実体を登録する。

**互換**: トップレベルの `@variant <name> (…);` (ボディなし) と、`@slot` を含む `@variant <name> { … }` は `@custom-variant` として扱う。

### 4.6 `@utility`

```
@utility <name> { <宣言 / ネストルール> }        静的ユーティリティ
@utility <name>-* { … --value(…) --modifier(…) … }  関数的ユーティリティ
```

- トップレベル限定。ボディが空ならエラー。
- 名前 (`\/` などのエスケープは解除) の妥当性: §7.6。無効ならエラー (末尾が `*` だが `-*` でない、`*` が途中にある、などメッセージを分ける)。
- 静的: 候補 `<name>` に対してボディをそのまま (クローンして) 返す。
- 関数的: §7.7 の規則で `--value(…)` / `--modifier(…)` を解決する。
- ボディ内の `@apply` は §4.9 で先に展開される (`@utility` 同士の依存はトポロジカルソート、循環はエラー)。
- 処理後、`@utility` ノードは削除する。

### 4.7 `@variant` (ネスト利用)

スタイルルールの中で使う `@variant <v1>[:<v2>...][, <v3>...] { … }`。

- カンマ区切りは OR (それぞれ別ルールを生成)。コロン区切りは重ね掛け (右から左へ適用)。
- 各バリアントを `parseVariant` し、`&` ルールに順に `applyVariant` (§8.3) する。未知のバリアント、空のバリアント、適用不能ならエラー。
- 結果のセレクタが `&` のままなら子だけを展開する。

`@custom-variant` のボディ内でも同じ処理を行う。

### 4.8 `@media` 内の `@tailwind utilities` / `@theme`

§4.1 の表を参照。

### 4.9 `@apply`

`@apply <candidate> [<candidate>...];` をルールの内側で使う。

- トップレベルの `@apply` は無視する。`@keyframes` 内はエラー。ボディがあればエラー。
- 引数が全て `--` で始まる (CSS mixin 構文) ならそのまま残す。`--` 始まりと通常の候補が混在していればエラー。
- 各候補を `compileCandidates` (`respectImportant: false`、つまり `@import … important` の影響を受けない。候補自身の `!` は有効) でコンパイルし、生成された各ルールの **子** (セレクタは捨てる) を `@apply` の位置に展開する。バリアントつき候補 (`hover:underline`) の場合は `&:hover { … }` のようなネストとして展開される。
- 無効な候補はエラー。エラーメッセージは状況で分ける: プレフィックス未指定、`@source not inline` で無効化済み、バリアントが存在しない、テーマが空 (`@import "tailwindcss"` 忘れ)、その他。
- `@utility` 内の `@apply` は他の `@utility` を参照できる。依存グラフをトポロジカルソートして順に展開する (循環はエラー)。

### 4.10 テーマ関数

宣言の値と、`@media` / `@custom-media` / `@container` / `@supports` の params の中で以下の関数を置換する。引数はカンマで分割 (括弧考慮) してトリムする。

| 関数 | 挙動 |
| --- | --- |
| `--spacing(<n>)` | `--spacing` テーマ値 (`m`) を使い `calc(m * n)`。`n` が `0` なら `0px`、`1` なら `m`。引数が無い / 複数、`--spacing` 未定義はエラー |
| `--alpha(<color> / <alpha>)` | `withAlpha(color, alpha)` (§7.3)。`/` で分割できなければエラー、引数が複数ならエラー |
| `--theme(<key>[, <fallback>...][ inline])` | `key` は `--` 始まり必須。`resolveThemeValue(key, inline)` (§6.3)。at-rule の params 内では常に inline。値が無ければ fallback、fallback も無ければエラー。fallback が `initial` なら解決値。解決値が `initial` なら fallback。解決値が `var(…)` / `theme(…)` / `--theme(…)` で始まるなら、その最内の fallback が無い / `initial` の場合に fallback を注入する |
| `theme(<key>[, <fallback>...])` | レガシー。`key` は引用符を外す。`resolveThemeValue(key)` (inline 固定)。無ければ fallback、それも無ければエラー。本仕様では `--` 始まりのキーのみサポート (ドット記法は互換レイヤーの担当) |

生成されたユーティリティの宣言値 (任意値に `theme(…)` を書いた場合など) にも同じ置換を行う。置換に失敗した候補は無効扱い (エラーにしない)。

---

## 5. デザインシステム

```
DesignSystem {
  theme: Theme
  utilities: Utilities         // root -> Utility[] (kind: 'static' | 'functional', compileFn, options?)
  variants: Variants           // name -> { kind, order, applyFn, compounds, compoundsWith }
  invalidCandidates: Set<string>
  important: boolean

  parseCandidate(raw): Candidate[]        // メモ化
  parseVariant(raw): Variant | null       // メモ化 (同じ文字列は同じオブジェクト。順序計算で同一性を使う)
  compileAstNodes(candidate, flags)       // メモ化。テーマ関数置換と @variant 展開も行う。失敗したら []
  getVariantOrder(): Map<Variant, number> // §8.4
  resolveThemeValue(path, forceInline)    // §6.3
  trackUsedVariables(raw)                 // 値中の var(--x) を "使用済み" にする
}
```

---

## 6. テーマ

### 6.1 データ構造

`Map<key, { value: string, options: ThemeOptions }>` と `@keyframes` の集合、`prefix: string | null`。

`ThemeOptions` ビット: `INLINE=1`, `REFERENCE=2`, `DEFAULT=4`, `STATIC=8`, `USED=16`。

### 6.2 `add(key, value, options)`

1. `key` が `-*` で終わる場合、`value` は `initial` でなければエラー。`--*` なら全消去、それ以外は `key` から `-*` を除いた名前空間を消去 (§6.2.1)。
2. `options` に `DEFAULT` があり、既存の値が `DEFAULT` でなければ何もしない (ユーザー定義がデフォルトに勝つ。順序に依らない)。
3. `value === 'initial'` ならキーを削除、そうでなければ登録 (上書き)。

#### 6.2.1 名前空間の消去 (`clearNamespace`)

`namespace` で始まる全キーを削除する。ただし「無視すべきキー表」に該当するものは残す。

| 名前空間 | 無視するキー (これらとその `-` 接尾辞) |
| --- | --- |
| `--font` | `--font-weight`, `--font-size` |
| `--inset` | `--inset-shadow`, `--inset-ring` |
| `--text` | `--text-color`, `--text-decoration-color`, `--text-decoration-thickness`, `--text-indent`, `--text-shadow`, `--text-underline-offset` |
| `--grid-column` | `--grid-column-start`, `--grid-column-end` |
| `--grid-row` | `--grid-row-start`, `--grid-row-end` |

この表は `resolve` / `keysInNamespaces` でも使う (例: `--text` 名前空間で `--text-shadow-sm` を `text-shadow-sm` として解決しない)。

### 6.3 解決

`resolveKey(candidateValue, namespaces)`:
名前空間を順に試し、`candidateValue` が null なら `namespace` そのもの、そうでなければ `${namespace}-${candidateValue}` が存在するキーを返す。存在しない場合、`candidateValue` に `.` が含まれていれば `.` を `_` に置換したキーも試す。無視キー表に該当すれば飛ばす。

`resolve(candidateValue, namespaces, options)`:
キーが見つかれば、`INLINE` (引数か登録時) なら生の値、そうでなければ `var(<escape(prefixKey(key))>)`。`REFERENCE` のキーは `var(<key>, <生の値>)` (参照モードでは変数が出力されないため)。

`resolveValue(...)`: 常に生の値。

`resolveWith(candidateValue, namespaces, nestedKeys)`: キー `k` を解決し、加えて `k + nestedKey` (例: `--text-lg` + `--line-height` = `--text-lg--line-height`) を同じ規則で解決して `{ nestedKey: value }` として返す。

`get(keys)`: キー列を順に見て最初に存在する生の値。

`namespace(ns)`: `ns` そのもの (キー null)、`ns-...` (接頭辞を除いたキー)、`ns--...` (`--` 付きの副キー) を列挙。

`resolveThemeValue(path, forceInline = true)`: `path` の最後の `/` 以降を modifier として切り出し、`resolve(null, [path], INLINE if forceInline)` の結果に `withAlpha` を適用する。

`prefixKey(--color-red-500)` は prefix が `tw` のとき `--tw-color-red-500`。

### 6.4 モード

| オプション | 効果 |
| --- | --- |
| `inline` | 利用側に `var()` ではなく生の値を埋め込む。変数自体は出力される |
| `reference` | 変数を出力しない。利用側は `var(--x, <値>)` |
| `default` | ユーザー定義があればそちらを優先 (同梱 `theme.css` が使う) |
| `static` | 未使用でも変数を出力する (§6.6 の削除対象外) |

### 6.5 出力

最初の `@theme` の位置の `:root, :host { … }` に、`REFERENCE` でない全テーマ変数を登録順に `escape(prefixKey(key)): value` として出力する。`@keyframes` は `Context { theme: true } > AtRoot` としてドキュメント末尾へ。`--default-*` など値が `initial` に解決したものは出力しない。

### 6.6 未使用テーマ変数・キーフレームの削除 (`optimizeAst` 内)

- `@theme` 由来の宣言 (`Context { theme: true }` の中の `--` 宣言) について、`STATIC` でも `USED` でもなく、それに依存する変数 (テーマ内で `var(--x)` を参照している別の変数) も使われていなければ削除する。
- `USED` は、生成された CSS (ユーティリティ、ユーザー CSS の宣言値) に `var(--x)` が出現したとき、あるいは `--x` という候補文字列が走査結果に含まれたとき (`build` の `--` 候補) に付く。
- `:root, :host` が空になったら削除し、その上の空になった `@layer` も削除する。
- テーマ内で定義した `@keyframes` は、`animation` 宣言、または使用済みの `--animate-*` 変数の値に名前が現れなければ削除する。

---

## 7. 候補 (candidate) の文法

### 7.1 データ型

```
Candidate =
  | { kind: 'static',     root, variants: Variant[], important, raw }
  | { kind: 'functional', root, value: Value | null, modifier: Modifier | null, variants, important, raw }
  | { kind: 'arbitrary',  property, value: string, modifier, variants, important, raw }

Value    = { kind: 'named', value, fraction: string | null } | { kind: 'arbitrary', dataType: string | null, value }
Modifier = { kind: 'named', value } | { kind: 'arbitrary', value }

Variant =
  | { kind: 'static',     root }
  | { kind: 'functional', root, value: { kind:'named'|'arbitrary', value } | null, modifier }
  | { kind: 'compound',   root, modifier, variant: Variant }
  | { kind: 'arbitrary',  selector, relative: boolean }
```

### 7.2 `parseCandidate(input)` — 0 個以上の候補解釈を返す

1. `segment(input, ':')` (§14.1) で分割。最後の要素が本体、それ以前がバリアント。
2. **プレフィックス**: `theme.prefix` があるとき、要素が 1 つだけ、または先頭要素がプレフィックスと一致しなければ無効。一致した先頭要素を除去。
3. バリアントを **右から左** に `parseVariant`。1 つでも null なら無効。結果の配列は右端のバリアントが先頭 (適用順)。
4. **important**: 本体末尾が `!` なら除去して `important = true`。そうでなく先頭が `!` (旧構文) でも同様。
5. **静的一致**: `utilities.has(base, 'static')` かつ `base` に `[` が無ければ `static` 候補を出力する (以降も続行)。
6. `segment(base, '/')` が 3 要素以上なら無効。1 要素目を本体、2 要素目があれば modifier 文字列。
7. modifier のパース (§7.2.1)。modifier 文字列があるのに null なら無効。
8. **任意プロパティ** (本体が `[` 始まり): `]` 終わり必須。2 文字目は `a-z` か `-`。`[` `]` を外し、最初の `:` で property / value に分割 (`:` が無い、先頭、末尾なら無効)。value は `decodeArbitraryValue` (§7.2.2) し、`isValidArbitrary` (§14.3) を満たさなければ無効。`arbitrary` 候補を出力して終了。
9. **任意値** (本体が `]` 終わり): 最初の `-[` の前をルートとし、`utilities.has(root, 'functional')` でなければ無効。ルート候補は `[root, "[...]"]` の 1 つ。
10. **変数省略形** (本体が `)` 終わり): 最初の `-(` の前をルート (functional 必須)。括弧内を `segment(':')` し、2 要素なら 1 つ目をデータ型。値は `--` 始まり必須、`isValidArbitrary` 必須。値を `[var(--x)]` または `[type:var(--x)]` に書き換えて 9 と同じ扱い。
11. それ以外: `findRoots(base, root => utilities.has(root, 'functional'))` (§7.2.3)。
12. 各 `[root, value]` について `functional` 候補を作る。`value === null` なら値なしで出力。
    - `value` に `[` が含まれる場合: `]` 終わり必須 (違えば全体無効)。中身を decode し、`isValidArbitrary` を満たさなければこの解釈を飛ばす。先頭から `[a-z-]+` を読んで直後が `:` なら型ヒント。空 (空白のみ) や型ヒントが空文字なら飛ばす。`{ kind:'arbitrary', dataType, value }`。
    - それ以外: `fraction` = modifier 文字列があり、かつ modifier が named なら `${value}/${modifier}`、さもなくば null。`value` が `/^[a-zA-Z0-9_.%-]+$/` を満たさなければ飛ばす。`{ kind:'named', value, fraction }`。

#### 7.2.1 modifier のパース

- `[x]`: decode + `isValidArbitrary` + 非空 → arbitrary。
- `(--x)`: `--` 始まり必須、`isValidArbitrary` → arbitrary、値は `var(--x)`。
- それ以外: `/^[a-zA-Z0-9_.%-]+$/` → named。満たさなければ null。

#### 7.2.2 `decodeArbitraryValue`

- `(` を含まない場合: `\_` → `_`、それ以外の `_` → 空白。
- 含む場合: 値パーサで関数呼び出し木にし、`url(…)` (と `*_url`) の中身は変換しない。`var(…)` / `theme(…)` の第 1 引数は `_` を保持 (エスケープ解除のみ)。その他は再帰的に `_` → 空白。最後に `calc()` 系関数内の演算子 `+ - * /` の前後に空白を入れる (`calc(1px+2px)` → `calc(1px + 2px)`。`-` は前後が数値/`)`/`(`の場合のみ演算子と見なす。実装は `utils/math-operators.ts`)。

#### 7.2.3 `findRoots(input, exists)`

1. `exists(input)` なら `[input, null]` を出力。
2. 最後の `-` から左へ順に切り詰め、`exists(prefix)` なら `[prefix, rest]` を出力。`rest` が空なら打ち切り。`prefix` が `@` で `@` が存在し、区切りが `-` の場合も打ち切り (`@-2xl` 対策)。
3. 入力が `@` 始まりで `exists('@')` なら最後に `['@', input.slice(1)]` を出力。

複数の解釈が返るので (例: `border-t-2` は `border-t`+`2` と `border`+`t-2`)、コンパイル時に CSS を生成できたものを採用する。

### 7.3 色と不透明度

`withAlpha(color, alpha)`: `alpha` が数値なら `alpha*100 + '%'` に変換。`100%` なら `color` のまま。それ以外は `color-mix(in oklab, <color> <alpha>, transparent)`。

`asColor(value, modifier, theme)`: modifier なしなら `value`。arbitrary modifier なら `withAlpha(value, modifier.value)`。named なら `--opacity` 名前空間で解決した値、無ければ modifier が 0.25 の倍数の数値 (§14.4) のとき `${modifier}%`、それ以外は null (無効)。

`resolveThemeColor(candidate, namespaces)`: `inherit` → `inherit`、`transparent` → `transparent`、`current` → `currentcolor`、それ以外は `theme.resolve(value, namespaces)`。結果に `asColor` を適用。

### 7.4 印字 (`printCandidate`)

`raw` をそのまま使う。選択子は `.${escape(raw)}` (§14.2 の CSS.escape 相当)。例: `.md\:hover\:bg-red-500\/50`。

### 7.5 組み込みユーティリティの定義パターン

参照実装は 4 つのヘルパーでほぼ全ユーティリティを定義している。移植版も同じ構造にすること。

**`staticUtility(name, declarations)`**: 候補 `name` → 宣言の配列 (または `AtRoot` を返す関数)。

**`functionalUtility(root, desc)`**:

```
desc = {
  supportsNegative?      // true なら `-root` も登録 (値を calc(v * -1) にする)
  supportsFractions?     // `w-1/2` のような分数を calc(1 / 2 * 100%) にする
  themeKeys?             // 名前つき値を解決する名前空間の優先順リスト
  defaultValue?          // 値なし候補 (例 `rounded`) の値。undefined なら themeKeys の名前空間そのもの (`--radius`) を使う
  staticValues?          // 値 → 宣言配列 の表 (例 `auto`, `full`)。負値・modifier つきでは使わない
  handleBareValue?(v)    // テーマに無い名前つき値の処理 (例 `z-10` → '10')。null で不可
  handleNegativeBareValue?(v)
  handle(value, dataType) // 最終値 → 宣言配列
}
```

値の決定順 (候補が functional のとき):

1. 値なし: modifier があれば無効。`defaultValue` または `theme.resolve(null, themeKeys)`。
2. 任意値: modifier があれば無効。値と型ヒントをそのまま `handle` へ。
3. 名前つき値: `theme.resolve(fraction ?? value, themeKeys)`。
   - 解決でき、modifier があり、fraction を消費していなければ無効 (`w-4/foo`)。
   - 未解決で `supportsFractions` かつ fraction あり: 分子分母が非負整数なら `calc(a / b * 100%)`。
   - 未解決で負値かつ `handleNegativeBareValue` あり: その結果 (`/` を含まない値で modifier があれば無効)。結果を `handle` へ (負号を二重にしない)。
   - 未解決で `handleBareValue` あり: その結果 (`/` を含まない値で modifier があれば無効)。
   - 未解決で非負・modifier なし・`staticValues` に該当: その宣言 (クローン)。
4. 値が null なら無効。負値なら `calc(<value> * -1)` にして `handle`。

**`colorUtility(root, { themeKeys, handle })`**: 値必須。任意値なら `asColor(value, modifier)`、名前つきなら `resolveThemeColor`。null なら無効。

**`spacingUtility(name, themeKeys, handle, { supportsNegative, supportsFractions, staticValues })`**:
`name-px` (値 `1px`) と、`supportsNegative` なら `-name-px` (`-1px`) を静的登録。加えて `functionalUtility(name, …)` を `defaultValue: null`、`handleBareValue: v が 0.25 の倍数 → "--spacing(v)"`、`handleNegativeBareValue: → "--spacing(-v)"` で登録。`--spacing` テーマ値が無ければ bare value は不可。`--spacing(v)` はその後 §4.10 で `calc(var(--spacing) * v)` に置換される。

**カスタム関数** (`utilities.functional(root, fn)`): `bg`、`text`、`border`、`font` などは値の型を推論して複数のプロパティに振り分ける (§7.8)。

`compileFn` の戻り値: `AstNode[]` = 成功、`undefined` = この定義では扱えない (次の定義へ)、`null` = 無効 (options.types があれば以降も打ち切り)。

### 7.6 `@utility` の名前規則

- 静的名: `/^-?[a-z][a-zA-Z0-9_-]*/` にマッチする root の後ろに、`a-zA-Z0-9_-`、`.` (前後が数字)、`%` (末尾のみ、直前が数字)、`/` (1 回まで、末尾不可) のみ。root が `-` で終わり残りが空なら無効。
- 関数的名: `-*` で終わり、残りが `/^-?[a-z][a-zA-Z0-9_-]*$/` 全体にマッチ。

### 7.7 `@utility name-*` の `--value()` / `--modifier()`

宣言値の中の `--value(<arg>[, <arg>...])` と `--modifier(...)` を候補の value / modifier で置換する。引数は左から順に試し、最初に解決できたものを採用する。

前処理 (Prettier 対策): `\*` → `*`、`--foo --bar` → `--foo-*--bar`、空白除去、連続する `-*` を 1 つに、`--x` (括弧も `-*` も無い) → `--x-*`。

| 引数 | 対象 | 解決 |
| --- | --- | --- |
| `'literal'` / `"literal"` | named | 候補値がリテラルと等しければその値 |
| `--ns-*` | named | `theme.resolve(value, ['--ns'])` |
| `--ns-*--sub` | named | `resolveWith(value, ['--ns'], ['--sub'])` の `--sub` 側 |
| `number` / `integer` / `ratio` / `percentage` | named | 型推論に合格した値。`ratio` は `fraction` を使い `a / b` (整数のみ)、`number` は 0.25 の倍数、`percentage` は整数 % |
| `[type]` | arbitrary | 型ヒントがあれば一致必須。無ければ推論で一致すれば値。`[*]` は何でも |
| `--default(<v>)` | 値なし | 候補に値が無いときの既定値 |

有効性の規則:

- `--value(…)` が 1 つも使われていない、または 1 つも解決しなかった → 無効。
- 解決できなかった `--value` / `--modifier` を含む宣言は削除する。
- `--modifier` を使っていて解決せず、候補に modifier がある → 無効。
- `ratio` で解決した場合、`--modifier` も解決していれば無効。`ratio` 解決時は ratio 以外で解決した宣言を削除する。
- 候補に modifier があるのに `ratio` でも `--modifier` でも消費されなかった → 無効。
- サポート外のデータ型名は警告して無視する。
- `--spacing(--value(number))` のような入れ子も可 (§4.10 の置換が後で走る)。

### 7.8 型推論 (`inferDataType(value, types)`)

`types` を順に試し、最初に合致した型名を返す。`var(…)` 始まりの値は常に null。主な述語:

| 型 | 条件 |
| --- | --- |
| `color` | 名前色、`#hex`、`rgb/rgba/hsl/hsla/oklch/oklab/lab/lch/color/color-mix/light-dark(...)`、`transparent`、`currentcolor` など |
| `length` | 数値 + 長さ単位 (`px rem em vh vw … cqw …`)、`calc/min/max/clamp(...)`、`--spacing(...)`、`0` |
| `percentage` | 数値 + `%`、`calc` 系 |
| `number` | 数値 |
| `integer` | 非負整数 |
| `ratio` | `a/b` (数値) |
| `url` | `url(...)` |
| `image` | `image/image-set/cross-fade/element(...)`、`*-gradient(...)` |
| `position` | `top/left/center...` の組み合わせ |
| `bg-size` | `cover/contain/auto`、長さや % の 1〜2 個 |
| `line-width` | `thin/medium/thick`、長さ |
| `absolute-size` / `relative-size` | `xx-small` 等 / `larger`/`smaller` |
| `family-name` / `generic-name` | フォント名 / `serif` 等 |
| `angle`, `vector` | `deg/rad/grad/turn` / `n n n` |

正確な正規表現は `src/utils/infer-data-type.ts` を参照。

### 7.9 必須ユーティリティセット

参照実装は静的約 490 / 関数的約 110 を持つ。移植版が最低限備えるべきセットを以下に示す (これ以外は `src/utilities.ts` を参照して同じパターンで追加できる)。`--tw-*` 変数を使う合成型ユーティリティ (transform、filter、shadow、ring、gradient、mask) は「追加」扱いとする。

**レイアウト (静的)**
`block inline-block inline flex inline-flex grid inline-grid hidden contents flow-root table table-* list-item`、`static fixed absolute relative sticky`、`visible invisible collapse`、`isolate isolation-auto`、`box-border box-content`、`overflow-{auto,hidden,clip,visible,scroll} overflow-{x,y}-*`、`float-{left,right,start,end,none} clear-*`、`sr-only not-sr-only`、`container-type 系 (@container)`。

**位置 / z / order (spacing 型または bare 整数)**
`inset inset-x inset-y inset-s inset-e top right bottom left` → `spacingUtility(name, ['--inset','--spacing'])` (負値・分数可) + `-auto` / `-full` / `--full`。`z` (bare 非負整数、`z-auto`、負値可、`--z-index`)。`order` (bare 非負整数、負値可、`--order`; `order-first`=`-9999`、`order-last`=`9999`)。

**Flex / Grid**
`flex-row flex-row-reverse flex-col flex-col-reverse flex-wrap flex-nowrap flex-wrap-reverse`、`flex-auto(auto) flex-initial(0 auto) flex-none`、`flex-<n>` (bare 非負整数 → `flex: <n>`)、`flex-<a>/<b>` (`calc(a/b * 100%)`)、`flex-[…]`、`grow[-n] shrink[-n]`、`basis-*` (spacing/container/分数)、`grid-cols-<n>` (`repeat(n, minmax(0, 1fr))`)、`grid-rows-<n>`、`col-span-<n>` (`span n / span n`)、`col-start/end-<n>`、`row-*`、`grid-flow-*`、`auto-cols/rows-*`、`gap gap-x gap-y` (`spacingUtility`, `--gap`)、`justify-* items-* content-* self-* justify-items-* justify-self-* place-*` (静的)、`space-x/y-<n>` (追加扱い)。

**スペーシング**
`p px py ps pe pt pr pb pl` → `padding*` (`--padding`,`--spacing`)。`m mx my ms me mt mr mb ml` → `margin*` (負値可) + `m*-auto`。

**サイズ**
`w min-w max-w` (`--width|--min-width|--max-width`, `--spacing`, `--container`; 分数可) + 静的 `w-auto w-full w-screen w-svw w-lvw w-dvw w-min w-max w-fit`、`h min-h max-h` 同様 (`--height`, `vh` 系)、`size-*` (width + height、`--tw-sort: size`)。

**タイポグラフィ**
`font-<family>` (`--font-*`, 副キー `--font-feature-settings` / `--font-variation-settings`) と `font-<weight>` (`--font-weight-*`; `--tw-font-weight` 変数 + `@property`)、`text-<size>` (`--text-*` + `--line-height` 副キー → `line-height: var(--tw-leading, var(--text-lg--line-height))`; modifier `/<leading>` で行間指定)、`text-<color>` (`--text-color`,`--color`)、`text-left/center/right/justify/start/end`、`leading-*` (`--leading`,`--spacing`; `--tw-leading`)、`tracking-*` (`--tracking`; 負値可)、`uppercase lowercase capitalize normal-case`、`italic not-italic`、`underline overline line-through no-underline`、`truncate text-ellipsis text-clip`、`whitespace-*`、`break-*`、`list-*`、`antialiased subpixel-antialiased`、`decoration-*`、`underline-offset-*`。

**背景 / ボーダー**
`bg-<color>` (`--background-color`,`--color`)、`bg-<image>` (`--background-image`)、任意値は型推論で `background-position` / `background-size` / `background-image` / `background-color` に振り分け、`bg-{auto,cover,contain} bg-{fixed,local,scroll} bg-{top,center,...} bg-{repeat,no-repeat,...} bg-none bg-clip-* bg-origin-*`。
`border[-{x,y,s,e,t,r,b,l}]` (値なし → `--default-border-width` か `1px`; 整数 → `<n>px`; 色 → `border-color`; `--tw-border-style` 変数と `border-style: var(--tw-border-style)` を伴う)、`border-{solid,dashed,dotted,double,hidden,none}`、`rounded[-{s,e,t,r,b,l,ss,se,ee,es,tl,tr,br,bl}]` (`--radius`; `rounded-none`=0、`rounded-full`=`calc(infinity * 1px)`)、`outline-*` (追加扱い)。

**効果 / その他**
`opacity-<n>` (`--opacity`; bare 0.25 刻み → `n%`)、`shadow-*` / `ring-*` / `blur-*` / `transform` 系 (追加扱い)、`transition[-*]` (`transition-property` + `--default-transition-timing-function` / `--default-transition-duration`)、`duration-<n>` (`n ms`)、`delay-<n>`、`ease-*` (`--ease`)、`animate-*` (`--animate`)、`cursor-*`、`select-*`、`pointer-events-*`、`resize*`、`appearance-*`、`scroll-*`、`will-change-*`、`content-[…]`、`aspect-*` (`--aspect`, `ratio`)、`columns-*`、`object-*`、`accent-* caret-* fill-* stroke-*` (color)。

各ユーティリティの正確なプロパティと順序は `src/utilities.ts` を正とする。

---

## 8. バリアント

### 8.1 レジストリ

```
Variants {
  variants: Map<name, { kind: 'static'|'functional'|'compound', order: number, applyFn,
                        compounds: Compounds, compoundsWith: Compounds }>
  compareFns: Map<order, (a, z) => number>   // グループ内比較関数
}
Compounds = Never(0) | AtRules(1) | StyleRules(2)   // ビット集合
```

- `order` は登録順に 1 ずつ増える。`group(fn, compareFn)` で登録した複数のバリアントは同じ `order` を共有し、グループ内の順序は `compareFn` で決める。
- 既存名を再登録すると `kind` / `applyFn` / `compounds` だけ更新され `order` は保たれる (`@custom-variant` の予約 → 実体登録に使う)。
- `compounds`: このバリアントが生成するルールの種類。`compoundsWith`: 複合バリアントが子として受け入れる種類。
- `compoundsWith(parent, child)`: parent が compound であり、child.compounds ≠ Never、parent.compoundsWith ≠ Never、両者のビット積 ≠ 0 のとき true。child が arbitrary の場合は `compoundsForSelectors([selector])` で計算。

`compoundsForSelectors(selectors)`: `@` 始まりのセレクタが `@media` / `@supports` / `@container` 以外なら Never。`::` を含めば Never。それ以外は at-rule → AtRules、スタイル → StyleRules をビット OR。

### 8.2 `parseVariant(raw)`

1. `[...]` 形式 (任意バリアント): `[@media…&…]` (at-rule と `&` の混在) は無効。中身を decode、`isValidArbitrary`、非空。`>` `+` `~` 始まりなら `relative = true`。relative でなく `@` 始まりでもなく `&` を含まなければ `&:is(<sel>)` に包む。
2. `segment(raw, '/')` が 3 要素以上なら無効。`[name, modifier]`。
3. `findRoots(name, variants.has)` の各解釈について kind で分岐:
   - **static**: 値や modifier があれば無効。
   - **functional**: modifier をパース (文字列があるのに null なら無効)。値なし → `value: null`。`[x]` → arbitrary (decode/valid/非空)。`(--x)` → `var(--x)`。それ以外は `/^[a-zA-Z0-9_.%-]+$/` を満たす named (満たさなければ次の解釈へ)。`x-[…]` のように `[` で始まらないのに `]` で終わる値は次の解釈へ。
   - **compound**: 値必須。`not` / `has` / `in` は modifier を子に転送 (`not-group-hover/name`)。子を `parseVariant` (null なら無効)。`compoundsWith(root, child)` でなければ無効。

### 8.3 `applyVariant(node, variant)` — ルールノードを破壊的に書き換える

- **arbitrary**: depth 0 で relative なら無効。`node.nodes = [rule(selector, node.nodes)]`。
- **static / functional**: `applyFn(node, variant)`。null を返せば無効。
- **compound**: 空の `@slot` at-rule を作り、子バリアントを適用 (depth+1)。`not` の場合、結果の子が 2 つ以上なら無効。各子 (rule / at-rule 以外があれば無効) に `applyFn` を適用。最後に、空の rule / at-rule の `nodes` を元の `node.nodes` で埋め、`node.nodes` を結果で置き換える。

静的バリアントの標準実装 `staticVariant(name, selectors)`: `r.nodes = selectors.map(sel => rule(sel, r.nodes))`。

### 8.4 順序づけ (`getVariantOrder`)

パース済みの全バリアント (メモ化により同一文字列は同一オブジェクト) を `variants.compare` でソートし、比較結果が等しい隣接要素に同じインデックスを振る。候補のソートキーは `Σ 1 << index` (BigInt)。

`compare(a, z)`:
1. 同一なら 0。null は最小。
2. arbitrary 同士はセレクタの文字列比較。arbitrary は非 arbitrary より後。
3. `order` の差。
4. 両方 compound なら子バリアントを再帰比較、次に modifier の文字列比較 (modifier なしが先)。
5. `order` にグループ比較関数があればそれ。
6. root の文字列比較。
7. functional の値: null が先、arbitrary は named より後、値の文字列比較。

### 8.5 組み込みバリアント (登録順)

登録順がそのまま出力順になるので、この順序を守ること。

| 名前 | 生成 | compounds |
| --- | --- | --- |
| `*` | `:is(& > *)` | Never |
| `**` | `:is(& *)` | Never |
| `not-<v>` (compound, compoundsWith = StyleRules\|AtRules) | 子のセレクタ `&:hover` を `&:not(:hover)` に、`@media (q)` を `@media not all and (q)` に、`@supports (q)` を `@supports not (q)` に、`@container (q)` を `@container not (q)` に否定する。子が複数ルールを生成する場合 (例 `hover` はセレクタ + `@media`) は、それぞれ独立に否定した 2 ルールになる。詳細は `variants.ts` の `negateConditions` | StyleRules |
| `group-<v>[/name]` (compound, StyleRules) | 子の各ルールの `&` を `:where(.group)` (名前つきは `:where(.group\/name)`、プレフィックス時は `.tw\:group`) に置換し、複数なら `:is(...)` に包み、`&:is(<sel> *)` にする。ネストしたスタイルルールがあれば無効 | StyleRules |
| `peer-<v>[/name]` | 同上だが `&:is(:where(.peer) ~ *)` | StyleRules |
| `first-letter` / `first-line` | `&::first-letter` / `&::first-line` | Never |
| `marker` | `& *::marker`, `&::marker`, `& *::-webkit-details-marker`, `&::-webkit-details-marker` | Never |
| `selection` | `& *::selection`, `&::selection` | Never |
| `file` | `&::file-selector-button` | Never |
| `placeholder` | `&::placeholder` | Never |
| `backdrop` | `&::backdrop` | Never |
| `details-content` | `&::details-content` | Never |
| `before` / `after` | `&::before { @property --tw-content (syntax "*", initial-value "", inherits false) を AtRoot; content: var(--tw-content); ... }` | Never |
| `first last only odd even first-of-type last-of-type only-of-type` | `&:first-child` 等 | StyleRules |
| `visited target` | `&:visited` `&:target` | |
| `open` | `&:is([open], :popover-open, :open)` | |
| `default checked indeterminate placeholder-shown autofill optional required valid invalid user-valid user-invalid in-range out-of-range read-only` | `&:<name>` | |
| `empty focus-within` | | |
| `hover` | `&:hover { @media (hover: hover) { … } }` | StyleRules |
| `focus focus-visible active enabled disabled` | `&:<name>` | |
| `inert` | `&:is([inert], [inert] *)` | |
| `in-<v>` (compound, StyleRules) | 子ルールのセレクタの `&` を `*` に置換し `:where(<sel>) &` にする (`:where(*:hover) &`)。modifier 不可 | |
| `has-<v>` (compound, StyleRules) | `&:has(<child-sel with & → *>)`。modifier 不可 | |
| `aria-<v>` (functional) | named: `&[aria-<v>="true"]`、arbitrary: `&[aria-<v>]` (値は `=` の右辺を必要なら引用符で包む) | |
| `data-<v>` (functional) | `&[data-<v>]` (同上の引用) | |
| `nth-<n>` `nth-last-<n>` `nth-of-type-<n>` `nth-last-of-type-<n>` | `&:nth-child(n)` 等。named は非負整数のみ、arbitrary は任意 | |
| `supports-<v>` (functional) | `@supports (<v>)`。`x` (`:` なし) は `(x: var(--tw))`、`not(...)` 等の関数形式はそのまま (`and`/`or`/`not` の前後に空白を保証) | AtRules |
| `motion-safe motion-reduce` | `@media (prefers-reduced-motion: no-preference / reduce)` | AtRules |
| `contrast-more contrast-less` | `@media (prefers-contrast: more / less)` | AtRules |
| `max-<bp>` (グループ、降順) | `@media (width < <bp>)` | AtRules |
| `<bp>` (テーマ `--breakpoint-*` ごとに静的) と `min-<bp>` (同一グループ、昇順) | `@media (width >= <bp>)` | AtRules |
| `@max-<w>[/name]` (グループ、降順) | `@container [name] (width < <w>)` (`--container-*`) | AtRules |
| `@<w>` と `@min-<w>` (同一グループ、昇順) | `@container [name] (width >= <w>)` | AtRules |
| `portrait landscape` | `@media (orientation: …)` | AtRules |
| `ltr rtl` | `&:where(:dir(ltr), [dir="ltr"], [dir="ltr"] *)` | StyleRules |
| `dark` | `@media (prefers-color-scheme: dark)` (ユーザーは `@custom-variant dark (&:where(.dark, .dark *));` で上書きする) | AtRules |
| `starting` | `@starting-style` | Never |
| `print` | `@media print` | AtRules |
| `forced-colors inverted-colors` | `@media (forced-colors: active)` / `(inverted-colors: inverted)` | AtRules |
| `pointer-{none,coarse,fine} any-pointer-{…}` | `@media (pointer: …)` / `(any-pointer: …)` | AtRules |
| `noscript` | `@media (scripting: none)` | AtRules |

ブレークポイント系の値解決: static (`md`) は `--breakpoint-md` の生の値、functional (`min-[600px]`, `max-md`) は任意値または `--breakpoint-*` の生の値。`var(` を含む値は無効。グループ内比較は単位ごとにバケット分けして数値で比較 (`compareBreakpoints`)。

---

## 9. コンパイル (`compileCandidates`)

### 9.1 手順

1. 各生候補について: `invalidCandidates` に含まれれば無効。`parseCandidate` が空なら無効。
2. 各候補解釈について `compileAstNodes` (§9.2)。ルールが 1 つも生成されなければ無効候補として通知 (`invalidCandidates` に追加)。
3. 生成した各ルールノードに `{ propertySort, variantOrder, candidate }` を紐づける。
4. ソート (§9.3)。

### 9.2 `compileAstNodes(candidate, flags)`

1. `compileBaseUtility`: arbitrary 候補なら `[decl(property, asColor(value, modifier))]` (modifier は不透明度と仮定)。それ以外は `utilities.get(root)` の各定義を試す (§7.5 の戻り値規則。`options.types` に `any` を含む「フォールバック」定義は他が全て失敗した後に試す)。
2. 各 AST について: `propertySort` を計算 (§9.3)。候補が `important` か、`designSystem.important && RespectImportant フラグ` なら全宣言 (`AtRoot` 内を除く) を `important = true`。
3. `StyleRule { selector: '.' + escape(raw), nodes }` を作り、`candidate.variants` を順に `applyVariant`。1 つでも無効なら `[]`。
4. デザインシステム側のメモ化層で、生成 AST にテーマ関数置換と `@variant` 展開を行う。例外が出たら `[]`。

### 9.3 ソート

`propertySort` = `{ order: number[], count: number }`:
- AST を幅優先で辿り、値が定義済みの宣言を数える (`count`)。
- 各宣言のプロパティを `GLOBAL_PROPERTY_ORDER` (`src/property-order.ts` の配列。`container-type, pointer-events, visibility, position, inset, …, forced-color-adjust`) で引き、見つかった添字を集合に入れる。`--tw-sort: <prop>` 宣言があれば、その `<prop>` の添字だけを採用して以降の宣言は見ない。
- `order` は添字の昇順ソート。

比較 (`a`, `z`):
1. `variantOrder` (BigInt) の昇順。
2. `order` を先頭から比較し、最初に異なる添字の昇順 (無ければ ∞)。
3. `count` の降順 (宣言が多いものが先)。
4. 候補文字列の自然順比較 (数字列は数値として比較。`utils/compare.ts`)。

---

## 10. AST 最適化と出力 (`optimizeAst`)

生成済み AST 全体 (入力 CSS + ユーティリティ) に対して実行する。

### 10.1 変換パス

深さ優先で新しい AST を構築する。

- **宣言**: `--tw-sort` と値未定義は捨てる。`Context { theme }` 内の `--` 宣言は §6.6 の追跡対象 (値 `initial` は捨てる)。値に `var(` があれば使用変数を追跡 (テーマ内の `--` 宣言は依存関係として、それ以外は使用として)。`animation` 宣言はキーフレーム名を使用済みに。
- **ルール**: 子を変換し、空になったら捨てる。
- **`@property`** (深さ 0): 同じ名前は 1 回だけ出力。ポリフィル有効時はフォールバック宣言を収集 (§10.4)。
- **その他 at-rule**: 子を変換。空になった at-rule は捨てるが、`@layer` `@charset` `@custom-media` `@namespace` `@import` `@apply` は空でも残す。`@theme` 由来の `@keyframes` は削除候補として記録。
- **`AtRoot`**: 子を深さ 0 として変換し、`atRoots` に退避 (後でドキュメント末尾へ)。
- **`Context`**: `reference` なら子ごと捨てる。それ以外は子を同じ親に展開 (context 情報は継承)。
- **コメント**: そのまま。

### 10.2 後処理

1. 未使用テーマ変数の削除 (§6.6)。
2. 未使用 `@keyframes` の削除。
3. `atRoots` を末尾に連結。
4. ポリフィル (§10.4)。
5. ネスト展開 (§10.3)。

### 10.3 ネスト展開 (`handleNesting`)

出力は **ネストしていない標準 CSS** にする。

- ネストしたスタイルルール: 親セレクタで `&` を置換 (`&` を含まないネストは暗黙の子孫結合子ではなく `&` を先頭に補う)。親がセレクタリストなら `:is(...)` に包む。`&` だけのルールは子をそのまま親に展開。
- 宣言を持つルールの中にネストしたルールは、親ルールの **後** に兄弟として出力。
- スタイルルールの中の at-rule (`@media` 等) は外側へ巻き上げ、その中にスタイルルールを置く。at-rule 同士のネストは保持する (`@media a { @media b { … } }`)。
- 同じ親の中で同じ (セレクタ, プロパティ) が並ぶ場合の重複宣言除去や、`@scope` の特殊扱いは任意。

例:
```
.hover\:underline { &:hover { @media (hover: hover) { text-decoration-line: underline; } } }
→
@media (hover: hover) {
  .hover\:underline:hover {
    text-decoration-line: underline;
  }
}
```

### 10.4 ポリフィル (任意)

- **AtProperty**: 各 `@property` について `initial-value` (無ければ `initial`) の宣言を集め、`inherits: true` なら `:root, :host { … }`、それ以外は `*, ::before, ::after, ::backdrop { … }` にまとめ、`@layer properties { @supports ((-webkit-hyphens: none) and (not (margin-trim: inline))) or ((-moz-orient: inline) and (not (color:rgb(from red r g b)))) { … } }` として末尾に出力。ドキュメント先頭 (ライセンスコメント・`@charset`・外部 `@import` の後) に `@layer properties;` を挿入。
- **ColorMix**: `color-mix(…)` を含む宣言のうち `var(--x)` を参照するものを、変数を生の値に置き換えた宣言 (色空間を `srgb` に) と `@supports (color: color-mix(in lab, red, red)) { 元の宣言 }` の 2 つにする。

---

## 11. ソース走査と候補抽出

### 11.1 ソースの決定 (CLI)

```
sources = []
if compiler.root === 'none':   何も追加しない
if compiler.root === null:     { base: <cwd>, pattern: '**/*', negated: false }
else:                          { ...compiler.root, negated: false }
sources += compiler.sources                                  // @source 由来
sources += { base: dirname(execPath), pattern: basename(execPath), negated: true }  // 自分自身を除外
if input file:  sources += { base: dirname(input), pattern: basename(input), negated: false }
```

### 11.2 自動検出 (`**/*` パターンのソース)

`base` 以下を再帰的に歩く。以下を除外する:

- `.gitignore` (階層ごと) に一致するもの。
- ディレクトリ: `.git .hg .jj .next .parcel-cache .pnpm-store .svelte-kit .svn .turbo .venv .vercel .yarn __pycache__ node_modules venv`
- 拡張子: `less lock sass scss styl log`、およびバイナリ拡張子一覧 (`crates/oxide/src/scanner/fixtures/binary-extensions.txt`: 画像・音声・動画・アーカイブ・フォント・実行ファイル等)。
- ファイル名: `package-lock.json pnpm-lock.yaml bun.lockb .gitignore .env .env.*`
- `negated` ソースに一致するパス。

明示的な `@source "<glob>"` は上記の除外 (gitignore 含む) を **無視して** 一致ファイルを含める (例: `@source "../node_modules/my-lib"`)。glob は `**` `*` `{a,b}` をサポートする。

### 11.3 走査 API

```
Scanner.new(sources)
scan() -> string[]                  // 全ファイルを走査し、これまでに見つかった全候補 (重複なし) を返す。2 回目以降は mtime の変わったファイルだけ再読込
scanFiles(changed: {file, extension}[]) -> string[]   // 指定ファイルだけ走査し、新規候補だけを返す
scannedFiles                        // 直近の scan で読み込んだファイル
```

`.css` ファイルも走査対象に入る (入力 CSS 自身が `sources` に含まれるため `@apply` や `@source inline` の文字列が候補になる)。

### 11.4 候補抽出 (Oxide 相当の簡易版)

抽出器の精度は **再現率 (取りこぼさないこと) が重要で、適合率は重要ではない**。無効な候補は `parseCandidate` で捨てられ CSS を生成しないため、多少の誤検出は無害である。ただし候補数はキャッシュに影響するので、明らかなゴミ (単語や数値) はできるだけ弾く。

入力をバイト列として走査し、以下を満たす部分文字列を候補として切り出す。

**境界**: 候補の直前は「開始境界」、直後は「終了境界」でなければならない。
- 共通: 空白 (`\t \n \f \r ' '`)、引用符 (`" ' \``)、入力の端。
- 開始のみ: `.` `}` `>`
- 終了のみ: `]` `{` `=` `\` `<`

**候補の構造**: `(<variant>:)* <utility>`

- variant: `[...]` (括弧が釣り合う任意バリアント) または名前つき (`@` 始まり可、`a-zA-Z0-9_-`、`-[…]` / `-(…)` の任意値、`/modifier`) の後に `:`。
- utility: `[prop:value]` (任意プロパティ) または名前つき。先頭は `a-zA-Z` `@` または `-` (直後が英数字。`-@` は不可)。以降 `a-zA-Z0-9_-` と、`-[…]` (釣り合う括弧)、`-(…)`、`.` (数字に挟まれる: `2.5`)、`%`、末尾の `/modifier` (`/[…]`, `/(…)`, `/名前`)、末尾の `!`。`-` `_` は末尾に来られない。単独の `!` 始まり (旧 important) も許容。
- CSS 変数 `--[a-zA-Z0-9_-]+` (境界条件は同じ) も候補として抽出する (§2.2 の使用申告)。
- 抽出後 `has_valid_boundaries` を満たさないものは捨てる。

拡張子ごとのプリプロセッサ (Vue の `<style>` 除去など) は本仕様では実装しない。

---

## 12. CLI

### 12.1 コマンドとフラグ

```
tailwindcss [build] [--input input.css] [--output output.css] [--watch] [--poll=ms] [options…]
```

| フラグ | 型 | 既定 | 意味 |
| --- | --- | --- | --- |
| `-i, --input <path>` | string | なし | 入力 CSS。`-` で stdin。省略時は `@import 'tailwindcss';` を入力とする |
| `-o, --output <path>` | string | `-` | 出力先。`-` で stdout |
| `-w, --watch [always]` | boolean \| `always` | false | 監視モード。`always` は stdin が閉じても継続 |
| `--poll [ms]` | boolean \| number | false | ファイルシステムイベントの代わりにポーリング (既定 250ms)。0 以下はエラー |
| `-m, --minify` | boolean | false | 最適化 + 最小化 |
| `--optimize` | boolean | false | 最小化なしの最適化 |
| `--cwd <dir>` | string | `.` | 基準ディレクトリ |
| `--map [path]` | boolean \| string | false | ソースマップ (本仕様では未対応と明示してよい) |
| `--silent` | boolean | false | エラー以外の出力を抑制 |
| `-h, --help` | | | ヘルプ |

- 入力パスが存在しなければエラー終了。入力と出力が同じパスならエラー終了。
- 引数なしで TTY ならヘルプを表示。
- バナー (`≈ tailwindcss v4.x.y`) と `Done in 12ms` は **stderr** に出す (`--silent` で抑制)。生成 CSS を stdout に出す場合、内容が前回と同じなら再出力しない。

### 12.2 初回ビルド

1. 入力 CSS を読み、`compile(css, { base: dirname(input) or cwd, loadStylesheet })` する。`loadStylesheet` は解決したファイルパスを「フルリビルド対象」に登録する。
2. §11.1 でソースを決め、`Scanner` を作る。
3. `scanner.scan()` → `compiler.build(candidates)` → 書き出し (§12.5)。

### 12.3 watch (イベント駆動)

走査対象ディレクトリ (ソースの `base` の集合) を再帰監視する。変更ファイル群を受け取ったら:

- 変更が出力ファイルだけなら無視 (無限ループ防止)。
- 変更ファイルにフルリビルド対象 (入力 CSS と、それが `@import` したファイル) が含まれれば **フルリビルド**: 入力を読み直し、コンパイラとスキャナを作り直し、`scan()` → `build()`。監視対象も作り直す。
- そうでなければ **増分ビルド**: `scanner.scanFiles(changed)` で新規候補を得る。空なら何もしない。`compiler.build(newCandidates)` → 書き出し。
- 例外は stderr に表示して監視を続ける。フルリビルド失敗時はフルリビルド対象の一覧を失敗前のものに戻す (削除→復元を検知できるようにするため)。
- `--watch=always` でなければ stdin の EOF で終了する。

### 12.4 watch (ポーリング)

`--poll` 指定時は一定間隔で `scanner.scan()` を呼び、`scannedFiles` (出力ファイルとマップを除く) が空でなければ §12.3 と同じ判定 (フル or 増分) でビルドする。増分では `scan()` が返した新規候補をそのまま `build` に渡す。

### 12.5 書き出し

1. `--minify` / `--optimize` なら最適化器を通す (CSS が前回と同じなら前回の結果を再利用)。参照実装は Lightning CSS (nesting とメディアクエリ範囲構文のダウンレベル、targets: Safari 16.4 / iOS 16.4 / Firefox 128 / Chrome 111)。移植版は任意の最小化器、または「空白除去のみ」でもよい。
2. `--output` がパスならファイルに書く (ディレクトリは作成)。`-` なら変更があったときだけ stdout に出す。

---

## 13. ゴールデン例 (最適化器なしの生出力)

以下は `compile` → `build` の出力 (`optimize` を通さない) の期待値。空行と字下げは §3.3 に従う。

### 13.1 基本

入力 CSS:
```css
@theme {
  --color-black: #000;
  --breakpoint-md: 768px;
}
@layer utilities {
  @tailwind utilities;
}
```
候補: `flex md:grid hover:underline dark:bg-black`

出力:
```css
:root, :host {
  --color-black: #000;
}
@layer utilities {
  .flex {
    display: flex;
  }
  @media (hover: hover) {
    .hover\:underline:hover {
      text-decoration-line: underline;
    }
  }
  @media (width >= 768px) {
    .md\:grid {
      display: grid;
    }
  }
  @media (prefers-color-scheme: dark) {
    .dark\:bg-black {
      background-color: var(--color-black);
    }
  }
}
```

`--breakpoint-md` は変数として参照されない (`@media` に生の値が入る) ため出力されない。

### 13.2 値の種類

| 候補 | 生成宣言 (テーマは同梱 `theme.css`) |
| --- | --- |
| `p-4` | `padding: calc(var(--spacing) * 4);` |
| `p-1` | `padding: var(--spacing);` |
| `p-0` | `padding: 0px;` |
| `p-px` | `padding: 1px;` |
| `-mt-2` | `margin-top: calc(var(--spacing) * -2);` |
| `w-1/2` | `width: calc(1 / 2 * 100%);` |
| `w-[13px]` | `width: 13px;` |
| `w-(--my-w)` | `width: var(--my-w);` |
| `max-w-md` | `max-width: var(--container-md);` |
| `bg-red-500` | `background-color: var(--color-red-500);` |
| `bg-red-500/50` | `background-color: color-mix(in oklab, var(--color-red-500) 50%, transparent);` |
| `bg-[#0088cc]` | `background-color: #0088cc;` |
| `bg-[url(/a_b.png)]` | `background-image: url(/a_b.png);` |
| `bg-[length:10px_20px]` | `background-size: 10px 20px;` |
| `text-lg` | `font-size: var(--text-lg); line-height: var(--tw-leading, var(--text-lg--line-height));` |
| `text-lg/8` | `font-size: var(--text-lg); line-height: calc(var(--spacing) * 8);` |
| `text-red-500` | `color: var(--color-red-500);` |
| `font-bold` | `@property --tw-font-weight {…}` (ルートへ) + `--tw-font-weight: var(--font-weight-bold); font-weight: var(--font-weight-bold);` |
| `rounded` | `border-radius: var(--radius);` (未定義なら無効) / `rounded-lg` → `var(--radius-lg)` |
| `rounded-full` | `border-radius: calc(infinity * 1px);` |
| `border` | `border-style: var(--tw-border-style); border-width: 1px;` + `@property --tw-border-style { syntax: "*"; inherits: false; initial-value: solid; }` |
| `border-2` | 同上で `border-width: 2px;` |
| `z-10` | `z-index: 10;` / `-z-10` → `z-index: calc(10 * -1);` |
| `opacity-50` | `opacity: 50%;` |
| `[mask-type:luminance]` | `mask-type: luminance;` |
| `[--my-var:1px]` | `--my-var: 1px;` |
| `underline!` / `!underline` | `text-decoration-line: underline !important;` |

### 13.3 バリアント

| 候補 | 出力セレクタ / 包み |
| --- | --- |
| `hover:flex` | `@media (hover: hover) { .hover\:flex:hover { … } }` |
| `focus:flex` | `.focus\:flex:focus` |
| `sm:flex` | `@media (width >= 40rem) { .sm\:flex { … } }` |
| `max-md:flex` | `@media (width < 48rem) { … }` |
| `min-[600px]:flex` | `@media (width >= 600px) { … }` |
| `@md:flex` | `@container (width >= 28rem) { … }` |
| `@md/main:flex` | `@container main (width >= 28rem) { … }` |
| `group-hover:flex` | `@media (hover: hover) { .group-hover\:flex:is(:where(.group):hover *) { … } }` |
| `group-hover/item:flex` | `.group-hover\/item\:flex:is(:where(.group\/item):hover *)` |
| `peer-checked:flex` | `.peer-checked\:flex:is(:where(.peer):checked ~ *)` |
| `has-[>img]:flex` | `.has-\[\>img\]\:flex:has(> img)` (任意バリアント `[>img]` 単独では無効、`has-` の子としてのみ有効) |
| `not-hover:flex` | `.not-hover\:flex:not(:hover)` と `@media not all and (hover: hover) { .not-hover\:flex { … } }` の 2 ルール |
| `not-supports-grid:flex` | `@supports not (grid: var(--tw)) { … }` |
| `in-data-visible:flex` | `:where([data-visible]) .in-data-visible\:flex` |
| `data-[state=open]:flex` | `.data-\[state\=open\]\:flex[data-state="open"]` |
| `aria-checked:flex` | `.aria-checked\:flex[aria-checked="true"]` |
| `nth-3:flex` | `.nth-3\:flex:nth-child(3)` |
| `[&_p]:flex` | `.\[\&_p\]\:flex p` |
| `[@media(width>=100px)]:flex` | `@media (width>=100px) { … }` |
| `*:flex` | `:is(.\*\:flex > *)` |
| `before:block` | `.before\:block::before { content: var(--tw-content); display: block; }` + `@property --tw-content` |
| `dark:hover:flex` | `@media (prefers-color-scheme: dark) { @media (hover: hover) { .dark\:hover\:flex:hover { … } } }` |

出力順は `variantOrder` によって決まり、同じ候補集合なら常に同じ順になる (例: `sm:` より `md:` が後、バリアント無しが先頭)。

### 13.4 `@utility` と `@apply`

```css
@utility tab-* {
  tab-size: --value(integer);
  tab-size: --value(--tab-size-*);
  tab-size: --value([integer]);
}
@utility content-auto { content-visibility: auto; }
.btn { @apply rounded-lg px-4 py-2 hover:bg-red-500; }
```
候補 `tab-4 tab-[8] content-auto` →
```css
.btn {
  border-radius: var(--radius-lg);
  padding-inline: calc(var(--spacing) * 4);
  padding-block: calc(var(--spacing) * 2);
}
@media (hover: hover) {
  .btn:hover {
    background-color: var(--color-red-500);
  }
}
.content-auto {
  content-visibility: auto;
}
.tab-4 {
  tab-size: 4;
}
.tab-\[8\] {
  tab-size: 8;
}
```

### 13.5 テーマ操作

```css
@import "tailwindcss";
@theme {
  --color-*: initial;            /* デフォルトの色を全部消す */
  --color-primary: oklch(0.6 0.2 250);
  --breakpoint-3xl: 120rem;      /* 3xl: バリアントが増える */
  --font-display: "Inter", sans-serif;
}
@custom-variant dark (&:where(.dark, .dark *));
```

- `bg-primary`、`3xl:flex`、`font-display` が使える。`bg-red-500` は無効になる。
- `dark:` はメディアクエリではなくクラスセレクタになる。

---

## 14. ユーティリティ関数 (共通部品)

### 14.1 `segment(input, sep)`

区切り文字で分割するが、`(…)` `[…]` `{…}`、引用符の内側、`\` エスケープ直後の文字は無視する。閉じ括弧はスタック先頭と一致するときだけポップする。

### 14.2 `escape(ident)` / `unescape`

`CSS.escape` と同等。先頭の数字、`-` の後の数字、制御文字は `\<hex> ` 形式、英数字 `-` `_` と非 ASCII はそのまま、それ以外は `\` を前置。

### 14.3 `isValidArbitrary(value)`

括弧 (`(` `[`) の対応を追跡し、対応しない閉じ括弧 (`)` `]` `}`)、トップレベルの `;` があれば false。`{` はスタックに積まない (トップレベルの `}` は false)。引用符内・エスケープはスキップ。

### 14.4 数値述語

- `isPositiveInteger(v)`: `Number(v)` が整数かつ `>= 0` かつ `String(Number(v)) === v` (先頭ゼロ等を拒否)。
- `isStrictPositiveInteger`: `> 0`。
- `isValidSpacingMultiplier(v)` / `isValidOpacityValue(v)`: 0.25 の倍数で、余分な先頭・末尾ゼロが無い。

### 14.5 `compare(a, z)`

文字列を先頭から比較し、両方が数字列の位置では数値として比較する自然順比較。

---

## 15. 実装順序の提案

1. §3 AST とパーサ、§3.3 `toCss`、§14 部品。
2. §6 テーマ、§4.3 `@theme`、§4.1 `@import` (同梱 CSS の読み込み)。
3. §7 候補パース (静的・関数的・任意値・modifier)、§7.5 ヘルパー、§7.9 のうち spacing / color / 静的レイアウト。
4. §9 コンパイルとソート、§10 最適化 (未使用変数削除、ネスト展開)。ここで §13.1〜13.2 が通る。
5. §8 バリアント (静的 → メディア → functional → compound)。§13.3 が通る。
6. §4.5〜4.10 (`@custom-variant`、`@utility`、`@variant`、`@apply`、テーマ関数)。§13.4〜13.5 が通る。
7. §11 走査・抽出、§12 CLI (単発ビルド → watch → poll → minify)。
8. 残りのユーティリティを `src/utilities.ts` の順に追加。ポリフィルは最後。

テストは参照実装のテスト (`src/*.test.ts`、特に `index.test.ts`、`candidate.test.ts`、`variants.test.ts`、`utilities.test.ts`、`apply.test.ts`、`css-functions.test.ts`) の入力と期待値をそのまま移植できる。ただし `run()` ヘルパーは Lightning CSS で整形しているため、期待値は `@media (width >= …)` が `(min-width: …)` に変換される等の差分を考慮すること。
