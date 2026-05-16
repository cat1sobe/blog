---
title: "社内独自LLMでもClaude Codeみたいなエージェント開発をしたい — Continue + AGENTS.md という解"
emoji: "🤖"
type: "tech"
topics: ["continue", "ai", "vscode", "llm", "agents"]
published: true
---

# 外部AIを使えない企業で、エージェンティックな開発体験が欲しい

ここ最近、Claude CodeやCursorのような「AIが自律的にツールを呼び出し、ファイルを編集し、コマンドを実行する」エージェンティックな開発体験が一気に普及しました。一度この体験に慣れてしまうと、もう昔のチャット型AIには戻れません。

一方で、企業の現場ではこんな状況も多いはずです。

- セキュリティ・コンプライアンス上の理由で **外部のAIサービス（ChatGPT / Claude / Cursor等）が使えない**
- ただし社内には **OpenAI互換のAPIを持つ独自LLMプロキシ** がある
- GitHub Copilotは導入できているが、**チャット中心で「自律的にツールを呼ぶ」用途には物足りない**
- せめて社内環境の中で、Claude Codeに近い体験を再現したい

この記事は、そんな環境で **Continue（VS Code拡張のOSS AIアシスタント）** を活用し、社内独自LLMでもエージェンティックな開発環境を構築するまでの実践メモです。あわせて、Continueには無い「Skills」機能を `AGENTS.md` で疑似的に再現したノウハウも紹介します。

## Continueとは — 何が解決できるのか

