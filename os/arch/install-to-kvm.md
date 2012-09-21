# Arch Linux の KVM へのインストール

Arch Linux を、 KVM にインストールします。

## 前提

Arch Linux 上の KVM に Arch Linux をインストールします。
KVM の管理に libvirt を使い、フロントエンドとして virt-manager を使っています。

QEMU や KVM や libvirt や virt-manager についての解説はしませんが、
一応仮想マシンを virt-manager から作成するところから始めます。

Arch Linux のインストールの解説記事として読む場合は、
関係ない部分は読み飛ばしてください。
KVM 特有の話はそれと分かるように書くつもりですが、
もし不親切な場合は自分で何とかしてください(๑╹◡╹๑)。

## 準備

あらかじめ archlinux-2012.09.07-dual.iso を手に入れておきます。

libvirt を使うので、 /var/lib/libvirt/images/ というディレクトリに
コピーしておくとよいです。

KVM で、ディスプレイとして VNC ではなく Spice を使いたいので、
qemu-kvm のかわりに AUR から qemu-kvm-spice をインストールしておきます。
qemu-kvm をインストールした状態だと依存パッケージのインストールが失敗する
ようなので、 qemu-kvm をアンインストールしてからインストールしましょう。
さらに Spice の画面を virt-manager で表示するために
spice-gtk をインストールします。

## 仮想マシンの作成

この節はまるごと virt-manager の話なので、読み飛ばしてください。

### virt-manager を起動

星のついてるアイコンをクリックすると仮想マシンの新規作成が始まります。

### Step 1 of 5

