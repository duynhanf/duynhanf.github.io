---
layout: post
title: Migration từ EC2(MySQL) sang Aurora MySQL
permalink: /aws/migration-ec2-mysql-sang-aurora-mysql/
tags: aws ec2 aurora mysql
---
<h2>Mở đầu</h2>
Mọi người đã thử sử dụng Amazon OpenSearch Service chưa?

Trong những công việc gần đây, tôi đã được tham gia vào việc đổi cấu hình của OpenSearch.
Vào lúc đó, tôi đã quan tâm đến cách thức triển khai, và tôi đã tổng hợp kết quả nghiên cứu của mình về sự khác biệt giữa chúng, bao gồm cả kết quả thử nghiệm.
Nhất định xem qua nhé!

<h2>Amazon OpenSearch Service là gì?</h2>


Tôi sẽ giới thiệu về Amazon OpenSearch Service một cách tổng quan.
Nó là một dịch vụ quản lý được cung cấp bởi AWS, cho phép bạn dễ dàng thực hiện phân tích dữ liệu rộng lớn như giám sát ứng dụng thời gian thực, phân tích log, tìm kiếm trên trang web, v.v. bằng cách sử dụng dashboard chuyên dụng.

Ban đầu, AWS cung cấp dịch vụ dựa trên mã nguồn của ElasticSearch, nhưng từ năm 2021, dịch vụ Amazon OpenSearch Service được tạo ra như một dịch vụ riêng của AWS. Vì vậy, dịch vụ này có nhiều điểm tương đồng với ElasticSearch.

<h2>Về cách triển khai</h2>
<h3>Các loại triển khai</h3>
Cụ thể, cách triển khai của OpenSearch được chia thành 2 loại chính.

1. Blue/Green deploy
2. Triển khai không có Blue/Green deploy

Về cách triển khai Blue/Green, khi deploy được tiến hành, một môi trường mới được tạo ra và sau khi cập nhật hoàn tất, hệ thống sẽ chuyển sang môi trường mới.

Về cách 2, không có cách triển khai chi tiết được công bố.
(Tôi đoán rằng nó sẽ được triển khai theo kiểu rolling deploy cho từng node)

Về cách triển khai nào sẽ được áp dụng, điều này sẽ được quyết định bởi logic bên trong của OpenSearch, vì vậy người dùng không thể chọn lựa. 
Tuy nhiên, bằng việc chạy dry-run, chúng ta có thể confirm trước xem là phương pháp deploy nào sẽ được dùng đối với config

Điều kiện để deploy Blue/Green

Nói thế thì, điều kiện lớn khi Blue/Green deploy được chạy thì được AWS công khai trên docs.
Tuy nhiên, dù không đủ điều kiện để chạy Blue/Green thì cũng không đảm bảo rằng sẽ không chạy Blue/Green. Do đó, trước khi deploy, việc confirm bằng dry-run là rất quan trọng

通常は Blue/Green デプロイの原因となる変更
- インスタンスタイプを変更する
- きめ細かなアクセスコントロールの有効化
- サービスソフトウェアのアップデートを行う
- ドメインに専用マスターノードがない場合に、データインスタンス数を変更する
- 専用マスターノードを有効または無効にする
- スタンバイなしのマルチ AZ を有効または無効にする
- ストレージタイプ、ボリュームタイプ、ボリュームサイズを変更する　※詳細は下記アップデート情報を参照
- 別の VPC のサブネットを選択する
- VPC セキュリティグループを追加または削除する
- OpenSearch Dashboards の Amazon Cognito 認証の有効化または無効化
- 別の Amazon Cognito ユーザープールまたは ID プールを選択する
- アドバンスト設定を変更する
- 新しい OpenSearch バージョンへのアップグレード
- 保管中のデータの暗号化または node-to-node 暗号化の有効化
- UltraWarm またはコールドストレージの有効化または無効化
- Auto-Tune を無効にし、変更をロールバックする
- ドメインへのオプションプラグインの関連付けとドメインからのオプションプラグインの関連付けの解除
- 専用マスターノードが 2 つあり、ゾーン認識が有効になっているドメインの専用マスターノード数を増やす

通常は Blue/Green デプロイが発生しない変更

- アクセスポリシーを変更する
- カスタムエンドポイントの変更
- Transport Layer Security (TLS) ポリシーを変更する
- 自動スナップショットの時間を変更する
- [HTTPS が必要] を有効または無効にする
- 変更内容をロールバックせずに Auto-Tune を有効または無効にする
- ドメインに専用マスターノードがある場合、データノードまたは UltraWarm ノード数を変更する
- ドメインに専用マスターノードがある場合、専用マスターインスタンスタイプまたはノード数を変更する (専用マスターが 2 つあり、ゾーン認識が有効になっているドメインを除く)
- CloudWatchへのエラーログ、監査ログ、またはスローログの発行の有効化または無効化
- ボリュームサイズを 3 TiB まで増やし、ボリュームタイプ、IOPS、スループットを変更する　※詳細は下記アップデート情報を参照
- タグの追加と削除
- Amazon OpenSearch Service での設定変更

