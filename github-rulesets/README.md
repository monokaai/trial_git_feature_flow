# GitHub Rulesets

このディレクトリには、環境ブランチ保護用のGitHub Rulesets設定ファイルが含まれています。

## ファイル

- `environment-branch-protection.json`: 環境ブランチ（dev、stg、main）保護用のRuleset設定

## 使用方法

### GitHub UIからインポート

1. GitHubのリポジトリ設定に移動: `Settings` → `Rules` → `Rulesets`
2. `Import ruleset` ボタンをクリック
3. `environment-branch-protection.json` を選択
4. 設定を確認して `Create ruleset` をクリック

### GitHub CLIで直接インポート

```bash
# リポジトリレベル
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  /repos/<owner>/<repo>/rulesets \
  --input github-rulesets/environment-branch-protection.json

# Organizationレベル
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  /orgs/<org>/rulesets \
  --input github-rulesets/environment-branch-protection.json
```

## 設定内容

このRulesetは以下の保護を提供します:

- ✅ 環境ブランチ（dev、stg、main）への直接pushを防止
- ✅ Pull Request必須
- ✅ 承認必須（1人以上）
- ✅ レビュースレッドの解決必須
- ✅ Squash mergeのみ許可
- ✅ ブランチ削除を防止
- ✅ 線形履歴を要求

詳細については、[BRANCH_PROTECTION.md](../BRANCH_PROTECTION.md) を参照してください。
