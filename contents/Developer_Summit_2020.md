---
title: 'Developer Summit 2020 1日目行ってきた'

date: '2020-02-14'

tags: ['others', 'devサミ']
---

## [[13-D-2] ぼくらの六十日間戦争 ～ｵﾝﾌﾟﾚからｸﾗｳﾄﾞへの移行～](https://speakerdeck.com/broadleaf/our60dayswar-migrationfromon-premisetocloud)

### ブロードリーフ会社概要

- 自動車アフターマーケット市場で様々なサービスを提供している
- Broad Leaf Cloud Platform
  - 自動車整備・鈑金・車両販売
  - 自動車部品商、リサイクル業　 etc
  - 新サービスは GCP 上にクラウドネイティブで構築している。k8s 使用
  - 旧サービスはオンプレミスで VM、RDB、Windows で運用していた ← 今回はこれをクラウド移行した話

### なぜクラウド移行したか

- HW 管理、ミドルウェアの保守からの解放
- IaC によるプロビジョニング自動化、作業属人化の排除
  - 手順書のメンテナンスができていない、そもそも手順書がないなどの状況だった。
- バックアップ、リストア、クラスタ維持の運用コスト削減
- 拡張性向上、TCO（Total Cost of Ownership）の削減
- 手のかからない状態にしたかった（新規サービスに）体力を割きたかった。

### 移行後のインフラ

- 基盤は AWS（EC2 ベース）
- IaC は Terraform
- AMI 作成は Packer & Ansible
- BtoB サービスでパフォーマンス予測はできるので、自動復旧（オートスケーリング）を目的とする
- DB は RDS を使用

### どうやったか

- 作業は夜間（ユーザー影響を防ぐ）
- DB のバックアップサイズ、S3 の転送時間を加味すると 40 時間くらいかかって間に合わない
  - 大きな DB は Database Mirroring を使用（DMS は使えなかった）
  - 小さな DB はフル BU でリストア
  - ファイルは毎日 FastCopy（無料で一番早い）で同期
  - AWS とオンプレは VPN 接続を行う

### 移行前の検証

- AP の動作検証
- 負荷検証
  - ここは専用のツールを自作し、複数スレッドで高負荷をかける
- DB のフェールオーバの動作検証（DB が FO したとき AP サーバーで接続をキャッシュしないようにする。）

### デプロイ

- デプロイパイプラインを作成する。開発環境はデプロイ後に EC2 がオートスケールし、入れ替えを行う
- 移行は連休を利用して行った。

### 移行後に問題発生

#### RDS の CPU 使用率が爆上がり

- 特定のクエリで CPU 使用率が上がる
  - お金でねじ伏せる →RDS インスタンスの垂直拡張 ← これじゃダメだった。6 日後にまた爆あがり

##### 原因は

- CPU 使用率が高いクエリがあり、実行プランがおかしくなっている。
  - 普通は Hint 句をつけるが、古の芸術的クエリを直すのはリスクが高い
  - クエリストア（SQL Server）を使用して、実行プランを元に戻した。
- 他のクエリもおかしくなっていた

##### 根本原因

- 統計情報の更新をしていなかった
- インデックスが断片化していた

##### 解決

- 実行プランの固定化

  - 検索の文字数で実行プラン変わってしまうクエリがあったので、愚直に一文字づつ固定化した

- 統計情報更新

- インデックス整備

### まとめ

- DB は移行前にきれいにしておく
  - インデックスのリビルドなどは定期的にやっておく
- 全パターンのクエリをテストする必要がある
- 移行時に統計情報更新を忘れずに全部やる
- 1 ヶ月分の Trace を再生する
  - オンプレの本番環境に再生可能な Trace を仕込み
  - テスト用 RDS にフルバックアップを展開し Trace を再生
  - 擬似的に 1 ヶ月分のパフォーマンスを再現テストできる

### 困難を乗り越えて成長するには

- **成長機会は平等ではない**
- **大事なのは積極性**
- **一歩踏み出す勇気**

## [[13-C-3] note の決して止まらないカイゼンを支える、エンジニアリングへの挑戦](https://www.slideshare.net/KonYuichi/note-227802322/KonYuichi/note-227802322)

### note のグロースモデル

- **単一の KPI に絞らない**

  例：投稿数だけを追う（クソ記事量産）、売り上げだけを追う（有料記事のみ量産）

