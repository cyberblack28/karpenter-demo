# 04.【デモ①】Pod を増やしてノード自動追加（スケールアウト）

いよいよ本番。**Pod を増やすと、Karpenter が自動でノードを追加する** 様子を見せます。

## デモの筋書き（話しながら実演）

> 「今、クラスタには Karpenter を動かす小さなノードしかありません。
> ここに CPU を要求する Pod をたくさん投入します。
> スケジューラは載せる場所がなくて困る（Pending になる）はずです。
> すると Karpenter がそれを検知して、ちょうど良いノードを **自動で** 立ち上げます。」

## 0. 準備（画面を 2〜3 分割しておくと映えます）

**ターミナルA：ノードの動きを監視**

```bash
kubectl get nodes -L karpenter.sh/nodepool -w
```

**ターミナルB：Karpenter が作るノード要求(NodeClaim)を監視**

```bash
kubectl get nodeclaims -w
```

**ターミナルC：操作用**（ここでコマンドを打つ）

## 1. デモ用 Deployment を投入（まずは 0 レプリカ）

```bash
kubectl apply -f manifests/inflate.yaml
```

この時点では `replicas: 0` なので何も起きません。

## 2. Pod を一気に増やす

各 Pod は CPU を 1 つ要求します。5 レプリカにして、既存ノードに載り切らない状況を作ります。

```bash
kubectl scale deployment/inflate --replicas=5
```

## 3. 何が起きるか観察する

数十秒で以下が順に起こります。

1. **Pod が Pending になる**（載せる場所がない）
   ```bash
   kubectl get pods -l app=inflate
   ```
2. **Karpenter が NodeClaim を作成**（ターミナルB に新しい行が出る）
   ```bash
   kubectl get nodeclaims
   ```
3. **OCI 上で新インスタンスが起動し、ノードとして Join**（ターミナルA に新ノードが `Ready` で出る）
4. **Pending だった Pod が新ノードに載って `Running` に**

Karpenter が「なぜそのノードを選んだか」はイベントで確認できます。

```bash
kubectl describe nodeclaim | head -n 40
```

## 4. 見せ場のトーク

- 「事前にノードプールのサイズを決めていないのに、**必要な分だけ** ノードが増えました」
- 「Pending を検知してから **数十秒〜数分** で新ノードが Ready になります」
- 「`limits.cpu: 16` を超える要求をしても、そこで頭打ちになります（安全弁）」

### （応用）上限に当てるデモ

さらに増やすと `limits` で頭打ちになる様子も見せられます。

```bash
kubectl scale deployment/inflate --replicas=30
```

一部の Pod が Pending のまま残り、`kubectl get events` に
「NodePool の上限に達した」旨のメッセージが出ます。
（見せ終わったら `--replicas=5` に戻しておくと次のデモにつなげやすいです）

```bash
kubectl scale deployment/inflate --replicas=5
```

## 次へ

→ [05.【デモ②】Pod を減らしてノード自動削除・統合](05-demo-consolidation.md)
