# 02. Helm で KPO をインストール

KPO（Karpenter Provider for OCI）を Helm でインストールします。

## 1. Helm リポジトリを追加

```bash
helm repo add karpenter-provider-oci https://oracle.github.io/karpenter-provider-oci/charts
helm repo update karpenter-provider-oci
```

利用可能なバージョンを確認します（[01](01-prerequisites.md) の対応表に合うものを選びます）。

```bash
helm search repo karpenter-provider-oci/karpenter --versions
```

## 2. values ファイルを作成

`values.yaml` を作成し、[01](01-prerequisites.md) で控えた値を埋めます。

```yaml
# values.yaml
settings:
  clusterCompartmentId: "<cluster-compartment-id>"
  vcnCompartmentId: "<vcn-compartment-id>"
  # 本デモは Flannel（overlay）前提のため false
  ociVcnIpNative: false
  # API サーバーのプライベート IP
  apiserverEndpoint: "<apiserver-private-ip>"

# 困ったときは info → debug にすると原因を追いやすい
logLevel: info
```

> **Note**
> `ociVcnIpNative: true` は OCI VCN-Native Pod Networking を使う場合の設定です。
> 本デモ（Flannel）では `false` のままにします。

## 3. インストール

名前空間 `karpenter` に導入します（IAM ポリシーの `namespace`／`service_account` と一致させること）。

```bash
helm install karpenter karpenter-provider-oci/karpenter \
  --version <chart-version> \
  --values values.yaml \
  --namespace karpenter \
  --create-namespace
```

## 4. 起動確認

karpenter Pod が `Running` になれば成功です。

```bash
kubectl get pods -n karpenter
```

出力イメージ：

```
NAME                         READY   STATUS    RESTARTS   AGE
karpenter-5f8c9c8b7d-abcde   1/1     Running   0          40s
```

ログも確認しておきます（`ERROR` が出ていないこと）。

```bash
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=50
```

> **Running にならない / CrashLoopBackOff の場合**
> - IAM ポリシー（[01](01-prerequisites.md)）の namespace / service_account / cluster_id が一致しているか
> - `apiserverEndpoint` が正しいプライベート IP か
> を確認してください。詳しくは [99-troubleshooting.md](99-troubleshooting.md)。

## 次へ

→ [03. OCINodeClass / NodePool を作成](03-nodepool.md)
