---
title: "Webフロントエンドで脱SSRして料金を6割節約した話"
emoji: "⏭️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nuxt", "ssr", "csr"]
published: false
publication_name: ventus
---


このブログ記事では、ORICALのフロントエンドのインフラ構成を移行し、SSR（サーバーサイドレンダリング）をやめてCSR（クライアントサイドレンダリング）に一本化した際の話について、移行前と移行後の構成の違いに焦点を当ててご紹介します。移行にあたっては様々な苦労がありましたが、それはまた別のブログ記事で詳しくお話ししたいと思います。

# ORICALとは

まず、ORICALについて簡単にご紹介します。ORICALは、選手や力士、キャラクターやVTuberの電子トレカをコレクションし、それらを使って対戦などができるサービスです。

https://ventus-inc.com/ 

デジタルなカードであることを活かして、レアリティの高いカードには多彩なエフェクトが適用されていたり、静止画ではなく動画であったりします。

# 移行前の構成

ORICALは複数のスポーツチームや芸能事務所等（弊社ではパートナーと読んでいます）と提携し、パートナーごとにサービスを展開しています。

![](/images/spa-ventus-before.png)

移行前の構成は上の通りです。

まず、クラウドには基本的にはAWSを用いています。そして、パートナーごとにNuxt.jsのSSRフロントエンドが存在し、それぞれが独立したElastic Beanstalk環境で稼働していました。

実際にはバックエンドサーバーは複数存在していますが、ここでは話をわかりやすくするために一つとして扱います。

# 移行前の状況と課題

各パートナー向けのNuxt.jsフロントエンドは、Elastic Beanstalk上に構築されていました。

新しく提携いただけるパートナー様とORICALで新サービスをローンチするときには、AWSコンソール上で既存のフロントエンドインフラをコピーしたうえで新サービス用の設定変更を加えるという作業を行なっていました。

デプロイ時には、Elastic Beanstalk上に立てたプロダクションサーバーでビルドを行い、そのまま配信していました。また、メンテナンスを行う際は、Application Load Balancer (ALB) のルールを切り替えて、トラフィックを別のサーバーに転送することで実現していました。

最適なインフラではないと認識しつつも、このような構成を採用した理由は、手動での管理のしやすさを優先したためです。Webフロントエンドのインフラに関する情報がほぼすべてElastic Beanstalkに集約されているため、Infrastructure as a Codeとして環境を整備しなくてもコンソールを操作することでインフラの作成や編集を行うことができました。また、サービス開始当初はパートナーが1つのみだったため、コスト削減よりも迅速な開発を重視し、リリースに必要な工数と知識を最小限に抑えることを選択しました。

当初はこの構成で問題ありませんでしたが、サービスの成長とともに様々な問題が顕在化してきました。

- 設定変更の手間の増大
    - パートナーの数だけ設定を手動で更新しなければならず、プラットフォームバージョンやランタイムバージョンのアップグレードなどに時間がかかっていました。
    - メンテナンスの時に更新のレートリミットに引っかかるようになり、デプロイ時間が伸びるようになりました。
- サーバー稼働料金の増大
    - ビルドをプロダクションサーバー上で行っていたため、サービス負荷に応じたインスタンスサイズよりも大きなインスタンスが必要となり、コストが増加していました。
    - アプリケーションが肥大化するにつれてビルドに必要なメモリも増加し、問題はさらに深刻化していました。
    - 常にサーバーを稼働させておく必要があったため、費用がかかっていました。
- 急なアクセスの増加に対応できない
    - アプリケーションのビルドと起動に時間がかかるため、スケーラビリティに課題がありました。
    - オートスケーリングを設定していても、急なアクセス増加に対応するには10分弱かかることが多く、間に合わないケースが頻発していました。
    - 緊急の修正を反映する場合にも、同様に時間がかかっていました。また、本番環境でのローリングアップデートにより残されたサーバーに負荷が集中したり、デプロイ後のサーバーの失敗時に切り戻すのにも時間がかかるリスクがありました。
- 設定が統一されない
    - パートナーごとにユーザー数が異なるため、負荷に応じて微妙に設定が異なるインフラが存在していました。そのため、一括で設定を更新する際には、新旧の設定が混在する期間が発生し、管理の複雑さを増していました。

そして最終的に、SSRでしか実現できない機能がほとんどないことに気づきました。クライアントサイドだけでも動作する機能が大半を占める中で、サーバー側でレンダリングを行う必要性について疑問が生じました。

## 移行前のまとめ

サービス開発初期においては手軽さが大きなメリットでしたが、サービスの成長とともにそのメリットは薄れ、最適ではない構成であるが故の欠点が大きくなっていきました。そのため、インフラの移行を決断しました。

# 移行作業

移行作業は、新しいインフラと古いインフラを並行して稼働させた状態で、Route 53のレコードを更新することで、ALBからCloudFrontへの切り替えを行いました。これにより、ダウンタイムを最小限に抑えながら移行を実現しました。

# 移行後の構成と改善点

![](/images/spa-ventus-after.png)

移行後の構成は、パートナーごとに静的なhtmlファイルとjsファイルなどのアセットを生成し、Amazon S3とCloudFrontで配信するように変更しました。ビルドはGitHub Actionsで実行するようにし、メンテナンス時の挙動はCloudFront Functionsで制御しています。

また、OGPについても手を加えています。SSRを廃止したことによりXシェアなどでリンクを共有したときの動的なOGPが機能しなくなりますが、ORICALではユーザーが自分のファン活動を表現できるように ORICAL 内でのアクティビティを X にシェアする機能が利用されているため、OGPの重要度は高いです。このため、代替OGPを静的なHTMLファイルとしてシェア時に生成し、CloudFront Functionsでユーザーエージェントに応じて出し分けるシステムを構築しました。具体的には、botからのアクセスにはシェア時に生成されたOGP情報が含まれるHTMLファイルを返し、ユーザーからのアクセスには通常の出力を返します。これらの代替OGPは一定期間後に自動的に削除されるように設定しています。

この移行により、いくつかの改善が見られました。

まず、アクセス数の増加による可用性の低下をほぼ無視できるようになったため、スケーラビリティが大幅に向上しました。静的配信になったことでCDNキャッシュが有効になり、急なアクセス増加にも対応できるようになりました。また、キャッシュの有効化により転送料金も大幅に削減できました。

さらに、TerraformによるInfrastructure as Codeを実現したことで、インフラの管理が容易になりました。特に、Terraformによってその複雑さを人間が扱いやすい形で管理できるようになりました。

# 移行による効果

移行による効果は大きく、インフラ費用を1ヵ月あたり6割以上削減することができました！

![](/images/spa-ventus-billing-change.png)

また、クライアントサイドのビルドのみになったため、ビルド時間も短縮され、緊急の修正反映なども迅速に行えるようになりました。この成功は、他のインフラのコード化を推進する機運を高める結果にもつながりました。

# 移行のまとめ

脱SSRを行い、CSRに一本化したことでフロントエンドのコストを6割以上削減しました。また、Infrastructure as Codeの導入により、複雑なインフラを持続可能な形で保守できるようになりました。

今回の大規模更新はORICALが開始してからはじめてのものであったため、（当初1ヵ月の予想だったのが3ヵ月かかるなど）様々な苦労がありましたが、それはまた別の記事で紹介いたしますので、ぜひお待ちください。