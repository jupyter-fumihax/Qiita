---
title: LTIカスタムパラメータによる Moodle + JupyterHub 連携システムの構築と運用（２）「Linux と Moodle のインストールと設定」
tags:
  - Docker
  - 教育
  - Moodle
  - JupyterHub
  - LTI
private: false
updated_at: '2025-12-27T14:01:49+09:00'
id: e5fc9a7c7dcd4740e3fb
organization_url_name: null
slide: false
ignorePublish: false
---
###### Moodle ＋ JupyterHub 連携システム シリーズ（２）

## 2. Linux と Moodle のインストールと設定

本記事では，次のいずれかの構成で Linux を最小構成インストールした環境を前提とします．OS は Rocky Linux 9 または Ubuntu Server 24 を用います．

- １台構成の場合は，１台の PC 上に Moodle と JupyterHub の両方をインストールします
- ２台構成の場合は，１台の PC に Moodle を，もう１台の PC に JupyterHub をインストールします


###### コマンド表記
なお各コマンドは **root** ユーザにスイッチ（**`su -`**）した状態でのコマンドです．**`sudo`** を使用する場合は，各コマンドの前に **`sudo`** を追加してください．
またコマンドの前に **$** がある場合は，一般ユーザで実行するコマンドであることを示します．

### 2.1 Linux のインストール・設定

ここでは，Linux 環境の設定について，ポイントを絞って説明します．基本的なインストール手順そのものは，他のサイトに詳しい解説が多数ありますので，本記事では扱いません．既に Linux の最小構成インストールが完了し，ログインできる状態になっていることを前提に話を進めます．

#### 2.1.1 Rocky Linux 9
###### 最小開発環境
Rocky Linux 9 の最小構成では，本当に何もコマンドがありませんので，必要最低限の開発環境をインストールします．
https://polaris.star-dust.jp/Linux/setup/rocky9-mini-devel.sh にスクリプトを用意しましたので，それを利用します．

```bash
dnf -y install wget
wget https://polaris.star-dust.jp/Linux/setup/rocky9-mini-devel.sh
cat  rocky9-mini-devel.sh
bash rocky9-mini-devel.sh
```

##### SELinux
好みにもよりますが，**SELinux** が有効だと後々設定が面倒になります．ここでは SELinux を **permissive** にします．完全停止（**disabled**）でも構いません．permissive と disabled の違いは，どちらも SELinux を無効にしますが，permissive は監視用のログを残します．

```bash
vi /etc/sysconfig/selinux         # SELINUX=permissive にする
systemctl reboot
```

---

#### 2.1.2 Ubuntu Server 24
###### 最小開発環境
最新版の Ubuntu Server 25 もありますが，今回は安定性を考慮して Ubuntu Server 24（LTS）を使用します．
最小構成だと入っていないソフトもあるので，必要最低限の開発環境をインストールします．
https://polaris.star-dust.jp/Linux/setup/ubuntu24-mini-devel.sh にスクリプトを用意しましたので，それを利用します．
```bash
wget https://polaris.star-dust.jp/Linux/setup/ubuntu24-mini-devel.sh
cat  ubuntu24-mini-devel.sh
bash ubuntu24-mini-devel.sh
```

---

### 2.2 ユーザ作成と設定
LTIContainerSpawner のデフォルトでは，ユーザのホームディレクトリを **/home/グループ名/ユーザ名** とします（変更は可能）．一方多くの Linux ディストリビューションでは，ユーザのホームディレクトリを **/home/ユーザ名** としますので，注意が必要です．
コマンド **usermod** でホームディレクトリやグループを変更することも可能ですが，ログインしているユーザは変更できない などの制約がありますので，ここでは新しいユーザ **alice** をグループ **users** で作成します．

```bash
mkdir /home/users
chgrp users /home/users
chmod a+rx /home/users
useradd alice -g users -m -d /home/users/alice -s /bin/bash
passwd alice         # パスワードを設定
```