[Continue](https://www.continue.dev/) は、VS Code / JetBrains 向けのオープンソースAIアシスタント拡張です。Claude Codeに近いことができ、かつ **接続先のLLMを自由に選べる** のが最大の強みです。

冒頭の課題に対しては、以下の点が刺さります。

| 課題 | Continueでの解決 |
|---|---|
| 外部AIが使えない | **任意のOpenAI互換エンドポイントに接続可**。社内LLMプロキシをそのまま指定できる |
| エージェンティックに動かしたい | **Agentモード** でツールを自律実行（ファイル編集・コマンド実行・MCPサーバー呼び出し） |
| プロジェクト固有のルールを守らせたい | **Rules機能** でシステムメッセージを常時注入 |
| 定型タスクを楽にしたい | **Prompts機能**（スラッシュコマンド）でテンプレ呼び出し |

要するに「Claude Codeの体験を、自分が選んだLLMで実現できる」のがContinueです。

## Continueの主要機能ざっくり整理

詳細は[公式ドキュメント](https://docs.continue.dev/)に譲り、ここでは「これだけ押さえれば動かせる」レベルで整理します。

| 機能 | 役割 |
|---|---|
| **Models** | 使用するLLMの定義。1つのConfigに複数モデルを登録し、用途（chat / autocomplete / edit / embed）ごとに使い分けられる |
| **Rules** | Agent / Chat / Edit のすべてのリクエストのシステムメッセージに追記される固定ルール。言語・スタイル・振る舞いを常時適用したいときに使う |
| **Prompts** | `/コマンド名` で呼び出せるカスタムプロンプトテンプレート |
| **Tools (MCP)** | Agentモードでモデルが外部ツールを呼び出す機能。MCP（Model Context Protocol）サーバーに対応 |
| **Configs** | 上記をまとめて定義する `config.yaml`。プロジェクトごと / グローバルで切り替え可 |

Configファイルは2種類の場所に置けます。

| 種類 | 場所 | 用途 |
|---|---|---|
| グローバル（Local Config） | `~/.continue/config.yaml` | 全プロジェクト共通のデフォルト |
| プロジェクト固有 | `.continue/agents/<名前>.yaml` | プロジェクトごとの設定。複数登録可、Configsドロップダウンから切替 |

シークレット（APIキー等）は `${{ secrets.KEY_NAME }}` 形式で参照し、実値は `.continue/.env` に記載します（`.gitignore` 必須）。

ここから先は、実際に **「社内LLMプロキシに接続する」** → **「疑似Skillsでエージェントを賢くする」** という流れで実践していきます。

## 実践①：社内LLM（OpenAI互換API）への接続

社内に `https://internal-llm.example.com/openai/v1` のようなOpenAI互換エンドポイントがある前提で、Continueから接続する設定を作ります。

### ステップ1：`.continue/agents/local.yaml` を作成する

プロジェクトルートに `.continue/agents/local.yaml` を作成し、以下のように記述します。

```yaml
%YAML 1.1
---
model_defaults: &model_defaults
  provider: openai
  apiBase: https://internal-llm.example.com/openai/v1
  requestOptions:
    headers:
      api-key: ${{ secrets.INTERNAL_LLM_API_KEY }}

name: local
version: 1.0.0
schema: v1

models:
  - name: internal-gpt-pro
    model: internal-gpt-pro
    roles: [chat, edit]
    <<: *model_defaults
  - name: internal-gpt-mini
    model: internal-gpt-mini
    roles: [chat, autocomplete]
    <<: *model_defaults
  - name: internal-gpt-codex
    model: internal-gpt-codex
    roles: [autocomplete]
    <<: *model_defaults
```

ポイントは2つ。

**① `model_defaults` （YAMLアンカー）で共通設定をDRYに**
複数モデルで同じ `apiBase` ・認証ヘッダーを使う場合、YAML 1.1 のアンカー機能で共通化できます。`<<: *model_defaults` で各モデルに展開されます。ファイル先頭の `%YAML 1.1` 宣言はアンカー機能を使う場合に必須なので忘れずに。

**② `roles` で用途を分ける**
`chat` はチャット・エージェント、`autocomplete` はタブ補完、`edit` はインラインコード編集、`embed` はコードベース検索（埋め込み）に使われます。モデルの得意分野に応じて割り振りましょう。

### ステップ2：`.continue/.env` にAPIキーを登録

`.continue/.env` を作成し、APIキーを記載します。

```
INTERNAL_LLM_API_KEY=（実際のAPIキー）
```

このファイルは必ず `.gitignore` に追加してください。

### ステップ3：Configsドロップダウンから選択

`.continue/agents/local.yaml` を保存すると、Continue画面上部のConfigsドロップダウンに `local` が自動で表示されます。選択すれば接続完了。

> 反映されない場合はConfigsの **Reload** を実行してください。

これでチャット・Agentモードのいずれも社内LLMで動くようになります。

## 実践②：「Skills」が無いContinueに、AGENTS.mdで疑似Skillsを実装する

接続できたら次は「賢く動かす」フェーズです。ここからが本記事の本題です。

### 背景：コンテキスト最適化が重要な時代

LLMの精度向上・トークン消費削減のため、近年は **コンテキストの最適化** が重要視されています。Claude CodeのSkillsやhooksのように「**必要なタイミングで必要なコンテキストだけをロードする**」仕組みが各AIツールに続々と追加されています。

中でもSkillsは、

- タスクに関係のない情報を常時コンテキストに含めない
- 特定の操作手順・APIルール・セキュリティ制約を一箇所に集約できる
- 新しい操作を追加するときは `SKILL.md` を1枚作るだけで完結する

という点で、運用面でも非常に優秀な仕組みです。

### しかしContinueにはSkillsが無い

ここで残念なお知らせがあります。**Continueには現時点でSkillsに相当する機能がありません**。

- Prompts（`/コマンド`）は **明示的に呼び出さないと発火しない**
- 「タスク内容に応じて自動で読み込む」という動作はできない

そこで考えたのが、`AGENTS.md` を使った疑似Skillsです。

### 鍵：ContinueはAGENTS.mdを自動でRuleとして読み込む

公式ドキュメントには明示的な記載が（執筆時点で）見当たらないのですが、実際の動作として **プロジェクトルートの `AGENTS.md` がContinueによって自動的にRuleとして読み込まれる** ことが確認できています。`AGENTS.md` / `agents.md` をデフォルトのルールファイルとして自動検出している模様です。

つまり、**`AGENTS.md` に「このタスクのときは対応するSKILL.mdを読んでから進めよ」というルールを書いておけば、AI側が自律的にSKILL.mdを参照しに行く**、という挙動を作れます。

### 構成例

スキルは `.github/skills/` 配下にフォルダごとに配置します（場所はプロジェクトの慣習で `.continue/skills/` でもOK）。

```
.github/skills/
  wiki-get/        SKILL.md
  wiki-update/     SKILL.md
  ticket-search/   SKILL.md
  monitoring/      SKILL.md
```

各 `SKILL.md` はフロントマター + 必要情報の構成。

```markdown
---
name: wiki-get
description: 社内Wikiのページ取得・子ページ一覧取得が必要なときに使う。
---

# 社内Wiki ページ取得スキル

## 接続情報
- エンドポイント: https://wiki.example.com/_api/v3
- 認証: Bearerトークン（環境変数 WIKI_API_TOKEN）

## 事前準備
- トークンの有効期限を確認

## セキュリティルール
- 取得したページ本文を外部に転送しない
- ログには本文を出力せず、ページIDのみ記録

## エンドポイント
- ページ取得: GET /pages/{id}
- 子ページ一覧: GET /pages/{id}/children

## サンプル（curl）
\`\`\`bash
curl -H "Authorization: Bearer $WIKI_API_TOKEN" \
  https://wiki.example.com/_api/v3/pages/123
\`\`\`
```

### AGENTS.md への定義

そして `AGENTS.md` 側に、スキル一覧と利用タイミングを表形式で定義します。

```markdown
# スキル（Skills）ルール — Continue 専用

> **このセクションは Continue 拡張機能専用のルールです。
> 他のAIツール（GitHub Copilot等）には適用されません。**

## 概要

以下のスキルが `.github/skills/` 配下に定義されている。
**タスクの内容がスキルの利用タイミングに該当する場合、
必ず対応する `SKILL.md` を読み込んでから処理を進めること。**

## スキル一覧と利用タイミング

| スキル名 | 利用タイミング | SKILL.md パス |
|---|---|---|
| `wiki-get` | 社内Wikiのページ取得・子ページ一覧取得が必要なとき。「Wikiのページを見て」「ページ内容を確認して」などの文脈 | `.github/skills/wiki-get/SKILL.md` |
| `wiki-update` | 社内Wikiのページを更新・書き込みするとき。「Wikiのページを更新して」「ページに書き込んで」などの文脈 | `.github/skills/wiki-update/SKILL.md` |
| `ticket-search` | 社内チケット管理システムのチケット調査・検索が必要なとき。「チケットを調べて」「対応中の起票を見て」などの文脈 | `.github/skills/ticket-search/SKILL.md` |
| `monitoring` | 監視システムのアラート確認・トリガー検索・グラフ取得が必要なとき。「アラートを確認して」「監視状況を見て」などの文脈 | `.github/skills/monitoring/SKILL.md` |

## スキル利用手順

1. ユーザーのリクエストが上記「利用タイミング」に該当するか判断する
2. 該当するスキルの `SKILL.md` を読み込む
3. `SKILL.md` に記載された手順・ルール・APIに従って処理を実行する
```

### 実際の動作

これを設定した状態で、たとえばチャットで「Wikiの〇〇というページを確認して」と依頼すると、AI側が `AGENTS.md` のルールに従って自律的に `.github/skills/wiki-get/SKILL.md` を読み込み、そこに書かれた認証方法・エンドポイント・セキュリティルールに従って処理を進めてくれます。

複雑な分岐がある動作だと精度はやや落ちますが、シンプルなタスクであれば概ね意図通りに機能しています。Claude CodeのSkillsには劣るものの、「無いよりは断然マシ」というレベルにはなりました。

### `AGENTS.md` を編集したらReloadを忘れずに

`AGENTS.md` を修正した場合は、ContinueのConfigsから **Reload** を実行しないと反映されません。ハマりポイントなので注意。

## まとめ

| やったこと | 効果 |
|---|---|
| Continueを`.continue/agents/local.yaml`で社内LLMに接続 | 外部AIが使えない環境でもVS Code上でエージェンティックな開発体験 |
| `AGENTS.md` + `.github/skills/*/SKILL.md` で疑似Skillsを実装 | Claude Code Skills相当の「自動コンテキストロード」を再現 |

「外部AIが使えない／でも社内LLMはある／Claude Codeみたいな体験は諦めたくない」という人にとって、Continue + AGENTS.md の組み合わせは現時点で最も現実的な解だと感じています。

なお、社内プロキシ環境でContinueを動かそうとすると `self signed certificate in certificate chain` エラーで詰まることがあります。この対処は別記事にまとめる予定です。
