# 33. Manufacturing and Factory

## 33.1 本章用途

量產流程的目標是讓每台機器都能被一致地燒錄、校驗、校準、寫入資產資料並留下可追溯紀錄。BMC 軟體需要提供穩定的工廠模式，但工廠模式不應在出貨後變成安全風險。

## 33.2 工廠站點常見項目

1. 燒錄 BMC image 與 bootloader。
2. 寫入序號、MAC address、FRU、資產標籤與客戶料號。
3. 校準 sensor offset、fan table 或 board-specific data。
4. 執行 basic hardware test：I2C scan、GPIO state、fan PWM、power cycle、Redfish sanity。
5. 產生製造紀錄：版本、hash、序號、測試結果、時間與站點。

## 33.3 出貨前檢查

- 清除 factory test account。
- 關閉 factory service 與 debug port。
- 確認資產資料與 Redfish/FRU/D-Bus 顯示一致。
- 確認 MAC address 不重複。
- 確認 event log 只保留需要的製造紀錄，不含測試密碼或內部 token。

## 33.4 建議輸出格式

```json
{
  "serial_number": "...",
  "bmc_version": "...",
  "image_hash": "...",
  "test_result": "PASS",
  "station": "FT-01",
  "timestamp": "..."
}
```
