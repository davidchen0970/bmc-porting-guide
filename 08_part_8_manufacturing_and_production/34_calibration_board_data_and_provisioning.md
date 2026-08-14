# 34. Calibration, Board Data, and Provisioning

## 34.1 本章用途

Board data 是平台差異的來源，例如 MAC、FRU、sensor offset、GPIO polarity、fan table、PSU profile 等。Provisioning 的原則是：資料來源清楚、格式可驗證、寫入可重試、結果可追溯。

## 34.2 資料分類

- **唯一資料**：序號、MAC address、UUID、資產標籤。
- **板級資料**：board revision、SKU、CPLD/FPGA version、strap setting。
- **校準資料**：sensor offset、gain、fan PWM mapping、thermal table。
- **安全資料**：憑證、device identity、secure boot key reference。

## 34.3 寫入流程

1. 驗證輸入資料格式與範圍。
2. 寫入 EEPROM、FRU、TPM、flash partition 或 D-Bus inventory source。
3. 重新讀回並比對。
4. 觸發相關 service reload。
5. 保存製造紀錄與 hash。

## 34.4 常見問題

- MAC address 重複或格式錯誤。
- FRU checksum 錯誤。
- 校準資料單位不一致。
- 出貨 image 仍指向測試 provisioning server。
