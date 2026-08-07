# 00. Karpenter とは / アーキテクチャ / 用語

## Karpenter とは

Karpenter（カーペンター）は、Kubernetes 向けの **ノードオートスケーラー**（ノードを自動で増減させる仕組み）です。
CNCF プロジェクトで、もともと AWS で生まれましたが、現在は **各クラウド向けのプロバイダー** が提供されています。

OCI 向けが **Karpenter Provider for OCI（KPO）** で、Oracle 公式が提供・サポートしています（GA 済み）。

## 何がうれしいのか（Cluster Autoscaler との違い）

Kubernetes には従来から **Cluster Autoscaler（CA）** というノード自動増減の仕組みがありました。
Karpenter は CA の課題を解決する、より新しいアプローチです。

| 観点 | Cluster Autoscaler（従来） | Karpenter（KPO） |
| --- | --- | --- |
| ノードのまとまり | 事前に **ノードプール（固定シェイプ）** を定義しておく必要がある | 固定シェイプを事前定義せず、**必要なスペックを都度選んで** 起動できる |
| スケール判断 | ノードプール単位でスケール | **未スケジュールの Pod を直接見て**、最適なインスタンスを即時プロビジョニング |
| 起動の速さ | 相対的に遅め | **速い**（Pod を検知して直接インスタンス起動） |
| 無駄の削減 | 限定的 | **Consolidation**（下記）でノードの削除・統合を自動化 |

**ひとことで言うと**：
「Pod が足りなくてスケジュールできない → Karpenter がちょうど良いサイズのノードを即座に用意する。
Pod が減って無駄が出た → Karpenter が自動で片付ける」——これを自動でやってくれます。

## Consolidation（統合）とは

上の表で出てきた **Consolidation** は Karpenter の重要な機能なので、ここで説明しておきます。

Karpenter が**継続的にクラスタを監視し、「今のノード構成には無駄がある」と判断したら、
自動でノードを整理する仕組み**です。「Pod が増えたらノードを足す」の**逆方向**、
つまり **コストを切り詰める側** の機能で、デモ②（[05](05-demo-consolidation.md)）の主役になります。

### 3 つのパターン

**① 空のノードを削除**

```
Node A [ Pod Pod ]     Node B [ 空 ]
                          ↓ 削除
```

Pod が居なくなったノードを消します。デモ②で主に見えるのはこれです。

**② Pod を寄せてノードを削除**

```
Node A [ Pod     ]     Node B [ Pod ]
          ↓ Pod を A に寄せられると判断
Node A [ Pod Pod ]     Node B → 削除
```

バラバラに動いている Pod を 1 台にまとめられるなら、Pod を退避（drain）させてノードを消します。

**③ より小さい（安い）ノードに置き換え**

```
Node [ 8 CPU の大きいノード / Pod は 1 つだけ ]
          ↓ 置き換え
Node [ 2 CPU の小さいノード / Pod 1 つ ]
```

過剰なスペックのノードを、より小さいノードに交換します。
**Cluster Autoscaler には無かった特徴**で、Karpenter が「無駄の削減」に強い理由です。

### 動作を決める 2 つの設定

[manifests/nodepool.yaml](../manifests/nodepool.yaml) の `disruption` がこれを制御します。

```yaml
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

| 設定値 | 挙動 |
| --- | --- |
| `WhenEmpty` | **①のみ**（空のノードだけ削除）＝控えめ・安全 |
| `WhenEmptyOrUnderutilized` | **①②③すべて**（使用率が低いノードも整理）＝本デモはこちら |
| `consolidateAfter` | 判断してから実行するまでの待ち時間。すぐ消して Pod が再び増えると無駄なため様子を見る |

> **なぜ嬉しいのか**
> Cluster Autoscaler では「ノードが完全に空でないと消せない」ため、
> **中途半端に空いたノードが残り続けて課金される**ことがよくありました。
> Consolidation は「まとめれば減らせる」を自動判断するので、無駄が出にくくなります。

> **Note：落としたくない Pod は保護できます**
> ②③では Pod の退避（drain）が発生します。停止させたくないワークロードは
> `PodDisruptionBudget` や `karpenter.sh/do-not-disrupt: "true"` アノテーションで保護できます。

## アーキテクチャ（ざっくり）

```
                ┌─────────────────────────────────────────────┐
                │                OKE クラスタ                    │
                │                                             │
  kubectl ───►  │  ┌──────────────┐   ①未スケジュールの        │
  (Podを増やす) │  │ kube-scheduler│      Pod を検知           │
                │  └──────┬───────┘                           │
                │         │                                   │
                │  ┌──────▼────────────┐                      │
                │  │ KPO コントローラー   │  ②必要スペックを計算   │
                │  │ (karpenter Pod)    │─────────┐           │
                │  └────────────────────┘         │           │
                └─────────────────────────────────┼───────────┘
                                                  │ ③OCI API で
                                                  ▼   インスタンス起動
                                    ┌───────────────────────────┐
                                    │  OCI Compute（新ノード）     │
                                    │  → OKE に自動 Join          │
                                    └───────────────────────────┘
```

- **KPO コントローラー** は、既存の（Karpenter 管理外の）ノード上で動きます。
  そのため、最初に **最低 1 つの管理対象ノードプール**（または self-managed ノード）が必要です。

## 覚えておきたい 2 つの CRD

KPO を使うときに書くのは、主に次の 2 つの Kubernetes リソース（CRD）です。

| リソース | 役割 | たとえるなら |
| --- | --- | --- |
| **OCINodeClass** | OCI 固有の「ノードの作り方」テンプレート。イメージ・ブートボリューム・サブネット・NSG など | **どんなマシンを作るか**（部品表） |
| **NodePool** | どんな条件でスケール／統合するか。シェイプ候補・容量タイプ・上限・Consolidation ポリシー | **いつ・どれだけ作る／消すか**（運用ルール） |

- `OCINodeClass` … API グループ `oci.oraclecloud.com/v1beta1`
- `NodePool` … API グループ `karpenter.sh/v1`（Karpenter 標準）

この 2 つを作成しておくと、あとは Pod を増減させるだけで Karpenter がノードを自動調整します。

## 次へ

→ [01. 事前準備](01-prerequisites.md)
