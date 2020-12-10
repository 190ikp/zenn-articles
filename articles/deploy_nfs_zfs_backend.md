---
title: "ZFSストレージバックエンドのNFSサーバを立てる"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ZFS", "NFS"]
published: true
---

※この記事は[Qiita](https://qiita.com/190ikp/items/a5e111cd9dbeca2f3a29)からの移行記事です

## やりたいこと

- RAID-Zで信頼性が高い冗長構成のストレージを組みたい
- NFSでZFSストレージを共有したい
- できるだけ楽したい

## やったこと

よくやるファイアウォールとかの設定はやっている前提です．

環境

- Ubuntu Server 18.04 LTS
- OpenZFS: `0.7.5-1ubuntu16.8`
    - 2020/6/25現在は`0.7.5-1ubuntu16.9`がaptでインストールできる最新のバージョンのようです
- RAID-Z用HDD 4TB x3
    - よく「SMRが〜」とか言われていますがSMRです．2.5インチだとSMRしか無かったので．
- ローカル: 172.16.3.0/24
- サーバのローカルIP: 172.16.3.10

HDDは別にこだわらなくてもいいと思います．壊れても良いための冗長構成なので．\
全部残さず使うのであれば，容量だけは揃えましょう．

(本当はRAID-Z x2でストライピングを組みたかったけど予算の関係で断念)

### ZFSストレージのセットアップ

先にパーティションを切っておきます．全部使う場合はこの工程はやらなくてもOKです．\
私はパーティションラベルをつけておきたかったのでやりました．\
(ラベルをつけることによるメリットは[ArchWiki](https://wiki.archlinux.jp/index.php/%E6%B0%B8%E7%B6%9A%E7%9A%84%E3%81%AA%E3%83%96%E3%83%AD%E3%83%83%E3%82%AF%E3%83%87%E3%83%90%E3%82%A4%E3%82%B9%E3%81%AE%E5%91%BD%E5%90%8D)にわかりやすく書かれています．)

```shell
$ sudo parted /dev/sdb --script \
> 'mklabel gpt mkpart disk0 zfs 2048s 7814037134s quit'

# 残り2つも同様に設定
```

partedの使い方については[こちら](https://www.saintsouth.net/blog/partition-management-with-parted-command/)の解説がわかりやすいです．\
後から気づいたのですが，パーティションの範囲は`%`でも指定できます．そのほうが楽です．

パーティションが切れたらZFSのストレージプールを作ります．\
先にOpenZFSをインストールしておきましょう．

```shell
$ sudo apt install -y zfsutils-linux
```
今回はRAID-Zで組むので

```shell
$ sudo zpool create tank raidz \
> /dev/disk/by-partlabel/disk0 \
> /dev/disk/by-partlabel/disk1 \
> /dev/disk/by-partlabel/disk2
```

を実行したらストレージプールの出来上がりです．\
生のディスクを使用する場合は

```shell
$ sudo zpool create tank raidz /dev/sdb /dev/sdc /dev/sdd
```

みたいな感じで実行してあげるとよしなにフォーマットしてストレージプールを作ってくれます．\
他の場所で使っていたディスクをそのまま放り込んでも大丈夫です．
作れたら確認してみましょう.

```shell
$ sudo zpool status
  pool: tank
 state: ONLINE
  scan: none requested
config:

	NAME           STATE     READ WRITE CKSUM
	 tank          ONLINE       0     0     0
	  raidz1-0     ONLINE       0     0     0
	        disk0  ONLINE       0     0     0
	        disk1  ONLINE       0     0     0
	        disk2  ONLINE       0     0     0

errors: No known data errors
```

つぎに，ストレージプールを用途ごとにデータセットに切り分けていきます．

```shell
$ sudo zfs create -o normalization=formD -o compression=lz4 tank/nfs-dataset
$ sudo zfs create -o normalization=formD -o compression=gzip-9 tank/backup-dataset
```

`-o`でデータセットを作る際のプロパティを指定します．\
ここで指定している`normalization=formD`はUnicodeの正規化，`compression`は圧縮方式の指定です．\
特に圧縮方式についてはどんな用途で使用するのかによって変えると効果的です．\
私はNFS用途ではそこそこ書込速度が欲しいので標準的な`lz4`，バックアップ用途では圧縮効率の良さが欲しいので`gzip-9`を指定しました．\
各圧縮方式を詳しく比較してくださっている記事が[こちら](https://www.nabe-intl.co.jp/takeruboost/zfs%E3%81%AE%E5%9C%A7%E7%B8%AE%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AE%E9%81%95%E3%81%84%E3%81%AB%E3%82%88%E3%82%8B%E5%9C%A7%E7%B8%AE%E7%8E%87%E3%80%81%E8%A8%88%E7%AE%97%E9%80%9F/)にあるので参考にすると良いと思います．

他にどのようなプロパティがあるかは`sudo zfs get all`で一覧を見ることができます．\
また，[OpenZFSのドキュメント](https://openzfs.github.io/openzfs-docs/man/8/zfsprops.8.html)には各プロパティの詳細が載っています．

作成されたデータセットは，デフォルトでは`/tank/nfs-dataset`のようにマウントされています．

### NFSサーバを設定する

ファイアウォールをきちんと機能させたいのでNFSのバージョンは`v4`を使用します．\
(v4だと使用するポートは`2049/tcp`に限定できます．)

まずNFSサーバ用のパッケージをインストールしておきます．

```shell
(server)$ sudo apt install -y nfs-kernel-server
```

インストールできたら，共有するディレクトリのルート(エクスポートルート)を設定していきます．\
ZFSの場合，この設定には2通りの方法があります．

#### 普通の方法

ZFS以外のストレージの場合と同じ方法です．

共有するディレクトリの設定を`/etc/exports`に書いていきます．\
なお，共有するディレクトリのパーミッションはあらかじめ適切な値に設定しておいてください．

```shell:/etc/exports
/tank/nfs-dataset 172.16.3.0/24(rw,sync,crossmnt,fsid=0,secure,no_subtree_check)
```

`fsid=0`を指定することで，NFSクライアントで設定するサーバのNFSエクスポートルートが`/`になります．\
クライアント側では`sudo mount -t nfs 172.16.3.10:/ [マウントポイント]`と打ち込んでやるだけでNFSストレージがマウントできるようになります．\
他のオプションについては，用途によって適宜変えてください．(参考: [man exports(5)](https://linux.die.net/man/5/exports))

ユーザのアクセスを制限するために，NFSではエクスポートルートにあたるディレクトリを専用のディレクトリにバインドマウントして使いますが，ZFSで同じ手法をとると使用できるディスクのサイズがクライアント側から見えなくなってしまいます．\
そのため，データセットのディレクトリを直接エクスポートルートに設定しています．

`/etc/exports`を書いたら一度`nfs-kernel-server.service`に読み込ませる必要があります．

```shell
(server)$ sudo exportfs -a # /etc/exportsから共有するディレクトリを追加
(server)$ sudo systemctl restart nfs-kernel-server
(server)$ showmount -e localhost # NFS用にマウントされてるか確認
Export list for localhost:
/tank/nfs-dataset 172.16.3.0/24
```

`exportfs`コマンドは`/etc/exports`のをチェックし，`/var/lib/nfs/etab`に載っていないものがあれば追加します．差分をチェックするわけではないので`/etc/exports`からエクスポートルートを削除しても`showmount`で確認すると残っています．\
共有ディレクトリを削除したい場合は`sudo exportfs -r`を実行します．これは現在保持しているエクスポートルートのリストを削除するオプションです．すべて削除されるので，この後に`sudo exportfs -a`で追加する必要があります．\
また，デフォルトのものも含めて，エクスポートルートに設定されているすべてのオプションを見たい場合は`/var/lib/nfs/etab`を参照すると良いです．

#### `sharenfs`プロパティを使う方法

ZFSの`sharenfs`プロパティを使うことで，NFSサーバで扱うZFSデータセットの設定が簡単にできます．\
[こちら](https://pthree.org/2012/12/31/zfs-administration-part-xv-iscsi-nfs-and-samba/)を参考にすると良いです．

まず，`/etc/exports`にダミーのエクスポートルートを設定をします．\
これは，エクスポートルートが何も設定されていない状態では`nfs-kernel-server.service`が起動しないためです．

```shell:/etc/exports
/mnt localhost(ro)
```

そのうえで`nfs-kernel-server.service`を起動します．

```shell
$ sudo systemctl start nfs-kernel-server.service
```

サービスが起動できたら，エクスポートしたいデータセットにsharenfsプロパティを設定します．

```shell
$ sudo zfs set sharenfs=on tank/nfs-dataset
$ sudo zfs get sharenfs tank/nfs-dataset
NAME              PROPERTY  VALUE     SOURCE
tank/nfs-dataset  sharenfs  on        local
$ showmount -e localhost
/tank/nfs-dataset *
/mnt              localhost.localdomain
```

さらにエクスポートのオプションを追加することもできます．

```shell
$ sudo zfs set \
> sharenfs="rw=@172.16.3.0/24,sync,crossmnt,fsid=0,secure,no_subtree_check,root_squash" \
> tank/nfs-dataset
$ sudo zfs get sharenfs tank/nfs-dataset
NAME              PROPERTY  VALUE                                                                       SOURCE
tank/nfs-dataset  sharenfs  rw=@172.16.3.0/24,sync,crossmnt,fsid=0,secure,no_subtree_check,root_squash  local
$ showmount -e localhost
/tank/nfs-dataset 172.16.3.0/24
/mnt              localhost.localdomain
```

指定できるオプションについては[ドキュメント](https://openzfs.github.io/openzfs-docs/man/8/zfsprops.8.html)に書いてあるとおり，`/etc/exports`で設定できるものについては大丈夫です．\
ちなみに本家にはある`share`プロパティはOpenZFSには存在しません．`sharenfs`か`sharesmb`のみです．

`sharenfs`プロパティを設定すると自動的にNFSでエクスポートされるようになります．\
エクスポートのon/offを明示的に切り替えたい場合は`zfs share|unshare`を使います．

Ubuntuの場合，以上の設定は次回からサーバ起動時に自動的に読み込まれます．\
サービスやcronで設定する必要はありません．

### NFSクライアントでマウント

サーバ側のセットアップができたのでクライアント側でマウントしてみます．

```shell
(client)$ sudo mount -t nfs -v 172.16.3.10:/ /mnt/nfs
```

ちなみに私のクライアント環境 (Ubuntu 18.04 LTS, `nfs-common`: 1:1.3.4) では，サーバ側のエクスポートルートをフルパス`/tank/nfs-dataset`にするとNFSのバージョンがv3になってしまい，ファイアウォールでブロックされてしまいました．\
`/mnt/nfs`はクライアント側のマウントポイントです．適当なディレクトリを設定してください．\
マウントがうまくいったら起動時に自動でマウントしてくれるよう，クライアントの`/etc/fstab`に書いておきましょう．

```shell:/etc/fstab
172.16.3.10:/ /mnt/nfs nfs noauto,x-systemd.automount,x-systemd.device-timeout=10,timeo=14,x-systemd.idle-timeout=1min 0 0
```

いい感じに設定してあげることでNFSストレージをアドホックにマウントできるようになります．[ArchWiki](https://wiki.archlinux.jp/index.php/NFS#systemd_.E3.81.A7_.2Fetc.2Ffstab_.E3.82.92.E4.BD.BF.E3.81.86)を参考にしてください．

## 最後に

今回はKerberos認証とかは使用していないので，このままグローバルに公開するのは危険です．\
~~え？LANだったら大丈夫だとでも思ってるんですか？~~\
(私の場合はnextcloudなりで適当なプライベートクラウドストレージを立てようと考えているのでNFSでユーザ認証をするつもりはありません．)\
インターネット経由でも使いたい場合は，別途ユーザ認証と通信の暗号化ができる環境を構築してください．

---
参考:

- OpenZFS
    - How to ZFS (日本語): https://chibiegg.gitbooks.io/how-to-zfs/content/
    - ArchWiki: https://wiki.archlinux.jp/index.php/ZFS
        - この2つはめちゃくちゃわかりやすいです
    - OpenZFS ドキュメント: https://openzfs.github.io/openzfs-docs/index.html
    - たぶん公式ドキュメントより詳しいHowTo(少し古め): https://pthree.org/2012/04/17/install-zfs-on-debian-gnulinux/
- NFS
    - Red Hat ドキュメント: https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/7/html/storage_administration_guide/nfs-serverconfig
