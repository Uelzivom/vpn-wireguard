# VPN própria com WireGuard (wg-easy) na VPS Oracle

Guia passo a passo para instalar sua própria VPN com WireGuard na VPS gratuita da Oracle, usando o wg-easy: o servidor WireGuard e um painel web para criar clientes, tudo em um container Docker.

No final você terá uma VPN só sua: conecta o celular ou o computador, e o tráfego sai pelo IP da VPS.

> **Como foi feito aqui:** este guia documenta a instalação real, **wg-easy v13**, conferida na VPS em 24/11/2026 (imagem `ghcr.io/wg-easy/wg-easy:13`, saudável). A versão corrente do projeto é a **v15** (v15.5.0, de 24/10/2026), que mudou o setup: na v15 as variáveis de ambiente abaixo não existem mais e tudo se configura no assistente da primeira execução, no painel. Se for instalar do zero hoje, leia a nota sobre a v15 no final — mas não misture os dois jeitos.
>
> Pré-requisito: a VPS do [projeto 1](../vps-oracle-gratis) (ou qualquer Linux com Docker e IP público), com o Docker e o plugin Compose instalados.

---

## O que você vai ter no final

| Item | Valor (instalação real) |
|---|---|
| Servidor WireGuard | dentro do container `wg-easy` (interface `wg0`) |
| Rede do túnel | `10.8.0.0/24` — servidor `10.8.0.1`, clientes `10.8.0.2` em diante |
| Endereço do servidor | `uelvpnoracle.duckdns.org` (hostname DuckDNS apontando para a VPS) |
| DNS dos clientes | `94.140.14.14, 94.140.15.15` (AdGuard DNS — bloqueia anúncios no túnel) |
| Porta do túnel | `51820/udp` |
| Painel web | `51821/tcp` (login com a senha definida no compose) |
| Dados (clientes e chaves) | pasta do projeto, montada em `/etc/wireguard` |
| Rede Docker | `wg-easy-oracle_default` (`172.19.0.0/16`, container em `172.19.0.2`) |

---

## Passo 1 — Abrir as portas na Oracle (security list)

O WireGuard precisa de duas portas liberadas na lista de segurança da VCN da instância:

| Porta | Protocolo | Origem | Para quê |
|---|---|---|---|
| 51820 | UDP | `0.0.0.0/0` | O túnel WireGuard |
| 51821 | TCP | `0.0.0.0/0` | O painel web do wg-easy |

No console da OCI: **Networking → Virtual Cloud Networks → sua VCN → Security Lists → sua security list → Add Ingress Rules**. (O SSH, porta 22, já vem liberado do projeto 1.)

> Segurança: o painel na 51821 fica exposto à internet — use senha forte. Se preferir, libere a 51821 só para o seu IP em vez de `0.0.0.0/0`.

---

## Passo 2 — Criar um hostname (DuckDNS)

