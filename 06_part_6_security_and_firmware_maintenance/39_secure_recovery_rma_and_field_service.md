# 39. Secure Recovery, RMA, and Field Service

## 39.1 本章用途

現場維修與 RMA 必須同時考慮可救援性與安全性。這章提供一套原則：讓工程師能復原設備，但不讓 debug 入口變成永久後門。

## 39.2 安全復原原則

- Recovery image 必須有簽章驗證。
- Recovery mode 應有明確觸發條件，例如實體 jumper、受控命令或一次性 token。
- Recovery 完成後應自動清除臨時權限與臨時憑證。
- RMA 前應清除客戶資料、帳號、憑證、log 中的敏感資訊。

## 39.3 現場服務紀錄

每次維修建議記錄：機台序號、BMC 版本、故障現象、操作人員、使用的 recovery image hash、執行時間、結果與是否更換硬體。

## 39.4 常見風險

- 使用固定 rescue 密碼。
- Recovery shell 可透過網路直接進入。
- 維修後忘記關閉 debug service。
- RMA dump 內含客戶 token、IP、帳號或憑證。

## 39.5 交付檢查

```bash
systemctl --failed
ss -lntup
journalctl -b -p warning --no-pager
```

確認沒有維修帳號、臨時憑證、額外開放 port 或殘留 debug flag。
