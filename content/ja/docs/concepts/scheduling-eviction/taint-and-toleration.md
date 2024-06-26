---
title: TaintとToleration
content_type: concept
weight: 50
---


<!-- overview -->
[_ノードアフィニティ_](/ja/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)は
{{< glossary_tooltip text="Pod" term_id="pod" >}}の属性であり、ある{{< glossary_tooltip text="ノード" term_id="node" >}}群を*引きつけます*（優先条件または必須条件）。反対に _taint_ はノードがある種のPodを排除できるようにします。

_toleration_ はPodに適用され、一致するtaintが付与されたノードへPodがスケジューリングされることを認めるものです。ただしそのノードへ必ずスケジューリングされるとは限りません。

taintとtolerationは組になって機能し、Podが不適切なノードへスケジューリングされないことを保証します。taintはノードに一つまたは複数個付与することができます。これはそのノードがtaintを許容しないPodを受け入れるべきではないことを示します。


<!-- body -->

## コンセプト

ノードにtaintを付与するには[kubectl taint](/docs/reference/generated/kubectl/kubectl-commands#taint)コマンドを使用します。
例えば、次のコマンドは

```shell
kubectl taint nodes node1 key1=value1:NoSchedule
```

`node1`にtaintを設定します。このtaintのキーは`key1`、値は`value1`、taintの効果は`NoSchedule`です。
これは`node1`にはPodに合致するtolerationがなければスケジューリングされないことを意味します。

上記のコマンドで付与したtaintを外すには、下記のコマンドを使います。
```shell
kubectl taint nodes node1 key1=value1:NoSchedule-
```

PodのtolerationはPodSpecの中に指定します。下記のtolerationはどちらも、上記の`kubectl taint`コマンドで追加したtaintと合致するため、どちらのtolerationが設定されたPodも`node1`へスケジューリングされることができます。

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

tolerationを設定したPodの例を示します。

{{% codenew file="pods/pod-with-toleration.yaml" %}}

`operator`のデフォルトは`Equal`です。

tolerationがtaintと合致するのは、`key`と`effect`が同一であり、さらに下記の条件のいずれかを満たす場合です。

* `operator`が`Exists`（`value`を指定すべきでない場合）
* `operator`が`Equal`であり、かつ`value`が同一である場合

{{< note >}}

2つ特殊な場合があります。

空の`key`と演算子`Exists`は全ての`key`、`value`、`effect`と一致するため、すべてのtaintと合致します。

空の`effect`は`key1`が一致する全てのeffectと合致します。

{{< /note >}}

上記の例では`effect`に`NoSchedule`を指定しました。代わりに、`effect`に`PreferNoSchedule`を指定することができます。
これは`NoSchedule`の「ソフトな」バージョンであり、システムはtaintに対応するtolerationが設定されていないPodがノードへ配置されることを避けようとしますが、必須の条件とはしません。3つ目の`effect`の値として`NoExecute`がありますが、これについては後述します。

同一のノードに複数のtaintを付与することや、同一のPodに複数のtolerationを設定することができます。
複数のtaintやtolerationが設定されている場合、Kubernetesはフィルタのように扱います。最初はノードの全てのtaintがある状態から始め、Podが対応するtolerationを持っているtaintは無視され外されていきます。無視されずに残ったtaintが効果を及ぼします。
具体的には、

* effect `NoSchedule`のtaintが無視されず残った場合、KubernetesはそのPodをノードへスケジューリングしません。
* effect `NoSchedule`のtaintは残らず、effect `PreferNoSchedule`のtaintは残った場合、Kubernetesはそのノードへのスケジューリングをしないように試みます。
* effect `NoExecute`のtaintが残った場合、既に稼働中のPodはそのノードから排除され、まだ稼働していないPodはスケジューリングされないようになります。

例として、下記のようなtaintが付与されたノードを考えます。

```shell
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

Podには2つのtolerationが設定されています。

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

この例では、3つ目のtaintと合致するtolerationがないため、Podはノードへはスケジューリングされません。
しかし、これらのtaintが追加された時点で、そのノードでPodが稼働していれば続けて稼働することが可能です。 これは、Podのtolerationと合致しないtaintは3つあるtaintのうちの3つ目のtaintのみであり、それが`NoSchedule`であるためです。

一般に、effect `NoExecute`のtaintがノードに追加されると、合致するtolerationが設定されていないPodは即時にノードから排除され、合致するtolerationが設定されたPodが排除されることは決してありません。
しかし、effect`NoExecute`に対するtolerationは`tolerationSeconds`フィールドを任意で指定することができ、これはtaintが追加された後にそのノードにPodが残る時間を示します。例えば、

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

この例のPodが稼働中で、対応するtaintがノードへ追加された場合、Podはそのノードに3600秒残り、その後排除されます。仮にtaintがそれよりも前に外された場合、Podは排除されません。

## ユースケースの例

taintとtolerationは、実行されるべきではないノードからPodを遠ざけたり、排除したりするための柔軟な方法です。いくつかのユースケースを示します。

* **専有ノード**: あるノード群を特定のユーザーに専有させたい場合、そのノード群へtaintを追加し(`kubectl taint nodes nodename dedicated=groupName:NoSchedule`) 対応するtolerationをPodへ追加します（これを実現する最も容易な方法はカスタム
[アドミッションコントローラー](/docs/reference/access-authn-authz/admission-controllers/)を書くことです）。
tolerationが設定されたPodはtaintの設定された（専有の）ノードと、クラスターにあるその他のノードの使用が認められます。もしPodが必ず専有ノード*のみ*を使うようにしたい場合は、taintと同様のラベルをそのノード群に設定し(例: `dedicated=groupName`)、アドミッションコントローラーはノードアフィニティを使ってPodが`dedicated=groupName`のラベルの付いたノードへスケジューリングすることが必要であるということも設定する必要があります。

* **特殊なハードウェアを備えるノード**: クラスターの中の少数のノードが特殊なハードウェア（例えばGPU）を備える場合、そのハードウェアを必要としないPodがスケジューリングされないようにして、後でハードウェアを必要とするPodができたときの余裕を確保したいことがあります。
これは特殊なハードウェアを持つノードにtaintを追加(例えば `kubectl taint nodes nodename special=true:NoSchedule` または
`kubectl taint nodes nodename special=true:PreferNoSchedule`)して、ハードウェアを使用するPodに対応するtolerationを追加することで可能です。
専有ノードのユースケースと同様に、tolerationを容易に適用する方法はカスタム
[アドミッションコントローラー](/docs/reference/access-authn-authz/admission-controllers/)を使うことです。
例えば、特殊なハードウェアを表すために[拡張リソース](/docs/concepts/configuration/manage-compute-resources-container/#extended-resources)
を使い、ハードウェアを備えるノードに拡張リソースの名称のtaintを追加して、
[拡張リソースtoleration](/docs/reference/access-authn-authz/admission-controllers/#extendedresourcetoleration)
アドミッションコントローラーを実行することが推奨されます。ノードにはtaintが付与されているため、tolerationのないPodはスケジューリングされません。しかし拡張リソースを要求するPodを作成しようとすると、`拡張リソースtoleration` アドミッションコントローラーはPodに自動的に適切なtolerationを設定し、Podはハードウェアを備えるノードへスケジューリングされます。
これは特殊なハードウェアを備えたノードではそれを必要とするPodのみが稼働し、Podに対して手作業でtolerationを追加しなくて済むようにします。

* **taintを基にした排除**: ノードに問題が起きたときにPodごとに排除する設定を行うことができます。次のセクションにて説明します。

## taintを基にした排除

{{< feature-state for_k8s_version="v1.18" state="stable" >}}

上述したように、effect `NoExecute`のtaintはノードで実行中のPodに次のような影響を与えます。

 * 対応するtolerationのないPodは即座に除外される
 * 対応するtolerationがあり、それに`tolerationSeconds`が指定されていないPodは残り続ける
 * 対応するtolerationがあり、それに`tolerationSeconds`が指定されているPodは指定された間、残される

Nodeコントローラーは特定の条件を満たす場合に自動的にtaintを追加します。
組み込まれているtaintは下記の通りです。

 * `node.kubernetes.io/not-ready`: ノードの準備ができていない場合。これはNodeCondition `Ready`が`False`である場合に対応します。
 * `node.kubernetes.io/unreachable`: ノードがノードコントローラーから到達できない場合。これはNodeCondition`Ready`が`Unknown`の場合に対応します。
 * `node.kubernetes.io/out-of-disk`: ノードのディスクの空きがない場合。
 * `node.kubernetes.io/memory-pressure`: ノードのメモリーが不足している場合。
 * `node.kubernetes.io/disk-pressure`: ノードのディスクが不足している場合。
 * `node.kubernetes.io/network-unavailable`: ノードのネットワークが利用できない場合。
 * `node.kubernetes.io/unschedulable`: ノードがスケジューリングできない場合。
 * `node.cloudprovider.kubernetes.io/uninitialized`: kubeletが外部のクラウド事業者により起動されたときに設定されるtaintで、このノードは利用不可能であることを示します。cloud-controller-managerによるコントローラーがこのノードを初期化した後にkubeletはこのtaintを外します。

ノードから追い出すときには、ノードコントローラーまたはkubeletは関連するtaintを`NoExecute`効果の状態で追加します。
不具合のある状態から通常の状態へ復帰した場合は、kubeletまたはノードコントローラーは関連するtaintを外すことができます。

{{< note >}}
コントロールプレーンは新しいtaintをノードに加えるレートを制限しています。
このレート制限は一度に多くのノードが到達不可能になった場合（例えばネットワークの断絶）に、退役させられるノードの数を制御します。
{{< /note >}}

Podに`tolerationSeconds`を指定することで不具合があるか応答のないノードに残る時間を指定することができます。

例えば、ローカルの状態を多数持つアプリケーションとネットワークが分断された場合を考えます。ネットワークが復旧して、Podを排除しなくて済むことを見込んで、長時間ノードから排除されないようにしたいこともあるでしょう。
この場合Podに設定するtolerationは次のようになります。

```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

{{< note >}}
Kubernetesはユーザーまたはコントローラーが明示的に指定しない限り、自動的に`node.kubernetes.io/not-ready`と`node.kubernetes.io/unreachable`に対するtolerationを`tolerationSeconds=300`にて設定します。

自動的に設定されるtolerationは、taintに対応する問題がノードで検知されても5分間はそのノードにPodが残されることを意味します。
{{< /note >}}

[DaemonSet](/ja/docs/concepts/workloads/controllers/daemonset/)のPodは次のtaintに対して`NoExecute`のtolerationが`tolerationSeconds`を指定せずに設定されます。

  * `node.kubernetes.io/unreachable`
  * `node.kubernetes.io/not-ready`

これはDaemonSetのPodはこれらの問題によって排除されないことを保証します。

## 条件によるtaintの付与

ノードのライフサイクルコントローラーはノードの状態に応じて`NoSchedule`効果のtaintを付与します。
スケジューラーはノードの状態ではなく、taintを確認します。
ノードに何がスケジューリングされるかは、そのノードの状態に影響されないことを保証します。ユーザーは適切なtolerationをPodに付与することで、どの種類のノードの問題を無視するかを選ぶことができます。

DaemonSetのコントローラーは、DaemonSetが中断されるのを防ぐために自動的に次の`NoSchedule`tolerationを全てのDaemonSetに付与します。

  * `node.kubernetes.io/memory-pressure`
  * `node.kubernetes.io/disk-pressure`
  * `node.kubernetes.io/out-of-disk` (*重要なPodのみ*)
  * `node.kubernetes.io/unschedulable` (1.10またはそれ以降)
  * `node.kubernetes.io/network-unavailable` (*ホストネットワークのみ*)

これらのtolerationを追加することは後方互換性を保証します。DaemonSetに任意のtolerationを加えることもできます。


## {{% heading "whatsnext" %}}

* [リソース枯渇の対処](/docs/tasks/administer-cluster/out-of-resource/)とどのような設定ができるかについてを読む
* [Podの優先度](/docs/concepts/configuration/pod-priority-preemption/)を読む
