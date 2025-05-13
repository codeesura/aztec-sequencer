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
* **Disk:** 1TB+ SSD
* **İnternet:** 25 Mbps yukarı ve aşağı hızlar

---

## Gerekli Paketlerin Kurulumu

```bash
sudo apt update && sudo apt install -y docker.io docker-compose unzip openssl curl
```

---

## Snapshot ile Hızlı Kurulum (Önerilen Yöntem)

### 📁 Klasör Yapısı

Kurulum işlemi `~/sepolia-node/` adlı bir klasör altında gerçekleşecektir.

### 🚀 Kurulum Adımları

Aşağıdaki komutları sırasıyla çalıştırın:

#### 1. Dizinleri oluştur ve JWT üret

```bash
mkdir -p ~/sepolia-node/{execution,beacon/data,jwt} && \
openssl rand -hex 32 > ~/sepolia-node/jwt/jwt.hex
```

#### 2. Snapshot'ı indir ve execution dizinine çıkar

```bash
BLOCK_NUMBER=$(curl -s https://snapshots.ethpandaops.io/sepolia/reth/latest) && \
docker run --rm \
  -v ~/sepolia-node/execution:/data \
  alpine /bin/sh -c "\
    apk add --no-cache curl tar zstd && \
    echo '📦 Downloading snapshot for block $BLOCK_NUMBER' && \
    curl -s -L https://snapshots.ethpandaops.io/sepolia/reth/$BLOCK_NUMBER/snapshot.tar.zst | \
    tar -I zstd -xvf - -C /data"
```

Snapshot indirimi tamamlandıktan sonra `~/sepolia-node/execution` klasöründe `chaindata/`, `static/` vb. klasörlerin oluştuğundan emin olun. Bu klasörler oluştuysa, Reth node'u snapshot'tan senkron başlamaya hazırdır.

#### 3. `docker-compose.yml` dosyasını oluştur

```bash
cat <<EOF > ~/sepolia-node/docker-compose.yml
version: "3.8"
services:
  reth:
    image: ghcr.io/paradigmxyz/reth:latest
    container_name: reth
    restart: unless-stopped
    command: >
      node
      --chain sepolia
      --datadir /data
      --http
      --http.addr 0.0.0.0
      --http.port 8545
      --ws
      --authrpc.jwtsecret /jwt/jwt.hex
      --authrpc.addr 0.0.0.0
      --authrpc.port 8551
      --config /data/reth.toml
    ports:
      - "8545:8545"
      - "8551:8551"
    volumes:
      - ./execution:/data
      - ./jwt:/jwt
    network_mode: host

  lighthouse:
    image: sigp/lighthouse:latest
    container_name: lighthouse
    restart: unless-stopped
    command: >
      lighthouse bn
      --network sepolia
      --checkpoint-sync-url https://checkpoint-sync.sepolia.ethpandaops.io
      --execution-endpoint http://localhost:8551
      --execution-jwt /jwt/jwt.hex
      --http
      --http-address 0.0.0.0
      --metrics
      --datadir /data
    ports:
      - "5052:5052"
    volumes:
      - ./beacon:/data
      - ./jwt:/jwt
    network_mode: host
EOF
```

#### 4. Node'ları başlat

```bash
cd ~/sepolia-node && docker compose up -d
```

---

## Kontroller

Kurulumdan sonra aşağıdaki adreslerden node'ların çalıştığını kontrol edebilirsiniz:

* **Execution RPC:** [http://localhost:8545](http://localhost:8545)
* **Beacon RPC:** [http://localhost:5052/eth/v1/node/syncing](http://localhost:5052/eth/v1/node/syncing)

Logları görmek için:

```bash
cd ~/sepolia-node && docker compose logs -f
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
