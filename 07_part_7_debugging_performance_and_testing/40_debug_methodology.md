# 40. Debug Methodology

## 40.1 核心觀念

Debug 不只是看 log，而是把問題縮小到可重現、可量測、可驗證的範圍。建議每個問題都照「現象、範圍、時間線、假設、實驗、結論」記錄。

## 40.2 五步驟流程

1. **定義現象**：誰看到、何時發生、影響範圍、錯誤訊息。
2. **建立時間線**：從 power on、bootloader、kernel、systemd、application 到 host interaction。
3. **縮小範圍**：硬體、Device Tree、driver、D-Bus service、Redfish/IPMI、policy 設定分層排查。
4. **設計實驗**：一次只改一個變因，保留 before/after log。
5. **留下結論**：根因、修正、驗證方法、回歸測試與風險。

## 40.3 Debug 筆記模板

```markdown
## 問題
- 現象：
- 影響版本/平台：
- 重現率：
- 最早發現時間：

## 證據
- journalctl：
- dmesg：
- busctl：
- scope / logic analyzer：

## 假設與實驗
| 假設 | 測試方法 | 結果 | 結論 |
|---|---|---|---|

## 修正與驗證
- 修正 commit：
- 驗證項目：
- 回歸風險：
```