これも好みに寄りますが，新しいユーザ **alice** のユーザ環境を整えます．自分でシェルの設定ファイルが書けるのであればそれが一番いいのですが，ここではシェルとして **bash** (B-Shell) を使用することを前提として，設定ファイルを polaris.star-dust.jp からダウンロードします．

alice でログイン後，以下のコマンドを実行します．ダウンロードした **.bashrc**, **.bash_profile**, **.exrc** は必ず中身を確認し，意に沿わない設定は修正しましょう!
なお Ubuntu では ~/.bash_profile が存在する場合，~/.profile は無視されます．
```bash
$ cd
$ rm -f .bashrc
$ rm -f .bash_profile
$ wget https://polaris.star-dust.jp/Linux/setup/.bashrc 
$ wget https://polaris.star-dust.jp/Linux/setup/.bash_profile
$ wget https://polaris.star-dust.jp/Linux/setup/.exrc
$ cp .exrc .vimrc
$ . .bash_profile        # 設定の反映
```
**`su -`** で root にスイッチする環境（sudoを使用しない環境）では，root 環境でも上記のコマンドを実行します．

---

### 2.3 Moodle 5 の（簡易）インストール
本システムでは，学習管理システムとして Moodle を用います．
執筆時点では Moodle v5系の安定版を想定しますが，4系であってもほぼ同様に動作するはずです．

ここでは **Moodle 5** の **簡単なインストール例**を示します．著者の好みの設定による**インストール例**です．
**あくまでも例です** ので，自分の環境に合わない場合は自分の環境に合わせて変更するか，そもそも設定の仕方が自分の好みに合わないなどの場合は，他の解説記事や公式ドキュメントを参考に自分で設定してください（ちょっとくどい言い回しです）．

また，各コマンドの詳細な説明や Moodle 自体の詳細な説明・設定についてはここでは解説しません（膨大な量になるので）．必要な場合は各自で別途調べてください．

#### 2.3.1 LAMP 環境
以下に **インストール例** として使用するLAMP環境を示します．
- **Linux** : ここでの **2.1**, **2.2** までの設定を終了した **Rocky Linux 9**
- **Apache** : Rocky Linux 9 の標準パッケージの **httpd-2.3**
- **MariaDB** : Rocky Linux 9 の標準パッケージの **mariadb-10.11**
- **PHP** : Rocky Linux 9 用の **remi リポジトリ** の **PHP-8.3**

#### 2.3.2 Apache のインストールと設定
###### Apache のインストール
まず Rocky Linux 9 に apache-2.3 をインストールします．

```bash
dnf -y install httpd
```
設定は，Apache の設定ファイル **/etc/httpd/conf/httpd.conf** を **例えば** 以下の様な内容になるように書き換えたり，追加を行います．

###### 設定の変更
```apache
ServerAdmin fumihax@star-dust.jp
ServerName polaris.star-dust.jp:80
DocumentRoot "/var/www/html"

<Directory "/var/www/html">
   Options FollowSymLinks MultiViews      # Indexes オプションは必ず外しましょう．
   AllowOverride All                      # 設定に柔軟性を持たせるため．None でもいい．
   Require all granted
</Directory>
```

###### 設定の追加

```apache
<IfModule dir_module>
    DirectoryIndex index.html  index.php       # index.php を追加
</IfModule>
```

```apache
AddType application/x-httpd-php .php
```

###### ドキュメントルートの設定
デフォルトのドキュメントルート **/var/www/html** は既に存在しますので，**/var/www** 以下の持ち主と **/var/www/html** のパーミッションを変更ます．
```bash
chown apache:apache /var/www
chown apache:apache /var/www/html
chmod g+rwxs,o-rwx  /var/www/html
```


#### 2.3.3 Apache の SSL モジュールのインストールと設定
###### SSL モジュールのインストール
HTTPS 通信のために apache の SSLモジュールをインストールします．
```bash
dnf -y install mod_ssl
```

###### 設定の変更
SSLモジュールの設定ファイル **/etc/httpd/conf.d/ssl.conf** を **例えば** 以下の様な内容になるように書き換えます．

```apache
DocumentRoot "/var/www/html_ssl"
ServerName polaris.star-dust.jp:443
```