- 要はバランスを大事にし、**勝手に伸びてくサービス**を目指す

  - サービスモデルをアクションでブロック分けし、悪いところを改善していく

### note のチーム

```
リリース頻度高
- 基盤チーム：グロースモデル全体を下支えする。
  - SEO対策、データ基盤、スパム対策 etc...
- 機能開発チーム：サービスのポテンシャルを上げる。短期間（3ヶ月）でメイン機能をリリースする
  - サークル機能、漫画ビューワ、タイムライン改修
- カイゼンチーム：1日〜2週間で、グロースモデルの弱いところをクイックに開発していく
リリース頻度低
```

### note のデータ

定性的にしか見れていなかったので、グロースモデルの状態が把握しづらくなっていた。

- note はサービスが把握しやすい。
- 定量的にどこが弱いのかを把握する必要があった。有効な施策を再現性高くやってきたかった。
  - 定量的に見るために、定義を決める必要がある（創作継続、離脱 etc...）
  - グロースモデルをドリルダウンし、よりモデルを定義的に

### 何をやっているのか

#### データ分析基盤

- データを一元管理する AWS で構築、意思決定陣が見やすいインターフェースを提供する
- データを元に、定義付け → 仮説検証 → 調査 を回す。

**サービスの性質により、指標は異なる**

### これから何をしていくのか

- パフォーマンス課題の解消

  - Nuxt.js、Rails、AWS それぞれにパフォーマンス課題

- レコメンド機能を向上させる

  note はなかなかコンテンツの種類が多く、レコメンドが難しい

  - 適切なチャネルは？
  - セグメントの定義は？
  - 使用する ML の技術は？
  - UI/UX の改善ポイントもある

## [[13-B-4]質とスピード](https://speakerdeck.com/twada/quality-and-speed)

### そもそも質とは

- 狩野モデル

  - 可能品質と当たり前品質

- 内部品質と外部品質 ≒ 非機能要件と機能要件がある

### 内部品質を犠牲にしてスピードをとる

- よく犠牲にされるのは、保守性

- 品質を犠牲にスピードをとるとき、品質 → 内部品質 → 保守性 → テスタビリティ、変更容易性、理解容易性を犠牲にされがち。**これでは現場が疲弊する**

### ではスピードを遅くすれば保守性は上がるのか

- 技術力の低い人が時間をかけてコードを書いても、いいコードが出てくるわけではない
- たしかに、「時間がないから、将来困るけど汚いコード書こう」って意識的にやることはない
- **保守性とスピードは真のトレードオフではない**
  - 「事実は、短期的にも長期的でも、崩壊したコードを書くほうがクリーンなコードを書くよりも常に遅い」
- 品質(保守性)→ スピードの関係は、「保守性が高い*にも関わらず*スピードがはやい」ではなく「_だからこそ_」
- スピード → 品質(保守性)の関係では、現在の市場において仮説検証のプロセスの速さで良いサービスを生み出すこと、が求められる中で、「スピードが遅ければ品質を保てる」は真ではないことは明らか

### 保守性とスピードの損益分岐点は？

- **テスト自動化の損益分岐点は約 4 回**（手動テストと自動テストの損益分岐点）
- **内部品質への投資の損益分岐点は 1 ヶ月以内に現れる。**
  - ここで重要なのは、1 ヶ月であれば、その損益を受けるのは自分たちであること

### 結論

- 「質」VS「スピード」の関係は局所的なものでしかない、**大局的には、質向上とスピード向上はサイクルの関係にある**

- ではどうやって、個人の質を上げるのか
  - 一番大事、かつ一番厄介なスキルは、**システムを設計するための判断力である。**これは、1 からのシステム構築運用経験が一番効果が高い

## [[13-B-5] Best Practices In Implementing Container Image Promotion Pipelines -知っておくべきコンテナイメージ・プロモーションの方法](https://jfrog.com/shownote/developers-summit-2020/)

## 技術の好き嫌い

- 技術を知れば知るほどその技術が好きでなくなっていってしまう。。。

### デプロイについて

- 本当にデファクト通りにやる必要があるのか？
- 今までのやり方と Docker は違う

#### Docker はどう違うのか

- `$ docker build`の存在 → 全部を Docker でやってしまいがち
  - Docker は簡単だが楽すると面倒なことになる