2024/2/14のアップデートでEBSの変更がBlue/Greenデプロイなしで可能になりました！
詳細は以下です。

＜前提条件＞
- 現行世代のインスタンスを使用する OpenSearch Service ドメインであること
- データノードあたりのボリュームサイズが 3 TiB 以内であること
＜変更内容＞
- ボリュームサイズの拡張
- ボリュームタイプの変更
- IOPSの変更
- スループットの変更

上記については、Blue/Greenデプロイなしで設定変更が可能です。
Amazon OpenSearch Service でブルー/グリーンデプロイを使用しないクラスターボリュームの更新が可能に

Blue/Greenデプロイ時の注意点
Blue/Greenデプロイ時は新環境と旧環境が存在するため、ノード数などが 2 倍になります。
※Amazon OpenSearch Service クォータの影響は受けません。
また、Blue/Greenデプロイ中においても検索およびインデックス作成リクエストに対応可能です。
ただし、環境が2つ作成されることによる負荷のため、次のようなパフォーマンスの問題が発生する可能性があります。

- リーダーノードの使用量が一時的に増加
- 検索とインデックス作成のレイテンシーが増加
- クラスターの負荷が増加による、受信リクエストの拒否の増加

パフォーマンスの低下を避けるため、Blue/Greenデプロイはクラスターが健全かつ、ネットワークトラフィックが低い状態で実行することが推奨されています。

さらに、新環境にルーティングを切り替えるさいに瞬断が発生する可能性がある点にも注意が必要です。

ブルー/グリーンデプロイのパフォーマンスへの影響

デプロイ方法はどちらがよいのか
Blue/Greenデプロイなしのデプロイの方がおすすめです。

理由は以下です。

パフォーマンスの低下を防げる
作業時間が短い
基本的にBlue/Greenデプロイありでもなしでも、下記は対応されています。

- 設定変更中においても検索およびインデックス作成リクエストに対応可能
- 設定変更によるドキュメントの欠損はない（※開発環境での検証は必要）

しかし、上記で述べたようにBlue/Greenデプロイは新環境と旧環境の2つ環境が高くなるため、一時的な負荷が高くなってしまいます。
それに比べてBlue/Greenデプロイなしの場合はOpenSearchの負荷状況を大きく気にする必要はありません。（もちろんピーク時に実行は避けた方がよいですが）
また後述の検証でも言及しますが、Blue/Greenデプロイは新環境の作成と、それによるデータ（シャード）のコピーが必要なため設定変更に時間がかかります。

そのため、Blue/Greenデプロイなしのデプロイの方が安全かつ早く設定変更をできるためよいという結論です。

やってみた
実際に2つのデプロイ方法それぞれでOpenSearchクラスターの設定変更をしてみました。
Amazon OpenSearch Service の開始方法

今回検証のために作成したOpenSearchの設定値は以下です。

- マスターノード
台数：3台
インスタンスタイプ：m6g.large.search

- データノード
台数：3台
インスタンスタイプ：r5.large.search
ボリュームタイプ：gp2
ボリュームサイズ：100 GiB

注意
今回の設定値はあくまで検証用のために設定したものです。
本番用に関してはシャードの設計含め入念な設計が必要です。
検討のさいには以下資料もご参考ください。
Amazon OpenSearch Service の運用上のベストプラクティス

ドライランの実行
最初にドライランの実行方法について簡単に紹介します。

1. [クラスター設定]の[編集]をクリックする
クラスター設定.png

2. 適当なパラメータを変更後、[概要]の[ドライラン分析]の実行にチェックを入れ、[ドライラン]をクリックする
設定変更画面.png

3. 画面下に[ドライラン分析が進行中です。]と表示され、数十秒後に結果が表示される。
ドライラン実行中.png

結果画面
BlueGreen必須.png
BlueGreen不要.png
余談
以前本番環境のOpenSearchクラスターに対して、設定変更のデプロイ方法を確認するためコンソール上でドライランを実行したところ、APIが返ってこなく画面上にも変化がないことがありました。
AWSに問い合わせを実施したところ、後日AWSの内部的な不具合であるとの回答が返ってきました。
本番環境にたいして実行したのでめっちゃ焦りました笑笑 こんな不具合あるんですね。（だからコンソールは信用できない・・・）
※ちなみにCLIでドライランを実行する方法もあります。
　update-domain-config

