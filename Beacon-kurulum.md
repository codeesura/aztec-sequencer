# Sepolia Execution + Beacon Node Kurulumu (Aztec için)

Bu rehber, Aztec Sequencer Node kurulumu öncesinde gerekli olan **Ethereum execution (Reth)** ve **consensus (Lighthouse)** node'larını nasıl kuracağınızı anlatır.

> Sunucu kiralama ve temel Linux kurulumu için [Hetzner-Kurulum-Rehberi.md](Hetzner-Kurulum-Rehberi.md) dosyasına göz atabilirsiniz.

## Giriş

Aztec Sequencer çalışabilmesi için iki temel altyapıya ihtiyaç duyar:

* **Ethereum RPC (Execution Layer)**: İşlem geçmişi ve state verisi
* **Beacon RPC (Consensus Layer)**: Blok doğrulama, finality bilgisi

Bu bileşenleri dışarıdan satın almak yerine aynı sunucuda, ücretsiz ve güvenli şekilde kendiniz kurabilirsiniz.

---

## Donanım Gereksinimleri

* **İşlemci:** 4+ çekirdek
* **RAM:** 8+ GB
* **Disk:** 200+ GB SSD
* **İnternet:** 25 Mbps yukarı ve aşağı hızlar

---

## Gerekli Paketlerin Kurulumu

```bash
sudo apt update && sudo apt install -y docker.io docker-compose unzip openssl curl
```

---

## Tek Komutla Execution + Beacon Kurulumu

### 📁 Klasör Yapısı

Kurulum işlemi `sepolia-node/` adlı bir klasör altında gerçekleşecektir.

### 🚀 Kurulum Komutu

Aşağıdaki komutu terminale tek satır olarak yapıştırın:

```bash
mkdir -p sepolia-node/{execution/data,beacon/data,jwt} && openssl rand -hex 32 > sepolia-node/jwt/jwt.hex && cat <<EOF > sepolia-node/execution/reth.toml
[prune]
block_interval = 5

[prune.segments]
sender_recovery = { distance = 10064 }
transaction_lookup = { distance = 10064 }
receipts = { distance = 10064 }
account_history = { distance = 10064 }
storage_history = { distance = 10064 }
EOF
cat <<EOF > sepolia-node/docker-compose.yml
services:
  reth:
    image: ghcr.io/paradigmxyz/reth:v1.3.12
    container_name: reth
    restart: unless-stopped
    command: >
      node
      --chain sepolia
      --http
      --http.addr 0.0.0.0
      --ws
      --authrpc.jwtsecret /jwt/jwt.hex
      --authrpc.addr 0.0.0.0
      --authrpc.port 8551
      --datadir /data
      --config /data/reth.toml
    ports:
      - "8545:8545"
      - "8551:8551"
    volumes:
      - ./execution:/data
      - ./jwt:/jwt

  lighthouse:
    image: sigp/lighthouse:latest
    container_name: lighthouse
    restart: unless-stopped
    depends_on:
      - reth
    command: >
      lighthouse bn
      --network sepolia
      --checkpoint-sync-url https://checkpoint-sync.sepolia.ethpandaops.io
      --execution-endpoint http://reth:8551
      --execution-jwt /jwt/jwt.hex
      --http
      --http-address 0.0.0.0
      --metrics
      --datadir /data
    ports:
      - "5052:5052"
    volumes:
      - ./beacon/data:/data
      - ./jwt:/jwt
EOF
cd sepolia-node && docker compose down && docker compose up -d
```

---

## Kontroller

Kurulumdan sonra aşağıdaki adreslerden node'ların çalıştığını kontrol edebilirsiniz:

* **Execution RPC:** [http://localhost:8545](http://localhost:8545)
* **Beacon RPC:** [http://localhost:5052/eth/v1/node/syncing](http://localhost:5052/eth/v1/node/syncing)

Logları görmek için:

```bash
cd sepolia-node && docker compose logs -f
```

### 🔍 Sync Durumunu Kontrol Etme

#### Execution (Reth):

```bash
curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'
```

* `false` dönerse → senkron tamam.
* JSON objesi dönerse → hâlâ sync devam ediyor.

#### Beacon (Lighthouse):

```bash
curl http://localhost:5052/eth/v1/node/syncing
```

* `"is_syncing": false` → senkron tamam.

---

## Aztec ile Entegrasyon

Aztec Sequencer çalıştırırken bu endpoint’leri CLI parametresi olarak kullanın:

```bash
--l1-rpc-urls http://localhost:8545 \
--l1-consensus-host-urls http://localhost:5052
```

Bu şekilde hiçbir ek RPC satın almadan kendi node’unuzla Aztec sequencer'ı besleyebilirsiniz.

---

## Sonuç

Kurulum tamamlandıktan sonra execution ve beacon node'larınız kendi sunucunuzda ücretsiz şekilde çalışır duruma gelecektir. Bu yapı Aztec gibi L2 sistemleri için minimum dışa bağımlılıkla maksimum performans sağlar.
