
正式上線時間 : 2026年6月中旬

# BearNetworkChain (BNES) 節點 P2P 網路連線指南

BearNetworkChain BNES - 官方節點客戶端 基於 Γ Physics Engine 的確定性區塊鏈執行環境 | 輕節點 / 全節點 / 權威節點 | TPS 工業級效能 | PQC + Halo2 ZK 融合

本指南說明如何在 BearNetworkChain 的三節點獨立架構（Light、Full、Authority）下，正確配置節點之間的 P2P 連線。

> **架構原則**：三個節點**各自獨立部署**，互不依賴啟動順序。節點之間透過各自的 `*-config.toml` 配置靜態與信任對端，任一節點可單獨啟動或重啟，不影響其他節點正常運行。

---

## 1. 節點角色總覽

| 節點 | 容器名稱 | Docker 內網 IP | HTTP RPC | P2P 埠 | 主要用途 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Light** | `bnes-light` | `172.35.0.5` | `:8545` | `:30303` | 輕量閘道、Nginx 公網入口 |
| **Full** | `bnes-full` | `172.35.0.6` | `:8546` | `:30304` | 完整歷史資料、BNCScan 後端 |
| **Authority** | `bnes-authority` | `172.35.0.7` | `:8547` | `:30305` | PoA 出塊、共識核心 |

---

## 2. 核心概念：StaticNodes vs TrustedNodes

節點的 P2P 對端**必須**透過各自的 `*-config.toml` 配置。

| 特性 | `StaticNodes` (靜態節點) | `TrustedNodes` (信任節點) |
| :--- | :--- | :--- |
| **連線行為** | 背景持續嘗試重連 | 背景持續嘗試重連 |
| **最大連線數限制** | **受限**於 `MaxPeers` 上限 | **無視** `MaxPeers`，無條件放行 |
| **適用場景** | 一般 P2P 擴展 | Authority ↔ Full 關鍵共識路徑 |

> **建議**：絕對不能斷線的核心通道（Authority ↔ Full / Light）請同時寫入 `StaticNodes` 與 `TrustedNodes`。

---

## 3. 設定檔對應表

```
deploy/configs/
├── light-config.toml    ← Light 節點的 P2P 對端設定（[Node.P2P]）
├── full-config.toml     ← Full 節點的 P2P 對端設定
├── auth-config.toml     ← Authority 節點的 P2P 對端設定
├── .env                 ← 所有節點共用的環境變數（一站式管理）
└── P2P_NETWORK_GUIDE.md ← 本說明文件
```

`*-config.toml` 格式範例（以 `full-config.toml` 為例）：
```toml
[Node.P2P]
MaxPeers = 25
NoDiscovery = false
ListenAddr = ":30304"
StaticNodes = [
    "enode://對端NodeID@對端IP:對端P2P埠",
    "enode://對端NodeID@對端IP:對端P2P埠"
]
TrustedNodes = [
    "enode://對端NodeID@對端IP:對端P2P埠",
    "enode://對端NodeID@對端IP:對端P2P埠"
]
EnableMsgEvents = false

[Node.HTTPTimeouts]
ReadTimeout = 30000000000
WriteTimeout = 30000000000
IdleTimeout = 120000000000
```
> ⚠️ 最後一個項目結尾**不可**有逗號

---

## 4. ⚠️ 重要：Enode IP 必須使用 Docker 內網 IP

BNES 節點透過 `--nat extip:IP` 決定自己對外廣播的 IP。
三個節點均已設定為固定的 Docker 內網 IP（見第 1 節），因此填入 `*-config.toml` 的 Enode 中的 IP **必須使用 Docker 內網 IP**，而非公網 IP 或容器 DNS 名稱。

| 節點 | 正確的 Enode 格式 |
| :--- | :--- |
| Light | `enode://...NodeID...@172.35.0.5:30303` |
| Full | `enode://...NodeID...@172.35.0.6:30304` |
| Authority | `enode://...NodeID...@172.35.0.7:30305` |

---

