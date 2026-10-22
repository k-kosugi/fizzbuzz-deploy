# fizzbuzz-deploy

[fizzbuzz](https://github.com/k-kosugi/fizzbuzz) の配備用リポジトリです。AKS 上の Flux (GitOps) が、このリポジトリの内容をクラスターに反映します。

アプリのコードやイメージのビルドは `fizzbuzz` リポジトリで行い、**何を、どの環境に配備するか** はここで管理します。

## 構成

```text
platform/controllers/      クラスター共通のコントローラー
  cert-manager/            cert-manager (HelmRepository、HelmRelease)
apps/fizzbuzz/             アプリの定義 (環境に依存しない部分)
  backend.yaml             backend の Deployment と Service
  frontend.yaml            frontend の Deployment と Service
  httproute.yaml           外部公開のルーティング (/api -> backend、/ -> frontend、HTTP -> HTTPS 転送)
  networkpolicy.yaml       Gateway の namespace からの通信を許可
clusters/<環境名>/          環境ごとの設定
  kustomization.yaml       配備するイメージの digest など (Flux の apps)
  config/                  Gateway の namespace、Let's Encrypt の発行元、Gateway (Flux の config)
```

Flux は、次の 3 つの Kustomization で、この順に適用します。

| Kustomization | 監視するパス | 内容 |
|---|---|---|
| `controllers` | `platform/controllers` | cert-manager。Ready になるまで待つ |
| `config` | `clusters/<環境名>/config` | 発行元と Gateway。`controllers` の後に適用 |
| `apps` | `clusters/<環境名>` | アプリ本体。他に依存しない |

- namespace `fizzbuzz` は、インフラ (`fizzbuzz` リポジトリの `infrastructure/`) が作成します。ここでは作成しません。
- AKS の Flux 拡張は、HelmRelease をその namespace の `flux-applier` として実行します。権限を持つのは `flux-system` のものだけなので、HelmRelease は `flux-system` に置き、導入先を `targetNamespace` で指定しています。

## 外部公開

`https://fizzbuzz-kkosugi.japaneast.cloudapp.azure.com/` で公開しています。独自ドメインは使わず、Gateway のパブリック IP に付けた Azure の DNS 名を使います。

- Gateway (`clusters/<環境名>/config/gateway.yaml`) の注釈 `service.beta.kubernetes.io/azure-dns-label-name` と、リスナーの `hostname` で名前を決めます。リージョン内で一意にしてください。
- 証明書は、Gateway の注釈 `cert-manager.io/cluster-issuer` で指定した発行元から、cert-manager が取得・更新します。
  - `letsencrypt-prod`: 本番 (現在)
  - `letsencrypt-staging`: 試験用。動作確認に使います (ブラウザーは信頼しません)
- **認証はありません。** 誰でも保存と取得ができます。

## 配備するバージョンの変え方

`clusters/<環境名>/kustomization.yaml` の `images` にある `digest` を、ACR に push 済みのイメージの digest に変更して、`main` にマージします。
digest は、`fizzbuzz` リポジトリの CI の実行結果 (ジョブのサマリー) に表示されます。

- イメージは、タグではなく digest で指定します。staging で検証したものと同じイメージを、そのまま本番に出せます。
- 変更は PR で行い、レビューを経て `main` にマージする運用を想定しています。

## 注意

- 現在 backend はデータをメモリ上に保存します。**replicas は 1 のままにしてください。** 複数にすると、Pod ごとにデータが分かれます。Pod の再起動でデータは消えます。
- namespace にリソースクォータがあるため、Pod には `requests` と `limits` が必須です。
- このリポジトリには、秘密情報を置かないでください。