O IP público da VPS pode mudar. Em vez de usar o IP cru no `WG_HOST`, crie um hostname gratuito no [DuckDNS](https://www.duckdns.org/) apontando para o IP da VPS — aqui é o `uelvpnoracle.duckdns.org`. Os clientes passam a se conectar pelo nome, e se o IP mudar basta atualizar o DuckDNS.

---

## Passo 3 — Instalar o wg-easy com Docker Compose

Na VPS, crie a pasta do projeto e o `docker-compose.yml` (este é o arquivo real da instalação, com a senha trocada por um placeholder):

```bash
mkdir -p ~/wg-easy-oracle
cd ~/wg-easy-oracle
```

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:13
    container_name: wg-easy
    environment:
      - WG_HOST=uelvpnoracle.duckdns.org
      - PASSWORD=<sua-senha-forte>
      - WG_PORT=51820
      - WG_DEFAULT_DNS=94.140.14.14,94.140.15.15
    volumes:
      - .:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
```

Pontos importantes:

- **Fixe a tag `:13`** (a versão instalada). A tag `:latest` aponta para a v14 — um compose com `:latest` instala outra versão, com outro jeito de configurar.
- **`WG_HOST`** é o endereço que vai dentro da configuração dos clientes: o hostname DuckDNS (ou o IP público, se preferir).
- **`PASSWORD`** é a senha do painel web. Guarde-a — não há `PASSWORD_HASH` nesta versão.
- **`WG_DEFAULT_DNS`** define o DNS dos clientes no túnel — aqui o AdGuard, que bloqueia anúncios e rastreadores para quem está na VPN.
- **O volume `.:/etc/wireguard`** monta a própria pasta do projeto como o diretório de dados: `wg0.conf`, chaves e clientes vivem ali. É isso que o backup precisa copiar.
- **`NET_ADMIN` e `SYS_MODULE` + os `sysctls`** são obrigatórios: sem eles o túnel não roteia o tráfego.

Suba o container:

```bash
docker compose up -d
docker compose logs -f   # Ctrl+C para sair; confirme que subiu sem erro
```

---

## Passo 4 — Criar o primeiro cliente e conectar

1. Abra o painel no navegador: `http://uelvpnoracle.duckdns.org:51821` (ou `http://IP-DA-VPS:51821`) e entre com a senha do `PASSWORD`.
2. Crie um cliente (ex.: `celular`).
3. Escaneie o **QR code** com o app WireGuard no celular, ou baixe o arquivo `.conf` e importe no app do computador.
4. Ative o túnel no app e confirme que a internet continua funcionando.
5. Confira o IP público do aparelho (ex.: buscando "meu ip"): ele deve ser o **IP da VPS**, não o da sua rede.

No painel dá para ver cada cliente, o tráfego e o último handshake — um cliente sem handshake nunca conectou.

---

## Backup (faça antes de qualquer atualização)

Tudo o que importa vive na pasta do projeto (`~/wg-easy-oracle`), que é o volume `/etc/wireguard`. Antes de atualizar ou mexer no container:

```bash
cd ~
tar -czf wg-easy-backup-AAAAMMDD.tar.gz wg-easy-oracle
```

Guarde esse backup fora da VPS. Sem ele, uma reinstalação gera chaves novas e todos os clientes precisam ser recriados.

## Atualização

Para atualizar dentro da mesma versão principal (ex.: 13.x → 13.y):

```bash
cd ~/wg-easy-oracle
docker compose pull
docker compose up -d
```

**Entre versões principais, leia as notas de lançamento antes.** A v15 foi uma reescrita completa, sem caminho de migração da v14: sair da v14 exige instalar a v15 do zero e importar os clientes. O caminho a partir da v13 é v13 → v14 primeiro, com backup e validação dos aparelhos, e só depois pensar na v15.

## Sobre a v15 (a versão atual)

Na v15, `WG_HOST`, `PASSWORD` e quase todas as variáveis de ambiente **não existem mais**: a configuração acontece no assistente da primeira execução, no painel web, e a imagem é `ghcr.io/wg-easy/wg-easy:15`. A v15 também traz 2FA, expiração de clientes e links de uso único. Quem instala do zero hoje deve seguir a documentação oficial da v15 — este guia documenta a v13, que é a instalação real aqui.

## Solução de problemas

- **Cliente conecta mas não navega:** confira os `sysctls` de encaminhamento (`net.ipv4.ip_forward=1`) e se a porta UDP 51820 está liberada na security list da Oracle **e** no firewall do host.
- **Painel não abre:** confira se a porta TCP 51821 está liberada e se o container está `Up` (`docker ps`).
- **Primeira tentativa de SSH cai com `Connection closed`:** comportamento conhecido do Cloud Shell — tente de novo, a segunda entra.
- **"Out of capacity" não se aplica aqui** — isso é da criação da instância (projeto 1), não da VPN.

---

## Próximos passos

- **Projeto 3:** instalar o agente de IA (Telegram) dentro da VPS — o painel do agente também pode ficar acessível só pela VPN.

---

## Confirmado na VPS em 24/11/2026

- [x] Imagem `ghcr.io/wg-easy/wg-easy:13`, container saudável, portas `51820/udp` e `51821/tcp`.
- [x] `docker-compose.yml` real em `~/wg-easy-oracle` (conteúdo acima, senha omitida).
- [x] `WG_HOST=uelvpnoracle.duckdns.org`, `WG_DEFAULT_DNS` AdGuard, volume `.:/etc/wireguard`.
- [x] Rede `wg-easy-oracle_default` (`172.19.0.0/16`, container em `172.19.0.2`).

## Licença

MIT — veja `LICENSE`.
