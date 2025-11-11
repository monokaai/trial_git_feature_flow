# Git Feature Flow 検証結果

## 概要

このドキュメントは、feature ブランチから dev および main ブランチへの Squash merge の動作検証結果をまとめたものです。

**重要な前提**: このGit Feature Flowでは、`dev`、`stg`、`main`ブランチはそれぞれ独立しており、**相互にマージされることはありません**。各環境ブランチは、featureブランチからのみSquash mergeを受け付けます。

## リポジトリ情報

- **リポジトリURL**: <https://github.com/monokaai/trial_git_feature_flow>
- **検証日時**: 2025-11-11
- **マージ方式**: Squash merge のみ使用

## ブランチ構成

```text
main (本番環境用)
├── be4c560 - feat: 初期コミット - README追加
└── a7e1e87 - feat: テスト機能の追加（main用） (#2) [Squash merge]

dev (開発環境用)
├── be4c560 - feat: 初期コミット - README追加
└── 97cae5b - feat: テスト機能の追加 (#1) [Squash merge]

feature/test-flow (機能開発用)
├── e087543 - feat: 機能1のファイルを追加
├── 949a644 - feat: 機能2のファイルを追加
└── 0175c74 - fix: 機能3のバグ修正とパフォーマンス改善
```

**ブランチの独立性**:

- `dev`、`stg`、`main`はそれぞれ独立した環境ブランチです
- これらのブランチ間での直接マージは**禁止**されています
- すべての変更はfeatureブランチから各環境ブランチへのSquash mergeのみで行います

## 実行したPR操作

### PR #1: feature/test-flow → dev

- **タイトル**: feat: テスト機能の追加
- **変更内容**:
  - 機能1: 機能A/Bの実装
  - 機能2: 機能C/Dの実装とテスト追加
  - 機能3: バグ修正とパフォーマンス改善
- **コミット数**: 3個のコミット
- **マージ方式**: Squash merge
- **マージコミット**: 97cae5b
- **結果**: ✅ 成功

### PR #2: feature/test-flow → main

- **タイトル**: feat: テスト機能の追加（main用）
- **変更内容**: PR #1と同じ
- **コミット数**: 3個のコミット（同じコミット）
- **マージ方式**: Squash merge
- **マージコミット**: a7e1e87
- **結果**: ✅ 成功

## 検証結果

### 1. Squash merge の動作

✅ **期待通りの動作**:

- feature ブランチの3つのコミットが、各ブランチで1つのコミットにまとめられた
- dev ブランチ: 1つのSquashコミット (97cae5b)
- main ブランチ: 1つのSquashコミット (a7e1e87)

### 2. コミット履歴の見え方

✅ **dev ブランチの履歴**:

```text
Commits on Nov 11, 2025
├── feat: テスト機能の追加 (#1) [Verified] - 97cae5b
└── feat: 初期コミット - README追加 - be4c560
```

✅ **main ブランチの履歴**:

```text
Commits on Nov 11, 2025
├── feat: テスト機能の追加（main用） (#2) [Verified] - a7e1e87
└── feat: 初期コミット - README追加 - be4c560
```

### 3. PR番号の表示

✅ **PR番号が正しく表示**:

- dev ブランチのコミットには `(#1)` が付与
- main ブランチのコミットには `(#2)` が付与
- GitHub上で該当PRへのリンクとして機能

### 4. 重複の扱い

✅ **コミットハッシュは異なる**:

- devブランチ: 97cae5b
- mainブランチ: a7e1e87
- 同じ変更内容でも、異なるSquashコミットとして作成された

✅ **設計上の意図**:

- 同じfeatureブランチから複数の環境ブランチ（dev、stg、main）にマージした場合、それぞれ独立したSquashコミットが生成される
- これは意図的な設計であり、各環境ブランチは独立して管理される
- dev、stg、main間での直接マージは行わないため、この独立性は維持される

## スクリーンショット一覧

検証の各ステップのスクリーンショットは `.playwright-mcp/screenshots/` に保存されています:

1. `01-pr-feature-to-dev-created.png` - PR #1 作成直後
2. `02-pr-feature-to-dev-commits.png` - PR #1 のコミット一覧
3. `03-pr-feature-to-dev-files-changed.png` - PR #1 の変更ファイル
4. `04-pr-feature-to-dev-merged.png` - PR #1 マージ完了
5. `05-dev-branch-history-after-squash.png` - devブランチのコミット履歴
6. `06-pr-feature-to-main-created.png` - PR #2 作成直後
7. `07-pr-feature-to-main-commits.png` - PR #2 のコミット一覧
8. `08-pr-feature-to-main-merged.png` - PR #2 マージ完了
9. `09-main-branch-history-after-squash.png` - mainブランチのコミット履歴
10. `10-all-prs-merged.png` - 全PRのマージ完了状態

## 結論

### ✅ Squash mergeのメリット（確認できた点）

1. **履歴が整理される**: 複数のコミットが1つにまとめられ、見やすい履歴になる
2. **PR番号が付与**: コミットメッセージにPR番号が自動的に付与され、トレーサビリティが向上
3. **Verified バッジ**: GitHubによって検証済みマークが付く

### ⚠️ 注意すべき点

1. **独立したコミット**: 同じ変更でも、異なる環境ブランチにマージすると別のコミットハッシュが生成される
2. **環境ブランチの独立性**: dev、stg、mainはそれぞれ独立した履歴を持つため、同じfeatureブランチからマージしても異なるコミットハッシュになる（これは正常な動作）
3. **元のコミット履歴**: featureブランチの詳細なコミット履歴は、Squashコミットには含まれない（PRから確認可能）

### 推奨事項

- ✅ Squash mergeは履歴を整理するのに有効
- ✅ featureブランチは各PR後に削除することを推奨
- ✅ 環境ブランチ（dev、stg、main）間での直接マージは禁止（保護ルールで防止推奨）
- ✅ すべての変更はfeatureブランチから各環境ブランチへのSquash mergeで行う

## 検証完了

すべてのPRが正常にSquash mergeされ、期待通りの動作を確認しました。
