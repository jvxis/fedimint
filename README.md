# Implementação do Fedimintd e Lightning Gateway

Este guia descreve os passos para implementar o Fedimintd e o Lightning Gateway em uma máquina que já possui o `bitcoind` e `lnd` rodando localmente ou remoto.

## Pré-requisitos (PRECISAM ESTAR PRONTOS ANTES DE RODAR O SCRIPT)
Certificar que as portas 443, 8173 e 8174 estão abertas.
Ter um nome de dominio, seudominio.com apontando para a maquina onde será instalado Fedimint
Executar o comando na conta ROOT. Use o comando `sudo su`


## Script de Instalação
```bash
bash <(curl -sSf https://raw.githubusercontent.com/fedimint/fedimint-docker/master/downloader.sh)
```
# Durante a Instalação o Script fará perguntas sobre o seu ambiente, opte por `Bitcoind` e `Remoto`
# Importante sempre que ele perguntar sobre o seu host, colocar o nome completo do dominio da maquina
# Após a instalação, pare os serviços:
```bash
docker compose down
```
# Copie o arquivo .env para env_old
```bash
cp .env env_old
```
# Edite o arquivo docker-compose.yaml
```bash
nano docker-compose.yaml
```
# Copie na totalidade abaixo em cima do original e altere o nome do seu dominio, e as credenciais do bitcoin:
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

  fedimintd:
    image: fedimint/fedimintd:v0.4.1
    container_name: fedimintd
    volumes:
      - fedimintd_data:/data
    ports:
      - "0.0.0.0:8173:8173"
    environment:
      - FM_DEFAULT_BITCOIND_RPC_KIND=bitcoind
      - FM_DEFAULT_BITCOIND_RPC_URL=http://rpc_user:rpc_pass@bitcoin.friendspool.club:8085
      - FM_BITCOIN_NETWORK=bitcoin
      - FM_BIND_P2P=0.0.0.0:8173
      - FM_P2P_URL=fedimint://fedimint.seu-dominio.com:8173
      - FM_BIND_API=0.0.0.0:8174
      - FM_API_URL=wss://fedimint.seu-dominio.com/ws/
      - FM_REL_NOTES_ACK=0_4_xyz
    restart: always
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.fedimintd.loadbalancer.server.port=8174"
      - "traefik.http.routers.fedimintd.rule=Host(`fedimint.seu-dominio.com`) && Path(`/ws/`)"
      - "traefik.http.routers.fedimintd.entrypoints=websecure"
      - "traefik.http.routers.fedimintd.tls.certresolver=myresolver"

  guardian-ui:
    image: fedimintui/guardian-ui:0.4.2
    container_name: guardian-ui
    environment:
      - PORT=80
      - REACT_APP_FM_CONFIG_API=wss://fedimint.seu-dominio.com/ws/
    depends_on:
      - fedimintd
    restart: always
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.guardian-ui.loadbalancer.server.port=80"
      - "traefik.http.routers.guardian-ui.rule=Host(`fedimint.seu-dominio.com`)"
      - "traefik.http.routers.guardian-ui.entrypoints=websecure"
      - "traefik.http.routers.guardian-ui.tls.certresolver=myresolver"

volumes:
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
docker logs fedimintd
```
Não pode ter erro.

Resultado Esperado:
```bash
Starting fedimintd
2024-09-03T14:10:35.531588Z  INFO fedimintd::fedimintd: Starting fedimintd (version: 0.4.1 version_hash: 45add3342c72bf3237256aa85d3120d3ceb0930c)
2024-09-03T14:10:35.634712Z  INFO fm::consensus: Starting config gen
2024-09-03T14:10:35.639441Z  INFO fm::net::peer::dkg: Created new config gen Api
2024-09-03T14:10:35.639480Z  INFO fm::net::api: Starting api on ws://0.0.0.0:8174
2024-09-03T14:10:35.639485Z  INFO fm::net::auth: Api available for public access
```
Teste Containers
```bash
docker ps
```
Resultado Esperado:
```bash
CONTAINER ID   IMAGE                          COMMAND                  CREATED       STATUS       PORTS                                           NAMES
5cb8c86b431b   fedimintui/guardian-ui:0.4.2   "docker-entrypoint.s…"   5 hours ago   Up 5 hours                                                   guardian-ui
bc8115f4f25e   traefik:v2.10                  "/entrypoint.sh --pr…"   5 hours ago   Up 5 hours   80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   traefik
fadcac59392e   fedimint/fedimintd:v0.4.1      "/nix/store/gf8fi918…"   5 hours ago   Up 5 hours   0.0.0.0:8173->8173/tcp, 8174/tcp                fedimintd
```

# Se não tem erro basta acessar o site com https://fedimint.seu-dominio.com (As vezes melhor acessar com Janela Anônima)

## IMPORTANTE

# Caso precise Reiniciar do ZERO
Apague o diretorio fedimint-services antes de executar novamente o script

