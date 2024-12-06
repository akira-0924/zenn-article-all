---
title: "ずっと曖昧だったプロキシについてここらで理解しておく"
emoji: "🔍"
type: "tech"
topics: [proxy, network]
published: false
---

結論、NAT が必要です。

ただ、App Runner からインターネットにアクセスする際に「なんで NAT が必要になるのか？」となったのでまとめました。

ある案件でコンテナサービスで App Runner を使用することにしました。
「AWS App Runner について簡潔に教えて」GPT に聞いてみました。

> AWS App Runner は、AWS が提供する完全に管理されたアプリケーション実行サービスです。AWS App Runner を使用すると、コンテナ化されたアプリケーションやコードリポジトリからのコードベースのアプリケーションを簡単にデプロイ、実行、スケーリングすることができます。
> AWS App Runner では、ユーザーがアプリケーションのソースコードやコンテナイメージを指定し、それを基に自動的にデプロイメントパイプラインを構築します。また、リアルタイムの監視、ログ収集、自動スケーリングなどの機能も提供されます。
> AWS App Runner は、開発者やチームがアプリケーションのデプロイメントに関連する複雑さを最小限に抑えることができるため、迅速かつ効率的なアプリケーションの実行が可能です。また、コンテナオーケストレーションやサーバーレスの利点を享受しながら、インフラストラクチャの管理に時間を費やすことなくアプリケーションの開発に集中することができます。
> AWS App Runner は、Web アプリケーション、API、バッチジョブなど、さまざまなタイプのアプリケーションの実行に使用できます。

要は、コンテナ化されたウェブアプリケーションや API を開発者が簡単かつ迅速にデプロイできるフルマネージド型サービスです。
開発者は Git のソースコードか、ECR をプッシュするだけであとは App Runner が自動デプロイまでしてくれます。（VPC とか、AutoScalling とか ALB とか全部いらない）

というわけで今回最初に作成した構成（一部）がこちら ↓（赤枠だけ見てください）
![スクリーンショット 2023-12-18 13.44.38.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3655825/2fc07a3e-2ab7-385c-22f3-5c2563944dad.png)

App Runner が 2022 年 2 月のアップデートで VPC コネクタを使用することで VPC 内のリソースとプライベートなアクセスができるようになったことは知っていました。↓
https://dev.classmethod.jp/articles/aws-app-runner-supports-privately-accessible-services-amazon-vpc/
<br>
（それまではインターネットからの公開アクセス専用でありプライベートアクセスに制限することはできなかったらしい。（プライベートサブネットにアクセスできなかったため RDS 等をパブリックサブネットに配置しなきゃいけなかった））
<br>
なので、App Runner から VPC コネクタ経由でプライベートサブネットの Aurora や ElastiCache にアクセス！さらに外部 API のコールが必要があったので、そのまま App Runner から外部 API をコール！としていました。
<br>
しかしこのままでは厳密には、App Runner は外部 API（図の右上にある Admin API 　 NOMOSHOP API）をコールすることはできません。
<br>
App Runner で VPC コネクタを使用する場合、全てのアウトバウンド通信は VPC 経由になるので直接外部の API をコールすることができないのです。

というわけでどうすればいいのか、以下に記載します。
<br>

## App Runner からインターネットにアクセスするには？

「App Runner はフルマネージド型サービスで、開発者は Git のソースコードか、ECR をプッシュするだけ」
この説明に頼りすぎて内部的に何が行われているのかを理解していませんでした。
<br>

結論、App Runner がインターネットにアクセスするには、NAT ゲートウェイとインターネットゲートウェイが必要になります。

正確には **「App Runner の内部の VPC にアタッチされている ENI のプライベート IP アドレスを介して、各 Fargate タスクにプライベートアクセスするので VPC ネットワーキングモードで使用する場合、NAT ゲートウェイとインターネットゲートウェイが必要になります」**
<br>

順に説明していきます。
<br>

### パブリックネットワーキングモードと VPC ネットワーキングモード

https://aws.amazon.com/jp/blogs/news/deep-dive-on-aws-app-runner-vpc-networking/

**パブリックネットワーキングモード**
2022 年 2 月のアップデートがあるまではこちらの方法でしか使用することができませんでした。App Runner は内部に VPC を所有しています。

![スクリーンショット 2023-12-18 13.53.28.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3655825/6a9fe0f2-8c83-9c17-fa95-76e24610076f.png)

・インバウンド通信は App Runner VPC 内部の NLB、ENI を経由して最終的にアプリケーションコンテナに到達します。
・アウトバウンド通信は App Runner VPC 内部の ENI、NAT Gateway、Internet Gateway を通ってインターネットに出ていきます
<br>

この図にあるように独自の VPC は登場しません。そしてアウトバウンド通信はインターネットから出て行くので、アプリケーションコンテナからアクセスできるのはインターネット経由で疎通できるサービスに限られます。つまり **S3 や DynamoDB にはインターネット経由で通信できるのでアクセスできますが、独自の VPC 内部のプライベートサブネットにあるリソース（Aurora や ElastiCache）にアクセスすることはできません** でした。上述した「プライベートサブネットにアクセスできない」はこのことです。

<br>
<br>
<br>
 
 
 **VPCネットワーキングモード** 
一方、こちらは2022年のアップデートで対応されたVPCサポートによるものになります。

![スクリーンショット 2023-12-18 13.54.46.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3655825/1f7a26f2-5648-e5fe-1d66-ed0e3675073f.png)

・インバウンド通信が ENI に到達するまではパブリックネットワーキングモードと同じです。
・アウトバウンド通信は AWS Hyperplane というものを用いて Fargate ENI から独自の VPC 内部の ENI へルーティングされます。
**→ つまりこの先は独自の VPC 内部のルーティングに依存してきます！**

↑ この部分に気づけませんでした...。
<br>

パブリックネットワーキングモードの場合は、図にあるように App Runner が管理している VPC に NAT Gateway や Internet Gateway があるのでお客様側で用意する必要はありませんでした。
<br>
一方 VPC ネットワーキングモードの場合は、アウトバウンド通信は全てお客様の VPC に行ってしまうので、そこからインターネットに出ていこうと思った時にルートテーブルの設定をしたり NAT Gateway、Internet Gateway などのリソースを作成する必要があるのです。（図の Customer の部分に NAT ゲートウェイや Internet Gateway がありますね）
<br>

というわけで、修正した AWS 構成時がこちら ↓
![スクリーンショット 2023-12-18 13.57.26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3655825/a1573203-5f78-784e-31de-142669269b61.png)

これで外部 API にインターネット経由で接続することができました！
便利な分ブラックボックスになっている部分が多くて構造を理解するのが大変ですね...
<br>
<br>

結論...
**App Runner で VPC コネクタを使用する場合にインターネットにアクセスしたいときは NAT が必要！**
App Runner を使用することがあるときはこの言葉を思い出してください。

## 参考

https://dev.classmethod.jp/articles/aws-app-runner-supports-privately-accessible-services-amazon-vpc/

https://aws.amazon.com/jp/blogs/news/deep-dive-on-aws-app-runner-vpc-networking/
