# Sepolia Execution + Beacon Node + Aztec Sequencer Kurulumu

Bu rehber, Aztec Sequencer Node kurulumu için gerekli olan **Ethereum execution (Reth)**, **consensus (Lighthouse)** ve **sequencer (aztec-node)** bileşenlerinin nasıl kurulacağını adım adım anlatır. Tüm sistem tek bir sunucuda ve minimum dışa bağımlılıkla çalışacak şekilde yapılandırılmıştır.

<details>
<summary>📋 Sunucu Kurulumu (Detaylı bilgi için tıklayın)</summary>

> **Not:** Sunucu kurulumu ile ilgili detaylı bilgi için [Hetzner-Kurulum-Rehberi.md](Hetzner-Kurulum-Rehberi.md) dosyasına bakabilirsiniz. Burada sunucu kiralama ve ilk kurulum adımları detaylı olarak anlatılmaktadır.

</details>

## Giriş ve Arkaplan

Aztec sequencer node, işlemleri sıralama ve blok üretmekten sorumlu kritik bir altyapı bileşenidir.

İşlemler ağa girdiğinde, sequencer node onları bloklara paketler ve gas limitleri, blok boyutu ve işlem geçerliliği gibi çeşitli kısıtlamaları kontrol eder. Bir blok yayınlanmadan önce, diğer sequencer node'ları (bu bağlamda doğrulayıcılar) tarafından doğrulanması gerekir. Bu doğrulayıcılar, bloğun geçerliliğini imzalayarak onaylar ve yeterli sayıda onay toplandığında (komitenin üçte ikisi artı bir), sequencer bloğu L1'e gönderebilir.

### Donanım Gereksinimleri

- **Ağ:** 25 Mbps yukarı/aşağı
- **İşlemci:** 8 çekirdek
- **RAM:** 16 GB
- **Depolama:** 1 TB SSD

### Sepolia ETH Gerekliliği

Sequencer node’un doğru çalışabilmesi için Sepolia ağında ETH’ye sahip olmanız gerekir. Bunun için:

* [https://testnetbridge.com/sepolia](https://testnetbridge.com/sepolia) üzerinden küçük miktarlarda Sepolia ETH satın alabilirsiniz (5–10 USD yeterlidir).
* Alternatif olarak [Sepolia PoW Faucet](https://sepolia-faucet.pk910.de/) ile de test ETH elde edebilirsiniz.

---

## Gerekli Paketlerin Kurulumu

```bash
sudo apt update && sudo apt upgrade -y && sudo apt install -y \
  git curl wget build-essential cmake pkg-config libssl-dev \
  ca-certificates gnupg lsb-release unzip apt-transport-https && \
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash && \
export NVM_DIR="$HOME/.nvm" && source "$NVM_DIR/nvm.sh" && \
nvm install 20.17.0 && nvm use 20.17.0 && \
sudo install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && \
sudo apt update && sudo apt install -y \
  docker-ce docker-ce-cli containerd.io docker-compose-plugin && \
sudo usermod -aG docker $USER && newgrp docker
```

> ⚠️ `gnupg` kurulumu sırasında ekranda "OK" tuşuna basmanız gerekebilir.

Kurulumdan sonra sistemi yeniden başlatın:

```bash
sudo reboot
```

---

## 2. Execution + Beacon Node Kurulumu (Snapshot ile)

### 2.1. Dizinleri oluştur ve JWT üret

```bash
mkdir -p ~/sepolia-node/{execution,beacon/data,jwt} && \
openssl rand -hex 32 > ~/sepolia-node/jwt/jwt.hex
```

### 2.2. Snapshot'ı indir ve `execution` dizinine çıkar

Aşağıdaki komut, en son Sepolia Reth snapshot'ını indirir ve `~/sepolia-node/execution` dizinine çıkarır:

```bash
BLOCK_NUMBER=$(curl -s https://snapshots.ethpandaops.io/sepolia/reth/latest) && \
docker run -d --name snapshot-downloader \
  -v ~/sepolia-node/execution:/data \
  alpine /bin/sh -c "\
    apk add --no-cache curl tar zstd && \
    echo '📦 Downloading snapshot for block $BLOCK_NUMBER' && \
    curl -s -L https://snapshots.ethpandaops.io/sepolia/reth/$BLOCK_NUMBER/snapshot.tar.zst | \
    tar -I zstd -xvf - -C /data && \
    echo '✅ Snapshot extraction complete.'"
```

#### Snapshot İndirme İlerlemesini Takip Etme

⚠️ **ÖNEMLİ:** Bir sonraki adıma geçmeden önce snapshot indirme işleminin tamamlanmasını beklemelisiniz.

* Snapshot indirme durumunu kontrol etmek için:

```bash
docker logs -f snapshot-downloader
```

* Snapshot tamamen çıkarıldığında `✅ Snapshot extraction complete.` mesajı görünecektir. Bu mesajı görene kadar bekleyin.

* İndirme işlemi tamamlandığında, snapshot konteynerini kaldırabilirsiniz:

```bash
docker rm -f snapshot-downloader
```

### 2.3. Execution ve Beacon node için `docker-compose.yml` oluştur

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
    depends_on:
      - reth
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

### 2.4. Execution ve Beacon node'ları başlat

```bash
cd ~/sepolia-node && docker compose up -d
```

### 2.5. Senkronizasyon Durumunu Kontrol Et

⚠️ **ÖNEMLİ:** Aztec node'unu başlatmadan önce Execution (Reth) ve Beacon (Lighthouse) node'larının senkronize olmasını beklemelisiniz.

#### Reth senkronizasyon durumunu kontrol etme:

```bash
curl -s -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' | grep result
```

* Senkronizasyon devam ediyorsa, çıktıda detaylı senkronizasyon bilgileri görünecektir.
* Senkronizasyon tamamlandığında, çıktıda `"result":false` ifadesi yer alacaktır.

#### Reth'in mevcut blok yüksekliğini kontrol etme:

```bash
curl -s -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' | \
  grep -o '"result":"[^"]*"' | cut -d'"' -f4 | xargs printf "%d\n"
```

* Bu komut, node'un şu anda senkronize olduğu blok numarasını ondalık formatta gösterecektir.
* Son Sepolia blok yüksekliğini [Sepolia Explorer](https://sepolia.etherscan.io/)'dan kontrol edebilirsiniz.

#### Lighthouse senkronizasyon durumunu kontrol etme:

```bash
cd ~/sepolia-node && \
curl -s -X GET "http://localhost:5052/eth/v1/node/syncing" | grep -o '"is_syncing":[^,]*' | cut -d':' -f2
```

* Senkronizasyon tamamlandığında, çıktıda `false` ifadesi yer alacaktır.

> **Not:** Senkronizasyon birkaç saat sürebilir. Senkronizasyon tamamlanana kadar düzenli olarak kontrol edin.

## 3. Aztec Sequencer Kurulumu

### 3.1. Bilgilerinizi Tanımlayın (.env dosyası oluştur)

Sequencer node kurulumu için gerekli değişkenleri bir `.env` dosyasında toplamak sistemin daha güvenli ve düzenli olmasını sağlar:

```bash
mkdir -p ~/aztec-node && cd ~/aztec-node 
cat > .env <<EOF
VALIDATOR_PRIVATE_KEY=0xyourprivatekeyhere
VALIDATOR_ADDRESS=0xyouraddresshere
P2P_IP=$(curl -s ifconfig.me)
GOVERNANCE_PROPOSER_PAYLOAD_ADDRESS=0x54F7fe24E349993b363A5Fa1bccdAe2589D5E5Ef
EOF
```

> ⚠️ **ÖNEMLİ:** `.env` dosyasındaki `VALIDATOR_PRIVATE_KEY` ve `VALIDATOR_ADDRESS` değerlerini kendi özel anahtarınız ve adresinizle değiştirin.

### 3.2. Docker Compose ile Aztec Node'u Başlatın

```bash
mkdir -p ~/aztec-node/data && cd ~/aztec-node

cat > docker-compose.yml << EOL
services:
  aztec-node:
    container_name: aztec-sequencer
    image: aztecprotocol/aztec:0.87.2
    restart: unless-stopped
    network_mode: host
    environment:
      - ETHEREUM_HOSTS=http://localhost:8545
      - L1_CONSENSUS_HOST_URLS=http://localhost:5052
      - DATA_DIRECTORY=/data
      - VALIDATOR_PRIVATE_KEY=${VALIDATOR_PRIVATE_KEY}
      - VALIDATOR_ADDRESS=${VALIDATOR_ADDRESS}
      - P2P_IP=${P2P_IP}
      - GOVERNANCE_PROPOSER_PAYLOAD_ADDRESS=${GOVERNANCE_PROPOSER_PAYLOAD_ADDRESS}
      - LOG_LEVEL=info
    volumes:
      - ./data:/data
    ports:
      - "40400:40400/tcp"
      - "40400:40400/udp"
      - "8080:8080/tcp"
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network alpha-testnet --node --archiver --sequencer --p2p.maxTxPoolSize 1000000000'
EOL

docker compose up -d && echo "✅ Aztec Sequencer başlatıldı! Logları görüntülemek için: docker compose logs -f"
```

### 3.3. Logları İzleyin

```bash
cd ~/aztec-node && docker compose logs -f
```

### 3.4. Aztec Node'un Senkronizasyon Durumunu Kontrol Edin

Aztec node'unun son indirdiği blok numarasını görmek için:

```bash
cd ~/aztec-node && docker compose logs --tail 20 | grep "Downloaded L2 block" | tail -1 | grep -o '"blockNumber":[0-9]*' | cut -d':' -f2
```

* Bu komut, node'un indirdiği en son bloğun numarasını gösterecektir (örn: `40187`)
* Bu blok numarasını [AztecScan](https://aztecscan.xyz/blocks/) üzerinden kontrol ederek senkronizasyonun güncel olup olmadığını anlayabilirsiniz
* Örneğin: https://aztecscan.xyz/blocks/40187 (kendi blok numaranızı kullanın)

Basit durum kontrolü:

```bash
curl -s http://localhost:8080/status
```

* Yanıt "OK" ise, node çalışıyor demektir.

### 3.5. Doğrulayıcı Olarak Kayıt Olun

Node'unuz tamamen senkronize olduktan sonra, aşağıdaki komutu çalıştırarak doğrulayıcı olarak kaydolabilirsiniz:

```bash
cd ~/aztec-node && \
docker compose exec aztec-node sh -c "node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js add-l1-validator \
  --l1-rpc-urls \$ETHEREUM_HOSTS \
  --private-key \$VALIDATOR_PRIVATE_KEY \
  --attester \$VALIDATOR_ADDRESS \
  --proposer-eoa \$VALIDATOR_ADDRESS \
  --staking-asset-handler 0xF739D03e98e23A7B65940848aBA8921fF3bAc4b2 \
  --l1-chain-id 11155111"
```

## Diğer Bilgiler

1. **Docker Konteynerlerinin Durumunu Kontrol Etme:**
   ```bash
   docker ps -a
   ```

2. **Reth Node Loglarını İnceleme:**
   ```bash
   cd ~/sepolia-node && docker compose logs -f reth
   ```

3. **Lighthouse Node Loglarını İnceleme:**
   ```bash
   cd ~/sepolia-node && docker compose logs -f lighthouse
   ```

4. **Aztec Node Loglarını İnceleme:**
   ```bash
   cd ~/aztec-node && docker compose logs -f
   ```

5. **Node'ları Yeniden Başlatma:**
   ```bash
   # Execution ve Beacon node'ları için:
   cd ~/sepolia-node && docker compose restart
   
   # Aztec node için:
   cd ~/aztec-node && docker compose restart
   ```

