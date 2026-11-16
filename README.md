# meuRadarr

Stack de automação de mídia rodando em containers separados, cada um com seu
próprio `docker-compose`, todos compartilhando a rede `arr-network`.

## Rede compartilhada

Os serviços se comunicam entre si pelo nome do container dentro da rede
`arr-network`. Ela precisa existir antes de subir qualquer serviço:

```
docker network create arr-network
```

(rode só uma vez; se já existir, o comando falha silenciosamente sem
problema)

## Radarr

Gerencia a biblioteca de filmes e dispara downloads.

- **Arquivo**: `docker-compose.radarr.yml`
- **Porta**: 7878 → http://localhost:7878
- **Config**: `/home/diego/radarr/config`
- **Bibliotecas**: `/mnt/storage/diego/radarr/{movies,series,anime,downloads}`

```
docker compose -f docker-compose.radarr.yml up -d
docker compose -f docker-compose.radarr.yml down
```

## Sonarr

Gerencia a biblioteca de séries e animes e dispara downloads.

- **Arquivo**: `docker-compose.sonarr.yml`
- **Porta**: 8989 → http://localhost:8989
- **Config**: `/home/diego/sonarr/config`
- **Bibliotecas**: `/mnt/storage/diego/radarr/{series,anime,downloads}` (as mesmas do Radarr)

```
mkdir -p /home/diego/sonarr/config
docker compose -f docker-compose.sonarr.yml up -d
docker compose -f docker-compose.sonarr.yml down
```

## qBittorrent

Cliente de torrent usado pelo Radarr/Prowlarr para baixar os arquivos.

- **Arquivo**: `docker-compose.qbittorrent.yml`
- **WebUI**: 8080 → http://localhost:8080
- **Porta de torrent**: 6881 (TCP/UDP)
- **Config**: `/home/diego/qbittorrent/config`
- **Downloads**: `/mnt/storage/diego/radarr/downloads` (mesma pasta do Radarr)

```
docker compose -f docker-compose.qbittorrent.yml up -d
docker compose -f docker-compose.qbittorrent.yml down
```

## Prowlarr

Indexador central que alimenta o Radarr com fontes de busca.

- **Arquivo**: `docker-compose.Prowlarr.yml`
- **Porta**: 9696 → http://localhost:9696
- **Config**: `/home/diego/prowlarr/config`

```
mkdir -p /home/diego/prowlarr/config
docker compose -f docker-compose.Prowlarr.yml up -d
docker compose -f docker-compose.Prowlarr.yml down
```

## FlareSolverr

Proxy que resolve desafios de Cloudflare para indexadores que o Prowlarr não
consegue acessar direto.

- **Arquivo**: `docker-compose.flaresolverr.yml`
- **Porta**: 8191 → http://localhost:8191
- Sem volume de config (estado em memória)

```
docker compose -f docker-compose.flaresolverr.yml up -d
docker compose -f docker-compose.flaresolverr.yml down
```

No Prowlarr, configurar o FlareSolverr como proxy usando a URL interna
`http://flaresolverr:8191` (nome do container, via `arr-network`).

## Transferir um filme do servidor para o computador pessoal via Tailscale

Com os dois dispositivos na mesma tailnet (`tailscale status` pra conferir),
dá pra mandar o arquivo direto sem precisar de SMB/FTP.

**No servidor** (onde o filme está, ex: pasta do Radarr):

```
tailscale file cp "/mnt/storage/diego/radarr/movies/Filme (2024)/Filme.mkv" nome-do-computador-pessoal:
```

Use o hostname do seu computador pessoal na tailnet — aparece em
`tailscale status` ou no admin console (`https://login.tailscale.com/admin/machines`).

**No computador pessoal**, para receber o arquivo pendente:

```
tailscale file get ~/Downloads/
```

O arquivo chega direto na pasta indicada, sem passar por servidor
intermediário — a transferência é ponto a ponto pela tailnet.