- ワンライナー docker で例えばバージョン指定しないと、勝手に最新バージョンになってしまう。それでは build ごとに docker イメージが同じであると保証できない

#### ではどうする

- ただバージョンを固定するだけでも不十分である。なぜならパッチが当たったらまた違うイメージになってしまう。
- バージョンのハッシュで指定したら？これでは過毒性がおちる。

  - これは、Python、Node.js などでも同じで、バージョン管理から逃げることはできない（ディペンデンシを考える）

- **安定したバイナリで管理すること**→ 一度ビルドしたイメージを各パイプラインフェーズで使う

#### 鉄板のパイプラインを作る

開発環境と本番環境を区別するには

- 案 1：LABEL をつける →typo とか怖い
- 案 2：dockerhub のリポジトリを分ける → プロジェクトごとの構成であり、環境ごとに分けることはできない
- 案 3：環境ごとにレジストリを分ける

#### 大事なこと

- 環境ごとにレジストリを分ける
- 全てのイメージにアクセスしやすくする
- Promotion は素早く行う
- 常に最新のものを使用する

#### どう実装するか

Docker Tag の構造

- 同一ホストに別のレジストリをもつとパニックになる
  - Vertiual hosts を使う

レジストリ間の Promotion をどうする

- jfrog のツールで対応する → 開発者からは各環境も同一のレジストリのように扱える

では最新のバージョンを使いつづけたい場合は？

- jfrog ではメタデータとして管理できる

#### JFrog を使うと

- 最新をシンプルに使える
- 依存関係を自分で管理することができる
- 自分でベースイメージ、インフラ、アプリケーションファイルを管理する → これをツールなど使って

### まとめ

- build は一回だけ
- 環境は分ける
- ビルドしたものをプロモートする
- 自分の依存関係は自分で管理する

## [[13-D-6]テストエンジニアが教える　テストコードを書き始める前に考えるべきテスト](https://speakerdeck.com/nihonbuson/developers-summit-2020)

### テストの目的

テストの 7 原則 ①：テストは欠陥がある、ことしか示せない（悪魔の証明的な）

- 欠陥の検出
- 品質保証
- 意思決定のため
- **欠陥の作り込みの防止** ← 「実装が終わったからテスト」ではない

### 欠陥のつくりこみ防止とは？

- 設計の段階で気になる項目を上げていくことを口頭テストとも言える
  - 要は、ぼぼ要件定義
- テスト内容はいろんな人と話をしたほうがいい

### テストケースはどう作る？

テストの 7 原則 ②：全テストケースを行うことは不可能である。

- テスト項目笑点的思考法
  - 「〜のテストをやるべきです（やるべきではないです。）」
  - 「ほう、それはどうしてだい？」
  - 「〜だからです」

### Checking と Testing

Checking

- 意図通り動くかどうか

Testing

- どうにかして製品を破壊する作業

### QA チームがやりたいこと

- Testing
- テストプロとしてのアドバイス
- 話を聞く、質問（意図を聞く）
  - よく使える質問
    - 「これないがしたいんだっけ」→How から What へ
    - 「ごめん、ちゃんと理解できてないからこの部分もう一度説明してもらっていい？」→ 開発者が自分で気付けなかったことに気づくことができる。

### 大事なこと

- テスト作業ではなく、**テスト活動をする**！

## [13-C-7]InterSystems IRIS Data Platform で高度なデータ分析のための基盤を整備しよう

### データ永続化の歴史

- ガバナンスや、コンプライアンスからデータレイクの考え方が生まれた。
- データレイクは技術要素というよりベストプラクティス

### 階層型 DB の勧め

- IRIS Data Platform はいいぞ！
- スケーラブルなプラットフォームを提供している。

### データ分析を行うためには

- トレーサビリティの高いツールを使う。
- バグデータが混入した際にその混入原因まで調査する。

## [13-D-7]エンジニアはものづくりの夢を見るか - AWS Loft Tokyo 入館アプリの開発事例 -

※途中から参加

### ポイント

- 開発したいモチベーションをどこに置くか、が大事
- **締め切り駆動開発！！**

### 使用技術

- AWS Amplify
- AWS Amplify Framework
  - Amplify for ~ :AWS バックエンドと簡単に統合できるクライアントライブラリ
  - UI Component:主要フレームワークに対応して、よくある UI をよしなに作ってくれる
