---
title: "[AWS] VPC周りの設定変更には気をつけようねを体感した話"
emoji: "🤔"
type: "tech"
topics: [AWS,CDK,VPC]
published: false
---

# はじめに

AWS CDKを使ってVPC周りの設定変更に苦戦したので、

# 1. マルチAZ構成に変更したい

「マネコンで操作する分には設定変更ページでポチポチできた(気がする)ので、CDKでも簡単にできるだろう。」
そう思っていました。イメージは`availabilityZones[]`の配列にazを増やして`cdk deploy`するだけ（または`maxAzs`を1→2へ）。
しかしそんな簡単な問題ではありませんでした。

![](https://storage.googleapis.com/zenn-user-upload/6335536625d0-20251210.png)

```
The CIDR 'xx.xx.xx.x/xx' conflicts with another subnet
```
と怒られたのです。内容はマルチAZにしようとしてサブネット作ろうとしたけど、すでにそのIP範囲は使っているから競合していると見ればなんとなくわかりますが、自分は納得ができませんでした。なぜこうなったかを詳細に説明していきます。

## 1AZで構築していた理由

この時、下記のように実装していました

```typescript
const vpc = new ec2.Vpc(this, 'VPC', {
  ipAddresses: ec2.IpAddresses.cidr('10.45.96.0/19'),
  availabilityZones: ['ap-northeast-1a'],
  subnetConfiguration: [
    {
      name: 'Public',
      subnetType: ec2.SubnetType.PUBLIC,
      cidrMask: 24,
    },
    {
      name: 'PrivateSubnetWithEgress',
      subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
      cidrMask: 22,
    },
    {
      name: 'PrivateSubnetIsolated',
      subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
      cidrMask: 24,
    },
  ],
});
```

L2コンストラクトを使ったシンプルなVPCの設定です。
さらにここに書いている、`ipAddresses`(CIDR)や、各サブネットマスクはSREチームに設計してもらった内容を反映しています。各サブネットに作成されるリソースを考慮して3AZまでならIPアドレスが枯渇することはないような設定です。その上で、「今マルチAZにするとお金もかかるし、その時が来たら増やせば良いだろう」そう思って最初は1AZで構築していました。

## 問題が発生した

実際に1AZで構築した際のサブネット構成は以下の通りでした：

- **VPC全体**: `10.45.96.0/19`
- **Public**: `10.45.96.0/24`
- **PrivateSubnetWithEgress**: `10.45.100.0/22`
- **PrivateSubnetIsolated**: `10.45.104.0/24`

さて、時が経ち、マルチAZ構成に変更する必要が出てきました。`availabilityZones`にAZを追加して`cdk deploy`を実行すると、冒頭で紹介したエラーが発生しました。

```
The CIDR 'xx.xx.xx.x/xx' conflicts with another subnet
```

なぜこのエラーが発生したのでしょうか？

## なぜCIDRコンフリクトが起きたのか

CDKのVPCコンストラクトは、AZを追加する際に**自動的にサブネットのCIDRを割り当てます**。しかし、この自動割り当てのロジックは、既存のサブネットのCIDR範囲と競合する可能性があります。
実際に[issue](https://github.com/aws/aws-cdk/issues/6683)もあり、まだ改善はされていないみたいです...

1AZで構築した時点では、以下のような割り当てになっていました：
- Public: `10.45.96.0/24`
- PrivateSubnetWithEgress: `10.45.100.0/22`
- PrivateSubnetIsolated: `10.45.104.0/24`

2AZ目を追加しようとすると、CDKは新しいサブネットのためにCIDRを自動割り当てしようとします。しかし、既存のサブネットのCIDR範囲と競合してしまう可能性があるのです。
今回の場合だとPublicとPrivate2のサブネットマスクが同じなので割り当てられるIP範囲がランダムなのであれば被ることがあり得るのです。


## 解決策: `reservedAzs`を使う

この問題を解決する方法として、**`reservedAzs`**というプロパティがあります。

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2-readme.html#reserving-availability-zones

これは、VPC作成時に将来使用するAZを予約しておくことで、サブネットのCIDR割り当てを事前に計画できるようにする機能です。

```typescript
const vpc = new ec2.Vpc(this, 'VPC', {
  ipAddresses: ec2.IpAddresses.cidr('10.45.96.0/19'),
  maxAzs: 1, // or availabilityZones
  reservedAzs: 2, // 将来使用するAZを2つ予約
  subnetConfiguration: [
    {
      name: 'Public',
      subnetType: ec2.SubnetType.PUBLIC,
      cidrMask: 24,
    },
    {
      name: 'PrivateSubnetWithEgress',
      subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
      cidrMask: 22,
    },
    {
      name: 'PrivateSubnetIsolated',
      subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
      cidrMask: 24,
    },
  ],
});
```

`reservedAzs: 2`を指定することで、CDKは最初から2AZ分のサブネットCIDRを計画して割り当てます。これにより、後からAZを追加する際にCIDRコンフリクトが発生しなくなります。

## `reservedAzs`を使った場合のCIDR割り当て

実際に`reservedAzs: 2`を指定して構築した場合、以下のようなCIDR割り当てになりました：

### 1AZ目（実際に作成されるサブネット）
- **Public**: `10.45.96.0/24`
- **PrivateSubnetWithEgress**: `10.45.100.0/22`
- **PrivateSubnetIsolated**: `10.45.112.0/24` ← 注意：1AZ目でも`10.45.104.0/24`ではなく`10.45.112.0/24`が割り当てられました

### 2AZ目（予約されているが、まだ作成されていない）
- **Public**: `10.45.97.0/24`
- **PrivateSubnetWithEgress**: `10.45.104.0/22`
- **PrivateSubnetIsolated**: `10.45.117.0/24`

diffだとこんな感じです。↓
![](https://storage.googleapis.com/zenn-user-upload/a956259c0f99-20251210.png)

このように、`reservedAzs`を使うと、**最初から複数AZ分のCIDRが計画されるため、サブネットの割り当てが変わります**。1AZ目でも、将来的に3AZ分を考慮したCIDR割り当てが行われるため、既存の設計通りのCIDR（`10.45.104.0/24`）ではなく、`10.45.112.0/24`が割り当てられました。

## 複数回試行して確認した結果

この挙動を確認するため、複数回デプロイを試行しましたが、何度試しても同じ結果になりました。つまり、`reservedAzs`を使うことで、CIDR割り当てが一貫して行われることが確認できました。

## 後から`reservedAzs`を追加した場合

もし、最初に`reservedAzs`を指定せずに1AZで構築してしまった場合、後から`reservedAzs`を追加するとどうなるでしょうか？

この場合、**既存のサブネットのCIDRが再割り当てされます**。この場合も複数回試して実験してみたのですが、最初から指定した場合と同様のIP範囲が毎回割り当たりました。
このときCDKでは新規サブネット作成→既存のサブネットを削除という手順を踏みます。結局サブネットの削除が発生してしまうのでそこにあるリソース次第ではうまく削除できない場合がほとんどなのかなと思っています。`reservedAzs`を指定してデプロイが通れば、その後のAZ追加（`maxAzs`を増やす）では、**CIDRコンフリクトが発生しなくなります**が、できるだけVPC作成の時点で指定しておくのがベストでしょう。

## 2. VPCを削除しようとしたら削除できない

VPC周りの設定変更で苦戦した話はまだ続きます。今度は、VPCを削除しようとした際に発生した問題です。

### 問題の発生

開発環境のVPCを削除しようと、CDKスタックを削除（`cdk destroy`）しようとしました。しかし、**15分×3回**のタイムアウトが発生し、結局削除できませんでした。

```
Error: The following resource(s) failed to delete: [VPC]
```

この問題は、VPC内に存在する**ENI（Elastic Network Interface）**が原因でした。特に、以下のリソースが作成するENIが問題となっていました：

- **VPC Lambda**: VPC内のリソースにアクセスするためにVPC設定を持つLambda関数
- **VPC Endpoint**: VPC内のサービスへのプライベート接続を提供するエンドポイント

### なぜ削除できないのか

VPC LambdaやVPC Endpointは、VPC内のリソースにアクセスするために**ENIを作成します**。これらのENIは、VPCに紐付いているため、VPCを削除する前に削除する必要があります。

しかし、CDKの`cdk destroy`では、これらのENIが自動的に削除されない場合があります。特に、以下のような状況で問題が発生しやすいです：

1. **Lambda関数のENI**: Lambda関数がVPC設定を持っている場合、関数が削除されてもENIが残ることがある
2. **VPC EndpointのENI**: VPC Endpointが削除されても、関連するENIが残ることがある
3. **削除の順序**: CDKがリソースを削除する順序によっては、ENIが残ってしまう場合がある

実際に、この問題は多くの開発者が遭遇しており、[GitHubのissue](https://github.com/aws/aws-cdk/issues)でも報告されています。

### 解決策: 手動でENIの紐付けを解除

結局、以下の手順で手動対応を行いました：

1. **Lambda関数のVPC設定を解除**
   - AWSマネジメントコンソールで、VPC設定を持つLambda関数を特定
   - 各Lambda関数の設定からVPC設定を削除
   - これにより、Lambda関数が作成したENIが削除される

2. **VPC Endpointを削除**
   - VPCコンソールで、VPC Endpointを特定
   - 不要なVPC Endpointを削除
   - これにより、VPC Endpointが作成したENIが削除される

3. **残っているENIを手動削除**
   - EC2コンソールで、VPCに紐付いているENIを確認
   - 削除可能なENIを手動で削除

4. **VPCを手動削除**
   - すべてのENIを削除した後、AWSマネジメントコンソールからVPCを手動で削除

### 注意点

- **削除前の確認**: VPC削除前に、VPC内のすべてのリソース（EC2インスタンス、ロードバランサー、NATゲートウェイなど）を削除する必要があります
- **ENIの確認**: ENIは、リソースを削除しても残ることがあるため、削除前に確認することが重要です
- **削除の順序**: リソース間の依存関係を考慮して、適切な順序で削除する必要があります

### 今後の対策

この問題を回避するために、以下の対策を検討しています：

1. **VPC削除前のチェックリスト**: VPC削除前に、ENIが残っていないか確認するスクリプトを作成
2. **リソースの整理**: 不要なVPC LambdaやVPC Endpointを定期的に整理
3. **削除順序の最適化**: CDKの削除順序を最適化する（ただし、これはCDKの制約により難しい場合がある）

## まとめ

### 1. マルチAZ構成への変更

- 後からマルチAZに変更する場合、CIDR設計をしていてもCDKの自動割り当てロジックによってコンフリクトが発生する可能性がある
- `reservedAzs`を使うことで、最初から複数AZ分のCIDRを計画できる
- `reservedAzs`を使うと、サブネットのCIDR割り当てが変わるため、既存の設計と異なる可能性がある
- 一度`reservedAzs`を指定してデプロイすれば、その後のAZ追加ではコンフリクトが発生しなくなる
- 後から`reservedAzs`を追加すると、既存のサブネットが再作成される可能性があるため、影響範囲を考慮する必要がある

### 2. VPC削除時のENI問題

- VPC LambdaやVPC Endpointが作成するENIが原因で、VPCの削除が失敗することがある
- CDKの`cdk destroy`では、これらのENIが自動的に削除されない場合がある
- 手動でENIの紐付けを解除し、コンソールからVPCを削除する必要がある場合がある
- VPC削除前には、ENIが残っていないか確認することが重要

VPC周りの設定変更や削除は、思っている以上に影響範囲が大きく、複雑な問題が発生する可能性があります。最初から将来の拡張性を考慮した設計をしておくこと、そして削除時には慎重に確認することが重要ですね。