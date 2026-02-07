# Linuxのシステム強化
Linuxのシステム強化技術について、その技術とコマンドの使い方などについて記述する。

## 目次
- [物理セキュリティ(GRUB)](#物理セキュリティgrub)
- [ファイルシステムのパーティション分割と暗号化](#ファイルシステムのパーティション分割と暗号化)

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


