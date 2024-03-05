## BPI-W2-bspをQua Station風味に調整したカーネル

### ベースにしたカーネル

https://github.com/BPI-SINOVOIP/BPI-W2-bsp/releases/tag/w2-4.9-v1.0

### おもに参考にしたサイトとリポジトリ

1. https://github.com/tsukumijima/QuaStation-Ubuntu / QuaStation-Ubuntu

2. https://blog2.k05.biz/2021/04/qua-station-archlinux-ubuntu.html / 【Qua station】Arch Linux、Ubuntu Baseを起動させてみる - 某氏の猫空

3. https://noriokun4649.blog.fc2.com/blog-entry-371.html / Qua Stationで起動できるUbuntu Base 22.04LTSのrootfsを作る - noriokunの気まぐれ日記

※U-bootからの起動手順は2.や3.とまったく同じなので説明を省略

### 概要

Qua Stationの改造についてはすでに様々な情報があるので、実機とすり合わせて要点を洗い最小限の改変でおもな機能を使えるようにしたカーネルです。
USB、SDスロット、eMMC、RTL8812AR、RTL8192ER、モデム、CPUファン、ボタン、LEDは認識や制御できている雰囲気です。
HWエンコーダー、Bluetoothは(あまり興味もなく)解析を省略しました。  
Qua Stationと直接関係ないですが、ある種のUSBチューナーに対応するため [参考1](https://github.com/tsukumijima/QuaStation-Ubuntu) と同様に [6b7b620](https://github.com/xtne6f/BPI-W2-bsp/commit/6b7b6208e68d17ad967ab99b3f805b29b015e05a) をバックポートしています。  
以下に例としてDebian(bookworm)のミニマムな構築手順を記します。構築環境はWindows11 22H2上のWSL2のDebian(bookworm)です。

### 1. カーネルのビルド

```bash
cd ~
sudo apt install bc build-essential dosfstools git libncurses-dev parted
git clone https://github.com/xtne6f/BPI-W2-bsp.git
cd BPI-W2-bsp
./build.sh 1  # ここで引数なしの./build.shとすればカーネルコンフィグ画面も選べる
```

「Build success!」がでたら成功。

### 2. USBメモリに書き込むファイルイメージの作成

WSL2はUSBメモリを直接マウントできないので、ループバックデバイスを使ってファイルイメージを作りあとでUSBメモリに書き込む。

```bash
dd if=/dev/zero of=~/usb.img bs=1M count=1900  # 大きさはUSBメモリに収まる範囲で任意
sudo losetup -P /dev/loop0 ~/usb.img
sudo parted -s -a optimal /dev/loop0 mklabel msdos
sudo parted -s -a optimal /dev/loop0 mkpart primary fat16 0 100M  # 100MBのFAT領域
sudo parted -s -a optimal /dev/loop0 mkpart primary ext4 100M 100%  # 残りEXT4領域
sudo mkfs.vfat -F 16 /dev/loop0p1
sudo mkfs.ext4 /dev/loop0p2
sudo e2label /dev/loop0p2 usbroot
sudo mkdir /mnt/usbfat /mnt/usbroot
```

### 3. FAT領域にカーネルをコピー

```bash
sudo mount /dev/loop0p1 /mnt/usbfat
sudo cp ~/BPI-W2-bsp/SD/bpi-w2/BPI-BOOT/bananapi/bpi-w2/linux/{bpi-w2.dtb,uImage} /mnt/usbfat/
sudo umount /mnt/usbfat
```

### 4. EXT4領域にDebian構築

```bash
sudo mount /dev/loop0p2 /mnt/usbroot
sudo apt install binfmt-support debootstrap qemu-user-static
sudo debootstrap --arch arm64 --foreign bookworm /mnt/usbroot http://ftp.jp.debian.org/debian/
sudo mount -t proc proc /mnt/usbroot/proc
sudo update-binfmts --enable qemu-aarch64  # WSL2ではこうしないとchrootに失敗することがある
sudo chroot /mnt/usbroot  # ここからchroot環境
/debootstrap/debootstrap --second-stage
echo "quastation" > /etc/hostname
echo -e "127.0.1.1\tquastation" >> /etc/hosts
exit  # いったんchrootを抜ける
```

### 5. カーネルモジュールをコピー

```bash
sudo cp -a ~/BPI-W2-bsp/linux-rt/output/lib/modules /mnt/usbroot/lib
sudo chown -R root:root /mnt/usbroot/lib/modules
```

### 6. Debian構築の続き

```bash
sudo mount -t proc proc /mnt/usbroot/proc  # --second-stageでアンマウントされるため
sudo chroot /mnt/usbroot  # ここからchroot環境
apt update
apt upgrade
apt install locales
dpkg-reconfigure locales  # ここでロケールを適切に設定
dpkg-reconfigure tzdata  # ここでタイムゾーンを適切に設定
apt install iw network-manager ntp pciutils ssh sudo usbutils  # 起動後に使いそうなものを入れておく
apt clean
systemctl enable serial-getty@ttyS0.service  # シリアルコンソールで入れるように
echo "LABEL=usbroot / ext4 defaults,noatime,errors=remount-ro 0 1" >> /etc/fstab
echo "gpio_isr" > /etc/modules-load.d/gpio_isr.conf  # 各種ボタンを使えるように
useradd -G sudo -m -s /bin/bash qua  # ユーザーを1つだけ作っておく
echo "qua:quasta" | chpasswd  # 各自ユニークなものを。rootにパスワードは与えないので管理はsudoで
```

#### 無線モジュールを無効化。電波法遵守

```bash
echo "blacklist 8192ee" > /etc/modprobe.d/blacklist-8192ee.conf  # 2.4GHz
echo "blacklist 8812ae" > /etc/modprobe.d/blacklist-8812ae.conf  # 5GHz
echo -e "blacklist cdc_acm\nblacklist cdc_ether" > /etc/modprobe.d/blacklist-cdc.conf  # モデム(ttyACM,enx*)
```

#### (日本国外向け) MACアドレスを固定。P～Mは任意の16進数

```bash
echo "options 8192ee rtw_initmac=00:e0:4c:PQ:RS:TU" > /etc/modprobe.d/initmac-8192ee.conf
echo "options 8812ae rtw_initmac=00:e0:4c:GH:JK:LM" > /etc/modprobe.d/initmac-8812ae.conf
echo -e "[device]\nwifi.scan-rand-mac-address=no" > /etc/NetworkManager/conf.d/wifi-no-rand-mac-address.conf
```

#### udevの「予測可能な命名」が誤爆してMACアドレスが入れ替わらないようインターフェース名を固定

```bash
echo 'ACTION=="add", SUBSYSTEM=="net", DRIVERS=="?*", ATTR{address}=="'$(cut -d= -f2 /etc/modprobe.d/initmac-8192ee.conf)'", NAME="wlan8"' > /etc/udev/rules.d/50-fix-ifname-8192ee.rules
echo 'ACTION=="add", SUBSYSTEM=="net", DRIVERS=="?*", ATTR{address}=="'$(cut -d= -f2 /etc/modprobe.d/initmac-8812ae.conf)'", NAME="wlan9"' > /etc/udev/rules.d/50-fix-ifname-8812ae.rules
```

#### (オプション) システム起動完了時に電源LEDの点滅を止める

```bash
cat > /etc/systemd/system/power-led.service << 'EOS'
[Unit]
After=sshd.service

[Service]
ExecStart=/bin/bash -c 'echo none > /sys/class/leds/pwr_led_g/trigger'
ExecStop=/bin/bash -c 'echo timer > /sys/class/leds/pwr_led_g/trigger'
RemainAfterExit=yes
Type=oneshot

[Install]
WantedBy=default.target
EOS
```

```bash
systemctl enable power-led
```

#### (オプション) ネットワーク起動時にWLANのLEDを点灯

```bash
cat > /etc/NetworkManager/dispatcher.d/99-wifi-led << 'EOS'
#!/bin/bash
if [ "$2" = up ]; then
  echo 0 > /sys/class/leds/wifi_led_g/brightness
elif [ "$2" = down ]; then
  echo 255 > /sys/class/leds/wifi_led_g/brightness
fi
EOS
```

```bash
chmod +x /etc/NetworkManager/dispatcher.d/99-wifi-led
```

#### (オプション) 電源ボタン長押しでHalt、リセットボタンでReboot

```bash
echo 'DRIVER=="gpio_isr", ENV{button}=="POWER", RUN="/usr/bin/systemctl poweroff"' > /etc/udev/rules.d/10-power-button.rules
echo 'DRIVER=="gpio_isr", ENV{button}=="RESET", RUN="/usr/bin/systemctl reboot"' > /etc/udev/rules.d/10-reset-button.rules
```

電源については自動では落ちないのでLEDが全点灯状態になった後で電源プラグを抜く。
[参考1](https://github.com/tsukumijima/QuaStation-Ubuntu) の`/etc/fw_env.config.py`のハックをそのまま活用できると思うが
(`/dev/mmcblk?`の`?`は環境によるので`/dev/disk/by-path`とか見たほうがよいかも)ミスると起動不能になるかもしれないので省略。

### 7. 後始末

```bash
exit  # chrootを抜ける
sudo umount /mnt/usbroot/proc
sudo umount /mnt/usbroot
sudo losetup -d /dev/loop0
sudo rmdir /mnt/usbfat /mnt/usbroot
```

できた`usb.img`を [Rufus](https://rufus.ie/ja/) などのいわゆる「DDモード」で書き込めるツールでUSBメモリに書き込んで完成。  
U-bootからの起動手順は上述の参考サイトと同じなので省略。  
シリアルコンソールまたはsshからユーザーqua、パスワードquastaでログイン可能。

#### (日本国外向け)

```bash
$ sudo iw dev wlan8 scan | grep "SSID:"  # たぶん2.4GHz
$ sudo iw dev wlan9 scan | grep "SSID:"  # たぶん5GHz
$ sudo nmcli device wifi connect 【SSID】 password 【password】 ifname wlan8  # 成功すれば再起動後も自動でつながる
$ sudo nmcli connection delete id 【SSID】  # 接続情報は/etc/NetworkManager/system-connectionsに保存されるので接続不要になったら削除
```
