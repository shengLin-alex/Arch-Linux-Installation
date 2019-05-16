# Arch Linux 安裝隨筆
主要是寫給自己看的

## Make Arch Linux Live USB

前往 [archlinux.org](https://archlinux.org)下載官方 iso

```bash
dd bs=4M if=/path/to/archlinux.iso of=/dev/sdx && sync
```

/dev/sdx 為 usb 裝置位置

## 進入bios 用USB開機

進 bios用 F2(ASUS)

## 驗證UEFI

```bash
ls /sys/firmware/efi/efivars
```

有顯示東西的話表示正確

## 設定網路連線

這個部份用無線網卡處理

```bash
ifconfig -a
```

找到介面卡之後

```bash
ifconfig "介面卡名稱" up
wpa_passphrase "無線分享SSID" "密碼" >> /etc/wpa_supplicant/wpa_supplicant.conf
wpa_supplicant -B -i "介面卡名稱" -c /etc/wpa_supplicant/wpa_supplicant.conf
dhclient "介面卡名稱"
ping archlinux.org
```
註:以上指令雙引號不建議加

## 分割磁區

這邊跳過(MBR 用 fdisk；GPT用 gdisk)，基本上就是分:

1. swap (/dev/sda2)

2. / (/dev/sda5)

3. /home (/dev/sda3)

4. /boot (/dev/sda1)

5. /usr(/dev/sda4 註:不建議，後面解釋)

上面sda數字其實隨意，下面格式化磁區會按照這邊的數字處理

## 格式化磁區

首先格式化boot

```bash
mkfs -t vfat /dev/sda1
```

然後swap

```bash
mkswap /dev/sda2
```

最後分別是檔案目錄

```bash
mkfs -t ext4 /dev/sda3
mkfs -t ext4 /dev/sda4
mkfs -t ext4 /dev/sda5
```

## 掛載磁區

這邊重點就是先把 / 掛到 /mnt，然後根據你獨立分割出來的資料夾例如: /boot /home /usr 下去建立資料夾再掛載

```bash
mount /dev/sda5 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
mkdir /mnt/home
mount /dev/sda3 /mnt/home
mkdir /mnt/usr
mount /dev/sda4 /mnt/usr
```

## 安裝

安裝基礎套件至 /mnt

```bash
pacstrap /mnt base base-devel
```

## 建立 fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

必要的話用blkid 查詢 uuid，再用 vim 來檢查

```bash
blkid
vim /mnt/etc/fstab
```

關於 SSD Trim 指令等安裝完整個系統再改就好

## chroot 到新系統

```bash
arch-chroot /mnt
```

完成之後 root 就會換成新的系統根目錄 /

## 設定 pacman 的 mirrorlist

由於指定 mirrorlist server 位置可以加速套件安裝，建議進行以下設定

```bash
vim /etc/pacman.conf
```

將以下區塊加入指定 Server:

```
[core]
Server = http://archlinux.cs.nctu.edu.tw/$repo/os/$arch
Include = /etc/pacman.d/mirrorlist

[extra]
Server = http://archlinux.cs.nctu.edu.tw/$repo/os/$arch
Include = /etc/pacman.d/mirrorlist

[community]
Server = http://archlinux.cs.nctu.edu.tw/$repo/os/$arch
Include = /etc/pacman.d/mirrorlist
```

使用交大 mirrorlist 為首選

## 設定時區

這邊其實不確定有沒有成功，反正到時候安裝桌面環境之後再處理也行

```bash
ln -sf /usr/share/zoneinfo/Asia/Taipei /etc/localtime
hwclock --systohc
```

## 設定語言

這邊記得先設定英文就好，總之進桌面環境再改

```bash
echo "en_US.UTF-8 UTF-8" > /etc/locale.gen;
echo "LANG=en_US.UTF-8" > /etc/locale.conf;
locale-gen
```

## 設定電腦名稱

```bash
echo "your-pc-name" > /etc/hostname
```

然後安裝 vim (這邊安裝完桌面環境之後再處理也可以)

```bash
pacman -Sy vim
vim /etc/hosts
```

加入以下

```bash
127.0.0.1 localhost.localdomain localhost
::1 localhost.localdomain localhost
```

## 建立開機映像

```bash
mkinitcpio -p linux
```

## 設定 root

```bash
passwd
```

## 啟動載入程式與 grub

```bash
pacman -Sy grub os-prober efibootmgr
```

grub 用於建立開機選單
os-prober 用於偵測其他 OS(不打算雙系統的話這個就直接跳過)

```bash
os-prober
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```

## 安裝必要網路工具

```bash
pacman -S net-tools
pacman -S wireless_tools
pacman -S dhclient
pacman -S wpa_supplicant
```

## 重啟系統

離開 chroot 之前先啟動 dhcpd.service

```bash
systemctl enable dhcpd.service
exit
umount -R /mnt
reboot
```

## 初次進入系統

這個時候你會發現你只會進入 rootfs，沒錯這邊就是上面 /usr 獨立磁區的伏筆，當然如果你沒有分割 /usr 為獨立磁區則以下操作直接跳過

首先先測試 /new_root/sbin

```bash
ls -l /new_root/sbin
```

應該會說目錄不存在

根據 Arch Linux wiki:

```
/usr as a separate partition
If you keep /usr as a separate partition, you must adhere to the following requirements:

Add the fsck hook, mark /usr with a passno of 0 in /etc/fstab. While recommended for everyone, it is mandatory if you want your /usr partition to be fsck'ed at boot-up. Without this hook, /usr will never be fsck'd.
If not using the systemd hook, add the usr hook. This will mount the /usr partition after root is mounted.
```

解決方法:

1. 透過 USB 進入 chroot

2. 接著編輯 mkinitcpio.conf

```bash
vim /etc/mkinitcpio.conf
```

找到 HOOK 那一行，後面加入 shutdown usr fsck，完成後看起來會類似以下

```bash
HOOKS=(base udev autodetect sata filesystems shutdown usr fsck)
```

接著重新 mkinitcpio

```bash
mkinitcpio -p linux
```

最後編輯 fstab 將 /usr 那一行 passno(最後面的數字)改成 0，看起來會類似以下

```bash
UUID="some-uuid"	/usr      	ext4      	noatime,nodiratime,discard,rw,relatime	0 0
```

完成之後退出 chroot 重開機，應該會成功進入系統

## 初次進入系統(正式)

#### 安裝桌面環境

可以選擇 gnome 或 kde 等等，以下使用 gnome

```bash
pacman -Sy gnome gnome-extra
```

設定開機啟動

```bash
systemctl enable NetworkManager
systemctl enable gdm
```

#### 安裝 sudo

```bash
pacman -S sudo
```

設定群組

```bash
vim /etc/sudoers
```

找到以下這行(約82行)

```
# %wheel ALL=(ALL) ALL
```

把前面的 # 刪除

#### 建立使用者

```bash
useradd -m -u 1001 "your-user-name"
passwd "your-user-name"
usermod "your-user-name" -G wheel
```

## 重開機進入 gnome

```bash
reboot
```

## 安裝 yay

yay 用來處理 AUR 套件，建議安裝

```bash
sudo vim /etc/pacman.conf
```

找到以下(約93行)，並刪除前面 # ，兩行都要

```
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```

接著安裝必要套件

```bash
pacman -Sy yajl git
```

clone 套件安裝源

```bash
git clone https://aur.archlinux.org/package-query.git
git clone https://aur.archlinux.org/yay.git
```

安裝

```bash
cd package-query
makepkg -si
cd ../yay
makepkg -si
```

## 安裝 fcitx

fcitx 用於輸入法

```bash
yay -Sy fcitx-im
yay -S fcitx-chewing
yay -S fcitx-configtool
```

使用 vim(或其他編輯器開啟 /etc/profile)

```
sudo vim /etc/profile
```

在最下面加入以下三行

```bash
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```

開啟 Fcitx Configuration 圖形界面，新增 input method，找到 Chewing 並新增

## 安裝字形

由於內建無中文，所以中文會呈現亂碼，以下安裝繁體中文、簡體中文與日文(你懂的)
這邊可以自行去找喜歡的字體

```bash
yay -S adobe-source-han-sans-cn-fonts
yay -S adobe-source-han-sans-tw-fonts
yay -S adobe-source-han-sans-jp-fonts
```

## 安裝 ntfs 支援(optional)

由於 Linux 預設不支援 ntfs，如果有外接硬碟需求(你懂的)，記得安裝

```bash
yay -S ntfs-3g
```

## 設定雙顯卡(optional)

由於一般筆電有雙顯卡(Intel & Nvidia)，因此這裡進行一些額外設定
網路上爬到到教學基本上都是 bumblebee + bbswitch，然而設定方式極為複雜，以下提供一個較簡易的設定方式

#### 安裝 bumblebee 與 bbswitch

```bash
yay -S bumblebee bbswitch
sudo gpasswd -a "username" bumblebee
sudo systemctl enable bumblebeed.service
```

#### 安裝 nvidia

```bash
yay -S nvidia opencl-nvidia lib32-nvidia-utils lib32-opencl-nvidia mesa lib32-mesa-libgl xf86-video-intel
```

#### 設定 bumblebee

```bash
sudo vim /etc/bumblebee/bumblebee.conf
```

修改以下

```bash
Driver=nvidia

[driver-nvidia]
PMMethod=bbswitch
```

接著重開機

#### 檢測

```bash
yay -S virtualgl
optirun glxspheres64
```

會看到一個繪圖視窗輸出，表示成功，此時使用以下指令檢視 nvidia 繪圖卡工作中

```bash
nvidia-smi
```

關閉繪圖輸出視窗後再重新輸入

```bash
nvidia-smi
```

會發現 nvidia 自動關閉了，同時可以使用以下指令來手動強制啟動或關閉 nvidia

```bash
sudo tee /proc/acpi/bbswitch <<< ON
sudo tee /proc/acpi/bbswitch <<< OFF
```

檢查 bbswitch 狀態

```bash
cat /proc/acpi/bbswitch
```

## 設定硬體加速(optional)

由於 Linux 之下 chromium 預設不支援硬體加速，使用瀏覽器播放高清影片會造成高 CPU 使用率，以下設定方式參考自 [ArchWiki](https://wiki.archlinux.org/index.php/Hardware_video_acceleration)

#### 安裝相關套件

##### VAAPI

```bash
yay -S libva libva-utils libva-intel-driver libva-mesa-driver libva-vdpau-driver
```

##### VDPAU

```bash
yay -S libvdpau mesa-vdpau libvdpau-va-gl nouveau-fw vdpauinfo
```

##### 指定 driver

```bash
sudo gedit /etc/profile
```

參考下表，於 profile 最下面加入環境變數:

| graphics card | LIBVA_DRIVER_NAME | VDPAU_DRIVER |
|---------------|-------------------|--------------|
| Intel         | i965              | va_gl        |
| NVIDIA        | nouveau           | nouveau      |
| AMD           | radeonsi          | radeonsi     |

註: 以下只列出常見裝置設定值，詳細請至 [Hardware_video_acceleration](https://wiki.archlinux.org/index.php/Hardware_video_acceleration)參考

注意: 如果是內顯加獨顯的電腦，要指定使用獨顯的話需要再設定 DRI_PRIME=1，使用獨顯會導致耗電！

#### 重開機並檢測

```bash
reboot
```

重開機後於登入時選擇桌面環境指定使用 xorg，目前筆者發現預設使用的 XDG_SESSION 為 wayland，因不明原因導致硬體加速失敗

1. 檢查 XDG_SESSION_TYPE

```bash
echo $XDG_SESSION_TYPE
```

xorg 會顯示 x11

2. 檢查 VPAPI & VDPAU

```bash
vainfo
vdpauinfo
```

#### 安裝 chromium

##### chromium-vaapi-bin

由於普通版本的 chromium 無法支援硬體加速，因此社群提供了 vaapi patch 版本的 chromium

```bash
yay -S chromium-vaapi-bin
```

##### h264ify

由於 Linux 版本 chromium 啟用硬體解碼之後無法解碼 VP8/VP9 之 4K 60FPS 影片，解法如下:

前往 chrome 擴充套件商店安裝 h264ify

##### 設定 chromium-flags

```bash
gedit ~/.config/chromium-flags.conf
```

加入

```
--enable-accelerated-mjpeg-decode
--enable-accelerated-video
```

#### 檢測 chromium

1. 前往 youtube 播放 4k 60fps 影片

2. 開一個 tab 前往 chrome://media-internals，找到開影片 blob 網頁的項目點開檢查:

| property         | value            |
|------------------|------------------|
| video_codec_name | h264             |
| video_decoder    | MojoVideoDecoder |

## 初次備份

建議先對系統進行一次基本備份，畢竟目前處於最乾淨的狀態，先安裝 rsync，再備份

```bash
yay -S rsync
rsync -aAXv /* /path/to/backup/folder --exclude={/dev/*,/proc/*,/sys/*,/tmp/*,/run/*,/mnt/*,/media/*,/lost+found}
```

## 完成

到這邊基本上已經完成安裝，剩下的遇到再說
