# WireGuard VPN Lab

Projeto desenvolvido para estudar o funcionamento do WireGuard, incluindo:

- instalação do servidor
- criação de clientes
- configuração de NAT
- IP Forwarding
- Firewall
- Handshake
- Troubleshooting

## Tecnologias

- Linux Mint
- WireGuard
- iptables
- UFW
- tcpdump
- systemd

## Objetivos

- Construir uma VPN funcional
- Entender como o WireGuard funciona
- Aprender técnicas de diagnóstico de redes

## Conceitos abordados

- VPN
- Túnel criptografado
- Criptografia assimétrica e simétrica
- Handshake
- Diffie-Hellman (Curve25519)
- ChaCha20
- Perfect Forward Secrecy (PFS)
- Interface `wg0`
- NAT (MASQUERADE)
- IP Forwarding
- Firewall (UFW)
- Captura de pacotes com `tcpdump`
- Diagnóstico de conectividade

## Estrutura da documentação

```text
wireguard-vpn-lab/
├── comandos/
    └── comandos_utilizados.md
├── configurações/
    ├── cliente.conf.example
    └── servidor.conf.example
├── docs/
    ├── arquitetura.md
    ├── instalacao.md
    ├── troubleshooting.md
    ├── comandos.md
    └── imagens/
        ├── Topologia.png
        └── Fluxo_pacotes.png
├── .gitignore
├── LICENSE
├── README.md
```
