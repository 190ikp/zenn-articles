---
title: "NVIDIAのGPUドライバを楽にインストールしたい"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GPU", "NVIDIA", "Linux", "Ubuntu"]
published: true
---

＊これは[KCS AdventCalendar2020](https://adventar.org/calendars/5690) 8日目の記事です＊

[←6日目](https://kcs1959.jp/archives/5941/general/物理情報工学科の紹介) | [9日目→]()

---
## はじめに

NVIDIAのGPUを使用する際，トラブルに見舞われる方の多くはドライバのインストールでコケていらっしゃると思います．\
また，NVIDIAのGPUドライバにはいくつかバリエーションがあり，どれをインストールしたらよいか分からない，という方もいらっしゃると思います．\
この記事がそのような方の一助となれば幸いです．

筆者の環境はUbuntu 18.04 LTS (Linux Kernel 4.15.0 および 5.4.0)です．ご自身の環境によって適宜読み替えてください．

## インストール方法

先にリポジトリ等の準備をしておきます．これは，これから紹介するどの方法においても共通です．

まず，GPUドライバを配布しているNVIDIAのリポジトリをパッケージマネージャに登録します．
CPUがx86系(Intel or AMD)でOSがUbuntu 18.04の場合は\
[https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64](https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64)\
を登録します．この際にパッケージ認証用のキーも登録しておきましょう．

```shell
$ sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/7fa2af80.pub
$ echo "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64 /" |
    sudo tee /etc/apt/sources.list.d/cuda.list
```

対応するディストリビューションの一覧は\
[https://developer.download.nvidia.com/compute/cuda/repos/](https://developer.download.nvidia.com/compute/cuda/repos/)\
から探してください．

次にカーネルのヘッダファイルとビルドツールをインストールします．

```shell
$ sudo apt update
$ sudo apt upgrade -y
$ sudo apt install -y \
    "linux-headers-$(uname -r)" \
    build-essential
```

これで準備完了です．

### 最もシンプルなパターン

ほとんどの方はこちらの方法で十分です．\
この方法でインストールしたドライバは，パッケージのアップデートに伴ってバージョンが変更されます．ドライバのバージョンを固定したい場合は次の[ドライバのバージョンを固定したい場合](#ドライバのバージョンを固定したい場合)を参照してください．

やり方は非常にシンプルで，`cuda-drivers`をインストールするだけです．

```shell
$ sudo apt install -y cuda-drivers
```

インストールができたらマシンを再起動しましょう．

:::message alert
パッケージインストール後の再起動はどの方法でも必要です．
:::

`nvidia-smi`を叩いて以下のような出力が得られればOKです．

```
$ nvidia-smi
Tue Dec  8 01:08:31 2020       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.80.02    Driver Version: 450.80.02    CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  GeForce RTX 208...  Off  | 00000000:3B:00.0 Off |                  N/A |
|  0%   37C    P0    62W / 300W |      0MiB / 11019MiB |      1%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   1  GeForce RTX 208...  Off  | 00000000:AF:00.0 Off |                  N/A |
| 32%   38C    P0    25W / 300W |      0MiB / 11019MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

`cuda-drivers`をインストールすると，

- 最新のGPUドライバ
- X11 / Wayland用のパッケージ\
  (よくわからない方は「GUIに関わるパッケージ」と考えていただければ十分です)
- 各種管理用ユーティリティ

等がインストールされます．詳しく知りたい方は依存関係を調べてみてください．\
また，よくトラブルの原因になるnouveauについては，自動的にブラックリストに入れられるため特に操作を行う必要はありません．

こちらの方法を使用する場合のUbuntu 18.04用スクリプトをgistに置いてあります．お使いください．

https://gist.github.com/190ikp/b368905ebc7e748d065e5f57d59892df

この後CUDAを使用する場合は，`cuda-toolkit-<version>`パッケージをインストールします．\
`cuda-<version>`と異なり，このパッケージにはGPUドライバが含まれていません．そのためデバイスドライバとCUDAのバージョンを別々に管理できるようになり，非常に便利です．

また，`cuda-drivers` + `nvidia-container-toolkit`をホストにインストールしてDocker / LXDコンテナ内からGPUを使用する場合，`cuda-<version>`をコンテナ内環境にインストールすると依存関係が壊されます．\
これは`cuda-<version>`パッケージに含まれるGPUドライバとホストのドライバが干渉するためです．\
そのため，この場合もコンテナ内には`cuda-toolkit-<version>`をインストールするようにしてください．

### ドライバのバージョンを固定したい場合

この場合は，`nvidia-driver-<version>`をインストールします．\
こちらを利用した場合は，ドライバのバージョンが`<version>`で指定したものに固定されます．\
インストール可能なバージョンは`apt search`で調べることができます．

```shell
$ apt search ^nvidia-driver
Sorting... Done
Full Text Search... Done
nvidia-driver-390/bionic-updates,bionic-security 390.138-0ubuntu0.18.04.1 amd64
  NVIDIA driver metapackage

nvidia-driver-410/unknown 410.129-0ubuntu1 amd64
  NVIDIA driver metapackage

nvidia-driver-418/bionic-updates 430.50-0ubuntu0.18.04.2 amd64
  Transitional package for nvidia-driver-430

nvidia-driver-418-server/bionic-updates 418.152.00-0ubuntu0.18.04.1 amd64
  NVIDIA Server Driver metapackage

nvidia-driver-430/unknown,unknown 455.45.01-0ubuntu1 amd64
  Transitional package for nvidia-driver-455

nvidia-driver-435/bionic-updates 455.38-0ubuntu0.18.04.1 amd64
  Transitional package for nvidia-driver-455

nvidia-driver-440/bionic-updates 450.80.02-0ubuntu0.18.04.2 amd64
  Transitional package for nvidia-driver-450

nvidia-driver-440-server/bionic-updates 440.95.01-0ubuntu0.18.04.1 amd64
  NVIDIA Server Driver metapackage

nvidia-driver-450/unknown 450.80.02-0ubuntu1 amd64
  NVIDIA driver metapackage

nvidia-driver-450-server/bionic-updates 450.80.02-0ubuntu0.18.04.3 amd64
  NVIDIA Server Driver metapackage

nvidia-driver-455/unknown,unknown 455.45.01-0ubuntu1 amd64
  NVIDIA driver metapackage
```

2020年12月現在の最新のバージョンは`nvidia-driver-455`です．\
なお，前項で`cuda-drivers`をインストールしましたが，これはその時点で最新の`nvidia-driver`をインストールするような挙動になります．つまり2020年12月時点では`nvidia-driver-455`がインストールされます．

また，`nvidia-driver-<version>-server`というパッケージが見えていますが，こちらは見ての通りbionic-updatesで提供されています．また，依存する各パッケージも`server`プレフィックスのついたものとなっています．必要に応じて選択してください．

### GUIが不要な場合

純粋にGPGPU用途でGPUを使用する場合，`headless`プレフィックスのついたパッケージを使用できます．
この場合は`nvidia-headless-<version>`をインストールしてください．\
こちらも`nvidia-headless-<version>-server`という，bionic-updatesで提供されるバージョンが選択できます．

なお，このパッケージには管理用のユーティリティも含まれません．
`nvidia-smi`等が必要な場合は`nvidia-utils-<version>` (`nvidia-utils-<version>-server`)をインストールしてください．

また，`headless`にはOpenGlやVulkan等の主要なグラフィックスライブラリも含まれません．\
レンダリングファーム等で使用する場合は前項の`nvidia-driver`か`cuda-drivers`をインストールされることをおすすめします．

## まとめ

- 右も左もわからない場合は`cuda-drivers`を入れておきましょう．幸せになれます．
- 特に支障がなければ`cuda-drivers`を入れておきましょう．楽です．
- サーバ用途であれば`nvidia-driver-<version>-server`を入れておきましょう．\
  幸せになれるかもしれません．
- Tesla欲しい．

以上です．

---

7日目が無いため7日目へのリンクを貼っておきます

[←6日目](https://kcs1959.jp/archives/5941/general/物理情報工学科の紹介) | [9日目→]()