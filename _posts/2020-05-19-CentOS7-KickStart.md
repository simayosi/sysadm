---
title: Kickstartを用いたCentOS7 VMの自動インストール（GPT構成）
---

# Kickstartを用いたCentOS7 VMの自動インストール（GPT構成）

動作検証などのために仮想マシン(VM)を作成することは良くあります。
しかし、毎回OSをインタラクティブにインストールするのは面倒です。  
そこで、KVM上にCentOS 7をKickstartを使ってGPT構成で全自動ネットワークインストールする方法をメモします。  
Kickstartの設定はKVM以外でも基本的に同じです。

{::options toc_levels="2,3" /}
* 目次
{:toc}

## 環境
- KVM
- virt-install, virsh
- CentOS 7

## Kickstartファイルの作成
KickstartはRedHat系Linuxディストリビューションを自動インストールする仕組み

CentOSの[Kickstartインストール公式ドキュメント](https://docs.centos.org/en-US/centos/install-guide/Kickstart2/)、[Kickstart公式ドキュメント](https://pykickstart.readthedocs.io/en/latest/kickstart-docs.html#chapter-3-kickstart-commands-in-red-hat-enterprise-linux)
を参照して
Kickstartファイルを作成する。  
なお、ここでは以下の構成とする。
* ネットワークインストール  
  （繰り返しインストールする場合はISOイメージを使う方が良い）
* GUIDパーティションテーブル(GPT)利用
* LVMを手動構成

以下のks.cfgファイルを適当な場所に作成

* \#で始まる行はコメント
* <>で囲んだパラメータは適当な値で置き換える（後述）

```
# テキストモードでXを設定せず全自動でインストール
cmdline
skipx

# 日本語キーボード, 言語を英語（米国）に設定
keyboard --vckeymap=jp --xlayouts='jp'
lang en_US.UTF-8

# タイムゾーンを設定（ハードウェアクロックはUTC, NTP無効）
timezone Asia/Tokyo --utc --nontp

# ミラーサイトからネットワークインストール（ISOイメージを使う場合は下の行）
install
url --mirrorlist http://mirrorlist.centos.org/?release=$releasever&arch=$basearch
# cdrom

# GUIDパーティションテーブルを作成
zerombr
clearpart --all --initlabel --disklabel=gpt
# ブートローダのstage 1.5をBIOSブートパーティションにインストール
bootloader --location=mbr --append=" crashkernel=auto"

# GPTの必須パーティションと/bootパーティションを作成
reqpart --add-boot

# 物理パーティション(pv.1)を作成
part pv.1 --size=<pv1_size>
# LVMグループcentos7をpv.1に作成
volgroup centos7 pv.1
# スワップパーティションを推奨サイズで作成
logvol swap --vgname=centos7 --name=swap --recommended
# ルートパーティションを残り全サイズで作成
logvol / --vgname=centos7 --name=root --fstype xfs --grow --size=1

# DHCP（IPv6無効）でネットワークを有効化（静的設定の例は下の行）
network --activate --bootproto=dhcp --noipv6
# network --bootproto=static --ip=10.0.2.15 --netmask=255.255.255.0 --gateway=10.0.2.254 --nameserver=10.0.2.1

# 認証の設定
auth --enableshadow --passalgo=sha512
rootpw --plaintext <password>

# インストール後にDVD-ROMをイジェクトして再起動（VM停止したい場合は下の行）
reboot --eject
# poweroff

%packages
# インストールしたいパッケージを指定（以下はbcの例）
bc
%end
```

#### パラメータ
* password: rootパスワード
* pv1_size: 物理ボリュームのサイズ (MiB)


## VMの作成
先ほど作成したKickstartファイルを指定してvirt-installでVMを作成
* \<centos_base\>は[CentOSミラーリスト](https://www.centos.org/download/mirrors/)の適当なミラーから選択
* ゲスト名やメモリ・CPU割当、ディスクサイズなどは適宜調整のこと
  （メモリ量が少ないとメモリ上にイメージが展開できず失敗するので注意）
```Shell
virt-install \
  --name test-centos7 \
  --memory 4096 --vcpus 1 \
  --location http://<centos_base>/7/os/x86_64/ \
  --os-variant centos7.0 \
  --disk path=system.img,size=10,format=qcow2 \
  --network bridge=br0 \
  --graphics none \
  --serial pty --console pty \
  --initrd-inject ks.cfg \
  --extra-args "inst.ks=file:/ks.cfg console=ttyS0"
```


## VMの起動・停止
### VM起動
ターミナルをコンソールとしてフォアグラウンドで起動
```Shell
virsh start test-centos7 --console
```

バックグラウンドで起動
```Shell
virsh start test-centos7
```

### VM停止
バックグラウンドで起動したVMを停止
```Shell
virsh shutdown test-centos7
```

### VM強制終了
```Shell
virsh destroy test-centos7
```

## VMの削除
不要になったVMを削除  
```Shell
virsh undefine test-centos7
```