## 5. 取得節點 Enode（NodeID 查詢）

節點啟動後，從日誌直接取得 Enode：
```bash
docker logs bnes-light 2>&1 | findstr "Started P2P"
docker logs bnes-full 2>&1 | findstr "Started P2P"
docker logs bnes-authority 2>&1 | findstr "Started P2P"
```

輸出範例：
```
Started P2P networking   self=enode://0dde88270b...@172.35.0.5:30303
```

---

## 6. 單機部署流程（同一台 VPS）

### 首次部署

**① 啟動所有節點（任意順序）**
```bash
docker compose -f docker-compose-light.yml --env-file ./deploy/configs/.env up -d
docker compose -f docker-compose-full.yml --env-file ./deploy/configs/.env up -d
docker compose -f docker-compose-authority.yml --env-file ./deploy/configs/.env up -d
```

**② 從日誌取得各節點的完整 Enode**
```bash
docker logs bnes-light 2>&1 | findstr "Started P2P"
docker logs bnes-full 2>&1 | findstr "Started P2P"
docker logs bnes-authority 2>&1 | findstr "Started P2P"
```

**③ 更新 `*-config.toml` 填入對端 Enode**

每個 config.toml 應填入**另外兩個節點**的 Enode（IP 使用 Docker 內網 IP）：

| 設定檔 | `StaticNodes` / `TrustedNodes` 應填入 |
| :--- | :--- |
| `light-config.toml` | Full (`172.35.0.6:30304`) + Authority (`172.35.0.7:30305`) |
| `full-config.toml` | Light (`172.35.0.5:30303`) + Authority (`172.35.0.7:30305`) |
| `auth-config.toml` | Light (`172.35.0.5:30303`) + Full (`172.35.0.6:30304`) |

**④ 套用設定（down + up，因 volume mount 已更新）**
```bash
docker compose -f docker-compose-light.yml down
docker compose -f docker-compose-full.yml down
docker compose -f docker-compose-authority.yml down

docker compose -f docker-compose-light.yml --env-file ./deploy/configs/.env up -d
docker compose -f docker-compose-full.yml --env-file ./deploy/configs/.env up -d
docker compose -f docker-compose-authority.yml --env-file ./deploy/configs/.env up -d
```

> **注意**：修改 `*-config.toml` 後需要 `down` 再 `up`（非 `restart`），因為 config.toml 是透過 Volume bind mount 掛載的，容器需要重建才能完整重新讀取。

## Docker Build Result

