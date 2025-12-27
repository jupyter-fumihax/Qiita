---
title: LTIカスタムパラメータによる Moodle + JupyterHub 連携システムの構築と運用
tags:
  - Docker
  - 教育
  - Moodle
  - JupyterHub
  - LTI
private: false
updated_at: '2025-12-26T02:46:25+09:00'
id: 95c0535cf18dcaeaadf4
organization_url_name: null
slide: false
ignorePublish: false
---
###### Moodle ＋ JupyterHub 連携システム シリーズ（１）

## 0．はじめに
本記事では **Moodle** と **JupyterHub** を **LTI** を用いて連携し，**SSO にとどまらず，LTI カスタムパラメータを活用した動的制御**により，受講者が最小限の操作で演習環境に入れる基盤の構築・運用方法を紹介します．説明では，授業利用を想定した **小規模オンプレミス環境** を例にしていますが，構成自体はクラウド環境にも応用可能です．

対象読者は，Linux の基本操作や Web サービスの運用にある程度馴染みがあり，Moodle や JupyterHub を用いた演習環境の整備・運用に関心を持つ方を想定しています．本記事では，環境構築の細部を網羅することよりも，**最小構成で確実に動作する設計と，実運用で得られた知見**に重点を置いて解説します．

記事中の設定例は，実際の授業運用で確認した構成をもとにした **最小構成の一例** です．組織のポリシや運用要件に応じて，適宜調整しながら利用してください．


## 1．概要

### 1.1 解説
本記事では **Moodle** と **JupyterHub** を **LTI** で連携し，**SSO に加えて LTI カスタムパラメータの受け渡しによる動的制御**を行う構成を扱います．これにより，受講者ごとに隔離された **JupyterLab 環境を最小限の操作で起動できる演習基盤** を構築できます．

本システムの設計は Moodle 固有のものではなく，Open edX，Sakai，ILIAS，RELATE（Python 製軽量 LMS）など，LTI を実装した他の LMS へも展開可能なアーキテクチャです．ただし，現時点で公開しているモジュールは Moodle 向けのみです．

実運用では，Moodle サーバと JupyterHub サーバを **別々のミドルレンジ構成の PC に分離**した構成で，**40～50 名程度の同時利用**においても安定して動作することを確認しています（※1）．本記事では，オンプレミス環境における **単一 JupyterHub 運用** を前提に解説しますが，必要に応じて **1 台の Moodle から複数の JupyterHub を制御する構成** も可能です．

>**※1： 実運用時の最低構成例**
>- Moodle 側：AMD Ryzen 7 2700X（8 Core），RAM 32GB，Rocky Linux 8（過去の運用時のOSバージョン）
>- JupyterHub 側：AMD Ryzen 9 5900X（12 Core），RAM 64GB，Rocky Linux 8（過去の運用時のOSバージョン）

---

当面は **LTI 1.1** による連携を前提とします．Moodle 側では **mod_lticontainer** モジュールが **LTI カスタムパラメータ**の設定を支援し，課題やコース情報を LTI Launch とともに JupyterHub へ送信します．JupyterHub 側では **LTIContainerSpawner** がこれらのパラメータを解釈し，コンテナイメージの選択，リソース制限，ボリューム割り当てなどを自動的に行ったうえでコンテナを起動します．

さらに，**configurable-http-proxy（CHP）** の代替として **ltictr_proxy** を組み合わせることで，Notebook のコードセル実行結果に基づく正答情報を収集し，Moodle 側へ可視化用データとして返す構成も選択できます（オプション）．

これらの連携により，受講者は端末ごとの環境構築から解放され，授業開始直後から **同一の実行環境で演習に入ることが可能**になります．また，教室外からの予習・復習についても，ブラウザのみで対応できます．

