# GitHub Workflows ドキュメント

このドキュメントでは、`.github/workflows`ディレクトリ内のワークフロー設定について説明します。

## 概要

このリポジトリには、Claude AIを活用した2つのGitHub Actionsワークフローが設定されています。

## ワークフロー一覧

### 1. Claude Code (`claude.yml`)

#### 目的
IssueやPull Requestのコメントで`@claude`とメンションすることで、Claudeに作業を依頼できるワークフローです。

#### トリガー条件
以下のイベントで自動的に実行されます：
- `issue_comment.created` - Issueコメントが作成されたとき
- `pull_request_review_comment.created` - PRレビューコメントが作成されたとき
- `issues.opened` - Issueが作成されたとき
- `issues.assigned` - Issueが割り当てられたとき
- `pull_request_review.submitted` - PRレビューが送信されたとき

#### 実行条件
以下の条件を満たす場合のみ実行されます：
- コメントまたはIssue本文に`@claude`が含まれている

#### 権限設定
```yaml
permissions:
  contents: read        # リポジトリ内容の読み取り
  pull-requests: read   # Pull Requestの読み取り
  issues: read          # Issueの読み取り
  id-token: write       # IDトークンの書き込み（認証用）
  actions: read         # CI結果の読み取り（PR用）
```

#### 主な設定
- **Runner**: `ubuntu-latest`
- **チェックアウト深度**: 1 (最新のコミットのみ)
- **使用アクション**: `anthropics/claude-code-action@v1`
- **認証**: `CLAUDE_CODE_OAUTH_TOKEN` シークレットを使用

#### 追加権限
- `actions: read` - ClaudeがPRのCI結果を読み取るために必要

#### カスタマイズ可能なオプション
- `prompt`: Claudeに与えるカスタムプロンプト（オプション）
- `claude_args`: Claudeの動作をカスタマイズする引数（オプション）

#### 使用例
```markdown
@claude このバグを修正してください
@claude PRのレビューをお願いします
@claude テストを追加してください
```

---

### 2. Claude Code Review (`claude-code-review.yml`)

#### 目的
Pull Requestが作成または更新されたときに、Claudeが自動的にコードレビューを実行するワークフローです。

#### トリガー条件
以下のPull Requestイベントで自動的に実行されます：
- `pull_request.opened` - PRが作成されたとき
- `pull_request.synchronize` - PRが更新されたとき

#### オプション設定（コメントアウト済み）

##### ファイルフィルター
特定のファイルパスの変更時のみ実行する設定（現在は無効）：
```yaml
# paths:
#   - "src/**/*.ts"
#   - "src/**/*.tsx"
#   - "src/**/*.js"
#   - "src/**/*.jsx"
```

##### PR作成者フィルター
特定のユーザーのPRのみレビューする設定（現在は無効）：
```yaml
# if: |
#   github.event.pull_request.user.login == 'external-contributor' ||
#   github.event.pull_request.user.login == 'new-developer' ||
#   github.event.pull_request.author_association == 'FIRST_TIME_CONTRIBUTOR'
```

#### 権限設定
```yaml
permissions:
  contents: read        # リポジトリ内容の読み取り
  pull-requests: read   # Pull Requestの読み取り
  issues: read          # Issueの読み取り
  id-token: write       # IDトークンの書き込み（認証用）
```

#### 主な設定
- **Runner**: `ubuntu-latest`
- **チェックアウト深度**: 1 (最新のコミットのみ)
- **使用アクション**: `anthropics/claude-code-action@v1`
- **認証**: `CLAUDE_CODE_OAUTH_TOKEN` シークレットを使用

#### レビュー観点
Claudeは以下の観点でコードレビューを実施します：
- コード品質とベストプラクティス
- 潜在的なバグや問題
- パフォーマンスの考慮事項
- セキュリティ上の懸念
- テストカバレッジ

#### 許可されたツール
セキュリティのため、Claudeは以下のコマンドのみ実行可能です：
- `gh issue view:*` - Issue情報の表示
- `gh search:*` - 検索
- `gh issue list:*` - Issue一覧の表示
- `gh pr comment:*` - PRへのコメント投稿
- `gh pr diff:*` - PRの差分表示
- `gh pr view:*` - PR情報の表示
- `gh pr list:*` - PR一覧の表示

#### レビュー結果の投稿
Claudeは`gh pr comment`コマンドを使用してPRにレビューコメントを投稿します。

---

## セットアップ手順

### 必要なシークレット
両方のワークフローを使用するには、以下のシークレットをリポジトリに設定する必要があります：

1. **CLAUDE_CODE_OAUTH_TOKEN**
   - Claude Code APIの認証に使用
   - リポジトリの Settings > Secrets and variables > Actions から設定

### シークレットの設定方法
1. GitHubリポジトリページで `Settings` タブを開く
2. 左サイドバーの `Secrets and variables` > `Actions` を選択
3. `New repository secret` をクリック
4. Name: `CLAUDE_CODE_OAUTH_TOKEN`
5. Secret: 取得したOAuthトークンを入力
6. `Add secret` をクリック

---

## トラブルシューティング

### ワークフローが実行されない場合
1. シークレット `CLAUDE_CODE_OAUTH_TOKEN` が正しく設定されているか確認
2. コメントに `@claude` が含まれているか確認（claude.yml）
3. PRが作成または更新されたか確認（claude-code-review.yml）
4. Actions タブでワークフローの実行ログを確認

### 権限エラーが発生する場合
- リポジトリの Settings > Actions > General で、Workflow permissions が適切に設定されているか確認
- 推奨設定: "Read and write permissions" または必要最小限の権限

---

## 参考リンク

- [Claude Code Action ドキュメント](https://github.com/anthropics/claude-code-action)
- [使用方法](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md)
- [CLIリファレンス](https://code.claude.com/docs/en/cli-reference)

---

## カスタマイズ例

### 特定のファイルのみレビュー対象にする
`claude-code-review.yml` の `paths` フィルターのコメントを解除：
```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - "src/**/*.ts"
      - "src/**/*.tsx"
```

### 新規コントリビューターのPRのみレビュー
`claude-code-review.yml` の `if` 条件のコメントを解除：
```yaml
jobs:
  claude-review:
    if: github.event.pull_request.author_association == 'FIRST_TIME_CONTRIBUTOR'
```

### カスタムプロンプトの設定
`claude.yml` または `claude-code-review.yml` の `prompt` パラメータを設定：
```yaml
with:
  claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
  prompt: 'カスタムの指示をここに記述'
```