```bash
[+] Building 61.6s (15/22)
[+] Building 61.8s (15/22)
[+] Building 61.9s (15/22)
[+] Building 62.1s (15/22)
[+] Building 62.2s (15/22)
[+] Building 62.4s (15/22)
[+] Building 62.5s (15/22)
[+] Building 62.6s (15/22)
[+] Building 158.8s (15/22)
[+] Building 158.9s (15/22)
[+] Building 159.1s (15/22)
[+] Building 161.2s (24/24) FINISHED

=> [internal] load local bake definitions                                  0.0s
=> => reading from stdin 558B                                              0.0s
=> [internal] load build definition from Dockerfile                        0.0s
=> => transferring dockerfile: 4.71kB                                      0.0s
=> [internal] load metadata for docker.io/library/golang:1.22-bookworm     1.2s
=> [internal] load metadata for docker.io/library/debian:bookworm-slim     1.2s
=> [internal] load .dockerignore                                           0.0s
=> => transferring context: 331B                                           0.0s
=> [internal] load build context                                           0.2s
=> => transferring context: 667.59kB                                       0.2s

=> [builder 1/7] FROM docker.io/library/golang:1.22-bookworm@sha256:3d69   0.0s
=> [stage-1 1/9] FROM docker.io/library/debian:bookworm-slim@sha256:96e3   0.0s

=> CACHED [stage-1 2/9] RUN apt-get update && apt-get install -y ...        0.0s
=> CACHED [stage-1 3/9] WORKDIR /app                                        0.0s
=> CACHED [builder 2/7] RUN apt-get update && apt-get install -y ...        0.0s
=> CACHED [builder 3/7] RUN curl --proto '=https' --tlsv1.2 ...             0.0s
=> CACHED [builder 4/7] WORKDIR /app                                        0.0s

=> [builder 5/7] COPY . .                                                   0.8s
=> [builder 6/7] RUN cd bnes-halo2-ffi && cargo build --release            58.7s
=> [builder 7/7] RUN set -e; mkdir -p build/bin; ARCH=$(uname -m) ...      98.0s

=> [stage-1 4/9] COPY --from=builder /app/build/bin/bnes /app/bnes          0.2s
=> [stage-1 5/9] COPY --from=builder /app/bnes-halo2-ffi/...                0.0s
=> [stage-1 6/9] RUN ldconfig                                               0.4s
=> [stage-1 7/9] COPY deploy/genesis_mainnet.json ...                       0.0s
=> [stage-1 8/9] COPY deploy/scripts/docker-entrypoint.sh ...               0.0s
=> [stage-1 9/9] RUN chmod +x /app/entrypoint.sh                            0.4s

=> exporting to image                                                       0.5s
=> => exporting layers                                                      0.5s
=> => writing image sha256:ba791fc7526ff36016784d2cc16a9fcd9c6346e6b7e07    0.0s

[+] up 4/4ing to docker.io/bearnetworkchain/bnes-node:light                 0.0s

✔ Image bearnetworkchain/bnes-node:light Built                            163.2s
✔ Network bnes_net Created                                                  0.1s
✔ Volume bnes_deployment_light-data Created                                0.0s
✔ Container bnes-light Started                                             0.5s
```

## Build Status

### Docker Image Build

| Component | Status |
|------------|----------|
| Docker Image | ✅ Built |
| Build Duration | 163.2s |
| Network | ✅ Created |
| Volume | ✅ Created |
| Container | ✅ Started |

### Generated Image

```text
bearnetworkchain/bnes-node:light
```

### Build Summary

```text
Image SHA256:
ba791fc7526ff36016784d2cc16a9fcd9c6346e6b7e07
```

### Deployment Resources

```text
Network:
bnes_net

Volume:
bnes_deployment_light-data

Container:
bnes-light
```

### Result

```text
Deployment completed successfully.
BNES Light Node is running.
```

---

## 7. 跨 VPS 全球互聯

遠端 VPS 的節點只需互換**公網 Enode**，填入各自的 `*-config.toml` 即可。

### 情境 A：擴展全球 P2P 網路（多台 Light 互聯）

將其他 VPS 的 Light 節點公網 Enode 填入本機的 `light-config.toml`：
```toml
StaticNodes = [
  "enode://本機Full的ID@172.35.0.6:30304",
  "enode://美國機LightID@美國公網IP:30303",
  "enode://日本機LightID@日本公網IP:30303"
]
```
> 跨 VPS 時用**公網 IP**，同機時用 **Docker 內網 IP（172.35.0.X）**

### 情境 B：Authority 跨國出塊共識

多台 Authority 節點間必須保持穩定，將對端公網 Enode 同時寫入 `StaticNodes` 與 `TrustedNodes`：
```toml
StaticNodes = [
  "enode://美國機FullID@美國公網IP:30304"
]
TrustedNodes = [
  "enode://美國機FullID@美國公網IP:30304"
]
```

---

## 8. 驗證 Peer 連線狀態

```bash
# 查看 Full 節點目前連線數與 Static 節點數
docker logs bnes-full 2>&1 | findstr "peercount"

# 查看詳細同步狀態（確認 Full 是否正在追上 Authority 出塊高度）
docker logs --tail 20 bnes-full
docker logs --tail 10 bnes-authority
```

成功連線的日誌特徵：
```
Looking for peers        peercount=1  tried=X  static=2   ← static=2 表示 config.toml 已生效
Block synchronisation started                              ← Full 開始同步
Imported new chain segment  number=100  blocks=2           ← Full 成功追上 Authority 出塊
```

