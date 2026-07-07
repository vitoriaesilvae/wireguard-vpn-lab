## Objetivo

O objetivo deste projeto foi instalar e configurar um servidor VPN utilizando o WireGuard em um sistema Linux, permitindo que um cliente Android estabelecesse um túnel VPN seguro e acessasse a Internet através do servidor. Durante o desenvolvimento também foram estudados conceitos de criptografia assimétrica, NAT, roteamento IP, firewall e diagnóstico de redes

## O que é uma VPN?

VPN significa Virtual Private Network, é um túnel criptografado que conecta dois dispositivos através de uma rede não confiável. Do cliente todo o tráfego sai criptografado para o servidor, que descriptografa e encaminha para o destino.

O wireguard é uma ferramenta que permite a configuração de um servidor vpn com pouco código, de maneira mais rápida. Antes da troca de dados, cliente e servidor executam um handshake criptográfico. Durante esse processo, ambos comprovam que possuem as chaves corretas e estabelecem chaves temporárias de sessão. Somente após um handshake bem-sucedido os pacotes passam a ser aceitos e descriptografados.

---

## Componentes do projeto

- Linux mint
- WireGuard
- Cliente Android
- Interface wg0
- Interface Wi-Fi (wlo1)
- UFW
- iptables
- NAT
- Internet

---

## Topologia

![Topologia da VPN](wireguard-vpn-lab/imagens/Topologia.png)

---

## Explicações

### Fluxo dos pacotes

Os pacotes saem do cliente pelo túnel criptografado da VPN e vão até o servidor. No servidor, os pacotes são descriptografados e enviados à internet com o endereço de origem do servidor (por meio da regra de NAT/MASQUERADE). Quando os pacotes chegam da internet ao servidor, o servidor os criptografa novamente e os envia ao cliente por meio do canal VPN.

![Fluxo dos pacotes](wireguard-vpn-lab/imagens/Fluxo_pacotes.png)

### Criptografia

O WireGuard combina criptografia assimétrica com criptografia simétrica. A criptografia assimétrica é utilizada para autenticar os dispositivos e estabelecer as chaves da sessão, enquanto a criptografia simétrica é responsável por criptografar os dados que trafegam pelo túnel VPN.

Cliente e servidor possuem um par permanente de chaves (pública e privada), compartilhando apenas suas chaves públicas entre si. Durante o handshake, ambos utilizam o algoritmo Diffie-Hellman sobre curvas elípticas (Curve25519). Nesse processo, cada dispositivo combina sua chave privada com a chave pública do outro para calcular, de forma independente, o mesmo segredo compartilhado, sem que esse segredo seja transmitido pela rede.

Em seguida, uma KDF (Key Derivation Function) utiliza esse segredo para derivar as chaves simétricas empregadas pelo algoritmo ChaCha20, responsável por criptografar os dados transmitidos pela VPN.

Além das chaves permanentes, durante cada handshake cliente e servidor geram novos pares de chaves efêmeras (temporárias). Essas chaves participam de novos cálculos Diffie-Hellman, juntamente com as chaves permanentes, produzindo segredos compartilhados adicionais. Todos esses resultados são combinados pela KDF para derivar as chaves simétricas da sessão.

Como um novo par de chaves efêmeras é gerado a cada handshake, mesmo que uma chave permanente seja comprometida no futuro, não será possível descriptografar sessões anteriores. Essa propriedade é conhecida como Perfect Forward Secrecy (PFS).

### NAT

O NAT (Network Address Translation) é uma técnica que traduz endereços IP públicos para privados e vice-versa, armazenando essa correspondência em uma tabela de rastreamento de conexões.
Nessa VPN, configuramos a tabela NAT para que os pacotes saindo do servidor com o IP virtual de origem (da rede 10.0.0.0/24) sejam traduzidos para o IP real da interface de rede do servidor (wlo1) antes de irem para a internet, é o que a regra MASQUERADE faz. 

Isso é necessário porque a internet não sabe rotear pacotes de volta para um endereço IP privado/virtual como 10.0.0.2. Quando a resposta chega da internet, o iptables consulta a tabela de conexões que ele mesmo criou no momento do MASQUERADE e faz o caminho inverso: troca o IP de destino do pacote (que chegou com o IP real do servidor) de volta para o IP virtual do cliente (10.0.0.2), permitindo que ele seja encaminhado corretamente pelo túnel.

