# 05.【デモ②】Pod を減らしてノード自動削除・統合（スケールイン / Consolidation）

Karpenter の真骨頂、**無駄なノードの自動片付け（Consolidation）** を見せます。

## デモの筋書き

> 「負荷が下がって Pod が減ると、ノードがガラ空きになります。
> Cluster Autoscaler 時代は『空いてるのに残ったまま』が起きがちでした。
> Karpenter は空き・低使用率のノードを検知して、**自動で削除・統合** し、
> コストの無駄を減らします。」

## 0. 監視（デモ①の監視をそのまま使ってOK）

```bash
# ターミナルA
kubectl get nodes -L karpenter.sh/nodepool -w
# ターミナルB
kubectl get nodeclaims -w
```

現状、デモ①で 5 Pod が動き、Karpenter ノードが数台ある状態を前提とします。

```bash
kubectl get nodes -L karpenter.sh/nodepool
```

## 1. Pod を大きく減らす

```bash
kubectl scale deployment/inflate --replicas=1
```

## 2. 何が起きるか観察する

NodePool の `consolidateAfter: 1m` の設定により、**約 1 分後** に統合が始まります。

1. 使われなくなったノードが **空 or 低使用率** と判定される
2. Karpenter が残った Pod を別ノードに **退避（drain）**
3. 不要になったノードの NodeClaim が削除され、**OCI インスタンスが終了**
4. ターミナルA からノードが消えていく

Karpenter の判断はログで確認できます。

```bash
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=50 | grep -i consolidat
```

`disrupting node(s) via consolidation` のようなログが出ます。

## 3. 0 まで減らして「全部片付く」様子も見せる

```bash
kubectl scale deployment/inflate --replicas=0
```

しばらくすると、Karpenter が作ったノードは **すべて削除** され、
最初の「Karpenter を動かす土台ノード」だけが残ります。

```bash
kubectl get nodes -L karpenter.sh/nodepool
```

## 4. 見せ場のトーク

- 「Pod を減らしただけで、**誰も手を動かさずに** ノードが減りました」
- 「`consolidationPolicy: WhenEmptyOrUnderutilized` は、空だけでなく
  『使用率が低くて 1 台にまとめられる』ケースも統合します」
- 「起動＝速く、片付け＝自動。**必要な分だけ払う** を実現できます」

> **補足：drain されたくない Pod がある場合**
> 本番では `PodDisruptionBudget` や `karpenter.sh/do-not-disrupt: "true"` アノテーションで
> 統合対象から守れます。デモでは触れずに「守る手段もあります」と一言添えるとよいです。

## 次へ

→ [06. 後片付け](06-cleanup.md)
