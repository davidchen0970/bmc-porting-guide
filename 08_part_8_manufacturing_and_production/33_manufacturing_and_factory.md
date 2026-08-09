# 32. Manufacturing / Factory

產線流程：進入生產模式 → 燒錄 MAC/Serial/UUID/FRU → 寫入 key/cert（如適用）→ board/SKU ID 檢查 → sensor quick test → fan test → network test → Redfish/IPMI smoke test → 出廠重置 → 關閉 debug / manufacturing mode。
