---
title: "Claude Code Action v1がGAリリース - GitHub Actionsでさらに強力な自動化を実現"
emoji: "🤖"
type: "tech"
topics: ["github", "claude", "automation", "ai", "devops"]
published: false
---

Anthropic社のClaude Code ActionがついにGA（一般提供）として v1.0 をリリースしました。これまでのベータ版から大幅にアップデートされ、より多くのGitHubイベントに対応し、簡素化されたAPIで手軽に導入できるようになりました。

> **公式リポジトリ**: [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action)
> **公式ドキュメント**: [Migration Guide](https://github.com/anthropics/claude-code-action/blob/main/docs/migration-guide.md) | [Solutions Guide](https://github.com/anthropics/claude-code-action/blob/main/docs/solutions.md)

## 新しくなった Claude Code Action v1 の主な特徴

### 1. より多くのGitHubイベントでのトリガー対応

ベータ版では `@claude` へのメンションに限定されていましたが、v1では以下のような様々なイベントに対応するようになりました。：

- **新規Issue作成**
- **CI/CDの失敗**
- **PR作成・更新**
- **カスタム条件での実行**

### 2. Subagent（サブエージェント）対応

Actionの中で複数のタスクを並行実行できるSubagent機能により、複雑な自動化ワークフローを構築可能になりました。

### 3. カスタマイズ可能なテンプレート

コードレビュー、Issue管理、CI修正などの一般的なワークフロー用に、すぐ使えるテンプレートが用意されています。

**公式サンプル例**: [examples/](https://github.com/anthropics/claude-code-action/tree/main/examples) から以下のワークフローテンプレートが利用可能：
- [CI失敗自動修正](https://github.com/anthropics/claude-code-action/blob/main/examples/ci-failure-auto-fix.yml)
- [Issue重複排除](https://github.com/anthropics/claude-code-action/blob/main/examples/issue-deduplication.yml)
- [Issue自動トリアージ](https://github.com/anthropics/claude-code-action/blob/main/examples/issue-triage.yml)
- [包括的PRレビュー](https://github.com/anthropics/claude-code-action/blob/main/examples/pr-review-comprehensive.yml)

## ベータ版からv1への移行（重要な破壊的変更）

既存のベータユーザーは、以下の破壊的変更に対応する必要があります：

### 必須の変更点

```yaml
# ベータ版（旧）
- uses: anthropics/claude-code-action@beta
  with:
    mode: "tag"
    direct_prompt: "Review this PR for security issues"
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    custom_instructions: "Follow our coding standards"
    max_turns: "10"
    model: "claude-3-5-sonnet-20241022"

# v1.0（新）
- uses: anthropics/claude-code-action@v1
  with:
    prompt: "Review this PR for security issues"
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    claude_args: |
      --system-prompt "Follow our coding standards"
      --max-turns 10
      --model claude-sonnet-4-20250514
```

### 変更点

| 旧（ベータ版） | 新（v1.0） | 説明 |
|---------------|-----------|------|
| `mode` | （削除） | 自動検知されるように |
| `direct_prompt` | `prompt` | 名前変更 |
| `custom_instructions` | `claude_args: --system-prompt` | CLI引数として指定 |
| `max_turns` | `claude_args: --max-turns` | CLI引数として指定 |
| `model` | `claude_args: --model` | CLI引数として指定 |

## 実践的な活用例

### 1. 基本的なワークフロー（@claudeメンションに対応）

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
jobs:
  claude:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          # @claude mentions in comments に自動応答
```

使用例：

```text
@claude このPRのセキュリティ面をレビューしてください
@claude バグの原因を特定して修正方法を提案して
@claude パフォーマンス改善点を分析してください
```

### 2. スラッシュコマンドを使ったコードレビュー

```yaml
name: Automated Code Review
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "/review"
          claude_args: "--max-turns 5"
```

### 3. CI失敗時の自動修正

新しいブランチを作成するのではなく、現在のブランチに直接修正をコミットする実践的なアプローチです：

```yaml
name: CI Failure Auto Fix
on:
  check_run:
    types: [completed]
jobs:
  auto-fix:
    runs-on: ubuntu-latest
    if: github.event.check_run.conclusion == 'failure'
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            CI FAILURE DETECTED - IMMEDIATE ACTION REQUIRED!

            You are an automated CI fixer. Your job is to:
            1. ANALYZE the error logs
            2. FIX the failing code
            3. COMMIT and PUSH the changes
            4. NOTIFY the PR

            ## Failure Details:
            - Failed CI: ${{ fromJSON(steps.failure_details.outputs.result).runUrl }}
            - Workflow: ${{ fromJSON(steps.failure_details.outputs.result).workflowName }}
            - Failed Job: ${{ join(fromJSON(steps.failure_details.outputs.result).failedJobs, ', ') }}
            - PR: #${{ fromJSON(steps.check_label.outputs.result).pr_number }}
            - Branch: ${{ github.event.workflow_run.head_branch }}

            ## Error Logs:
            ${{ toJSON(fromJSON(steps.failure_details.outputs.result).errorLogs) }}

            ## MANDATORY EXECUTION STEPS:

            STEP 1: Read error logs and identify the problem files
            STEP 2: Use Edit/MultiEdit to fix lint errors, type errors, syntax errors
            STEP 3: Run these git commands (REQUIRED):

            git status
            git add -A
            git commit -m "🔧 Auto-fix CI failures

            Fixed issues:
            - Workflow: ${{ fromJSON(steps.failure_details.outputs.result).workflowName }}
            - Failed Jobs: ${{ join(fromJSON(steps.failure_details.outputs.result).failedJobs, ', ') }}

            🤖 Generated with [Claude Code](https://claude.ai/code)"
            git push origin ${{ github.event.workflow_run.head_branch }}

            STEP 4: Add comment to PR #${{ fromJSON(steps.check_label.outputs.result).pr_number }} with:
            gh pr comment ${{ fromJSON(steps.check_label.outputs.result).pr_number }} --body "✅ Auto-fixed CI failures. Changes have been committed and pushed."

            DO NOT just analyze - you MUST fix the code and commit the changes.
            START IMMEDIATELY!
          claude_args: "--max-turns 10"
```

このアプローチの利点：
- **即座の問題解決**: 新しいPRを作らずに元のブランチで直接修正
- **レビューフローの継続**: 同じPR内で議論とレビューが続行可能
- **シンプルな運用**: 追加のブランチ管理が不要

### 4. 定期的なレポート生成

```yaml
name: Daily Progress Report
on:
  schedule:
    - cron: "0 9 * * *"
jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "Generate a summary of yesterday's commits and open issues"
          claude_args: "--model claude-opus-4-1-20250805"
```

### 5. Issue トリアージの自動化

```yaml
name: Issue Triage
on:
  issues:
    types: [opened]
jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Analyze this issue and:
            1. Add appropriate labels
            2. Assign to the right team member
            3. Check for duplicates
          claude_args: "--max-turns 3"
```

## CLAUDE.mdファイルでプロジェクト固有の設定

リポジトリルートに `CLAUDE.md` ファイルを作成することで、Claudeにプロジェクト固有のルールを教えることができます：

```markdown
# プロジェクト設定

## コードレビューの観点
- セキュリティ脆弱性の確認
- パフォーマンス最適化の提案
- TypeScript型定義の適切性

## 使用技術スタック
- React 18 + TypeScript
- Tailwind CSS
- Prisma ORM
- Next.js App Router

## コーディング規約
- ESLint + Prettier使用
- 関数はarrow functionを優先
- 型定義は厳密に
```

## セキュリティのベストプラクティス

### API キーの管理

```yaml
# 🚫 NG: ハードコード
anthropic_api_key: "sk-ant-api03-..."

# ✅ OK: GitHub Secrets 使用
anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### 権限の最小化

```yaml
jobs:
  claude:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      issues: write
      pull-requests: write
    steps: # ...
```

## コストの最適化

Claude Code Action使用時のコスト要因：

### GitHub Actions コスト
- GitHub-hosted runnerの利用分数
- プライベートリポジトリでは無料枠を超えると課金

### Claude API コスト
- プロンプトと応答の長さに応じたトークン消費
- 複雑なタスクやコードベースが大きいほど消費量増加

### 最適化のコツ

```yaml
# 無駄な API 呼び出しを避ける
- uses: anthropics/claude-code-action@v1
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    prompt: "Specific task instructions here"
    claude_args: |
      --max-turns 5
      --model claude-sonnet-4-20250514
  timeout-minutes: 10  # タイムアウト設定でランナウェイ防止
```

## まとめ

Claude Code Action v1のGAリリースにより、GitHub上での開発ワークフローがさらに強力になりました。ベータ版からの移行には破壊的変更が含まれますが、新機能と改善されたAPIにより、より柔軟で効率的な自動化が可能になります。

主な改善点：

- **多様なGitHubイベントへの対応**
- **Subagent機能による複雑なワークフロー**
- **簡素化されたAPI設計**
- **豊富なテンプレート例**

セキュリティ、コスト最適化、CLAUDE.mdによるプロジェクト設定を適切に行うことで、チームの開発効率を大幅に向上させることができるでしょう。

## 参考リンク

- [Claude Code Action 公式リポジトリ](https://github.com/anthropics/claude-code-action)
- [移行ガイド](https://github.com/anthropics/claude-code-action/blob/main/docs/migration-guide.md)
- [ソリューションガイド](https://github.com/anthropics/claude-code-action/blob/main/docs/solutions.md)
- [公式サンプル例](https://github.com/anthropics/claude-code-action/tree/main/examples)
- [GitHub Actions Marketplace](https://github.com/marketplace/actions/claude-code-action)
