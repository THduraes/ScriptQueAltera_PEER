# 🔄 Automação de Failover de VPN IPsec (Openswan) no Linux

Este repositório documenta a implementação de Alta Disponibilidade (HA) em uma conexão VPN IPsec utilizando Openswan. A solução resolve intermitências na entrega de configurações da rede (Mode Config/XAUTH) automatizando a troca de rotas (peers) de forma inteligente.

**Autor:** Thiago dos Santos Durães – Técnico em Informática I  

---

## 1. Motivo das Alterações
A máquina virtual Linux (ISH Box) responsável por fechar o túnel VPN com a matriz apresentava travamentos frequentes. O serviço do IPsec (`ipsec auto --status`) mostrava que as Fases 1 e 2 da criptografia fechavam corretamente, mas a conexão ficava paralisada no seguinte estado:

> `STATE_XAUTH_I1: XAUTH client - awaiting CFG_set`

**Diagnóstico:** O firewall primário remoto (Fortigate) autenticava o usuário, mas falhava em devolver as rotas e o IP virtual. Para restabelecer a rede, era necessário intervir manualmente e trocar a conexão para o link secundário da matriz. O objetivo desta implementação foi criar um mecanismo autônomo de *Failover* para eliminar a necessidade de intervenção humana.

---

## 2. Quais Foram as Alterações
Para alcançar a automação, a arquitetura da VPN foi modificada em três frentes:

1. **Separação de Peers no IPsec:** O perfil único de conexão foi dividido em dois blocos distintos (`fortigate-primario` e `fortigate-secundario`) no arquivo `/etc/ipsec.conf`.
2. **Script Watchdog (Monitoramento):** Criação de um script Shell que monitora o tráfego de dados reais por dentro do túnel disparando pings para um IP interno confiável (`172.19.2.2`). 
3. **Gerenciamento de Rotas Ocupadas:** Inclusão do parâmetro `--unroute` no momento da queda da VPN para forçar o Linux a limpar a tabela de roteamento, evitando o erro clássico `cannot route -- route already in use`.
4. **Agendamento Automático:** Utilização do `cron` para executar a validação de conectividade a cada 1 minuto.

---

## 3. Como Implementar (Passo a Passo e Comandos)

### Passo 1: Configurar os Peers no Openswan
Edite o arquivo de configuração do IPsec:
```bash
nano /etc/ipsec.conf

