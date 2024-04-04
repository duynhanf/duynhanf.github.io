---
layout: post
title: Những típ vẽ sơ đồ AWS bằng draw.io
permalink: /aws/tip-su-dung-draw.io-de-ve-so-do-aws/
tags: aws draw.io tips
---

<!-- 案件でAWSの構成図を作成する機会があったので備忘兼ねて投稿します。 -->
<br>
※Thời gian đọc 5 phút

<h2>1. Tạo những groups từ trong ra ngoài</h2>

Cấu trúc nhóm cơ bản của AWS như sau:
![draw.io](https://camo.qiitausercontent.com/0c92df42c6f1ab19403036a16eb43efb25cf4ee6/68747470733a2f2f71696974612d696d6167652d73746f72652e73332e61702d6e6f727468656173742d312e616d617a6f6e6177732e636f6d2f302f333734343733302f39636633393030612d623463342d663561332d636135622d6137366430393733663066342e706e67)

Theo ý kiến cá nhân, tôi recommend tạo theo thứ tự sau Public subnet or Private subnet > Availability Aone > VPC > Region > AWS Cloud. Lý do bởi vì 
<!-- 添付の場合、個人的には
Public subnet or Private subnet > Availability Aone > VPC > Region > AWS Cloudの順番で作成することをオススメします。理由は内側のグループが肥大すると外側のグループの手直しが発生するためです。
今回作成した時に外側から作成してしまい、めっちゃ時間がかかってしまいました... -->

<!-- <h2>2. グループの左上を掴む</h2> -->
<h2>2. グループの左上を掴む</h2>

<p>日本語が下手ですみません。なぜ左上を掴まないといけないか？試しにPublic subnetをクリックしてドラッグをすると、添付の様になりました。</p>

![draw.io](https://camo.qiitausercontent.com/1502d6eed18c401f51790249d7bae96d16dfddaf/68747470733a2f2f71696974612d696d6167652d73746f72652e73332e61702d6e6f727468656173742d312e616d617a6f6e6177732e636f6d2f302f333734343733302f30343030313165382d616637662d356365332d646639372d6265343130653763343630652e706e67)
<p>クリックをするとグループの外から選択されてしまうため、選択したグループ内に存在するグループも一緒に動いてしまうのです。複数選択をする時も左上をクリックしないと複数選択が解除されます。そのため、グループの左上、つまり添付の赤枠をクリックしてドラッグすることで対象のグループのみ動かすことができますし、複数選択も対応できる様になります。</p>
