# nv-uefi-unlock-playbook

NVIDIA GeForce RTX 20／30／40 的 Tensor 解鎖操作手冊。本倉庫只收文件，供人與 AI 依步驟部署 [nv-uefi-unlocker](https://github.com/physicsdolphin/nv-uefi-unlocker)。韌體與可執行檔請從上游 release 取得，不隨本專案發佈。

## 範圍

工具在 UEFI 階段、顯示驅動載入之前，對 GPU 寫入易失覆蓋，解除部分張量路徑的出廠鉗制。不修改 VBIOS，也不寫入主機板 BIOS 映像。覆蓋在斷電後失效，需於下次冷開機重新執行。

| 文件 | 用途 |
|------|------|
| [GUIDE.md](GUIDE.md) | 相容判斷、製盤、內建分區、BIOS、驗證與還原 |
| [AGENTS.md](AGENTS.md) | 給自動化代理的執行順序與限制 |

## 實測

本人電腦、RTX 3060 Ti、內建 FAT32 分區引導。工作負載為 4096³ dense GEMM，取 20 次中位數。

| 精度 | 鎖定 | 解鎖後 |
|------|------|--------|
| FP16 | 30 TFLOPS | 62.7 TFLOPS |
| BF16 | 30 TFLOPS | 62.9 TFLOPS |
| TF32 | 10 TFLOPS | 20.0 TFLOPS |
| FP32 | 10.0 TFLOPS | 10.0 TFLOPS |

鎖定欄為該卡常見出廠水準；解鎖後欄為本機實測。FP32 為對照，幾乎不變。

## 風險

若黑屏或進錯開機項，按 F11 或 F12，改選 Windows Boot Manager。

## 授權

[GPL-2.0](LICENSE)。
