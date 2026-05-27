# 遷移過程紀錄

- 單純紀錄遷移過程，非教學。
- 原先目標是在不使用 USB 隨身碟或外部安裝媒體的情況下，直接從既有 Ubuntu 系統引導 Arch Linux ISO，並完成系統替換。實際過程中，前半段確實完成了 Direct ISO Boot、`copytoram`、重新分割硬碟與 `pacstrap` 安裝；但在重開機後，由於 UEFI 韌體未能正確識別新安裝的 GRUB 開機項目，最後仍使用 Rufus 製作 Arch Linux USB 安裝媒體，進入 Live 環境後重新 `arch-chroot` 修復 GRUB。

> **警告：會清除原本硬碟上的 Ubuntu 系統與所有資料**
>
> 執行前確認：
>
> 1. 資料已備份。
> 2. 系統可能在中途無法開機，需要使用外部 Live USB 修復。
> 3. 磁碟名稱此處以 `/dev/sda` 為例。

---

## 1. 安裝環境 & 成果

原環境：
- Original OS：Ubuntu 24.04 LTS
- Target OS：Arch Linux
- Laptop：Samsung Notebook 7 Spin (2019)
- 原目標：
  - 不使用 USB 隨身碟
  - 不使用外部儲存媒體
  - 直接從 Ubuntu 內部引導 Arch ISO
  - 完成硬碟重新分割與 Arch Linux 安裝

實際結果：
- [x] 從 Ubuntu 準備 Arch ISO
- [x] 透過 GRUB 嘗試 Direct ISO Boot
- [x] 使用 `copytoram=y` 讓 Arch Live 環境在 RAM 中執行
- [x] 重新分割與格式化原本 Ubuntu 所在硬碟
- [x] 使用 `pacstrap` 安裝 Arch Linux 基礎系統
- [ ] 第一次重開機後，筆電 UEFI 無法找到可用開機項目，出現 `All boot options are tried`
- [ ] 最後使用 Rufus 製作 Arch Linux USB 安裝媒體
- [ ] 透過 USB Live 環境進入系統後，重新掛載硬碟並 `arch-chroot`
- [ ] 使用 `grub-install --removable` 補上 UEFI fallback boot path
- [ ] 最終成功從硬碟直接啟動 Arch Linux

---

## 2. 計畫趕不上變化

1. 原本計畫**無 USB 安裝**：
    1. 在 Ubuntu 中下載 Arch Linux ISO
    2. 將 ISO 放在 Ubuntu 硬碟中，例如 `/boot/archlinux-x86_64.iso`
    3. 修改 GRUB 啟動參數，直接從硬碟上的 ISO 啟動 Arch Live
    4. 使用 `copytoram=y` 將 Live 環境複製到 RAM
    5. 在 Live 環境中重新分割硬碟並安裝 Arch
2. 預期：
    *只要 Arch Live 系統完整載入 RAM，理論上原本存放 ISO 的硬碟就可以被重新分割與格式化。*

