# Git Feature Flow 検証リポジトリ

このリポジトリは、Git feature flowの動作を検証するために作成されました。

## 検証内容

- featureブランチからdevブランチへのSquash merge
- featureブランチからmainブランチへのSquash merge
- マージ後の履歴の見え方の確認

## ブランチ構成

- `main`: 本番環境用のメインブランチ
- `dev`: 開発環境用のブランチ
- `feature/*`: 機能開発用のブランチ
