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
  # 本デモは OCI VCN-Native Pod Networking 前提のため true
  ociVcnIpNative: true
  # API サーバーのプライベート IP
  apiserverEndpoint: "<apiserver-private-ip>"

# 困ったときは info → debug にすると原因を追いやすい
logLevel: info
```

> **⚠️ 重要：`ociVcnIpNative` は CNI と必ず一致させること**
> - **VCN-Native** クラスタ … `true`（本デモ）
> - **Flannel** クラスタ … `false`
>
> ここが実際の CNI と食い違うと、ノードは起動・クラスタ参加まで進むものの
> **CNI が初期化されず NotReady のまま**になります（[99](99-troubleshooting.md) 参照）。
> [01](01-prerequisites.md) の `kubectl get ds -n kube-system` で必ず確認してください。

## 3. インストール

名前空間 `karpenter` に導入します（IAM ポリシーの `namespace`／`service_account` と一致させること）。

```bash
helm install karpenter karpenter-provider-oci/karpenter \
  --version <chart-version> \
  --values values.yaml \
  --namespace karpenter \
  --create-namespace
```

インストール後に `values.yaml` を変更した場合は、`upgrade` で反映します。

```bash
helm upgrade karpenter karpenter-provider-oci/karpenter \
  --values values.yaml \
  --namespace karpenter
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
