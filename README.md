# Arch Linux 安裝隨筆
主要是寫給自己看的

## Make Arch Linux Live USB
```bash
dd bs=4M if=/path/to/archlinux.iso of=/dev/sdx && sync
```
## 進入bios 用USB開機
進bios用F2(ASUS)
### 驗證UEFI
```bash
ls /sys/firmware/efi/efivars
```
有顯示東西的話表示正確
### 設定網路連線
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
### 分割磁區
這邊跳過(反正就是用 fdisk)，基本上就是分:
1. swap (/dev/sda2)
2. / (/dev/sda5)
3. /home (/dev/sda3)
4. /boot (dev/sda1)
5. /usr(/dev/sda4 註:不建議，後面解釋)
上面sda數字其實隨意，下面格式化磁區會按照這邊的數字處理

### 格式化磁區
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
mkfs -t ext4  /dev/sda3
mkfs -t ext4  /dev/sda4
mkfs -t ext4  /dev/sda5
```
### 掛載磁區
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
### 安裝
安裝基礎套件至 /mnt
```bash
pacstrap /mnt base base-devel
```
### 建立 fstab
