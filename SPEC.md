# Loom Compiler Specification

Status: Draft v1 (language-agnostic)

Purpose: Define a compiler that turns a stylesheet plus a set of class-name candidates into a
complete CSS document, and a command-line tool that drives that compiler from the filesystem.

## Normative Language

The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHOULD`, `SHOULD NOT`, `RECOMMENDED`, `MAY`, and
`OPTIONAL` in this document are to be interpreted as described in RFC 2119.

`Implementation-defined` means the behavior is part of the implementation contract, but this
specification does not prescribe one universal policy. Implementations MUST document the selected
behavior.

## 1. Problem Statement

Loom is a utility-first CSS compiler. Authors write markup that uses short, composable class names
such as `flex`, `p-4`, `bg-red-500/50`, or `md:hover:underline`. Loom reads one entry stylesheet,
scans the project's source files for class names that look like utilities, and emits only the CSS
rules that those class names require.

The compiler solves four problems:

- It generates CSS on demand from a compact class-name grammar, so the output contains only rules
  that are actually used.
- It keeps design tokens (colors, spacing, breakpoints, fonts) as CSS custom properties declared in
  the stylesheet itself, so the theme is versioned with the project and readable by the browser.
- It lets authors extend the grammar with new utilities and variants using plain CSS directives.
- It supports an incremental development loop: candidates only accumulate, so a watch process can
  rebuild by scanning only the files that changed.

Important boundary:

- Loom reads source files but never modifies them.
- Loom's output is deterministic: the same stylesheet and the same candidate set always produce the
  same CSS, byte for byte, regardless of the order in which candidates were discovered.
- Class names that do not resolve to a utility produce no output and no error. Only the stylesheet
  itself can produce compile errors.

## 2. Goals

- Parse a stylesheet into an AST that preserves nesting, at-rules, and license comments.
- Resolve `@import` recursively through a caller-supplied loader.
- Collect theme tokens from `@theme` blocks and expose them to utilities as `var(...)` references.
- Parse candidate class names into a structured form covering variants, roots, named values,
  arbitrary values, variable shorthands, modifiers, negation, and importance.
- Compile candidates into CSS rules using a registry of built-in utilities and variants.
- Allow stylesheet-defined utilities (`@utility`), variants (`@custom-variant`), inline variant
  blocks (`@variant`), and utility inlining (`@apply`).
- Order generated rules deterministically by variant, then by property, then by name.
- Serialize flat, non-nested CSS with stable formatting.
- Remove theme variables and keyframes that nothing references.
- Discover source files automatically, honor `@source` directives, and extract candidates from
  arbitrary text with high recall.
- Provide a CLI with single-shot, watch, and polling modes and optional minification.

## 3. System Overview

### 3.1 Main Components

1. `Stylesheet Parser`
   - Parses CSS text into the AST defined in Section 4.1.1.
   - Reports syntax errors with source positions.

2. `Import Resolver`
   - Expands `@import` and `@reference` using a loader callback.
   - Wraps imported content in `@layer`, `@media`, and `@supports` as requested.

3. `Directive Collector`
   - Walks the AST once and registers `@theme`, `@source`, `@utility`, `@custom-variant`,
     `@variant`, and `@loom utilities`.
   - Removes directives that must not appear in the output.

4. `Theme`
   - Stores design tokens keyed by custom-property name.
   - Resolves candidate values to `var(...)` references or inline values by namespace.

5. `Design System`
   - Owns the theme, the utility registry, the variant registry, memoization caches, and the set
     of known-invalid candidates.

6. `Candidate Parser`
   - Converts a raw class-name string into zero or more structured `Candidate` interpretations.

7. `Utility Registry` and `Variant Registry`
   - Map roots to compile functions (utilities) and to selector or at-rule transforms (variants).

8. `Compiler`
   - Compiles candidates into rule nodes, applies variants and importance, and sorts the result.

9. `Optimizer` and `Serializer`
   - Deduplicates property registrations, hoists root-level nodes, prunes unused theme values,
     flattens nesting, and prints CSS text.

10. `Scanner`
    - Enumerates source files from globs and auto-detection rules, extracts candidates, and tracks
      file modification times for incremental scans.

11. `CLI`
    - Wires the scanner and the compiler to files, stdin, stdout, watchers, and a minifier.

### 3.2 Abstraction Levels

1. `Stylesheet Layer` (author-defined)
   - The entry stylesheet, its imports, theme tokens, and custom utilities and variants.

2. `Grammar Layer`
   - Candidate syntax, variant syntax, value decoding, and validity rules.

3. `Registry Layer`
   - Built-in utilities and variants, plus those registered from the stylesheet.

4. `Compilation Layer`
   - Candidate to AST, variant application, ordering, optimization, serialization.

5. `Discovery Layer`
   - File walking, ignore rules, candidate extraction, incremental scanning.

6. `Host Layer`
   - CLI argument handling, file IO, watching, polling, minification.

### 3.3 External Dependencies

- A local filesystem for reading the entry stylesheet, bundled stylesheets, and source files.
- OPTIONAL filesystem event notification for watch mode (polling is the fallback).
- OPTIONAL CSS minifier for `--minify` and `--optimize`.

## 4. Core Domain Model

### 4.1 Entities

#### 4.1.1 AST Node

The compiler operates on a list of nodes. Every node has a `kind`:

- `rule`
  - `selector` (string)
  - `nodes` (list of nodes)
- `at-rule`
  - `name` (string, including the leading `@`, for example `@media`)
  - `params` (string, possibly empty)
  - `nodes` (list of nodes; empty for statement at-rules such as `@import`)
- `declaration`
  - `property` (string)
  - `value` (string or undefined; undefined declarations are never printed)
  - `important` (boolean)
- `comment`
  - `value` (string, the text between `/*` and `*/`)
- `context`
  - `context` (map of string to string or boolean)
  - `nodes` (list of nodes)
  - Never printed. Carries metadata such as `base`, `reference`, `theme`, `source`, and
    `sourceBase` to its subtree.
- `at-root`
  - `nodes` (list of nodes)
  - Never printed in place. Its children are hoisted to the end of the document during
    optimization (Section 12).

The helper `rule(selector, nodes)` MUST create an `at-rule` when `selector` starts with `@`
(splitting on the first whitespace into `name` and `params`) and a `rule` otherwise.

#### 4.1.2 Theme Entry

- `key` (string, a custom-property name such as `--color-red-500`)
- `value` (string)
- `options` (bit set)
  - `INLINE` (1): consumers embed the raw value instead of `var(...)`.
  - `REFERENCE` (2): the variable is not printed; consumers embed `var(key, value)`.
  - `DEFAULT` (4): a later non-default entry with the same key wins even if it was added first.
  - `STATIC` (8): the variable is printed even when unused.
  - `USED` (16): something referenced the variable.

#### 4.1.3 Candidate

A parsed class name.

- `kind` (`static`, `functional`, or `arbitrary`)
- `raw` (string, the original class name including variants and the important marker)
- `variants` (list of `Variant`, in application order: the rightmost variant in the source text is
  first)
- `important` (boolean)
- For `static`: `root` (string)
- For `functional`: `root` (string), `value` (`Value` or null), `modifier` (`Modifier` or null)
- For `arbitrary`: `property` (string), `value` (string), `modifier` (`Modifier` or null)

#### 4.1.4 Value

- `named`
  - `value` (string, for example `red-500`)
  - `fraction` (string or null, for example `1/2` when a slash segment could be a fraction)
- `arbitrary`
  - `value` (string, decoded)
  - `dataType` (string or null, an explicit type hint such as `color`)

#### 4.1.5 Modifier

- `named`
  - `value` (string, for example `50`)
- `arbitrary`
  - `value` (string, decoded; variable shorthands are stored as `var(--name)`)

#### 4.1.6 Variant

- `static`
  - `root` (string)
- `functional`
  - `root` (string)
  - `value` (`{ kind: named | arbitrary, value }` or null)
  - `modifier` (`Modifier` or null)
- `compound`
  - `root` (string)
  - `modifier` (`Modifier` or null)
  - `variant` (`Variant`, the inner variant)
- `arbitrary`
  - `selector` (string)
  - `relative` (boolean, true when the selector starts with `>`, `+`, or `~`)

#### 4.1.7 Utility Definition

- `kind` (`static` or `functional`)
- `compile` (function from `Candidate` to one of: a list of nodes, `undefined`, or `null`)
  - A list of nodes means success.
  - `undefined` means this definition does not handle the candidate; try the next definition.
  - `null` means the candidate is invalid for this definition. When the definition declares
    `types`, the compiler MUST stop trying further definitions for this root.
- `types` (OPTIONAL list of data-type names; a definition whose `types` has more than one entry and
  includes `any` is a fallback definition tried only after all others fail)

Several definitions MAY share one root.

#### 4.1.8 Variant Definition

- `kind` (`static`, `functional`, or `compound`)
- `order` (integer, registration order)
- `apply` (function that mutates a rule node in place, or returns `null` to reject)
- `compounds` (bit set: `NEVER` = 0, `AT_RULES` = 1, `STYLE_RULES` = 2; the kinds of rules this
  variant generates)
- `compoundsWith` (bit set; the kinds of inner rules a compound variant accepts)

#### 4.1.9 Design System

- `theme` (`Theme`)
- `utilities` (map from root to list of `Utility Definition`)
- `variants` (map from name to `Variant Definition`, plus per-order comparison functions)
- `invalidCandidates` (set of raw strings)
- `important` (boolean; when true every generated declaration is marked `!important`)
- Memoized operations: `parseCandidate(raw)`, `parseVariant(raw)`, `compileAstNodes(candidate,
  flags)`, `getVariantOrder()`.
- `parseVariant` MUST return the same object for the same input string within one design system,
  because variant ordering (Section 9.4) relies on identity.

#### 4.1.10 Source Entry

- `base` (absolute directory path)
- `pattern` (glob relative to `base`)
- `negated` (boolean)

#### 4.1.11 Compiler Handle

Returned by `compile` (Section 15.1).

- `sources` (list of `Source Entry` collected from `@source`)
- `root` (`null`, the string `none`, or a `Source Entry` without `negated`; from
  `@loom utilities source(...)`)
- `features` (bit set: `AT_APPLY` = 1, `AT_IMPORT` = 2, `THEME_FUNCTION` = 8, `UTILITIES` = 16,
  `VARIANTS` = 32, `AT_THEME` = 64)
- `build(candidates)` (function from a list of raw strings to CSS text)

### 4.2 Stable Identifiers and Normalization Rules

- `Class selector`
  - The selector for a candidate is `.` followed by the CSS-escaped `raw` string. Escaping MUST
    match `CSS.escape`: a leading digit, a digit after a leading `-`, and control characters
    become `\<hex> `; ASCII letters, digits, `-`, `_`, and non-ASCII pass through; everything
    else is prefixed with `\`. A lone `-` becomes `\-`.
- `segment(input, separator)`
  - Splits on a single-character separator, ignoring separators inside `(...)`, `[...]`, `{...}`,
    inside single or double quotes, and immediately after a backslash. A closing bracket pops the
    stack only when it matches the most recent opener.
- `decodeArbitraryValue(input)`
  - When `input` contains no `(`: replace `\_` with `_` and every other `_` with a space.
  - Otherwise parse the value into words, separators, and function calls. Inside `url(...)` (and
    any function whose name ends in `_url`) nothing is replaced. Inside `var(...)` and
    `theme(...)` the first argument keeps its underscores (only `\_` is unescaped); other arguments
    are decoded recursively. Everywhere else `_` becomes a space. Finally insert spaces around
    `+`, `-`, `*`, and `/` inside math functions (`calc`, `min`, `max`, `clamp`, and similar) when
    they act as operators, so `calc(1px+2px)` becomes `calc(1px + 2px)`.
- `isValidArbitrary(input)`
  - Track `(` and `[` on a stack. Return false on a closing `)`, `]`, or `}` with an empty stack,
    on a mismatched closer, or on a top-level `;`. Quoted text and backslash-escaped characters are
    skipped. `{` is not pushed.
- `Named value pattern`
  - Named values and named modifiers MUST match `^[a-zA-Z0-9_.%-]+$`.
- `Variant name pattern`
  - Custom variant names MUST match `^@?[a-z0-9][a-zA-Z0-9_-]*` and MUST NOT end in `_` or `-`.
- `Prefix pattern`
  - A theme prefix MUST match `^[a-z]+$`.
- `Numeric predicates`
  - `isPositiveInteger(v)`: `Number(v)` is an integer, is greater than or equal to 0, and
    `String(Number(v)) == v`.
  - `isStrictPositiveInteger(v)`: as above with greater than 0.
  - `isMultipleOfQuarter(v)`: `Number(v)` is a multiple of 0.25 with no redundant leading or
    trailing zeros. Used for spacing multipliers and opacity values.
- `compare(a, z)`
  - Natural string comparison: compare character by character, but when both strings have a digit
    at the current position, compare the full digit runs numerically (then lexically on tie).
- `Brace expansion`
  - `{a,b,c}` enumerates; `{1..5}`, `{10..0}`, and `{0..20..5}` produce integer ranges (negative
    bounds allowed, a step of zero is an error); nesting is allowed; unbalanced braces are an
    error.

## 5. Stylesheet Input Contract

### 5.1 Parsing Requirements

- The parser MUST accept standard CSS including nested rules, nested at-rules, the `&` nesting
  selector, and custom properties.
- The parser MUST drop ordinary comments and MUST keep comments that start with `/*!` as `comment`
  nodes.
- The parser MUST split `!important` from a declaration value and set `important`.
- The parser MUST accept a missing `;` before `}`.
- `;`, `{`, and `}` inside quotes or parentheses MUST NOT be treated as delimiters.
- Malformed input (for example an unclosed block) MUST raise a syntax error carrying the source
  position.
- Statement at-rules (those terminated by `;`) MUST produce `at-rule` nodes with empty `nodes`.

### 5.2 Serialization Format

The serializer prints nodes recursively with two-space indentation per depth and `\n` after every
line:

- Declaration: `<indent><property>: <value>;` with ` !important` inserted before `;` when
  `important` is true. Declarations whose `value` is undefined are skipped.
- Rule: `<indent><selector> {`, the children at depth + 1, then `<indent>}`.
- At-rule with children: `<indent><name> <params> {` (or `<indent><name> {` when `params` is
  empty), the children, then `<indent>}`.
- At-rule without children: `<indent><name> <params>;`.
- Comment: `<indent>/*<value>*/`.
- Context: children printed at the same depth.

## 6. Directive Processing

### 6.1 Processing Order

`compile` wraps the parsed AST in a `context` node carrying `base` and then performs these steps in
order:

1. Resolve `@import` and `@reference` (Section 6.2).
2. Walk the AST once and collect directives (Sections 6.4 through 6.8).
3. Build the design system from the theme. Apply `important` if requested. Add every
   `@source not inline(...)` candidate to `invalidCandidates`.
4. Reserve every custom variant name in stylesheet order, then register custom variants in
   topological order of their `@variant` dependencies (Section 6.7).
5. Register custom utilities (Section 6.8).
6. Replace the first `@theme` with `:root, :host { ... }` containing every theme variable that is
   not `REFERENCE` (Section 7.5). Hoist theme keyframes to the document root.
7. Expand nested `@variant` blocks (Section 6.9).
8. Substitute theme functions (Section 6.11).
9. Expand `@apply` (Section 6.10).
10. Convert the `@loom utilities` node into an empty `context` node; `build` fills it later.
11. Remove any remaining `@utility` nodes.

### 6.2 `@import` and `@reference`

Syntax: `@import "<uri>" [layer(<name>)] [supports(<condition>)] [<media query list>];`

- The URI MUST be quoted. `url(...)` imports, `data:` URIs, and `http://` or `https://` URIs MUST
  be left untouched in the output.
- The loader callback receives `(uri, base)` and returns `{ path, base, content }`. The loaded
  content is parsed and its own imports are resolved recursively. Recursion deeper than 100 levels
  MUST raise an error.
- The loaded AST is wrapped in `context { base: <loaded base> }`, then in `@layer <name>` when
  `layer(...)` is present, then in `@media <query>` when a media query list is present, then in
  `@supports (<condition>)` when `supports(...)` is present.
- `layer(...)` MUST appear before `supports(...)` and before media queries; otherwise raise an
  error.
- `@reference "<uri>";` is equivalent to `@import "<uri>" reference;`.
- `AT_IMPORT` is added to `features` whenever an import is resolved.

Media-position parameters recognized after import expansion (the resolver turns them into
`@media <params> { ... }`; the collector interprets them and removes the ones it consumes; when
every parameter is consumed the `@media` wrapper is removed and its children are spliced in place):

- `reference`: wrap the children in `context { reference: true }`.
- `theme(<options>)`: append `<options>` to the `params` of every `@theme` inside. When the options
  include `reference`, any non-`@theme` rule inside MUST raise an error.
- `prefix(<ident>)`: append `prefix(<ident>)` to every `@theme` inside.
- `important`: set the design system's `important` flag.
- `source(<path>)`: rewrite the first `@loom utilities` inside to `@loom utilities source(<path>)`
  and wrap it in `context { sourceBase: <base of the importing file> }`.

### 6.3 Bundled Stylesheets

An implementation MUST ship four stylesheets and MUST resolve the import id `loom` to the entry
file among them:

- `index.css`: `@layer theme, base, components, utilities;` followed by `@import './theme.css'
  layer(theme);`, `@import './preflight.css' layer(base);`, and `@import './utilities.css'
  layer(utilities);`.
- `theme.css`: a single `@theme default { ... }` block declaring the default color palette,
  `--spacing`, breakpoints, container widths, font families, text sizes with `--line-height`
  sub-keys, font weights, tracking, leading, radii, shadows, easing curves, animations with their
  `@keyframes`, blur values, and `--default-*` settings.
- `preflight.css`: base element resets.
- `utilities.css`: the single statement `@loom utilities;`.

The default theme values that this specification's examples depend on are:

- `--spacing: 0.25rem`
- `--breakpoint-sm: 40rem`, `--breakpoint-md: 48rem`, `--breakpoint-lg: 64rem`,
  `--breakpoint-xl: 80rem`, `--breakpoint-2xl: 96rem`
- `--container-md: 28rem` (and the other container sizes from `3xs` to `7xl`)
- `--text-lg: 1.125rem` with `--text-lg--line-height: calc(1.75 / 1.125)`
- `--radius-lg: 0.5rem`
- `--font-weight-bold: 700`
- `--color-red-500: oklch(63.7% 0.237 25.331)`
- `--default-transition-duration: 150ms`
- `--default-transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1)`

### 6.4 `@loom utilities [source(...)]`

- Only the first occurrence is kept; later occurrences MUST be removed.
- An occurrence inside `context { reference: true }` MUST be removed and ignored.
- `source(none)` sets `root` to `none` (auto-detection disabled).
- `source("<path>")` sets `root` to `{ base: sourceBase or base, pattern: <path> }`. An unquoted
  path MUST raise an error.
- `UTILITIES` is added to `features`.

### 6.5 `@theme [options] { ... }`

- `params` is a whitespace-separated list of `reference`, `inline`, `default`, `static`, and
  `prefix(<ident>)`.
- A `@theme` inside `context { reference: true }` is treated as `reference`.
- An invalid prefix MUST raise an error. A valid prefix is stored on the theme.
- Children MUST be custom-property declarations, `@keyframes`, or comments. Anything else MUST
  raise an error that includes a snippet of the offending block.
- Declarations are registered with `theme.add(unescape(property), value, options)`
  (Section 7.2). `@keyframes` blocks are stored on the theme.
- The first `@theme` is replaced with an empty `:root, :host` rule (filled in step 6 of
  Section 6.1). Later `@theme` blocks are removed.
- `AT_THEME` is added to `features`.

### 6.6 `@source`

Forms:

- `@source "<glob>";` adds `{ base, pattern, negated: false }` to `sources`, where `base` is the
  `base` of the enclosing `context`.
- `@source not "<glob>";` adds the same entry with `negated: true`.
- `@source inline("<patterns>");` splits the quoted text on whitespace, brace-expands each item, and
  appends the results to the inline candidate list. Inline candidates are included in every
  `build` as if they had been scanned.
- `@source not inline("<patterns>");` does the same but adds the results to `invalidCandidates`.

A `@source` with a body, a nested `@source`, or an unquoted path MUST raise an error. The
directive is removed from the output.

### 6.7 `@custom-variant`

Both forms MUST be top level (nesting is an error), and the name MUST match the variant name
pattern. A definition with both a selector and a body, or with neither, MUST raise an error.

Selector form: `@custom-variant <name> (<selector>[, <selector>...]);`

- The parenthesized text is split on commas with `segment`. Empty items MUST raise an error.
- Items starting with `@` are at-rule selectors; the rest are style selectors.
- At application time, the variant produces one `rule` whose selector joins the style selectors
  with `, ` (when any exist) followed by one `at-rule` per at-rule item, each receiving the
  utility's nodes as children.
- `compounds` is computed with `compoundsForSelectors` (Section 9.1).

Body form: `@custom-variant <name> { ... @slot; ... }`

- At application time the body is cloned, every `@slot` is replaced by the utility's nodes, and
  `@keyframes` and `@property` inside the body are wrapped in `at-root`.
- Nested `@variant <other>` inside the body is allowed. The set of referenced names forms the
  variant's dependencies; registration happens in topological order and a cycle MUST raise an
  error naming the cycle.
- `compounds` is computed from the selectors and at-rule names found in the body.

Compatibility: a top-level `@variant <name> (...)` without a body, and a top-level
`@variant <name> { ... }` whose body contains `@slot`, MUST be treated as `@custom-variant`.

### 6.8 `@utility`

Forms:

- `@utility <name> { ... }` registers a static utility. Compiling the candidate `<name>` returns a
  clone of the body.
- `@utility <name>-* { ... }` registers a functional utility whose body uses `--value(...)` and
  `--modifier(...)` (Section 10.6).

Rules:

- MUST be top level. An empty body MUST raise an error.
- The name is unescaped first (so `@utility foo-1\/2` defines `foo-1/2`).
- A name that satisfies neither the static nor the functional name rule (Section 10.5) MUST raise
  an error. The message SHOULD distinguish a name ending in `*` but not `-*`, a `*` in the middle,
  and other invalid names.
- `@apply` inside a `@utility` body is expanded first (Section 6.10).

### 6.9 `@variant` (Nested Form)

Inside a style rule: `@variant <v1>[:<v2>...][, <v3>...] { ... }`

- Comma-separated groups are independent alternatives; each produces its own rule.
- Colon-separated names within a group stack; they are applied from right to left.
- For each group, create a `rule` with selector `&` containing the block's children (cloning the
  children for every group but the last), parse each variant name, and apply it
  (Section 9.3). An empty name, an unknown name, or a rejected application MUST raise an error.
