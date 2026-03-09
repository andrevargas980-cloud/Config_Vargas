# Laboratório GNS3 — Provedor do Zero (PPPoE)

Este laboratório demonstra, passo a passo, como construir um **provedor de internet do zero** usando **GNS3 + MikroTik**, simulando:

- Uplink com provedor (PPPoE Client)
- Rede de gerência com acesso à internet
- Servidor PPPoE para clientes
- Autenticação de clientes (ppp-secrets)
- NAT/CGNAT
- Estrutura base para evolução (MK-AUTH, BGP, etc.)

---

## 1. Objetivo do Laboratório

Criar um cenário funcional onde:

- O roteador **ISP-Padrao2** atua como o provedor
- Recebe internet via **PPPoE Client** (PROVEDOR-Y)
- Entrega internet para:
  - Rede de **Gerência**
  - **Clientes PPPoE** (Luca, Débora, Isabela)

---

## 2. Topologia (Visão Lógica)

Componentes principais:

- **NAT1 (Cloud)**: Simula a internet
- **PROVEDOR-Y**: Provedor upstream
- **ISP-Padrao2 (MikroTik)**: Roteador principal do provedor
- **OLT-SW-ACCESS**: Switch de acesso
- **Roteadores Clientes**: Luca, Débora, Isabela
- **PC Gerência**: Máquina administrativa
- **MK-AUTH**: Autenticação (backlog)

---

## 3. Plano de Endereçamento (Exemplo)

### 3.1 Uplink (ISP → Provedor)

- Tipo: PPPoE Client
- Interface: WAN (ex: ether2)
- IP: Atribuído pelo provedor
- DNS: Via PPP ou manual

### 3.2 Rede de Gerência

| Item | Valor |
|---|---|
| Rede | 192.168.0.0/24 |
| Gateway | 192.168.0.1 |
| PC Gerência | DHCP ou 192.168.0.2 |

### 3.3 Pool de Clientes (CGNAT)

| Item | Valor |
|---|---|
| Pool Clientes | 100.64.1.0/24 |
| Local Address PPP | 10.64.1.1 |
| DNS Clientes | 8.8.8.8 |

---

## 4. Configuração do ISP-Padrao2 (MikroTik)

### Etapa A — PPPoE Client (Uplink)

1. Criar PPPoE Client na interface WAN
2. Informar usuário e senha do provedor
3. Habilitar:
   - `Add Default Route`
   - `Use Peer DNS`

**Validação**
- `ping 8.8.8.8`
- `ping google.com`
- Verificar rota default

---

### Etapa B — Rede de Gerência + NAT

1. Atribuir IP à interface de gerência:
2. Criar servidor DHCP (opcional)
3. Criar regra NAT:
- `src-address=192.168.0.0/24`
- `out-interface=PPPoE-WAN`
- `action=masquerade`

**Validação**
- PC gerência navega na internet
- DNS resolve nomes

---

### Etapa C — Servidor PPPoE (Clientes)

1. Criar Pool de IPs:


2. Criar PPP Profile:
- Local Address: `10.64.1.1`
- Remote Address: Pool Clientes
- DNS: `8.8.8.8`

3. Criar PPPoE Server:
- Interface: ACESSO (ligada ao switch)
- Profile: Criado acima

4. Criar NAT para clientes:
- `src-address=100.64.1.0/24`
- `out-interface=PPPoE-WAN`
- `action=masquerade`

---

### Etapa D — Criar Clientes PPPoE

Criar os usuários (PPP Secrets):

| Cliente | Usuário |
|---|---|
| Luca | luca |
| Débora | debora |
| Isabela | isabela |

Parâmetros:
- Service: `pppoe`
- Profile: Perfil PPP criado
- Password: Definir

**Validação**
- PPP → Active Connections
- Cliente recebe IP
- Cliente navega

---

## 5. Troubleshooting Comum

### Gerência não navega
- IP e gateway corretos
- DNS configurado
- NAT ativo
- Firewall não bloqueando forward

### PPPoE cliente conecta/desconecta
- Interface errada no PPPoE Server
- Usuário inexistente
- Pool esgotado
- Loop no switch

### Tráfego anormal em cliente
- Verificar Torch
- Conferir loops L2
- Downloads automáticos ativos

---

## 6. Checklist de Conclusão

- [ ] PPPoE Client conectado
- [ ] Rota default ativa
- [ ] Gerência com acesso à internet
- [ ] PPPoE Server ativo
- [ ] Clientes autenticando
- [ ] Clientes navegando
- [ ] Configuração exportada

---

## 7. Próximos Passos (Backlog)

- Integração com MK-AUTH (RADIUS)
- Planos de velocidade (Queues)
- CGNAT avançado
- Firewall por perfil
- BGP com provedor
- Monitoramento (Zabbix)

---

## 8. Observações

Este laboratório é **didático** e tem como objetivo ensinar os conceitos fundamentais de um provedor real.

Não deve ser usado diretamente em produção sem ajustes de segurança e capacidade.

# Aula 02 — Integração com Sistema de Gestão, Planos e Financeiro