- AWS Chalice：Python でサーバレスアプリを迅速に作るための OSS フレームワーク

### これからのお話(運用と改善)

- 空席情報わかるようにしたい
- 本人確認書類提出をやめたい

## [[13-E-8]チームをつくるモブプログラミング ～内側と外側から語る～](https://speakerdeck.com/yattom/mob-programming-to-build-teams)

### モブプロを始める

始めるときは。。。

- モビングを実験とする
- 行番号 ON
- 起こらない etc...

#### 経験から得られたコツ

- 目的とゴールを定める・・・長期的なゴールと、局所的なセッションのゴールを決める。このとき、「目的はこれ。いいですか？」をリーダーが主導してはいけない。

- 進み方を共有する・・・ToDo リスト作成、作業の区切りを確認しながら進む。 **ドライバーが常にぶつぶつする**

- その場でフィードバックを得る・・・**後で一人でテストしたり、リファクタしたりはだめ**

- 振り返りをする・・・ただ、一人復習の方が集中できる場合もあるので、自由

> Q. モブプロ始めるとき周りをどう説得するか
>
> A. 試験的にやってみる。外部の人や、経験者のお墨付きを得る。
>
> Q. 「一人で作業したい」と言われたら。
>
> A. とりあえずやってみてもらう！！別に全員でやる必要もない。

### チームをブーストするモブプロ

##### 目的

- 教育的強化
- チームの生産性強化

##### スキルトランスファー

- 経験値が低い人をドライバにする・・・経験のある人がガリガリ進めると、周りが気後れしまう。
- 仕事を止めることができる権利を渡す・・・知識のジャストインタイム
- 聴きながら手を動かして学ぶ
- 参加者全員で共有できる

ここで、暗黙知（コードの組み立て方、ツールの便利機能、What/Why/How）も共有すること

##### レビュー

レビューおじさん問題（スキルアップして、開発したいのにレビュアーになっちゃう）を救う。
レビューの目的は、**検査、学習、強化**。モブプロは**検査、学習**をカバーする。

- リアルタイムレビュー
- タイポは親の仇のように指摘してくれる
- チーム全員の合意をもとにコードかける
- 忖度せずにチーム全員でベストなコードを目指すことができる

レビューと組み合わせて 3 要素全て補填していく。

Re-View（再び観る）の意味を考える。普通だと、レビュアーは First-View。開発時の目線と冷静な目線両方持つことができる。

> Q. 喧嘩にならない？
>
> A. 建設的な議論であれば OK。絶対に強い言葉を使わないことが大事。無意味
>
> Q. 「わからないです」って言えない人出てきちゃうのでは？
>
> A. ドライバを任せる、期間区切って振り返りの時間をこまめに取る。

### 仕事をドライブするモブプロ

#### モブプロの真価とは

- 全員の知識を生かせる
- 個々の総和以上の成果
- エンゲージメントを引き出す
- オーバーヘッドが最小になる。

####モブプロが活きない場面

- みんな知っていること
- 簡単な問題の積み重ね
- 単純作業、ルーチンワーク

#### モブプロの目的にあった方式を使う

- 仕事のドライブ or チームをブースト

回数重ねてうまく回り始めると、両方ハイパフォーマンスを実現する方向へ収束していく

**あとは結局、心理的安全が大事**。心理的安全が担保されていると、ほどよい学習状態（プレッシャーと能力のバランスが取れている状態）を作り出すことができる

> Q. 仕事を分担してやると、生産性が落ちちゃうんじゃない？
>
> A. 短期的生産性 UP、単純作業、ルーチンワークが目的の場合はそう。あとは人数が多すぎてもあまり意味がない。中長期的に見て生産性を上げる戦術として使うもの
>
> Q. ほどよい学習状態に身を置くための工夫とは？
>
> A. 新しい技術をモブプロ題材にするなど
>
> Q. モブプロがうまく行っているチームとは
>
> A. 全員が均等に発言している。楽しそうにしている。偉い人がそのチームをほっといてくれている状態。

## モブプロの全体性

- チームを、内側の向上 → 外側の向上 →...と拡張させていくと、最終、会社の話になってくる。
- インテグラル理論、タックマン理論
- 決して **モブプロをやらないといけないわけではない** 。→「一緒に行動すること」ではなく「一緒に考えること」が重要