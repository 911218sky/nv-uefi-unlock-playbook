# 操作指南

相容性以內上游矩陣為準：[compatibility.md](https://github.com/physicsdolphin/nv-uefi-unlocker/blob/main/docs/compatibility.md)。

## 1. 確認顯卡

```powershell
Get-CimInstance Win32_VideoController |
  Where-Object { $_.PNPDeviceID -match 'VEN_10DE' } |
  Select-Object Name, PNPDeviceID, DriverVersion
```

以 `DEV_xxxx` 對應家族，不以行銷名稱為準。

| 系列 | Device ID（概略） | `-Family` | 備註 |
|------|-------------------|-----------|------|
| RTX 20 | Turing | `tu10x` | Windows 通常需先切到 GSP 模式 |
| RTX 30 | `0x22xx`、`0x24xx`、`0x25xx` | `ga102` | 例如 3060 Ti `2414` |
| RTX 40 | `0x26xx`–`0x28xx` | `ad102` | 無 Turing 的 GSP 前置 |
| RTX 50、Pascal 及更早 | — | 不支援 | 停止 |

建議使用 **v2.1.10** 或更新版本。

## 2. 選擇部署方式

| 方式 | 適用 |
|------|------|
| A. U 盤 | 第一次嘗試，或不想改動內建磁碟 |
| B. 內建 FAT32 分區 | 長期使用、不必每次插入 USB |

兩種方式的目錄相同。載入程式從**它所在磁碟區的根目錄**讀取 `.bin`，資產不可只放在 `EFI\BOOT\` 旁邊。

```text
<磁碟區根>\
  EFI\BOOT\BOOTX64.EFI
  booter_load_<family>.bin
  gsp_boot_<family>.bin
  gsp_boot_desc_<family>.bin
  fwimage_<family>.bin
  fwsignature_<family>.bin
  scrubber_ad10x.bin
  v67-<family>-four-plm.bin
  i_know_what_i_am_doing.txt
```

`v67-*.bin` 存在時為解鎖模式；移除後只做診斷回放。`i_know_what_i_am_doing.txt` 存在時約 5 秒後自動確認，期間按任意鍵即中止。

系統 EFI 系統分區通常只有約 100 MB，且寫壞會影響 Windows 開機。預設使用獨立 FAT32 資料分區，不使用系統 ESP。

## 3. 產生資產

自 [Releases](https://github.com/physicsdolphin/nv-uefi-unlocker/releases) 取得 `BOOTX64.EFI`、`make-usbroot.ps1`、`assets-x86_64-windows.exe`、`SHA256SUMS.txt` 與 `ssdiag`。下載後核對 SHA-256。

```powershell
cd $env:USERPROFILE\Downloads\nv-uefi-unlocker-v2.1.11
$ProgressPreference = 'SilentlyContinue'
.\make-usbroot.ps1 -Family ga102 -Out .\usbroot -UtilsSource local -Work .\inputs
```

`-Family` 改成目標家族；在目標機上可改用 `localhost`。Windows PowerShell 5.1 對 `-Out E:\` 可能失敗，先輸出到資料夾再複製。

## 4. 部署

### 4.1 U 盤

1. 將 USB 格式化為 FAT32。
2. 執行 `xcopy .\usbroot\* X:\ /E /H /Y`（`X:` 為 U 盤代號）。
3. BIOS 設為 UEFI，關閉 CSM；初次建議關閉 Secure Boot。
4. 從開機選單選擇該 USB 的 UEFI 項目。
5. 確認後應進入 Windows。拔除 U 盤後，機器回到原有開機項。

### 4.2 內建分區

1. 在 GPT 磁碟上使用未配置空間，或縮小資料磁碟區約 0.5 GB 至 2 GB。先列出磁碟與分區，確認 DiskNumber。
2. 以系統管理員建立分區：

```powershell
$part = New-Partition -DiskNumber 1 -Size 1500MB -AssignDriveLetter
Format-Volume -Partition $part -FileSystem FAT32 -NewFileSystemLabel NVUNLOCK -Confirm:$false
```

3. 執行 `xcopy .\usbroot\* E:\ /E /H /Y`，盤符改成實際代號。
4. 需要自動確認時：

```powershell
Set-Content E:\i_know_what_i_am_doing.txt "armed" -Encoding ASCII
```

5. 需要常駐開機項時：

```powershell
bcdedit /copy {bootmgr} /d "NV-UEFI-Unlocker"
bcdedit /set {GUID} device partition=E:
bcdedit /set {GUID} path \EFI\BOOT\BOOTX64.EFI
bcdedit /set {fwbootmgr} displayorder {GUID} /addfirst
bcdedit /enum firmware
```

將 `{GUID}` 換成 `bcdedit /copy` 印出的識別碼。第一次先用開機選單手動選擇，確認能回到 Windows，再把它設為第一順位。部分主板會忽略 Windows 寫入的開機順序，此時以 BIOS 的 Boot 選單為準。

## 5. BIOS

開機模式使用 UEFI，CSM 設為 Disabled。初次建議關閉 Secure Boot。

| 廠牌 | 進入 BIOS | 一次性開機選單 |
|------|-----------|----------------|
| MSI | Del | F11 |
| ASUS | Del 或 F2 | Esc 或 F8 |
| Gigabyte | Del | F12 |
| ASRock | F2 或 Del | F11 |

筆記型電腦盡量使用獨顯直連。混合輸出在進入系統後可能把解鎖狀態清掉。

## 6. 驗證

成功時，開機磁碟區根目錄的 `loader.log` 應包含目標 `10de:xxxx`、正確家族、`V67 present - unlock mode`、`plm_open=true`，以及對 `bootmgfw.efi` 的 chainload。沒有新的 `loader.log`，表示 Unlocker 沒有執行。

進入系統後以 `nvidia-smi` 確認顯示卡正常。`ssdiag` 若顯示 `UNLOCKED` 可作系統內讀回；若被 Smart App Control 攔截，仍以 `loader.log` 為準。

效能可在既有的 PyTorch CUDA 環境重跑同一 GEMM。參考讀數：FP16 由 30 TFLOPS 至 62.7 TFLOPS，BF16 由 30 TFLOPS 至 62.9 TFLOPS，TF32 由 10 TFLOPS 至 20.0 TFLOPS，FP32 維持 10.0 TFLOPS。

## 7. 還原

1. 長按電源關機，再開機選單，選擇 Windows Boot Manager。
2. 將 Unlocker 移出第一順位，或刪除 `i_know_what_i_am_doing.txt`。
3. 以 `bcdedit /delete {GUID}` 移除開機項，或在 BIOS 改回 Windows。
4. 可刪除 `NVUNLOCK` 分區。刪除前確認它不是系統磁碟區，也不是 EFI 系統分區。

常見失敗：CSM 未關閉導致黑屏；選到 Legacy USB；`-Out` 指向磁碟根失敗；在 RTX 30 上啟用 Turing 專用的 GSP 登錄。
