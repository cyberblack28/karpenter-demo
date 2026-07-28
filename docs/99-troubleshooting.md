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

## ノードは起動したが `NotReady` / OKE に Join しない

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