設定変更
OpenSearchのデプロイは以下の順番で実行されます。

1. 初期化
2. 検証
3. 変更の適用

デプロイ状況についてはコンソールで確認することが可能です。

3.に関しては設定変更内容によって表示される項目が異なります。
詳細は以下資料をご確認ください。
　設定変更のステージ

注意

設定変更を実行後、検証に失敗した場合をのぞき変更のキャンセルはできないためご注意ください。
検証に失敗した場合は変更のキャンセル・再試行・編集が可能です。
　Amazon OpenSearch Service の設定変更が、新たな視認性の改善でより簡単に追跡可能に
　検証エラーのトラブルシューティング
もし設定変更中にスタックしてしまった場合はAWS側で解消が必要なため、AWSサポートにご相談ください。しかし解消には時間がかかります（海外部署との連携が必要なため）。

Blue/Greenデプロイの実行
実際にBlue/Greenデプロイで設定変更を実行します。

変更内容は以下です。

データノードのインスタンスタイプの変更
r5.large.search → r6g.large.search
1. [クラスター設定]の[編集]をクリックする
クラスター設定.png

2. [データノード]の[インスタンスタイプ]で"r6g.large.search"を選択する
設定変更.png

3. [概要]の[ドライラン分析]の実行にチェックを入れ、[ドライラン]をクリックする

4. 画面下に[ドライラン分析が進行中です。]と表示され、数十秒後に結果が表示される。
　⇒ブルー/グリーンデプロイ必須と結果が出る
BlueGreen必須.png

5. [変更の実行]をクリックする

6. 上部に"ドメインが正常に変更されました"と表示がされる
変更中画面.png

→青枠の詳細を表示をクリックすることで、デプロイ状況が確認できる
デプロイ状況.png

7. [更新の処理中]が100％となり、変更が完了する
設定変更完了後画面.png

結果
所要時間：30分　（データはほぼ格納されていない状態）
設定変更の項目
新しい環境の作成
新しいノードのプロビジョニング
シャードの新しいノードへのコピー
古いリソースの削除
ノードの入れ替わり：あり
　⇒新環境が作成され、切り替えられたことがかった

Blue/Greenデプロイなしで実行
次にBlue/Greenデプロイなしのデプロイ方法で設定変更を実行します。

変更内容は以下です。

データノードのEBSボリュームタイプの変更
gp2 → gp3
1. [クラスター設定]の[編集]をクリックする
クラスター設定.png

2. [データノード]の[EBSボリュームタイプ]で"汎用SSD-gp3"を選択する
設定変更.png

3. [概要]の[ドライラン分析]の実行にチェックを入れ、[ドライラン]をクリックする

4. 画面下に[ドライラン分析が進行中です。]と表示され、数十秒後に結果が表示される。
　⇒ブルー/グリーンデプロイ不要と結果が出る
BlueGreen不要.png

5. [変更の実行]をクリックする

6. 上部に"ドメインが正常に変更されました"と表示がされる
設定変更中画面.png

　→青枠の詳細を表示をクリックすることで、デプロイ状況が確認できる
デプロイ状況.png

7. [更新の処理中]が100％となり、変更が完了する
設定変更後画面.png

結果
所要時間：5分
設定変更の項目
ボリューム関連の変更の適用
ノードの入れ替わり：なし
　⇒ノードの入れ替わりは発生しておらず、新環境へ切り替えはされていないことがわかる
まとめ
デプロイの状況の詳細は確認することが可能である
Blue/Greenデプロイが発生した場合は、ノードの入れ替わりが発生する
Blue/Greenデプロイとそれ以外のデプロイを比較した結果、所要時間が大幅に異なる
Blue/Greenデプロイは今回はデータがほぼないため30分でしたが、データ量が増えるにつれて所要時間も増えることが見込まれます。
逆にBlue/Greenデプロイ以外のデプロイでは、データ量に依存せず5分程度で設定変更が終わります。（30倍ほどデータが異なる環境で比較検証済み）

最後に
ここまでOpenSearchの設定変更にともなうデプロイ方式について紹介しました。
いかがだったでしょうか？
単純に設定変更をするだけといっても、デプロイ方式によって大きく違いがあることがわかっていただけたと思います。
特に所要時間に対しては、OpenSearchに格納されているデータ量が多ければ多いほど作業に響いてきます。
それを踏まえて、事前のドライランの実行や開発環境での検証が大事ですね。

この記事がお役に立てば幸いです。

以上、ここまでお読みくださりありがとうございました！

Reference: https://qiita.com/Kamibayashi_Mai/items/794e1f42db192f00b377
