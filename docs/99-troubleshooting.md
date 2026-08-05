# 99. トラブルシューティング（よくあるハマりどころ）

デモ当日に慌てないための、代表的な症状と対処です。

## karpenter Pod が Running にならない

```bash
kubectl get pods -n karpenter
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=100
```

- **IAM の不一致**：Helm の `namespace` / `serviceAccount` と、ポリシーの
  `request.principal.namespace` / `service_account` / `cluster_id` が一致しているか。
- **`apiserverEndpoint` 間違い**：API サーバーのプライベート IP を再確認。
  ```bash
  kubectl get endpoints kubernetes -o jsonpath='{.subsets[0].addresses[0].ip}'
  ```

## Pod は Pending のままでノードが増えない

```bash
kubectl describe nodeclaim
kubectl get events --sort-by=.lastTimestamp | tail -n 30
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=100
```

チェック：

- **NodePool の `limits` に到達**：`cpu` / `memory` の上限で頭打ちになっていないか。
- **`requirements` が厳しすぎる**：指定シェイプがそのリージョン/AD で在庫切れ、など。
  一時的にシェイプ候補を増やす。
- **IAM 権限不足**：`manage instance-family` / `manage volume-family` /
  `manage virtual-network-family` が付いているか。
- **サブネット/NSG の OCID 誤り**：`OCINodeClass` の値を再確認。

## ノードは起動したが `NotReady` のまま（★実際に遭遇した事例）

**最も遭遇しやすいトラブル**です。ノードが `kubectl get nodes` に**出てくる**のに
`NotReady` が続く場合、CLUSTER_JOIN は成功しており、原因は **CNI** にあります。

### 症状

```bash
kubectl describe node <ノード名>
```

`Conditions` の `Ready` にこのメッセージが出ます。

```
Ready  False  KubeletNotReady
  container runtime network not ready: NetworkReady=false
  reason:NetworkPluginNotReady
  message:Network plugin returns error: no CNI configuration file in /etc/cni/net.d/.
```

`/etc/cni/net.d/` に設定を置くのは **CNI の DaemonSet Pod** の仕事なので、
「**CNI Pod がこのノードに乗っていない**」ことを意味します。

### 切り分け手順

**① そのノードに何が乗っているか**（`describe node` の `Non-terminated Pods` でも可）

```bash
kubectl get pods -A -o wide | grep <ノード名>
```

`csi-oci-node` / `kube-proxy` / `proxymux-client` はあるのに **CNI Pod が無い**なら確定です。

**② CNI DaemonSet の配置状況を見る（決定的）**

```bash
kubectl get ds -n kube-system
```

`DESIRED` の数を比べます。

```
NAME                 DESIRED   CURRENT   READY
csi-oci-node             4         4       4     ← 全ノードに乗っている
vcn-native-ip-cni        2         2       2     ← Karpenter ノードに乗っていない！
```

`csi-oci-node` が 4 なのに `vcn-native-ip-cni` が 2 → **Karpenter ノード 2 台が CNI の対象外**。

### 原因と対処

| クラスタの CNI | 必要な設定 |
| --- | --- |
| `vcn-native-ip-cni` がある（**VCN-Native**） | Helm values `ociVcnIpNative: true` ＋ `OCINodeClass` に `secondaryVnicConfigs` |
| `kube-flannel-ds` がある（**Flannel**） | Helm values `ociVcnIpNative: false` ＋ `secondaryVnicConfigs` は**不要** |

**VCN-Native なのに設定漏れだった場合**の修正手順：

```bash
# 1. values.yaml を ociVcnIpNative: true に修正して反映
helm upgrade karpenter karpenter-provider-oci/karpenter --values values.yaml --namespace karpenter
```

```bash
# 2. OCINodeClass に secondaryVnicConfigs を追加して適用
kubectl apply -f manifests/ocinodeclass.yaml
```

```bash
# 3. 詰まったノードを作り直させる
kubectl delete nodeclaims --all
```

`vcn-native-ip-cni` の `DESIRED` が既存ノード数から増えれば成功のサインです。

## ノードが OKE に Join しない（`kubectl get nodes` に出てこない）

こちらは上とは別の問題です。ノードが**そもそも一覧に現れない**場合：

- **CLUSTER_JOIN ポリシー漏れ**：動的グループとその
  `{CLUSTER_JOIN}` ポリシー（[01](01-prerequisites.md) の 4-2）を確認。
- **サブネットのセキュリティルール**：ワーカー ⇔ API サーバー間の通信が許可されているか。

## Pod が ImageInspectError / イメージを取得できない

OKE ノードは **cri-o の short-name モード** により、**完全修飾イメージ名** を要求します。

- ❌ `nginx` / `pause` などの短縮名
- ⭕ `registry.k8s.io/pause:3.9`、`docker.io/library/nginx:1.27` のように **レジストリ込み** で指定

本リポジトリの [manifests/inflate.yaml](../manifests/inflate.yaml) は `registry.k8s.io/pause:3.9` を
使っているので、独自マニフェストを持ち込むときは同様に完全修飾名にしてください。

## Consolidation（統合）が起きない

- `consolidateAfter` の時間だけ待つ（本デモは `1m`）。
- `consolidationPolicy` が `WhenEmpty` だと「完全に空」でないと消えません。
  低使用率でも統合したいなら `WhenEmptyOrUnderutilized`。
- 対象 Pod に `karpenter.sh/do-not-disrupt` や PDB が付いていないか。

## 便利な確認コマンド集

```bash
# Karpenter 関連リソースの一覧
kubectl get ocinodeclass,nodepool,nodeclaims

# ノードと所属 NodePool
kubectl get nodes -L karpenter.sh/nodepool

# 直近のイベント
kubectl get events --sort-by=.lastTimestamp | tail -n 30

# Karpenter ログをレベル debug で詳しく見る（values.yaml の logLevel: debug + helm upgrade）
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter -f
```

## 参考リンク

- Karpenter Provider for OCI（GitHub）: https://github.com/oracle/karpenter-provider-oci
- OKE で KPO を使う（Oracle 公式ドキュメント）: https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/conteng-kpo.htm
- KPO サポート リリースノート: https://docs.oracle.com/en-us/iaas/releasenotes/conteng/conteng-Karpenter-Support-Release-Note.htm
