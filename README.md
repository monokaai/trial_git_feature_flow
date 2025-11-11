# Git Feature Flow 検証リポジトリ

このリポジトリは、Git feature flowの動作を検証するために作成されました。

## 検証内容

- featureブランチからdevブランチへのSquash merge
- featureブランチからmainブランチへのSquash merge
- マージ後の履歴の見え方の確認

## ブランチ構成

- `main`: 本番環境用のメインブランチ
- `dev`: 開発環境用のブランチ
- `stg`: ステージング環境用のブランチ（オプション）
- `feature/*`: 機能開発用のブランチ

**重要な原則**: `dev`、`stg`、`main`ブランチはそれぞれ独立しており、**相互にマージされることはありません**。すべての変更はfeatureブランチから各環境ブランチへのSquash mergeのみで行います。

## ブランチ保護

環境ブランチ間のマージを防ぐための保護手法については、[BRANCH_PROTECTION.md](./BRANCH_PROTECTION.md) を参照してください。

- **GitHub Rulesets**: JSONファイルからインポート可能（GitHub UIまたはGitHub CLI）
- **GitHub Actions**: 環境ブランチ間のPRを自動検出・ブロック
- **Lefthook**: Git hooks管理（`lefthook install`で初期設定）

## 検証結果

検証結果の詳細については、[VERIFICATION_RESULTS.md](./VERIFICATION_RESULTS.md) を参照してください。