### Handshake 
Para estabelecer o túnel, cliente e servidor trocam duas mensagens (handshake de 1 Round Trip time, uma ida e volta):
O cliente gera um novo par de chaves efêmeras e envia uma mensagem de iniciação ao servidor. Essa mensagem contém sua chave pública efêmera, sua chave pública estática (criptografada), um timestamp para evitar ataques de repetição e um MAC (Message Authentication Code), utilizado para garantir a integridade e autenticidade da mensagem. A criptografia e a autenticação dessas informações são baseadas nos resultados dos cálculos Diffie-Hellman, utilizando a chave pública estática do servidor, previamente conhecida pelo cliente.

Ao receber a mensagem, o servidor utiliza a chave pública estática do cliente, previamente configurada no arquivo wg0.conf, para validar a identidade do cliente e verificar a autenticidade da mensagem. Em seguida, gera seu próprio par de chaves efêmeras e envia uma mensagem de resposta contendo sua chave pública efêmera e um MAC de confirmação.

Após a troca dessas duas mensagens, ambos os dispositivos executam os cálculos Diffie-Hellman envolvendo suas chaves estáticas e efêmeras. Os segredos compartilhados resultantes são processados pela KDF, que deriva as chaves simétricas da sessão. A partir desse momento, o handshake é concluído e todo o tráfego passa a ser criptografado pelo algoritmo ChaCha20.

### Roteamento

O roteamento é o processo pelo qual o sistema decide, com base no endereçamento IP e na tabela de rotas, por qual interface e para qual próximo salto um pacote deve seguir até seu destino. Na configuração dessa VPN, a regra de NAT foi colocada na cadeia POSTROUTING, que é aplicada depois que a decisão de roteamento já foi tomada pelo kernel, mas antes de o pacote sair pela interface de rede.

### Firewall

O firewall funciona como um filtro de pacotes: todo pacote que entra ou sai da máquina passa por ele antes de ser processado ou encaminhado. No Linux, o firewall é implementado pelo netfilter (o subsistema do kernel) e configurado através do iptables. Ele organiza as regras em tabelas (como filter, nat, mangle) e, dentro de cada tabela, em cadeias (como INPUT, OUTPUT, FORWARD, PREROUTING, POSTROUTING), que são percorridas em pontos específicos do fluxo de entrada e saída dos pacotes.

### Uso do UDP

O WireGuard utiliza o protocolo UDP (User Datagram Protocol) como camada de transporte para encapsular e transmitir os pacotes da VPN. Como o UDP não estabelece uma conexão antes da transmissão dos dados, não realiza controle de fluxo, retransmissão de pacotes ou confirmação de recebimento, reduz a sobrecarga e a latência da comunicação.

A escolha do UDP evita problemas conhecidos como TCP-over-TCP, que podem ocorrer quando um túnel VPN utiliza TCP para transportar outro tráfego TCP. Nessa situação, ambos os protocolos tentam realizar controle de congestionamento e retransmissão de pacotes, causando perda de desempenho. Utilizando UDP, o WireGuard deixa que esses mecanismos sejam tratados apenas pelos protocolos das camadas superiores quando necessário.

Além disso, o UDP permite que o próprio WireGuard implemente seus mecanismos de handshake, gerenciamento das chaves criptográficas e controle das sessões, sem depender das funcionalidades oferecidas pelo protocolo de transporte. Por padrão, o WireGuard utiliza a porta UDP 51820, embora qualquer outra porta UDP possa ser configurada conforme a necessidade da aplicação.

### Interface wg0

Ao iniciar o WireGuard, é criada uma interface de rede virtual chamada wg0. Essa interface funciona como um adaptador de rede lógico, semelhante a uma interface Ethernet (eth0) ou Wi-Fi (wlo1), porém sem estar associada diretamente a um dispositivo físico.

No projeto desenvolvido, a interface wg0 recebeu o endereço 10.0.0.1/24, enquanto o cliente Android recebeu o endereço 10.0.0.2/24. Esses endereços pertencem exclusivamente à rede privada da VPN e são utilizados para a comunicação entre os dispositivos conectados ao túnel.

Quando uma aplicação envia um pacote destinado à rede da VPN, o sistema operacional o encaminha para a interface wg0. O WireGuard intercepta esse pacote, criptografa seu conteúdo utilizando as chaves da sessão e o encapsula em um pacote UDP, que é transmitido pela interface física da máquina (no projeto, wlo1).

No lado do destinatário, ocorre o processo inverso: o pacote UDP é recebido pela interface física, o WireGuard valida sua autenticidade, descriptografa o conteúdo e injeta o pacote original na interface wg0, permitindo que o sistema operacional o trate como um pacote IP comum.
