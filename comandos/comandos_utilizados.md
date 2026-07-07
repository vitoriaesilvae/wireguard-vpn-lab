# Comandos utilizados

## Atualização do sistema

Atualiza a lista de pacotes disponíveis:

```bash
sudo apt update
```

Atualiza os pacotes instalados:

```bash
sudo apt upgrade
```

---

## Instalação do WireGuard

Instala o WireGuard:

```bash
sudo apt install wireguard
```

Verifica se o WireGuard está instalado e exibe as interfaces ativas:

```bash
wg
```

---

## Geração das chaves

Gera um par de chaves (privada e pública):

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

---

## Configuração da interface

Cria o arquivo de configuração do servidor:

```bash
sudo touch /etc/wireguard/wg0.conf
```

Edita o arquivo de configuração:

```bash
sudo nano /etc/wireguard/wg0.conf
```

---

## Encaminhamento de pacotes (IP Forwarding)

Edita as configurações do kernel:

```bash
sudo nano /etc/sysctl.conf
```

Aplica as alterações sem reiniciar o sistema:

```bash
sudo sysctl -p
```

Verifica se o encaminhamento de pacotes está habilitado:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

---

## Interfaces de rede

Lista todas as interfaces de rede:

```bash
ip a
```

Exibe a interface WireGuard:

```bash
ip addr show wg0
```

Exibe a interface Wi-Fi:

```bash
ip addr show wlo1
```

Mostra os endereços IP do computador:

```bash
hostname -I
```

---

## Tabela de roteamento

Exibe a tabela de roteamento:

```bash
ip route
```

---

## Configuração do NAT

Adiciona a regra de NAT para permitir que os clientes da VPN acessem a Internet:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o wlo1 -j MASQUERADE
```

---

## Visualização das regras do iptables

Mostra as regras NAT da cadeia POSTROUTING:

```bash
sudo iptables -t nat -L POSTROUTING
```

Mostra as regras da cadeia FORWARD:

```bash
sudo iptables -L FORWARD
```

Mostra as regras da cadeia INPUT com estatísticas:

```bash
sudo iptables -L INPUT -n -v
```

---

## Controle do WireGuard

Habilita o serviço para iniciar automaticamente com o sistema:

```bash
sudo systemctl enable wg-quick@wg0
```

Inicia a interface WireGuard:

```bash
sudo systemctl start wg-quick@wg0
```

Para a interface WireGuard:

```bash
sudo systemctl stop wg-quick@wg0
```

Reinicia a interface WireGuard:

```bash
sudo systemctl restart wg-quick@wg0
```

Verifica o status do serviço:

```bash
sudo systemctl status wg-quick@wg0
```

Mostra informações sobre a VPN, peers, handshakes e transferência de dados:

```bash
sudo wg
```

---

## Firewall (UFW)

Exibe as regras do firewall:

```bash
sudo ufw status
```

Exibe informações detalhadas:

```bash
sudo ufw status verbose
```

Libera a porta UDP utilizada pelo WireGuard:

```bash
sudo ufw allow 51820/udp
```

Desabilita temporariamente o firewall:

```bash
sudo ufw disable
```

Habilita novamente o firewall:

```bash
sudo ufw enable
```

---

## Captura de pacotes

Captura pacotes UDP destinados à porta 51820 em qualquer interface:

```bash
sudo tcpdump -ni any udp port 51820
```

Captura todo o tráfego UDP:

```bash
sudo tcpdump -ni any udp
```

Captura tráfego UDP na interface Wi-Fi:

```bash
sudo tcpdump -ni wlo1 udp
```

Captura apenas os pacotes trocados com um dispositivo específico:

```bash
sudo tcpdump -ni wlo1 host <IP_DO_ANDROID>
```

---

## Verificação de portas

Lista todas as portas UDP abertas:

```bash
sudo ss -lun
```

Verifica se o WireGuard está escutando na porta UDP 51820:

```bash
sudo ss -lun | grep 51820
```

---

## Geração do QR Code

Instala o utilitário para gerar QR Codes:

```bash
sudo apt install qrencode
```

Gera um QR Code a partir do arquivo de configuração do cliente:

```bash
qrencode -t ansiutf8 < cliente.conf
```

---

## Testes de conectividade

Testa a comunicação com o Android:

```bash
ping <IP_DO_ANDROID>
```

Testa a comunicação com o servidor:

```bash
ping <IP_DO_SERVIDOR>
```