教員側にとっては，環境差異やライブラリ依存関係に起因するトラブル対応を最小限に抑えつつ，授業設計や教材作成に集中できる点が利点です．
コンテナによる隔離により安全性と安定性を確保し，多言語環境，CPU 数，メモリ上限，実行時間制限，作業領域などを **課題単位で柔軟に切り替える運用**が可能です．付属ツールを用いれば，教材の配布・回収・採点も運用に沿って一元化できます．

## 1.2 システムの構成

### 1.2.1 本シリーズでの動作確認環境（※2）

本シリーズで解説する構成は，以下の環境において動作確認を行っています．いずれも最小構成での検証であり，大規模環境やクラウド構成を前提としたものではありません．

* **OS**： **Rocky Linux 9** または **Ubuntu Server 24**（各最小構成）
* **Moodle**： 執筆時点での最新安定版
* **Python**： 3.11
* **JupyterHub**： 4.x
* **configurable-http-proxy（CHP）**： 4.x

ただし，本構成の原型となる 旧バージョンでは，**Moodle サーバ 1 台，JupyterHub サーバ 5 台（うち 1 台は予備）** による構成で，**約 250 名規模の同時利用を想定した授業運用を，3 年間にわたり継続的に実施** してきました．本シリーズでは，これらの運用実績を踏まえつつ，まずは再現性の高い最小構成に焦点を当てて解説します．

なお，過去の授業運用では比較的高性能なサーバを用いていましたが，**本シリーズで紹介する構成は，それらの運用経験を踏まえたミドルレンジのサーバ構成** を前提としています．


>**※2：補足**
>* **Docker** または **Podman** が動作する環境であれば，記載以外の Linux ディストリビューションでも基本的に動作します
>* Moodle についても，必ずしも最新版である必要はありません
>* 本記事では，Python は **運用実績のある安定版である 3.11** を使用します．Python 3.12 については，**本記事で扱う構成としての体系的な検証は行っておらず**，関連ライブラリの対応状況を踏まえ，今後の検証課題とします．

---

### 1.2.2 本シリーズで紹介する OSS（著者らによる開発）

本システムは，既存の OSS を組み合わせるだけでなく，以下のコンポーネントを新たに開発・拡張することで実現しています．

