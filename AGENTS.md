# 代理執行說明

在使用者本人的電腦上部署 [nv-uefi-unlocker](https://github.com/physicsdolphin/nv-uefi-unlocker)。開始前閱讀 [README.md](README.md) 與 [GUIDE.md](GUIDE.md)，並依 GUIDE 執行。

## 順序

1. 記錄 GPU 的 `DEV_xxxx`、主機板、Secure Boot，以及磁碟與未配置空間。
2. 對照 GUIDE 判定家族。不在支援範圍內則停止。
3. 與使用者確認部署方式（U 盤或內建分區）、是否放置自動確認檔、是否修改開機順序。
4. 使用官方 release 與 `make-usbroot`。先輸出到資料夾，再複製到 FAT32 根目錄。
5. 告知該主機板的 BIOS 熱鍵，以及開機選單中應選擇的項目名稱。
6. 以 `loader.log` 驗證。`ssdiag` 若被系統攔截，須在回報中說明。效能測試使用機器上既有的 CUDA 環境，並以 TFLOPS 寫出各精度的鎖定值與解鎖後數值。
7. 回報結果與還原步驟。

## 限制

- 不刷寫 VBIOS，不刷寫主機板 BIOS。
- 未核對 DiskNumber 之前，不建立或刪除分區。
- 預設不寫入系統 ESP。
- 不使用非官方的一鍵套件取代上游 release。
- 不把主機名稱、帳號或序號寫入公開文件。
- 說明風險與還原方式，不保證硬體不會受影響。

## 完成條件

`loader.log` 顯示目標卡解鎖讀回成功，作業系統已啟動，且 GPU 狀態正常。若未能執行 `ssdiag`，回報中註明僅有 EFI 側證據。
