---
title: AWS ALBのdeletion_protectionにハマった話
tags:
  - AWS
  - kubernetes
  - SRE
private: false
updated_at: '2025-08-25T09:40:01+09:00'
id: 409ebadb50fd8a25f12d
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Kubernetes上でALB Ingress Controllerを利用している環境でIngressを削除しようとしたら、下記のエラーで失敗してしまいました。

```
{"level":"error","ts":"2025-08-25T02:40:50Z","msg":"Reconciler error","controller":"ingress","object":{"name":"xxx},"namespace":"","name":"xxx,"reconcileID":"xxx","error":"deletion_protection is enabled, cannot delete the ingress: xxx"}
```

マニフェストを見ても`deletion_protection.enabled=false`にしてるし、なぜ消えないのか意味不明🫠
この記事では、ALB Ingress Controllerのdeletion_protectionの正体と、解決までにやったことをまとめます。


## 前提: ALB Ingress Controllerとdeletion_protection

ALB Ingress Controller（現: AWS Load Balancer Controller）は、KubernetesのIngressリソースを解析して、AWS ALBを背後で自動生成してくれるコントローラです。

https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html

その構成の1つとして、ALBに対して`deletion_protection.enabled`を指定できます。デフォルトは`false`ですが、これが`true`だとIngressを`kubectl delete`しても、背後のALBが削除されずに残る場面があります。

```yaml
alb.ingress.kubernetes.io/load-balancer-attributes: deletion_protection.enabled=false
```

## 問題： Ingressを消せない

環境からIngressを削除しようとしたところ、マニフェストでは`deletion_protection.enabled=false`にしていたにも関わらず、ALBが削除できず以下のようなログを吐いて失敗しました。

```
deletion_protection is enabled, cannot delete the ingress
```

**🧠 どうやらALBの属性（Attributes）はImmutableなものが多く、一部はIngress Controllerからは更新不可らしい**
つまり、この属性はマニフェストを変えてもALB側には反映されないのです。

## 解決までにやったこと

### Step 1: マニフェストの修正

```yaml
alb.ingress.kubernetes.io/load-balancer-attributes: deletion_protection.enabled=false
```

当然、再適用しても何も変わりませんでした。

### Step 2: AWS Consoleで手動無効化

[EC2 > Load Balancers]から該当のALBを見つけ、Deletion protectionをOFFにしました。

### Step 3: ALBの再起動

それでもなお削除できない。
原因はIngress Controller側のcache保持による同期ズレだったらしい。

最終的に、関連Ingressを全部`kubectl rollout restart deployment`して、ALBをクリーンにしたらやっと削除できました。

## 何がむずいの？

- ALB Ingress Controllerの”Reconciler”は、マニフェストと実際の統合をやっているけど、ALBの実装側にはImmutableなフラグもある
- deletion_protection はその一つ。Kubernetesからは更新不可なため、転えて手動で変更する必要がある

## おわりに

なんとかIngressは削除できましたが、不気味な押し切り感は否めません。

GitOpsの実現においては、こういう「マニフェストで管理しているつもりだけど実は手動設定が残る」ようなケースを知っておくと、驚かずに対処できるようになります。

同じハマり方をしてる人の参考になれば幸いです🤓

## 参考

https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/

https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_LoadBalancerAttribute.html