- If the resulting selector is still `&`, splice its children in place; otherwise replace the
  `@variant` node with the rule.
- `VARIANTS` is added to `features`.

### 6.10 `@apply`

Inside a rule: `@apply <candidate> [<candidate>...];`

- A top-level `@apply` is left untouched. An `@apply` inside `@keyframes` MUST raise an error.
  An `@apply` with a body MUST raise an error.
- If every argument starts with `--`, the node is a CSS mixin invocation and MUST be left
  untouched. Mixing `--` arguments with utility candidates MUST raise an error.
- Otherwise the candidates are compiled with `compileCandidates` using `respectImportant = false`
  (the design-system-wide `important` flag does not apply; a candidate's own `!` does), and the
  `@apply` node is replaced by the children of every generated rule. Generated rule selectors are
  discarded. Variant wrappers are kept, so `@apply hover:underline` yields a nested
  `&:hover { ... }` (with its `@media`) inside the host rule.
- Every candidate MUST compile. Otherwise raise an error; the message SHOULD distinguish: a
  missing prefix when a prefix is configured, a candidate disabled by `@source not inline`, a
  variant that does not exist, an empty theme (the bundled stylesheet was not imported), and an
  unknown utility.
- `@apply` inside `@utility` bodies MAY reference other custom utilities. Build a dependency graph
  from `@utility` roots referenced by each `@apply`, sort it topologically, and expand in that
  order. A cycle MUST raise an error naming the offending candidate.
- `AT_APPLY` is added to `features`.

### 6.11 Theme Functions

The following functions are substituted in declaration values and in the `params` of `@media`,
`@custom-media`, `@container`, and `@supports`. Arguments are split on commas with `segment` and
trimmed. `THEME_FUNCTION` is added to `features` whenever a substitution happens.

- `--spacing(<n>)`
  - Let `m` be the raw `--spacing` theme value. Return `0px` when `n` is `0`, `m` when `n` is
    `1`, and `calc(m * n)` otherwise.
  - Missing argument, extra arguments, or a missing `--spacing` value MUST raise an error.
- `--alpha(<color> / <alpha>)`
  - Return `withAlpha(color, alpha)` (Section 10.3). A missing slash or extra arguments MUST
    raise an error.
- `--theme(<key>[, <fallback>...][ inline])`
  - `key` MUST start with `--`; otherwise raise an error.
  - A trailing ` inline` forces inline resolution. Inside an at-rule `params` resolution is always
    inline.
  - Resolve with `resolveThemeValue(key, inline)` (Section 7.3). When nothing resolves, return the
    joined fallback, or raise an error when there is none.
  - When a fallback exists: if the fallback is `initial`, return the resolved value; if the
    resolved value is `initial`, return the fallback; if the resolved value starts with `var(`,
    `theme(`, or `--theme(`, inject the fallback into the innermost such call that has no fallback
    or whose fallback is `initial`.
- `theme(<key>[, <fallback>...])`
  - Legacy form. Strip surrounding quotes from `key`. Resolve with inline resolution. Return the
    fallback when nothing resolves, or raise an error when there is none.

Values of generated utilities are processed by the same substitution inside `compileAstNodes`. A
substitution failure there MUST make the candidate invalid instead of raising.

## 7. Theme Resolution

### 7.1 Storage

An ordered map from key to `Theme Entry`, an ordered set of `@keyframes` nodes, and `prefix`
(string or null).

### 7.2 `add(key, value, options)`

1. If `key` ends in `-*`: `value` MUST be `initial` (otherwise raise an error). `--*` clears every
   entry; any other key clears the namespace `key` minus `-*` (Section 7.2.1).
2. If `options` includes `DEFAULT` and an existing entry for `key` lacks `DEFAULT`, return without
   changing anything.
3. If `value` is `initial`, delete `key`. Otherwise store or overwrite the entry.

#### 7.2.1 Namespace Clearing and Ignored Keys

Clearing a namespace deletes every key that starts with the namespace, except keys that belong to
an ignored sub-namespace. The ignored sub-namespaces (each also covers keys with a further `-`
suffix) are:

- under `--font`: `--font-weight`, `--font-size`
- under `--inset`: `--inset-shadow`, `--inset-ring`
- under `--text`: `--text-color`, `--text-decoration-color`, `--text-decoration-thickness`,
  `--text-indent`, `--text-shadow`, `--text-underline-offset`
- under `--grid-column`: `--grid-column-start`, `--grid-column-end`
- under `--grid-row`: `--grid-row-start`, `--grid-row-end`

The same list applies to resolution: resolving `shadow-sm` in the `--text` namespace MUST NOT
match `--text-shadow-sm`.

### 7.3 Resolution Operations

- `resolveKey(candidateValue, namespaces)`
  - For each namespace in order: the key is the namespace itself when `candidateValue` is null,
    otherwise `<namespace>-<candidateValue>`. When the key is absent and `candidateValue` contains
    `.`, also try the key with every `.` replaced by `_`. Skip ignored keys. Return the first key
    found, or null.
- `resolve(candidateValue, namespaces, options)`
  - Find the key. Return null when absent. When either the call's `options` or the entry's
    `options` includes `INLINE`, return the raw value. When the entry is `REFERENCE`, return
    `var(<escaped prefixed key>, <raw value>)`. Otherwise return `var(<escaped prefixed key>)`.
- `resolveValue(candidateValue, namespaces)`
  - Find the key and return the raw value, or null.
- `resolveWith(candidateValue, namespaces, nestedKeys)`
  - Find the key `k`. For each nested key `n`, look up `k + n` (for example `--text-lg` plus
    `--line-height` gives `--text-lg--line-height`) and resolve it with the same inline or
    `var(...)` rule. Return the main value and a map from nested key to value.
- `get(keys)`
  - Return the raw value of the first key that exists, or null.
- `namespace(ns)`
  - Return a map with a null key for `ns` itself, keys with the `<ns>-` prefix removed, and keys
    starting with `<ns>--` with only `<ns>` removed (so sub-keys keep their leading `--`).
- `keysInNamespaces(namespaces)`
  - Every key under each namespace, with the prefix removed, excluding keys that contain a
    second `--` and excluding ignored keys.
- `prefixKey(key)`
  - When `prefix` is set, `--<prefix>-<key without leading -->`.
- `markUsedVariable(key)`
  - Set `USED` on the (unprefixed, unescaped) key. Return true when the flag was not set before.
- `resolveThemeValue(path, forceInline = true)`
  - Split off a modifier after the last `/` in `path` (trimmed). Resolve `path` with
    `resolve(null, [path], INLINE when forceInline)`. When a modifier exists, return
    `withAlpha(value, modifier)`.

### 7.4 Modes

- `inline`: consumers embed raw values; the variable is still printed.
- `reference`: the variable is not printed; consumers embed `var(key, value)`.
- `default`: user-defined entries win regardless of order. The bundled `theme.css` uses this.
- `static`: the variable is printed even when unused.

### 7.5 Emission

The `:root, :host` rule that replaced the first `@theme` receives one declaration per entry that is
not `REFERENCE`, in insertion order, as `<escape(prefixKey(key))>: <value>`. The declarations are
wrapped in `context { theme: true }`. Each stored `@keyframes` is appended to the document as
`context { theme: true } > at-root > @keyframes`. Entries whose value resolved to `initial` are
not printed.

### 7.6 Unused Value Removal

During optimization (Section 12.2):

- A theme declaration is removed unless its entry is `STATIC` or `USED`, or some other theme
  declaration that references it via `var(...)` is itself used (transitively).
- `USED` is set when a `var(--key)` appears in any non-theme declaration value in the final AST,
  or when a candidate string of the form `--key` is passed to `build`.
- When the `:root, :host` rule becomes empty it is removed, together with any enclosing `@layer`
  rules that become empty.
- A theme `@keyframes` is removed unless its name appears in an `animation` declaration or in the
  value of a used `--animate-*` entry.

## 8. Candidate Grammar and Parsing

### 8.1 Surface Grammar

```text
candidate  := [prefix ":"] { variant ":" } ["!"] utility ["!"]
utility    := arbitrary-property | static-name | functional
arbitrary-property := "[" property ":" value "]" [modifier]
functional := root [ "-" named-value | "-[" [type ":"] value "]" | "-(" [type ":"] "--" ident ")" ] [modifier]
modifier   := "/" ( named-value | "[" value "]" | "(" "--" ident ")" )
variant    := "[" selector-or-at-rule "]" | name [ "-" named-value | "-[" value "]" | "-(" "--" ident ")" ] ["/" modifier-value]
```

Only one `!` is allowed; the trailing form is canonical and the leading form is accepted for
compatibility. A leading `-` on the root selects the negative form of a utility that supports it
(for example `-mt-2`).

### 8.2 `parseCandidate(input)` Algorithm

The function yields zero or more `Candidate` interpretations. Any step that says "invalid" ends the
function without further output; "skip" abandons only the current interpretation.

1. `rawVariants = segment(input, ":")`. The last element is the base; the rest are variants.
2. If the theme has a prefix: with a single element the candidate is invalid; if the first element
   is not the prefix the candidate is invalid; otherwise drop the first element.
3. Parse the variants from right to left with `parseVariant`. Any null result makes the candidate
   invalid. The resulting list is in application order.
4. If the base ends in `!`, remove it and set `important`. Otherwise, if it starts with `!`, remove
   it and set `important`.
5. If a static utility named `base` exists and `base` contains no `[`, yield a `static` candidate.
   Continue.
6. `parts = segment(base, "/")`. Three or more parts is invalid. The first part is the base without
   modifier; the second, when present, is the modifier text.
7. Parse the modifier (Section 8.2.1). Modifier text that parses to null is invalid.
8. Arbitrary property (base starts with `[`): the base MUST end in `]`; the second character MUST
   be `a`-`z` or `-`; strip the brackets; find the first `:` (absent, first, or last position is
   invalid); `property` is the text before it; `value` is `decodeArbitraryValue` of the text after
   it and MUST satisfy `isValidArbitrary`. Yield an `arbitrary` candidate and stop.
9. Arbitrary value (base ends in `]`): find the first `-[`; absent is invalid. The root is the
   text before it and MUST be a registered functional root; otherwise invalid. The single root
   interpretation is `(root, "[...]")`.
10. Variable shorthand (base ends in `)`): find the first `-(`; absent is invalid. The root is the
    text before it and MUST be a registered functional root. Split the parenthesized text on `:`;
    two parts give a data type and a value. The value MUST start with `--` and satisfy
    `isValidArbitrary`. Rewrite the value as `[var(--x)]` or `[<type>:var(--x)]` and treat it as
    step 9.
11. Otherwise `roots = findRoots(base, isFunctionalRoot)` (Section 8.2.2).
12. For each `(root, value)`: create a `functional` candidate with the parsed modifier. When
    `value` is null, yield it as is.
    - When `value` contains `[`: it MUST end in `]` (otherwise invalid). Decode the bracket
      contents; skip when `isValidArbitrary` fails. Read a type hint: consume characters `a`-`z`
      and `-` from the start; if the next character is `:`, the consumed text is the hint and the
      rest is the value. Skip when the value is empty or whitespace, or when the hint is the empty
      string. Set `value = { arbitrary, dataType: hint or null, value }`.
    - Otherwise: `fraction` is `<value>/<modifier text>` when modifier text exists and the parsed
      modifier is named, else null. Skip when `value` fails the named value pattern. Set
      `value = { named, value, fraction }`.
    - Yield the candidate.

#### 8.2.1 Modifier Parsing

- `[x]`: decode `x`; it MUST satisfy `isValidArbitrary` and be non-empty; result is arbitrary.
- `(--x)`: the inner text MUST start with `--` and satisfy `isValidArbitrary`; result is arbitrary
  with value `var(--x)`.
- Otherwise the text MUST match the named value pattern; result is named. Anything else is null.

#### 8.2.2 `findRoots(input, exists)`

1. If `exists(input)`, yield `(input, null)`.
2. Starting from the last `-`, repeatedly cut the input at that `-` and test the left part. When
   it exists, the right part is the value; if the right part is empty, stop. If the left part is
   `@`, `@` exists, and the separator is `-`, stop. Otherwise yield and move to the previous `-`.
3. If the input starts with `@` and `@` exists, finally yield `("@", input without the @)`.

Multiple interpretations MAY be yielded (`border-t-2` yields both `border-t` + `2` and `border` +
`t-2`). The compiler emits every interpretation that produces CSS; built-in definitions are
designed so that at most one does.

### 8.3 `parseVariant(input)` Algorithm

1. Arbitrary variant (`[...]`): a value starting with `@` that also contains `&` is null. Decode
   the contents; they MUST satisfy `isValidArbitrary` and be non-empty. `relative` is true when the
   selector starts with `>`, `+`, or `~`. When the selector is not relative, does not start with
   `@`, and contains no `&`, wrap it as `&:is(<selector>)`.
2. `parts = segment(input, "/")`; three or more parts is null. The first part is the name; the
   second, if present, is the modifier text.
3. For each `(root, value)` from `findRoots(name, variantExists)`, branch on the registered kind:
   - `static`: any value or modifier is null. Return `{ static, root }`.
   - `functional`: parse the modifier (text present but null result is null). A null value gives
     `{ functional, root, value: null, modifier }`. A value ending in `]` MUST start with `[`
     (otherwise try the next root); decode, validate, non-empty; the value is arbitrary. A value
     ending in `)` MUST start with `(`; decode, validate, non-empty, MUST start with `--`; the
     value is arbitrary with text `var(--x)`. Otherwise the value MUST match the named value
     pattern (else try the next root); the value is named.
   - `compound`: a null value is null. When the root is `not`, `has`, or `in` and a modifier
     exists, move the modifier onto the value (`value = value + "/" + modifier`). Parse the value
     as a variant (null is null). The pair MUST satisfy `compoundsWith(root, inner)`
     (Section 9.1). Parse the remaining modifier. Return `{ compound, root, modifier, variant }`.
4. Return null.

## 9. Variants

### 9.1 Registry Semantics

- `static(name, apply, { compounds })`, `functional(name, apply, { compounds })`, and
  `compound(name, compoundsWith, apply, { compounds })` register definitions. `compounds` defaults
  to `STYLE_RULES`. Non-compound definitions have `compoundsWith = NEVER`.
- Each new name receives `order = lastOrder + 1`. Inside `group(fn, compareFn)` every name
  registered by `fn` shares one `order`, and `compareFn` is stored for that order.
- Re-registering an existing name replaces `kind`, `apply`, and `compounds` but keeps `order`.
- `compoundsWith(parent, child)` is true only when the parent is a compound definition, the child's
  `compounds` is not `NEVER`, the parent's `compoundsWith` is not `NEVER`, and the bitwise AND of
  the two is non-zero. For an arbitrary child the `compounds` value is computed with
  `compoundsForSelectors([selector])`.
- `compoundsForSelectors(selectors)`: return `NEVER` if any selector starts with `@` but not with
  `@media`, `@supports`, or `@container`, or if any selector contains `::`. Otherwise OR together
  `AT_RULES` for at-rule selectors and `STYLE_RULES` for the rest.

### 9.2 Application Contract

Every `apply` function receives a rule node and mutates `node.nodes` in place, wrapping the
existing children in new rules or at-rules. Returning `null` rejects the candidate.

The standard static helper `staticVariant(name, selectors)` sets `node.nodes` to one
`rule(selector, children)` per selector and computes `compounds` with `compoundsForSelectors`.

### 9.3 `applyVariant(node, variant, depth = 0)`

- `arbitrary`: when `relative` and `depth` is 0, reject. Otherwise
  `node.nodes = [rule(selector, node.nodes)]`.
- `static` and `functional`: call the registered `apply`; propagate rejection.
- `compound`: create an isolated at-rule `@slot` with no children and apply the inner variant to
  it at `depth + 1` (propagate rejection). When the root is `not` and the isolated node now has
  more than one child, reject. For each child (any child that is not a rule or at-rule rejects),
  call the compound definition's `apply` on that child (propagate rejection). Finally walk the
  isolated node's children and give every rule or at-rule with no children the original
  `node.nodes`; then set `node.nodes` to the isolated node's children.

### 9.4 Ordering

`getVariantOrder()` sorts every parsed variant object with `compare` and assigns an index that
increases each time `compare` reports a difference between neighbors. Equal variants share an
index. The result MAY be cached until a new variant string is parsed.

`compare(a, z)`:

1. Identical objects compare equal. Null sorts first.
2. Two arbitrary variants compare by selector text. An arbitrary variant sorts after any other
   kind.
3. Compare registered `order`.
4. When both are compound: compare the inner variants recursively, then compare modifiers by text
   (a missing modifier sorts first).
5. When a group comparison function exists for the order, use it.
6. Compare roots by text.
7. Functional values: a null value sorts first; an arbitrary value sorts after a named value;
   otherwise compare value text.

### 9.5 Built-in Variants

Registration order determines output order and MUST be preserved as listed. Each entry gives the
name, what it wraps the utility's nodes in, and the `compounds` value when it is not the default
`STYLE_RULES`.

- `*`: `:is(& > *)`; `NEVER`.
- `**`: `:is(& *)`; `NEVER`.
- `not-<v>` (compound; `compoundsWith = STYLE_RULES | AT_RULES`): negates each rule produced by the
  inner variant. A style selector `&:hover` becomes `&:not(:hover)`; `@media (q)` becomes
  `@media not all and (q)`; `@supports (q)` becomes `@supports not (q)`; `@container (q)` becomes
  `@container not (q)`. When the inner variant produces several sibling rules, each is negated
  independently (`not-hover` yields both `.x:not(:hover)` and `@media not all and (hover: hover)
  { .x }`).
- `group-<v>[/<name>]` (compound; `compoundsWith = STYLE_RULES`): for each rule produced by the
  inner variant, replace `&` in its selector with `:where(.group)` (or `:where(.group\/<name>)`;
  with a theme prefix `p`, `:where(.p\:group)`), wrap a selector list in `:is(...)`, and set the
  selector to `&:is(<selector> *)`. Reject when the inner variant produced nested style rules or
  a relative arbitrary selector.
- `peer-<v>[/<name>]`: as `group` with `.peer` and `&:is(<selector> ~ *)`.
- `first-letter`: `&::first-letter`; `NEVER`.
- `first-line`: `&::first-line`; `NEVER`.
- `marker`: `& *::marker`, `&::marker`, `& *::-webkit-details-marker`,
  `&::-webkit-details-marker`; `NEVER`.
- `selection`: `& *::selection`, `&::selection`; `NEVER`.
- `file`: `&::file-selector-button`; `NEVER`.
- `placeholder`: `&::placeholder`; `NEVER`.
- `backdrop`: `&::backdrop`; `NEVER`.
- `details-content`: `&::details-content`; `NEVER`.
- `before` and `after`: `&::before` (or `&::after`) whose children are an `at-root` holding
  `@property --lm-content { syntax: "*"; initial-value: ""; inherits: false; }`, then
  `content: var(--lm-content);`, then the utility's nodes; `NEVER`.
- `first`, `last`, `only`, `odd`, `even`, `first-of-type`, `last-of-type`, `only-of-type`:
  `&:first-child`, `&:last-child`, `&:only-child`, `&:nth-child(odd)`, `&:nth-child(even)`,
  `&:first-of-type`, `&:last-of-type`, `&:only-of-type`.
- `visited`, `target`: `&:visited`, `&:target`.
- `open`: `&:is([open], :popover-open, :open)`.
- `default`, `checked`, `indeterminate`, `placeholder-shown`, `autofill`, `optional`, `required`,
  `valid`, `invalid`, `user-valid`, `user-invalid`, `in-range`, `out-of-range`, `read-only`:
  `&:<name>`.
- `empty`, `focus-within`: `&:<name>`.
- `hover`: `&:hover { @media (hover: hover) { ... } }`.
- `focus`, `focus-visible`, `active`, `enabled`, `disabled`: `&:<name>`.
- `inert`: `&:is([inert], [inert] *)`.
- `in-<v>` (compound; `compoundsWith = STYLE_RULES`): for each rule produced by the inner variant,
  replace `&` with `*` and set the selector to `:where(<selector>) &`. A modifier rejects.
- `has-<v>` (compound; `compoundsWith = STYLE_RULES`): replace `&` with `*` and set the selector
  to `&:has(<selector>)`. A modifier rejects.
- `aria-<v>` (functional): a named value gives `&[aria-<v>="true"]`; an arbitrary value gives
  `&[aria-<v>]` where an unquoted right-hand side after `=` is wrapped in double quotes (a trailing
  ` i` or ` s` flag is preserved outside the quotes). A modifier rejects. The attribute selector
  MUST parse; otherwise reject.
- `data-<v>` (functional): `&[data-<v>]` with the same quoting rule.
- `nth-<n>`, `nth-last-<n>`, `nth-of-type-<n>`, `nth-last-of-type-<n>` (functional):
  `&:nth-child(n)`, `&:nth-last-child(n)`, `&:nth-of-type(n)`, `&:nth-last-of-type(n)`. A named
  value MUST be a positive integer; an arbitrary value is used verbatim.
- `supports-<v>` (functional; `AT_RULES`): when the value matches `^[\w-]*\s*\(` it is used as the
  condition verbatim except that bare `and`, `or`, and `not` function names receive surrounding
  spaces; when the value contains no `:` it becomes `(<v>: var(--lm))`; otherwise it is wrapped
  in parentheses unless already parenthesized. Wrap in `@supports <condition>`.
- `motion-safe`, `motion-reduce`: `@media (prefers-reduced-motion: no-preference)`,
  `@media (prefers-reduced-motion: reduce)`; `AT_RULES`.
- `contrast-more`, `contrast-less`: `@media (prefers-contrast: more)`,
  `@media (prefers-contrast: less)`; `AT_RULES`.
- Breakpoint group `max` (one group, descending comparison): `max-<bp>` wraps in
  `@media (width < <value>)`; `AT_RULES`.
- Breakpoint group `min` (one group, ascending comparison): one static variant per
  `--breakpoint-*` theme key wrapping in `@media (width >= <value>)`, plus functional `min-<bp>`
  with the same output; `AT_RULES`.
- Container group `@max` (descending): `@max-<w>[/<name>]` wraps in
  `@container [<name> ](width < <value>)`; `AT_RULES`.
- Container group `@` (ascending): `@<w>[/<name>]` and `@min-<w>[/<name>]` wrap in
  `@container [<name> ](width >= <value>)`; `AT_RULES`.
- `portrait`, `landscape`: `@media (orientation: portrait)`, `@media (orientation: landscape)`;
  `AT_RULES`.
- `ltr`, `rtl`: `&:where(:dir(ltr), [dir="ltr"], [dir="ltr"] *)` and the `rtl` equivalent.
- `dark`: `@media (prefers-color-scheme: dark)`; `AT_RULES`. Authors override it with
  `@custom-variant dark (&:where(.dark, .dark *));`.
- `starting`: `@starting-style`; `NEVER`.
- `print`: `@media print`; `AT_RULES`.
- `forced-colors`, `inverted-colors`: `@media (forced-colors: active)`,
  `@media (inverted-colors: inverted)`; `AT_RULES`.
- `pointer-none`, `pointer-coarse`, `pointer-fine`, `any-pointer-none`, `any-pointer-coarse`,
  `any-pointer-fine`: `@media (pointer: ...)` and `@media (any-pointer: ...)`; `AT_RULES`.
- `noscript`: `@media (scripting: none)`; `AT_RULES`.

Breakpoint and container values resolve as follows: a static breakpoint name uses the raw
`--breakpoint-<name>` value; a functional value uses the arbitrary text or the raw
`--breakpoint-*` (or `--container-*`) value of the named key; a modifier on `min`/`max` rejects;
a value containing `var(` rejects. Within a group, variants compare by bucketing values by unit
(or by function name when the value is a function call) and then numerically in the group's
direction; values that cannot be resolved sort first in ascending groups and last in descending
groups.

## 10. Utilities

### 10.1 Registry Semantics

- `static(name, compile)` and `functional(name, compile, options)` append a definition to the list
  for `name`.
- `has(name, kind)` is true when at least one definition of that kind exists for `name`.
- `get(name)` returns the list (possibly empty).

### 10.2 Definition Helpers

Implementations SHOULD define built-in utilities through these helpers so that behavior stays
uniform.

`staticUtility(name, declarations)` registers a static definition returning the given
`(property, value)` pairs as declarations, or the result of calling a supplied function (used for
`at-root` nodes).

`functionalUtility(root, description)` registers a functional definition. The description fields
are:

- `supportsNegative` (boolean): also register `-<root>`; its values are wrapped as
  `calc(<value> * -1)`.
- `supportsFractions` (boolean): a named value with a fraction resolves to
  `calc(<a> / <b> * 100%)` when both parts are positive integers.
- `themeKeys` (list of namespaces) used to resolve named values and the default value.
- `defaultValue` (string, null, or absent): the value for a candidate without a value segment.
  When absent, resolve the first namespace itself (`theme.resolve(null, themeKeys)`).
- `staticValues` (map from named value to node list): consulted last, only for non-negative
  candidates without a modifier.
- `handleBareValue(value)` and `handleNegativeBareValue(value)`: return a value string for a named
  value that is not in the theme, or null.
- `handle(value, dataType)`: return the declarations for a final value.

Value resolution for a functional candidate:

1. No value: a modifier makes the candidate invalid. Use `defaultValue` when present; otherwise
   resolve the namespace itself.
2. Arbitrary value: a modifier makes the candidate invalid. Pass the value and data type to
   `handle`.
3. Named value: `theme.resolve(fraction or value, themeKeys)`.
   - When this resolved and a modifier exists but no fraction was consumed, the candidate is
     invalid (`w-4/foo`).
   - When unresolved and `supportsFractions` and a fraction exists: both parts MUST be positive
     integers; the value is `calc(a / b * 100%)`.
   - When unresolved, the candidate is negative, and `handleNegativeBareValue` exists: use its
     result; a result without `/` combined with a modifier is invalid; pass the result to
     `handle` directly (no further negation).
   - When unresolved and `handleBareValue` exists: use its result; a result without `/` combined
     with a modifier is invalid.
   - When unresolved, non-negative, no modifier, and `staticValues` has the value: return a clone
     of those nodes.
4. A null value means no output. A negative candidate wraps the value as `calc(<value> * -1)`
   before `handle`.

`colorUtility(root, { themeKeys, handle })` registers a functional definition that requires a
value. An arbitrary value goes through `asColor(value, modifier)`; a named value goes through
`resolveThemeColor`. A null result means no output.

`spacingUtility(name, themeKeys, handle, options)` registers `<name>-px` (value `1px`),
`-<name>-px` when negative values are supported (value `-1px`), and a `functionalUtility` with
`defaultValue = null`, `handleBareValue` returning `--spacing(<v>)` when `v` is a multiple of
0.25 and the `--spacing` theme value exists (null otherwise), and `handleNegativeBareValue`
returning `--spacing(-<v>)` under the same conditions. Theme function substitution later turns
`--spacing(4)` into `calc(var(--spacing) * 4)`.

### 10.3 Colors and Opacity

- `withAlpha(color, alpha)`: when `alpha` parses as a number, replace it with
  `<number * 100>%`. When the result is `100%`, return `color`. Otherwise return
  `color-mix(in oklab, <color> <alpha>, transparent)`.
- `asColor(value, modifier)`: no modifier returns `value`. An arbitrary modifier returns
  `withAlpha(value, modifier.value)`. A named modifier first tries the `--opacity` namespace;
  when absent, the modifier MUST be a multiple of 0.25 (else null) and the alpha is
  `<modifier>%`.
- `resolveThemeColor(candidate, themeKeys)`: the named values `inherit`, `transparent`, and
  `current` map to `inherit`, `transparent`, and `currentcolor`; anything else resolves through
  the theme. Apply `asColor` to the result.

### 10.4 Custom Property Registrations

Utilities that compose through internal variables (for example font weight) emit an `at-root`
node containing `@property --lm-<name> { syntax: "*"; inherits: false; [initial-value: <v>;] }`
alongside their declarations. The optimizer prints each registration once (Section 12.1).

### 10.5 Utility Name Rules for `@utility`

- Static name: the root MUST match `^-?[a-z][a-zA-Z0-9_-]*`. The remainder MAY contain letters,
  digits, `_`, `-`, `.` (only between digits), `%` (only at the end and only after a digit), and
  at most one `/` (not at the end). A root ending in `-` with an empty remainder is invalid.
- Functional name: MUST end in `-*`, and the text before `-*` MUST match `^-?[a-z][a-zA-Z0-9_-]*$`.

### 10.6 `--value(...)` and `--modifier(...)` Resolution

Inside a functional `@utility` body, each declaration value is parsed and every `--value(...)` or
`--modifier(...)` call is replaced by the first argument that resolves. Before resolution the
arguments are normalized: `\*` becomes `*`, `--a --b` becomes `--a-*--b`, whitespace is removed,
repeated `-*` collapses to one, and a bare `--x` (no parentheses, no `-*`) becomes `--x-*`.

Argument forms and what they resolve against (`--value` uses the candidate's value, `--modifier`
uses the modifier):

- `'literal'` or `"literal"`: a named value equal to the literal.
- `--ns-*`: `theme.resolve(value, ["--ns"])`.
- `--ns-*--sub`: the `--sub` entry from `resolveWith(value, ["--ns"], ["--sub"])`.
- `number`, `integer`, `ratio`, `percentage`: a named value that type-checks. `ratio` uses the
  candidate's `fraction` and both parts MUST be positive integers; `number` MUST be a multiple of
  0.25; `percentage` MUST be an integer followed by `%`. A `ratio` result is printed as
  `<a> / <b>`.
- `[type]`: an arbitrary value. `[*]` accepts anything. When the candidate carries a type hint it
  MUST equal `type`. Otherwise the value MUST infer as `type` (Section 10.7).
- `--default(<v>)`: used only when the candidate has no value (or no modifier).

Any other bare word is an unsupported data type; implementations SHOULD warn and MUST ignore it.

Validity of the whole utility for a candidate:

- At least one `--value(...)` MUST be present and at least one MUST resolve; otherwise no output.
- A declaration whose `--value` or `--modifier` did not resolve is removed.
- When `--modifier(...)` was used, did not resolve, and the candidate has a modifier, no output.
- When a `ratio` value resolved and a `--modifier` also resolved, no output.
- When the candidate has a modifier that was consumed neither by `ratio` nor by `--modifier`, no
  output.
- When a `ratio` value resolved, remove every declaration that resolved a non-ratio `--value`.

### 10.7 Data Type Inference

`inferDataType(value, types)` returns the first type in `types` whose predicate accepts `value`,
or null. A value starting with `var(` never matches. The predicates are:

- `color`: named colors, `#` hex, `rgb`, `rgba`, `hsl`, `hsla`, `oklch`, `oklab`, `lab`, `lch`,
  `color`, `color-mix`, `light-dark` function calls, `transparent`, `currentcolor`.
- `length`: a number followed by a length unit (`px`, `rem`, `em`, `vh`, `vw`, `svh`, `cqw`, and
  the other CSS length units), `0`, math functions, `--spacing(...)`.
- `percentage`: a number followed by `%`, or a math function.
- `number`: a number.
- `integer`: a non-negative integer.
- `ratio`: `<number>/<number>` with optional spaces.
- `url`: `url(...)`.
- `image`: `image`, `image-set`, `cross-fade`, `element` calls and any `*-gradient(...)`.
- `position`: combinations of `top`, `right`, `bottom`, `left`, `center` and lengths.
- `bg-size`: `cover`, `contain`, `auto`, or one or two lengths or percentages.
- `line-width`: `thin`, `medium`, `thick`, or a length.
- `absolute-size`: `xx-small` through `xxx-large`; `relative-size`: `larger`, `smaller`.
- `family-name` and `generic-name`: font family names and the generic families.
- `angle`: a number with `deg`, `rad`, `grad`, or `turn`; `vector`: three space-separated numbers.

### 10.8 Built-in Utility Catalog

The catalog below lists the utilities a conforming implementation MUST provide, grouped by area.
Each entry gives the class pattern and the CSS it produces. Namespaces in parentheses are the
`themeKeys` used for named values, in order.

Layout:

- `block`, `inline-block`, `inline`, `flex`, `inline-flex`, `grid`, `inline-grid`, `hidden`,
  `contents`, `flow-root`, `table`, `table-cell`, `table-row`, `list-item`: `display: <value>`
  (`hidden` is `display: none`).
- `static`, `fixed`, `absolute`, `relative`, `sticky`: `position: <value>`.
- `visible`, `invisible`, `collapse`: `visibility: visible | hidden | collapse`.
- `isolate`, `isolation-auto`: `isolation: isolate | auto`.
- `box-border`, `box-content`: `box-sizing: border-box | content-box`.
- `overflow-{auto,hidden,clip,visible,scroll}` and the `overflow-x-*`, `overflow-y-*` forms.
- `float-{left,right,start,end,none}`, `clear-{left,right,start,end,both,none}`.
- `sr-only`: `position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow:
  hidden; clip-path: inset(50%); white-space: nowrap; border-width: 0`. `not-sr-only` reverses
  it.
- `inset`, `inset-x`, `inset-y`, `inset-s`, `inset-e`, `top`, `right`, `bottom`, `left`:
  `spacingUtility` (`--inset`, `--spacing`) targeting `inset`, `inset-inline`, `inset-block`,
  `inset-inline-start`, `inset-inline-end`, `top`, `right`, `bottom`, `left`; negative and
  fractions supported; plus static `<name>-auto`, `<name>-full` (`100%`), `-<name>-full`
  (`-100%`).
- `z-<n>`: `z-index` (`--z-index`); bare non-negative integers; negative supported; `z-auto`.
- `order-<n>`: `order` (`--order`); bare non-negative integers; negative supported;
  `order-first` is `-9999`, `order-last` is `9999`.

Flexbox and grid:

- `flex-row`, `flex-row-reverse`, `flex-col`, `flex-col-reverse`: `flex-direction`.
- `flex-wrap`, `flex-nowrap`, `flex-wrap-reverse`: `flex-wrap`.
- `flex-auto` (`flex: auto`), `flex-initial` (`flex: 0 auto`), `flex-none` (`flex: none`),
  `flex-<n>` (bare non-negative integer, `flex: <n>`), `flex-<a>/<b>` (`flex: calc(a/b * 100%)`),
  `flex-[...]`.
- `grow`, `grow-<n>`, `shrink`, `shrink-<n>`: `flex-grow` and `flex-shrink` (default `1`).
- `basis-*`: `flex-basis` (`--flex-basis`, `--spacing`, `--container`); fractions supported;
  `basis-auto`, `basis-full`.
- `grid-cols-<n>`: `grid-template-columns: repeat(<n>, minmax(0, 1fr))`; `grid-cols-none`,
  `grid-cols-subgrid`; arbitrary values verbatim. `grid-rows-*` likewise.
- `col-span-<n>`: `grid-column: span <n> / span <n>`; `col-span-full`: `grid-column: 1 / -1`;
  `col-start-<n>`, `col-end-<n>`, `col-<n>`; `row-*` likewise.
- `grid-flow-{row,col,dense,row-dense,col-dense}`; `auto-cols-{auto,min,max,fr}`,
  `auto-rows-*`.
- `gap`, `gap-x`, `gap-y`: `spacingUtility` (`--gap`, `--spacing`) targeting `gap`, `column-gap`,
  `row-gap`.
- `justify-{normal,center,start,end,between,around,evenly,stretch,baseline}`,
  `justify-items-*`, `justify-self-*`, `items-{center,start,end,baseline,stretch}`,
  `content-*`, `self-*`, `place-content-*`, `place-items-*`, `place-self-*`.

Spacing:

- `p`, `px`, `py`, `ps`, `pe`, `pt`, `pr`, `pb`, `pl`: `spacingUtility` (`--padding`,
  `--spacing`) targeting `padding`, `padding-inline`, `padding-block`, `padding-inline-start`,
  `padding-inline-end`, `padding-top`, `padding-right`, `padding-bottom`, `padding-left`.
- `m`, `mx`, `my`, `ms`, `me`, `mt`, `mr`, `mb`, `ml`: `spacingUtility` (`--margin`,
  `--spacing`) on the corresponding `margin*` properties; negative supported; plus `<name>-auto`.

Sizing:

- `w`, `min-w`, `max-w`: `spacingUtility` (`--width` or `--min-width` or `--max-width`,
  `--spacing`, `--container`) with fractions; `h`, `min-h`, `max-h` (`--height` and friends,
  `--spacing`) with fractions; `size-*` sets `width` and `height` together and emits
  `--lm-sort: size` first.
- Static sizes: `w-auto`, `w-full`, `w-screen`, `w-svw`, `w-lvw`, `w-dvw`, `w-min`, `w-max`,
  `w-fit`, and the `h-*` counterparts with `vh` units.

Typography:

- `font-<family>`: `font-family` from `--font-*` with the sub-keys `--font-feature-settings` and
  `--font-variation-settings` emitted as `font-feature-settings` and `font-variation-settings`
  when present.
- `font-<weight>`: from `--font-weight-*`; emits `at-root @property --lm-font-weight`,
  `--lm-font-weight: <value>`, and `font-weight: <value>`. An arbitrary value infers `number`
  (weight) versus `family-name` or `generic-name` (family).
- `text-<size>`: from `--text-*` with sub-keys `--line-height`, `--letter-spacing`,
  `--font-weight`; emits `font-size`, then `line-height: var(--lm-leading, <sub>)`,
  `letter-spacing: var(--lm-tracking, <sub>)`, `font-weight: var(--lm-font-weight, <sub>)` for
  the sub-keys that exist. A modifier `/<leading>` resolves against `--leading`, then as a
  spacing multiplier (`--spacing(<n>)`), then `none` as `1`, and replaces the line-height
  declaration; an unresolvable modifier invalidates the candidate.
- `text-<color>`: `color` (`--text-color`, `--color`). An arbitrary value infers `color` versus
  `length`, `percentage`, `absolute-size`, `relative-size` (font size).
- `text-left`, `text-center`, `text-right`, `text-justify`, `text-start`, `text-end`:
  `text-align`.
- `leading-*`: `spacingUtility` (`--leading`, `--spacing`) emitting `at-root @property
  --lm-leading`, `--lm-leading: <v>`, `line-height: <v>`; `leading-none` is `1`.
- `tracking-*`: `letter-spacing` (`--tracking`); negative supported; emits `--lm-tracking`.
- `uppercase`, `lowercase`, `capitalize`, `normal-case`: `text-transform`.
- `italic`, `not-italic`: `font-style`.
- `underline`, `overline`, `line-through`, `no-underline`: `text-decoration-line`.
- `truncate` (`overflow: hidden; text-overflow: ellipsis; white-space: nowrap`), `text-ellipsis`,
  `text-clip`.
- `whitespace-{normal,nowrap,pre,pre-line,pre-wrap,break-spaces}`, `break-{normal,all,keep}`,
  `list-{none,disc,decimal}`, `list-inside`, `list-outside`, `antialiased`,
  `subpixel-antialiased`.
- `underline-offset-<n>`: `text-underline-offset` (`--text-underline-offset`); bare integers
  become `<n>px`; negative supported; `underline-offset-auto`.
- `indent-*`: `spacingUtility` (`--text-indent`, `--spacing`) on `text-indent`; negative
  supported.

Backgrounds and borders:

- `bg-<value>`: named values resolve first as a color (`--background-color`, `--color`) giving
  `background-color`, then as an image (`--background-image`) giving `background-image`. An
  arbitrary value infers, in order, `image`, `color`, `percentage`, `position`, `bg-size`,
  `length`, `url`; `percentage` and `position` give `background-position`; `bg-size` and
  `length` give `background-size`; `image` and `url` give `background-image`; anything else is
  treated as a color with the modifier applied. A modifier on a non-color value invalidates the
  candidate.
- `bg-auto`, `bg-cover`, `bg-contain`, `bg-fixed`, `bg-local`, `bg-scroll`, `bg-top`,
  `bg-center`, `bg-bottom`, `bg-left`, `bg-right`, `bg-repeat`, `bg-no-repeat`, `bg-repeat-x`,
  `bg-repeat-y`, `bg-none`, `bg-clip-{border,padding,content,text}`,
  `bg-origin-{border,padding,content}`.
- `border`, `border-x`, `border-y`, `border-s`, `border-e`, `border-t`, `border-r`, `border-b`,
  `border-l`: no value gives width `--default-border-width` (theme) or `1px`; a named color
  (`--border-color`, `--color`) gives the color declarations; a named width from
  `--border-width` or a bare non-negative integer `<n>` (as `<n>px`) gives width; an arbitrary
  value infers `color`, `line-width`, `length`. Width output is `at-root @property
  --lm-border-style { initial-value: solid }`, then `border-style: var(--lm-border-style)`, then
  the `border-width` (or side-specific width) declaration.
- `border-solid`, `border-dashed`, `border-dotted`, `border-double`, `border-hidden`,
  `border-none`: `--lm-border-style: <v>; border-style: <v>`.
- `rounded`, `rounded-s`, `rounded-e`, `rounded-t`, `rounded-r`, `rounded-b`, `rounded-l`,
  `rounded-ss`, `rounded-se`, `rounded-ee`, `rounded-es`, `rounded-tl`, `rounded-tr`,
  `rounded-br`, `rounded-bl`: `functionalUtility` (`--radius`) on the corresponding
  `border-*-radius` properties; `rounded-none` is `0`; `rounded-full` is
  `calc(infinity * 1px)`.

Effects, transitions, interactivity:

- `opacity-<n>`: `opacity` (`--opacity`); a bare value that is a multiple of 0.25 becomes
  `<n>%`.
- `transition`: `transition-property: color, background-color, border-color, outline-color,
  text-decoration-color, fill, stroke, --lm-gradient-from, --lm-gradient-via, --lm-gradient-to,
  opacity, box-shadow, transform, translate, scale, rotate, filter, -webkit-backdrop-filter,
  backdrop-filter, display, content-visibility, overlay, pointer-events` followed by
  `transition-timing-function: var(--default-transition-timing-function)` and
  `transition-duration: var(--default-transition-duration)`; `transition-none`,
  `transition-all`, `transition-colors`, `transition-opacity`, `transition-shadow`,
  `transition-transform` select subsets.
- `duration-<n>`: `transition-duration: <n>ms` (`--transition-duration`); `delay-<n>` likewise;
  `ease-*`: `transition-timing-function` (`--ease`); `ease-linear`, `ease-initial`.
- `animate-*`: `animation` (`--animate`); `animate-none`.
- `cursor-*`, `select-{none,text,all,auto}`, `pointer-events-{none,auto}`, `resize`,
  `resize-{none,x,y}`, `appearance-{none,auto}`, `scroll-{auto,smooth}`, `will-change-*`,
  `content-[...]` (`--lm-content` plus `content`), `aspect-*` (`--aspect`; `ratio` values as
  `a / b`; `aspect-square`, `aspect-video`, `aspect-auto`), `columns-*`, `object-{contain,
  cover,fill,none,scale-down}`, `accent-*`, `caret-*`, `fill-*`, `stroke-*` (`colorUtility`).

Implementations MAY provide additional utilities (shadows, rings, gradients, masks, transforms,
filters, and others) using the same helpers and the same variable conventions.

## 11. Compilation and Ordering

### 11.1 `compileCandidates(rawCandidates, designSystem, options)`

1. For each raw candidate: skip it when it is in `invalidCandidates`; parse it; skip it (and
   report it invalid) when parsing yields nothing.
2. For each interpretation, call `compileAstNodes` (Section 11.2). When no interpretation yields
   rules, report the raw candidate invalid.
3. Attach `{ propertySort, variantOrder, candidate }` to every generated rule node, where
   `variantOrder` is the bitwise OR of `1 << index` for each variant's index from
   `getVariantOrder()`.
4. Sort the rule nodes (Section 11.3).

Options: `respectImportant` (default true) controls whether the design-system-wide `important`
flag applies; `onInvalidCandidate` receives each invalid raw candidate.

### 11.2 `compileAstNodes(candidate, flags)`

1. Compile the base utility:
   - An `arbitrary` candidate yields `[declaration(property, asColor(value, modifier))]`; a null
     color makes it invalid.
   - Otherwise iterate the definitions for `candidate.root` whose `kind` matches, first the
     regular definitions and then the fallback definitions, applying the return-value contract of
     Section 4.1.7. Collect every successful node list.
2. For each node list: compute `propertySort` (Section 11.3); when the candidate is `important`
   or the design system is `important` and `flags` includes `RESPECT_IMPORTANT`, mark every
   declaration `important` except those inside `at-root` nodes.
3. Create `rule(".<escape(raw)>", nodes)` and apply `candidate.variants` in order with
   `applyVariant`. Any rejection makes the whole candidate produce nothing.
4. Substitute theme functions in the result and expand nested `@variant`; on failure produce
   nothing.

### 11.3 Ordering

`propertySort` for a node list:

- Walk the nodes breadth first. Count every declaration with a defined value (`count`).
- For each declaration, look up its property in the global property order (Section 11.4). Add
  the index when found. A declaration `--lm-sort: <property>` contributes the index of
  `<property>` and stops further property lookups for this node list.
- `order` is the sorted list of collected indices.

Sort comparison for two rule nodes:

1. Ascending `variantOrder` (as an arbitrary-precision integer).
2. The first differing entry of `order`, ascending; a missing entry counts as infinity.
3. Descending `count`.
4. `compare(candidateA, candidateZ)`.

### 11.4 Global Property Order

The ordered list of properties that determines the relative position of generated rules. An
implementation MUST use this list verbatim.

```text
container-type pointer-events visibility position inset inset-inline inset-block
inset-inline-start inset-inline-end inset-block-start inset-block-end top right bottom left
isolation z-index order grid-column grid-column-start grid-column-end grid-row grid-row-start
grid-row-end float clear --lm-container-component margin margin-inline margin-block
margin-inline-start margin-inline-end margin-block-start margin-block-end margin-top margin-right
margin-bottom margin-left box-sizing display field-sizing aspect-ratio height max-height
min-height width max-width min-width flex flex-shrink flex-grow flex-basis table-layout
caption-side border-collapse border-spacing --lm-border-spacing-x --lm-border-spacing-y
transform-origin translate --lm-translate-x --lm-translate-y --lm-translate-z scale --lm-scale-x
--lm-scale-y --lm-scale-z rotate --lm-rotate-x --lm-rotate-y --lm-rotate-z --lm-skew-x
--lm-skew-y transform zoom animation cursor touch-action --lm-pan-x --lm-pan-y --lm-pinch-zoom
resize scroll-snap-type --lm-scroll-snap-strictness scroll-snap-align scroll-snap-stop
scroll-margin scroll-margin-inline scroll-margin-block scroll-margin-inline-start
scroll-margin-inline-end scroll-margin-block-start scroll-margin-block-end scroll-margin-top
scroll-margin-right scroll-margin-bottom scroll-margin-left scroll-padding scroll-padding-inline
scroll-padding-block scroll-padding-inline-start scroll-padding-inline-end
scroll-padding-block-start scroll-padding-block-end scroll-padding-top scroll-padding-right
scroll-padding-bottom scroll-padding-left scrollbar-width scrollbar-color scrollbar-gutter
list-style-position list-style-type list-style-image appearance columns break-before
break-inside break-after grid-auto-columns grid-auto-flow grid-auto-rows grid-template-columns
grid-template-rows flex-direction flex-wrap place-content place-items align-content align-items
justify-content justify-items gap column-gap row-gap --lm-space-x-reverse --lm-space-y-reverse
divide-x-width divide-y-width --lm-divide-y-reverse divide-style divide-color place-self
align-self justify-self overflow overflow-x overflow-y overscroll-behavior overscroll-behavior-x
overscroll-behavior-y scroll-behavior border-radius border-start-radius border-end-radius
border-top-radius border-right-radius border-bottom-radius border-left-radius
border-start-start-radius border-start-end-radius border-end-end-radius border-end-start-radius
border-top-left-radius border-top-right-radius border-bottom-right-radius
border-bottom-left-radius border-width border-inline-width border-block-width
border-inline-start-width border-inline-end-width border-block-start-width
border-block-end-width border-top-width border-right-width border-bottom-width
border-left-width border-style border-inline-style border-block-style border-inline-start-style
border-inline-end-style border-block-start-style border-block-end-style border-top-style
border-right-style border-bottom-style border-left-style border-color border-inline-color
border-block-color border-inline-start-color border-inline-end-color border-block-start-color
border-block-end-color border-top-color border-right-color border-bottom-color
border-left-color background-color background-image --lm-gradient-position
--lm-gradient-stops --lm-gradient-via-stops --lm-gradient-from --lm-gradient-from-position
--lm-gradient-via --lm-gradient-via-position --lm-gradient-to --lm-gradient-to-position
mask-image --lm-mask-top --lm-mask-top-from-color --lm-mask-top-from-position
--lm-mask-top-to-color --lm-mask-top-to-position --lm-mask-right --lm-mask-right-from-color
--lm-mask-right-from-position --lm-mask-right-to-color --lm-mask-right-to-position
--lm-mask-bottom --lm-mask-bottom-from-color --lm-mask-bottom-from-position
--lm-mask-bottom-to-color --lm-mask-bottom-to-position --lm-mask-left --lm-mask-left-from-color
--lm-mask-left-from-position --lm-mask-left-to-color --lm-mask-left-to-position
--lm-mask-linear --lm-mask-linear-position --lm-mask-linear-from-color
--lm-mask-linear-from-position --lm-mask-linear-to-color --lm-mask-linear-to-position
--lm-mask-radial --lm-mask-radial-shape --lm-mask-radial-size --lm-mask-radial-position
--lm-mask-radial-from-color --lm-mask-radial-from-position --lm-mask-radial-to-color
--lm-mask-radial-to-position --lm-mask-conic --lm-mask-conic-position
--lm-mask-conic-from-color --lm-mask-conic-from-position --lm-mask-conic-to-color
--lm-mask-conic-to-position box-decoration-break background-size background-attachment
background-clip background-position background-repeat background-origin mask-composite
mask-mode mask-type mask-size mask-clip mask-position mask-repeat mask-origin fill stroke
stroke-width object-fit object-position padding padding-inline padding-block
padding-inline-start padding-inline-end padding-block-start padding-block-end padding-top
padding-right padding-bottom padding-left text-align text-indent vertical-align font-family
font-feature-settings font-size line-height font-weight letter-spacing text-wrap overflow-wrap
word-break text-overflow hyphens white-space tab-size color text-transform font-style
font-stretch font-variant-numeric text-decoration-line text-decoration-color
text-decoration-style text-decoration-thickness text-underline-offset -webkit-font-smoothing
placeholder-color caret-color accent-color color-scheme opacity background-blend-mode
mix-blend-mode box-shadow --lm-shadow --lm-shadow-color --lm-ring-shadow --lm-ring-color
--lm-inset-shadow --lm-inset-shadow-color --lm-inset-ring-shadow --lm-inset-ring-color
--lm-ring-offset-width --lm-ring-offset-color outline outline-width outline-offset
outline-color --lm-blur --lm-brightness --lm-contrast --lm-drop-shadow --lm-grayscale
--lm-hue-rotate --lm-invert --lm-saturate --lm-sepia filter --lm-backdrop-blur
--lm-backdrop-brightness --lm-backdrop-contrast --lm-backdrop-grayscale
--lm-backdrop-hue-rotate --lm-backdrop-invert --lm-backdrop-opacity --lm-backdrop-saturate
--lm-backdrop-sepia backdrop-filter transition-property transition-behavior transition-delay
transition-duration transition-timing-function will-change contain content
forced-color-adjust
```

## 12. Output Optimization and Serialization

`optimizeAst(ast)` runs on the whole document (stylesheet plus generated utilities) before every
serialization.

### 12.1 Transformation Pass

Build a new AST depth first:

- Declaration: drop `--lm-sort` and undefined values. Inside `context { theme: true }` a `--`
  declaration is tracked for pruning (Section 7.6); a value of `initial` is dropped. When the
  value contains `var(`, record variable usage (as a dependency when the declaration is itself a
  theme variable, as a use otherwise). An `animation` declaration records keyframe names (split
  on whitespace and commas).
- Rule: transform children; drop the rule when it becomes empty.
- `@property` at depth 0: print each registration name once; later duplicates are dropped.
- Other at-rules: transform children. Drop empty at-rules except `@layer`, `@charset`,
  `@custom-media`, `@namespace`, `@import`, and `@apply`. Record `@keyframes` found inside
  `context { theme: true }` as prunable.
- `at-root`: transform children at depth 0 and collect them into a separate list.
- `context`: when `reference` is set, drop the subtree. Otherwise transform children into the
  current parent while merging the context map.
- Comment: keep.

### 12.2 Post-processing

1. Prune unused theme variables and keyframes (Section 7.6).
2. Append the collected root-level nodes to the end of the document.
3. Flatten nesting (Section 12.3).

### 12.3 Nesting Flattening

The output MUST be flat CSS:

- A nested style rule is merged with its parent selector by replacing `&`. A nested selector
  without `&` is treated as `& <selector>` only when it is a relative selector; otherwise `&` is
  prepended. A parent selector list is wrapped in `:is(...)` before substitution.
- A rule whose selector is exactly `&` is replaced by its children.
- At-rules nested inside style rules are hoisted above the rule; the rule is emitted inside the
  at-rule. Nested at-rules remain nested in each other.
- Declarations that precede nested rules stay with the parent rule, which is emitted before the
  hoisted children.

Example: `.hover\:underline { &:hover { @media (hover: hover) { text-decoration-line: underline; } } }`
becomes:

```css
@media (hover: hover) {
  .hover\:underline:hover {
    text-decoration-line: underline;
  }
}
```

### 12.4 Polyfills (OPTIONAL)

- Property registration fallback: for every `@property`, collect a declaration
  `<name>: <initial-value or initial>`; group those with `inherits: true` under `:root, :host`
  and the rest under `*, ::before, ::after, ::backdrop`; emit them at the end of the document
  as `@layer properties { @supports ((-webkit-hyphens: none) and (not (margin-trim: inline))) or
  ((-moz-orient: inline) and (not (color:rgb(from red r g b)))) { ... } }` and insert an empty
  `@layer properties;` statement after any leading license comments, `@charset`, and external
  `@import` statements.
- Color-mix fallback: for each declaration using `color-mix(...)` with `var(...)` arguments,
  emit a copy with the variables replaced by their raw theme values (color space `srgb`)
  followed by `@supports (color: color-mix(in lab, red, red)) { <original> }`.

## 13. Source Scanning and Candidate Extraction

### 13.1 Source Set Assembly (Host Responsibility)

```text
sources = []
if compiler.root == "none":       (add nothing)
else if compiler.root == null:    sources += { base: cwd, pattern: "**/*", negated: false }
else:                             sources += { ...compiler.root, negated: false }
sources += compiler.sources
sources += { base: dirname(executable), pattern: basename(executable), negated: true }
if input file:                    sources += { base: dirname(input), pattern: basename(input), negated: false }
```

### 13.2 Auto-detection Rules

For a `**/*` source the scanner walks `base` recursively and excludes:

- Paths matched by any `.gitignore` encountered along the way.
- Directories named `.git`, `.hg`, `.jj`, `.next`, `.parcel-cache`, `.pnpm-store`,
  `.svelte-kit`, `.svn`, `.turbo`, `.venv`, `.vercel`, `.yarn`, `__pycache__`, `node_modules`,
  `venv`.
- Files with the extensions `less`, `lock`, `sass`, `scss`, `styl`, `log`.
- Files with common binary extensions (images, audio, video, archives, fonts, executables).
- Files named `package-lock.json`, `pnpm-lock.yaml`, `bun.lockb`, `.gitignore`, `.env`, and
  `.env.*`.
- Paths matched by a negated source.

An explicit `@source "<glob>"` bypasses the ignore rules above so that ignored directories can be
opted in. Globs support `**`, `*`, and `{a,b}`.

### 13.3 Scanner Interface

- `new(sources)`.
- `scan()`: walk every source, read files whose modification time changed since the last scan
  (all files on the first scan), extract candidates, and return the full deduplicated candidate
  set seen so far.
- `scanFiles(changed)`: read only the given files and return only candidates not seen before.
- `scannedFiles`: the files read by the most recent `scan()`.

### 13.4 Extraction Rules

Extraction MUST favor recall over precision: an unrecognized candidate produces no CSS and is
harmless, while a missed candidate is a visible bug. Implementations SHOULD still reject obvious
prose to keep the candidate set small.

Scan the input as bytes and emit every maximal span that satisfies:

- The byte before the span is a start boundary: whitespace, a quote (`"`, `'`, or backtick),
  start of input, `.`, `}`, or `>`.
- The byte after the span is an end boundary: whitespace, a quote, end of input, `]`, `{`, `=`,
  `\`, or `<`.
- The span matches `(variant ":")* utility` where:
  - a variant is `[...]` with balanced brackets, or a name starting with a letter or `@`
    followed by letters, digits, `_`, `-`, an optional `-[...]` or `-(...)`, and an optional
    `/modifier`;
  - a utility is `[property:value]` with balanced brackets, or a name starting with a letter,
    `@`, or `-` followed by a letter or digit (`-@` is not allowed), continuing with letters,
    digits, `_`, `-`, `.` between digits, `%`, `-[...]`, `-(...)`, an optional trailing
    `/modifier` (`/[...]`, `/(...)`, or a name), and an optional trailing `!`;
  - `-` and `_` MUST NOT end a name; a leading `!` is accepted.
- Additionally emit every `--` followed by letters, digits, `_`, or `-` under the same boundary
  rules (custom property references).

## 14. Command-Line Interface

### 14.1 Invocation

```text
loom [build] [--input input.css] [--output output.css] [--watch] [--poll=ms] [options...]
```

Options:

- `-i, --input <path>`: entry stylesheet; `-` reads stdin. When omitted the input is the single
  line `@import 'loom';`.
- `-o, --output <path>`: output file; `-` (the default) writes stdout.
- `-w, --watch [always]`: rebuild on changes. `always` keeps watching after stdin closes.
- `--poll [ms]`: poll instead of using filesystem events; the default interval is 250 ms. A
  non-positive interval is an error.
- `-m, --minify`: optimize and minify the output.
- `--optimize`: optimize without minifying.
- `--cwd <dir>`: base directory (default `.`).
- `--silent`: suppress everything except errors.
- `-h, --help`: usage.

Behavior:

- A missing input file, or identical input and output paths, MUST exit with status 1 and a
  message.
- With no arguments on a TTY the tool prints usage.
- The banner and `Done in <duration>` MUST go to stderr so that stdout stays clean for CSS.
- When writing to stdout, output MUST be printed only when it differs from the previous write.

### 14.2 Single Build

1. Read the input. Call `compile(css, { base: dirname(input) or cwd, loadStylesheet })`. The
   loader resolves relative ids against `base`, resolves the id `loom` to the bundled
   `index.css`, and records every loaded path as a full-rebuild path.
2. Assemble sources (Section 13.1) and create the scanner.
3. `candidates = scanner.scan()`; `css = compiler.build(candidates)`; write (Section 14.5).

### 14.3 Watch Mode (Event Driven)

Watch every distinct source `base` directory recursively. On a batch of changed paths:

- Ignore the batch when it contains only the output file.
- If any changed path is a full-rebuild path: re-read the input, recreate the compiler and the
  scanner, run `scan()`, rebuild, replace the watchers, and write.
- Otherwise: `newCandidates = scanner.scanFiles(changed)`; when empty, do nothing; otherwise
  `compiler.build(newCandidates)` and write.
- Report errors to stderr and keep watching. When a full rebuild fails, restore the previous
  full-rebuild path list so that a later change to a deleted dependency still triggers a rebuild.
- Exit when stdin reaches end of file unless `--watch=always`.

### 14.4 Watch Mode (Polling)

Every interval: `candidates = scanner.scan()`; let `files` be `scannedFiles` minus the output
file; when `files` is empty, do nothing. If any file is a full-rebuild path, perform a full
rebuild as above. Otherwise, when `candidates` is non-empty, `compiler.build(candidates)` and
write.

### 14.5 Writing

1. When `--minify` or `--optimize` is set, pass the CSS through the minifier (skipping it when
   the CSS equals the previous build's CSS). The minifier is implementation-defined; at minimum
   it MUST preserve semantics, and it SHOULD flatten `@media` range syntax to `min-width` and
   `max-width` for older browsers.
2. Write to the output file (creating directories as needed) or to stdout.

## 15. Reference Algorithms (Language-Agnostic)

### 15.1 Compile

```text
function compile(css, options):
  ast = parse_css(css)
  ast = [context({base: options.base}, ast)]
  features = substitute_at_imports(ast, options.base, options.loadStylesheet)

  theme = new Theme()
  state = collect_directives(ast, theme)      # Sections 6.4 .. 6.8
  design = build_design_system(theme)
  design.important = state.important
  design.invalidCandidates += state.ignoredCandidates

  for name in state.customVariants (stylesheet order):
    design.variants.static(name, noop)         # reserve order
  for name in topological_sort(state.customVariantDependencies):
    state.customVariants[name](design)
  for register in state.customUtilities:
    register(design)

  emit_theme_variables(state.firstThemeRule, design.theme)
  features |= substitute_at_variant(ast, design)
  features |= substitute_functions(ast, design)
  features |= substitute_at_apply(ast, design)
  convert_utilities_node_to_context(state.utilitiesNode)
  remove_at_utility_nodes(ast)

  return make_handle(ast, design, state, features, options)
```

### 15.2 Build

```text
function build(candidates):
  if features == NONE: return original_css
  if utilitiesNode == null: return serialize(optimize(ast))

  changed = pendingInlineCandidates
  for c in candidates:
    if c in design.invalidCandidates: continue
    if c starts with "--":
      changed |= design.theme.markUsedVariable(c)
    else:
      changed |= validCandidates.add(c)
  if not changed: return cached_output

  nodes = compile_candidates(validCandidates, design, on_invalid = add_to_invalid).nodes
  if nodes.length == previousCount and no variable was newly marked: return cached_output
  previousCount = nodes.length
  utilitiesNode.nodes = nodes
  cached_output = serialize(optimize(ast))
  return cached_output
```

### 15.3 Compile Candidates

```text
function compile_candidates(raws, design, options):
  matches = []
  for raw in raws:
    if raw in design.invalidCandidates: options.on_invalid(raw); continue
    parsed = design.parseCandidate(raw)
    if parsed is empty: options.on_invalid(raw); continue
    matches.append((raw, parsed))

  order = design.getVariantOrder()
  rules = []
  for (raw, parsed) in matches:
    found = false
    for candidate in parsed:
      for (node, propertySort) in design.compileAstNodes(candidate, flags(options)):
        found = true
        variantOrder = OR over v in candidate.variants of (1 << order[v])
        rules.append((node, propertySort, variantOrder, raw))
    if not found: options.on_invalid(raw)

  sort rules by (variantOrder asc, first differing property index asc, count desc, compare(raw))
  return rules
```

### 15.4 Apply Variant

```text
function apply_variant(node, variant, depth = 0):
  if variant.kind == "arbitrary":
    if variant.relative and depth == 0: return REJECT
    node.nodes = [rule(variant.selector, node.nodes)]
    return OK

  definition = variants[variant.root]
  if variant.kind == "compound":
    isolated = at_rule("@slot")
    if apply_variant(isolated, variant.variant, depth + 1) == REJECT: return REJECT
    if variant.root == "not" and len(isolated.nodes) > 1: return REJECT
    for child in isolated.nodes:
      if child.kind not in (rule, at-rule): return REJECT
      if definition.apply(child, variant) == REJECT: return REJECT
    fill_empty_rules(isolated.nodes, with = node.nodes)
    node.nodes = isolated.nodes
    return OK

  return definition.apply(node, variant)
```

### 15.5 Watch Tick (Event Driven)

```text
on_changes(files):
  if files == [output_path]: return
  if any(f in full_rebuild_paths for f in files):
    input = read_input()
    (compiler, scanner) = create_compiler(input)
    candidates = scanner.scan()
    replace_watchers(scanner)
    write(compiler.build(candidates))
  else:
    new_candidates = scanner.scanFiles(files)
    if new_candidates is empty: return
    write(compiler.build(new_candidates))
```

## 16. Conformance Examples

All examples use the bundled default theme and show the serializer's raw output (before any
minifier). Whitespace follows Section 5.2.

### 16.1 Basic Document

Stylesheet:

```css
@theme {
  --color-black: #000;
  --breakpoint-md: 768px;
}
@layer utilities {
  @loom utilities;
}
```

Candidates: `flex`, `md:grid`, `hover:underline`, `dark:bg-black`.

Output:

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

`--breakpoint-md` is not printed because no `var(...)` references it.

### 16.2 Value Forms

Each line gives a candidate and the declarations it produces.

```text
p-4                  padding: calc(var(--spacing) * 4);
p-1                  padding: var(--spacing);
p-0                  padding: 0px;
p-px                 padding: 1px;
-mt-2                margin-top: calc(var(--spacing) * -2);
w-1/2                width: calc(1 / 2 * 100%);
w-[13px]             width: 13px;
w-(--my-w)           width: var(--my-w);
max-w-md             max-width: var(--container-md);
bg-red-500           background-color: var(--color-red-500);
bg-red-500/50        background-color: color-mix(in oklab, var(--color-red-500) 50%, transparent);
bg-[#0088cc]         background-color: #0088cc;
bg-[url(/a_b.png)]   background-image: url(/a_b.png);
bg-[length:10px_20px] background-size: 10px 20px;
text-lg              font-size: var(--text-lg); line-height: var(--lm-leading, var(--text-lg--line-height));
text-lg/8            font-size: var(--text-lg); line-height: calc(var(--spacing) * 8);
text-red-500         color: var(--color-red-500);
font-bold            --lm-font-weight: var(--font-weight-bold); font-weight: var(--font-weight-bold);
                     (plus a hoisted @property --lm-font-weight)
rounded-lg           border-radius: var(--radius-lg);
rounded-full         border-radius: calc(infinity * 1px);
border               border-style: var(--lm-border-style); border-width: 1px;
                     (plus a hoisted @property --lm-border-style with initial-value: solid)
border-2             border-style: var(--lm-border-style); border-width: 2px;
z-10                 z-index: 10;
-z-10                z-index: calc(10 * -1);
flex-1               flex: 1;
opacity-50           opacity: 50%;
[mask-type:luminance] mask-type: luminance;
[--my-var:1px]       --my-var: 1px;
underline!           text-decoration-line: underline !important;
```

### 16.3 Variant Forms

Each line gives a candidate and the selector or wrapper it produces around `.<escaped name>`.

```text
hover:flex             @media (hover: hover) { .hover\:flex:hover { ... } }
focus:flex             .focus\:flex:focus
sm:flex                @media (width >= 40rem) { .sm\:flex { ... } }
max-md:flex            @media (width < 48rem) { ... }
min-[600px]:flex       @media (width >= 600px) { ... }
@md:flex               @container (width >= 28rem) { ... }
@md/main:flex          @container main (width >= 28rem) { ... }
group-hover:flex       @media (hover: hover) { .group-hover\:flex:is(:where(.group):hover *) { ... } }
group-hover/item:flex  .group-hover\/item\:flex:is(:where(.group\/item):hover *)
peer-checked:flex      .peer-checked\:flex:is(:where(.peer):checked ~ *)
has-[>img]:flex        .has-\[\>img\]\:flex:has(> img)
in-data-visible:flex   :where([data-visible]) .in-data-visible\:flex
not-hover:flex         .not-hover\:flex:not(:hover)  and  @media not all and (hover: hover) { .not-hover\:flex { ... } }
not-supports-grid:flex @supports not (grid: var(--lm)) { ... }
data-[state=open]:flex .data-\[state\=open\]\:flex[data-state="open"]
aria-checked:flex      .aria-checked\:flex[aria-checked="true"]
nth-3:flex             .nth-3\:flex:nth-child(3)
[&_p]:flex             .\[\&_p\]\:flex p
[@media(width>=100px)]:flex   @media (width>=100px) { ... }
*:flex                 :is(.\*\:flex > *)
before:block           .before\:block::before { content: var(--lm-content); display: block; }
dark:hover:flex        @media (prefers-color-scheme: dark) { @media (hover: hover) { .dark\:hover\:flex:hover { ... } } }
```

### 16.4 Custom Utilities and `@apply`

Stylesheet:

```css
@utility tab-* {
  tab-size: --value(integer);
  tab-size: --value(--tab-size-*);
  tab-size: --value([integer]);
}
@utility content-auto {
  content-visibility: auto;
}
.btn {
  @apply rounded-lg px-4 py-2 hover:bg-red-500;
}
@loom utilities;
```

Candidates: `tab-4`, `tab-[8]`, `content-auto`.

Output:

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

### 16.5 Theme Customization

```css
@import "loom";
@theme {
  --color-*: initial;
  --color-primary: oklch(0.6 0.2 250);
  --breakpoint-3xl: 120rem;
  --font-display: "Inter", sans-serif;
}
@custom-variant dark (&:where(.dark, .dark *));
```

- `bg-primary`, `3xl:flex`, and `font-display` compile; `bg-red-500` does not.
- `dark:` produces a class-based selector instead of a media query.

## 17. Test and Validation Matrix

A conforming implementation SHOULD include tests that cover the behaviors defined in this
specification.

### 17.1 Stylesheet Parsing and Serialization

- Nested rules, nested at-rules, and `&` parse into the documented node shapes
- Ordinary comments are dropped and `/*!` comments are kept
- `!important` is split from values
- A missing trailing `;` before `}` is accepted
- Unbalanced blocks raise a positioned syntax error
- Serialization matches Section 5.2 byte for byte

### 17.2 Directives

- `@import` with `layer()`, `supports()`, and media queries wraps content in the documented order
- `layer()` after `supports()` raises an error
- `url()`, `data:`, and `http(s)` imports are preserved verbatim
- `@reference` behaves like `@import ... reference` and prints nothing
- `@import "loom" important`, `prefix(...)`, `source(...)`, and `theme(reference)` take effect
- `@theme` rejects non-custom-property children with a snippet in the error
- `--color-*: initial` clears the namespace but keeps ignored sub-namespaces
- `@theme default` values lose to author values regardless of order
- `@source`, `@source not`, `@source inline(...)` with brace expansion, and
  `@source not inline(...)` behave as specified and raise the documented errors
- `@custom-variant` selector form, body form with `@slot`, at-rule selectors, dependency ordering,
  and cycle detection
- `@utility` static and functional forms, invalid names, empty bodies, and `--value` /
  `--modifier` rules including `ratio` exclusivity
- Nested `@variant` with comma alternatives and colon stacking; unknown names raise
- `@apply` inlines utilities, keeps variant wrappers, ignores mixins, rejects mixed arguments,
  rejects unknown candidates with the documented messages, and detects cycles
- `--spacing()`, `--alpha()`, `--theme()` with fallbacks and `inline`, and legacy `theme()`

### 17.3 Candidate Parsing

- Static, functional, arbitrary property, arbitrary value, and variable shorthand forms
- Type hints, empty arbitrary values, and `;` or `}` inside arbitrary values
- Modifiers in named, bracket, and parenthesis forms; more than one modifier is invalid
- Fractions are recorded only for named modifiers
- Leading and trailing `!`
- Prefix enforcement when a prefix is configured
- `findRoots` yields every valid split and stops on an empty remainder
- Underscore decoding, `\_`, `url()` and `var()` exemptions, and math operator spacing

### 17.4 Variants and Utilities

- Every built-in variant in Section 9.5 produces the documented output
- Compound variants reject incompatible inner variants and relative arbitrary selectors at the
  top level
- `not` rejects inner variants that yield more than one rule
- Breakpoint and container groups sort ascending or descending by value
- Every utility in Section 10.8 produces the documented declarations for named, arbitrary,
  negative, fraction, and default forms
- Opacity modifiers use `color-mix` and reject non-quarter values
- `text-*` line-height modifiers and `bg-*` type inference

### 17.5 Compilation and Output

- Output order is independent of candidate discovery order
- Variant order, property order, declaration count, and natural name order act as tie breakers
- `important` from the stylesheet and from candidates marks declarations, except inside
  hoisted registrations
- `@property` registrations print once and are hoisted to the end
- Unused theme variables and keyframes are pruned; `static` values and values referenced by
  scanned `--name` candidates survive
- Nesting is flattened as specified
- `build` returns identical output for a second call with no new candidates

### 17.6 Scanning and CLI

- Auto-detection respects `.gitignore` and the built-in ignore lists; explicit globs bypass them
- Extraction finds candidates in HTML attributes, template literals, and object keys and respects
  boundary rules
- `scanFiles` returns only new candidates
- Single build writes to a file or stdout; stdout is skipped when unchanged
- Missing input and identical input and output exit with status 1
- Watch mode performs incremental rebuilds for source changes and full rebuilds for stylesheet
  changes, and keeps running after an error
- Polling mode rebuilds on modification time changes

## 18. Implementation Checklist (Definition of Done)

### 18.1 REQUIRED for Conformance

- CSS parser and serializer per Section 5
- Import resolution with a loader callback and the bundled `index.css`, `theme.css`,
  `preflight.css`, and `utilities.css`
- `@theme` with all four modes, namespace clearing, ignored sub-namespaces, and prefixing
- `@source` in all four forms with brace expansion
- `@custom-variant` in both forms with dependency ordering
- `@utility` static and functional forms with `--value` and `--modifier`
- Nested `@variant`, `@apply`, and the four theme functions
- Candidate parser per Section 8 with memoization
- Variant registry, built-in variants in the documented order, compound application, and
  ordering per Section 9
- Utility registry, definition helpers, color handling, and the catalog in Section 10.8
- Compilation, importance, and ordering per Section 11 with the global property order
- Optimization per Section 12 including pruning and flattening
- Scanner with auto-detection, explicit globs, incremental scanning, and the extraction rules
- CLI with single build, event-driven watch, polling watch, and the documented flags

### 18.2 RECOMMENDED Extensions

- The property registration and color-mix polyfills of Section 12.4
- The additional utility families named at the end of Section 10.8
- A minifier that downlevels media range syntax and nesting for older browsers
- Warnings for unsupported `--value` data types with a caret pointing at the offending argument
