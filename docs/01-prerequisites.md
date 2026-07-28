# 01. 事前準備（OKE クラスタ・ツール・IAM ポリシー）

デモを始める前に、以下を準備します。

## 1. 手元のツール

| ツール | 用途 | 確認コマンド |
| --- | --- | --- |
| OCI CLI | OCID の確認・OCI 操作 | `oci --version` |
| kubectl | クラスタ操作 | `kubectl version --client` |
| helm | KPO インストール | `helm version` |

## 2. OKE クラスタ

以下の条件を満たす OKE クラスタを用意します（既存クラスタでも可）。

- **Kubernetes バージョン**：KPO のバージョン対応表に従って選びます。

  | Kubernetes | 必要な KPO バージョン |
  | --- | --- |
  | 1.31 – 1.34 | v1.0.0 以降 |
  | 1.35 | v1.1.0 以降 |
  | 1.36 | v1.3.0 以降 |

  > 初心者デモでは、対応バージョンの中でも安定した版（例：1.33 系）を選ぶと無難です。

- **既存の管理対象ノードプールが最低 1 つ**（重要）
  KPO コントローラー（karpenter Pod）を動かす場所が必要です。
  ここは **小さめのノード 1〜2 台** で十分です（例：`VM.Standard.E5.Flex` / OCPU 1〜2）。
  ※ このノードプールは「Karpenter を動かす土台」であり、以降デモで自動追加されるノードとは別物です。

- **CNI**：本デモは **Flannel（OKE デフォルトの overlay）** を前提に、シンプルな構成で進めます。
  OCI VCN-Native Pod Networking を使う場合は、`OCINodeClass` に secondary VNIC 設定が必要です（[03](03-nodepool.md) の Note 参照）。

`kubectl` がクラスタに接続できることを確認しておきます。

```bash
kubectl get nodes
```

## 3. 控えておく環境固有の値（プレースホルダー一覧）

以降の YAML・コマンドで使う値です。**先にメモしておく** とスムーズです。

| プレースホルダー | 意味 | 取得のヒント |
| --- | --- | --- |
| `<cluster-ocid>` | OKE クラスタの OCID | OCI コンソール → Kubernetes Engine → クラスタ詳細 |
| `<cluster-compartment-id>` | クラスタが属するコンパートメント OCID | クラスタ詳細 |
| `<vcn-compartment-id>` | VCN が属するコンパートメント OCID | ネットワーキング → VCN |
| `<apiserver-private-ip>` | Kubernetes API サーバーのプライベート IP | クラスタ詳細 or `kubectl get endpoints kubernetes` |
| `<worker-subnet-ocid>` | ワーカーノード用サブネットの OCID | VCN → サブネット |
| `<worker-nsg-ocid>` | ワーカーノード用 NSG の OCID（任意） | VCN → ネットワークセキュリティグループ |

参考コマンド（API サーバー IP の確認例）：

```bash
kubectl get endpoints kubernetes -o jsonpath='{.subsets[0].addresses[0].ip}'
```

## 4. IAM ポリシー（KPO が OCI リソースを操作できるようにする）

KPO は OCI API を叩いて **インスタンスやボリュームを作成／削除** します。
そのための権限を、**ワークロードアイデンティティ（Workload Identity）** で付与します。

> **なぜ必要？**
> KPO（karpenter Pod）が「あなたの代わりに」コンピュートインスタンスを作るため、
> OCI 側で「この Pod にこの操作を許可する」と宣言しておく必要があります。

### 4-1. KPO 本体のポリシー

以下を OCI コンソール（アイデンティティ → ポリシー）で作成します。
`<namespace>` は KPO を入れる名前空間（本デモでは `karpenter`）、
`<service-account>` は Helm が作る SA 名（デフォルト `karpenter`）です。

```
Allow any-user to manage instance-family in compartment <compartment-name> where all {
  request.principal.type = 'workload',
  request.principal.namespace = 'karpenter',
  request.principal.service_account = 'karpenter',
  request.principal.cluster_id = '<cluster-ocid>'
}

Allow any-user to manage volume-family in compartment <compartment-name> where all {
  request.principal.type = 'workload',
  request.principal.namespace = 'karpenter',
  request.principal.service_account = 'karpenter',
  request.principal.cluster_id = '<cluster-ocid>'
}

Allow any-user to manage virtual-network-family in compartment <compartment-name> where all {
  request.principal.type = 'workload',
  request.principal.namespace = 'karpenter',
  request.principal.service_account = 'karpenter',
  request.principal.cluster_id = '<cluster-ocid>'
}

Allow any-user to inspect compartments in compartment <compartment-name> where all {
  request.principal.type = 'workload',
  request.principal.namespace = 'karpenter',
  request.principal.service_account = 'karpenter',
  request.principal.cluster_id = '<cluster-ocid>'
}
```

> `volume-family` / `virtual-network-family` は、細かく `volumes` `volume-attachments` に分けても構いません。
> デモでは family でまとめるとシンプルです。

### 4-2. 新ノードがクラスタに参加するためのポリシー

Karpenter が作った新しいインスタンスが OKE に **Join** できるよう、動的グループとポリシーを作成します。

**動的グループ**（例：`kpo-nodes-dyn-grp`）のマッチングルール：

```
ALL {instance.compartment.id = '<compartment-ocid>'}
```

**ポリシー**：

```
Allow dynamic-group kpo-nodes-dyn-grp to {CLUSTER_JOIN} in compartment <compartment-name>
  where { target.cluster.id = '<cluster-ocid>' }
```

> **チェックポイント**
> ここまでで「KPO が OCI インスタンスを作れる」＋「作ったノードが OKE に参加できる」状態になります。
> 権限不足はデモ当日の代表的なハマりどころです。→ [99-troubleshooting.md](99-troubleshooting.md)

## 次へ

→ [02. Helm で KPO をインストール](02-install-kpo.md)
