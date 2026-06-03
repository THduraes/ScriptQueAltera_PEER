# 🔄 Automação de Failover de VPN IPsec (Openswan) no Linux

Este repositório documenta a implementação de Alta Disponibilidade (HA) em uma conexão VPN IPsec utilizando Openswan. A solução resolve intermitências na entrega de configurações da rede (Mode Config/XAUTH) automatizando a troca de rotas (peers) de forma inteligente.

**Autor:** Thiago dos Santos Durães – Técnico em Informática I  

---

## 1. Motivo das Alterações (Diagnóstico)
A máquina virtual Linux (ISH Box) responsável por fechar o túnel VPN com a matriz apresentava travamentos frequentes. O serviço do IPsec (`ipsec auto --status`) mostrava que as Fases 1 e 2 da criptografia fechavam corretamente, mas a conexão ficava paralisada no seguinte estado:

> `STATE_XAUTH_I1: XAUTH client - awaiting CFG_set`

O firewall primário remoto (Fortigate) autenticava o usuário, mas falhava intermitentemente em devolver as rotas e o IP virtual. O objetivo desta implementação foi criar um mecanismo autônomo de *Failover* (troca de links) para eliminar a necessidade de intervenção humana em caso de falhas.

---

## 2. Implementação Passo a Passo (Os 4 Pilares)

Para alcançar a automação, a arquitetura foi implementada seguindo 4 etapas fundamentais descritas abaixo.

### PILAR 1: Separação de Peers (Configuração do IPsec)
**O que fazemos:** O perfil único de conexão precisou ser dividido em dois blocos distintos (`fortigate-primario` e `fortigate-secundario`), permitindo que o sistema operacional trate as rotas de forma independente.

#### **Como fazer:**
#### Edite o arquivo de configuração do IPsec:

```bash
nano /etc/ipsec.conf

# Add connections here

conn fortigate-primario
        type=tunnel
        authby=secret
        auto=start
        pfs=yes
        compress=yes
        aggrmode=yes
        ikev2=no
# Rede do Linux
        leftid=@ishboxh
        leftxauthclient=yes
        leftmodecfgclient=no
        left=192.168.1.201
        leftxauthusername=IH4658338
        leftsourceip=10.114.243.247
# Rede Fortigate - Link Principal
        right=teftgv.martins.com.br
        rightsubnet=0.0.0.0/0
        rightnexthop=%defaultroute
        rightxauthserver=yes
        rightmodecfgserver=yes
# Phase 1 e 2
        ike=3des-sha1;modp1536
        keylife=28800s
        phase2=esp
        phase2alg=aes128-sha1;modp1536

conn fortigate-secundario
        type=tunnel
        authby=secret
        auto=add
        pfs=yes
        compress=yes
        aggrmode=yes
        ikev2=no
# Rede do Linux (Mesma origem)
        leftid=@ishboxh
        leftxauthclient=yes
        leftmodecfgclient=no
        left=192.168.1.201
        leftxauthusername=IH4658338
        leftsourceip=10.114.243.247
# Rede Fortigate - Link Secundário (Backup)
        right=teftgv2.martins.com.br
        rightsubnet=0.0.0.0/0
        rightnexthop=%defaultroute
        rightxauthserver=yes
        rightmodecfgserver=yes
# Phase 1 e 2
        ike=3des-sha1;modp1536
        keylife=28800s
        phase2=esp
        phase2alg=aes128-sha1;modp1536
```
#### Após salvar o arquivo, reinicie o serviço para injetar os novos perfis na memória:
```bash
service ipsec restart
```

### PILARES 2 e 3: Script Watchdog e Limpeza de Rotas

O que fazemos: Criamos um script Shell para monitorar o tráfego real. Ele dispara pings contínuos para um IP interno confiável (172.19.2.2). Se o ping falhar, ele atua ativando o parâmetro --unroute para forçar o Linux a limpar a tabela de roteamento (evitando o erro cannot route), e então sobe o peer reserva.

#### Como fazer:
Crie o arquivo do script:

```bash
nano /opt/scripts/vpn-failover.sh
```
#### Cole a inteligência da automação abaixo:
```bash
#!/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# IP interno da Matriz para teste de fluxo de dados
ALVO="172.19.2.2"

# Arquivo na memória (RAM) para salvar a posição atual
MEMORIA="/tmp/vpn_status.txt"

# Define o primário como padrão na primeira execução
if [ ! -f "$MEMORIA" ]; then
    echo "primario" > "$MEMORIA"
fi

STATUS_ATUAL=$(cat "$MEMORIA")

# Tenta pingar o alvo por dentro do túnel
if ping -c 4 -W 2 $ALVO > /dev/null 2>&1; then
    exit 0 # Respondeu, túnel OK. Encerra a execução.
else
    # PING FALHOU - Inicia processo de Failover e Limpeza de Rotas
    if [ "$STATUS_ATUAL" == "primario" ]; then
        ipsec auto --down fortigate-primario
        ipsec auto --unroute fortigate-primario # Etapa crítica: Limpa a rota travada
        ipsec auto --up fortigate-secundario
        echo "secundario" > "$MEMORIA"
    else
        ipsec auto --down fortigate-secundario
        ipsec auto --unroute fortigate-secundario # Etapa crítica: Limpa a rota travada
        ipsec auto --up fortigate-primario
        echo "primario" > "$MEMORIA"
    fi
fi
```
#### Após salvar, transforme o arquivo em um executável:
```bash
chmod +x /opt/scripts/vpn-failover.sh
```

### PILAR 4: Agendamento Automático
O que fazemos: Utilizamos o daemon cron nativo do sistema para executar o Script Watchdog de forma contínua e silenciosa a cada 1 minuto.

#### Como fazer:
Abra as tarefas agendadas do Linux:
```bash
crontab -e
```
Adicione a instrução abaixo na última linha (lembre-se de dar um Enter no final para criar uma linha em branco abaixo dela):
```bash
* * * * * /opt/scripts/vpn-failover.sh
```

### -------------------------TESTE----------------------------
## 3. Como Testar e Acompanhar a Troca de Peers
Para comprovar a eficiência da separação de peers e da limpeza de rotas, simule um incidente utilizando dois terminais:

Monitorando a Tomada de Decisão (Terminal 1)
Prenda a tela na leitura do arquivo de memória do script, com atualização a cada 2 segundos:
```bash
watch -n 2 cat /tmp/vpn_status.txt
```
(Deve exibir primario).

Derrubando o Peer (Terminal 2)
Force a queda manual do link principal:
```bash
ipsec auto --down fortigate-primario
```

Validação do Failover
No Terminal 1, assim que o minuto virar, o script identificará a perda de pacotes e efetuará a limpeza (--unroute) e a subida do link de backup. A tela mudará automaticamente de primario para secundario.

No Terminal 2, confirme o fluxo de dados atestando o funcionamento do peer secundário:
```bash
ping 172.19.2.2
```
Se houver resposta, o mecanismo autônomo está totalmente operacional.


## 4. Administração e Comandos Úteis no Dia a Dia

Para facilitar o monitoramento e a manutenção da VPN pela equipe de infraestrutura, abaixo estão os principais comportamentos e comandos de verificação do sistema.

### 4.1 Comportamento do Restart Manual e Sincronização
Por padrão estrutural, sempre que o serviço do IPsec for reiniciado manualmente (`service ipsec restart`), o Openswan limpará as rotas e lerá o arquivo `/etc/ipsec.conf` do zero. Como o peer principal possui a flag `auto=start`, **o sistema sempre tentará subir pelo link primário após um restart.**

⚠️ **Atenção (Sincronização de Estado):** Caso o sistema estivesse operando pelo link secundário antes do restart, o arquivo de memória do script manterá a palavra `secundario` gravada, gerando uma dessincronização com o estado real da rede. Para evitar falsos positivos no script, sempre que executar um restart manual, rode o comando abaixo em seguida para "resetar" a memória do Failover:

```bash
service ipsec restart
echo "primario" > /tmp/vpn_status.txt
```
### 4.2 Como verificar qual Peer está ativo no momento
Existem três formas de validar qual túnel está roteando os pacotes da rede:

## Opção A: Consulta Rápida (Memória do Script)
Lê a última decisão tomada pelo script de automação.
```bash
cat /tmp/vpn_status.txt
```

## Opção B: Status Oficial do IPsec (Fase 2)
Filtra os logs do serviço Openswan para exibir apenas o túnel que fechou a criptografia e estabeleceu o SA (Security Association) com sucesso. O nome do peer ativo aparecerá no início da frase de saída.
```bash
ipsec auto --status | grep "IPsec SA established"
```

## Opção C: Tabela Criptográfica do Kernel (Avançado)
Inspeciona diretamente o Kernel do Linux para listar os endereços IP reais (origem e destino) que estão criptografando e descriptografando os pacotes (XFRM state) naquele exato milissegundo.
```bash
ip xfrm state
```
