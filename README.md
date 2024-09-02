# Implementação do Fedimintd e Lightning Gateway

Este guia descreve os passos para implementar o Fedimintd e o Lightning Gateway em uma máquina que já possui o `bitcoind` e `lnd` rodando localmente.

## Pré-requisitos

- **bitcoind**: Bitcoin Core já instalado e rodando.
- **lnd**: Lightning Network Daemon já instalado e rodando.
- **Sistema Operacional**: Linux (Ubuntu recomendado).
- **Git**: Para clonar repositórios.
- **Rust e Cargo**: Para compilar o código-fonte, se necessário.

## 1. Preparação do Ambiente

### Instalar Dependências

Certifique-se de que as seguintes dependências estão instaladas:

```bash
sudo apt update
sudo apt install git build-essential
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Siga as instruções para completar a instalação do Rust.

## 2. Instalar o Fedimintd

### Clonar o Repositório do Fedimint

```bash
git clone https://github.com/fedimint/fedimint.git
cd fedimint
```

### Compilar o Fedimintd

```bash
cargo build --release
```

### Mover o Binário Compilado

```bash
sudo mv target/release/fedimintd /usr/local/bin/
```

### Configurar o Fedimintd

Crie um arquivo de configuração chamado `fedimint.conf`:

```bash
nano ~/fedimint.conf
```

Exemplo de configuração:

```toml
[general]
network = "bitcoin"
bitcoind_rpc = "http://localhost:8332"
bitcoind_rpc_user = "seu_usuario"
bitcoind_rpc_password = "sua_senha"

[federation]
members = ["node1_public_key", "node2_public_key", "node3_public_key"]
threshold = 2
```

### Iniciar o Fedimintd

```bash
fedimintd --config ~/fedimint.conf
```

## 3. Instalar o Lightning Gateway

### Clonar o Repositório do Lightning Gateway

```bash
git clone https://github.com/lightning-gateway/lightning-gateway.git
cd lightning-gateway
```

### Compilar o Lightning Gateway

```bash
cargo build --release
```

### Mover o Binário Compilado

```bash
sudo mv target/release/lightning-gateway /usr/local/bin/
```

### Configurar o Lightning Gateway

Crie um arquivo de configuração chamado `gateway.conf`:

```bash
nano ~/gateway.conf
```

Exemplo de configuração:

```toml
[lightning]
type = "lnd"
host = "localhost"
port = 10009
macaroon_path = "/caminho/para/admin.macaroon"
tls_cert_path = "/caminho/para/tls.cert"

[fedimint]
host = "localhost"
port = 8080
```

### Iniciar o Lightning Gateway

```bash
lightning-gateway --config ~/gateway.conf
```

## 4. Automatizar a Inicialização (Opcional)

### Criar um Serviço systemd para o Fedimintd

```bash
sudo nano /etc/systemd/system/fedimintd.service
```

Adicione o seguinte conteúdo:

```ini
[Unit]
Description=Fedimintd
After=network.target

[Service]
ExecStart=/usr/local/bin/fedimintd --config /home/seu_usuario/fedimint.conf
User=seu_usuario
Restart=always

[Install]
WantedBy=multi-user.target
```

Ativar e iniciar o serviço:

```bash
sudo systemctl enable fedimintd
sudo systemctl start fedimintd
```

### Criar um Serviço systemd para o Lightning Gateway

```bash
sudo nano /etc/systemd/system/lightning-gateway.service
```

Adicione o seguinte conteúdo:

```ini
[Unit]
Description=Lightning Gateway
After=network.target

[Service]
ExecStart=/usr/local/bin/lightning-gateway --config /home/seu_usuario/gateway.conf
User=seu_usuario
Restart=always

[Install]
WantedBy=multi-user.target
```

Ativar e iniciar o serviço:

```bash
sudo systemctl enable lightning-gateway
sudo systemctl start lightning-gateway
```

## 5. Testar a Integração

Realize transações de teste entre o Fedimintd e a Lightning Network para garantir que tudo esteja configurado corretamente.

## 6. Monitoramento e Logs

Monitore os logs dos serviços para identificar e resolver possíveis problemas:

```bash
journalctl -u fedimintd -f
journalctl -u lightning-gateway -f
```

## Considerações Finais

Este guia fornece um passo a passo básico para implementar o Fedimintd e o Lightning Gateway. Ajuste as configurações conforme necessário para atender às suas necessidades específicas. Se encontrar problemas, consulte a documentação oficial ou a comunidade para suporte adicional.
