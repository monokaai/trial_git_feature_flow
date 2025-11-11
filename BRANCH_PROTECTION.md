# ブランチ保護 - 環境ブランチ間のマージ防止

## 概要

このドキュメントでは、`dev`、`stg`、`main`ブランチ間での直接マージやpushを防ぐための手法を説明します。

## 保護対象ブランチ

- `dev` (開発環境)
- `stg` (ステージング環境)
- `main` (本番環境)

これらのブランチは相互に独立しており、直接マージされることはありません。

## 保護手法

### 1. GitHub Rulesets（推奨・必須）

GitHub Rulesetsを使用して、環境ブランチへの直接pushと、環境ブランチ間のPRマージを防止します。Rulesetsはブランチ保護ルールよりも柔軟で、より細かい制御が可能です。

#### インポート方法

事前に定義されたJSONファイルを使用して、簡単にRulesetをインポートできます。

**JSONファイルの場所**: `github-rulesets/environment-branch-protection.json`

##### GitHub UIからインポート

1. リポジトリの `Settings` → `Rules` → `Rulesets` に移動
2. `Import ruleset` ボタンをクリック
3. `github-rulesets/environment-branch-protection.json` を選択
4. 設定を確認して `Create ruleset` をクリック

##### GitHub CLIで直接インポート

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

#### 設定内容

JSONファイルには以下の設定が含まれています:

- **対象ブランチ**: `dev`、`stg`、`main`（正規表現パターン: `^dev$`, `^stg$`, `^main$`）
- **Pull Request Rules**:
  - ✅ PR必須（non_fast_forward）
  - ✅ 承認必須: 1人以上
  - ✅ レビュースレッドの解決必須
  - ✅ ステールレビューの自動却下
- **Merge Queue**:
  - ✅ Squash mergeのみ許可
- **Deletion Rules**:
  - ✅ ブランチ削除を防止
- **Linear History**:
  - ✅ 線形履歴を要求
- **Update Rules**:
  - ✅ 直接pushを防止

#### Rulesetsの利点

- 複数のルールを1つのRulesetにまとめられる
- Organizationレベルで管理可能
- APIやTerraformで管理可能
- より柔軟な条件設定が可能

#### 制限事項

GitHub Rulesetsだけでは、環境ブランチ間のPR作成を直接防ぐことはできません。そのため、以下のGitHub Actionsによるチェックと組み合わせることを推奨します。

---

### 2. GitHub Actions による PR チェック（推奨・必須）

GitHub Actionsを使用して、環境ブランチ間のPRを自動的に検出し、マージをブロックします。

#### 実装例

`.github/workflows/prevent-env-branch-merge.yml`:

```yaml
name: Prevent Environment Branch Merges

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  check-branch-merge:
    runs-on: ubuntu-latest
    steps:
      - name: Check for environment branch merges
        run: |
          BASE_BRANCH="${{ github.base_ref }}"
          HEAD_BRANCH="${{ github.head_ref }}"
          
          ENV_BRANCHES=("dev" "stg" "main")
          
          # 環境ブランチ間のマージをチェック
          if [[ " ${ENV_BRANCHES[@]} " =~ " ${BASE_BRANCH} " ]] && \
             [[ " ${ENV_BRANCHES[@]} " =~ " ${HEAD_BRANCH} " ]] && \
             [[ "$BASE_BRANCH" != "$HEAD_BRANCH" ]]; then
            echo "❌ ERROR: Environment branches (dev, stg, main) cannot be merged into each other."
            echo "   Base branch: $BASE_BRANCH"
            echo "   Head branch: $HEAD_BRANCH"
            echo "   All changes must come from feature branches via Squash merge."
            exit 1
          fi
          
          # 環境ブランチへの直接pushをチェック（featureブランチからのみ許可）
          if [[ " ${ENV_BRANCHES[@]} " =~ " ${BASE_BRANCH} " ]]; then
            if [[ ! "$HEAD_BRANCH" =~ ^(feature|fix|hotfix)/ ]]; then
              echo "❌ ERROR: Only feature/fix/hotfix branches can be merged into environment branches."
              echo "   Base branch: $BASE_BRANCH"
              echo "   Head branch: $HEAD_BRANCH"
              exit 1
            fi
          fi
          
          echo "✅ Branch merge check passed"
```

#### 使用方法

1. `.github/workflows/` ディレクトリを作成（存在しない場合）
2. 上記のワークフローファイルを配置
3. PRが作成されると自動的にチェックが実行される
4. 環境ブランチ間のPRの場合、チェックが失敗しマージがブロックされる

---

### 3. Lefthook（クライアント側・推奨）

開発者のローカル環境で、環境ブランチへの直接pushを防ぐためのGit hookを管理します。Lefthookを使用することで、Git hooksを設定ファイル（`.lefthook.yml`）で管理でき、チーム全体で一貫した設定を維持できます。

#### 実装例

`.lefthook.yml`:

```yaml
# Lefthook configuration
# https://github.com/evilmartians/lefthook
#
# 環境ブランチへの直接pushを防ぐためのpre-push hook

pre-push:
  commands:
    prevent-env-branch-push:
      run: |
        # 環境ブランチの定義
        ENV_BRANCHES=("dev" "stg" "main")
        
        # push先のブランチ名を取得
        while read local_ref local_sha remote_ref remote_sha
        do
          # リモートブランチ名から環境ブランチ名を抽出
          BRANCH_NAME=$(echo "$remote_ref" | sed 's|refs/heads/||')
          
          # 環境ブランチへの直接pushをチェック
          for env_branch in "${ENV_BRANCHES[@]}"; do
            if [ "$BRANCH_NAME" = "$env_branch" ]; then
              echo "❌ ERROR: Direct push to environment branch '$env_branch' is not allowed."
              echo "   Please create a Pull Request from a feature branch instead."
              exit 1
            fi
          done
        done
```