## Objetivo da aula
Nesta etapa o laboratório evolui de “rede funcionando” para um **provedor operacional**, integrando:
- Autenticação de clientes com o sistema de gestão
- Criação de **planos de velocidade**
- Associação de clientes aos planos
- Testes de velocidade
- Início da estrutura financeira (contas e boletos)

---

## Pré-requisitos
- Aula 01 concluída
- PPPoE Server funcionando no MikroTik
- Clientes (Luca, Débora, Isabela) conectando via PPPoE
- Acesso ao **sistema de gestão do provedor** (web)

---

## Etapa 1 — Acesso e validação do sistema de gestão
1. Acessar o painel web do sistema de gestão
2. Validar:
   - Login funcionando
   - Dashboard carregando
   - Comunicação com a VM ativa

**Checklist**
- [ ] Sistema acessível via navegador
- [ ] Dashboard exibindo clientes
- [ ] Sessão estável (sem erros de login)

---

## Etapa 2 — Criação dos planos de internet
1. Acessar: **Controle → Planos / Produtos**
2. Criar planos de teste:
   - **Plano 100 Mbps**
     - Velocidade: 100 Mbps
     - Valor: R$ 99,00
   - **Plano 600 Mbps**
     - Velocidade: 600 Mbps
     - Valor: R$ 299,00
3. Salvar os planos

> Obs: nesta etapa os planos **ainda não limitam a banda automaticamente**, pois a integração com o MikroTik ainda está sendo ajustada.

**Checklist**
- [ ] Plano 100 Mbps criado
- [ ] Plano 600 Mbps criado
- [ ] Planos visíveis no sistema

---

## Etapa 3 — Cadastro de clientes no sistema
Criar clientes no sistema de gestão (nomes de teste):

### Clientes
- Luca
- Débora
- Isabela

Para cada cliente:
1. Informar nome e dados básicos
2. Criar login de acesso
3. Salvar cadastro

**Checklist**
- [ ] Luca cadastrado
- [ ] Débora cadastrada
- [ ] Isabela cadastrada

---

## Etapa 4 — Associação de clientes aos planos
1. Abrir cadastro do cliente
2. Vincular o **plano de internet**
3. Salvar alterações

> Importante: após associar o plano, pode ser necessário **desconectar e reconectar o PPPoE** para aplicar corretamente.

**Checklist**
- [ ] Cliente vinculado ao plano correto
- [ ] Alterações salvas com sucesso

---

## Etapa 5 — Ajustes de autenticação (integração com o MikroTik)
Durante os testes, foi identificado que:
- Alguns parâmetros não estavam definidos no profile
- Foi necessário revisar:
  - Perfil do cliente
  - Pool de IP
  - Associação correta do plano

Ações realizadas:
1. Revisar perfil do cliente no sistema
2. Garantir que o cliente herda os parâmetros do profile
3. Derrubar e subir a sessão PPPoE novamente

**Checklist**
- [ ] Cliente autentica corretamente
- [ ] Sessão aparece ativa no MikroTik
- [ ] Cliente navega após reconexão

---

## Etapa 6 — Testes de velocidade
1. Executar teste de velocidade nos clientes
2. Observar:
   - Velocidade entregue
   - Latência
   - Estabilidade

Observações do laboratório:
- Velocidade abaixo do esperado em alguns testes
- Testes diretos (sem switch intermediário) apresentaram melhor desempenho
- Limitação pode estar relacionada à VM ou ao ambiente GNS3

**Checklist**
- [ ] Cliente navega
- [ ] Teste de velocidade executado
- [ ] Gargalos identificados

---

## Etapa 7 — Estrutura financeira (contas e boletos)
1. Criar **conta financeira do provedor**
   - Tipo: Boleto próprio do provedor
   - Dados básicos da empresa (teste)
2. Associar conta aos clientes
3. Gerar cobranças manualmente

Ações realizadas:
- Criação de conta financeira
- Geração de cobrança para clientes
- Registro de pagamento manual (dinheiro/teste)

**Checklist**
- [ ] Conta financeira criada
- [ ] Cliente com cobrança gerada
- [ ] Registro financeiro funcionando

---

## Etapa 8 — Validação final
Verificar no dashboard:
- Total de clientes
- Clientes online
- Clientes com cobrança
- Status geral do sistema

**Resultado esperado**
- Clientes conectados via PPPoE
- Planos atribuídos
- Sistema de gestão refletindo o estado real da rede

---

## Problemas encontrados (lições da aula)
- Parâmetros ausentes em profiles causam falha de autenticação
- Mudanças exigem reconexão do cliente
- Testes de velocidade em laboratório virtual sofrem limitações
- Financeiro exige configuração completa antes de automatizar

---

## Definição de pronto — Aula 02
- [ ] Clientes autenticando via sistema
- [ ] Planos criados e vinculados
- [ ] Testes de velocidade executados
- [ ] Cobrança gerada no sistema
- [ ] Dashboard refletindo clientes online

---

## Próxima aula (Aula 03 — sugestão)
- Limitação real de banda (queues / profiles)
- Integração Radius (MK-AUTH)
- Automação total: cliente conecta → plano aplica → cobrança gera