實際上確實可以進到 `pacstrap` 完成；問題在重開機後，新安裝的 GRUB 沒有被 UEFI 韌體正確識別，導致系統無法從硬碟開機。
因此最後使用 [Rufus](https://rufus.ie/zh_TW/) 製作 USB 安裝媒體，目的不是重新安裝 Arch，而是進入一個可用的 Arch Live 環境，對已經安裝好的硬碟系統進行開機修復。

---

## 3. 過程中遇到的主要問題與原因

### 3.1 `ERROR: '' device did not show up...`

這個錯誤通常出現在 Arch ISO 沒有被正確掛載，或 Arch initramfs 找不到 ISO 所在裝置時。

原因有二：

1. GRUB 設定中的 ISO 路徑錯誤  
   原本的路徑中多寫了不存在的 `/grml/` 目錄，導致 GRUB 無法正確找到 Arch ISO。

2. 核心啟動參數缺少正確的 ISO 所在裝置  
   在 Direct ISO Boot 的情況下，需要指定：

   ```text
   img_dev=/dev/disk/by-uuid/[UUID]
   ```

   如果沒有正確指定 ISO 所在分割區，Arch initramfs 在開機階段就無法找到安裝映像檔，最後會掉進 initramfs shell。

---

### 3.2 在 initramfs/rootfs 中執行 `iwctl` 失敗

曾經在 rootfs/initramfs 階段嘗試執行 `iwctl`，但出現類似：

```text
Inappropriate ioctl for device
```

或與 D-Bus 相關的錯誤。

原因：
當系統掉進 initramfs 的極簡環境時，D-Bus、systemd、NetworkManager、iwd 等服務通常都尚未正常啟動。`iwctl` 這類工具需要依賴完整的使用者空間服務，因此在 initramfs 階段無法正常工作。

正確作法是：  
不要在錯誤的 initramfs shell 中繼續嘗試連線，而是修正 GRUB 與 ISO 啟動參數，確保系統能進入完整的 Arch Live 環境。

---

### 3.3 Ubuntu GUI 無法開啟新終端機

在嘗試將系統環境轉移至 RAM 後，Ubuntu GUI 仍可能短暫停留在畫面上，但無法再開啟新的終端機或應用程式。

原因：
系統根目錄與硬碟之間的關係被改變後，原本GUI所需的執行檔、動態連結庫與桌面服務可能已無法從硬碟正常讀取。
因此，一旦進入這種狀態，應改用 TTY 純文字介面操作，例如：

```text
Ctrl + Alt + F3
```

---

### 3.4 重開機後出現 `All boot options are tried`

在完成 Arch 安裝並重開機後出現：

```text
All boot options are tried
```

表示 UEFI 韌體沒有找到可用的開機項目。

可能原因：

1. 原本 Ubuntu 的 GRUB 已經在重新分割硬碟時被清除。
2. 新安裝的 Arch GRUB 雖然已寫入硬碟，但 UEFI NVRAM 開機項目沒有被成功建立或沒有被韌體正確識別。
3. 部分品牌筆電的 UEFI 韌體對 NVRAM 開機項目的處理較不穩定，可能需要使用 fallback boot path。
4. 若 BIOS/UEFI 中啟用了 Fast Boot 或類似選項，也可能導致硬碟或 EFI 開機項目偵測不完整。

需要注意的是：
這不代表 Arch Linux 沒有安裝成功。當時硬碟中的 Arch 系統實際上已經存在，只是 UEFI 韌體沒有成功找到 GRUB 開機程式。

---

## 4. 最後仍然需要 Rufus 製作 USB ISO

原本的無 USB 流程在安裝階段是可行的，但失敗點出現在重開機後的開機鏈。

重新分割硬碟時：

1. 原本 Ubuntu 的分割表被重建。
2. 原本 Ubuntu 的 GRUB 被清除。
3. 原本存放在 Ubuntu 硬碟中的 Arch ISO 也不再可用。
4. 雖然 Arch Live 環境曾經透過 `copytoram=y` 存活在 RAM 中，但一旦重開機，RAM 內容就會消失。

因此，當第一次重開機失敗後，系統沒有任何可用的本機引導環境：

- 原本 Ubuntu 已不存在
- 原本硬碟上的 ISO 已不存在
- 新 Arch 的 GRUB 尚未被 UEFI 正確識別
- RAM 中的 Live 環境已因重開機消失

此時最實際的修復方式就是使用外部 Live USB，因此最後使用 Rufus 製作 Arch Linux USB 安裝媒體，用途：

1. 提供一個可以開機的 Live Arch 環境
2. 掛載已經安裝好的 Arch Linux root 分割區
3. 進入 `arch-chroot`
4. 重新安裝 GRUB
5. 補上 UEFI fallback boot path
6. 讓筆電可以從硬碟直接啟動 Arch Linux

---

## 5. `--removable` 的作用

修復關鍵：

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --removable --recheck
```

其中 `--removable` 的作用是建立 UEFI fallback boot path

標準 UEFI fallback path 通常是：

```text
EFI/BOOT/BOOTX64.EFI
```

也就是在這次掛載方式下：

```text
/boot/EFI/BOOT/BOOTX64.EFI
```

路徑意義：
即使 UEFI NVRAM 中沒有成功建立 `GRUB` 或 `Arch Linux` 的開機項目，韌體仍可能在偵測磁碟時嘗試讀取這個標準 fallback 路徑。

因此，對於 UEFI 開機項目寫入不穩定、NVRAM 註冊失敗，或韌體相容性不佳的筆電，`--removable` 是一個有效的修復方法。

---

## 6. 核心概念：Direct ISO Boot 與 `copytoram`

前半流程關鍵是透過 GRUB 直接引導硬碟中的 Arch Linux ISO，而不是一開始就使用 USB 隨身碟。

Arch ISO 啟動時可使用：

```text
copytoram=y
```

它會將 Live 系統的 root filesystem 複製到 RAM 中執行。  
完成後，系統就不再依賴原本存放 ISO 的硬碟分割區，因此可以重新分割與格式化硬碟。

範例 GRUB 啟動參數：

```text
linux (loop)/arch/boot/x86_64/vmlinuz-linux img_dev=/dev/disk/by-uuid/[UUID] img_loop=/boot/archlinux-x86_64.iso archisobasedir=arch archisolabel=ARCH_202605 copytoram=y cow_spacesize=4G nomodeset text
initrd (loop)/arch/boot/x86_64/initramfs-linux.img
```

其中：

- `img_dev=`
  指定 ISO 所在的實體分割區
- `img_loop=`
  指定 ISO 檔案在該分割區中的路徑
- `archisobasedir=arch`
  指定 Arch ISO 內部基礎目錄
- `archisolabel=ARCH_202605`
  指定 ISO 標籤，需要與實際 ISO 版本一致
- `copytoram=y`
  將 Live 系統複製到 RAM 中執行
- `cow_spacesize=4G`
  指定 copy-on-write 可用空間大小
- `nomodeset`
  避免部分顯示卡或舊筆電在 Live 環境啟動時發生黑畫面
- `text`
  優先進入文字模式，降低圖形介面造成的變數
---

## 7. 在 Ubuntu 中準備 RAM 環境

目標是在 Ubuntu 執行中建立一個暫時的 RAM root，並將必要工具與 Arch ISO 複製進去。

1. 切換為 root：
    ```bash
    sudo -i
    ```

2. 建立 RAM root：
    ```bash
    mkdir /tmp/ramroot
    mount -t tmpfs -o size=4G tmpfs /tmp/ramroot
    ```

3. 建立基本目錄結構：
    ```bash
    mkdir -p /tmp/ramroot/{dev,proc,sys,run,sbin,bin,etc,lib,lib64,usr}
    ```
4. 複製必要指令：
    ```bash
    cp -a /bin/{bash,mount,umount,ls,cat,mkdir,rm,cp,ln,grep,sed} /tmp/ramroot/bin/
    cp -a /sbin/{fdisk,mkfs*,losetup} /tmp/ramroot/sbin/ 2>/dev/null || true
    ```
5. 複製必要的動態連結庫：
    ```bash
    cp -a /lib/* /tmp/ramroot/lib/ 2>/dev/null || true
    cp -a /lib64/* /tmp/ramroot/lib64/ 2>/dev/null || true
    cp -a /usr/lib* /tmp/ramroot/usr/ 2>/dev/null || true
    ```
6. 將 Arch ISO 複製到 RAM root 中：
    ```bash
    mkdir -p /tmp/ramroot/boot
    cp /boot/archlinux-x86_64.iso /tmp/ramroot/boot/
    ```

---

## 8. 切換至 RAM root

1. 將必要的虛擬檔案系統 bind mount 到新的 RAM root：
    ```bash
    mount --bind /dev /tmp/ramroot/dev
    mount --bind /proc /tmp/ramroot/proc
    mount --bind /sys /tmp/ramroot/sys
    mount --bind /run /tmp/ramroot/run
    ```
2. 建立舊 root 掛載點：
    ```bash
    mkdir -p /tmp/ramroot/oldroot
    ```
3. 切換到新的 RAM root：
    ```bash
    cd /tmp/ramroot
    pivot_root . oldroot
    ```
4. 在新的 root 環境中解除舊 root：
    ```bash
    chroot . /bin/bash -c "umount -l /oldroot"
    ```
    此時原本的 Ubuntu 圖形環境可能已經無法正常開啟新程式。  
    如果無法繼續操作，可以切換到 TTY，或重新開機並進入 GRUB 編輯模式，使用 Direct ISO Boot 參數啟動 Arch Live ISO。

---

## 9. 進入 Arch Live 環境

成功進入 Arch Live 環境後，應看到類似下列提示字元：

```text
root@archiso ~ #
```

1. 確認網路是否可用：
    ```bash
    ping -c 3 archlinux.org
    ```

2. 如果使用 Wi-Fi，可使用：
    ```bash
    iwctl
    ```
    或在有 NetworkManager 的環境下使用：
    ```bash
    nmtui
    ```

---

## 10. 確認目標磁碟

重新分割前，必須確認目標磁碟名稱。

1. 列出磁碟：
    ```bash
    lsblk
    ```
    常見磁碟名稱：
    ```text
    /dev/sda
    /dev/nvme0n1
    ```
    以下以 `/dev/sda` 為例。

    如果你的磁碟是 NVMe，後續分割區名稱通常會是：
    ```text
    /dev/nvme0n1p1
    /dev/nvme0n1p2
    ```
    而不是：
    ```text
    /dev/sda1
    /dev/sda2
    ```

---

## 11. 重新分割硬碟

執行：

```bash
fdisk /dev/sda
```

在 `fdisk` 中嚴格依序操作以下細項：

### 11.1 建立新的 GPT 分割表

輸入：

```text
g
```

這會建立新的 GPT 分割表，原本硬碟上的分割資訊會被清除。

---

### 11.2 建立 EFI System Partition

輸入：

```text
n
```

Partition number 直接按 Enter
First sector 直接按 Enter
Last sector 輸入：

```text
+1G
```

這會建立第一個分割區 `/dev/sda1`，大小為 1GB，用於 UEFI 開機。

如果出現下列提示：

```text
Partition #1 contains a vfat signature. Do you want to remove?
```

請輸入：

```text
Y
```

以清除舊有檔案系統簽章。

---

### 11.3 建立 root 分割區

再次輸入：

```text
n
```

Partition number 直接按 Enter
First sector 直接按 Enter
Last sector 直接按 Enter

這會將剩餘空間全部分配給第二個分割區 `/dev/sda2`，作為 Arch Linux root filesystem。

---

### 11.4 設定第一分割區為 EFI System

輸入：

```text
t
```

選擇 partition number：

```text
1
```

Partition type 輸入：

```text
1
```

將第一分割區設定為：

```text
EFI System
```

---

### 11.5 寫入分割表

確認無誤後，輸入：

```text
w
```

將分割表寫入硬碟並離開 `fdisk`。

---

## 12. 格式化分割區

1. 格式化 EFI 分割區為 FAT32：
    ```bash
    mkfs.vfat -F 32 /dev/sda1
    ```
2. 格式化 root 分割區為 ext4：
    ```bash
    mkfs.ext4 /dev/sda2
    ```

- 若使用 NVMe，改成類似：
    ```bash
    mkfs.vfat -F 32 /dev/nvme0n1p1
    mkfs.ext4 /dev/nvme0n1p2
    ```

---

## 13. 掛載分割區

1. 掛載 root 分割區：
    ```bash
    mount /dev/sda2 /mnt
    ```
2. 建立 `/mnt/boot`：
    ```bash
    mkdir -p /mnt/boot
    ```
3. 掛載 EFI 分割區：
    ```bash
    mount /dev/sda1 /mnt/boot
    ```
- 若使用 NVMe，改成：
    ```bash
    mount /dev/nvme0n1p2 /mnt
    mkdir -p /mnt/boot
    mount /dev/nvme0n1p1 /mnt/boot
    ```

---

## 14. 安裝 Arch Linux 基礎系統

執行：

```bash
pacstrap -K /mnt base base-devel linux linux-firmware nano networkmanager grub efibootmgr
```

套件用途：
- `base`：Arch Linux 基礎系統
- `base-devel`：基本編譯工具
- `linux`：Linux kernel
- `linux-firmware`：硬體韌體
- `nano`：文字編輯器
- `networkmanager`：網路管理工具
- `grub`：開機管理器
- `efibootmgr`：UEFI 開機項目管理工具

---

## 15. 產生 fstab

1. 安裝完基礎系統後，產生 `/etc/fstab`：
    ```bash
    genfstab -U /mnt >> /mnt/etc/fstab
    ```
2. 建議檢查一次內容：
    ```bash
    cat /mnt/etc/fstab
    ```

確認 root 與 boot 分割區都有正確列入。

---

## 16. 進入新系統環境

使用 `arch-chroot` 進入剛安裝好的系統：

```bash
arch-chroot /mnt
```

後續指令都會在新 Arch Linux 系統內執行。

---

## 17. 設定時區
1. 設定時區為台北：
    ```bash
    ln -sf /usr/share/zoneinfo/Asia/Taipei /etc/localtime
    ```
2. 同步硬體時鐘：
    ```bash
    hwclock --systohc
    ```

---

## 18. 設定語系

1. 編輯 `/etc/locale.gen`：
    ```bash
    nano /etc/locale.gen
    ```
2. 找到並取消註解以下兩行：
    ```text
    en_US.UTF-8 UTF-8
    zh_TW.UTF-8 UTF-8
    ```
3. 儲存並離開 nano：
    ```text
    Ctrl + O
    Enter
    Ctrl + X
    ```
4. 產生 locale：
    ```bash
    locale-gen
    ```
5. 設定系統預設語言為英文，以避免純文字 TTY 環境因中文字型不足而顯示亂碼：
    ```bash
    echo "LANG=en_US.UTF-8" > /etc/locale.conf
    ```

---

## 19. 設定主機名稱

設定 hostname，此處以`arch-laptop`為例：

```bash
echo "arch-laptop" > /etc/hostname
```

---

## 20. 設定 root 密碼

執行：

```bash
passwd
```

依提示輸入兩次新的 root 密碼（輸入密碼時畫面不會顯示任何字元）。

---

## 21. 第一次安裝 GRUB

1. 在 UEFI 系統上安裝 GRUB：
    ```bash
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --recheck
    ```
2. 產生 GRUB 設定檔：
    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
3. 啟用 NetworkManager：
    ```bash
    systemctl enable NetworkManager
    ```
4. 離開 chroot：
    ```bash
    exit
    ```
5. 解除掛載：
    ```bash
    umount -R /mnt
    ```
6. 重新開機：
    ```bash
    reboot
    ```

---

## 22. 第一次重開機失敗

重開機後，筆電沒有成功進入 GRUB，而是出現：

```text
All boot options are tried
```

這表示 UEFI 韌體沒有找到可用開機項目。

此時原本 Ubuntu 已被清除，硬碟中的 Arch ISO 也因重新分割而不再存在，因此無法再依靠原本的 Direct ISO Boot 流程修復。

解法是使用另一台可用電腦，透過 Rufus 製作 Arch Linux USB 安裝媒體。

---

## 23. 使用 Rufus 製作 Arch Linux USB 安裝媒體

在 Windows 電腦上準備：

1. 下載 Arch Linux ISO
2. 下載並開啟 Rufus
3. 選擇 USB 隨身碟
4. 選擇 Arch Linux ISO
5. 分割區配置通常選擇：
   - GPT
   - UEFI
6. 開始寫入

- 完成後將 USB 插入目標，進入 Boot Menu 或 BIOS/UEFI，從 USB 啟動。

---

## 24. 從 USB Live 環境修復 GRUB
1. 進入 Arch USB Live 環境後，確認硬碟分割區：
    ```bash
    lsblk
    ```

    假設：
    - EFI 分割區是 `/dev/sda1`
    - Arch root 分割區是 `/dev/sda2`
2. 掛載 root 分割區：
    ```bash
    mount /dev/sda2 /mnt
    ```
3. 掛載 EFI 分割區：
    ```bash
    mount /dev/sda1 /mnt/boot
    ```
4. 進入已安裝的 Arch 系統：
    ```bash
    arch-chroot /mnt
    ```
5. 重新安裝 GRUB，這次加入 `--removable`：
    ```bash
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --removable --recheck
    ```
6. 重新產生 GRUB 設定檔：
    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
7. 離開 chroot：
    ```bash
    exit
    ```
8. 解除掛載：
    ```bash
    umount -R /mnt
    ```
9. 重新開機：
    ```bash
    reboot
    ```
- 拔除 USB 後，系統應可從硬碟直接進入 Arch Linux。

---

## 25. 第一次成功登入 Arch Linux

重新開機後，若 GRUB 與 UEFI 設定正常，系統應會進入 Arch Linux。

在登入畫面輸入：

```text
root
```

接著輸入先前設定的 root 密碼。

---

## 26. 設定 Wi-Fi 自動連線

登入後啟動 NetworkManager 的文字介面：

```bash
nmtui
```

步驟：

1. 選擇：
   ```text
   Activate a connection
   ```
2. 選取 Wi-Fi SSID
3. 輸入 Wi-Fi 密碼
4. 成功連線後，該 Wi-Fi 前方會出現 `*`
5. 回到主選單後，選擇：
   ```text
   Edit a connection
   ```
6. 選取剛剛連線的 Wi-Fi
7. 找到：
   ```text
   Automatically connect
   ```
8. 按空白鍵勾選
9. 選擇 `<OK>` 儲存
10. 選擇 `<Quit>` 離開
- 確認網路：
    ```bash
    ping -c 3 archlinux.org
    ```

---

## 27. 建議後續設定（以下為參考用）

完成基本安裝後，建議繼續進行以下設定。
以[Appinstallation.md](AppsInstallation.md)與[DailyUse.md](DailyUse.md)為主，此處為參考。

### 27.1 建立一般使用者

**絕對不要長期使用 root 作為日常帳號！！！**

1. 建立使用者：
    ```bash
    useradd -m -G wheel -s /bin/bash your_username
    ```
2. 設定密碼：
    ```bash
    passwd your_username
    ```
3. 安裝 sudo：
    ```bash
    pacman -S sudo
    ```
4. 編輯 sudoers：
    ```bash
    EDITOR=nano visudo
    ```
5. 找到並取消註解：
    ```text
    %wheel ALL=(ALL:ALL) ALL
    ```
- 之後即可使用一般使用者登入，並透過 `sudo` 執行管理指令。

---

### 27.2 安裝圖形介面

例如要安裝 GNOME：

```bash
pacman -S gnome gdm
systemctl enable gdm
reboot
```

如果要安裝 KDE Plasma：

```bash
pacman -S plasma sddm
systemctl enable sddm
reboot
```

---

### 27.3 安裝常用工具

```bash
pacman -S git vim wget curl man-db man-pages bash-completion
```

---

## 28. 常見問題整理

### 28.1 進不去 Arch Live，只掉進 initramfs

優先檢查：

1. ISO 路徑是否正確
2. `img_dev=` 是否指向正確 UUID
3. `img_loop=` 是否指向正確 ISO 檔案位置
4. `archisolabel=` 是否與 ISO 版本相符
5. 是否有使用 `copytoram=y`

---

### 28.2 重開機後找不到 GRUB

可以嘗試：

1. 進入 BIOS/UEFI，確認硬碟是否被偵測到
2. 關閉 Fast Boot 或 Fast BIOS Mode
3. 確認 EFI 分割區是否存在
4. 確認 `/boot/EFI/BOOT/BOOTX64.EFI` 是否存在
5. 使用 Arch USB Live 環境重新進入系統並執行：
   ```bash
   mount /dev/sda2 /mnt
   mount /dev/sda1 /mnt/boot
   arch-chroot /mnt
   grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --removable --recheck
   grub-mkconfig -o /boot/grub/grub.cfg
   exit
   umount -R /mnt
   reboot
   ```

---

### 28.3 無線網路無法使用
1. 先確認 NetworkManager 是否啟用：
    ```bash
    systemctl status NetworkManager
    ```
    若未啟用：
    ```bash
    systemctl enable --now NetworkManager
    ```
2. 使用 `nmtui` 設定 Wi-Fi：
    ```bash
    nmtui
    ```

---

## 29. 總結
1. 原目標是完全不使用 USB，直接從 Ubuntu 硬碟中的 Arch ISO 啟動
2. Direct ISO Boot 需要正確設定 `img_dev`、`img_loop`、`archisobasedir` 與 `archisolabel`
3. `copytoram=y` 可讓 Arch Live 環境載入 RAM，使原本硬碟可以被重新分割
4. 前半段安裝流程成功完成，包含重新分割、格式化與 `pacstrap`
5. 第一次重開機後，UEFI 沒有正確找到新安裝的 GRUB
6. 因為 Ubuntu 已被清除，原本硬碟中的 ISO 也不存在，無法再從本機修復
7. 最後使用 Rufus 製作 Arch Linux USB 安裝媒體
8. USB 的用途不是重灌，而是進入 Live 環境修復已安裝好的 Arch 系統
9. 透過 `arch-chroot` 回到硬碟中的 Arch 系統
10. 使用 `grub-install --removable` 補上 UEFI fallback boot path
11. 修復後，系統成功從硬碟直接啟動 Arch Linux

---

## 30. 完整指令摘要
```bash
# Ubuntu 中準備 RAM root
sudo -i
mkdir /tmp/ramroot
mount -t tmpfs -o size=4G tmpfs /tmp/ramroot
mkdir -p /tmp/ramroot/{dev,proc,sys,run,sbin,bin,etc,lib,lib64,usr}
cp -a /bin/{bash,mount,umount,ls,cat,mkdir,rm,cp,ln,grep,sed} /tmp/ramroot/bin/
cp -a /sbin/{fdisk,mkfs*,losetup} /tmp/ramroot/sbin/ 2>/dev/null || true
cp -a /lib/* /tmp/ramroot/lib/ 2>/dev/null || true
cp -a /lib64/* /tmp/ramroot/lib64/ 2>/dev/null || true
cp -a /usr/lib* /tmp/ramroot/usr/ 2>/dev/null || true
mkdir -p /tmp/ramroot/boot
cp /boot/archlinux-x86_64.iso /tmp/ramroot/boot/

# 切換至 RAM root
mount --bind /dev /tmp/ramroot/dev
mount --bind /proc /tmp/ramroot/proc
mount --bind /sys /tmp/ramroot/sys
mount --bind /run /tmp/ramroot/run
mkdir -p /tmp/ramroot/oldroot
cd /tmp/ramroot
pivot_root . oldroot
chroot . /bin/bash -c "umount -l /oldroot"

# Arch Live 環境中
ping -c 3 archlinux.org
lsblk
fdisk /dev/sda

# 格式化
mkfs.vfat -F 32 /dev/sda1
mkfs.ext4 /dev/sda2

# 掛載
mount /dev/sda2 /mnt
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot

# 安裝基本系統
pacstrap -K /mnt base base-devel linux linux-firmware nano networkmanager grub efibootmgr

# 產生 fstab
genfstab -U /mnt >> /mnt/etc/fstab

# chroot
arch-chroot /mnt

# 系統設定
ln -sf /usr/share/zoneinfo/Asia/Taipei /etc/localtime
hwclock --systohc

nano /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

echo "arch-laptop" > /etc/hostname
passwd

# 第一次安裝 GRUB
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --recheck
grub-mkconfig -o /boot/grub/grub.cfg

# 啟用網路
systemctl enable NetworkManager

# 離開並重開機
exit
umount -R /mnt
reboot
```

若重開機後出現 `All boot options are tried`，使用 Rufus 製作 Arch Linux USB，從 USB 進入 Live 環境後修復：

```bash
# 從 Arch USB Live 環境修復已安裝好的系統
lsblk

mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot

arch-chroot /mnt

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --removable --recheck
grub-mkconfig -o /boot/grub/grub.cfg

exit
umount -R /mnt
reboot
```

---

## 31. 注意事項

不同筆電、不同 UEFI 韌體、不同磁碟類型與不同 Arch ISO 版本可能會需要調整參數，特別是以下內容不應直接照抄：
- 磁碟代號，例如 `/dev/sda`
- 分割區代號，例如 `/dev/sda1`、`/dev/sda2`
- ISO 路徑
- UUID
- `archisolabel`
- hostname
- Wi-Fi 設定
- 是否需要 `nomodeset`
- 是否需要 `--removable`
- 是否需要透過 USB Live 環境修復 GRUB

操作前，建議先完整閱讀 Arch Wiki 的 Installation Guide，並準備可用的救援方式。
即使原本目標是無 USB 安裝，也仍建議事先準備可用的外部 Live USB，因為一旦重開機後 UEFI 無法識別新系統，就必須依靠外部環境進行修復。