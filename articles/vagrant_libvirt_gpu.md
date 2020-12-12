---
title: "サルでもできるGPUパススルー"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Vagrant", "libvirt", "GPU", "NVIDIA"]
published: false
---

＊これは[KCS AdventCalender2020](https://adventar.org/calendars/5690) 13日目の記事です＊

[←12日目](https://kcs1959.jp/archives/) | [14日目→]()

---
## これはなに

Linux上の仮想マシンでGPUを使いたいとき，ありますよね？\
それで調べてみると，情報が色々と錯綜していてつらくなった思い出は誰しも抱いてらっしゃると思います．\
（ん？お持ちでない？思い出が無いならつくりましょう．自家製サーバはいいぞ．）

今回はそんな時代の荒波に流されがちな仮想化界隈において特に翻弄されがちなGPUのパススルーについて，ホストマシンでやるべきことを書いていこうと思います．\
(GPUに限らずPCIデバイスであれば同じ方法でパススルー可能です)

## 環境

- Ubuntu 18.04 LTS (Linux Kernel 4.15)
  - ディストリビューションによってはGRUB2のconfigが配置されているディレクトリは違います\
    適宜読み替えてください
- GRUB2
  - 最近のLinuxカーネルならデフォルトでこれになっているはずです
- Intel Xeon Silver 4208 x2
  - CPUのベンダーは後ほど関係してきます
- NVIDIA RTX 2080Ti x2
- ASPEED AST2500 BMC (ホストの映像出力用)

別の環境でもテストを行ったため，そちらも載せておきます

- Ubuntu 18.04 LTS (Linux Kernel 5.4)
- AMD Ryzen Threadripper 1950X
- NVIDIA RTX 2070 SUPER
- NVIDIA GTX 750Ti (ホストの映像出力用)

仮想マシンにパススルーしたGPUはホストから一切使用できなくなるので，ホストの映像出力が必要な場合は別途用意しておきましょう．\
(仮想化ソフトウェアによっては一応ホストと共有する方法はあるのですが，パフォーマンスがガタ落ちするためあまりおすすめしません．)\
また仮想マシンの作成・起動にはVagrant + libvirt + KVMを使用しています．もちろんVirtualBoxやVMWareなどの別の方法でも大丈夫です，

## やってみる

### やること

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

```shell
$ sudo apt remove --purge -y cuda-drivers nvidia-container-toolkit \
  && sudo apt autoremove --purge -y

# プロプライエタリドライバ関連のパッケージが残っていないか確認
$ apt list -i | grep -e nvidia -e cuda 
```

### カーネルオプションの設定

GPUを仮想マシン内から利用するためVFIOを使用します．
先ほどBIOSからIOMMUを有効化したのはそのためです．

まず起動時にGRUBに渡すオプションを変更します．

```
$ sudo sed -i -e \
    "s/^GRUB_CMDLINE_LINUX=\"\"$/GRUB_CMDLINE_LINUX=\"quiet splash intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1 iommu=pt\"/g" \
    /etc/default/grub
```

また，以下の設定を書き込んだファイルを`/etc/modprobe.d/`に配置します．

```shell:/etc/modules-load.d/vfio-pci.conf
pci_stub
vfio
vfio_iommu_type1
vfio_pci
kvm
kvm_intel
```

完了したら設定を反映させましょう．

```shell
$ sudo update-grub
$ sudo reboot # マシンを再起動する必要があります
```

### GPUをブラックリストに追加

以下ではNVIDIAのGPUを使用しています．\
AMDの場合は随時読み替えてください．

まず，`lspci -nn | grep -i nvidia`でGPUのPCI IDを調べます．

```shell
$ lspci -nn | grep -i nvidia
3b:00.0 VGA compatible controller [0300]: NVIDIA Corporation GV102 [10de:1e07] (rev a1)
3b:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:10f7] (rev a1)
3b:00.2 USB controller [0c03]: NVIDIA Corporation Device [10de:1ad6] (rev a1)
3b:00.3 Serial bus controller [0c80]: NVIDIA Corporation Device [10de:1ad7] (rev a1)
af:00.0 VGA compatible controller [0300]: NVIDIA Corporation GV102 [10de:1e07] (rev a1)
af:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:10f7] (rev a1)
af:00.2 USB controller [0c03]: NVIDIA Corporation Device [10de:1ad6] (rev a1)
af:00.3 Serial bus controller [0c80]: NVIDIA Corporation Device [10de:1ad7] (rev a1)
```

ここで出力の表示形式は以下のようになっています．

```shell
<PCIアドレス> <デバイスクラス> [<クラスコード>]: <デバイス名> [ベンダーID:デバイスID] <リビジョン番号>
```

`ベンダーID:デバイスID`の形式で表されたものをPCI IDといいます．\
上記の出力が得られた場合，`10de:1e07`，`10de:10f7`，`10de:1ad6`，`10de:1ad7`が使用するPCI IDになります．

:::message alert
パススルーするGPUについては，VGA compatible controllerだけでなく付随するPCIデバイス(USB controller等)もすべてパススルーする必要があります．
:::

なお，`grep`で絞り込みをしなくても`lspci -d <ベンダーID>:<デバイスID>`で絞り込むことができるので，例えばNVIDIAのGPUなら

```shell
$ lspci -nn -d 10de:
```

でPCIデバイスを絞り込みことができます．\
ここで`10de`はNVIDIAのベンダーIDです．AMDの場合は`1002`になります．\
PCIデバイスのベンダーID・デバイスIDはあらかじめ決められているため，[こちらのサイト](https://pcilookup.com/)などで検索することもできます．

パススルーするPCIデバイスのPCI IDがわかったら，それらを`/etc/modprobe.d/`に作成した適当なファイルに書き込んでいきましょう．

```shell:/etc/modprobe.d/vfio.conf
options vfio-pci ids=10de:1e07,10de:10f7,10de:1ad6,10de:1ad7
```

後は`initramfs`を更新するだけです．

```shell
$ sudo update-initramfs -u
```

ビルドが完了したらマシンを再起動しましょう．

```shell
$ dmesg | grep -i vfio
```

を実行して出力が得られればOKです．\
あとはお好みの仮想化ソフトウェアを使ってGPUのパススルーができます．簡単ですね！

### 実際にパススルーしてみる

最初に述べたように今回はVagrantでお手軽にやっていきます．\
使用したバージョンは

- Vagrant: 2.2.10
- vargant-libvirt: 0.2.1
- QEMU: 2.11

です．\
Vagrant等のインストールは済んでいる前提で進めていきます．

まず適当な`Vagrantfile`を用意します．boxはlibvirt用のものを使用してください．

```ruby:Vagrantfile
Vagrant.configure("2") do |config|

  config.vm.box = "generic/ubuntu1804"

  config.vm.provider "libvirt" do |v|
    v.kvm_hidden = true
    # GPUパススルーを行う際はこのオプションをtrueにする必要があります
  end

  config.vm.define :"gpu-instance-0" do |domain|
    domain.vm.provider "libvirt" do |v|
      v.cpus = 4
      v.memory = 8192
      v.pci :domain => '0x0000', :bus => '0x3b', :slot => '0x00', :function => '0x0'
      v.pci :domain => '0x0000', :bus => '0x3b', :slot => '0x00', :function => '0x1'
      v.pci :domain => '0x0000', :bus => '0x3b', :slot => '0x00', :function => '0x2'
      v.pci :domain => '0x0000', :bus => '0x3b', :slot => '0x00', :function => '0x3'
    end
  end
end
```

ここで最も重要なのは`v.kvm_hidden`オプションを有効化することです．\
GPUが仮想マシン上で動いていることを検知すると，プロプライエタリなデバイスドライバをゲスト上でインストールできなくなることがあります．\
そのため，仮想マシンであることをゲストOSが検知しないよう，このオプションを有効化します．

また，PCIアドレスは前項で`lspci`を叩いて表示したものを使用します．\
上記の`Vagrantfile`の場合だと

```
3b:00.0 VGA compatible controller [0300]: NVIDIA Corporation GV102 [10de:1e07] (rev a1)
3b:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:10f7] (rev a1)
3b:00.2 USB controller [0c03]: NVIDIA Corporation Device [10de:1ad6] (rev a1)
3b:00.3 Serial bus controller [0c80]: NVIDIA Corporation Device [10de:1ad7] (rev a1)
```

の4つを使用しています．

:::message alert
パススルーしたいGPUに付随するデバイスもすべてパススルーさせる必要があります．\
また，すべてのアドレスは16進数の接頭辞`0x`を付けて記述する必要があります．
:::

必要な設定項目を書き加えたら，`vagrant up`で仮想マシンを立ち上げてみましょう．\
仮想マシンにログインして`lspci`などを叩いてみるとGPUがパススルーされていることが確認できると思います．

今回の内容は[こちらのリポジトリ](https://github.com/190ikp/libvirt-gpu)にまとめて置いてあります．ご自由にお使いください．

以上です．

---

参考:

- DeepOps Virtual: https://github.com/NVIDIA/deepops/blob/master/virtual/README.md#enabling-virtualization-and-gpu-passthrough
- vagrant-libvirt: https://github.com/vagrant-libvirt/vagrant-libvirt#pci-device-passthrough

---

[←12日目](https://kcs1959.jp/archives/) | [14日目→]()
