# coder-dev-platform

[English](./README.md)

[Coder](https://coder.com/) をベースとした Kubernetes 上の開発者プラットフォームです。PostgreSQL には [CloudNativePG](https://cloudnative-pg.io/)、セキュアなネットワークアクセスには [Tailscale](https://tailscale.com/) を使用し、全コンポーネントを [Helmfile](https://helmfile.readthedocs.io/) で管理しています。

## アーキテクチャ

| コンポーネント | チャート | 役割 |
|---------------|---------|------|
| Tailscale Operator | `tailscale/tailscale-operator` | Tailscale によるセキュアなネットワークアクセスと Ingress |
| CloudNativePG Operator | `cnpg/cloudnative-pg` | Kubernetes 上の PostgreSQL オペレーター |
| PostgreSQL Cluster | `cnpg/cluster` | Coder 用データベース（CloudNativePG で管理） |
| Coder | `coder-v2/coder` | クラウド開発プラットフォーム |

### デプロイ順序

Helmfile の `needs` により正しい順序でデプロイされます:

```
tailscale-operator  (独立)
cnpg operator  -->  coder-db (PostgreSQL)  -->  coder
```

## 前提条件

- [mise](https://mise.jdx.dev/) -- CLI ツールのバージョン管理（helm, helmfile, kubectl）
- Kubernetes クラスタへのアクセス（`kubectl` のコンテキスト設定済み）
- Tailscale の OAuth クライアント資格情報（`client_id` と `client_secret`）

## セットアップ

### 1. CLI ツールのインストール

```bash
mise install
```

`mise.toml` に定義されたバージョンがインストールされます:

| ツール | バージョン |
|--------|-----------|
| helm | 4.1.1 |
| helmfile | 1.3.0 |
| kubectl | 1.35.1 |

### 2. Tailscale の namespace と OAuth Secret を作成

```bash
kubectl create ns tailscale

kubectl create secret generic operator-oauth \
  --from-literal=client_id=<YOUR_CLIENT_ID> \
  --from-literal=client_secret=<YOUR_CLIENT_SECRET> \
  -n tailscale
```

### 3. 全コンポーネントをデプロイ

```bash
helmfile apply
```

Tailscale operator、CloudNativePG operator、PostgreSQL クラスタ、Coder の順にデプロイされます。

### その他のコマンド

```bash
# 適用前に差分を確認
helmfile diff

# テンプレートをローカルでレンダリング（ドライラン）
helmfile template

# チャートの values を検証
helmfile lint

# 全リリースを削除
helmfile destroy
```

## 設定

values ファイルは `values/` ディレクトリに整理されています:

| ファイル | コンポーネント | 内容 |
|---------|---------------|------|
| `values/tailscale.yaml` | Tailscale Operator | OAuth 設定、オペレータータグ、プロキシ設定、API サーバープロキシ |
| `values/cnpg-operator.yaml` | CloudNativePG Operator | オペレーターのレプリカ数、モニタリング |
| `values/cnpg-cluster.yaml` | PostgreSQL Cluster | インスタンス数、ストレージ、PostgreSQL バージョン、データベース/ユーザー |
| `values/coder.yaml` | Coder | DB 接続、アクセス URL、Ingress（Tailscale）、OAuth |

### 主要な設定項目

#### Coder (`values/coder.yaml`)

| 設定 | 説明 |
|------|------|
| `coder.env` `CODER_PG_CONNECTION_URL` | CloudNativePG の Secret（`coder-db-app`）から自動取得 |
| `coder.env` `CODER_ACCESS_URL` | ワークスペースが Coder に接続するための URL（`https://coder`） |
| `coder.ingress.className` | Tailscale Ingress を使用するため `tailscale` を指定 |

#### PostgreSQL (`values/cnpg-cluster.yaml`)

| 設定 | 説明 |
|------|------|
| `cluster.instances` | PostgreSQL のレプリカ数（デフォルト: `1`） |
| `cluster.storage.size` | 永続ボリュームのサイズ（デフォルト: `10Gi`） |
| `databases.coder.owner` | データベースの所有ユーザー（デフォルト: `coder`） |

#### Tailscale (`values/tailscale.yaml`)

| 設定 | 説明 |
|------|------|
| `oauth.clientSecret` | OAuth 資格情報を含む Kubernetes Secret の名前 |
| `operatorConfig.defaultTags` | オペレーターに割り当てる ACL タグ（デフォルト: `tag:k8s-operator`） |
| `proxyConfig.defaultTags` | プロキシに割り当てる ACL タグ（デフォルト: `tag:k8s`） |
| `apiServerProxyConfig.mode` | API サーバープロキシのモード（`true`, `false`, `noauth`） |

## Coder へのアクセス

Coder は Tailscale Ingress 経由で公開されます。デプロイ後:

1. Tailnet で [HTTPS 証明書を有効化](https://tailscale.com/docs/how-to/set-up-https-certificates)する
2. `https://coder`（Tailscale MagicDNS 経由）で Coder にアクセス
3. Web UI から最初の管理者ユーザーを作成

## 参考リンク

- [Coder ドキュメント -- Kubernetes インストール](https://coder.com/docs/install/kubernetes)
- [CloudNativePG ドキュメント](https://cloudnative-pg.io/documentation/)
- [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator)
- [HTTPS 証明書のセットアップ](https://tailscale.com/docs/how-to/set-up-https-certificates)
- [Helmfile ドキュメント](https://helmfile.readthedocs.io/)
- [mise ドキュメント](https://mise.jdx.dev/)
