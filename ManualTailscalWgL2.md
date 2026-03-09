# Manual NCX — MikroTik NAT-safe com Tailscale + WireGuard + EoIP (L2)

## Objetivo
Conectar duas pontas MikroTik atrás de NAT sem IP público fixo:
- **Underlay:** Tailscale (container)
- **L3 overlay:** WireGuard (`wg-lab`)
- **L2 overlay:** EoIP sobre WG (`eoip-wg`) + bridge com `ether3`

---

## 0) Pré-requisitos
- RouterOS v7 com `container=yes`
- Disco/`root-dir` disponível (ex.: `disk1/ts`)
- Acesso ao painel Tailscale Admin
- (Recomendado) AuthKey **reusable + preauthorized + non-ephemeral**

---

## 1) Endereçamento de referência
- **Router A**
  - `bridge-tailscale`: `172.17.0.1/30`
  - `veth-ts`: `172.17.0.2/30`
  - WG: `10.77.77.1/30`
- **Router B**
  - `bridge-tailscale`: `172.17.0.5/30`
  - `veth-ts`: `172.17.0.6/30`
  - WG: `10.77.77.2/30`

---

## 2) Setup base no MikroTik (container + rede local do container)

### 2.1 Router A
```rsc
/interface bridge add name=bridge-tailscale
/ip address add address=172.17.0.1/30 interface=bridge-tailscale
/interface veth add name=veth-ts address=172.17.0.2/30 gateway=172.17.0.1
/interface bridge port add bridge=bridge-tailscale interface=veth-ts

/ip firewall nat add chain=srcnat src-address=172.17.0.0/30 action=masquerade comment="ts-out"
/ip firewall nat add chain=srcnat src-address=172.17.0.0/30 dst-address=172.17.0.4/30 action=accept comment="no-nat-ts-bridge"
/ip route add dst-address=172.17.0.4/30 gateway=172.17.0.2

/container config set registry-url=https://registry-1.docker.io tmpdir=disk1/tmp
/container add remote-image=tailscale/tailscale:latest interface=veth-ts root-dir=disk1/ts logging=yes start-on-boot=yes
```

### 2.2 Router B
```rsc
/interface bridge add name=bridge-tailscale
/ip address add address=172.17.0.5/30 interface=bridge-tailscale
/interface veth add name=veth-ts address=172.17.0.6/30 gateway=172.17.0.5
/interface bridge port add bridge=bridge-tailscale interface=veth-ts

/ip firewall nat add chain=srcnat src-address=172.17.0.4/30 action=masquerade comment="ts-out"
/ip firewall nat add chain=srcnat src-address=172.17.0.4/30 dst-address=172.17.0.0/30 action=accept comment="no-nat-ts-bridge"
/ip route add dst-address=172.17.0.0/30 gateway=172.17.0.6

/container config set registry-url=https://registry-1.docker.io tmpdir=disk1/tmp
/container add remote-image=tailscale/tailscale:latest interface=veth-ts root-dir=disk1/ts logging=yes start-on-boot=yes
```

---

## 3) Processo completo do Tailscale (com e sem AuthKey)

## 3A) Fluxo recomendado com AuthKey (via `container envs`)
> Use a **mesma key** nos dois ou uma por roteador (preferível uma por roteador).

### Router A
```rsc
/container/envs/add list=ts_envs key=TS_AUTHKEY value="<TSKEY_AQUI>"
/container/envs/add list=ts_envs key=TS_STATE_DIR value=/var/lib/tailscale
/container/envs/add list=ts_envs key=TS_USERSPACE value=true
/container/envs/add list=ts_envs key=TS_EXTRA_ARGS value="--accept-routes=true --advertise-routes=172.17.0.0/30"

/container/set [find where name="tailscale:latest"] envlist=ts_envs start-on-boot=yes
/container/stop [find where name="tailscale:latest"]
:delay 2s
/container/start [find where name="tailscale:latest"]
```

### Router B
```rsc
/container/envs/add list=ts_envs key=TS_AUTHKEY value="<TSKEY_AQUI>"
/container/envs/add list=ts_envs key=TS_STATE_DIR value=/var/lib/tailscale
/container/envs/add list=ts_envs key=TS_USERSPACE value=true
/container/envs/add list=ts_envs key=TS_EXTRA_ARGS value="--accept-routes=true --advertise-routes=172.17.0.4/30"

/container/set [find where name="tailscale:latest"] envlist=ts_envs start-on-boot=yes
/container/stop [find where name="tailscale:latest"]
:delay 2s
/container/start [find where name="tailscale:latest"]
```

