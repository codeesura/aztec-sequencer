# Aztec Network Sequencer Node Kurulumu

Bu rehber, Aztec Network Testnet üzerinde Sequencer Node kurulumunu adım adım anlatmaktadır.

<details>
<summary>📋 Sunucu Kurulumu (Detaylı bilgi için tıklayın)</summary>

> **Not:** Sunucu kurulumu ile ilgili detaylı bilgi için [Hetzner-Kurulum-Rehberi.md](Hetzner-Kurulum-Rehberi.md) dosyasına bakabilirsiniz. Burada sunucu kiralama ve ilk kurulum adımları detaylı olarak anlatılmaktadır.

</details>

## Giriş ve Arkaplan

Aztec sequencer node, işlemleri sıralama ve blok üretmekten sorumlu kritik bir altyapı bileşenidir.

İşlemler ağa girdiğinde, sequencer node onları bloklara paketler ve gas limitleri, blok boyutu ve işlem geçerliliği gibi çeşitli kısıtlamaları kontrol eder. Bir blok yayınlanmadan önce, diğer sequencer node'ları (bu bağlamda doğrulayıcılar) tarafından doğrulanması gerekir. Bu doğrulayıcılar, bloğun geçerliliğini imzalayarak onaylar ve yeterli sayıda onay toplandığında (komitenin üçte ikisi artı bir), sequencer bloğu L1'e gönderebilir.

## Donanım Gereksinimleri

- **Ağ:** 25 Mbps yukarı/aşağı
- **İşlemci:** 8 çekirdek
- **RAM:** 16 GB
- **Depolama:** 1 TB SSD

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

Paketlerin kurulumu tamamlandıktan sonra sistemi yeniden başlatın:

```bash
sudo reboot
```

## RPC ve Beacon RPC Erişimi

Sequencer node'unuzu çalıştırmak için iki farklı RPC endpoint'ine ihtiyacınız var:
- **Ethereum RPC URL:** Ethereum ağına erişim için
- **Beacon RPC URL:** Ethereum Beacon Chain'e erişim için

### Ethereum RPC URL (Tenderly)

Tenderly, ücretsiz bir public RPC hizmeti sunmaktadır. Herhangi bir hesap oluşturmanıza gerek kalmadan doğrudan aşağıdaki URL'yi kullanabilirsiniz:

```
https://sepolia.rpc.tenderly.co
```

Bu RPC URL doğrudan kullanılabilir ve herhangi bir API anahtarı gerektirmez.

### Beacon RPC URL (Chainstack)

1. [Chainstack](https://chainstack.com/) üzerinde bir hesap oluşturun
2. Kayıt olup giriş yaptıktan sonra dashboard'a gidin
3. "Create a new project" butonuna tıklayın ve projenize bir isim verin
4. Ardından "Explore All nodes" butonuna basın
5. Ethereum'u seçtikten sonra biraz daha aşağı inip Sepolia Testnet'i seçin
6. Her şey tamamlandıktan sonra "Consensus client HTTPS endpoint" RPC' Url'yi kaydedin

Bu URL sizin Beacon RPC URL'nizdir ve Aztec Sequencer Node kurulumunda kullanacaksınız.

## Sepolia Test ETH Alma

Sequencer node'unuzu çalıştırmak için Sepolia test ağında ETH'ye ihtiyacınız olacak. Sepolia PoW Faucet kullanarak ETH alabilirsiniz.

### Sepolia PoW Faucet

[Sepolia PoW Faucet](https://sepolia-faucet.pk910.de/), tarayıcınızda mining yaparak Sepolia test ETH kazanmanızı sağlayan bir hizmettir.

#### Nasıl Kullanılır:

1. [Sepolia PoW Faucet](https://sepolia-faucet.pk910.de/) adresine gidin
2. Ethereum adresinizi girin
3. "Start Mining" butonuna tıklayın
4. Tarayıcınızda mining işlemi başlayacaktır
5. İstediğiniz miktarda ETH kazandığınızda "Claim Rewards" butonuna tıklayın
6. ETH, belirttiğiniz adrese gönderilecektir

Bu mining işlemi, gerçek ETH için değil, sadece test ağı için geçerli olan ETH'yi kazanmanızı sağlar. Mining süresi ne kadar uzun olursa, o kadar fazla test ETH kazanırsınız.

## Docker Compose ile Tek Komutla Kurulum (En Basit Yöntem)

### 1. Bilgilerinizi Tanımlayın

Aşağıdaki komutu kendi bilgilerinizle düzenleyerek çalıştırın:

```bash
# RPC URL'lerinizi tanımlayın
export ETHEREUM_HOSTS="https://sepolia.gateway.tenderly.co"
export L1_CONSENSUS_HOST_URLS="https://ethereum-sepolia.core.chainstack.com/beacon/YOUR_PROJECT_ID"

# Ethereum cüzdanınızı tanımlayın
export VALIDATOR_PRIVATE_KEY="0xYourPrivateKeyHere"
export VALIDATOR_ADDRESS="0xYourEthereumAddress"

# IP adresinizi otomatik olarak alın
export P2P_IP=$(curl -s ifconfig.me)
```

### 2. Portları Açın

```bash
sudo ufw allow 22/tcp
sudo ufw allow 40400/tcp
sudo ufw allow 40400/udp
sudo ufw allow 8080/tcp
sudo ufw enable
```

### 3. Docker Compose ile Node'u Başlatın

Aşağıdaki tek komutu çalıştırarak node'unuzu başlatın:

```bash
mkdir -p ~/aztec-node/data && cd ~/aztec-node

cat > docker-compose.yml << EOL
version: '3'
services:
  aztec-node:
    container_name: aztec-sequencer
    image: aztecprotocol/aztec:alpha-testnet
    restart: unless-stopped
    environment:
      - ETHEREUM_HOSTS=\${ETHEREUM_HOSTS}
      - L1_CONSENSUS_HOST_URLS=\${L1_CONSENSUS_HOST_URLS}
      - DATA_DIRECTORY=/data
      - VALIDATOR_PRIVATE_KEY=\${VALIDATOR_PRIVATE_KEY}
      - VALIDATOR_ADDRESS=\${VALIDATOR_ADDRESS}
      - P2P_IP=\${P2P_IP}
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

### 4. Logları İzleyin

```bash
cd ~/aztec-node && docker compose logs -f
```

### 5. Doğrulayıcı Olarak Kayıt Olun

Node'unuz tamamen senkronize olduktan sonra, aşağıdaki komutu çalıştırarak doğrulayıcı olarak kaydolabilirsiniz:

```bash
cd ~/aztec-node && \
docker compose exec aztec-node sh -c "node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js add-l1-validator \
  --l1-rpc-urls $ETHEREUM_HOSTS \
  --private-key $VALIDATOR_PRIVATE_KEY \
  --attester $VALIDATOR_ADDRESS \
  --proposer-eoa $VALIDATOR_ADDRESS \
  --staking-asset-handler 0xF739D03e98e23A7B65940848aBA8921fF3bAc4b2 \
  --l1-chain-id 11155111"
```

## Diğer Bilgiler

**Sepolia ETH:**
Sepolia ETH ihtiyacınız için:
- [Sepolia PoW Faucet](https://sepolia-faucet.pk910.de/) kullanabilirsiniz
- Discord topluluğundan isteyebilirsiniz

