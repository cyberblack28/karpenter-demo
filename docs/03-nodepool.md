# 03. OCINodeClass / NodePool を作成

Karpenter に「どんなノードを、いつ作る／消すか」を教える 2 つのリソースを作成します。
（それぞれの役割は [00-overview.md](00-overview.md) の CRD 表を参照）

## 1. OCINodeClass（ノードの作り方）

[manifests/ocinodeclass.yaml](../manifests/ocinodeclass.yaml) の `<...>` を自分の環境の値に置き換えます。

置き換えるのは次の 3 つ（[01](01-prerequisites.md) で控えた値）：

- `subnetId: <worker-subnet-ocid>` … ワーカーノード用
- `networkSecurityGroupId: <worker-nsg-ocid>`（NSG 未使用なら該当ブロックごと削除）
- `subnetId: <pod-subnet-ocid>` … **Pod 用（VCN-Native では必須）**。ワーカーと同じ OCID でも可

適用します。

```bash
kubectl apply -f manifests/ocinodeclass.yaml
```

主なフィールドの意味：

| フィールド | 意味 |
| --- | --- |
| `shapeConfigs` | Flex シェイプの OCPU / メモリの候補 |
| `volumeConfig.bootVolumeConfig.imageConfig` | 使う OS イメージ（`OKEImage` で OKE 対応イメージを自動選択） |
| `networkConfig.primaryVnicConfig` | **ノード本体**を置くサブネット / NSG |
| `networkConfig.secondaryVnicConfigs` | **Pod 用**のセカンダリ VNIC。VCN-Native では**必須** |

> **⚠️ 最大のハマりどころ：`secondaryVnicConfigs` の設定漏れ**
> VCN-Native クラスタでこれを書き忘れると、ノードは起動して**クラスタ参加まで成功する**のに、
> CNI（`vcn-native-ip-cni`）が乗らず **NotReady のまま**になります。
> 症状と切り分け手順は [99-troubleshooting.md](99-troubleshooting.md) にまとめています。

> **`ipCount` の決め方**
> そのノードで Pod に割り当てる IP の数です（2 の累乗、最大 256）。
> 既存ノードプールの `max-pods-per-node` と同程度にしておくと揃った挙動になります
> （例：`31` なら `32`）。デモ用途なら `16` で十分です。

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

`OCINodeClass` の **READY が `True`** になっていることが重要です。ここが `True` なら、
サブネット OCID とイメージフィルタの解決に成功しています（OCID の誤りがあると `False` になります）。

両方が表示されれば準備完了です。まだノードは増えません（未スケジュールの Pod がないため）。
**NODES が `0`** なのが正しい状態です。実際に Pod を増やして動きを見るのは次のデモです。

## （参考）Flannel クラスタの場合

本デモは VCN-Native 前提です。CNI が **Flannel** のクラスタでは、Pod がノードの
overlay ネットワークを使うため **セカンダリ VNIC は不要**です。次の 2 点を変更してください。

1. `OCINodeClass` から `secondaryVnicConfigs` ブロックを**削除**する
2. Helm values を `settings.ociVcnIpNative: false` にする（[02](02-install-kpo.md)）

どちらの CNI かは [01](01-prerequisites.md) の `kubectl get ds -n kube-system` で判別できます。

## 次へ

→ [04.【デモ①】Pod を増やしてノード自動追加](04-demo-scaleout.md)
