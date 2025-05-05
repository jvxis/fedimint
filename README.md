# Implementação do Fedimintd e Lightning Gateway

Este guia descreve os passos para implementar o Fedimintd e o Lightning Gateway em uma máquina que já possui o `bitcoind` e `lnd` rodando localmente ou remoto.

## Pré-requisitos (PRECISAM ESTAR PRONTOS ANTES DE RODAR O SCRIPT)
Certificar que as portas 80, 443, 8173 e 8174 estão abertas. ** IMPORTANTE: Porta 8173 deve estar aberta para UDP também. **
Ter um nome de dominio, seudominio.com apontando para a maquina onde será instalado Fedimint
Executar o comando na conta ROOT. Use o comando `sudo su`


## Script de Instalação
```bash
bash <(curl -sSf https://raw.githubusercontent.com/fedimint/fedimint-docker/master/downloader.sh)
```
# Durante a Instalação o Script fará perguntas sobre o seu ambiente, opte por `Bitcoind` e `Remoto`
# Importante sempre que ele perguntar sobre o seu host, colocar o nome completo do dominio da maquina
# Após a instalação, vá para o diretório e pare os serviços:
```bash
cd fedimint-service
docker compose down
```
# Copie o arquivo .env para env_old
```bash
cp .env env_old
```
# Edite o arquivo .env e substitua integralmente pelo conteúdo abaixo, ajustando o nome de domínio e credenciais do Bitcoin
```bash
nano .env
```
```bash
# Bitcoin auth information
# generate with https://jlopp.github.io/bitcoin-core-rpc-auth-generator/,
# default is user: bitcoin, pass: bitcoin
#BITCOIND_RPC_AUTH=bitcoin:54ae356e13a76dc8068e960eb43193cb$$efeeb347a1f0b4a7b7832cc26e68861bc46f89126a8e136ff9932cad47739041

# This domain should point to the machine fedimintd is being deployed to
FM_DOMAIN=fedixx.br-ln.com

# Where bitcoind is reachable
FM_BITCOIN_RPC_KIND=bitcoind
FM_BITCOIN_RPC_URL=http://rpc_user:rpc_pass@bitcoin.br-ln.com:8085

# For testing or fallback one can also use esplora
# FM_BITCOIN_RPC_KIND=esplora
# FM_BITCOIN_RPC_URL=https://blockstream.info/api/

# fedimintd image
FEDIMINTD_IMAGE=fedimint/fedimintd:v0.7.0
```
# Edite o arquivo docker-compose.yaml
```bash
nano docker-compose.yaml
```
# Copie na totalidade abaixo em cima do original, não altere nada:
```bash
# See the .env file for config options

services:
  traefik:
    image: "traefik:v2.10"
    container_name: "traefik"
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "443:443"
    volumes:
      - "letsencrypt_data:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    restart: unless-stopped
    networks:
      - fedimint_network

  fedimintd:
    image: ${FEDIMINTD_IMAGE}
    volumes:
      - fedimintd_data:/data
    ports:
      - 8173:8173/tcp # p2p tls
      - 8173:8173/udp # p2p iroh
      - 8174:8174/udp # api iroh
      - 127.0.0.1:8175:8175/tcp # ui
    environment:
      - FM_BITCOIN_RPC_KIND=${FM_BITCOIN_RPC_KIND}
      - FM_BITCOIN_RPC_URL=${FM_BITCOIN_RPC_URL}
      - FM_BITCOIN_NETWORK=bitcoin
      - FM_BIND_P2P=0.0.0.0:8173
      - FM_P2P_URL=fedimint://${FM_DOMAIN}:8173
      - FM_API_URL=wss://${FM_DOMAIN}/ws/
      - FM_BIND_API_WS=172.20.0.11:8174
      - FM_BIND_UI=172.20.0.11:8175
      - FM_REL_NOTES_ACK=0_4_xyz
    restart: always
    labels:
      - "traefik.enable=true"
      # API Service
      - "traefik.http.routers.fedimintd-api.rule=Host(`${FM_DOMAIN}`) && Path(`/ws/`)"
      - "traefik.http.routers.fedimintd-api.entrypoints=websecure"
      - "traefik.http.routers.fedimintd-api.tls.certresolver=myresolver"
      - "traefik.http.routers.fedimintd-api.service=fedimintd-api"
      - "traefik.http.services.fedimintd-api.loadbalancer.server.port=8174"
      # UI Service
      - "traefik.http.routers.fedimintd-ui.rule=Host(`${FM_DOMAIN}`)"
      - "traefik.http.routers.fedimintd-ui.entrypoints=websecure"
      - "traefik.http.routers.fedimintd-ui.tls.certresolver=myresolver"
      - "traefik.http.routers.fedimintd-ui.service=fedimintd-ui"
      - "traefik.http.services.fedimintd-ui.loadbalancer.server.port=8175"
    networks:
      fedimint_network:
        ipv4_address: 172.20.0.11

networks:
  fedimint_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1

volumes:
  bitcoind_data:
  letsencrypt_data:
  fedimintd_data:
```
# Salve CTRL+X - YES
# Reinicie os Serviços
```bash
docker compose up -d
```
# Verifique se tudo certo
Teste fedimintd
```bash
docker logs fedimint-service-fedimintd-1
```
Não pode ter erro.

Resultado Esperado:
```bash
Starting fedimintd
2025-05-05T12:39:32.206336Z  INFO fedimintd::fedimintd: Starting fedimintd (version: 0.7.0 version_hash: b983d25d4c3cce1751c54e3ad0230fc507e3aeec)
2025-05-05T12:39:32.206484Z  WARN fm::core: FM_BITCOIN_RPC_KIND is obsolete, use FM_DEFAULT_BITCOIN_RPC_KIND instead
2025-05-05T12:39:32.206496Z  WARN fm::core: FM_BITCOIN_RPC_URL is obsolete, use FM_DEFAULT_BITCOIN_RPC_URL instead
2025-05-05T12:39:32.289114Z  INFO fm::consensus: Starting config gen
2025-05-05T12:39:32.290131Z  INFO fm::net::api: Starting http api on ws://172.20.0.11:8174
2025-05-05T12:39:32.290166Z  INFO fm::net::auth: Api available for public access
2025-05-05T12:39:32.291648Z  INFO fm::consensus: Setup UI running at http://172.20.0.11:8175 🚀
```
Teste Containers
```bash
docker ps
```
Resultado Esperado:
```bash
CONTAINER ID   IMAGE                       COMMAND                  CREATED          STATUS          PORTS                                                                                          NAMES
a53920ac3da1   traefik:v2.10               "/entrypoint.sh --pr…"   12 seconds ago   Up 11 seconds   80/tcp, 0.0.0.0:443->443/tcp                                                                   traefik
2613a8507146   fedimint/fedimintd:v0.7.0   "/nix/store/i2n2xcgj…"   12 seconds ago   Up 11 seconds   0.0.0.0:8173->8173/tcp, 0.0.0.0:8173-8174->8173-8174/udp, 8174/tcp, 127.0.0.1:8175->8175/tcp   fedimint-service-fedimintd-1
```

# Se não tem erro basta acessar o site com https://fedimint.seu-dominio.com (As vezes melhor acessar com Janela Anônima)
![image](https://github.com/user-attachments/assets/225cb7a5-2aa1-46ed-a580-58c1f83fa3f4)


## IMPORTANTE

# Caso precise Reiniciar do ZERO
Apague o diretorio fedimint-services antes de executar novamente o script

