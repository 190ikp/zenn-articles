---
title: "サルでもできるGPUパススルー"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Vagrant", "libvirt", "GPU", "NVIDIA"]
published: false
---

＊これは[KCS AdventCalender2020](https://adventar.org/calendars/5690) 13日目の記事です＊

## これはなに

Linux上の仮想マシンでGPUを使いたいとき，ありますよね？\
で調べてみると，情報が色々錯綜していてつらくなった思い出は誰しも抱いていらっしゃると思います．\
（ん？お持ちでない？思い出がないならつくりましょう．自家製サーバはいいぞ．）

今回はそんな時代の荒波に流されがちな仮想化界隈において特に翻弄されがちなGPUのパススルーについて，ホストマシンでやるべきことを書いていこうと思います．\
(GPUに限らずPCIデバイスであれば同じ方法でパススルー可能です)

## 環境

- Ubuntu 18.04 LTS (Linux Kernel 4.15)
  - ディストリビューションによってはGRUB2のconfigが配置されているディレクトリが違います\
    適宜読み替えてください
- GRUB2
  - 最近のLinuxカーネルならデフォルトでこれになっているはずです
- Intel Xeon Silver 4208 x2
  - CPUのベンダーは後ほど関係してきます
- パススルーしたいGPU
  - 今回はNVIDIA RTX 2080Ti x2を使用します

別の環境でもテストを行ったため，そちらも載せておきます

- Ubuntu 18.04 LTS (Linux Kernel 5.4)
- AMD Ryzen Threadripper 1950X
- NVIDIA RTX 2070 SUPER
- NVIDIA GTX 750Ti (ホストの映像出力用)

仮想マシンにパススルーしたGPUはホストから一切使用できなくなるので，ホストの映像出力が必要な場合は別途用意しておきましょう．\
(仮想化ソフトウェアによっては一応ホストと共有する方法はあるのですが，パフォーマンスがガタ落ちするためあまりおすすめしません．)\
また仮想マシンの作成・起動にはVagrant + libvirtを使用しています．もちろんVirtualBoxやVMWareなどの別の方法でも大丈夫です，

## やってみる

### やるべきこと

- 仮想化支援機能・IOMMUが使えるようにBIOS / UEFIを設定する
- プロプライエタリなドライバ(`cuda-drivers`など)が入ってる場合はアンインストール
- GPUをホストが使えないようにする(ブラックリストに入れる)
- 必要に応じてカーネルオプションを変更
- 起動時に読み込まれるGRUBの設定に，変更したブラックリスト / カーネルオプションを反映させる

おおまかこんな流れです．

では順にやっていきましょう

### BIOS / UEFIの設定

え？BIOSよくわからんって？大丈夫です．特に難しいことはしません．\
まずはCPUの仮想化支援機能からみていきましょう．

お使いのマシンのCPUが


- Intelなら`Intel Virtualization Technology (VT)`
- AMDなら`AMD-V`/`SVM mode`

というような項目があると思います．まずはこれを有効化しましょう．有効化することで仮想マシンの実行速度が目に見えて速くなります．

次にIOMMU[^1]を有効化しましょう．\
こちらは

- Intelなら`Intel Virtualization Technology for Directed I/O (VT-d)`
- AMDなら`IOMMU`

を有効化してください．
また，同時に`Above 4G Decoding`[^2]という項目も有効化してください．

[^1]: IOMMUは，仮想マシン上のメモリが持つ仮想アドレスと，ホストマシンのメモリが持つ物理アドレスの対応づけ(マッピング)を行う機能です．

[^2]: Above 4G Decodingは，仮想アドレスから物理アドレスへのマッピングを4GiB以上の領域でも可能にするためのオプションです．

また，Secure Bootを有効化しているとトラブることがあります．特に支障がなければ無効化しておきましょう．

設定できたらマシンを再起動し，ログインしてください．

### デバイスドライバのアンインストール

`nouveau`などの汎用ドライバを使用している，もしくはホストの映像出力に使用するGPUがパススルーするGPUと同じドライバを使用している(上記の2つ目の構成のような場合)のであれば，この手順は飛ばしていただいて大丈夫です．

私の場合，`cuda-drivers` + `nvidia-container-toolkit`でコンテナにGPUへのアクセスを与えていたため．まずはこのNVIDIAのドライバをアンインストールします

```bash
$ sudo apt remove --purge -y cuda-drivers nvidia-container-toolkit \
  && sudo apt autoremove --purge -y
$ apt list -i | grep -e nvidia -e cuda # ドライバ関連のファイルが残っていないか確認
```

### カーネルオプションの設定

GPUを仮想マシン内から利用するため[VFIO](https://www.kernel.org/doc/Documentation/vfio.txt)を使用します．
先ほどBIOSからIOMMUを有効化したのはそのためです．

```
pci_stub
vfio
vfio_iommu_type1
vfio_pci
kvm
kvm_intel
```

---

[←12日目](https://kcs1959.jp/archives/) | [14日目→](https://kcs1959.jp/archives/)
