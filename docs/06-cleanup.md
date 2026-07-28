# 06. 後片付け

デモ後の掃除です。**課金され続けないよう**、Karpenter が作ったノードを確実に消します。

## 1. デモ用アプリを削除

```bash
kubectl delete -f manifests/inflate.yaml
```

## 2. NodePool を削除（＝Karpenter 管理ノードを撤収）

NodePool を消すと、その NodePool が作ったノードもすべて片付きます。

```bash
kubectl delete -f manifests/nodepool.yaml
```

ノードが消えていくのを見届けます。

```bash
kubectl get nodeclaims -w
kubectl get nodes -l karpenter.sh/nodepool -w
```

Karpenter 管理ノードが 0 になったのを確認してから次へ進みます。

## 3. OCINodeClass を削除

```bash
kubectl delete -f manifests/ocinodeclass.yaml
```

## 4.（必要なら）KPO 本体をアンインストール

```bash
helm uninstall karpenter -n karpenter
kubectl delete namespace karpenter
```

## 5. OCI 側の後片付け（任意）

- デモ用に作った **IAM ポリシー / 動的グループ**（[01](01-prerequisites.md)）を残すか消すか判断
- OKE クラスタ自体を消す場合は OCI コンソール or `oci ce cluster delete`

> **確認**
> OCI コンソールの Compute → インスタンス一覧に、Karpenter が作ったインスタンスが
> 残っていないことを最後に目視確認しておくと安心です。

## 次へ

うまくいかなかった場合 → [99. トラブルシューティング](99-troubleshooting.md)
