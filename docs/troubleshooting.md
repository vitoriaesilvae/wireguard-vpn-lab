## Problemas encontrados e soluções

### 1 Falha ao iniciar o serviço

Ao iniciar o serviço com `sudo systemctl start wg-quick@wg0`, aparecia a mensagem:

```
Job for wg-quick@wg0.service failed…
```

**Diagnóstico:**

1. Rodei o comando `sudo systemctl status wg-quick@wg0` para verificar erros no arquivo wg0;
2. O comando retornou a linha na qual havia um erro de digitação

**Solução:** corrigir os erros de ortografia no arquivo de configuração.

### 2 Cliente conectava, mas não acessava a internet

**Diagnóstico, passo a passo:**

1. Verifiquei se a interface `wg0` estava roteando:
    
    ```bash
    ip route #verifica a tabela de roteamento
    ```
    
    A interface `wg0` aparecia na lista de roteamento.
    
2. Verifiquei se a regra na tabela NAT estava funcionando:
    
    ```bash
    sudo iptables -t nat -L POSTROUTING
    ```
    
    O MASQUERADE da VPN estava ativo.
    
3. Verifiquei se a porta UDP 51820 estava liberada no firewall:
    
    ```bash
    sudo ufw status
    ```
    
    A porta estava ativa e aberta.
    
    **OBS :** Se a porta não aparecer liberada, o comando para liberá-la é `sudo ufw allow 51820/udp`.
    
4. Com o cliente conectado, verifiquei se a porta UDP 51820 estava recebendo pacotes do Android:
    
    ```bash
    sudo tcpdump -ni any udp port 51820
    ```
    
    Se nenhuma linha aparecer, significa que a porta não está recebendo pacotes. Nesse caso, a porta estava recebendo os pacotes normalmente.
    
    - `tcpdump`: analisador de pacotes que captura o tráfego que passa pela interface de rede e exibe suas informações (uma ferramenta muito usada para diagnóstico de redes). Sem filtro, captura todos os pacotes que passam pela interface padrão.
    - `-n`: não converte os IPs em nomes de domínio, deixando a captura mais rápida.
    - `-i any`: captura em qualquer interface de rede.
    - `udp`: filtra apenas pacotes UDP.
    - `port 51820`: considera apenas pacotes que usam a porta 51820 (a porta configurada para a VPN).
5. Verifiquei se as chaves do cliente e do servidor no arquivo `wg0.conf` estavam corretas, e não estavam.
    - O servidor e o cliente conseguiam se conectar, mas a autenticação via handshake falhava e os pacotes recebidos pelo servidor eram descartados.
    - Após corrigir as chaves, reiniciei o WireGuard:
        
        ```bash
        sudo systemctl restart wg-quick@wg0
        ```
        
    - Após reiniciar o serviço, consegui acessar a internet normalmente pela VPN.

---

 **Solução:** corrigir as chaves do arquivo wg0.
