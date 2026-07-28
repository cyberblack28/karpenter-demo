# 03. OCINodeClass / NodePool を作成

Karpenter に「どんなノードを、いつ作る／消すか」を教える 2 つのリソースを作成します。
（それぞれの役割は [00-overview.md](00-overview.md) の CRD 表を参照）

## 1. OCINodeClass（ノードの作り方）

[manifests/ocinodeclass.yaml](../manifests/ocinodeclass.yaml) の `<...>` を自分の環境の値に置き換えます。

置き換えるのは主に次の 2 つ（[01](01-prerequisites.md) で控えた値）：

- `subnetId: <worker-subnet-ocid>`
- `networkSecurityGroupId: <worker-nsg-ocid>`（NSG 未使用なら該当ブロックごと削除）

適用します。

```bash
kubectl apply -f manifests/ocinodeclass.yaml
```

主なフィールドの意味：

| フィールド | 意味 |
| --- | --- |
| `shapeConfigs` | Flex シェイプの OCPU / メモリの候補 |
| `volumeConfig.bootVolumeConfig.imageConfig` | 使う OS イメージ（`OKEImage` で OKE 対応イメージを自動選択） |
| `networkConfig.primaryVnicConfig` | ノードを置くサブネット / NSG |

## 2. NodePool（スケールの運用ルール）

[manifests/nodepool.yaml](../manifests/nodepool.yaml) をそのまま適用します（環境依存の値はありません）。

```bash
kubectl apply -f manifests/nodepool.yaml
```

このデモ用 NodePool のポイント：

| 設定 | 値 | 意味 |
| --- | --- | --- |
| `requirements` (capacity-type) | `on-demand` | オンデマンドのみ（Spot は使わない） |
| `requirements` (instance-shape) | `E5.Flex` / `E4.Flex` | 使ってよいシェイプ候補 |
| `disruption.consolidationPolicy` | `WhenEmptyOrUnderutilized` | 空 or 低使用率のノードを統合して削減 |
| `disruption.consolidateAfter` | `1m` | 1 分様子を見てから統合（デモ用に短め） |
| `limits.cpu` | `16` | 暴走防止。この NodePool 全体で最大 16 CPU まで |

> **安全弁について**
> `limits` は必ず入れておきます。設定ミスや想定外の Pod 増加でノードが無制限に増えるのを防ぎます。

## 3. 作成確認

```bash
kubectl get ocinodeclass
kubectl get nodepool
```

両方が表示されれば準備完了です。まだノードは増えません（未スケジュールの Pod がないため）。
実際に Pod を増やして動きを見るのは次のデモです。

## （参考）VCN-Native Pod Networking を使う場合

本デモは Flannel 前提です。OCI VCN-Native CNI を使う環境では、`OCINodeClass` に Pod 用の
secondary VNIC 設定が必要になります。

```yaml
  networkConfig:
    # ... primaryVnicConfig は同じ ...
    secondaryVnicConfigs:
      - subnetConfig:
          subnetId: <pod-subnet-ocid>
        ipCount: 32   # 2 の累乗（最大 256）
```

あわせて Helm values の `settings.ociVcnIpNative: true` も設定します（[02](02-install-kpo.md)）。

## 次へ

→ [04.【デモ①】Pod を増やしてノード自動追加](04-demo-scaleout.md)
