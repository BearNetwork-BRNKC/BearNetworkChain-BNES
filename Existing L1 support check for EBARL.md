
## 現有 L1 對 EBARL 的支援檢查

| EBARL 需求 | 對應 L1 設計 | 現狀 |
|---|---|---|
| 確定性排序（tx_order） | Clique PoA 出塊者 | ✅ 已具備 |
| 完整 ExecutionTrace 可 replay | Full Node --gcmode archive | ✅ 已具備 |
| 完整歷史狀態 | Full Node --syncmode full | ✅ 已具備 |
| ZK 證明可驗證 | bnes-halo2-ffi | ✅ 已具備 |
| PQC trust root 不漂移 | pqc.identity.uri | ✅ 已具備 |
| 物理不變量（Γ） | BNES Γ 引擎 | ✅ 已具備 |
| runtime_version 固定 | image tag = latest | ✅ 已具備 |
