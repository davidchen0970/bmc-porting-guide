# 25. Firmware Update

更新流程：上傳 image → 驗證 manifest/signature/version/machine → 建立 software object → activation → progress → reboot 或切換 slot → health check → commit / rollback。

Power loss 測試必做：更新前、寫入 bootloader、寫入 kernel、寫入 rootfs、切 slot、首次開機、commit 前斷電。
