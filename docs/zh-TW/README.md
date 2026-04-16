# 硬碟 I/O 錯誤資料救援指南

> 實測環境：WDC WD40PURZ-85TTDY0 4TB，1 個 pending sector，Ubuntu 24.04 LTS Live USB、ddrescue 1.27、TestDisk 7.1

**Languages / 語言 / 言語:** [English](../../README.md) | [繁體中文](README.md) | [日本語](../ja/README.md)


---

## 📋 使用情境：先看這裡，判斷是否適合你

| 情境 | 是否適用 | 說明 |
|------|----------|------|
| 硬碟出現 I/O 裝置錯誤，但 Windows/BIOS 仍能偵測到 | ✅ 適用 | 本文主要情境 |
| 硬碟有少量壞軌（SMART C5 > 0） | ✅ 適用 | 越早處理成功率越高 |
| 分割表損壞（GPT damaged） | ✅ 適用 | 用 sgdisk --restore 還原 |
| Windows 無法掛載但 BIOS 看得到 | ✅ 適用 | Linux 掛載容忍度更高 |
| SMART 顯示 C6 無法校正磁區大量增加 | ⚠️ 謹慎 | 盡快操作，硬碟可能快速惡化 |
| 硬碟摔過或有異聲（喀喀聲） | ⚠️ 謹慎 | 可嘗試，但磁頭損傷需無塵室處理 |
| PCB 電路板燒毀、完全無法偵測 | ❌ 不適用 | 需要硬體層面維修 |
| 磁頭損壞、開機有持續異聲 | ❌ 不適用 | 需無塵室換磁頭，費用 5000～20000 台幣起 |

**核心判斷原則：只要硬碟還能被偵測到，就有機會救資料。**

---

## 🚨 救援黃金原則：開始之前先讀這裡

硬碟已在損壞邊緣，每一次通電、每一次寫入都在消耗它剩下的壽命。在正式開始救援前，請先記住這幾件事：

**1. 立刻停止對壞碟的任何寫入**  
不要下載、不要安裝、不要讓系統自動同步。任何寫入都可能把資料蓋掉，或加速磁頭損壞。

**2. 不要讓 Windows 自動修復**  
Windows 偵測到異常硬碟時會嘗試執行 chkdsk，這在資料還沒備份前是危險的。如果 Windows 彈出修復提示，直接取消或拔線。

**3. 不要反覆通電測試**  
每次開機都在讓磁頭重新定位、讓馬達重新啟動，對已損傷的硬碟是額外的物理壓力。確認好步驟再開始，不要邊查邊試。

**4. 資料複製出來之前，壞碟就是唯讀的**  
任何修復、格式化、分割表操作，都等資料安全備份到另一顆碟之後再做。唯一例外是步驟 6B 的分割表還原，且前提是映像備份已完成或確認無法完成。

**5. 時間就是資料**  
壞軌通常會隨時間擴散。發現問題的當下就是黃金救援時機，不要拖到「有空再說」。

---

## 🗂️ 本次救援背景

這份指南來自一次實際的救援紀錄。

**事件經過：**  
一顆使用約 7800 小時的 WD 4TB 紫標硬碟（WD40PURZ，監控用途磁碟）突然出現「因為 I/O 裝置錯誤，所以無法執行請求」，Windows 的檔案總管無法讀取，但裝置管理員仍看得到。CrystalDiskInfo 顯示只有 1 個 pending sector（C5 = 1），其餘數值正常。

**救援結果：**  
硬碟內約 2～3TB 的影片、圖片資料全數成功救回。

---

## 🛠️ 需要準備的設備

### 本次實際使用

| 設備 | 用途 | 備註 |
|------|------|------|
| 壞碟（WD 4TB） | 救援來源 | SATA 3.5" HDD |
| 健康碟（WD 4TB） | 存放映像檔與最終資料 | 與壞碟同容量，導致映像被截斷（見常見問題） |
| 28GB 隨身碟 | Ubuntu Live USB 開機碟 | 同時也被系統佔用無法另作他用 |
| 另一支隨身碟（任意容量） | 備份分割表 | 幾 KB 就夠，但需另備一支 |
| 另一台電腦 | 製作 Live USB | 不建議用壞碟所在的電腦製作 |