> Se for reaplicar, remova entradas antigas antes de recriar:
```rsc
/container/envs/print where list=ts_envs
# remover por .id se necessário:
#/container/envs/remove <id>
```

### Aprovação no Tailscale Admin
- `Machines` -> confirmar os dois nodes online
- `Routes` -> aprovar:
  - `172.17.0.0/30`
  - `172.17.0.4/30`

## 3B) Fallback sem key (login por link)
Se AuthKey falhar por policy:

```rsc
/log print where message~"AuthURL"
```

Abrir links `https://login.tailscale.com/a/...` de cada roteador.
Depois aprovar as duas routes no Admin.

## 3C) Se as máquinas forem removidas do Admin
1. Reiniciar container:
```rsc
/container stop [find where name="tailscale:latest"]
:delay 2s
/container start [find where name="tailscale:latest"]
```
2. Capturar novos `AuthURL` e autenticar novamente.

## 3D) Verificações obrigatórias do Tailscale
```rsc
/container print
/log print where message~"peerapi: serving on http://100"
/log print where message~"AuthURL"
```

- `container` deve estar `R` (running)
- deve haver `peerapi: serving on http://100.x.x.x:porta`
- não pode ficar ciclando em `AuthURL` sem autenticar

---

## 4) WireGuard overlay

### 4.1 Criar interface e IP
#### Router A
```rsc
/interface wireguard add name=wg-lab listen-port=51820
/ip address add address=10.77.77.1/30 interface=wg-lab
```
#### Router B
```rsc
/interface wireguard add name=wg-lab listen-port=51821
/ip address add address=10.77.77.2/30 interface=wg-lab
```

### 4.2 Coletar public-keys WG
```rsc
/interface wireguard print detail
```

### 4.3 Criar peers WG
#### Router A (peer = Router B)
```rsc
/interface wireguard peers add interface=wg-lab public-key="<PUBKEY_WG_B>" endpoint-address=<TS_IP_B> endpoint-port=<PORTA_TS_B> allowed-address=10.77.77.2/32 persistent-keepalive=10s
```

#### Router B (peer = Router A)
```rsc
/interface wireguard peers add interface=wg-lab public-key="<PUBKEY_WG_A>" endpoint-address=<TS_IP_A> endpoint-port=<PORTA_TS_A> allowed-address=10.77.77.1/32 persistent-keepalive=10s
```

> **Importante:** se o endpoint Tailscale mudar (100.x), atualizar manualmente `endpoint-address` no peer WG.

### 4.4 Validar WG
- A:
```rsc
ping 10.77.77.2 count=4
```
- B:
```rsc
ping 10.77.77.1 count=4
```

---

## 5) EoIP sobre WireGuard (L2)

### Router A
```rsc
/interface eoip add name=eoip-wg local-address=10.77.77.1 remote-address=10.77.77.2 tunnel-id=88 mtu=1500
/interface bridge add name=bridge-l2test
/interface bridge port add bridge=bridge-l2test interface=ether3
/interface bridge port add bridge=bridge-l2test interface=eoip-wg
```

### Router B
```rsc
/interface eoip add name=eoip-wg local-address=10.77.77.2 remote-address=10.77.77.1 tunnel-id=88 mtu=1500
/interface bridge add name=bridge-l2test
/interface bridge port add bridge=bridge-l2test interface=ether3
/interface bridge port add bridge=bridge-l2test interface=eoip-wg
```

Validar:
```rsc
/interface eoip print detail
/interface bridge port print where bridge=bridge-l2test
```

---

## 6) Testes finais
- `ping 10.77.77.1/10.77.77.2` OK
- `eoip-wg` sem `I` (inactive)
- Dispositivos nas duas `ether3` com L2 ponta-a-ponta
- Teste MTU 1500 em host Linux:
```bash
ping -M do -s 1472 <ip-destino>
```

---

## 7) Troubleshooting
1. **Container parado**
```rsc
/container start [find where name="tailscale:latest"]
```
2. **`invalid key: unable to validate API key`**
- key inválida/revogada/policy bloqueando
3. **AuthURL toda hora**
- sessão não persistida/policy forçando novo login
4. **WG sem RX/TX**
- endpoint Tailscale remoto desatualizado
5. **EoIP inactive**
- WG sem handshake

---

## 8) Scripts da skill
- `skills/ncx-mikrotik-tailscale-wg-eoip/scripts/router-a-setup.rsc`
- `skills/ncx-mikrotik-tailscale-wg-eoip/scripts/router-b-setup.rsc`
- `skills/ncx-mikrotik-tailscale-wg-eoip/scripts/check-status.rsc`
- `skills/ncx-mikrotik-tailscale-wg-eoip/scripts/rollback-disable-l2.rsc`
