# 25. Security Baseline

## 25.1 本章用途

本章整理 BMC bring-up 與產品化前應完成的最低安全基線。重點不是把系統一次鎖到最嚴，而是先避免常見的出廠風險：預設密碼、未限制的 debug 入口、未定義的更新信任鏈、以及缺少稽核紀錄。

## 25.2 必做檢查清單

- **帳號與認證**：停用預設帳號或強制首次登入改密碼；限制 root 遠端登入；確認 Redfish、SSH、Web UI 使用一致的帳號政策。
- **憑證與 TLS**：出廠前替換開發用憑證；避免使用固定私鑰；確認 TLS 設定符合客戶資安要求。
- **服務暴露面**：只開必要服務；關閉測試用 HTTP、telnet、tftp、debug shell；用 `ss -lntup` 盤點 listening port。
- **權限與 D-Bus policy**：檢查敏感 D-Bus interface 是否只允許必要服務呼叫，避免一般使用者能觸發 power、firmware update 或 inventory 修改。
- **韌體更新信任鏈**：確認映像檔簽章、版本 rollback policy、雙分割區切換與失敗復原流程。
- **日誌安全**：記錄安全事件，但不要輸出密碼、token、session cookie、API key、私鑰、完整憑證或完整 Authorization header。

## 25.3 Bring-up 到量產的分段做法

1. **EVT/DVT**：先保留 debug 能力，但用 build flag 或 image variant 明確隔離。
2. **PVT**：預設關閉 debug 入口，只允許受控開啟，並要求開啟後留下稽核紀錄。
3. **MP**：移除測試帳密、測試憑證與不必要工具；記錄交付版本的 SBOM、來源 commit、簽章與 build log。

## 25.4 常用驗證指令

```bash
ss -lntup
systemctl --failed
journalctl -b -p warning --no-pager
busctl tree xyz.openbmc_project.User.Manager
busctl introspect xyz.openbmc_project.User.Manager /xyz/openbmc_project/user
```

## 25.5 參考資料

- OpenBMC security documentation: <https://github.com/openbmc/docs/tree/master/security>
- DMTF Redfish standard: <https://www.dmtf.org/standards/redfish>
- DMTF standards overview: <https://www.dmtf.org/standards>
