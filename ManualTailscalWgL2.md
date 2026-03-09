# Manual NCX — MikroTik NAT-safe com Tailscale + WireGuard (L2)

## Objetivo
Conectar duas pontas MikroTik atrás de NAT sem IP público fixo:
- **Underlay:** Tailscale (container)
- **L3 overlay:** WireGuard (`wg-lab`)
- **L2 overlay:** EoIP sobre WG (`eoip-wg`) + bridge com `ether3`

## Topologia de referência
- Router A (site A)
- Router B (site B)
- `bridge-l2test` em ambos com `ether3 + eoip-wg`

## Endereçamento padrão
- A: `172.17.0.1/30` (bridge), `172.17.0.2/30` (veth)
- B: `172.17.0.5/30` (bridge), `172.17.0.6/30` (veth)
- WG: A=`10.77.77.1/30`, B=`10.77.77.2/30`

---

## Passo 1 — Aplicar scripts base
No Router A, aplicar:
- `skills/ncx-mikrotik-tailscale-wg-eoip/scripts/router-a-setup.rsc`

No Router B, aplicar:
- `skills/ncx-mikrotik-tailscale-wg-eoip/scripts/router-b-setup.rsc`

> Substituir placeholders:
> - `{{REMOTE_WG_PUBKEY}}`
> - `{{REMOTE_TS_IP}}`

---

## Passo 2 — Subir Tailscale com AuthKey (recomendado)
> Use uma AuthKey **reusable + preauthorized + non-ephemeral**.

No Router A:
```rsc
/container set [find where name="tailscale:latest"] env="TS_AUTHKEY=<TSKEY_AQUI>;TS_STATE_DIR=/var/lib/tailscale;TS_USERSPACE=true;TS_EXTRA_ARGS=--accept-routes=true --advertise-routes=172.17.0.0/30"
/container stop [find where name="tailscale:latest"]
:delay 2s
/container start [find where name="tailscale:latest"]
```

No Router B:
```rsc
/container set [find where name="tailscale:latest"] env="TS_AUTHKEY=<TSKEY_AQUI>;TS_STATE_DIR=/var/lib/tailscale;TS_USERSPACE=true;TS_EXTRA_ARGS=--accept-routes=true --advertise-routes=172.17.0.4/30"
/container stop [find where name="tailscale:latest"]
:delay 2s
/container start [find where name="tailscale:latest"]
```

No Tailscale Admin, aprovar rotas:
- `172.17.0.0/30`
- `172.17.0.4/30`

## Passo 2B — Autenticar via link (fallback quando key falhar por policy)
Em cada roteador:
```rsc
/log print where message~"AuthURL"
```
Abrir link `https://login.tailscale.com/a/...`.

---

## Passo 3 — Ajustar endpoint WG para IP Tailscale atual
Como o IP 100.x pode mudar após reboot/reauth:

1. Capturar IP atual da ponta remota:
```rsc
/log print where message~"peerapi: serving on http://100"
```

2. Atualizar peer WG:
```rsc
/interface wireguard peers set [find where interface=wg-lab] endpoint-address=<IP_100_REMOTO> endpoint-port=<PORTA_REMOTA>
```

---

## Passo 4 — Validar túnel
### L3
- Router A:
```rsc
ping 10.77.77.2 count=4
```
- Router B:
```rsc
ping 10.77.77.1 count=4
```

### L2
Conectar 1 dispositivo em cada `ether3` e validar broadcast/DHCP/ping entre pontas.

---

## Persistência e estabilidade
- `start-on-boot=yes` no container
- Em alguns cenários/policies, o container pode voltar pedindo auth URL
- Preferir AuthKey reutilizável + preauthorized (quando policy permitir)
- Manter script de diagnóstico para recuperação rápida

Arquivo de diagnóstico:
- `skills/ncx-mikrotik-tailscale-wg-eoip/scripts/check-status.rsc`

Rollback rápido:
- `skills/ncx-mikrotik-tailscale-wg-eoip/scripts/rollback-disable-l2.rsc`

---

## Troubleshooting rápido
1. **Container parado**
```rsc
/container start [find where name="tailscale:latest"]
```
2. **WG sem RX/TX**: endpoint TS remoto desatualizado
3. **EoIP inactive**: WG sem handshake
4. **AuthURL toda hora**: sessão não persistida/policy exige novo auth
5. **invalid key: unable to validate API key**: chave inválida/revogada ou bloqueada por policy do tailnet
