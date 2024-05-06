# ArchLinux + KDE 桌面環境安裝筆記

## 1. 製作開機隨身碟
1. 至[Arch官網](https://archlinux.org/download/)下載臺灣 ArchLinux ISO 檔案
2. 使用[Etcher](https://www.balena.io/etcher/)將ISO檔案寫入隨身碟

## 2. 安裝 ArchLinux

### 2.1 開機隨身碟安裝

1. 隨身碟插著電腦，開機按Delete進入BIOS，關閉Secure Boot。 
2. 調整BIOS開機順序，以UEFI模式隨身碟開機，進入Arch Linux，用鍵盤選第一個選項，按Enter進入安裝媒體。
3. 載入Arch Linux系統後會進入終端機(顯示root@archiso)，系統應該會自動連上網路。  
ping看看Arch Linux檢查網路是否正常:

```bash
ping -c 3 archlinux.org
```
4. 檢查是否為UEFI模式開機，應會回傳「64」。若顯示No such file or directory的話，輸入poweroff關機，退回BIOS啟用UEFI再重新來過。

```bash
cat /sys/firmware/efi/fw_platform_size
```

### 2.2 分割磁碟

1. 查看目前的硬碟分區

```bash
fdisk -l
```

2. 選取要安裝系統的硬碟

```bash
fdisk /dev/sda | /dev/nvme0n1
```

3. (可選擇) 輸入`g`刪除全部分區，建立GPT分割表。
4. 建立EFI分割區

```bash
n # 新增分割區
1 # 第一個分割區
First Sector: (Enter) # 預設
Last Sector: +512M # 512MB大小
# 遇到Do you want to remove the signature?的問題就輸入yes
t # 更改分割區類型
1 # 第一個分割區
uefi # 更改成EFI分割區
```

5. 建立SWAP分割區(交換分區，建議為記憶體的2倍)

```bash
n # 新增分割區
2 # 第二個分割區
First Sector: (Enter) # 預設
Last Sector: +32G # 若記憶體為16G，則建立32G的SWAP分割區
t # 更改分割區類型
2 # 第二個分割區
swap # 更改成SWAP分割區
```

6. 建立ROOT分割區

```bash
n # 新增分割區
3 # 第三個分割區
First Sector: (Enter) # 預設
Last Sector: (Enter) # 預設
t # 更改分割區類型
3 # 第三個分割區
linux # 更改成Linux分割區
```

7. 輸入`p`檢查分割區是否正確，輸入`w`寫入分割區表並離開。

```bash
# 分割區應呈現
/dev/sda1 512M EFI
/dev/sda2 32G SWAP
/dev/sda3 XXXG Linux
```

8. 格式化分割區，EFI分割區格式化為FAT32，ROOT分割區格式化為EXT4，SWAP分割區格式化為SWAP。

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda3
mkswap /dev/sda2
swapon /dev/sda2
```

9. 掛載分割區至`/mnt`目錄

```bash
mount /dev/sda3 /mnt
```

### 2.3 安裝基本系統

1. 掛載EFI分割區至`/mnt/boot`目錄

```bash
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

2. 用`pacstrap`安裝Linux基本系統

```bash
pacstrap -K /mnt base linux linux-firmware
```

3. 用genfstab工具設定開機後硬碟掛載規則

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

4. 檢視掛載規則是否正確

```bash
cat /mnt/etc/fstab
```

5. 進入新系統

```bash
arch-chroot /mnt
```

### 2.4 安裝驅動程式

#### 2.4.1 顯示卡驅動程式

Intel顯示卡的驅動含在開源mesa套件，開箱即用，通常不需要特別安裝。下面是包含Vulkan驅動的套件：

```bash
pacman -S intel-media-driver vulkan-intel
```

AMD顯示卡的驅動跟Intel一樣，含在mesa套件。有需要Vulkan再額外安裝：

```bash
pacman -S vulkan-radeon libva-mesa-driver mesa-vdpau
```

至於Nvidia顯示卡，Arch的儲存庫有提供Nvidia顯示卡的專有驅動，不需要額外加套件庫。安裝後nouveau會自動被停用。

```bash
pacman -S linux-headers nvidia-dkms nvidia-settings
```

Nvidia顯示卡(驅動版本 > 545)的用戶建議編輯Nvdiia核心模組參數，啟用DRM框架，KDE Wayland才會有畫面。

```bash
mkdir /etc/modprobe.d
echo "options nvidia_drm modeset=1 fbdev=1" >>  /etc/modprobe.d/nvidia.conf
```

#### 2.4.2 無線網卡驅動程式

用`lsci`查看硬體裝置的型號

```bash
lspci | egrep -i 'wifi|wireless|intel|broadcom|realtek'
```

如果網卡是用USB外接的，使用`lsusb`指令檢查

```bash
pacman -S usbutils
lsusb
```

它應該會印出一組`英數:英數`的代碼，到 [Linux Wireless wiki](https://wireless.wiki.kernel.org/en/users/drivers) 查詢該裝置有無驅動可用。

如果你運氣好，裝好`linux-firmware`套件，有含在主線Linux核心的專有驅動就會在開機後自動載入。

```bash
pacman -S linux-firmware
```

運氣不好，裝驅動得用到AUR甚至DKMS編譯，那請後面開機再處理吧。

### 2.6 安裝KDE桌面環境

1. 安裝KDE桌面環境以及常用套件

```bash
pacman -S sudo networkmanager vim firefox noto-fonts-cjk noto-fonts-emoji
pacman -S xorg xorg-server pipewire wireplumber pipewire-pulse intel-ucode nvtop
pacman -S sddm plasma-meta kde-applications packagekit-qt6
pacman -S fcitx5-im fcitx5-chewing fcitx5-qt fcitx5-gtk fcitx5-chinese-addons
pacman -S git openssh fakeroot base-devel
```

### 2.7 設定系統

1. 設定時區

```bash
ln -sf /usr/share/zoneinfo/Asia/Taipei /etc/localtime
hwclock --systohc
```

2. 編輯語言檔案，取消`zh_TW.UTF-8`的註解

```bash
vim /etc/locale.gen
```

3. 產生語言檔案，設定系統語言為正體中文

```bash
locale-gen
echo "LANG=zh_TW.UTF-8" >> /etc/locale.conf
```

4. 設定主機名稱(在此以ArchLinux為例)

```bash
echo "ArchLinux" >> /etc/hostname
```

5. 設定host檔案

```bash
echo "127.0.0.1 localhost" >> /etc/hosts
echo "::1 localhost" >> /etc/hosts
echo "127.0.1.1 ArchLinux" >> /etc/hosts # 主機名稱
```

6. 開機時啟動NetworkManager和SDDM

```bash
systemctl enable NetworkManager.service
systemctl enable sddm.service
systemctl enable sshd.service # 遠端連線(可選)
```

7. 設定root密碼

```bash
passwd
# 輸入密碼兩次
```

8. 建立新使用者username(可自選名稱)

```bash
useradd -m -g users -G wheel,audio,video,storage -s /bin/bash username
passwd username
# 輸入密碼兩次
```

9. (可選) 編輯sudoers檔案，使username(使用者名稱)可以使用sudo權限

```bash
username ALL=(ALL) ALL
```

### 2.8 安裝開機引導程式

1. 安裝GRUB引導程式

```bash
pacman -S grub efibootmgr
```

2. 將EFI分區掛載到`/boot`目錄

```bash
mount /dev/sda1 /boot
```

3. 安裝GRUB至EFI分區

```bash
grub-install --target=x86_64-efi --bootloader-id=GRUB --efi-directory=/boot
grub-mkconfig -o /boot/grub/grub.cfg
```

4. 檢查`/boot`目錄是否有安裝GRUB和Linux核心，應會列出`grub`目錄和`initramfs-linux.img`。

```bash
ls /boot
```

5. 安裝完畢，離開chroot環境，取消掛載，重新開機

```bash
unmount /mnt/boot
unmount /mnt
shutdown now
```

6. 移除隨身碟，重新開機，進入GRUB引導程式，選擇ArchLinux進入系統。

## 3. 系統後續設定

如果Arch Linux開機後沒畫面，按CTRL+ALT+F2切換至tty，登入root帳號後再除錯。或是用開機隨身碟chroot到系統進行修復(重複上文2.4的步驟)。

一切順利的話，應該會看到登入畫面，輸入密碼後就能登入KDE桌面，下面是一些小優化。

### 3.1 安裝AUR套件

1. 安裝`yay`套件管理程式

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

2. 編輯：`sudo vim /etc/makepkg.conf`，找到`MAKEFLAGS="-j2"`這行，取消註解，再將後面改成`"-j$(nproc)"`。這樣在編譯AUR套件時即會使用全部CPU。
3. 編輯：`sudo vim /etc/pacman.conf`，取消註解`Color`和`ParallelDownloads`，開啟顏色和平行下載套件。

### 3.2 安裝Flatpak

1. 安裝Flatpak

```bash
sudo pacman -S flatpak
flatpak remote-add --user --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

### 3.3 中文輸入法

1. 確認已安裝必要的Fcitx5套件

```bash
sudo pacman -S fcitx5-im fcitx5-chewing fcitx5-qt fcitx5-gtk fcitx5-chinese-addons
```

2. 編輯`sudo vim /etc/environment`，加入以下內容

```bash
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
GLFW_IM_MODULE=fcitx
```

### 3.4 設定防火牆

1. 安裝UFW防火牆

```bash
sudo pacman -S ufw
sudo ufw enable
sudo systemctl enable ufw
sudo systemctl start ufw
```

2. 設定防火牆規則

```bash
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh # 遠端連線(可選)
sudo ufw reload
```

### 3.5 補齊缺少的驅動程式

最簡單的方式是重裝Linux核心

```bash
sudo pacman -S linux linux-firmware
```

如果還是不行，可以從AUR補齊驅動程式

```bash
yay -S mkinitcpio-firmware
```

### 3.6 常用軟體安裝

#### 3.6.1 程式開發

##### Alacritty

```bash
sudo pacman -S alacritty ttf-ubuntu-mono-nerd # 終端機、字型
```

設定Alacritty字型

```bash
vim ~/.config/alacritty/alacritty.toml

# 加入以下內容

[font.normal]
family = "UbuntuMono Nerd Font"
style = "Regular"

[font.bold]
family = "UbuntuMono Nerd Font"
style = "Bold"

[font.italic]
family = "UbuntuMono Nerd Font"
style = "Italic"

[font]
size = 14
```

##### Neovim

```bash
sudo pacman -S gcc make git ripgrep fd unzip neovim lazygit
git clone https://github.com/ZongBen/kickstart.nvim.git "${XDG_CONFIG_HOME:-$HOME/.config}"/nvim
```

##### Node.js

```bash
yay -S nvm
```

##### Docker

```bash
sudo pacman -S docker docker-compose
sudo systemctl enable docker # 開機啟動
sudo systemctl start docker # 啟動服務
sudo usermod -aG docker username # 將使用者加入docker群組
```

##### Dotnet

```bash
sudo pacman -S dotnet-sdk dotnet-runtime aspnet-runtime
```

#### 3.6.2 日常應用

##### Steam

取消註解`[multilib]`和`Include = /etc/pacman.d/mirrorlist`，再更新套件庫

```bash
sudo vim /etc/pacman.conf
```

下載Steam
```bash
sudo pacman -S steam
```

若Steam下載速度緩慢
```bash
sudo vim /home/username/.steam/steam/steam_dev.cfg

# 加入以下內容
@nClientDownloadEnableHTTP2PlatformLinux 0
@fDownloadRateImprovementToAddAnotherConnection 1.0
```

##### Bottles

Bottles是一個輕量級的Wine前端，可以讓你在Linux上執行Windows應用程式。

```bash
flatpak remote-add --user --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo # 執行後重開機
flatpak install flathub com.usebottles.bottles
```

##### LINE

使用Bottles安裝LINE後，需要的相依套件

```bash
cjkfonts、vcredist2012、d3dcompiler_46、iertutil
```

##### OpenVpn

```bash
sudo pacman -S openvpn networkmanager-openvpn
```


