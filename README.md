# EGO-XZ7 (ZYNQ-7020) PetaLinux 2018.3 建置與打包

Demo影片：https://drive.google.com/file/d/1cXWoMSiuZfFyhBrbIf7ZA_nGF_4qaOvH/view?usp=drive_link

自有一台 Ubuntu 伺服器，系統為 Ubuntu Server 26.04 LTS，並使用 Portainer 管理容器。  
由於 PetaLinux 2018.3 適合在 Ubuntu 18.04 環境下運作，所以用 Portainer 新增一個 Ubuntu 18.04 容器，在裡面建立專案並編譯開機檔。  

第一次編譯開機檔的時候會比較耗效能，這台伺服器的規格大約十分鐘內可完成。  
Intel Core i5-6500、DDR4 16GB RAM、120GB SATA SSD  

安裝 MobaXterm 透過 SSH 與 SFTP 連入伺服器，但接下來的所有安裝指令不是打在這，可用 htop 看效能狀況。  
左側是 SFTP 檔案管理介面，可直觀的建立目錄，不需要用 mkdir。

## 階段一：Vivado 2018.3 硬體設計與設定要點
### 1. ZYNQ IP 核基本設定：

Enable PS 端的 UART 1、SD 0、Ethernet 0 (RGMII)，記得 Ethernet 0 下拉選單勾選 MDIO 並確認右側 MIO 分配到 52、53。
I/O 電壓設定：將 Bank 1 (RGMII/Ethernet 專用) 設定為 1.8V。

時脈與匯流排依賴陷阱：  
若 Block Design 啟用了 M_AXI_GP0 介面，必須將 FCLK_CLK0 繞線接回 M_AXI_GP0_ACLK。若懸空將導致 Vivado 報錯或 AXI 匯流排無法連線。

### 2. 驗證與匯出：

執行 Validate Design 通過後，產生 HDL Wrapper。  
點選 Generate Bitstream -> Export Hardware（勾選 Include bitstream），產出 .hdf 檔。  
到 Project 資料夾，找到 Project.sdk 資料夾，裡面會有剛剛產出的 .hdf 檔案。

### 3. 下載 PetaLinux 2018.3

到 AMD 官網註冊帳號後，在此下載：  
https://www.amd.com/zh-tw/support/downloads/adaptive-socs-and-fpgas/embedded-software.html  
往下滾動，可以看到各種版本的檔案，在 Version 選單選擇 2018.3，再按"PetaLinux 2018.3 Installer"下載。  
檔名為"petalinux-v2018.3-final-installer.run"。 

## 階段二：Portainer / Docker 容器環境準備
### 1. 建立 Ubuntu 18.04 容器：  
Command 設為 sleep infinity（防止容器離線）。  

在 Portainer 的 Advanced container settings -> Volumes 裡，點選 map additional volume。  
看到類型下拉選單（原本是 Volume），把它改成 Bind。  
切換成 Bind 後，原本的下拉選單會變成可輸入文字的欄位  
Container：/workspace  
Host：輸入本機路徑（/home/你的帳號/petalinux_work）

開啟 Console：  
勾選 Interactive & TTY (-i -t)（在 Console 區塊中），然後點擊 Deploy the container。  

### 2. 檔案準備：  
將 petalinux-v2018.3-final-installer.run 與 Vivado 匯出的 .hdf 放入掛載資料夾 "/home/你的帳號/petalinux_work"。

## 階段三：容器內部依賴套件與 PetaLinux SDK 安裝
### 所有指令在容器內執行：

### 1. 修正舊版 Ubuntu 18.04 套件庫 (若 apt-get 404 時執行)
```
sed -i 's|http://archive.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g; s|http://security.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' /etc/apt/sources.list
```

### 2. 安裝 32 位元與 PetaLinux 依賴套件 (root 權限)
```dpkg --add-architecture i386
apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
  tofrodos iproute2 gawk xvfb git make net-tools libncurses5-dev \
  tftpd zlib1g-dev libssl-dev flex bison libselinux1 gnupg wget \
  diffstat chrpath socat xterm autoconf libtool tar unzip texinfo \
  zlib1g-dev:i386 gcc-multilib build-essential libsdl1.2-dev libglib2.0-dev \
  screen pax gzip cpio rsync python python3 locales
```

### 3. 強制生成 en_US.UTF-8 locale 與修正 /bin/sh (關鍵避坑點)
```sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
ln -sf /bin/bash /bin/sh
```

### 4. 建立普通使用者 puser 並設定權限
```useradd -m -s /bin/bash puser  
mkdir -p /opt/pkg/petalinux  
chown -R puser:puser /opt/pkg/petalinux /workspace
```

### 5. 切換至 puser 執行安裝
```
su - puser  
export LC_ALL=en_US.UTF-8  
export LANG=en_US.UTF-8  
chmod +x /workspace/petalinux-v2018.3-final-installer.run  
/workspace/petalinux-v2018.3-final-installer.run /opt/pkg/petalinux  
```

授權條款按 Enter 翻頁，按 q 離開並輸入 y 同意。若提示目錄非空輸入 y 覆蓋。

