---
title: "自宅サーバーでサクッとDokuWikiを立ち上げる方法（RHEL9編）"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Linux", "RHEL", "RHEL9", "Wiki"]
published: true
published_at: 2025-08-21 00:00
---

# 自宅サーバーでサクッとDokuWikiを立ち上げる方法（RHEL9編）

## 1. はじめに

今回は自宅サーバーにDokuWikiをサクッと立ち上げる手順をまとめます。
データベース不要でPHPだけで動くので、Linux初心者でも比較的簡単にセットアップできます。

## 2. DokuWikiとは？

>DokuWikiは、データベースを前提としない、使い易く汎用性の高いオープンソースのウィキソフトウェアです。​きれいで可読性の高い構文は利用者に愛されています。 管理・バックアップ・統合化が容易なところは管理者に好まれます。 DokuWiki はアクセス制御機能と認証への接続機能を内蔵しているので、特に企業環境内での利用に向いています。 活気に満ちたコミュニティから寄与された膨大なプラグインによって、伝統的なウィキ用途を超えた広い範囲の使用方法が可能です。
>https://www.dokuwiki.org/ja:dokuwiki

## 3. インストール手順

### 3.1 サーバー環境

```bash
cat /etc/redhat-release
Red Hat Enterprise Linux release 9.6 (Plow)
```

### 3.2 必要パッケージ

```bash
dnf update
dnf install httpd php
```

### 3.3 ユーザー作成

```bash
useradd -m -d /home/dokuwiki/ -s /usr/sbin/nologin dokuwiki
```

### 3.4 DokuWikiダウンロード

```bash
wget https://download.dokuwiki.org/src/dokuwiki/dokuwiki-stable.tgz
tar xzf dokuwiki-stable.tgz
ls -ld dokuwiki-*
drwxr-xr-x  8 dokuwiki  118    4096  5月 26 22:34 dokuwiki-2025-05-14a
-rw-r--r--. 1 root     root 4255545  5月 26 22:34 dokuwiki-stable.tgz

mv dokuwiki-2025-05-14a/ /home/dokuwiki/public_html/
```

### 3.5 権限変更

```bash
chmod 701 /home/dokuwiki/
chown dokuwiki:apache /home/dokuwiki/
chown -R apache:apache /home/dokuwiki/public_html/
find /home/dokuwiki/public_html/ -type d -exec chmod 750 {} \;
find /home/dokuwiki/public_html/ -type f -exec chmod 640 {} \;
```

### 3.6 セキュリティ無効化

自宅サーバは家庭内LANのみで運用しており、外部から直接アクセスされないため、以下の設定を行っています（SELinuxやfirewalldの設定はご自身のサーバ環境や運用方法に合わせて調整してください）。
```
SELinux:    無効
Firewalled: 無効
```

SELinux無効化
```diff:/etc/selinux/config
 #
 #    grubby --update-kernel ALL --remove-args selinux
 #
-SELINUX=enforcing
+SELINUX=disabled
 # SELINUXTYPE= can take one of these three values:
 #     targeted - Targeted processes are protected,
 #     minimum - Modification of targeted policy. Only selected processes are protected.
```

filewalld無効化
```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```

サーバー再起動
```bash
reboot
```

### 3.7 Apache起動＆設定

Apache起動
```bash
systemctl enable --now httpd.service
```

`/etc/httpd/conf.d/welcome.conf`を開き、内容をすべてコメントアウトします。
ファイルを保存した後にApacheを再起動すると、トップページにWelcomeページは表示されなくなります。
 ```diff:/etc/httpd/conf.d/welcome.conf
 #
 # NOTE: if this file is removed, it will be restored on upgrades.
 #
-<LocationMatch "^/+$">
-    Options -Indexes
-    ErrorDocument 403 /.noindex.html
-</LocationMatch>
-
-<Directory /usr/share/httpd/noindex>
-    AllowOverride None
-    Require all granted
-</Directory>
-
-Alias /.noindex.html /usr/share/httpd/noindex/index.html
-Alias /poweredby.png /usr/share/httpd/icons/apache_pb3.png
-Alias /system_noindex_logo.png /usr/share/httpd/icons/system_noindex_logo.png
+#<LocationMatch "^/+$">
+#    Options -Indexes
+#    ErrorDocument 403 /.noindex.html
+#</LocationMatch>
+#
+#<Directory /usr/share/httpd/noindex>
+#    AllowOverride None
+#    Require all granted
+#</Directory>
+#
+#Alias /.noindex.html /usr/share/httpd/noindex/index.html
+#Alias /poweredby.png /usr/share/httpd/icons/apache_pb3.png
+#Alias /system_noindex_logo.png /usr/share/httpd/icons/system_noindex_logo.png
```

DokuWiki用の設定ファイル作成
```diff:/etc/httpd/conf.d/dokuwiki.conf
+<VirtualHost *:80>
+        CustomLog logs/access.log common
+        ErrorLog logs/error.log
+
+        Alias /dokuwiki /home/dokuwiki/public_html/
+
+        <Directory /home/dokuwiki/>
+                Options +FollowSymLinks -Indexes +MultiViews
+                AllowOverride None
+        </Directory>
+
+        <Directory /home/dokuwiki/public_html/>
+                Options +FollowSymLinks -Indexes +MultiViews
+                AllowOverride All
+                Require all granted
+        </Directory>
+</VirtualHost>
```

設定再読み込み
```bash
httpd -t
systemctl reload httpd.service
```

### 3.9 Webブラウザで初期セットアップ

`http://<サーバーIPアドレス>/dokuwiki/install.php`

セットアップ完了後は`install.php`が不要となるので削除
```bash
rm /home/dokuwiki/public_html/install.php
```

## 参考

DokuWiki マニュアル

https://www.dokuwiki.org/ja:dokuwiki

ディレクティブ一覧 - Apache HTTP サーバ バージョン 2.4

https://httpd.apache.org/docs/current/ja/mod/directives.html