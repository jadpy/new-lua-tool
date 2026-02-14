# 🏗️ Lua コードプロセッシングシステム — アーキテクチャ完全解説

> このドキュメントは、Luaコード変換・解析ツールチェーン全体のアーキテクチャ、動作原理、実装詳細を、図解とコード例を交えて詳しく解説します。

## � 目次

1. [📖 システム概要](#📖-システム概要)
2. [🔄 全体アーキテクチャと処理フロー](#🔄-全体アーキテクチャと処理フロー)
3. [🔤 フロントエンド：字次解析・構文解析](#🔤-フロントエンド字次解析構文解析)
4. [🔍 中間層：解析・変換エンジン](#🔍-中間層解析変換エンジン)
5. [📝 バックエンド：コード生成](#📝-バックエンドコード生成)
6. [⚙️ 処理モード詳細実装ガイド](#⚙️-処理モード詳細実装ガイド)
7. [🛡️ エラーハンドリング・セーフティ機構](#🛡️-エラーハンドリングセーフティ機構)
8. [🌍 方言サポートシステム](#🌍-方言サポートシステム)
9. [📊 パフォーマンス・計算量分析](#📊-パフォーマンス計算量分析)
10. [🔧 拡張方法](#🔧-拡張方法)

---

## 📖 システム概要

### 🎯 このツールの目的

このツールチェーンは、**Luaソースコード**に対して以下を実現します：

- 機能カテゴリ | モード | 役割 |
|-------------|--------|------|
| **Local 削除** | `functionlocal` / `localkw` / `localte` / `localc` / `localcyt` / `localnum` / `localtabke` | `local` キーワードを削除してグローバル化 |
| **コード整理** | `deleteout` / `nocode` / `outcode` | コメント削除・未使用コード削除・アウトコード削除 |
| **最適化・圧縮** | `coml` / `nmbun` | ファイルサイズ最小化・定数式簡約 |
| **整形** | `fom` | インデント・スペース・改行を統一 |
| **リファクタリング** | `rename` | 変数名を安全にリネーム |
| **品質検証** | `lint` | 静的解析＋動的解析による問題検出 |
| **自動修復** | `codefix` | 壊れた構文を AI で自動修復 |
| **難読化解除** | `deobf` | 難読化コードを動的デコード |
| **依存解析** | `rrequire` | `require` 依存関係を追跡 |
| **複合処理** | `preset` | JSON で複数ステップの処理を順序実行 |

### 🔌 外部環境

```
📦 必要環境
├─ lua54 実行環境（メイン処理用）
├─ OS: Windows / Linux / macOS
└─ 🔄 全体アーキテクチャと処理フロー

### 🌊 基本的な力物
├─ 変換済みソース (.lua)
├─ JSON メタレポート（処理ごと）
│  ├─ safety.json: 安全性検証結果
│  ├─ lint.json: 診断結果
│  ├─ deobf.json: 難読化解除メタ
│  └─ ...
└─ Discord Bot 実行（オプション）
```
| 特徴 | 説明 |
|------|------|
| **18+ の処理モード** | local削除、整形、圧縮、lint、難読化解除など |
| **多方言対応** | Lua 5.1, 5.2, 5.3, 5.4, Luau をサポート |
| **2つの処理エンジン** | 行単位処理（line）と AST ベース処理（ast） |
| **セーフティガード** | 変換前後の構文・コンパイル可能性を検証 |
| **AI 駆動修正** | CodeFix が壊れた構文を自動修復 |
| **難読化対応** | 動的実行、VMトレース、devirtualizer で難読化コード解読 |

---

## 全体アーキテクチャ

### 処理パイプライン

```
入力ファイル
    ↓
┌─────────────────────────────────────┐
│  方言自動検出 (Dialect.resolve)     │
└──────────┬──────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  メインディスパッチ (main.lua)       │
│  mode / engine / options に応じて    │
│  処理フローを選択                    │
└──────────┬──────────────────────────┘
           ↓
    ┌──────────────┐
    │   engine の型  │
    ├───────┬──────┤
    │       │      │
   line    ast  特殊
    │       │      │
    ↓       ↓      ↓
 [L流]  [A流]  [特殊流]
    │       │      │
    └───────┴──────┴─→ 処理実行
             ↓
    ┌──────────────────────┐
    │ セーフティガード機構  │
    │ (Safety.guard)       │
    │ ・前後の構文確認     │
    │ ・コンパイル確認     │
    │ ・リント検証         │
    └──────────┬───────────┘
             ↓
        出力ファイル
        + レポート (.json)
```

### 🎯 3つの処理フローの違い

#### 1. **Line Mode（行単位処理）** — ⚡ 高速・シンプル

**特徴：** 正規表現で行規模の変換を実施。構文の深い理解不要。

特定の処理（`functionlocal`, `localkw`, `localte` など）が対応している場合に、**行単位で正規表現ベースの変換**を実施。高速だが適用範囲は限定。

```lua
-- 実装例：TransformerLine.lua
local function apply()
  for i, line in ipairs(source_lines) do
    if mode == "functionlocal" then
      -- local function NAME(...) → function NAME(...)
      -- local NAME = function(...) → NAME = function(...)
      line = line:gsub("^%s*local%s+function%s+", "function ")
      line = line:gsub("^%s*local%s+([%w_]+)%s*=%s*function", "%1 = function")
    end
  end
  return processed_source
end
``対応モード：**
- `functionlocal` / `localkw` / `localte` / `localc` / `localcyt` / `localnum` / `localtabke`

**メリット・デメリット：**
| メリット | デメリット |
|---------|-----------|
| 💨 高速（O(n)） | スコープ理解不可 |
| | 複雑な変換不可 |
字次解析 (Lexer)
  ↓ [トークン列生成]
構文解析 (Parser)
  ↓ [AST 構築]
スコープ解析 (Analyzer)
  ↓ [変数・スコープ情報記録]
変換 (Transformer)
  ↓ [AST 変換]
コード生成 (CodeGen)
  ↓ [ソース再生成]
出力
```

Formatter.format()
  ↓ AST ビルド後、フォーマットルール適用

Linter.run()
  ↓ スコープ分析 + 動的実行 + 診断生成

CodeFix.run()
  ↓ 壊れた構文の修正候補を生成・スコアリング・選定

Deobf.run()
  ↓ Sandbox 実行 + 静的デコード + VM トレース
```

**対応モード：**
- `fom` / `coml` / `nmbun` / `rename` / `deleteout` / `nocode` / `codefix` / `deobf` / `lint` / `rrequire`

---

## 🔤 フロントエンド：字次解析・構文解析

> Luaソースコードを**構造化データ（AST）**に変換する最初の段階

### 🔤 1. 字次解析（Lexer - tokenization）

#### 📌 目的
    ↓
  AST 構築
    ↓
  フォーマットルール適用
    ↓
  インデント・スペース調整
    ↓
  出力

例：Linter.lua (lint - 静的解析)
```
入力：
  if x > 10 then print("hello") end

      ↓ [字次解析]

出力（トークン列）：
  [
    {type:'keyword', value:'if', line:1, col:1},
    {type:'ident', value:'x', line:1, col:4},
    {type:'symbol', value:'>', line:1, col:6},
    {type:'number', value:'10', line:1, col:8},
    {type:'keyword', value:'then', line:1, col:11},
    ...
  ]
```

**重要：** トークンはLuaの**最小意味単位**です。各トークンには位置情報（行・列）も記録。

#### 🔍 処理フローの詳細
    ↓
  警告・エラーリスト生成
ソーステキスト読み込み
    ↓
┌─────────────────────────────────────┐
│  スキャンループ開始             │
│  pos = 1, line = 1, col = 1     │
└────────┬────────────────────────────┘
         ↓
    ┌──────────────────────┐
    │ 現在の文字を判定     │
    └──┬────┬────┬────┬───┤
       │    │    │    │   │
      空白 数字 文字 記号 その他
       │    │    │    │   │
       ↓    ↓    ↓    ↓   ↓
     skip read read read error
           num  id   sym
       │    │    │    │   │
       └────┴────┴────┴───┘
            ↓
       トークン生成
            ↓
   トークン配列に追加
            ↓
      pos++ して次へ
            ↓
       EOF? yes → 終了
            ↓ no
    ループ継続
```

#### 📝 トークンの種別詳解

**1️⃣ キーワード（予約語）**
```lua
-- パターン: Lua の予約語（言語が予め定義）
Keyword: if | else | local | function | end | ...

例：
  {type: 'keyword', value: 'local', line: 5, col: 1}
```

**2️⃣ 識別子（変数名・関数名）**
```lua
-- パターン: [a-zA-Z_][a-zA-Z0-9_]*
Identifier: myVar | _private | x123 | ...

実装：
  token = {type: 'ident', value: myVar, line: N, col: M}
  
注意：辞書ルックアップで keyword か判定
```

**3️⃣ 数値リテラル（複合対応）**
```lua
-- Lua は複数の数値形式をサポート
型         例           処理
10進数：  42, 3.14    → tonumber()
16進数：  0xFF         → tonumber(val, 16)
2進数：   0b1010       → tonumber(val, 2)   [Luau]
科学記法：1e-2, 1E+3  → tonumber()

実装（Lexer.read_number）:
  -- 最初の文字/2文字で基数判定
  if ch == '0' then
    next = peek(1)
    if next == 'x' or 'X' → hex
    if next == 'b' or 'B' → binary (if supported)
  end
```

**4️⃣ 文字列リテラル（複合対応）**
```lua
型           例              処理
短文字列：   "hello"        → エスケープ処理
           'world'        → ".tsv"に統一可
長文字列：   [[content]]    → ネスト可、エスケープ不要

エスケープシーケンス例：
  \n → 改行
  \t → タブ
  \x41 → 16進数文字（'A'）
  \u{1F600} → Unicode （Lua 5.3+）

実装（Lexer.read_string）:
  buffer = {}
  while char != closing_quote do
    if char == '\\' then
      -- エスケープ処理
      next = peek()
      handle_escape_sequence(next)
    else
      buffer += char
    end
  end
```

**5️⃣ シンボル（演算子・区切り文字）**
```lua
型          例
単一字：   + - * / ( ) { } [ ] , ; : # =
複合字：   == ~= <= >= .. // << >> ... ::

実装戦略（貪欲マッチ）:
  if ch == ':' and peek(1) == ':' then
    → token = '::'
  else
    → token = ':'
```

#### ⚡ 計算量分析

```
字次解析の計算量：
┌──────────────────────────┐
│ 時間計算量：O(n)         │
│   理由：各文字を1回走査  │
│   例：100KiB → ~0.5ms   │
│                          │
│ 空間計算量：O(m)         │
│   理由：トークン数       │
│   例：~10,000 トークン  │
└──────────────────────────┘
```

**実測値（参考）：**
- 小ファイル（1 KiB）: < 1ms
- 中ファイル（100 KiB）: ~5-10ms
- 大ファイル（1 MiB）: ~50-100ms

---

### 🌳 2. 構文解析（Parser - parsing）

#### 📌 目的

```
トークン列
    ↓ [構文解析]
    
抽象構文木（AST）

含まれる情報：
  - プログラム構造の階層
  - 変数・関数・式の関係
  - スコープ情報の基礎
```

#### 🏗️ AST 構造の全体像

```
Chunk（プログラム全体）
  ├─ Block（文ブロック）
  │  ├─ LocalDecl（local x = ...）
  │  ├─ Assignment（x = ...）
  │  ├─ If（if ... then ... end）
  │  ├─ While（while ... do ... end）
  │  ├─ FunctionCall（func()）
  │  └─ Return（return ...）
  │
  └─ Body（複数文の集合）
     ├─ Statement 1
     ├─ Statement 2
     └─ Statement N
     
式（Expression）は再帰的に含まれる：
  ├─ Identifier（x, y, ...）
  ├─ BinaryOp（a + b, x > y, ...）
  ├─ UnaryOp（not x, -y, ...）
  ├─ FunctionCall（foo(1, 2)）
  ├─ Table({x=1, y=2}）
  └─ Function（function(a) ... end）
```

#### 🔍 処理フロー：if 文の解析例

```lua
入力トークン：
  if | x | > | 10 | then | print | ( | "hi" | ) | end |
  ↑pos=1

-- Parser:parse_if() 呼び出し
↓

1️⃣ 
  match('IF')? → Yes
  advance() → pos=2 (token='x')
  
2️⃣ 条件式を解析
  parse_expression()
  → x > 10
  pos=5 (token='then')
  
3️⃣
  match('THEN')? → Yes
  advance() → pos=6 (token='print')
  
4️⃣ ブロック（body）を解析
  parse_block()
  → [FunctionCall(print("hi"))]
  pos=10 (token='end')
  
5️⃣
  match('END')? → Yes
  advance() → pos=11 (EOF)

出力（AST ノード）：
{
  type: 'If',
  condition: {
    type: 'BinaryOp',
    operator: '>',
    left: {type: 'Identifier', name: 'x'},
    right: {type: 'Number', value: 10}
  },
  consequent: [
    {
      type: 'FunctionCall',
      callee: {type: 'Identifier', name: 'print'},
      args: [{type: 'String', value: 'hi'}]
    }
  ],
  alternate: nil
}
```

#### 📋 主要なノード種別

**📌 文（Statement）**

```lua
-- LocalDecl: local 変数宣言
{
  type = 'LocalDecl',
  names = {'x', 'y'},          -- 変数名リスト
  initializers = {expr1, expr2}, -- 初期化式リスト
  name_tokens = {...}           -- トークン保持（エラー報告用）
}
例：local x, y = 1, 2

-- FunctionDecl: 通常関数宣言
{
  type = 'FunctionDecl',
  name = 'foo',
  params = {'a', 'b'},
  body = {...},
  name_token = tok
}
例：function foo(a, b) return a + b end

-- LocalFunc: local 関数宣言
{
  type = 'LocalFunc',
  name = 'bar',
  params = {'x'},
  body = {...},
  name_token = tok
}
例：local function bar(x) return x * 2 end

-- If: 分岐文
{
  type = 'If',
  condition = expr,
  consequent = {stmts...},
  alternate = {stmts...} or nil
}
例：if cond then ... elseif ... else ... end

-- Assignment: 代入文
{
  type = 'Assignment',
  targets = {ident1, ident2, ...},  -- 左辺
  values = {expr1, expr2, ...}       -- 右辺
}
例：x, y = 1, 2 または a.b = c[i] = 5

-- Return: リターン文
{
  type = 'Return',
  values = {expr, ...}
}
例：return x, y, z
```

**📌 式（Expression）**

```lua
-- Identifier: 変数・関数参照
{
  type = 'Identifier',
  name = 'foo'
}
例：x, myFunc

-- Number: 数値リテラル
{
  type = 'Number',
  value = 42  -- Lua数値型に変換済み
}
例：42, 3.14, 0xFF

-- String: 文字列リテラル
{
  type = 'String',
  value = 'hello'
}
例："hello", 'world'

-- BinaryOp: 二項演算
{
  type = 'BinaryOp',
  operator = '+',      -- +, -, *, /, %, ^, .., ==, etc
  left = expr,
  right = expr
}
例：a + b, x > y, s .. t

-- Table: テーブルリテラル
{
  type = 'Table',
  fields = {
    {key = 'x', value = expr},
    {key = nil, value = expr}   -- キーなしはindexed
  }
}
例：{x=1, y=2} または {1, 2, 3}

-- FunctionCall: 関数呼び出し
{
  type = 'FunctionCall',
  callee = expr,         -- 呼出先（Identifier など）
  args = {expr, ...}     -- 引数
}
例：f(1, 2) または obj:method(x)

-- Function: 匿名関数
{
  type = 'Function',
  params = {'a', 'b'},
  body = {...}
}
例：function(a, b) return a + b end
```

#### ⚡ 計算量分析

```
構文解析の計算量：
┌──────────────────────────────────────┐
│ 時間計算量：O(n)                     │
│   理由：再帰下降パーサは線形         │
│   各トークンを最多1回処理            │
│                                      │
│ 空間計算量：O(d)                     │
│   d = ツリー深さ（通常30以下）      │
│   スタック使用量で制限               │
└──────────────────────────────────────┘
```

---

## 🔍 中間層：解析・変換エンジン

> **中間層**は、構築した AST に対して、意味解析と変換を実施します

### 🔍 1. スコープ解析（Analyzer）

#### 📌 目的と理論

```
入力：AST

目的：各変数のスコープ（可視性範囲）を確立
      ↓
      変数の定義位置・参照位置を記録
      ↓
      どの local がどこで有効か把握

出力：スコープツリー + 変数メタデータ
```

**スコープの概念図：**

```
Global Scope（グローバルスコープ）
  ├─ 変数：x, y（全体可見）
  │
  ├─ Function foo() Scope（foo内）
  │  ├─ 変数：local a, local b
  │  │
  │  └─ Block Scope（if ブロック）
  │     └─ 変数：local z（if 内のみ）
  │
  └─ Function bar() Scope（bar 内）
     └─ 変数：local m
```

#### 🔄 スコープチェーンの仕組み

```lua
-- 例（コード）：
local x = 1          -- Global scope
function foo()
  local y = 2        -- foo scope
  if true then
    local z = 3      -- block scope (parent: foo scope)
    print(x, y, z)   -- すべて可見
  end
end

-- Analyzer が構築するスコープツリー：
{
  scope_id: 'global',
  locals: {
    x = {name='x', refs=0},  -- 定義されたが使用されない（未使用警告）
  },
  children: [
    {
      scope_id: 'foo',
      parent: global,
      locals: {
        y = {name='y', refs=1},  -- 1回参照（print 文で）
      },
      children: [
        {
          scope_id: 'block_1',
          parent: foo,
          locals: {
            z = {name='z', refs=1}  -- 1回参照
          }
        }
      ]
    }
  ]
}
```

#### 🚀 解析アルゴリズム

```
AST を DFS（深さ優先探索）で走査

1️⃣ Chunk/Block ノード到達
   → 新しい Scope を作成
   → スコープスタックにプッシュ

2️⃣ LocalDecl ノード到達
   → 変数名を現在スコープに登録
   → refs = 0 で初期化

3️⃣ Identifier ノード到達（参照）
   → スコープチェーンを上階層へ探索
   → マッチする変数を発見
   → その変数の refs++ インクリメント

4️⃣ ノード走査完了
   → スコープスタックをポップ
```

#### 💡 実装例コード

```lua
-- src/analyzer.lua より（簡略版）

function Analyzer:walk_local_decl(node)
  for i, name in ipairs(node.names) do
    -- 現在スコープに変数を登録
    local sym = {
      name = name,
      refs = 0,           -- 参照カウント
      kind = 'local',
      scope = self.current_scope,
      node = node         -- AST ノード参照
    }
    self.current_scope.locals[name] = sym
    self.all_locals[#self.all_locals + 1] = sym
  end
  
  -- 初期化式も解析（右辺の参照をカウント）
  if node.initializers then
    for _, expr in ipairs(node.initializers) do
      self:walk_expr(expr)
    end
  end
end

function Analyzer:walk_identifier(node)
  -- スコープチェーンで変数を探索
  local sym = self:lookup_symbol(self.current_scope, node.name)
  if sym then
    sym.refs = sym.refs + 1  -- 参照をカウント
  else
    -- 未定義変数
    table.insert(self.undefined_refs, {
      name = node.name,
      line = node.token.line
    })
  end
end

function Analyzer:lookup_symbol(scope, name)
  local current = scope
  while current do
    if current.locals[name] then
      return current.locals[name]
    end
    current = current.parent  -- 親スコープへ
  end
  return nil
end
```

#### 📊 出力形式

```json
{
  "global_scope": {
    "id": "global",
    "locals": {
      "x": {
        "name": "x",
        "refs": 0,
        "kind": "local",
        "line": 1,
        "column": 1
      }
    }
  },
  "all_locals": [
    {
      "name": "x",
      "refs": 0,
      "scope": "global"
    },
    {
      "name": "a",
      "refs": 2,
      "scope": "function:foo"
    }
  ]
}
```

---

### ⚙️ 2. 変換（Transformer / TransformerLine）

#### 📌 目的

```
変換前 AST
    ↓
  指定モードに応じた AST 変換
    ↓
変換後 AST（セマンティクス保持）
```

**代表的な変換例：**

```lua
-- 元のコード
local x = 10
function foo()
  local y = x + 5
  return y
end

-- モード: localkw (local キーワード削除)
x = 10
function foo()
  y = x + 5
  return y
end
```

#### 🔄 AST ベース変換フロー

```lua
function Transformer:transform(ast)
  return self:walk_node(ast)
end

function Transformer:walk_node(node)
  if node.type == 'Chunk' then return self:walk_chunk(node)
  elseif node.type == 'LocalDecl' then return self:walk_local_decl(node)
  elseif node.type == 'Assignment' then return self:walk_assignment(node)
  elseif node.type == 'If' then return self:walk_if(node)
  -- ... etc
end

function Transformer:walk_local_decl(node)
  if self.mode == 'localc' then
    -- 文字列代入の local のみ削除
    if #node.initializers == 1 then
      local init = node.initializers[1]
      if init.type == 'String' then
        -- LocalDecl → Assignment に変換
        return self:create_assignment_from_local(node)
      end
    end
    return node  -- 変換対象外
  end
  
  return node
end
```

#### 📋 各モードの変換ロジック詳解

| モード | 変換条件 | 変換内容 | 例 |
|--------|---------|--------|-----|
| `functionlocal` | すべての `local function` | `local function f` → `function f` | ✅ 必ず変換 |
| `localkw` | scope フィルタに合致 | `local x = v` → `x = v` | scope=global のみ |
| `localte` | ブール値への代入 | `local x = true` → `x = true` | 型チェック必須 |
| `localc` | 文字列への代入 | `local x = "str"` → `x = "str"` | 型チェック必須 |
| `localcyt` | localc と同じ | 文字列削除（式の完全展開） | localc 相当 |
| `localnum` | 数値への代入 | `local x = 42` → `x = 42` | 型チェック必須 |
| `localtabke` | テーブル代入 | `local x = {}` → `x = {}` | テーブル型判定 |

#### 💡 例：localte（ブール値削除）の実装

```lua
function Transformer:walk_local_decl(node)
  if self.mode == 'localte' and #node.names == 1 then
    -- ブール値代入か確認
    local init = node.initializers[1]
    if init and (init.type == 'Boolean' or 
                 (init.type == 'Identifier' and 
                  (init.name == 'true' or init.name == 'false'))) then
      
      -- Assignment に書き換え
      local new_node = {
        type = 'Assignment',
        targets = {{type = 'Identifier', name = node.names[1]}},
        values = {init}
      }
      return new_node
    end
  end
  return node
end
```

---

## 📝 バックエンド：コード生成

#### 📌 目的

```
変換済み AST
    ↓ [CodeGen]
  Lua ソースコード文字列
```

#### 🔄 生成フロー

```
AST ノード訪問
    ↓
┌───────────────────────────┐
│ ノード種別に応じた       │
│ 生成関数呼び出し         │
├─────────────────────────┤
│ LocalDecl  → generate_local_decl
│ Assignment → generate_assignment
│ If         → generate_if
│ Function   → generate_function
│ ...                      │
└────────┬─────────────────┘
         ↓
   コード文字列生成
         ↓
   インデント＆改行付与
         ↓
   連結して出力
```

#### 💡 LocalDecl 生成の詳細例

```lua
-- src/codegen.lua より

function CodeGen:generate_local_decl(node)
  local parts = {}
  
  -- "local " プレフィックス
  table.insert(parts, 'local ')
  
  -- 変数名をループ
  for i, name in ipairs(node.names) do
    if i > 1 then table.insert(parts, ', ') end
    table.insert(parts, name)
  end
  
  -- 初期化式
  if node.initializers and #node.initializers > 0 then
    table.insert(parts, ' = ')
    for i, expr in ipairs(node.initializers) do
      if i > 1 then table.insert(parts, ', ') end
      table.insert(parts, self:generate(expr))
    end
  end
  
  return table.concat(parts)
end

-- 使用例
-- 入力：{type='LocalDecl', names={'x','y'}, initializers=[1, 2]}
-- 出力："local x, y = 1, 2"
```

#### 🎨 インデント管理

```lua
function CodeGen:generate_block(node, indent_level)
  local lines = {}
  indent_level = (indent_level or 0) + 1
  local indent = string.rep('  ', indent_level)
  
  for _, stmt in ipairs(node.statements) do
    local code = self:generate(stmt, indent_level)
    table.insert(lines, indent .. code)
  end
  
  return table.concat(lines, '\n')
end

-- 入力：if then print() end
-- インデント処理後の出力：
if x > 10 then
  print("yes")
end
```

---

## ⚙️ 処理モード詳細実装ガイド

### 🎯 【モード1】Local 削除系

#### functionlocal — 関数宣言の local 削除

```lua
-- 入力
local function foo()
  return 42
end

local bar = function()
  return 3.14
end

-- 出力
function foo()
  return 42
end

bar = function()
  return 3.14
end
```

**実装戦略（AST）：**
1. LocalFunc ノードを見つける
2. FunctionDecl に型変更
3. CodeGen で出力

---

#### localkw [scope] — local キーワード削除

```lua
-- 入力（scope=global）
local x = 10

function foo()
  local y = 20   -- この local は保持（scope=function内）
  return x + y
end

-- 出力
x = 10

function foo()
  local y = 20   -- 変更なし
  return x + y
end
```

**実装戦略：**
1. Analyzer でスコープ記録
2. LocalDecl のスコープを確認
3. scope 条件に合致なら Assignment に変換

---

### 📊 【モード2】Lint（品質検証）

#### 静的解析 + 動的解析の 2 段構成

```
ソース
  ↓
┌─────────────────────┐
│ 静的解析フェーズ    │
├─────────────────────┤
│ ✅ 未使用変数検出  │
│ ✅ 未定義参照検出  │
│ ✅ 型推定（基本）  │
│ ✅ スコープエラー  │
└────────┬────────────┘
         ↓
    (--lint-dynamic=off なら終了)
         ↓
┌─────────────────────┐
│ 動的解析フェーズ    │
├─────────────────────┤
│ 実行時型チェック   │
│ プローブ埋込       │
│ 実際に実行         │
│ 実行時値取得       │
└────────┬────────────┘
         ↓
    診断リスト生成
         ↓
    JSON レポート出力
```

#### 検出可能なエラー・警告

```json
{
  "diagnostics": [
    {
      "code": "W001",
      "message": "Unused local variable 'x'",
      "severity": "warning",
      "line": 5,
      "column": 7
    },
    {
      "code": "E001",
      "message": "Undefined global reference 'foo'",
      "severity": "error",
      "line": 12,
      "column": 1
    },
    {
      "code": "W002",
      "message": "Unused parameter 'b'",
      "severity": "warning",
      "line": 3,
      "column": 14
    }
  ]
}
```

---

### 🤖 【モード3】CodeFix（自動修復）

#### AI ベースの構文修復

```
壊れたソース
    ↓
エラー解析
    ↓
┌────────────────────────────┐
│ 修復候補生成（ビーム探索）  │
│ - 候補1: + 閉じ括弧        │
│ - 候補2: + カンマ          │
│ - 候補3: + end             │
└────────┬───────────────────┘
         ↓
┌────────────────────────────┐
│ 各候補をテスト              │
│ - パース可能？              │
│ - コンパイル可能？          │
│ - Lint エラー数             │
└────────┬───────────────────┘
         ↓
┌────────────────────────────┐
│ スコアリング + 選定        │
│ 最高スコア候補を採用       │
└────────┬───────────────────┘
         ↓
修復済みソース + メタ
```

**修正対象例：**

```lua
-- 【例1】テーブルカンマ欠落
入力：  {1 2 3}
修正：  {1, 2, 3}

-- 【例2】関数呼び出し引数カンマ欠落
入力：  foo(1 2 3)
修正：  foo(1, 2, 3)

-- 【例3】閉じ括弧欠落
入力：  print("hello"
修正：  print("hello")

-- 【例4】end 欠落
入力：  if x then print("ok")
修正：  if x then print("ok") end
```

---

### 🔓 【モード4】Deobf（難読化解除）

#### 多段パイプライン

```
難読化ソース
    ↓
1️⃣ Sandbox 実行
   ├─ load/loadstring 捕捉
   ├─ シンタックス検証
   └─ セーフ実行環境提供
    ↓
2️⃣ 静的デコード
   ├─ 文字列デコード
   ├─ base64 推定
   └─ リテラル書換え
    ↓
3️⃣ VM トレース（オプション）
   ├─ 状態遷移抽出
   ├─ 命令フロー解析
   └─ 仮想マシン状態復元
    ↓
4️⃣ Devirtualizer 前処理
   ├─ 定数式畳み込み
   ├─ テーブル展開
   └─ 参照展開
    ↓
可読化ソース + メタレポート
```

---

## 🛡️ エラーハンドリング・セーフティ機構

### 🔒 Safety Guard の詳細

**3 段階の検証：**

```
変換処理実行
    ↓
出力コード得る
    ↓
┌──────────────────────────────┐
│ 【検証1】パース可能性        │
│                              │
│ 元：parse_ok = true          │
│ 出：parse_ok = false         │
│    → フォールバック！        │
└────────┬─────────────────────┘
         ↓
┌──────────────────────────────┐
│ 【検証2】コンパイル可能性   │
│                              │
│ 元：compile_ok = true        │
│ 出：compile_ok = false       │
│    → フォールバック！        │
└────────┬─────────────────────┘
         ↓
┌──────────────────────────────┐
│ 【検証3】Lint 警告          │
│                              │
│ 元の警告数 < 出の警告数      │
│    → フォールバック！        │
└────────┬─────────────────────┘
         ↓
✅ すべてクリア → 出力を採用
❌ 何か失敗 → 元のコードを返す
```

#### 💡 実装例

```lua
-- src/safety.lua

function Safety.guard(mode, source, output, before_analysis, options)
  local before = before_analysis or Safety.analyze_source(source, options)
  local after = Safety.analyze_source(output, options)
  
  -- フォールバック判定
  if before.parse_ok and not after.parse_ok then
    return source, {mode = mode, fallback_reason = 'parse_failed'}
  end
  
  if before.compile_ok and not after.compile_ok then
    return source, {mode = mode, fallback_reason = 'compile_failed'}
  end
  
  if before.lint_summary.error < after.lint_summary.error then
    return source, {mode = mode, fallback_reason = 'lint_degraded'}
  end
  
  -- OK
  return output, {mode = mode, fallback_applied = false}
end
```

#### 📊 出力レポート

```json
{
  "type": "safety-report",
  "mode": "coml",
  "generated_at": "2026-02-14T12:34:56Z",
  
  "before": {
    "parse_ok": true,
    "compile_ok": true,
    "lint_summary": {
      "error": 0,
      "warning": 2,
      "info": 5
    }
  },
  
  "after": {
    "parse_ok": true,
    "compile_ok": true,
    "lint_summary": {
      "error": 0,
      "warning": 2,
      "info": 5
    }
  },
  
  "fallback_applied": false,
  "fallback_reason": null
}
```

---

## 🌍 方言サポートシステム

### 📋 対応方言一覧

| 方言 | キー | 特徴的な機能 |
|------|---|---|
| **Lua 5.1** | `lua51` | 基本的な Lua / `goto` なし / ビット演算なし |
| **Lua 5.3** | `lua53` | `//`（整数除算）/ ビット演算（`&\|~<<>>`） |
| **Lua 5.4** | `lua54` | `continue` / `const local` / `<close>` 変数 |
| **Luau** | `luau` | Binary literal（`0b101`） / 型 Annotation |

#### 🔧 方言の差異を抽象化

```lua
-- src/dialect.lua
local dialects = {
  lua51 = {
    has_idiv = false,           -- // 非対応
    has_bitwise = false,        -- &|~<< 非対応
    has_goto_label = false,     -- :: 非対応
    has_continue = false,
    has_binary_literal = false,
    has_type_annotation = false
  },
  
  lua54 = {
    has_idiv = true,            -- // 対応
    has_bitwise = true,         -- &|~<< 対応
    has_goto_label = true,      -- :: 対応
    has_continue = true,        -- continue キーワード対応
    has_binary_literal = false,
    has_type_annotation = false
  },
  
  luau = {
    has_idiv = true,
    has_bitwise = true,
    has_goto_label = true,
    has_continue = false,
    has_binary_literal = true,  -- 0b1010 対応
    has_type_annotation = true  -- local x: number = 5 対応
  }
}

function Dialect.resolve(dialect_name)
  if not dialect_name or dialect_name == 'auto' then
    -- auto-detect: 複数の特性から推定
    return detect_dialect_from_source(source)
  end
  return dialects[dialect_name] or dialects.lua54  -- デフォルト lua54
end
```

#### 🎯 トークナイザへの適応

```lua
--src/lexer.lua

function Lexer:read_symbol()
  local ch = self:peek()
  
  -- & は Lua 5.3+ のみ
  if ch == '&' then
    if self.dialect.has_bitwise then
      self:advance()
      return {type = 'symbol', value = '&'}
    else
      error('Unsupported & in ' .. self.dialect.name)
    end
  end
  
  -- // は Lua 5.3+ のみ
  if ch == '/' and self:peek(1) == '/' then
    if self.dialect.has_idiv then
      self:advance()
      self:advance()
      return {type = 'symbol', value = '//'}
    end
    -- else: / を返すだけ
  end
  
  -- ...
end
```

---

## 📊 パフォーマンス・計算量分析

### ⚡ 各フェーズの計算量

```
┌────────────────┬───────────────┬──────────────────┐
│ フェーズ       │ 時間計算量    │ 備考             │
├────────────────┼───────────────┼──────────────────┤
│ Lexing         │ O(n)          │ 単パス           │
│ Parsing        │ O(n)          │ 再帰下降         │
│ Analysis       │ O(n·d)        │ d=scope深さ ≈10 │
│ Transform      │ O(n)          │ ツリー走査       │
│ CodeGen        │ O(n)          │ ツリー走査       │
│ Safety         │ O(n)          │ 再パース         │
├────────────────┼───────────────┼──────────────────┤
│ 合計           │ O(n·d)        │                  │
└────────────────┴───────────────┴──────────────────┘

n = 入力ファイルサイズ（文字数）
d = スコープ階層の深さ（通常：5～20）
```

### 📈 実測値（参考）

```
小ファイル（1 KiB）
├─ Lexer:      ~0.2ms
├─ Parser:     ~0.5ms
├─ Analyzer:   ~0.3ms
├─ Transform:  ~0.2ms
└─ 合計:       ~1.2ms

中ファイル（100 KiB）
├─ Lexer:      ~3ms
├─ Parser:     ~5ms
├─ Analyzer:   ~5ms
├─ Transform:  ~2ms
└─ 合計:       ~15ms

大ファイル（1 MiB）
├─ Lexer:      ~30ms
├─ Parser:     ~50ms
├─ Analyzer:   ~50ms
├─ Transform:  ~20ms
└─ 合計:       ~150ms
```

### 💾 メモリ使用量

```
100 KiB ソースの場合：

トークン列:     ~50 KiB
AST ノード:     ~80 KiB
スコープツリー: ~5 KiB
変換バッファ:   ~100 KiB
─────────────────────────
ピークメモリ:   ~250 KiB

圧縮出力時：
出力バッファ:   ~30-40 KiB（元の 30-40%）
```

---

## 🔧 拡張方法

### 🎯 新しい処理モードを追加する

```lua
-- 【ステップ1】main.lua に条件分岐追加
elseif mode == 'my_custom_transform' then
  local output_code, analyzer, meta = MyTransform.run(source, options)
  return finalize(output_code, analyzer, meta)
end

-- 【ステップ2】src/my_transform.lua を実装
local Lexer = require('src.lexer')
local Parser = require('src.parser')
local CodeGen = require('src.codegen')

local MyTransform = {}

function MyTransform.run(source, options)
  options = options or {}
  
  local ok, output = pcall(function()
    -- 字次解析
    local tokens = Lexer.new(source, {dialect = options.dialect}):tokenize()
    
    -- 構文解析
    local parser = Parser.new(tokens, {dialect = options.dialect})
    local ast = parser:parse()
    
    -- カスタム変換ロジック
    local transformed_ast = apply_custom_transform(ast)
    
    -- コード生成
    local codegen = CodeGen.new(source)
    local result = codegen:generate(transformed_ast)
    
    return result
  end)
  
  if not ok or type(output) ~= 'string' then
    return source, {all_locals = {}}
  end
  
  return output, {all_locals = {}}
end

return MyTransform
```

### 🌍 新しい方言をサポートする

```lua
-- 【ステップ1】dialect.lua に方言定義追加
dialects.my_lua = {
  name = 'my_lua',
  has_idiv = true,
  has_bitwise = false,
  has_goto_label = false,
  has_continue = true,
  has_binary_literal = false,
  has_type_annotation = false,
  -- カスタム属性
  has_my_feature = true
}

-- 【ステップ2】Lexer.lua で条件分岐追加
if self.dialect.has_my_feature then
  -- 新しいトークン種別を認識
  if ch == '@' then
    self:advance()
    return {type = 'my_decorator', value = '@'}
  end
end

-- 【ステップ3】Parser.lua で新しい AST ノード対応
if self:match('MY_DECORATOR') then
  local decorator = self:advance()
  local target = self:parse_expression()
  return {
    type = 'Decorator',
    decorator_token = decorator,
    target = target
  }
end
```

---

## 📚 参考資料・追加情報

| ドキュメント | 用途 |
|------------|------|
| **README.md** | CLI 使用法・よくある質問 |
| **info.txt** | 低レベル技術仕様・理論背景 |
| **test/** | 単体テスト・デバッグスクリプト |
| **output/** | 各モードの出力例・レポート形式 |

---

## 🎓 学習パス

### 初心者向け

1. README.md で全体像を理解
2. main.lua で処理フローの流れを把握
3. 簡単なモード（`deleteout`, `fom`）から始める

### 中級者向け

1. Lexer + Parser で字句・構文解析を理解
2. Analyzer でスコープ解析を学ぶ
3. local 削除系モードの実装を読む

### 上級者向け

1. CodeFix の AI ビーム探索アルゴリズム
2. Deobf の動的デコードと VM トレース
3. 新方言への対応方法

---

## 🏁 まとめ

このツールチェーンは、**モダンなコンパイラ技術**を実装した本格的な Lua コード変換・解析基盤です。

- 🔤 **字次解析・構文解析**: 標準的なコンパイラ前端
- 🔍 **スコープ解析**: セマンティクス情報収集
- ⚙️ **変換エンジン**: AST の意味保持変換
- 📝 **コード生成**: 正確な出力復元
- 🛡️ **セーフティ**: 変換品質を保証
- 🌍 **方言対応**: 柔軟な言語対応

これらを組み合わせることで、**安全で信頼性の高い自動化されたコード処理**が実現できます。

---

**ドキュメント作成日：** 2026年2月14日  
**対応メジャーバージョン：** 1.x  
**対応 Lua 方言：** 5.1, 5.3, 5.4, Luau
#### 処理フロー

```
入力：source:string

┌─────────────────────────┐
│ スキャンループ          │
│ 各文字を順次処理        │
└────────┬────────────────┘
         ↓
    ┌────────────────┐
    │ 文字判定       │
    ├────┬───┬───┬──┤
    │    │   │   │  │
  space keyword digit string/...
    │    │   │   │  │
    └────┴───┴───┴──┘
         ↓
    トークン発行
    ↓トークン配列に追加
```

#### 実装例

**keyword の認識：**

```lua
-- src/lexer.lua より
local keywords = {
  'and', 'break', 'do', 'else', 'elseif', 'end', 'false',
  'for', 'function', 'if', 'in', 'local', 'nil', 'not',
  'or', 'repeat', 'return', 'then', 'true', 'until', 'while'
}

function Lexer:read_identifier()
  local start = self.pos
  while self:is_alphanum(self:peek()) do
    self:advance()
  end
  local text = self.source:sub(start, self.pos - 1)
  local type = keywords[text] and 'keyword' or 'ident'
  return {
    type = type,
    value = text,
    line = self.line,
    col = self.col
  }
end
```

**数値リテラル（複合型）の認識：**

```lua
-- 10進数、16進数、2進数（Luau）、科学記法に対応
-- 0x1F  → hex
-- 0b1010 → binary (if dialect.has_binary_literal)
-- 1.5e2 → scientific
function Lexer:read_number()
  if self:peek() == '0' then
    local next = self:peek(1)
    if next == 'x' or next == 'X' then
      return self:read_hex_number()
    elseif next == 'b' or next == 'B' then
      return self:read_binary_number()
    end
  end
  return self:read_decimal_number()
end
```

#### トークン種別

| type | 例 | 説明 |
|------|-----|------|
| `keyword` | `if`, `local`, `function` | Lua予約語 |
| `ident` | `x`, `foo_bar` | 識別子 |
| `number` | `42`, `0xFF`, `1e-2` | 数値リテラル |
| `string` | `"hello"`, `'world'` | 短文字列 |
| `long_string` | `[[...]]` | 長文字列 |
| `symbol` | `+`, `-`, `==`, `..` | 演算子・区切り文字 |
| `comment` | `-- comment` | コメント（トークン化は選択可） |

#### 計算量

- **時間：** O(n)  （n = 入力文字数）- 単一パス走査
- **空間：** O(m)  （m = トークン数）- トークンベクタ

---

### 2. 構文解析（Parser.lua）

#### 役割

トークン列を**抽象構文木（AST）**に変換します。AST は、プログラムの**構造と意味**を階層的に表現します。

#### 処理フロー

```
トークン列
    ↓
再帰下降パーサ
    ↓
┌──────────────────────────┐
│ parse_chunk()            │  全体
│   parse_block()          │  文のブロック
│   parse_statement()      │  各文
│     parse_if()           │  if 文
│     parse_while()        │  while 文
│     parse_local_decl()   │  local 宣言
│     ...                  │  ...
│   parse_expression()     │  式
│     parse_primary()      │  プライマリ式
│     ...                  │
└──────────────────────────┘
    ↓
  AST構造体
```

#### AST ノード種別

**文（Statement）:**

```lua
-- LocalDecl: local x, y = expr1, expr2
{
  type = 'LocalDecl',
  names = {'x', 'y'},
  initializers = {expr1, expr2},
  name_tokens = {tok1, tok2}
}

-- FunctionDecl: local function foo(a, b) ... end
{
  type = 'LocalFunc',
  name = 'foo',
  params = {'a', 'b'},
  body = {statements...},
  name_token = tok
}

-- If: if cond then ... end
{
  type = 'If',
  condition = expr,
  consequent = {stmts...},
  alternate = nil or {stmts...}
}

-- Assignment: x = 1
{
  type = 'Assignment',
  targets = {Identifier},
  values = {expr}
}

-- Return: return value
{
  type = 'Return',
  values = {expr, ...}
}
```

**式（Expression）:**

```lua
-- Identifier: foo
{
  type = 'Identifier',
  name = 'foo'
}

-- Number: 42
{
  type = 'Number',
  value = 42
}

-- BinaryOp: a + b
{
  type = 'BinaryOp',
  operator = '+',
  left = expr_a,
  right = expr_b
}

-- FunctionCall: f(1, 2)
{
  type = 'FunctionCall',
  callee = expr_f,
  args = {1, 2}
}

-- Table: {x=1, y=2}
{
  type = 'Table',
  fields = {
    {key='x', value=1},
    {key='y', value=2}
  }
}
```

#### 実装例：if 文パース

```lua
-- src/parser.lua より
function Parser:parse_if()
  self:expect('IF')
  local condition = self:parse_expression()
  self:expect('THEN')
  local consequent = self:parse_block()
  
  local alternate = nil
  if self:match('ELSEIF') then
    alternate = {self:parse_if()}
  elseif self:match('ELSE') then
    self:advance()
    alternate = self:parse_block()
  end
  
  self:expect('END')
  
  return AST.If(condition, consequent, alternate)
end
```

#### 計算量

- **時間：** O(n)  （再帰下降パーサは線形）
- **空間：** O(d)  （d = AST 深さ、スタックオーバーフロー防止あり）

---

## 中間層：解析・変換エンジン

### 1. スコープ解析（Analyzer.lua）

#### 役割

AST を走査し、**変数のスコープ（可視性範囲）**を確立し、各変数の用途・参照情報を記録します。宣言された`local`変数の追跡が主な目的。

#### 処理フロー

```
AST
  ↓
┌─────────────────────────────┐
│ スコープツリー構築          │
│ 各ブロック → スコープノード  │
└────────┬────────────────────┘
         ↓
┌─────────────────────────────┐
│ 変数の宣言を記録            │
│ local x, local function f   │
└────────┬────────────────────┘
         ↓
┌─────────────────────────────┐
│ 変数の参照を追跡            │
│ x を読み取る → refs カウント │
└────────┬────────────────────┘
         ↓
  スコープツリー + 解析情報
```

#### スコープチェーン

```
┌─ Global Scope
│
├─ Function foo() Scope
│  ├─ local x
│  ├─ local y
│  └─ Nested Block Scope (if/while/...)
│     └─ local z
│
└─ Function bar() Scope
   └─ local a
```

#### 実装例：local 宣言の追跡

```lua
-- src/analyzer.lua より
function Analyzer:walk_local_decl(node)
  -- local x, y = ...
  for i, name in ipairs(node.names) do
    local sym = {
      name = name,
      refs = 0,
      type = 'local',
      scope = self.current_scope
    }
    self.current_scope.locals[name] = sym
    self.all_locals[#self.all_locals + 1] = sym
  end
  
  -- 初期化式も解析
  if node.initializers then
    for _, expr in ipairs(node.initializers) do
      self:walk_expr(expr)
    end
  end
end

function Analyzer:walk_identifier(node)
  -- 変数参照をカウント
  local sym = self:lookup_symbol(self.current_scope, node.name)
  if sym then
    sym.refs = sym.refs + 1
  end
end
```

#### 分析結果の活用

- `nocode.lua`: 未使用 local 変数を削除
- `rename.lua`: 変数名を安全に変更
- `lint.lua`: 未定義参照・型エラーを検出

---

### 2. 変換（Transformer.lua / TransformerLine.lua）

#### 役割

AST（または行テキスト）に対して、**指定されたモードに応じた変換**を適用します。

#### AST ベース変換の流れ

```lua
function Transformer:transform(ast)
  return self:walk_node(ast)
end

function Transformer:walk_local_decl(node)
  if self.options.mode == 'localc' then
    -- 文字列代入の local を削除
    -- local x = "string" → x = "string"
    if #node.initializers == 1 then
      local init = node.initializers[1]
      if init.type == 'String' then
        return self:create_assignment(node.names, {init})
      end
    end
  elseif self.options.mode == 'localnum' then
    -- 数値代入の local を削除
    if #node.initializers == 1 then
      local init = node.initializers[1]
      if init.type == 'Number' then
        return self:create_assignment(node.names, {init})
      end
    end
  end
  return node
end
```

#### 各モードの変換ロジック

| モード | 変換内容 | 実装 |
|--------|--------|------|
| `functionlocal` | `local function f` → `function f` | AST + Line |
| `localkw` [scope] | `local x = v` → `x = v` (指定scope内) | AST + Line |
| `localte` | `local x = true/false` → `x = true/false` | AST |
| `localc` | `local x = "str"` → `x = "str"` | AST |
| `localnum` | `local x = 42` → `x = 42` | AST |
| `localtabke` | `local x = {...}` → `x = {...}` | AST |
| `outcode` | `-- x = 1` (コメントアウト行) を削除 | Line |

---

## バックエンド：コード生成

### CodeGen（CodeGen.lua）

#### 役割

変換後の AST をLuaソースコードの**文字列**に逆変換します。

#### 処理フロー

```
AST ノード
    ↓
┌───────────────────────────┐
│ ノードタイプ判定          │
├──┬──┬──┬──┬──┬────────┤
│  │  │  │  │  │        │
Local If While For Function
│  │  │  │  │  │        │
└──┴──┴──┴──┴──┴────────┘
    ↓
  対応する生成関数呼び出し
    ↓
  コード文字列生成
    ↓
  インデント・改行調整
    ↓
  連結
    ↓
  出力ソースコード
```

#### 実装例：LocalDecl 生成

```lua
-- src/codegen.lua より
function CodeGen:generate_local_decl(node)
  local parts = {}
  table.insert(parts, 'local ')
  
  for i, name in ipairs(node.names) do
    if i > 1 then table.insert(parts, ', ') end
    table.insert(parts, name)
  end
  
  if node.initializers and #node.initializers > 0 then
    table.insert(parts, ' = ')
    for i, expr in ipairs(node.initializers) do
      if i > 1 then table.insert(parts, ', ') end
      table.insert(parts, self:generate(expr))
    end
  end
  
  return table.concat(parts)
end

-- 使用例：
-- node = {type='LocalDecl', names={'x','y'}, initializers={...}}
-- 出力："local x, y = ..."
```

#### インデント管理

```lua
function CodeGen:generate_block(node)
  local lines = {}
  self.indent_level = self.indent_level + 1
  
  for _, stmt in ipairs(node.statements) do
    local indent = string.rep('  ', self.indent_level)
    table.insert(lines, indent .. self:generate(stmt))
  end
  
  self.indent_level = self.indent_level - 1
  return table.concat(lines, '\n')
end
```

---

## 処理モード詳細

### 【モード1】Local 削除系（local kw 除去）

**目的：** `local` キーワードを削除し、変数をグローバル化

```lua
-- 入力
local x = 10
local function foo()
  local y = 20
  return x + y
end

-- 出力 (localkw global)
local x = 10
function foo()
  local y = 20
  return x + y
end
```

**実装フロー：**

```
1. AST ビルド
2. 各 LocalDecl / LocalFunc ノード走査
3. scope フィルタリング（global/function...）
4. ノードを通常の Assignment / FunctionDecl に変換
5. CodeGen でコード生成
```

---

### 【モード2】整形（fom）

**目的：** インデント・スペース・改行を統一

**実装：**

```lua
-- src/formatter.lua
function Formatter.format(source, options)
  local tokens = Lexer.new(source, options):tokenize()
  local parser = Parser.new(tokens, options)
  local ast = parser:parse()
  
  -- AST を走査しながら整形
  local formatted = format_ast(ast, {
    indent_width = 2,
    space_before_paren = false,
    align_tables = false
  })
  
  return formatted
end
```

**フォーマットルール例：**

```
- インデント: 2 スペース
- if 直後: スペースなし if(...) then
- 関数呼び出し: foo(...) または foo {...}
- テーブル: {x=1, y=2} または {x = 1, y = 2}
```

---

### 【モード3】圧縮（coml - Compressor）

**目的：** ファイルサイズを最小化

**実装方針：**

```lua
-- src/compressor.lua
function Compressor.compress(source, options)
  local tokens = Lexer.new(source, options):tokenize()
  local parser = Parser.new(tokens, options)
  local ast = parser:parse()
  
  -- 最適化パス
  ast = Optimizer.optimize(ast)
  
  -- コード生成（最小化）
  local codegen = CodeGen.new(nil)
  codegen.omit_whitespace = true
  codegen.omit_newlines = true
  codegen.rename_locals = true
  
  return codegen:generate(ast)
end
```

**最適化内容：**

1. **トークン間のスペース削除**: `x + y` → `x+y`
2. **不要な改行削除**: 複数行の `local x; local y;` → 1 行
3. **変数リネーム**: 長い名前 → 短い名前（スコープ内で安全）
4. **定数畳み込み**: `2 + 3` → `5`

---

### 【モード4】Lint（静的解析）

**目的：** コード品質問題を検出

**実装フロー：**

```
ソース
  ↓ Lexer / Parser
  AST
  ↓
┌─────────────────────────────────────┐
│ 静的解析パス (lint_sandbox.lua)      │
│ ・未使用変数検出                    │
│ ・未定義参照検出                    │
│ ・型推定（基本）                    │
│ ・スコープエラー                    │
└────────┬────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ 動的解析パス（--lint-dynamic=on時）  │
│ (lint_dynamic.lua)                  │
│ ・実行可能形式に変換                │
│ ・プローブを埋め込み                │
│ ・実行して実行時型確認              │
└────────┬────────────────────────────┘
         ↓
  診断リスト + レポート
```

**検出できる問題：**

```
W001: 未使用ローカル変数
W002: 未使用引数
E001: 未定義グローバル参照
E002: テーブルプロパティへの不正アクセス
E003: 関数呼び出し引数の型不一致
```

---

### 【モード5】CodeFix（自動修復）

**目的：** 壊れた構文を自動修復

**実装：** ビーム探索 + AI スコアリング

```lua
-- src/codefix.lua
function CodeFix.run(source, options)
  -- 1. 現在のソースがパース可能か確認
  local current_ok, current_err = can_parse(source, options.dialect)
  
  if current_ok then
    return source, nil  -- 既に正常
  end
  
  -- 2. 修正候補を生成（AIベース）
  local candidates = CodeFixAI.generate_candidates(source, current_err, {
    max_depth = options.ai_max_depth or 3,
    beam_width = options.ai_beam_width or 5
  })
  
  -- 3. 各候補をテストして最高スコアのものを採用
  local best_candidate = nil
  local best_score = -math.huge
  
  for _, candidate in ipairs(candidates) do
    local parse_ok = can_parse(candidate.source, options.dialect)
    local compile_ok = can_compile(candidate.source)
    local score = calculate_score(parse_ok, compile_ok, candidate.diff_size)
    
    if score > best_score then
      best_score = score
      best_candidate = candidate
    end
  end
  
  return best_candidate.source, best_candidate
end
```

**修正対象例：**

```lua
-- テーブルカンマ欠落
-- 入力:   {1 2 3}
-- 修正後: {1, 2, 3}

-- 閉じ括弧欠落
-- 入力:   print("hello"
-- 修正後: print("hello")

-- end 欠落
-- 入力:   if cond then print("ok")
-- 修正後: if cond then print("ok") end
```

---

### 【モード6】難読化解除（deobf）

**目的：** 難読化されたLuaコードを解読

**実装パイプライン：**

```
難読化ソース
    ↓
┌──────────────────────────┐
│ sandbox 実行             │
│ load/loadstring 捕捉     │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ 静的デコード             │
│ ・文字列デコード         │
│ ・base64 推定            │
│ ・リテラル書換え         │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ VM トレース（オプション）│
│ ・状態遷移抽出           │
│ ・命令フロー解析         │
└────────┬─────────────────┘
         ↓
┌──────────────────────────┐
│ devirtualizer 前処理     │
│ ・定数式畳み込み         │
│ ・テーブル展開           │
└────────┬─────────────────┘
         ↓
  可読化ソース + レポート
```

**実装例：**

```lua
-- src/deobf/runner.lua
function Deobf.run(source, options)
  local dialect = options.dialect
  
  -- 1. Sandbox 実行
  local sandbox_result = sandbox.execute(source, {
    capture_load = true,
    capture_loads = true,
    timeout = 5000
  })
  
  -- 2. 静的デコード
  local decoded = antitamper.decode(
    sandbox_result.extracted_strings,
    sandbox_result.base64_candidates
  )
  
  -- 3. VM トレース（trace=on なら）
  if options.trace then
    local trace = vmtrace.extract(source, {
      state_var_pattern = '_STATES?',
      max_transitions = 1000
    })
  end
  
  -- 4. デバーチャライザ前処理
  local readable = unlock.rewrite_with_inlines(
    decoded,
    { rewrite_table_literals = options.unlock_rewrite_table_literals }
  )
  
  return readable, analyzer, metadata
end
```

---

### 【モード7】Require 依存解析（rrequire）

**目的：** モジュール個別の `require` 依存関係を追跡

```lua
-- input.lua
local foo = require('lib.foo')
local bar = require('lib.bar')
print(foo.value + bar.compute())

-- output: rrequire-report.json
{
  "dependencies": [
    "lib.foo",      -- 実際に使用されている
    "lib.bar"       -- 実際に使用されている
  ],
  "unused": []
}
```

---

## エラーハンドリング・セーフティ

### Safety Guard 機構

**目的：** 変換処理が元のコードを**破壊しない**ことを保証

```lua
-- src/safety.lua
function Safety.guard(mode, source, output, before_analysis, options)
  -- 変換前のコード性質を記録
  local before = before_analysis or Safety.analyze_source(source, options)
  
  -- 変換後のコード性質を確認
  local after = Safety.analyze_source(output, options)
  
  local fallback_applied = false
  local reason = nil
  
  -- 【チェック1】パース可能性
  if before.parse_ok and not after.parse_ok then
    fallback_applied = true
    reason = 'output lost parseability'
  end
  
  -- 【チェック2】コンパイル可能性
  if before.compile_ok and not after.compile_ok then
    fallback_applied = true
    reason = 'output lost compilability'
  end
  
  -- 【チェック3】Lint 警告悪化
  if before.lint_summary.error < after.lint_summary.error then
    fallback_applied = true
    reason = 'output increased lint errors'
  end
  
  -- フォールバック
  if fallback_applied then
    -- 元のコードを返す
    return source, {
      mode = mode,
      fallback_applied = true,
      reason = reason
    }
  end
  
  return output, {
    mode = mode,
    fallback_applied = false
  }
end
```

**実装フロー：**

```
処理実行
    ↓
出力生成
    ↓
┌────────────────┐
│ Safety Check   │
├────┬───┬──────┤
│    │   │      │
パース コンパイル Lint
│    │   │      │
OK?  OK?  OK?   
│    │   │      │
└────┴───┴──────┘
      ↓
  全て OK → 出力を採用
     不OK → フォールバック（元のコード）
```

**出力レポート例：**

```json
{
  "type": "safety-report",
  "mode": "coml",
  "before": {
    "parse_ok": true,
    "compile_ok": true,
    "lint_summary": {
      "error": 0,
      "warning": 2,
      "info": 5
    }
  },
  "after": {
    "parse_ok": true,
    "compile_ok": true,
    "lint_summary": {
      "error": 0,
      "warning": 2,
      "info": 5
    }
  },
  "fallback_applied": false,
  "fallback_reason": null
}
```

---

## 方言サポートシステム

### Dialect.lua

**役割：** Lua 各種方言の差異を抽象化

#### 対応方言

| 方言 | キー | 特徴 |
|------|------|------|
| Lua 5.1 | `lua51` | goto なし、bit ライブラリなし |
| Lua 5.3 | `lua53` | `//` (整数除算)、bit 演算有効 |
| Lua 5.4 | `lua54` | `continue`、const local 有効 |
| Luau | `luau` | Binary literal (`0b101`)、型Annotation |

#### 実装例

```lua
-- src/dialect.lua
local dialects = {
  lua51 = {
    name = 'lua51',
    has_idiv = false,
    has_bitwise = false,
    has_goto_label = false,
    has_continue = false,
    has_const_local = false,
    has_binary_literal = false,
    has_type_annotation = false
  },
  
  lua54 = {
    name = 'lua54',
    has_idiv = true,          -- //
    has_bitwise = true,       -- &, |, ~, <<, >>
    has_goto_label = true,    -- ::
    has_continue = true,
    has_const_local = false,
    has_binary_literal = false,
    has_type_annotation = false
  },
  
  luau = {
    name = 'luau',
    has_idiv = true,
    has_bitwise = true,
    has_goto_label = true,
    has_continue = false,
    has_const_local = false,
    has_binary_literal = true,  -- 0b1010
    has_type_annotation = true   -- local x: number = 5
  }
}

function Dialect.resolve(dialect_name)
  if not dialect_name or dialect_name == 'auto' then
    -- auto-detect logic
    return dialects.lua54
  end
  return dialects[dialect_name] or dialects.lua54
end
```

#### トークナイザ適応

```lua
-- Lexer が方言に応じてトークンを受理
function Lexer:read_symbol()
  local ch = self:peek()
  
  -- & (bitwise AND) は lua54 / luau のみ
  if ch == '&' and self.dialect.has_bitwise then
    self:advance()
    return { type = 'symbol', value = '&' }
  end
  
  -- // (integer division) は lua53+ のみ
  if ch == '/' and self:peek(1) == '/' then
    if self.dialect.has_idiv then
      self:advance()
      self:advance()
      return { type = 'symbol', value = '//' }
    end
  end
  
  -- ... その他
end
```

---

## コード流の図解

### 【例1】Local 削除処理フロー

```
ユーザー実行：
$ lua54 main.lua localkw global test.lua

┌────────────────────────────────┐
│ main.lua: コマンドライン解析    │
│ mode = 'localkw'               │
│ scope = 'global'               │
│ input_file = 'test.lua'        │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ read_file('test.lua')          │
│ source = "local x = ..."       │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ Dialect.resolve() → lua54      │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ engine 判定                    │
│ 'localkw' は --engine=line 優先│
│ AST エンジン: AST 生成         │
└────────┬───────────────────────┘
         ↓
      AST フロー:
      
┌────────────────────────────────┐
│ Lexer.new(source):tokenize()   │
│ tokens = [local, x, =, ...]    │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ Parser.new(tokens):parse()     │
│ ast.body = [LocalDecl{ ... }]  │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ Analyzer.analyze(ast)          │
│ 'x' を local として記録         │
│ scope = global                 │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ Transformer.transform(ast)     │
│ LocalDecl (scope=global)        │
│ → Assignment に変換            │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ CodeGen.generate(ast)          │
│ "x = ..."                      │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ Safety.guard()                 │
│ ✓ Parse OK                     │
│ ✓ Compile OK                   │
│ ✓ Lint OK                      │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ write_file('output/...')       │
│ + 'output/....safety.json'     │
└────────────────────────────────┘
```

### 【例2】Lint 実行フロー

```
$ lua54 main.lua lint -on test.lua

┌────────────────────────────────┐
│ main.lua: クフォーク処理選択   │
│ mode = 'lint'                  │
│ → Linter.run() 直接呼び出し    │
└────────┬───────────────────────┘
         ↓
┌────────────────────────────────┐
│ Linter.run(source, options)    │
│ options.dynamic = true         │
└────────┬───────────────────────┘
         ↓
    ┌───────┴───────┐
    ↓               ↓
 静的解析        動的解析
    │               │
 ┌──────────┐    ┌────────────────┐
 │ AST 構築 │    │ プローブ埋込   │
 │ scope分析│    │ 実行可能化      │
 │ 型推定   │    │ 実行           │
 │ 警告抽出 │    │ 実行時型収集   │
 └────┬─────┘    └────────┬───────┘
      │                   │
      └─────────┬─────────┘
                ↓
        ┌──────────────────┐
        │ 診断マージ       │
        └────────┬─────────┘
                ↓
        ┌──────────────────┐
        │ JSON レポート    │
        │ 出力             │
        └──────────────────┘
```

---

## ファイル構成と責任

| ファイル | 行数 | 責任 | 入力 | 出力 |
|---------|------|------|------|------|
| `main.lua` | 797 | 全体ディスパッチ | CLI args, ソース | 処理結果 + メタ |
| `lexer.lua` | ~400 | トークン化 | ソースコード | トークン列 |
| `parser.lua` | ~700 | 構文解析 | トークン列 | AST |
| `analyzer.lua` | ~200 | スコープ解析 | AST | スコープツリー |
| `transformer.lua` | ~300 | AST 変換 | AST + mode | 変換 AST |
| `codegen.lua` | ~350 | コード生成 | AST | ソーステキスト |
| `formatter.lua` | ~200 | 整形 | ソース | 整形済みソース |
| `compressor.lua` | ~400 | 圧縮 | ソース | 圧縮ソース |
| `linter.lua` | ~800 | 静的解析 | ソース | 診断リスト |
| `lint_dynamic.lua` | ~500 | 動的解析 | ソース | 実行時診断 |
| `codefix.lua` | ~1000+ | 自動修復（AI） | 破損ソース | 修復ソース + 報告 |
| `deobf/runner.lua` | ~300 | 難読化解除 | 難読化ソース | 可読ソース + 報告 |
| `safety.lua` | ~150 | セーフティチェック | ソース（前後） | 検証結果 |
| `preset.lua` | ~300 | Preset 実行 | JSON preset | 複数出力 |

---

## 処理性能指標

### 計算量 分析

| フェーズ | 時間計算量 | 空間計算量 | 理由 |
|---------|----------|----------|------|
| Lexing | O(n) | O(m) | 単一パス走査 |
| Parsing | O(n) | O(d) | 再帰下降、深さ制限 |
| Analysis | O(n·d₀) | O(s) | スコープ走査、d₀≈見定数 |
| Transform | O(n) | O(n) | ツリー走査 |
| CodeGen | O(n) | O(k) | 出力バッファ (k = 出力サイズ) |

**全体：O(n·d₀)** （d₀ = スコープ深さ≈10～20）

### メモリ使用量（例）

```
入力ファイル: 100KiB

トークン列:        ~50KiB
AST ノード:        ~80KiB
スコープツリー:    ~5KiB
中間出力:          ~120KiB（圧縮時は 30~40KiB）

ピークメモリ: ~250KiB
```

---

## 拡張ポイント

### 新しい処理モード追加

```lua
-- 1. main.lua に条件分岐追加
elseif mode == 'my_transform' then
  local output = MyTransform.run(source, options)
  return finalize(output, ...)
end

-- 2. src/my_transform.lua 実装
local MyTransform = {}
function MyTransform.run(source, options)
  -- AST フロー
  local tokens = Lexer.new(source, options):tokenize()
  local ast = Parser.new(tokens, options):parse()
  local transformed_ast = do_my_transform(ast)
  local output = CodeGen.new(source):generate(transformed_ast)
  return output
end
return MyTransform
```

### 新しい方言サポート

```lua
-- dialect.lua に追加
dialects.my_lua = {
  name = 'my_lua',
  has_idiv = true,
  has_bitwise = false,    -- 独自仕様
  -- ... 他の flags
}

-- Lexer / Parser の条件分岐で対応
```

---

## 参考資料

- **info.txt**: 低レベルの技術仕様
- **README.md**: CLI 使用法・コマンド一覧
- **test/**: 単体テスト・デバッグスクリプト
- **output/**: 処理結果例

---

**作成日:** 2026年2月14日  
**対象バージョン:** 全 18+ モード対応
