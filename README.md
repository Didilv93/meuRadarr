# meuRadarr

Stack de automação e reprodução de mídia rodando em containers separados, cada um com seu
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

## Jellyfin

Servidor de mídia: catálogo de filmes, séries e animes, reprodução no
navegador, celular e TV (converte na hora o que o aparelho não toca),
legendas e controle por usuário.

- **Arquivo**: `docker-compose.jellyfin.yml`
- **Porta**: 8096 → http://localhost:8096
- **Config**: `/home/diego/jellyfin/config`
- **Bibliotecas**: `/movies`, `/series` e `/anime` dentro do container (as
  mesmas pastas do Radarr/Sonarr). Montadas com escrita para o Jellyfin poder
  apagar itens e gravar legendas ao lado do vídeo.

```
mkdir -p /home/diego/jellyfin/config
docker compose -f docker-compose.jellyfin.yml up -d
docker compose -f docker-compose.jellyfin.yml down
```

Na primeira abertura, o assistente cria o usuário administrador. Depois:

- **Bibliotecas**: Painel → Bibliotecas → adicionar "Filmes" (`/movies`),
  "Séries" (`/series`) e "Animes" (`/anime`, tipo Séries).
- **Legendas**: Painel → Plugins → Catálogo → "Open Subtitles"; depois, em
  cada biblioteca, marcar os idiomas de legenda a baixar.
- **Remover**: o usuário só apaga se tiver "Permitir exclusão de mídia" no
  perfil dele (Painel → Usuários).
- **Aceleração de hardware** (opcional): se o servidor tiver GPU Intel/AMD,
  adicionar `devices: [/dev/dri:/dev/dri]` ao serviço e ativar VAAPI/QSV em
  Painel → Reprodução → Transcodificação.

## Seerr

Tela de pedidos: busca filmes e séries e repassa o pedido ao Radarr/Sonarr.
É o sucessor do Jellyseerr e do Overseerr (os dois projetos se fundiram em
2026). Entra com as contas do Jellyfin.

- **Arquivo**: `docker-compose.seerr.yml`
- **Porta**: 5055 → http://localhost:5055
- **Config**: `/home/diego/seerr/config`. O container roda com o UID 1000,
  então a pasta precisa ser dele.

```
mkdir -p /home/diego/seerr/config
sudo chown -R 1000:1000 /home/diego/seerr/config
docker compose -f docker-compose.seerr.yml up -d
docker compose -f docker-compose.seerr.yml down
```

No assistente inicial, usar as URLs internas da `arr-network`:

- Jellyfin: `http://jellyfin:8096`
- Radarr: `http://radarr:7878`, pasta raiz `/movies`
- Sonarr: `http://sonarr:8989`, pastas raiz `/series` e `/anime`

As chaves de API ficam em Configurações → Geral de cada serviço.

### Login pelo trem (Authentik)

O botão **trem** na tela de login usa o OIDC do Seerr, que por enquanto só
existe na imagem de teste `seerr/seerr:preview-new-oidc`. A imagem fica presa
ao digest testado: o tag é atualizado pelos desenvolvedores. Quando o OIDC
chegar à versão estável, voltar para `ghcr.io/seerr-team/seerr:latest`; a
configuração e os vínculos continuam valendo.

O cliente OIDC `seerr` e o grupo `hub:seerr` ficam no trem.AUTH. No
`settings.json`, dentro de `main`, `"oidcLogin": true`, e:

```
"oidc": { "providers": [ { "slug": "authentik", "name": "trem",
  "issuerUrl": "https://auth.didilv93.com/application/o/seerr/",
  "clientId": "seerr", "clientSecret": "<SEERR_OIDC_CLIENT_SECRET>",
  "newUserLogin": true } ] }
```

`main.applicationUrl` precisa estar vazio ou ser `http://100.116.16.24:5055`:
o endereço de retorno é montado a partir dele.

Conta que já existe no Seerr (vinda do Jellyfin) precisa ser **vinculada antes**:
entrar com o Jellyfin, Settings → Linked Accounts → Link Account → trem. Sem o
vínculo, o login pelo trem é recusado quando o e-mail já existe; não cria conta
duplicada. Contas novas entram com as permissões padrão; o admin continua sendo
definido no próprio Seerr.

## Bazarr

Baixa legendas automaticamente para o que entra no Radarr e no Sonarr.

- **Arquivo**: `docker-compose.bazarr.yml`
- **Porta**: 6767 → http://localhost:6767
- **Config**: `/home/diego/bazarr/config`
- **Bibliotecas**: `/movies`, `/series` e `/anime`, nos mesmos caminhos que o
  Radarr e o Sonarr veem; por isso não precisa de mapeamento de caminhos.

```
mkdir -p /home/diego/bazarr/config
docker compose -f docker-compose.bazarr.yml up -d
docker compose -f docker-compose.bazarr.yml down
```

Em Configurações: ligar o Radarr (`http://radarr:7878`) e o Sonarr
(`http://sonarr:8989`) com as chaves de API, criar um perfil de idiomas (ex.:
Português do Brasil e Inglês) e adicionar os provedores de legenda (ex.:
OpenSubtitles.com, que pede uma conta gratuita).

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