仮想マシンを作成します。
arch-505b816e という名前にしました。
5(ry ってのはこの仮想マシンを作った時の Unix エポックからの秒数の
16進数表記ですね。最近はいろんな名前をつけるのによくこの数字を使ってます。

iso からインストールするので Local install media を選択します。

### Step 2 of 5

Use ISO image を選択し、 Browse でさっきコピーしたファイルが
一覧に出ているので、選択します。

OS type は Linux 、Version は Show all OS options を選択し、
増えた選択肢の中の 'Generic 2.6.25 or later kernel with virtio' 
を選択しました。

### Step 3 of 5

CPU は 6 コア、メモリは 1024 MB にしました。
1 コアと 512 MB ぐらいでも良さそう。

### Step 4 of 5

ディスクの大きさは 32 GB にしました。
Allocate entire disk now するとたぶん時間がかかるので、チェックを外しました。

### Step 5 of 5

Customize configuration before install にチェックを入れます。

### 設定のカスタマイズ

NIC を選択し、Source device を NAT から Host device eth0 に変更しました。
自分のネットワークに合わせて設定してください。

Display を選択し、 Type を VNC から Spice に変更しました。

Display を Spice にするとインストールできないので、
インストールする時点では VNC のままにしておきます。

### インストール開始

Begin Installation をおせば、インストール開始です。

## インストール

Arch Linux のインストールメディアの起動画面が出てきます。
'Boot Arch Linux (x86\_64)' を選択します。
i686 な人は i686 を選択してください。

起動が終われば、コンソール画面が出てくるはずです。

    root@archiso ~ # 

驚くべきことに、ログインシェルが zsh です。先進的ですね。
（これはインストールメディアの話で、
インストールされた Arch Linux のデフォルトのログインシェルは bash です）。

### インストール中のキーボード配列の設定

106 キーボードを使っている人は

    # loadkeys jp106

すると 106 キーボードになります。

### パーティションを切る

cfdisk でパーティションを切ります。

/boot 等を分けず、 / とスワップパーティションのふたつだけにします。
最近の Arch Linux はブートローダが GRUB2 のみサポートされるようになっており、
GRUB2 は ext4 からのブートをサポートしているので、
ext4 を使う場合、 /boot を分けることは必須ではないでしょう。
しかし Btrfs はサポートしていないようなので、 Btrfs を使いたい場合は
/boot を別にする必要があるでしょう。

virtio の場合、デバイスファイルは /dev/vd\* になります。
違う人は /dev/sda 等、適当に読み替えてください。

    # cfdisk /dev/vda

コンソールの UI が出てきます。

New → Primary → 2048 → Beginning でスワップ用のパーティションを作成(vda1)。

'↓' キーで残りの Free Space を選択し、
New → Primary → Enter で残りの領域のパーティションを作成(vda2)。

vda1 を選択し、 Type → 82 で Linux swap パーティションに。

vda2 を選択し、 Type → 83 で Linux パーティションに。

最後に Write → yes で結果を書き込んだ後、 Quit しました。

### ファイルシステムをつくる

    # mkfs.ext4 /dev/vda2

### マウント

インストールのためにマウントします。

    # mount /dev/vda2 /mnt

パーティションを複数に分けた場合は、サブディレクトリのパーティションも
マウントしてください。

    # mount /dev/vda2 /mnt
    # mkdir /mnt/boot
    # mount /dev/vda3 /mnt/boot
    # mkdir /mnt/home
    # mount /dev/vda4 /mnt/home
    # ...

### ネットワークをつなぐ

有線なので dhcpcd を起動します。

    # dhcpcd eth0

### ベースシステムをインストールする

pacstrap というスクリプトを実行して、ベースシステムを /mnt
にインストールします。
base グループと base-devel グループを
インストールすればよいです。

    # pacstrap /mnt base base-devel

### grub-bios をインストールする

grub-bios をインストールします。

    # pacstrap /mnt grub-bios

### genfstab

基本的な fstab を自動生成します。

    # genfstab -p /mnt >> /mnt/etc/fstab

### /mnt に chroot

以降は /mnt に chroot して作業します。

    # arch-chroot /mnt

### Vim をインストール

Arch Linux の vi は ex-vi です。
不便なので Vim を使う人は Vim をインストールしましょう。

    # pacman -S --noconfirm vim

注意すべきは、 vim と gvim という競合する別々のパッケージが
あることです。
gvim は依存パッケージが増える代わりに GUI やスクリプトとの
interop の機能が有効になっています
（ gvim にも /usr/bin/vim はあります）。
どうせ gtk 等をインストールする場合は、最初から gvim を
インストールしてしまってもよいかもしれません（試してはいません）。

    # pacman -S --noconfirm gvim

Vim を使わない人はテキストエディタとして nano を使いましょう。

    # nano ＜ファイル名＞

テキストエディタが用意できたら、いよいよ設定を始めます。
/etc をまるごと Git レポジトリにしてしまったりしたいところですが、
僕がいままでそういうことをしたことがないので、とりあえずやめておきます。

### ホスト名

/etc/hostname に hostname をかきます。
適当にホスト名を考えて書きます。

    # echo 'kvmarch' >> /etc/hostname

### ローカル時刻

ローカル時刻を設定します。シンボリックリンクを貼ります。

    # ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

### ロケールの設定

ロケールの設定をします。
僕は en\_US.UTF-8 にしています。

    # echo 'LANG=en_US.UTF-8' >> /etc/locale.conf

日本語にしたい場合は ja\_JP.UTF-8 にします。

    # echo 'LANG=ja_JP.UTF-8' >> /etc/locale.conf

次に /etc/locale.gen を編集して、
必要なロケールをコメントアウトします。

    # vim /etc/locale.gen

    ...
    #en_SG ISO-8859-1
    en_US.UTF-8 UTF-8
    #en_US ISO-8859-1
    ...
    #ja_JP.EUC-JP EUC-JP
    ja_JP.UTF-8 UTF-8
    #ka_GE.UTF-8 UTF-8
    ...

### mkinitcpio

virtio なディスクを使っている場合、 /etc/mkinitcpio.conf に
ロードすべきモジュールを指定しておく必要があります。

    # vim /etc/mkinitcpio.conf

    ...
    MODULES=""
    ...

を次のように書き換えます。

    ...
    MODULES="virtio_blk virtio_pci virtio_net"

virtio を使っていない場合、これは必要ないでしょう。

次に mkinitcpio します。

    # mkinitcpio -p linux

### GRUB2 

ブートローダとして GRUB2 をインストールします。

まずは grub-bios というパッケージをインストールします。

    # pacman -S --noconfirm grub-bios

次に grub-install で MBR にブートローダをインストールします。

    # grub-install /dev/vda

grub.cfg を自動生成します。

    # grub-mkconfig -o /boot/grub/grub.cfg

### パスワードの設定

root のパスワードを設定しておきます。

    # passwd
    Enter new UNIX password: 
    Retype new UNIX password: 
    passwd: password updated successfully

### 再起動

ここまでうまくできていれば、 exit で chroot から抜けて再起動し、
インストールされた Arch Linux を起動できるはずです。

    sh-4.2# exit
    exit
    arch-chroot /mnt  9.63s user 5.89s system 0% cpu 39:58.25 total
    root@archiso ~ # reboot

### ログイン

以下のようなログイン画面が出てくれば、成功ですね。

    Arch Linux 3.5.4-1-ARCH (tty1)
    
    kvmarch login:

さっき設定したパスワードで root でログインします。

    kvmarch login: root
    Password:
    [root@kvmarch ~]# 

キーボード配列を変更する場合は、
ひとまずまた loadkeys しておきます。

    loadkeys jp106

### swap

さいごに、スワップをつくります。

    # mkswap /dev/vda1

fstab にスワップパーティションの設定を追加します。

    # vim /etc/fstab

    /dev/vda1 swap swap defaults 0 0 

設定したとおりにスワップを有効にできるか確かめてみます。

    # free
    ...
    Swap:    0 ...
    # swapon -a
    # free
    ...
    Swap:    1999836 ...

たしかに swapon -a （ /etc/fstab のとおりにスワップを有効にする）で
スワップ領域ができていますね。

### シャットダウン

シャットダウンしてみます

   # shutdown -h now

## virt-manager で、ディスプレイを Spice にする

Spice が使えるようになっているはずです。

ツールバーの星アイコンをクリックして表示される virt-manager の設定画面で、
Display を選択し、 Type を Spice に変更して Apply します。
Channel を追加するといってくるので Yes します。
ゲストを再起動すると Spice のディスプレイになっているはずです。

##　おわり

これでインストールをおわりとします。
これ以降のはなしは別の記事にかきます。

