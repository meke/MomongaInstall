# はじめに
このドキュメントは、MomongaLinux開発版を最新の環境にインストールするまでの手順となる。MomongaLinuxの最新のインストーラは2014年作成であり、最近のPCでは利用できない。

このため、Fedoraを利用して、MomongaLinuxの新規インストールを行うための手順を記述する

# 環境
試した環境は以下の通り

* Qemuで作成した仮想環境
* 実機（Intel Corei 12世代のCPU/GPU）
  * Intelの内蔵GPUは「BIOS互換モード」では動作しない。このため、UEFIネイティブ環境を作成する必要がある。

# 現時点でわかっていること

* bootマネージャーはFedoraのものを一式利用する必要がある
  * grub.cfg は /boot/efi/EFI/fedora/ 配下に置く必要がある

# ざっくり手順

* パーティションを作成する
  * UEFIで起動するための設定を行う
  * UEFIのパーティションの作成
* FedoraでMomongaのchroot環境を構築
  * dnfコマンドを使って、MomongaLinuxのレポジトリのファイルをインストールする
* chroot環境にMomonga Linuxをインストールする
  * 初期インストール後の初期設定を行う
* パッケージの追加や、必要なデータのコピー
  * ブートするのに必要な設定を行う
* 再起動して、正常に起動してきたら成功

# 実際の手順

### 起動用パーティションを作成する

* UEFIで起動するための設定を行う
  * パーティションをGTPにする
    * ```parted /dev/vda mklabel gpt```
  * ```fdisk /dev/vda``` を実行し、```「Disklabel type: gpt」```になっていることを確認する
* UEFIのパーティションの作成
  * EFI用のパーティションを作成する
    * UEFIでBootするためには、UEFIに対応したファイルシステムが必要になる
    * ```parted /dev/vda mkpart fat32 2048s 1GB```
    * ```parted /dev/vda set 1 boot on``` でブートパーティション設定
  * /bootのパーティションを作成する
    * ```parted /dev/vda mkpart ext4 1GB 2GB```
  * システム領域を作成する
    * ```parted /dev/vda mkpart xfs 2GB 100%```
    * この資料ではswap領域は作成しない
 

### ファイルシステムの作成
パーティションを作成したので、ファイルシステムを作成する

* EFIの領域
  * ```mkfs.vfat /dev/vda1```
* /bootの領域
  * ```mkfs.ext4 /dev/vda2```
* システム領域
  * ```mkfs.xfs /dev/vda3```


### Mountの設定

* LiveCD上に作成したパーティションをmountする
  * ※ このときLiveCDの/media配下にパーティションを作成する

* システム領域のマウント
    * ``` mount /dev/vda3 /media ```
  *  /bootになるディレクトリをマウント
    * ``` mkdir -p /media/boot ```
    * ``` mount /dev/vda2 /media/boot ```
  *  /boot/efi/EFIになるディレクトリをマウント
    * ``` mkdir -p /media/boot/efi/ ```
    * ``` mount /dev/vda1 /media/boot/efi ```
    * ``` mkdir -p /media/boot/efi/EFI/ ```

* yum コマンドが利用するマウントも行います
  *  ```mkdir /media/dev /media/proc /media/sys```
  * mount --bind /dev /media/dev
  * mount --bind /proc /media/proc
  * mount --bind /sys /media/sys

### FedoraでMomongaのchroot環境を構築

* dnfの設定でMomongaLinuxを参照するようにする
* 最初にそれ以外のレポジトリをOFFにする
  * ``` dnf config-manager --disable "*" ```
* Momongaのレポジトリを追加 
  * ``` dnf config-manager --add-repo https://momonga.ooini.com/rpms/x86_64/ ```

### chroot環境にMomonga Linuxをインストールする
* 最初にパッケージのインストール
  * ``` dnf install  --nogpgcheck --installroot=/media/ -y --releasever=9 kernel grub2-efi-modules grub2 sudo passwd vim-minimal build-momonga NetworkManager-tui```
* EFI起動に必要なファイルをLiveCDからいただく
  * cp -r  /boot/efi/EFI/BOOT /media/boot/efi/EFI/
  * cp -r  /boot/efi/EFI/fedora /media/boot/efi/EFI/

## パッケージの追加や、必要なデータのコピー

* DNS用の/etc/resolv.conf作成
  * ```echo nameserver 8.8.8.8 > /media/etc/resolv.conf```


* ソフトウェア追加用のレポジトリの追加
  * ```/media/etc/yum.repos.d/meke.repo``` に以下のように書く

```
[meke-repo]
name=$basearch Meke's Repo
baseurl=https://momonga.ooini.com/rpms/x86_64/
enabled=1
gpgcheck=0
```

* ```/etc/fstab``` の作成
```
/dev/vda3          /                       xfs     defaults        0 0
/dev/vda2          /boot                   ext4    defaults        0 0
/dev/vda1          /boot/efi               vfat    umask=0077,shortname=winnt 0 2
```


```chroot /media``` でMomongaの環境に入り込む

FedoraのDNFだと、rpmのDB等がうまく作られないようなので、chroot下で同じコマンドを実行する
  * ```yum install  --nogpgcheck -y --releasever=9 kernel grub2-efi-modules grub2 sudo passwd vim-minimal build-momonga NetworkManager-tui```

しばらくはFedoraのshimやgrubを利用するので、/boot/efi/EFI/fedora/ 配下にgrub.cfgを吐くように調整
```
mv /etc/grub2-efi.cfg{,.orig}
ln -s ../boot/efi/EFI/fedora/grub.cfg /etc/grub2-efi.cfg
grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg 
```



* rootユーザのパスワード設定
  * ```passwd```  

* ユーザーの追加
  *``` useradd momonga```
  * ```passwd momonga```

### 再起動に向けて

* システムに必要な領域のアンマウント
  * ``` umount /media/dev ```
  * ``` umount /media/proc ```
  * ``` umount /media/sys ```
  
*ディスクのアンマウント
  * ``` umount /media/boot/efi/EFI ``` 
  * ``` umount /media/boot ```
  * ``` umount /media ```

その後、再起動する。インストールがうまくいっていると、MomongaLinuxが起動してくるので、自分で環境構築を行う
