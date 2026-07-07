## 1. Atualização do sistema

Atualizei os pacotes do sistema com:

```bash
sudo apt update    # Baixa a lista mais recente de pacotes disponíveis
sudo apt upgrade   # Instala as versões mais novas dos pacotes já instalados
```

## 2. Instalação do WireGuard

Instalei o WireGuard com:

```bash
sudo apt install wireguard   # Instala ferramentas, módulo de kernel e utilitários
wg                            # Confirma a instalação (exibe interfaces ativas, se houver)
```

## 3. Geração das chaves pública e privada

O WireGuard utiliza criptografia de chave pública. Cada dispositivo possui um par de chaves (privada e pública). A chave pública é compartilhada com os outros participantes da VPN, enquanto a chave privada permanece secreta e é utilizada para comprovar a identidade do dispositivo durante o handshake.

Primeiro criei uma pasta no meu diretório pessoal para guardar as chaves:

```bash
mkdir wireguard    # Cria um diretório chamado "wireguard" no diretório atual
cd wireguard       # Entra no diretório wireguard
```

Dentro da pasta, gerei o par de chaves com:

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

- `wg genkey`: cria uma chave privada aleatória.
- `tee privatekey`: grava a chave em um arquivo chamado `privatekey` e também a encaminha (via pipe | ) para o próximo comando.
- `wg pubkey > publickey`: gera a chave pública correspondente à chave privada recebida e a salva no arquivo `publickey`.

## 4. Criação da interface VPN

Primeiro criei o arquivo de configuração da interface:

```bash
sudo touch /etc/wireguard/wg0.conf
```

- `sudo`: privilégio de administrador do sistema.
- `touch`: comando usado para criar arquivos no Linux.

Depois editei o arquivo:

```bash
sudo nano /etc/wireguard/wg0.conf
```

```
[Interface]
Address = [IP da rede virtual]
ListenPort = 51820          # Porta UDP onde o servidor ficará aberto a requisições
PrivateKey = [chave privada do servidor, nunca divulgue]
```

Depois, adicionei ao arquivo wg0, abaixo da interface  as regras postup e postdown:

**PostUp** : regras que entram em ação assim que o WireGuard sobe a interface `wg0`:

```
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o wlo1 -j MASQUERADE
```

- `iptables -A FORWARD -i wg0 -j ACCEPT`: adiciona uma regra ao final da cadeia **FORWARD** (que controla pacotes que entram por uma interface e saem por outra), permitindo que pacotes que chegam pela interface `wg0` sejam encaminhados.
    - `-A`: adiciona a regra ao final da cadeia.
    - `-i wg0`: especifica a interface de entrada.
    - `-j ACCEPT`: aceita/permite o encaminhamento dos pacotes que casarem com a regra.
- `iptables -A FORWARD -o wg0 -j ACCEPT`: permite que pacotes que **saem** pela interface `wg0` sejam encaminhados.
    - `-o wg0`: especifica a interface de saída.
- `iptables -t nat -A POSTROUTING -o wlo1 -j MASQUERADE`: substitui o IP de origem dos pacotes que saem por `wlo1` pelo IP dessa interface.
    - `-t nat`: usa a tabela NAT.
    - `-A POSTROUTING`: adiciona a regra à cadeia POSTROUTING (aplicada logo antes do pacote sair do servidor).
    - `-o wlo1`: especifica a interface de saída.
    - `-j MASQUERADE`: substitui o IP de origem pelo IP da interface `wlo1`.

**PostDown :** regras que desfazem as anteriores quando a interface `wg0` é desligada:

```
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o wlo1 -j MASQUERADE
```

- `iptables -D FORWARD -i wg0 -j ACCEPT`: remove a regra que aceitava pacotes entrando por `wg0`.
- `iptables -D FORWARD -o wg0 -j ACCEPT`: remove a regra que aceitava pacotes saindo por `wg0`.
- `iptables -t nat -D POSTROUTING -o wlo1 -j MASQUERADE`: remove a regra de MASQUERADE criada na tabela NAT.

## 5. Habilitando o encaminhamento de pacotes (IP forwarding)

Para que o servidor encaminhe os pacotes recebidos do cliente para a internet, editei o arquivo `sysctl.conf`:

```bash
sudo nano /etc/sysctl.conf
```

No arquivo, localizei a linha `#net.ipv4.ip_forward=1` e removi o `#` do início, habilitando o encaminhamento de pacotes.

```bash
sudo sysctl -p    # Aplica as alterações feitas no sysctl.conf sem precisar reiniciar
```

## 6. Configuração do NAT

Antes de configurar o NAT, verifiquei as interfaces de rede disponíveis:

```bash
ip a    # Mostra todas as interfaces de rede
```

Regra aplicada:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o wlo1 -j MASQUERADE
```

- `iptables`: comando para configurar as tabelas do firewall embutido no kernel do Linux.
- `-t nat`: especifica que estamos usando a tabela **nat** do iptables.
- `-A POSTROUTING`: `A` significa "append" (anexar/adicionar), adicionando uma regra à cadeia **POSTROUTING**, aplicada quando o pacote está saindo do servidor (pós-roteamento).
- `-s 10.0.0.0/24`: especifica os pacotes cuja origem é a rede `10.0.0.0/24`.
- `-o wlo1`: especifica que a regra só se aplica à interface de saída `wlo1`.
- `-j MASQUERADE`: `j` é de "jump", o pacote que corresponder aos critérios anteriores é direcionado para a ação **MASQUERADE**, que substitui o endereço IP de origem dos pacotes pelo endereço IP da interface `wlo1`. Posteriormente, o roteador da rede local realiza um novo NAT para o endereço IP público utilizado na Internet.

## 7. Iniciando o WireGuard

```bash
sudo systemctl enable wg-quick@wg0    # Habilita a interface para iniciar automaticamente no boot
sudo systemctl start wg-quick@wg0     # Inicia a interface agora
sudo wg                               # Verifica se a interface está ativa
```

## 8. Criação do cliente

Gerei um par de chaves para o cliente dentro do diretório `wireguard`:

```bash
wg genkey | tee client_private | wg pubkey > client_public
```

Adicionei o cliente no arquivo `wg0.conf`, na seção `[Peer]`:

```
[Peer]
PublicKey = [chave pública do cliente]
AllowedIPs = 10.0.0.2/32
```

Criei o arquivo de configuração do cliente:

```bash
touch cliente.conf
```

```
[Interface]
PrivateKey = [chave privada do cliente]
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = [chave pública do servidor]
Endpoint = [ip do servidor]:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Instalei o aplicativo que gera QR code a partir de uma string de texto:

```bash
sudo apt install qrencode
```

Gerei o QR code a partir do arquivo `cliente.conf`:

```bash
qrencode -t ansiutf8 < cliente.conf
```

- `-t ansiutf8`: gera o QR code usando os próprios caracteres UTF-8 do terminal, em vez de criar um arquivo PNG.
- `< cliente.conf`: usa o conteúdo do arquivo como entrada para o QR code.

Baixei o aplicativo WireGuard no celular Android, escaneei o QR code exibido no terminal, salvei a interface com um nome de minha preferência e ativei a conexão.