### 建議準備（避免踩坑）

| 設備 | 原因 |
|------|------|
| 目標碟容量**大於壞碟至少 10%** | 容量相同時映像檔末端會被截斷，剛好截掉 GPT 備份分割表 |
| 額外一支小隨身碟（任意容量） | 專門用來備份分割表，與 Live USB 分開 |
| SATA 對 USB 外接盒或多 SATA 接口的電腦 | 需要同時接壞碟、目標碟、Live USB |

---

## 一、起：確認狀況

### 方法 A：Windows 輕度損傷（先試這個）

如果硬碟在 Windows 還看得到，先試看看能不能直接複製資料。

**檢查 SMART 狀態：**

下載並執行 [CrystalDiskInfo](https://crystalmark.info/en/software/crystaldiskinfo/)，重點觀察：

| 項目 | 說明 |
|------|------|
| C5 等候重定的磁區計數 | 大於 0 代表有問題磁區，是 I/O 錯誤的主因 |
| C4 重定位事件計數 | 已確認壞軌數量 |
| C6 無法校正的磁區計數 | 大於 0 代表嚴重損傷 |
| 01 底層資料讀取錯誤率 | Seagate 正常會有大數值，WD 則應接近 0 |

**確認硬碟序號（區分同型號硬碟）：**

```powershell
wmic diskdrive get index,model,serialnumber
```

**嘗試用 Robocopy 複製資料（輕度損傷適用）：**

Robocopy 是 Windows 內建指令，不需安裝，直接在命令提示字元執行：

```cmd
robocopy D:\ E:\backup_D\ /E /R:1 /W:1 /LOG:C:\rescue_log.txt
```

| 參數 | 說明 |
|------|------|
| `/E` | 包含所有子資料夾 |
| `/R:1` | 遇到錯誤只重試 1 次就跳過 |
| `/W:1` | 重試等待 1 秒 |
| `/LOG` | 記錄所有操作，之後可搜尋 ERROR 確認哪些檔案失敗 |

複製完成後用記事本開啟 `C:\rescue_log.txt`，搜尋 `ERROR` 確認哪些檔案失敗。

> ⚠️ 如果 Robocopy 一開始就出現 `錯誤 1117 (0x0000045D) 正在掃描來源目錄`，代表壞軌卡在根目錄或 MFT，需要直接跳到方法 B（Linux 救援）。

> ⚠️ **絕對不要讓 Windows 執行 chkdsk 修復壞碟**，在資料備份完成前，chkdsk 可能讓損壞情況惡化。等資料完全備份到目標碟後，才可以對壞碟執行 chkdsk。

---

### 方法 B：Linux 救援（重度損傷或 Windows 完全無法讀取）

如果 Robocopy 無法執行，或硬碟重開機後 Windows 已無法掛載，繼續往下。

---

## 二、承：建立 Live USB 並開機

### 1. 製作 Live USB（在另一台電腦操作）

1. 下載 [Ubuntu 24.04 LTS ISO](https://ubuntu.com/download/desktop)
2. 下載 [Rufus](https://rufus.ie/)
3. 插入隨身碟，開啟 Rufus
4. 選擇 ISO 檔案
5. **⚠️ 重要：「映像選項」請選「DD 映像模式」**（不是預設的 ISO 模式），否則 Ubuntu 無法正常開機
6. 按下開始燒錄

### 2. 將所有硬碟接上電腦

同時接上：
- 壞碟（來源）
- 目標碟（目的地）
- 備份用小隨身碟

### 3. 從 USB 開機

進 BIOS 選擇從 USB 開機，進入 Ubuntu 後選擇「**Try Ubuntu**」（不要安裝）。

進入桌面後，先把睡眠關掉：Settings → Power → 全部設成 **Never**，避免長時間執行中途中斷。

---

## 三、轉：執行救援

> ⚠️ **以下步驟的順序非常重要，請勿跳過或調換。**

### 步驟 1：安裝必要工具

```bash
sudo apt update && sudo apt install gddrescue gdisk -y
```

### 步驟 2：確認各硬碟對應的裝置名稱

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL
```

對照序號確認每顆硬碟對應的裝置代號，例如：
- 壞碟 → `/dev/sdb`
- 目標碟 → `/dev/sda`
- 備份隨身碟 → `/dev/sdd`

> ⚠️ 以下指令中的 `/dev/sdb`、`/dev/sda` 等請依照你實際的情況替換，**搞錯裝置代號可能導致資料被覆寫**。

### 步驟 3：立即備份分割表 ⭐ 最重要的步驟

掛載備份用小隨身碟：

```bash
sudo mkdir /mnt/usb
sudo mount /dev/sdd1 /mnt/usb
```

> 如果隨身碟是全新未格式化，先執行：`sudo mkfs.fat -F32 /dev/sdd1`，再重新 mount。

備份分割表與 GPT 頭尾：

```bash
# 備份完整分割表
sudo sgdisk --backup=/mnt/usb/partition_table.bak /dev/sdb

# 備份 GPT 開頭（含主分割表）
sudo dd if=/dev/sdb of=/mnt/usb/gpt_head.bin bs=1M count=1

# 備份 GPT 結尾（含備份分割表）
sudo dd if=/dev/sdb bs=512 skip=$(( $(sudo blockdev --getsz /dev/sdb) - 33 )) count=33 of=/mnt/usb/gpt_tail.bin
```

確認三個檔案都存在：

```bash
ls -lh /mnt/usb
```

應看到 `partition_table.bak`、`gpt_head.bin`、`gpt_tail.bin` 三個檔案，加起來不到 2MB。

> 為什麼這步驟最重要？GPT 格式的硬碟在最末尾儲存了一份備份分割表。如果目標碟容量不夠，ddrescue 映像檔會從末尾截斷，剛好把這份備份分割表截掉，導致後續完全無法掛載。先備份到小隨身碟，就算映像截斷也能還原。

### 步驟 4：掛載目標碟

```bash
sudo mkdir /mnt/edata
sudo mount /dev/sda2 /mnt/edata
```

### 步驟 4.5：用 smartctl 評估壞軌程度，決定備份策略

> 這個步驟只讀取不寫入，對壞碟完全安全。  
> 目的是在花 6～15 小時跑 ddrescue 之前，先用幾分鐘判斷壞軌嚴重程度，讓你提前決定是否需要找容量更大的目標碟。

安裝工具：

```bash
sudo apt install smartmontools -y
```

先看目前的 SMART 數值：

```bash
sudo smartctl -A /dev/sdb
```

重點看這三項的 `RAW_VALUE` 欄位：

| 屬性名稱 | ID | 說明 |
|----------|----|------|
| Current_Pending_Sector | C5 | 待確認的問題磁區數，大於 0 就是造成 I/O 錯誤的元兇 |
| Offline_Uncorrectable | C6 | 已確認無法修復的壞軌數，大於 0 代表嚴重損傷 |
| Reallocated_Sector_Ct | 05 | 已重定向的壞軌累計數，持續增加代表硬碟在惡化 |

接著跑 SMART short test（約 2 分鐘）：

```bash
sudo smartctl -t short /dev/sdb
sleep 120
sudo smartctl -l selftest /dev/sdb
```

**根據結果決定策略：**

| 狀況 | 建議策略 |
|------|----------|
| C5=0、C6=0、short test passed | 直接跑 ddrescue，風險低，走步驟 6A（映像路線） |
| C5 為個位數、C6=0 | 建議跑 ddrescue，但要確保目標碟空間夠，走步驟 6A |
| C5 或 C6 達數十個以上 | 壞軌較多，ddrescue 仍可跑但時間會拉長；若無足夠大的目標碟，考慮直接走步驟 6B |
| short test failed，出現 `Completed: read failure` | 壞軌嚴重，建議先想辦法取得容量更大的目標碟再做 ddrescue，不然映像截斷風險很高 |

> 如果手邊沒有足夠大的目標碟，且評估壞軌不多，可以直接跳到步驟 6B（分割表修復路線），用原碟直接掛載複製資料，跳過 ddrescue。  
> **但這代表沒有映像備份，一旦複製中途硬碟完全死亡，資料就無法再救。請自行評估風險。**

### 步驟 5：用 ddrescue 製作磁碟映像

> ⚠️ **執行前先確認目標碟剩餘空間大於壞碟總容量**，否則映像檔會被截斷：
> ```bash
> df -h /mnt/edata
> ```
> 如果空間不夠，**不要繼續執行 ddrescue**，直接跳到步驟 6B（分割表修復路線）。

空間確認足夠後執行：

```bash
sudo ddrescue -d -r1 /dev/sdb /mnt/edata/rescue.img /mnt/edata/rescue.log
```

| 參數 | 說明 |
|------|------|
| `-d` | 直接讀取，繞過系統快取 |
| `-r1` | 壞軌最多重試 1 次就跳過，避免卡死 |

ddrescue 執行時會顯示進度畫面，正常狀態：
- `error rate: 0 B/s` — 目前沒碰到壞軌
- `bad areas: 0` — 尚未發現損壞區域
- `average rate: 30～80 MB/s` — 正常 SATA HDD 速度

整個過程預計 6～15 小時（視硬碟容量與壞軌數量而定）。中途如果被打斷也沒關係，重新執行相同指令會從 rescue.log 記錄的位置繼續。

ddrescue **完成後**，先確認映像品質再決定走哪條路：

```bash
# 查看最終結果
cat /mnt/edata/rescue.log | grep -E "error|bad"
```

根據結果選擇分支：

---

### 步驟 6A：映像無錯誤 → 從映像檔掛載資料（較安全）

**適用條件：** ddrescue 結束時 `read errors: 0`、`bad areas: 0`，且目標碟空間足夠映像完整寫入。

> ⚠️ **執行前確認映像完整性**，用 gdisk 確認分割表沒有損壞：
> ```bash
> sudo gdisk -l /mnt/edata/rescue.img
> ```
> 如果看到 `GPT: damaged` 或找不到正確分割區，代表映像末端被截斷，**改走步驟 6B**。

確認映像正常後，取得分割區的起始 sector：

```bash
sudo gdisk -l /mnt/edata/rescue.img
```

找到 `Basic data partition`（Type 0700）那行，記下 `Start` 欄位的數字（例如 32768），計算 offset：

```
offset = Start sector × 512
例如：32768 × 512 = 16777216
```

掛載映像檔：

```bash
sudo mkdir /mnt/rescue
sudo mount -o loop,ro,offset=16777216 /mnt/edata/rescue.img /mnt/rescue
ls /mnt/rescue
```

看到資料夾結構代表掛載成功，繼續步驟 7A。

---

### 步驟 6B：映像有錯誤或空間不足 → 修復原碟分割表直接掛載

**適用條件：** ddrescue 有 `read errors` 或 `bad areas`，或目標碟空間不足導致映像被截斷，無法從映像正常掛載。

> ⚠️ **這個步驟會直接寫入壞碟**，雖然只是修復分割表不動資料，仍有極低機率加劇損壞。  
> 前提是步驟 3 的分割表備份已完成，才能執行這步。  
> 如果分割表備份還沒做，**先回去做步驟 3**。

用備份的分割表還原壞碟的 GPT：

```bash
sudo sgdisk --restore=/mnt/usb/partition_table.bak /dev/sdb
```

建立掛載點並掛載原碟的資料分割區（唯讀模式）：

```bash
sudo mkdir /mnt/rescue
sudo mount -o ro /dev/sdb2 /mnt/rescue
ls /mnt/rescue
```

看到資料夾結構代表掛載成功，繼續步驟 7B。

---

### 步驟 7A：從映像複製資料到目標碟

> ⚠️ **執行前先確認目標碟有足夠空間存放資料**（映像檔 + 解出的資料會同時佔用空間）：
> ```bash
> df -h /mnt/edata
> du -sh /mnt/rescue
> ```
> 如果空間不夠，先刪除 rescue.img 再複製（見步驟 7B 的刪除方式）。

直接複製：

```bash
sudo cp -av /mnt/rescue/* /mnt/edata/
```

`-av` 會顯示每個複製的檔案，方便監控進度。複製完成後終端機會回到命令提示符 `ubuntu@ubuntu:~$`。

---

### 步驟 7B：從原碟複製資料到目標碟（映像路線的空間不足補救）

> ⚠️ **刪除 rescue.img 前請確認你已經完成了步驟 3 的分割表備份**。rescue.img 刪掉後無法復原，但分割表備份在小隨身碟上，後續還可以用來還原。

先刪除映像檔騰出空間：

```bash
sudo rm /mnt/edata/rescue.img
```

確認空間已釋放：

```bash
df -h /mnt/edata
```

開始從原碟複製資料到目標碟：

```bash
sudo cp -av /mnt/rescue/* /mnt/edata/
```

複製完成後終端機會回到命令提示符 `ubuntu@ubuntu:~$`。

---

## 四、合：收尾與驗證

### 確認救援結果

```bash
df -h /mnt/edata
ls /mnt/edata
```

確認容量有合理增加且資料夾存在即完成。

### 插回 Windows 後

- 目標碟（健康碟）直接插回即可正常讀取，無需額外操作
- 壞碟插回後 Windows 可能要求執行 chkdsk，**此時可以放心讓它跑**，因為資料已安全備份到目標碟
- 壞碟建議退役，不再作為主要儲存使用

---

## 附錄：事前預防

```
3-2-1 備份原則：
  3 份資料
  2 種儲存媒介（例如 HDD + 雲端）
  1 份異地備份
```

定期用 CrystalDiskInfo 監控 SMART，發現 C5 > 0 立即備份，不要等到完全壞掉。

---

## 附錄：常見問題

**Q：Rufus 要用哪個模式燒錄？**  
A：選「DD 映像模式」，不是預設的 ISO 模式。ISO 模式製作出來的 Live USB 可能無法正常開機。

**Q：Robocopy 一開始就回報 I/O 錯誤，連掃描都失敗？**  
A：代表壞軌卡在根目錄或 MFT（主檔案表），Windows 工具完全無法繞過，需要直接用 Linux 的 ddrescue 處理。

**Q：ddrescue 速度很慢，每秒只有幾 KB？**  
A：碰到壞軌在重試是正常現象，`-r1` 讓它重試一次就跳過，不會永遠卡在那裡。如果大部分時間速度正常，只有偶爾卡一下，整體進度還是會持續推進。

**Q：映像檔 mount 失敗，顯示 `NTFS signature is missing`？**  
A：通常是映像檔被截斷（目標碟空間不足），導致 GPT 備份分割表遺失。如果有事先備份分割表，用 `sgdisk --restore` 直接還原原碟分割表後掛載原碟即可。這也是為什麼步驟 3 要先備份分割表。

**Q：TestDisk 找到很多奇怪的分割區，都不是我的資料？**  
A：GPT 損壞時 TestDisk 的掃描結果會有大量誤判，優先用備份的分割表直接還原，比 TestDisk 掃描更可靠。

**Q：目標碟容量跟壞碟一樣大可以嗎？**  
A：技術上可以，但映像檔末端的幾百 MB 會被截斷，剛好截掉 GPT 備份分割表，導致 mount 失敗。解決方式是在 ddrescue 前先備份分割表（步驟 3），之後用 `sgdisk --restore` 直接還原原碟，不依賴映像檔的分割表。

**Q：救援過程中電腦進入睡眠怎麼辦？**  
A：ddrescue 和 cp 都會中斷。進入 Ubuntu 後記得到 Settings → Power，把所有睡眠和螢幕關閉設定改為 Never。ddrescue 中斷後重新執行同樣指令可從上次位置繼續，cp 則需要重新開始。

---

*作者：實際踩坑紀錄整理，如有錯誤歡迎 PR 指正。*
