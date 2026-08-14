# 38. Firmware Update

## 38.1 本章用途

Firmware update 是 BMC 產品化最容易出問題的路徑之一。本章用「更新前檢查、更新流程、失敗復原、驗證項目」整理實作與 debug 的基本框架。

## 38.2 更新前檢查

- 確認映像檔來源、簽章、版本與目標平台相符。
- 確認目前 boot side、active side、backup side 與剩餘空間。
- 確認電源狀態符合更新要求，例如 host 是否必須關機。
- 確認 Redfish UpdateService、phosphor-software-manager 與底層 flash layout 設定一致。

## 38.3 建議流程

1. 上傳 image。
2. 驗證 manifest、簽章與版本策略。
3. 寫入非 active slot 或暫存區。
4. 更新 BMC 或 host component 的 activation 狀態。
5. 重開機或觸發 component reset。
6. 開機後確認版本、health、event log 與 fallback 狀態。

## 38.4 失敗復原設計

- 寫入中斷：不得破壞目前可開機分割區。
- 驗證失敗：不得切換 active side。
- 開機失敗：bootloader 應能 fallback 到上一個可用版本。
- 版本回退：依產品安全政策決定是否允許 downgrade。

## 38.5 驗證指令範例

```bash
busctl tree xyz.openbmc_project.Software.BMC.Updater
busctl tree xyz.openbmc_project.Software.Version
journalctl -u phosphor-image-updater.service -b --no-pager
curl -k https://$BMC/redfish/v1/UpdateService
```

## 38.6 參考資料

- OpenBMC code update architecture: <https://github.com/openbmc/docs/tree/master/architecture/code-update>
- DMTF Redfish UpdateService: <https://www.dmtf.org/standards/redfish>