### 6. 修正 Yocto source/arm 路徑缺層問題 (關鍵除錯步驟)
安裝器會將相關檔案直接解至 components/yocto/source/，但建置時會往 components/yocto/source/arm/ 尋找
```
ln -sfn /opt/pkg/petalinux/components/yocto/source /opt/pkg/petalinux/components/yocto/source/arm
ls -la /opt/pkg/petalinux/components/yocto/source/arm/environment-setup-cortexa9hf-neon-xilinx-linux-gnueabi
ls -la /opt/pkg/petalinux/components/yocto/source/arm/layers/core/oe-init-build-env  
```
備註：  
複製訊息時勿按 Ctrl+C，否則會中斷解壓縮 installer。若中斷請先 rm -rf /opt/pkg/petalinux/* 再重跑。  
PetaLinux 禁止以 root 身分執行安裝與建置，務必切換為 puser。  
若 petalinux-config 或 petalinux-build 出現 ERROR: Failed to source bitbake，可先翻看 build.log 確認是否為 /opt/pkg/petalinux/components/yocto/source/arm/...: No such file or directory，並確認第 6 步的 arm 符號連結已建立。

## 階段四：專案建立、套用 HW Description 與打包 Boot 檔案
### 1. 載入環境變數 (puser 身分)
```export LC_ALL=en_US.UTF-8  
export LANG=en_US.UTF-8  
source /opt/pkg/petalinux/settings.sh  
```
若出現 WARNING: No tftp server found... 屬於無害提示，不影響本機打包與 SD 卡開機。

### 2. 建立 ZYNQ 專案並匯入 .hdf
```cd /workspace  
petalinux-create --type project --template zynq --name ego_xz7_petalinux  
cd /workspace/ego_xz7_petalinux  
petalinux-config --get-hw-description=/workspace  
```
跳出 menuconfig 時，確認 AUTO Settings 無誤後直接 Exit -> Save (Yes)

### 2.5 設定 Ethernet PHY (Marvell 88E1518) Device Tree 與編譯 DTB
在 system-user.dtsi 中補齊 mdio 與 Marvell 88E1518 節點（若容器內無 nano 或 vi，可用 cat << 'EOF' 寫入）：
```
cat << 'EOF' > /workspace/ego_xz7_petalinux/project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
/include/ "system-conf.dtsi"
/ {
};

&gem0 {
    phy-handle = <&phy0>;
    phy-mode = "rgmii-id";
    status = "okay";

    mdio {
        #address-cells = <1>;
        #size-cells = <0>;

        phy0: ethernet-phy@0 {
            compatible = "marvell,88e1510", "marvell,88e1518";
            device_type = "ethernet-phy";
            reg = <0>;
        };
    };
};
EOF
```
接著編譯 Device Tree：  
```petalinux-build -c device-tree```

### 3. 完整編譯 Kernel / FSBL / U-Boot / Rootfs
```petalinux-build```

結尾出現 NOTE: Failed to copy built images to tftp dir: /tftpboot 屬正常現象，不影響產出檔；看到 [INFO] successfully built project 即完成。

### 4. 打包 SD 卡 Boot 檔案
```petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf --fpga images/linux/system.bit --u-boot --force```

### 5. 複製產出檔至掛載目錄 (方便取出至 SD 卡)
```
cp images/linux/BOOT.BIN /workspace/  
cp images/linux/image.ub /workspace/  
```
SD 卡開機部署：  
將 SD 卡格式化為 FAT32 格式，將 /workspace/BOOT.BIN 與 /workspace/image.ub 複製到 SD 卡根目錄，將 EGO-XZ7 開機 jumper/switch 切至 SD 模式，接上 UART（鮑率 115200）即可啟動 Linux。

# EGO-XZ7 PetaLinux 實機測試與部署 SOP
## 階段一：部署開機檔案至 SD 卡
格式化 SD 卡：  
將 SD 卡格式化為 FAT32 檔案系統（建議配備 MBR 分割區）。

複製開機必要檔案：  
將打包產出的 BOOT.BIN 與 image.ub 直接複製至 SD 卡根目錄（請勿建立任何子資料夾）。  
確保 SD 卡容量與讀寫能力正常，寫入完成後安全移除 SD 卡。

## 階段二：開發板硬體指撥開關與接線
SD 卡插拔：  
將格式化好的 SD 卡插入 EGO-XZ7 板卡上的 MicroSD 卡槽。

Boot Mode（開機模式）跳線/指撥設定：  
調整板上的 Boot 選擇 Switch，將開機來源設定為 SD Card Boot（請參照 EGO-XZ7 板卡手冊對應的指撥引腳狀態）。

周邊連接：  
UART 串口：使用 Micro-USB 線連接 EGO-XZ7 的 USB-UART 埠至 PC。

電源供給：連接板卡電源線並確認開關暫時處於 OFF 狀態。

## 階段三：終端機連線與開機測試
開啟 MobaXterm / Serial Terminal：選擇 Serial 連線。

COM Port：選擇 PC 裝置管理員中識別到的 ZYNQ UART Port。

Baud Rate（鮑率）：設定為 115200（Data: 8, Parity: None, Stop: 1, Flow Control: None）。

開啟 EGO-XZ7 電源開關，觀察開機 Console 輸出：

Stage 1 (FSBL)：印出 Xilinx First Stage Boot Loader 資訊。

Stage 2 (U-Boot)：印出 U-Boot 版本、DRAM 容量，並載入 image.ub。

Stage 3 (Kernel)：終端機顯示 2 隻 Linux Tux 企鵝（代表 2 核心 Cortex-A9 載入完成），並輸出系統初始化 log。

## 階段四：系統登入與基本硬體驗證
Linux 登入：

帳號：root

密碼：root

初次登入可能要求更換密碼，依照提示輸入新密碼即可。

系統驗證指令：

1. 驗證 Kernel 版本與硬體架構  
uname -a

2. 驗證 2 個 Cortex-A9 核心狀態  
cat /proc/cpuinfo

3. 檢查記憶體與 Swap 使用量  
free -h

4. 驗證 RootFS 與 SD 卡掛載  
df -h