#### 設定手順

1. **lefthook をインストール**

   ```bash
   # macOS
   brew install lefthook

   # Linux
   curl -1sLf 'https://dl.cloudsmith.io/public/evilmartians/lefthook/setup.sh' | bash

   # または npm/pnpm
   npm install -g lefthook
   # または
   pnpm add -g lefthook
   ```

2. **セットアップスクリプトを実行**

   ```bash
   ./scripts/setup-git-hooks.sh
   ```

   または手動で:

   ```bash
   lefthook install
   ```

3. **動作確認**

   ```bash
   lefthook version
   ```

#### Lefthookの利点

- ✅ 設定ファイル（`.lefthook.yml`）でGit hooksを管理
- ✅ バージョン管理システムで設定を共有可能
- ✅ チーム全体で一貫した設定を維持
- ✅ 複数のhooksを簡単に管理可能
- ✅ 並列実行や条件分岐などの高度な機能に対応

#### 注意点

- このhookはクライアント側で動作するため、強制されない可能性がある
- GitHub Actionsと組み合わせて使用することを推奨
- 新しくクローンしたリポジトリでは、`lefthook install` を実行する必要がある

---

### 4. サーバーサイドフック（参考）

Gitサーバー（GitHub以外のGitサーバーを使用する場合）で、サーバーサイドフックを設定することで、環境ブランチへの直接pushを完全に防止できます。

#### 実装例（GitLab、Gitea等の場合）

`hooks/update`:

```bash
#!/bin/bash

REFNAME="$1"
OLDREV="$2"
NEWREV="$3"

ENV_BRANCHES=("dev" "stg" "main")
BRANCH_NAME=$(echo "$REFNAME" | sed 's|refs/heads/||')

for env_branch in "${ENV_BRANCHES[@]}"; do
    if [ "$BRANCH_NAME" = "$env_branch" ]; then
        echo "❌ ERROR: Direct push to environment branch '$env_branch' is not allowed."
        echo "   Please create a Merge Request from a feature branch instead."
        exit 1
    fi
done

exit 0
```

**注意**: GitHubではサーバーサイドフックを直接設定することはできません。GitHub Actionsを使用してください。

---

## 推奨される組み合わせ

### 最小構成（必須）

1. ✅ **GitHub Rulesets**: 環境ブランチへの直接pushを防止、PR必須、承認必須、Squash mergeのみ許可
2. ✅ **GitHub Actions チェック**: 環境ブランチ間のPRを検出してブロック

### 推奨構成

上記に加えて：

1. ✅ **Pre-push Git Hook**: 開発者のローカル環境で早期にエラーを検出

---

## 設定チェックリスト

### GitHub Rulesets

- [ ] `github-rulesets/environment-branch-protection.json` を確認
- [ ] GitHub UIまたはGitHub CLIでRulesetをインポート
- [ ] GitHub UIでRulesetが正しく作成されていることを確認
- [ ] 必要に応じて設定をカスタマイズ

### GitHub Actions

- [ ] `.github/workflows/prevent-env-branch-merge.yml` を作成
- [ ] ワークフローが正常に動作することを確認
- [ ] テストPRを作成して動作を検証

### Lefthook（オプション）

- [ ] `.lefthook.yml` を作成
- [ ] `scripts/setup-git-hooks.sh` を作成
- [ ] チームメンバーにlefthookのインストールとセットアップ手順を共有
- [ ] READMEにセットアップ手順を記載

---

## テスト方法

### 1. 環境ブランチ間のPR作成テスト

1. `dev` ブランチから `main` ブランチへのPRを作成
2. GitHub Actionsがエラーを検出することを確認
3. PRがマージできないことを確認

### 2. 直接pushテスト

1. ローカルで `dev` ブランチに切り替え
2. 変更をコミット
3. `git push origin dev` を実行
4. GitHubがpushを拒否することを確認（保護ルールにより）

### 3. 正常なPRテスト

1. featureブランチから `dev` ブランチへのPRを作成
2. チェックがパスすることを確認
3. Squash mergeが正常に動作することを確認

---

## トラブルシューティング

### Q: GitHub Actionsのチェックが実行されない

A: ワークフローファイルのパスとトリガー設定を確認してください。`.github/workflows/` ディレクトリに正しく配置されているか確認してください。

### Q: Rulesetが適用されない

A: ターゲットブランチの設定が正しいか確認してください。また、リポジトリまたはOrganizationの管理者権限が必要です。複数のRulesetが存在する場合、優先順位を確認してください。

### Q: Lefthookのhookが動作しない

A: 以下を確認してください:

- lefthookがインストールされているか: `lefthook version`
- hooksがインストールされているか: `lefthook install`
- `.lefthook.yml` が正しく配置されているか
- 設定ファイルの構文が正しいか: `lefthook run pre-push --dry-run`

---

## まとめ

環境ブランチ間のマージを防ぐには、**GitHub Rulesets**と**GitHub Actions**の組み合わせが最も効果的です。これにより、誤ったマージを自動的に検出し、ブロックできます。

### Rulesetsの管理

Rulesetsは以下の方法で管理できます：

- **GitHub UI**: リポジトリまたはOrganizationの設定から
- **GitHub API**: REST APIまたはGraphQL APIを使用
- **Terraform**: `github_branch_protection_v3` リソースを使用（Terraform Provider for GitHub）
- **GitHub CLI**: `gh api` コマンドを使用

### 参考リンク

- [GitHub Rulesets Documentation](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [GitHub API - Rulesets](https://docs.github.com/en/rest/repos/rules)