###### 設定の追加
```apache
<Directory "/var/www/html_ssl">
    Options FollowSymLinks MultiViews
    AllowOverride All
    Require all granted
</Directory>
```
###### HTTPS 用のサーバ証明書
次に **サーバ証明書** を用意します．本番環境ではお金を出して買うか，**Let's Encrypt** を使用しますが，組織内部だけの使用やテスト使用では**オレオレ証明書**でも十分です．

オレオレ証明書は，例えば以下のように作成します．
```bash
openssl req -x509 -newkey rsa:2048 -sha256 -days 365 -nodes \
  -keyout /etc/pki/tls/private/mdl_test.key \
  -out    /etc/pki/tls/certs/mdl_test.crt \
  -subj "/CN=localhost"
```

上記で作成した秘密鍵と証明書のファイルを ssl.conf の該当箇所に設定します．

```apache
SSLCertificateFile    /etc/pki/tls/certs/mdl_test.crt
SSLCertificateKeyFile /etc/pki/tls/private/mdl_test.key
```

###### HTTPS　用のドキュメントルートの設定
HTTPS用のドキュメントルート **/var/www/html_ssl** を作成し，**/var/www** 以下の持ち主とパーミッションを変更ます．
```bash
mkdir /var/www/html_ssl
chown apache:apache /var/www/html_ssl
chmod g+rwxs,o-rwx  /var/www/html_ssl
```


#### 2.3.4 Apache の実効ユーザのホームディレクトリの変更

一般的な構成ではあまり行われませんが，本システムを動作させるうえで **重要な設定** があります．
本システムでは，Apache の実効ユーザ（Rocky Linux では apache）のホームディレクトリに対して，**apache 自身が書き込み可能である必要があります**．

これは，Moodle 上のモジュールが JupyterHub 上のコンテナ管理機構と連携する際に，**SSH ポートフォワードを用いた制御処理を行うため**です．この処理では，Apache の実効ユーザが一時的な制御情報や鍵情報を保持できる作業領域を必要とします．

しかし，Rocky Linux の標準設定では，apache ユーザのホームディレクトリは /usr/share/httpd に設定されていますが，このディレクトリは root 所有であり，apache ユーザは書き込みを行うことができません．

そこで，本システムでは apache 用に新たに **/var/lib/apache** ディレクトリを作成し，これを apache ユーザのホームディレクトリとして設定します．
この構成により，Web 公開ディレクトリ（/var/www）とは分離された安全な作業領域を確保しつつ，本システムに必要な制御処理を安定して行うことができます．

```bash
mkdir -p /var/lib/apache
chown apache:apache /var/lib/apache
chmod u+rwx,go-rwx /var/lib/apache
usermod -d /var/lib/apache apache
```


#### 2.3.5 Apache の起動と常時起動の設定
まず firewalld が動いている場合は停止させるか，TCPの 443番ポートの解放を行います．
```bash
systemctl stop firewalld 
```
または
```bash
firewall-cmd --add-port=443/tcp --permanent
firewall-cmd --reload
```

次に Apache を起動してみて 問題無く起動する場合は，常時起動するようにします．

```bash
systemctl start httpd
systemctl enable httpd
```

簡単な **/var/www/html_ssl/index.html** を作成し，Web ブラウザで **`https://サーバ名 (or IPアドレス)/`** にアクセスします．**警告のページが表示されますが，無視して接続** した場合に index.html が正確に表示されるか確認します． 

---

#### 2.3.6 MariaDB のインストールと設定
mariadb をインストールし，そのまま起動します．問題無く起動できた場合は，常時起動に設定します．
```bash
dnf -y module install mariadb
systemctl start mariadb
systemctl enable mariadb
```

次に MariaDB に管理者のパスワードを設定します．設定が完了したら，そのパスワードでログイン可能かどうか確認します．
```bash
mariadb-admin -u root password *****     # パスワードを設定
mariadb -u root -p
Enter password:                          # 設定したパスワードを入力

MariaDB [(none)]>                        # ログインに成功すると，MariaDB のプロンプトが表示される 
```

