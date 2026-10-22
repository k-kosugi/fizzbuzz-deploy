# fizzbuzz-deploy

[fizzbuzz](https://github.com/k-kosugi/fizzbuzz) の配備用リポジトリです。AKS 上の Flux (GitOps) が、このリポジトリの内容をクラスターに反映します。

アプリのコードやイメージのビルドは `fizzbuzz` リポジトリで行い、**何を、どの環境に配備するか** はここで管理します。

## 構成

```text
apps/fizzbuzz/            アプリの定義 (環境に依存しない部分)
  backend.yaml            backend の Deployment と Service
  frontend.yaml           frontend の Deployment と Service
clusters/<環境名>/         環境ごとの設定。Flux はこのディレクトリを監視する
  kustomization.yaml      配備するイメージの digest など
```

- Flux は、環境名のディレクトリ (例: `clusters/staging`) を監視します。
- namespace `fizzbuzz` は、インフラ (`fizzbuzz` リポジトリの `infrastructure/`) が作成します。ここでは作成しません。

## 配備するバージョンの変え方

`clusters/<環境名>/kustomization.yaml` の `images` にある `digest` を、ACR に push 済みのイメージの digest に変更して、`main` にマージします。
digest は、`fizzbuzz` リポジトリの CI の実行結果 (ジョブのサマリー) に表示されます。

- イメージは、タグではなく digest で指定します。staging で検証したものと同じイメージを、そのまま本番に出せます。
- 変更は PR で行い、レビューを経て `main` にマージする運用を想定しています。

## 注意

- 現在 backend はデータをメモリ上に保存します。**replicas は 1 のままにしてください。** 複数にすると、Pod ごとにデータが分かれます。Pod の再起動でデータは消えます。
- namespace にリソースクォータがあるため、Pod には `requests` と `limits` が必須です。
- Ingress は含まれません。外部からの公開が必要なときは、Ingress コントローラーと合わせて追加します。
- このリポジトリには、秘密情報を置かないでください。