* **mod_lticontainer**
  [https://github.com/moodle-fumihax/mod_lticontainer](https://github.com/moodle-fumihax/mod_lticontainer)
  Moodle 用の LTI 連携モジュール．LTI カスタムパラメータの設定を支援します．

* **LTIContainerSpawner**
  [https://github.com/jupyter-fumihax/lticontainerspawner](https://github.com/jupyter-fumihax/lticontainerspawner)
  JupyterHub の DockerSpawner を継承した Spawner クラス．
  LTI カスタムパラメータを解釈し，コンテナイメージの選択，リソース制限，ボリューム割り当てなどを自動化します．

* **Jnotice**
  [https://pypi.org/project/jnotice/](https://pypi.org/project/jnotice/)
  LTIContainerSpawner 向けの JupyterLab 補助プラグイン．
  ファイルで用意した固定メッセージを JupyterLab 上に表示できます．

* **JupyterLab-Broccoli**
  [https://github.com/jupyter-fumihax/jupyterlab-broccoli](https://github.com/jupyter-fumihax/jupyterlab-broccoli)
  JupyterLab 上で動作するブロック型ビジュアルプログラミング環境．
  [JupyterLab-Blockly](https://github.com/QuantStack/jupyterlab-blockly) をベースに改良しています．

* **JupyterLab-Broccoli-Turtle**
  [https://github.com/jupyter-fumihax/jupyterlab-broccoli-turtle](https://github.com/jupyter-fumihax/jupyterlab-broccoli-turtle)
  JupyterLab-Broccoli 上で動作するタートルグラフィックス環境です

>![Fig.0.1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3959968/78def515-b04e-4381-9287-7132c957d0b1.png)
**JupyterLab-Broccoli-Turtle の画面**

---

### 1.2.3 システム構成の概略

以下に本システムの全体構成の概略を示します．

>![Fig.5.2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3959968/e16a7d78-5234-403c-aee9-fc8f2d9aa1cb.png)
**システムの全体像の概略**

本構成で新たに実装・導入した主な要素は，下記のとおりです．

* Moodle 用モジュール **mod_lticontainer**
* JupyterHub の Spawner クラス **LTIContainerSpawner**
* CHP の代替 Proxy である **ltictr_proxy**（オプション）
* コンテナ初期化用スクリプト **start.sh**

**LTIContainerSpawner** は JupyterHub の DockerSpawner を継承しており，**Docker** および **Podman** のいずれでも利用可能です．

mod_lticontainer は，主として **LTI カスタムパラメータの設定** を支援します．ここで設定されたパラメータは，SSO を伴う LTI Launch とともに JupyterHub へ送信され，後段で LTIContainerSpawner が解釈します．LTIContainerSpawner は受信した情報に基づいて，コンテナイメージの選択，リソース制限，ボリューム割り当てなどを決定し，起動後はコンテナ内で **start.sh** が各種パラメータを読み込んで Jupyter 環境の初期化を行います．

学習状況データの収集については，オプションとして **ltictr_proxy** を利用できます．これは，ユーザのブラウザとコンテナ間の WebSocket 通信に介在し，コードセル実行結果（正常終了またはエラー）を解析し，REST を介して mod_lticontainer に通知します．現バージョンでは，収集対象は実行結果の種別に限定しています．
なお，本機能は将来的に **LTI 1.3** を用いた構成に置き換えられる可能性があります．

**SSH ポートフォワーディング** は，mod_lticontainer が JupyterHub サーバ上のコンテナ環境（コンテナイメージ一覧など）の情報を取得するために使用します．この機能についても，今後は LTI 1.3 を利用した構成に統合される可能性があります．



---

## シリーズ総目次

本ページは **「0．はじめに／1．概要」** です．以降は章ごとに別記事として公開します．

* **2．Linux と Moodle のインストール・設定** … [記事はこちら](https://qiita.com/fumi-hax/items/e5fc9a7c7dcd4740e3fb)
* **3．JupyterHub（LTIContainerSpawner）のインストールと設定** … [記事はこちら](https://qiita.com/fumi-hax/items/0ac199a514af9fa4d269)
* **4．Moodle モジュール mod_lticontainer のインストールと設定** … [記事はこちら](https://qiita.com/fumi-hax/items/99db197bddd0959c173c)
* **5．Moodle と JupyterHub の連携** … [記事はこちら](https://qiita.com/fumi-hax/items/3d00c8d58a5ad33120fb)
* **6．運用方法** … **`[書きかけ]`**
* **7．コンテナイメージ** … [記事はこちら](https://qiita.com/fumi-hax/items/ff28167133ffbfc6f73d)
* **8．応用（オプション機能，大規模運用，NFS）** … **`[書きかけ]`**
* **9．トラブルシューティング** … [記事はこちら](https://qiita.com/fumi-hax/items/ff3e9d94589f626b8ee3)

---
## 参考文献・URL
- 「LTI カスタムパラメータによる Moodle - JupyterHub 連携に関する研究」 井関 文一, 浦野 真典, 日本ムードル協会全国大会発表論文集10 pp.20-25, 2022年7月22日, https://moodlejapan.org/pluginfile.php/17442/mod_resource/content/1/proceedings2022.pdf
- GitHub Wiki： https://github.com/jupyter-fumihax/lticontainerspawner/wiki/Moodle-&-JupyterHub-(J)
- 「Moodle-JupyterHub連携システムの拡張・改善について」 井関 文一, 川勝 英史, 日本教育工学会 2025 年春季全国大会 講演論文集 pp.565-566, 2025年3月8日 

---
> シリーズ「LTI カスタムパラメータによる Moodle＋JupyterHub 連携システムの構築と運用」第１回  
> [次へ：Linux と Moodle のインストールと設定](https://qiita.com/fumi-hax/items/e5fc9a7c7dcd4740e3fb)