ここで，あらかじめ Moodle 用の DB を作成しておきます．ここではデータベース名を **moodle_db**, アクセス用ユーザ名を **moodle_user**, moodle_user のパスワードを **moodle_pass** とします．

```mysql
MariaDB [(none)]> create database moodle_db default character set utf8mb4 collate utf8mb4_unicode_ci;
MariaDB [(none)]> grant all on moodle_db.* to  moodle_user identified by 'moodle_pass';
MariaDB [(none)]> flush privileges;
MariaDB [(none)]> exit
```

---

#### 2.3.7 PHP のインストールと設定
Moodle 5 は v8.2 以上の PHP を要求します．ここでは **remi リポジトリ** を利用して **8.3** をインストールします．
```bash
dnf -y install https://rpms.remirepo.net/enterprise/remi-release-9.rpm
dnf module reset php
dnf -y module install php:remi-8.3
dnf -y install php-devel
php --version
```

つぎに，Moodle に必要な PHP の拡張機能をインストールします．
```bash
dnf -y install php-zip
dnf -y install php-mysqlnd
dnf -y install php-sodium
dnf -y install php-gd php-intl php-soap php-opcache
```

最後に，Moodle 用に PHP の最低限の設定を行います．

**/etc/php.ini** をエディタで編集し，**max_input_vars** を少なくとも **5000** 以上にします．

```bash
vi /etc/php.ini           # 例) max_input_vars = 10000
```

#### 2.3.8 php-fpm と httpd の再起動
全てのインストールと設定が終わったら，PHP のプロセスマネージャ **php-fpm** と Apache サーバ **httpd** を再起動します．
```bash
systemctl restart php-fpm httpd
```

---

#### 2.3.9 Moodle のダウンロードと設定

2025 12/19時点での最新バージョン 5.1.1+ (LTS) をダウンロードして展開します．
```bash
cd /var/www
wget https://download.moodle.org/download.php/direct/stable501/moodle-latest-501.tgz
tar xzfv moodle-latest-501.tgz
mv moodle moodle-5.1.1+                      # 色々なバージョンを試すことがあるので，名前を変更
chown -R apache:apache moodle-5.1.1+/
rm  moodle-latest-501.tgz 
cd html_ssl
ln -s ../moodle-5.1.1+/public  moodle        # リンク先が public ディレクトリであることに注意
ls -l                                        # 確認
```
Moodle 5 以降では，従来は意識する必要のなかった **vendor ディレクトリ** を，明示的に作成・管理する場面が増えています．ここでは，この vendor ディレクトリを作成します．
```bash
cd /var/www/moodle-5.1.1+
dnf -y install composer
export COMPOSER_ALLOW_SUPERUSER=1
composer install --no-dev --prefer-dist --optimize-autoloader --classmap-authoritative --no-interaction
chown -R apache:apache vendor/
systemctl restart php-fpm httpd
```

#### 2.3.10 Moodle の初期設定

Webブラウザで **`https://サーバ名 (or IPアドレス)/moodle/`** にアクセスし，後は画面の指示に従って作業する．
途中選択する データベースドライバ には **MariaDB** を指定します．
またデータベース名，データベースユーザ，データベースパスワードの入力ではそれぞれ，**2.3.4** の **`grant` コマンド** で指定した **moodle_db**, **moodle_user**, **moodle_pass** を指定します．

最後に 自動で作成された **/var/www/moodledata** のパーミッションが，**rwxrwxrwx** と恐ろしいことになっている筈なので，これを修正します．

```bash
cd /var/www
chown -R apache:apache moodledata
chmod -R o-rwx moodledata
ls -ld moodledata                   # 確認
```

---
> シリーズ「LTI カスタムパラメータによる Moodle＋JupyterHub 連携システムの構築と運用」第２回  
> [第１回に戻る](https://qiita.com/fumi-hax/items/95c0535cf18dcaeaadf4) ｜ [次へ：JupyterHub (LTIContainerSpawner) のインストールと設定](https://qiita.com/fumi-hax/items/0ac199a514af9fa4d269)
