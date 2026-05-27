# 安裝
## 在Arch CLI安裝Niri
### ArchLinux GUI 簡介
- 分為兩種：
    1. Desktop Environments
        - 完整的GUI，包含視窗管理器、檔案管理器、系統設定等等
        - 可使用pacman(Arch的套件管理器)安裝
        - 常見有`KDE Plasma`、`GNOME`、`XFCE`、`Cinnamon / MATE`
    2. Window Managers / WMs
        - 只負責管理視窗，其餘狀態列、桌布、快捷鍵等都要手動設定，但極快且省資源，分為兩種：
            1. 平鋪式(Tiling WM)：視窗會自動填滿螢幕，完全不用滑鼠純鍵盤操作，如`i3wm`、`bspwm`、`Wayland`、`Hyperland`、`Niri`
            2. 堆疊式(Floating WM)：像Windows可以疊視窗、用滑鼠拖曳，如`Openbox`
### 安裝Niri
```bash
# 安裝
sudo pacman -S niri waybar fuzzel mako alacritty xorg-xwayland
# 出現"There are 10 providers available for xdg-desktop-portal-impl"
## 選擇portal組件在Wayland下相容性最好的"xdg-desktop-portal-gnome"
# 出現"There are 2 providers for jack: 1. jack2, 2. piewire-jack"
## 選擇Linux音訊生態系轉向的pipewire

# 如果安裝中有error: ... 404，通常表示套件太舊
sudo pacman -Syu niri waybar fuzzel mako alacritty xorg-xwayland // 同步資料庫並更新

# 啟動Niri
niri
```

## 建議先執行指令與設定
- 使用者名稱以下用`USER_NAME`表示
- 新增使用者（在初始只會有root）
    ```bash
    useradd -m -G wheel USER_NAME
    passwd USER_NAME
    ```
- 允許使用者使用`sudo`
    1. 開設定檔
        ```bash
        EDITOR=nano visudo # 用nano或是vi
        ```
    2. 找到`# %wheel ALL=(ALL:ALL) ALL`移除註解且開頭不可留空格
        ```bash
        ## Uncomment to allow members of group wheel to execute any command
        # %wheel ALL=(ALL:ALL) ALL
        ```
    3. `Ctrl+O`儲存，`Enter`存檔，`Ctrl+X`離開編輯器
- 切換使用者
    ```bash
    su - USER_NAME
    ```
- 安裝完`pipewire-jack`後最好執行的指令以免Brave影片播放沒有聲音：
    ```bash
    # 安裝Pipewire的PulseAudio相容層"pipewire-pulse"
    sudo pacman -S pipewire-pulse wireplumber libpulse

    # 重新安裝pavucontrol避免pavucontrol: command not found
    sudo pacman -S pavucontrol

    # 此時執行pavucontrol會啟動Volumn Control
    # 如果影片播放時在Volumn Control的Playback頁面沒看到Brave或還是沒聲音

    # 關閉Brave後重新整理音訊管道
    systemctl --user restart pipewire pipewire-pulse wireplumber

    # 檢查與喚醒pupewire音訊服務
    systemctl --user status pipewire pipewire-pulse wireplumber

    # 此時pipewire, pipewire-pulse, wireplumber三個組件應該是綠色的且顯示enable
    ```

## 安裝日常使用工具
### 官方倉庫本身有的
- `git` + `base-devel`
    ```bash
    sudo pacman -S git base-devel
    ```
- `ARU`
    ```bash
    # 切到暫存資料夾後clone source code
    cd /tmp
    git clone https://aur.archlinux.org/paru.git

    # build
    cd paru
    makepkg -si

    # There are 2 providers available for cargo: 1. rust, 2. rustup
    ## 選擇rustup，官方推薦的Rust版本管理器

    # 遇到"error: rustup could not choose a version of cargo to run, because one wasn't specified explicitly, and no default is configured."
    ## 因為剛裝好的 rustup 還只是個沒有核心工具鏈的空殼，所以當 makepkg 試圖呼叫 cargo 來編譯 paru 時，它不知道該用哪一個版本的 Rust，於是直接拋出錯誤並中斷
    rustup default stable # 自動連進Rust official server下載最新版並設rustup為使用者帳號的預設值
    makepgk -si # 重新打包編譯安裝
    ```
### 官方倉庫沒有的
- `Brave` + `LocalSend` + 字型
    ```bash
    # 安裝
    paru -S brave-bin localsend-bin
    # 出現"There are 11 providers available for ttf-font"
    ## 選擇2. noto-fonts因為兼容性最好

    # 安裝完成後順便安裝CJK與顏文字
    sudo pacman -S noto-fonts-cjk noto-fonts-emoji

    # 執行
    brave # 或是Windows+D選擇Brave
    ```
- 如果用`root`啟動`niri`
    - 報錯：
        ```bash
        error: XDG_RUNTIME_DIR is invalid or not set in the environment.
        Failed to connect to Wayland display: No such file or directory
        ```
    1. 先`Windows+Shift+E`退出niri
    2. 用方才新增的一般使用者登入啟動niri
- 嘗試執行Brave出現發現報錯：
    - 直接執行Brave報錯
        ```bash
        brave
        ## Segmentation fault (core dumped)
        ## [ERROR:ozone_platform_x11.cc:257] Missing X server or $DISPLAY
        ## [ERROR:env.cc:246] The platform failed to initialize. Exiting.
        ```
    - 原因：
        - 只輸入`brave`，他會啟動預設的圖形渲染引擎，此引擎在Linux上預設使用X11(X Server)：
            1. 尋找X11 pipe：Brave啟動後會找`$DISPLAY`環境變數
            2. Niri是一個純Wayland的合成器，預設沒有啟動傳統X Server服務
            3. Brave呼叫X11 API但找不到server崩潰導致segmentation fault
    - 解法：
        1. 執行此指令才可開啟brave
            ```bash
            brave --ozone-platform=wayland --enable-features=WaylandWindowDecorations
            ```
            - `--ozone-platform=wayland`：命令Chromium不得用X11，直接用原生`Wayland`渲染
            - `--enable-features=WaylandWindowDecorations`：啟用 Wayland 下的視窗邊框裝飾，避免視窗縮放或標題列出問題
        2. 設為brave啟動預設指令：
            1. 建立設定檔並用`nano`開啟
                ```bash
                mkdir -p ~/.config
                nano ~/.config/brave-flags.conf
                ```
            2. 寫入指令
                ```text
                --ozone-platform=wayland
                --enable-features=WaylandWindowDecorations
                ```
            3. `Ctrl+O`->`Enter`->`Ctrl+X`