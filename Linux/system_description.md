# Linuxのシステム強化
Linuxのシステム強化技術について、その技術とコマンドの使い方などについて備忘録として記述する。    
> 情報源：TryHackMe [***Linux System Hardening***](https://tryhackme.com/room/linuxsystemhardening)

## 目次
- [物理セキュリティ(GRUB)](#物理セキュリティgrub)
- [ファイルシステムのパーティション分割と暗号化](#ファイルシステムのパーティション分割と暗号化)
- [リモートアクセス](#リモートアクセス)
- [ユーザアカウントの保護](#ユーザアカウントの保護)
- [監査とログ](#監査とログ)

## 物理セキュリティ(GRUB)
Linuxのブートローダーの一つ  
ブートローダーとは、BIOS/UEFIの初期化に続いて実行されるプログラム。  

### 役割
ストレージからOSのカーネルをメインメモリに読み込み起動させること  
→ブートローダー自体の役割はOSを起動すること！



### GRUBのパスワード設定
> コマンド：grub2-mkpasswd-pbkdf2

GRUBのブート設定にアクセスするためのパスワード認証を作る  

システムが起動するまでの流れ
電源ON → UEFI/BIOS → GRUB → Linuxカーネル → ログイン画面

デバイスの起動時、無意識に上記の流れを繰り返していて、パスワード認証がないと物理的なアクセスが誰でも可能になる。

方法(Linux)
1. Shiftキー長押し
2. Escキー連打

GRUBに触れてできること  
- 起動オプションを変えられる  
GRUB画面で e を押すと、一時的に起動設定を編集できる。  
例）ログイン不要でrootシェル起動、rootパスワードのリセット、OSの修復などが可能になる
- root権限を奪える  
ログインパスワード、sudo制限があってもOS起動前の物理アクセスで全部回避可能

***⇒OS起動前の物理アクセスをさせないためにGRUBのパスワード認証も必要***

<br>

## ファイルシステムのパーティション分割と暗号化
LUKS：ディスク・ブロックデバイス(ストレージ)暗号化する技術  
Windowsのbitlockerと同じレイヤー

### 役割
ストレージの暗号化をすること  
LUKSがなかったら...?  
 ⇒ OS起動前にディスク内のバルクデータが閲覧できる。

### パーティション分割
LUKSで暗号化されているとき、ディスクレイアウトは以下のようになる
1. LUKS phdr：使用されている暗号、暗号モード、キーの長さ、マスターキーのチェックサムに関する情報などが記載される

2. KM：Key Materialの略で、KM1、KM2、が存在する。KMではマスターキーが暗号化されている。KMが複数存在するのは、KM1=管理者、KM2=復旧用、KM3=一時作業用、などマスターキーの暗号パスワードをそれぞれで設定できる

3. バルクデータ：マスターキーで暗号化されたデータ  

### LUKSの主な動作
OS起動前に、マスターキーを復号するためのパスワード認証が存在する

<mark>注意</mark>：OS起動前に復号されるのはマスターキーであり、バルクデータはまだ暗号化されたまま
<br><br>

#### バルクデータが復号/暗号されるタイミング  
基本的にブロックデバイス単位でそれぞれ暗号化されており、該当ファイルのブロックデバイスのみ復号/暗号する。

### 設定方法
ほとんどのディストリビューションでは、GUIを使ってづらいを暗号化できる。  
CLIからLUKSを設定する場合にはいくつかの手順を踏む必要がある。  
詳細：[tryhackme Linux-system-Harden > Filesystem Partitioning and Encryption](https://tryhackme.com/room/linuxsystemhardening)

### bitlockerとLUKSの違い
| | bitlocker | LUKS |
| --- | --- | --- |
| 鍵保管 | TPM | ユーザ入力 |
| 普段の入力 | 不要 | 必要 |
| 異常時 | 回復キー要求 | 普段と同じ入力 |
| OS依存 | Windows | Linux |

<br>

## ファイアーウォール
どのパケットを受け入れて、どのパケットを拒否するかを決めることが目的

システムの通信のセキュリティを高める

### Linux Firewall
1. ステートレスファイアーウォール  
**各パケットを独立に評価**  
IPヘッダーとTCP/UDPヘッダーの特定のフィールドを検査してパケットを判定
⇒過去の通信は一切覚えていない  
例）内部PC → 外部webサーバ（80番）にアクセス
　　webサーバ → 内部PCに返信パケット  
<u>個別に戻り通信を許可するルールを手動で書く必要がある。</u>


2. ステートフルファイアーウォール  
**通信開始時に状態を記録**  
過去の通信をもとに許可/拒否が可能

 ### Netfilter
 パケットフィルタリングソフトウェアを提供している。  
 管理するには「iptables」「nftables」などのフロントエンドが必要

#### iptables
Netfilterを操作するための従来ツール  

使い方

> iptables -A INPUT -p tcp --dport 22 -j ACCEPT
- -A INPUT：システム宛のパケット
- -p tcp --dport 22：宛先ポート22のTCPプロトコルに適用
- -j ACCEPT：ルールを許可

> iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT
- -A OUTPUT：システムから送信されるパケット
- -p tcp --sport 22：送信元ポート22のTCPプロトコルに適用

> iptables -A INPUT -j DROP
- それまでのルール以外のすべての着信トラフィックをブロックする

> iptables -A OUTPUT -j DROP
- それまでのルール以外のすべての送信トラフィックをブロックする

> iptables -F
- 以前のルールをフラッシュ(削除)できる

#### nftables
拡張性とパフォーマンスの面でiptablesよりも改善されている。

使い方

**1．ipptablesとは異なり、ルールを追加する前に、必要なテーブルとチェーンを追加する必要がある**

> nft add table TABLE_NAME (fwfilter)
- add(追加)、delete(削除)、list(テーブル内のルールとチェーンの一覧表示)、flush(すべてのチェーンとルールのクリア)
- テーブルの作成

**2．新しく作成したテーブルに、入力チェーンと出力チェーンをそれぞれ追加する**
> nft add chain fwfilter fwinput { type filter hook input priority 0 \; }  
nft add chain fwfilter fwoutput { type filter hook output priority 0 \; }

**それぞれのチェーンにSSHトラフィックを許可するために必要なルールを追加する**
> nft add fwfilter fwinput tcp dport 22 accept
- ローカルシステムの宛先ポート22へのTCPトラフィックを受け入れる

> nft add fwfilter fwoutput tcp sport 22 accept
- ローカルシステムの送信元ポート22からのTCPトラフィックを受け入れる

> nft list table fwfilter
- テーブルの形状を確認できる

### UFW
Uncomplicated Firewall　その名の通り設定を簡素化することができる。

> ufw allow 22/tcp
- ファイアーウォールをTCPポート22へのトラフィックに設定
<br><br>

もっと詳しく[<u>Firewall</u>](https://tryhackme.com/room/redteamfirewalls)について学びたくなったとき

<br>

## リモートアクセス
システムへのリモートアクセスの提供 → 物理的にいない場合でもシステムやファイルにアクセスできる  
 ⇔ 一方で、攻撃者が攻撃しやすい要因にもなる  
1. パスワードスニッフィング  
Telnetプロトロコルなどで通信が平文で送信されることで、パケットキャプチャで傍受できる。  
解決法：SSHサーバで通信を暗号化

2. パスワード推測とブルートフォース攻撃  
覚えやすさや入力の手間を省くために、脆弱なパスワードを設定しやすい。  
解決法：複雑なパスワードに設定? ← 現実的ではない  

~~パスワード認証~~  〇公開認証キー　

### 設定方法
OpenSSHサーバの設定は、sshd_configファイルで設定できる

> /etc/ssh/sshd_config  
- 「PermitRootLogin no」：rootログインを無効にすることができる
- 「PubkeyAuthentication yes」：公開鍵認証を有効にする
- 「PasswordAuthentication no」：パスワード認証を無効にする

> ssh-keygen -t rsa
- SSHキーペアを作成する
> ssh-copy-id username@server
- SSHサーバが公開鍵認証を行うには、公開鍵をSSHサーバにコピーする必要がある
<br><br>

## ユーザアカウントの保護
root権限 = なんでもできる　日常的な作業では非rootアカウントの使用が良い

rootログインを避ける良い方法　「sudo」

「sudoers」というsudoグループに所属することで、rootログインできる<u>権利</u>が得られる  
下記のコードでusernameをsudoersに追加できる。
> usermod -aG sudo username
- usermod：ユーザアカウントを変更
>deluser username sudo
- sudoersからユーザを削除する。不必要なユーザは削除するべき

### ルートを無効にする
管理目的でアカウントを作成し、sudo/wheelグループに追加したら、rootアカウントの無効化を検討

簡単な方法  
「/etc/pass」を編集して、rootシェルを「/sbin/nologin」に変更

「root:x:0:0:root:/root:<u>/bin/bash</u>」  
    ↓  
「root:x:0:0:root:/root:<u>/sbin/nologin</u>」

<mark>直接編集は注意：構文ミスでログイン不能などが起こり得るから</mark>

現代Linuxでの推奨方法
1. rootログインをロック
> sudo passwd -l root
2. SSHでroot禁止
> /etc/ssh/sshd_config  
PermitRootLogin no


### 強力なパスワードポリシーを適用する
***Libpwquality***ライブラリは、パスワード制約に関する多くのオプションを提供。

RedHat / Fedora
> /etc/security/pwquality.conf
Debian / Ubuntu
>/etc/pam.d/common-password

オプション：
- `difok`：古いパスワードに使われていなかった文字数を新しいパスワードに指定できる
- `minlen`：パスワードの最小文字数を指定する
- `minclass`：必要な文字クラス(大文字、小文字、記号、数字)の最小数を指定。
- `badwords`：パスワードに使えない文字を指定する
- `retry=N`：エラーを返す前に、N回試すことができる

**rootユーザに限らず、利用しなくなったユーザは無効化する！**

<br>

## 監査とログ
Linuxシステムのほとんどのログファイルは`/var/log`ディレクトリに保存

監査ログの種類
- `/var/log/messages`：Linuxシステムの一般的なログ
- `/var/log/auth.log`：すべての認証試行をリストするログファイル(Debianベース)
- `/var/log/secure`：すべての認証試行をリストするログファイル(Red Hat/Federeベース)
- `/var/log/utmp`：現在システムにログインしているユーザに関する情報を含むアクセスログ
- `/var/log/wtmp`：システムにログイン及びログアウトしたすべてのユーザの情報を含むアクセスログ
- `/var/log/kern.log`：カーネルからのメッセージを含むログファイル
- `/var/log/boot.log`：起動メッセージとブート情報を含むログファイル
