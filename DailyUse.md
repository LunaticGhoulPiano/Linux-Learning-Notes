# 日常使用
## Niri設定
### 建立預設設定檔
```bash
# 列出已安裝套件檔案路徑
pacman -Ql niri | grep config.kdl
## 輸出: niri /usr/share/doc/niri/default-config.kdl

#  建立設定檔資料夾
mkdir -p ~/.config/niri

# 複製模板
cp /usr/share/doc/niri/default-config.kdl ~/.config/niri/config.kdl

# 確認
ls -l ~/.config/niri/config.kdl
```
### 有關Shell
- 查看當前shell
    ```bash
    # 直接查看全域變數
    echo $SHELL
    ## 輸出: /usr/bin/bash
    
    # 查看當前正在執行的shell process (bash)
    ps -p $$ -o comm=
    ## ps: process status
    ## -p $$: 指定PID為目前shell視窗本身的進程
    ## -o comm=: 只輸出進程執行檔名稱
    ```
- 查看所有shell
    ```bash
    cat /etc/shells
    ## 通常下載新的shell會自動加到此資料夾
    ```
- 更換/套用shell
    - Fish
        - 原生支援syntax highlight
        - 不用另外下載extensions
        - 有些bash script不支援，例如：
            ```bash
            ps -p $$ -o comm=
            ```
            出現錯誤：
            ```bash
            fish: $$ is not the pid. In fish, please use $fish_pid.
            ps -p $$ -o comm=
                   ^
            ```
            要改為
            ```bash
            ps -p $fish_pid -o comm=
            ```
        - 指令：
            ```bash
            # 安裝fish
            sudo pacman -S fish
            ## 用cat查看會看到/usr/bin/fish和/bin/fish，實際都指向同一路徑

            # 變更shell
            chsh -s /usr/bin/fish
            ## 需要重新登入或試reboot才可生效
            ```
        - 美化(syntax highlight)：使用starship
            1. 安裝
                ```bash
                # 安裝
                sudo pacman -S starship
                # 建立設定檔
                mkdir -p ~/.config/fish

                # 把自啟動寫入fish設定檔
                echo 'starship init fish | source' >> ~/.config/fish/config.fish
                ## 需要重開視窗
                ```
            2. 注意starship默認不顯示使用者，需要更改設定檔：
                1. 建立與開啟設定檔
                    ```bash
                    mkdir -p ~/.config
                    nano ~/.config/starship.toml
                    ```
                2. 貼上
                    ```toml
                    # 讓 username 模組永遠顯示
                    [username]
                    show_always = true
                    format = "[$user]($style) "

                    # 如果你連主機名稱 (hostname) 也想一起顯示，可以加上這段
                    [hostname]
                    ssh_only = false
                    format = "at [$hostname]($style) "
                    ```
### Wifi
- `niri`中使用`NetworkManager(nmcli)`連線
    - 預設會記住連過的wifi
    - 連線
        ```bash
        # 掃描
        nmcli device wifi rescan
        nmcli device wifi list
        
        # 連接
        nmcli device wifi connect "名稱" password "密碼" # 用名稱
        nmcli device wifi connect <wifi的SSID> password "密碼" # 用SSID
        ```
    - 列出記住的連線
        ```bash
        nmcli connection show
        ```
    - 中斷wifi連接
        ```bash
        # 找出連接中的裝置
        nmcli device
        ## 會列出
        ## DEVICE    TYPE STATE     CONNECTION
        ## <網卡名稱> wifi connected <名稱>
        ## 假設網卡名稱是"wlp1s0"

        # 中斷
        nmcli device disconnect wlp1s0
        ```
    - 開關wifi
        ```bash
        nmcli radio wifi off # 關
        nmcli radio wifi on # 開
        ```
    - 重新連接記住的wifi
        無論wifi/行動數據熱點分享/手機eSIM分享都一樣
        ```bash
        # 列出來
        nmcli connection show

        # 連接
        nmcli connection up "名稱" # 使用名稱
        nmcli connection up uuid <對應的UUID> # 使用UUID
        ```
### 狀態列
- 常用`Waybar`
    1. 安裝
        ```bash
        # 安裝
        sudo pacman -S waybar

        # 寫入niri設定檔自啟動
        nano ~/.config/niri/config.kdl
        spawn-at-startup "waybar" # 找到spawn-at-startup區塊，如果沒有這行就寫入
        ```
    2. 設定檔
        ```bash
        mkdir -p ~/.config/waybar
        cp /etc/xdg/waybar/config.jsonc ~/.config/waybar/config.jsonc
        cp /etc/xdg/waybar/style.css ~/.config/waybar/style.css
        ```
        