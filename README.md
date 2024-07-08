# *Home Server*

Esse repositório servirá para auxiliar a configuração do servidor doméstico, para montar esse guia foi utilizada as referências indicadas no final desse documento.

## *Hardware* Recomendado

Para essa *home server* os *hardwares* escolhidos foram:

<details>
    <summary>Intel NUC NUC7PJYH</summary>

    - Processador Intel Pentium J5040
        - N° de Núcleos de CPU: 4
        - N° de Threads de CPU: 4
        - Clock : 2.0Ghz à 3.2Ghz
        - TDP: 10W
    - Placa Mãe Intel NUC7JYB
        - Memória: 2 x DDR4 SO-DIMM - Suporta até 8 GB
        - Rede: Realtek 8111H-CG / Intel® Wireless-AC 9462
    - Vídeo Integrado
        - Modelo: UHD Intel® 605
        - Frequência: 800 Mhz
        - Memória: Max. 8GB

</details>

<details>
    <summary>SSD QVO 870 SATA III 2.5"</summary>

    - Capacidade: 4TB
    - Leitura sequencial: até 560 MB/s
    - Gravação sequencial: até 530 MB/s
    - Conectividade: SATA 6 Gb/s

</details>

<details>
    <summary>SSD Portátil Samsung T7</summary>

    - Capacidade: 1TB
    - Leitura sequencial: até 1.050 MB/seg
    - Gravação sequencial: até 1.000 MB/seg
    - Conectividade: USB 3.2 Gen.2 (10Gbps)

</details>

## Configuração Básica
Será utilizado o Ubuntu Server 24.04 LTS com o Docker e Docker Compose para disponibilizar os serviços necessários, para preparar o servidor será necessário executar:

```bash
sudo apt update
sudo apt upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo apt install docker-compose-plugin
```

Também será necessário baixar o repositório atual e acessar a pasta do [docker-compose](/docker-compose):

```bash
git clone git@github.com:calimanfilho/homeServer.git
cd homeServer/docker-compose
```

E finalmente colocar os *containers* em execução (nem todos são necessários):

```bash
docker compose -f docker-compose-media.yml up -d
docker compose -f docker-compose-network.yml up -d
docker compose -f docker-compose-transcoder.yml up -d
docker compose -f docker-compose-utils.yml up -d
```

Ao decorrer desse guia será utilizado o nome do serviço como *hostname*, isso funciona porque cada docker-compose cria sua própria *network* e ficam na mesma VLAN, e por padrão o Docker registra como *hostname* o nome do serviço configurado no arquivo do Docker Compose.

## Utilitários

[Portainer](https://www.portainer.io/) é um gerenciador visual dos recursos gerenciados pelo serviço do Docker localmente, ao acessar o Portainer pode ser feito um pequeno ajuste para os hiperlinks dos *containers* funcionarem corretamente, ao acessar as configurações do ambiente e alterar o `Public IP` de `0.0.0.0` para `localhost` ou o IP do Home Server.

![Portainer Public IP](/images/portainer-public-ip.gif)

Para criar um painel simples com o *link* de todos os serviços, será utilizado o [Homarr](https://homarr.dev/), nos *tiles* devem ser informado o `App name` com o nome da aplicação; em `Internal address` o `http://<LOCAL_IP>:<LOCAL_PORT>` para que seja feito o *status checker* e por fim o `External address` onde é informado a URL pra alcançar a aplicação, poderia ser o `http://<LOCAL_IP>:<LOCAL_PORT>`, mas para ficar mais fácil o acesso, foi configurado no roteador o DNS `homeserver.local` apontando para o IP interno do *Home Server*, por conta disso esse campo é preenchido no formato `http://homeserver.local:<LOCAL_PORT>`.

Em utilitários também tem o [Watchtower](https://containrrr.dev/watchtower/), que monitora atualização das imagens de todos os *containers*, quando uma nova versão da imagem for publicada, ela será baixada e de forma automática a aplicação serão reiniciada com a nova imagem.

E tem o [ZeroTier](https://www.zerotier.com/) que, assim como [TailScale](https://tailscale.com/), oferece uma boa alternativa de serviço fácil de VPN, com baixa configuração, pra facilitar acessar aos serviços locais fora de casa. Para utilizar o ZeroTier é preciso fazer uma conta no [site oficial](https://www.zerotier.com/) e criar uma rede para obter o *Network ID*, com isso basta iniciar o *container* e executar o comando `docker exec zerotier zerotier-cli join <NETWORK_ID>`, seguido da autorização do dispositivo pela *interface web* para que seja obtido o endereço IP que será usado para o acesso remoto, bastando que o outro dispositivo também seja adicionado como membro na *interface web*.

![ZeroTier](/images/zerotier-members.gif)

Caso seja necessário recodificar ou "re-encodar" os filmes para um formato melhor (ou menor) pode ser utilizado o [HandBrake](https://handbrake.fr/). Para esse programa, a melhor opção é ter uma placa de vídeo NVIDIA, com suporte a *encoding* de video via *hardware* NVENC, que permite acelerar o processo em múltiplas vezes. As configurações que serão utilizadas são do perfil pré-configurado `Official > Matroska > H.265 MKV 2160p60 4K` com alteração no `Constant Quality` para `15 CQ` e do `Video Encoder` para `H.265 10-bit (NVEnc)`, assim, o *encoding* será rápido, com uma boa compressão e sem grande perda de qualidade.

> Para *container* no Windows, utilizando WSL, o `Video Encoder` pode não ter a opção `H.265 10-bit (NVEnc)`. Na teoria só deveria ser preciso realizar o mapeamento do dispositivo `/dev/dri` no *container*, mas somente isso não funcionou. Lembrando que o *encoding* via *software* é mais lento mas terá qualidade superior ao *encoding* via *hardware*, em comparação com o NVENC, que consegue manter uma excelente qualidade com uma boa velocidade, ou Intel QSV, que tem uma qualidade de video inferior mas também é rápido.

![HandBrake Config](/images/handbrake-config.gif)

## Network

[Pi-Hole](https://pi-hole.net/) é um servidor de DNS que tem funcionalidade de bloquear o acesso a anúncios, rastreadores e *malware*. Nessa *stack* o Pi-Hole é utilizado em conjunto com o [CloudFlared](https://github.com/cloudflare/cloudflared) para que as consultas e respostas DNS sejam criptografadas com o DoH (DNS-Over-HTTPS), isso significa que a conexão do dispositivo ao servidor DNS é segura e não pode ser facilmente monitorada, adulterada ou bloqueada.

> No Pi-Hole ao utilizar o `network_mode` como `host`, o *container* não inicia e fica apresentando os erros `Stopping pihole-FTL` e `pihole-FTL: no process found`, possivelmente ocasionado por conflito de portas no WSL, como ele não será utilizado no *desktop* principal, esse problema não será investigado ou corrigido.

## Media

Antes de iniciar, é preciso criar os diretórios que os *container* irão armazenar os dados no *host*, a configuração do caminho padrão que incentiva a usar montagens como `/movies`, `/tv`, `/books` ou `/downloads` é muito abaixo do ideal e faz com que pareçam dois ou três sistemas de arquivos, mesmo que não sejam (porque de como funcionam os volumes do Docker), resultando em uma cópia e exclusão mais lenta e com maior intensidade de E/S. Para solucionar esse problema a estrutura de diretórios desenhada foi a seguinte:

```bash
data
├── torrents
│   ├── books
│   ├── movies
│   ├── music
│   └── tv
├── usenet
│   ├── incomplete
│   └── complete
│       ├── books
│       ├── movies
│       ├── music
│       └── tv
└── media
    ├── books
    ├── movies
    ├── music
    └── tv
```

Para iniciar a *stack* de Media, será utilizado o [qBittorrent](https://www.qbittorrent.org/) como aplicativo cliente P2P, provavelmente o melhor e mais completo aplicativo pra baixar torrent, para o primeira acesso será necessário obter a senha temporária do usuário *admin* nos logs do *container*.

As primeiras alterações nas configurações do qBittorrent será em `Tools` > `Options` > `Downloads`:

* Marcar `Delete .torrent files afterwards`;
* Marcar `Pre-allocate disk space for all files`;
* Selecionar em `Default Torrent Management Mode`: `Automatic`;
* Selecionar em `When Default Save Path changed`: `Realocate affected torrents`;
* Selecionar em `When Category Save Path changed`: `Realocate affected torrents`;
* Escrever em `Default Save Path`: `/data/torrents`;
* Salvar.

> 1. O local de *download* é definido no cliente de download.
> 2. O cliente de *download* somente faz download para a pasta/local de download.
> 3. Os Starr Apps importam do local de download (copiar/mover/*hardlink*) para a pasta/biblioteca de mídia.
> 4. O Plex SÓ deve ter acesso à pasta/biblioteca de mídia.
> 5. O *download* e biblioteca de mídia **NUNCA** devem ser o mesmo local.

![qBittorrent Downloads](/images/qbittorrent-downloads.gif)

As próximas alterações ficam em `Tools` > `Options` > `Connection`:

* Selecionar em `Peer connection protocol`: `TCP`;
* Desmarcar `Use UPnP / NAT-PMP port forwarding from my router`;
* Salvar.

![qBittorrent Connection](/images/qbittorrent-connection.gif)

Essa alteração não precisa ser feita, mas apenas para informar que essa configuração que permite deletar os download concluídos do qBittorrent e mover as mídias para as pastas corretas, caso o torrent continue semeando o Radarr ou Sonarr criará *hardlinks* dos arquivos sem nenhum prejuízo na configuração, para continuar basta acessar `Tools` > `Options` > `BitTorrent`:

* Marcar `When ratio reaches`;
* Escrever em `When ratio reaches`: `0`;
* Salvar.

![qBittorrent BitTorrent](/images/qbittorrent-bittorrent.gif)

As próximas mudanças são em `Tools` > `Options` > `Web UI`:

* Escrever em `Password`: `<ANY_PASSWORD>`;
* Marcar `Bypass authentication for clients on localhost`;
* Marcar `Bypass authentication for clients in whitelisted IP subnets` e adicionar a rede interna de onde ficará o servidor;
* Salvar.

![qBittorrent Web UI](/images/qbittorrent-web-ui.gif)

No menu lateral em `CATEGORIES` > `Add category` na janela que aparecer deverá ser adicionado em `Category` e `Save Path` as categorias `books`, `movies`, `music` e `tv`. Não precisa adicionar o caminho completo em `Save Path`, pois será usado o caminho de *download* raiz configurado em `Tools` > `Options` > `Downloads` > `Default Save Path`.

![qBittorrent Categories](/images/qbittorrent-categories.gif)

O qBitTorrent tem suporte a *plugins* para pesquisas manuais, permitindo adicionar mais opções de sites de torrents, para isso será utilizado o [Jackett](https://github.com/Jackett/Jackett), que funciona como servidor *proxy* para pesquisar em diversos indexadores. Para que o Jackett funcione no qBitTorrent é necessário passar a chave de API em um arquivo de configuração, mostrado [nesse procedimento](https://github.com/qbittorrent/search-plugins/wiki/How-to-configure-Jackett-plugin), logo abaixo está a configuração utilizada nesse projeto. Para aparecer a pasta `nova3` é necessário acessar o qBittorrent, ir em `Search` > `Search plugins` > `Check for Updates`. Será necessário trocar o campo `<JACKETT_API_KEY>` pela API do Jackett que pode ser encontrada na página inicial da ferramenta.

```json
$ cat ~/qbittorrent/config/qBittorrent/data/nova3/engines/jackett.json
{
    "api_key": "<JACKETT_API_KEY>",
    "thread_count": 20,
    "tracker_first": false,
    "url": "http://jackett:9117"    
}
```

> No Docker Compose do Jackett foi retirado o *volume* `/path/to/blackhole:/downloads`, pois o *blackhole* é um diretório que o Jackett utiliza para colocar os arquivos `.torrent` para que os clientes torrent coloque automaticamente esse arquivo na fila de *downloads*, mas como o qBitTorrent pode utilizar a API do Jackett isso não é necessário.

Para configurar o Jackett com os indexadores públicos desejados, basta clicar em `Add indexer` e clicar no `+` ao lado do nome dos indexadores, mas para os indexadores privados é necessário um pouco mais de configuração como cadastro no site para passar usuário e senha para o Jackett.

![Jackett](/images/jackett.png)

Em conjunto com o Jackett, também é utilizado o [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr), que é um servidor proxy para contornar a proteção Cloudflare e DDoS-GUARD, ele pode ser configurado em Jacket Configuration no campo `FlareSolverr API URL` com a URL `http://flaresolverr:8191/`, agora toda vez que um site de torrent verificar se quem está acessando é humano ou robô, o FlareSolver utilizará o Selenium para resolver o desafio.

Para não precisar ficar procurando os filmes de forma manual, será utilizado o [Radarr](https://radarr.video/), ele permite baixar filmes torrent e usenet de forma automática, os filmes serão constantemente procurados para possuir o melhor release possível.

Após acessar o Radarr pela primeira vez será solicitado algumas configurações de autenticação, pode ser colocadas as seguintes:

* Selecionar em `Authentication Method`: `Forms (Login Page)`;
* Selecionar em `Authentication Required`: `Disabled for Local Addresses`;
* Escrever em `Username`: `admin`;
* Escrever em `Password` e `Password Confirmation`: `<ANY_PASSWORD>`;
* Salvar.

![Radarr Authentication](/images/radarr-authentication.png)

> Todas as configurações do Radaar que serão feitas pela *interface web* é necessário que as configurações avançadas estejam aparecendo, caso não estejam, basta clicar em `Show Advanced` na barra superior.

Nessa etapa será configurado o gerenciamento de mídia em `Settings` > `Media Management`, não será preciso fazer alterações nos nomes dos arquivos e pastas porque outra aplicação irá fazer os ajustes futuramente.

* Marcar `Delete empty folders`;
* Selecionar em `Propers and Repacks`: `Do Not Prefer`;
* Selecionar em `Add Root Folder` : `/data/media/movies`;
* Salvar.

![Radaar Media Management](/images/radaar-media-management.gif)

Agora será configurado o cliente de *download* torrent em `Settings` > `Download Clients` > `qBittorrent`:

* Escrever em `Host`: `qbittorrent`;
* Escrever em `Username`: `admin`;
* Escrever em `Pasword`: `<QBITTORRENT_PASSWORD>`;
* Escrever em `Category`: `movies`;
* Salvar.

![Radaar qBittorrent Config](/images/radaar-qbittorrent-config.gif)

Outra opção interessante é o cliente de *download* Usenet, como o [SABnzbd](https://sabnzbd.org/), talvez o melhor cliente Usenet atual, ele é um *binary newsreader*, em outras palavras, serve para baixar arquivos binários de servidores Usenet, esses servidores também são conhecidos como *newserver*, eles armazenam os arquivos nos *newsgroup*. Uma forma simples de encontrar os arquivos binários nos *newsgroup* é utilizando os indexadores Usenet, como o [NZBKing](https://www.nzbking.com/) ou [NZBIndex](https://www.nzbindex.com/), os indexadores percorrem os *newsgroups* para encontrar todas as partes do arquivo e cria um arquivo NZB que aponta para essas partes e por fim disponibiliza o NZB para que os usuários utilizem e façam o download desse conteúdo. Existem indexadores gratuitos e pagos, diferente dos provedores Usenet, não há provedores gratuitos, eles são essencial para utilizar Usenet porque possibilitam acessar aos *newserver* que os arquivos NZB estão apontando, um dos provedores mais famosos é o [Newhosting](https://www.newshosting.com/), mas como os planos de provedores Usenet são pagos essa opção não será utilizada.

![SABnzbd Server Details](/images/sabnzbd-server-details.png)

As próximas configurações do Radarr são bastante demoradas para serem feitas de forma manual (demonstração dessas configurações no tópico Extra no fim do guia), por conta disso não serão feitas pela *interface web*, para isso será utilizado os scripts do aplicativo [Recyclarr](https://recyclarr.dev/wiki/) que sincroniza as configurações do [TRaSH Guides](https://trash-guides.info/) com a aplicação local, todas as instruções são passadas pelos YAMLs na pasta `/config/configs` do `container`.

Os arquivos iniciais do Recyclarr foram criados pelo comando `docker exec recyclarr recyclarr config create -t uhd-bluray-web -t hd-bluray-web`, esse mesmo comando deverá ser utilizado novamente para a criação da estrutura inicial do Recyclarr, os templates `uhd-bluray-web` e `hd-bluray-web` possuem as configurações desejadas para o Radarr mas ainda é necessário fazer junção dessas duas configurações porque o Recyclarr não permite dois *templates* apontando para a mesma URL do servidor, o arquivo final pode ser encontrado no diretório `/config/recyclarr` desse repositório, para que ele seja utilizado os arquivos `uhd-bluray-web.yml` e `hd-bluray-web.yml` deverão ser excluídos do *container* e o arquivo `radarr.yml` deverá ser movido para o diretório `~/recyclarr/config` do *Home Server*.

> Ao utilizar o *template* de configuração do Recyclarr, ele também referência outros templates, como o de `quality-definition`, `quality-profile` e `custom-formats`, para ver todas os *templates* disponíveis pode ser utilizado o comando `docker exec recyclarr recyclarr config list templates --includes`, os YAMLs de configurações podem ser encontrados em [/recyclarr/config-templates](https://github.com/recyclarr/config-templates).

Para utilizar o script será preciso realizar a alteração no campo `api_key` do arquivo `radarr.yml` do diretório `~/recyclarr/config/configs`, a chave pode ser obtida no Radarr em  `Settings` > `General` > `API Key`. Após tudo pronto, para iniciar a sincronização das configurações deverá ser executado o comando:

```bash
docker exec recyclarr recyclarr sync
```

As configurações que serão alteradas pelo script são as seguintes:

1. Ajuste nos `Movie Naming` em `Settings` > `Media Management`. 

    > Se, por qualquer motivo, precisar fazer uma reinstalação ou uma reimportação completa no Starr Apps ou Plex, é bom ter todas essas informações no nome do arquivo para que ele seja importado corretamente e não seja incorretamente correspondido como HDTV ou WEB-DL etc.

2. Ajuste nas `Quality Definitions` em `Settings` > `Quality`.

    > As definições de qualidade estipula limites em megabytes por minuto para cada qualidade.

3. Inclusão de `Custom Formats` em `Settings` > `Custom Formats`. 

    > Os formatos personalizados são adicionados para que os perfis possam utilizar eles para classificar as mídias.

4. Inclusão de `Quality Profiles` em `Settings` > `Profiles`.

    > Os perfis são necessários para informar os formatos de arquivo que são desejados e os que não são desejados, estipulando uma nota para cada e assim classificar quais arquivos permanecerão os mesmo, receberão *upgrade* ou que nem serão baixados.

Caso tudo dê certo, os logs que aplicação retornará serão parecidos com esse:

```bash
===========================================
Processing Radarr Server: [movies]
===========================================

[INF] Created 56 New Custom Formats
[INF] Total of 56 custom formats were synced
[INF] Created 2 Profiles: ["HD Bluray + WEB","UHD Bluray + WEB"]
[INF] A total of 2 profiles were synced. 2 contain quality changes and 2 contain updated scores
[INF] Processing Quality Definition: movie
[INF] Number of updated qualities: 14
[INF] Media naming has been updated
[INF] Completed at 06/09/2024 23:57:48
```

Para finalizar as configurações do Sonarr, será necessário uma última configuração manual, em `Settings` > `Profiles`, nos dois perfis criados, `HD Blueray + WEB` e `UHD Bluray + WEB`, deverá ser alterado a *Language* de `English` para `Original`.

Similar ao Radarr existe o [Sonarr](https://sonarr.tv/), mas para séries. Ele permite baixar séries torrent e usenet de forma automática, as séries serão constantemente procuradas para possuir o melhor release possível.

Após acessar o Sonarr pela primeira vez será solicitado algumas configurações de autenticação, pode ser colocadas as seguintes:

* Selecionar em `Authentication Method`: `Forms (Login Page)`;
* Selecionar em `Authentication Required`: `Disabled for Local Addresses`;
* Escrever em `Username`: `admin`;
* Escrever em `Password` e `Password Confirmation`: `<ANY_PASSWORD>`;
* Salvar

![Sonarr Authentication](/images/sonarr-authentication.png)

> Todas as configurações do Sonaar que serão feitas pela *interface web* é necessário que as configurações avançadas estejam aparecendo, caso não estejam, basta clicar em `Show Advanced` na barra superior.

Nessa etapa será configurado o gerenciamento de mídia em `Settings` > `Media Management`, não será preciso fazer alterações nos nomes dos arquivos e pastas porque o `Recyclarr` irá fazer os ajustes futuramente.

* Marcar `Delete empty folders`;
* Selecionar em `Propers and Repacks`: `Do Not Prefer`;
* Selecionar em `Add Root Folder` : `/data/media/movies`;
* Salvar.

![Sonarr Media Management](/images/sonarr-media-management.gif)

Agora será configurado o cliente de *download* torrent em `Settings` > `Download Clients` > `qBittorrent`:

* Escrever em `Host`: `qbittorrent`;
* Escrever em `Username`: `admin`;
* Escrever em `Pasword`: `<QBITTORRENT_PASSWORD>`;
* Escrever em `Category`: `tv`;
* Salvar.

![Radaar qBittorrent Config](/images/sonaar-qbittorrent-config.gif)

Assim como no Radaar, utilizar cliente de *download* Usenet no Sonarr é interessante, mas devido aos planos de provedores serem pagos essa opção não será utilizada.

> Para saber mais sobre Usenet, voltar para a seção de cliente de *download* Usenet nas configurações do Radaar.

As próximas configurações do Sonarr são bastante demoradas para serem feitas de forma manual, para isso será utilizado os scripts do aplicativo [Recyclarr](https://recyclarr.dev/wiki/) que sincroniza as configurações do [TRaSH Guides](https://trash-guides.info/) com a aplicação local, todas as instruções são passadas pelos YAMLs na pasta `/config/configs` do `container`.

Os arquivos iniciais do Recyclarr foram criados pelo comando `docker exec recyclarr recyclarr config create -t web-1080p-v4 -t web-2160p-v4`, esse mesmo comando deverá ser utilizado novamente para a criação da estrutura inicial do Recyclarr, os templates `web-1080p-v4` e `web-2160p-v4` possuem as configurações desejadas para o Sonarr mas ainda é necessário fazer junção dessas duas configurações porque o Recyclarr não permite dois *templates* apontando para a mesma URL do servidor, o arquivo final pode ser encontrado no diretório `/config/recyclarr` desse repositório, para que ele seja utilizado os arquivos `web-1080p-v4.yml` e `web-2160p-v4.yml` deverão ser excluídos do *container* e o arquivo `sonarr.yml` deverá ser movido para o diretório `~/recyclarr/config` do *Home Server*.

> Para mais informações sobre os *templates*, voltar para a seção do Recyclarr nas configurações do Radaar.

Para utilizar o script será preciso realizar a alteração no campo `api_key` do arquivo `sonarr.yml` do diretório `~/recyclarr/config/configs`, a chave pode ser obtida no Sonarr em  `Settings` > `General` > `API Key`. Após tudo pronto, para iniciar a sincronização das configurações deverá ser executado o comando:

```bash
docker exec recyclarr recyclarr sync
```

> Para mais informações sobre as configurações alteradas pelo script, voltar para a seção do Recyclarr nas configurações do Radaar.

Caso tudo dê certo, os logs que aplicação retornará serão parecidos com esse:

```bash
===========================================
Processing Sonarr Server: [series]
===========================================

[INF] Created 43 New Custom Formats
[INF] Total of 43 custom formats were synced
[INF] Created 2 Profiles: ["WEB-1080p","WEB-2160p"]
[INF] A total of 2 profiles were synced. 2 contain quality changes and 2 contain updated scores
[INF] Processing Quality Definition: series
[INF] Number of updated qualities: 14
[INF] Media naming has been updated
[INF] Completed at 06/09/2024 23:57:51
```

Para realizar os *downloads* automáticos dos filmes e séries, será necessário configurar o [Prowlarr](https://prowlarr.com/), ele é um indexador das fontes de torrent. É através dele que o Radarr e o Sonarr entrarão em contato com os sites que disponibilizam os torrents. O Prowlarr também permite baixar outros arquivos disponíveis nos indexadores além de filmes e séries.

Após acessar o Prowlarr pela primeira vez será solicitado algumas configurações de autenticação, pode ser colocadas as seguintes:

* Selecionar em `Authentication Method`: `Forms (Login Page)`;
* Selecionar em `Authentication Required`: `Disabled for Local Addresses`;
* Escrever em `Username`: `admin`;
* Escrever em `Password` e `Password Confirmation`: `<ANY_PASSWORD>`;
* Salvar.

![Prowlarr Authentication](/images/prowlarr-authentication.png)

O próximo passo é configurar o Radarr em `Settings` > `Apps`, em `Applications` clicar em `+` e selecionar o `Radarr`. 

* Escrever em `Prowlarr Server`: `http://prowlarr:9696`;
* Escrever em `Radarr Server`: `http://radarr:7878`;
* Escrever em `API Key`: `<RADARR_API_KEY>`, essa chave pode ser obtida no Radarr em `Settings` > `General` > `API Key`;
* Salvar.

E também o Sonarr, ainda em `Settings` > `Apps`, em `Applications` clicar em `+` e selecionar o `Sonarr`.

* Escrever em `Prowlarr Server`: `http://prowlarr:9696`;
* Escrever em `Radarr Server`: `http://sonarr:8989`;
* Escrever em `API Key`: `<SONARR_API_KEY>`, essa chave pode ser obtida no Sonarr em `Settings` > `General` > `API Key`;
* Salvar.

![Prowlarr Apps Config](/images/prowlarr-apps-config.gif)

Assim como no Jackett, o Prowlarr utiliza o FlareSolverr como servidor proxy para contornar a proteção Cloudflare e DDoS-GUARD, o caminho para adicionar o FlareSolverr é em `Settings` > `Indexers` > `+` > `FlareSolverr`.

* Escrever em `Tags`: `flaresolverr`;
* Escrever em `Host`: `http://flaresolverr:8191/`;
* Salvar.

> Algumas informações importantes sobre o FlareSolverr:
> * O FlareSolverr só será usado para solicitações se e somente se o Cloudflare for detectado pelo Prowlarr.
> * O FlareSolverr só será usado para solicitações se e somente se o Proxy e o Indexador tiverem *tags correspondentes*.
> * O Flaresolverr configurado sem tags ou sem indexadores com tags correspondentes será desabilitado.

![Prowlarr Indexers Flaresolverr](/images/prowlarr-indexers-flaresolverr.gif)

Também será necessário adicionar todos os indexadores desejados em `Indexers` > `Add Indexer`, alguns indexadores precisam de ajustes antes de serem adicionados, por exemplo, o 1337x não responde na URL principal, mas na `https://1337x.st/` funciona corretamente, por isso a URL de `Base Url` deve ser alterada, ele também precisa utilizar o `flaresolverr` para resolver o captcha da Cloudflare, que pode ser visto ao acessar o site diretamente pelo navegador, por conta disso a tag `flaresolverr` deve ser adicionada ao indexador. Já o Badass Torrent não precisa de nenhuma configuração adicional, basta selecionar e salvar conforme demonstração abaixo.

> Uma possível seleção de indexadores é adicionar todos os indexadores que aparecer com o filtro `Privacy`: `Public` e `Categories`: `Movies` e `TV`

![Prowlarr Indexers Add](/images/prowlarr-indexers-add.gif)

Após serem adicionados todos os indexadores desejados, eles serão sincronizados com o Radarr e ao Sonarr, basta clicar em `Indexers` > `Sync App Indexers`, após a sincronização ser concluída os indexadores poderão ser visto no Radarr e Sonarr em `Settings` > `Indexers`.

![Prowlarr Indexers Sync](/images/prowlarr-indexers-sync.png)

Agora que o *download* de filmes e séries já está automatizado, é hora de configurar o download automático das legendas com o [Bazarr](https://www.bazarr.media/), da mesma forma que o Radarr e Sonarr procura constantemente os melhores filmes e séries, o Bazarr procura a melhor legenda disponível.

Para iniciar, será realizada a configuração do Radarr em `Settings` > `Radarr`:

* Escrever em `Address`: `radarr`;
* Escrever em `API Key`: `<RADARR_API_KEY>`, essa chave pode ser obtida no Radarr em `Settings` > `General` > `API Key`;
* Mover o `Minimum Score` para `80` (opcional);
* Salvar.

E do Sonarr em `Settings` > `Sonarr`:

* Escrever em `Address`: `sonarr`;
* Escrever em `API Key`: `<SONARR_API_KEY>`, essa chave pode ser obtida no Radarr em `Settings` > `General` > `API Key`;
* Mover o `Minimum Score` para `90` (opcional);
* Salvar.

> Não será necessário adicionar `Path Mappings` pois o mapeamento dos Bazarr é o mesmo do Radarr e Sonarr, por conta disso o Bazarr sabe encontrar os filmes e séries.

![Bazarr Settings Radarr/Sonarr](/images/bazarr-settings-radarr-sonarr.gif)

Nesse parte será configurado o idioma das legendas em `Settings` > `Languages`:

* Adicionar em `Languages Filter`: `Brazilian Portugues`;
* Clicar em `Add New Profile`, escrever em `Name`: `Brazilian Portuguese`, marcar `Exclude If Matching Audio` e salvar;
* Em `Default Settings`, marcar `Series` e `Movies` com o `Profile`: `Brazilian Portuguese`;
* Salvar.

![Bazarr Settings Languages](/images/bazarr-settings-languages.gif)

Agora será selecionado os provedores de legendas que serão utilizados em `Settings` > `Providers` > `+`. Para isso será necessário criar uma conta nos site dos provedores escolhidos, nessa configuração será utilizada somente o [opensubtitles.com](https://www.opensubtitles.com/).

* Selecionar em `Provider`: `Brazilian Portugues`;
* Clicar em `Add New Profile`, escrever em `Name`: `Brazilian Portuguese`, marcar `Exclude If Matching Audio` e salvar;
* Em `Default Settings`, marcar `Series` e `Movies` com o `Profile`: `Brazilian Portuguese`;
* Salvar.

> Caso seja utilizado provedores que possuem captcha, será necessário criar a conta em sites como o [Anti Captcha](https://anti-captcha.com/) que possuem esse serviço pago.

![Bazarr Settings Providers](/images/bazarr-settings-providers.gif)

Para finalizar as configurações do Bazarr, será feita algumas configuração extra para as legendas em `Settings` > `Subtitles`.

* Desmarcar `Use Embedded Subtitles`;
* Em `Upgrading Subtitles` mover o *slider bar* para `30`;
* Desmarcar `Upgrade Manually Downloaded or Translated Subtitles`;
* Marcar `Automatic Subtitles Synchronization`;
* Marcar `Series Score Threshold`;
* Mover o `Series Score Threshold` para `96` (opcional);
* Marcar `Movies Score Threshold`;
* Mover o `Movies Score Threshold` para `86` (opcional).

![Bazarr Settings Subtitles](/images/bazarr-settings-subtitles.gif)

Nesse ponto todo o sistema já está pronto pra baixar filmes automaticamente legendados em português, faltando somente a aplicação que irá possibilitar isso, que é o [Plex](https://www.plex.tv/pt-br/), ela mostra *thumbnails*, descrições, detalhes do filme ou série e até sugestões do que assistir, sugestões de filme baseado no que está assistindo e informações de atores, além disso tem outras funcionalidades bastante usadas no dia a dia, como botão de "pular abertura", "resumir a partir de onde parou", "pular pro próximo episódio" e "pesquisar e trocar legenda". Existem alternativas de código aberto, como o Jellyfin, mas que não está no mesmo nível que o Plex e é disponibilizado em bem menos plataformas.

O Plex possui alguns detalhes que é interessante saber, ele faz *transcoding* dos videos caso o dispositivo não tenha capacidade pra tocar o codec de vídeo, ou seja, ele vai re-encodar o vídeo antes de mandar para o *app* ou navegador que vai tocar, mas caso o dispositivo tenha suporte ao codec do video em questão ele usará o *Direct Play* ou *Direct Stream* para o *streaming* de mídia, teoricamente o Plex vai passar os bits direto, sem tentar converter, para saber mais leia o artigo [Streaming Media: Direct Play and Direct Stream](https://support.plex.tv/articles/200250387-streaming-media-direct-play-and-direct-stream/). Outro ponto é que mesmo que o dispositivo suporte o codec, mas a TV for fraca e não suporte 4K, por exemplo, o Plex teria que fazer transcoder entre qualidades diferentes, como de H.264 4K para H.264 1080p.

A maioria do conteúdo hoje em dia é codificado em H.264 ou H.265 ou até AV1, que é mais novo e mais eficiente, o problema que se o dispositivo for velho talvez não possua suporte para todos os *encode*, mas com o Plex como intermediário pode ser verificado com o dispositivo qual codificação ele suporta e fazer o *transcode* para essa versão antes de fazer o *streaming* da mídia. Mas se o servidor do Plex não possuir *hardware* o suficiente, possa ser que o *streaming* fique travando, outro motivo é que o *transcoding* via *hardware* possa não está ocorrendo, devido a não possuir o Plex Pass (versão paga) ou configuração errada do *container*, sem o Plex Pass o *transcoding* seria por *software* utilizando somente a CPU. O *Home Server* deverá ter no mínimo a capacidade para codificar vídeo em *hardware*, com ao menos as intruções Intel QSV (somente CPUs acima de 10 anos não posuem) com capacidade para H.264. Se for um servidor forte, com GPU NVidia, vai ter NVENC, que dá suporte a H.265. Se for CPU Intel moderna, vai ter suporte a AV1. Em todo caso, em Linux, essas instruções devem ficar expostas no dispositivo `/dev/dri`, por isso esse caminho está mapeado no *container* Docker.

Outro detalhe são as legendas, existem diversos tipos, como .SRT (SubRip), cSSA (SubStation Alpha), TTML (Timed Text Markup Language), VTT (Web Video Text Track), VOBSUB, etc. A maioria das trilhas de legenda foram feitas pra serem enviadas ao tocador em paralelo, o dispositivo recebe trilha de video, trilha de áudio e a trilha de legenda, mas existe um outro formato que tem capacidades gráficas, o PGS (Presentation Graphic Stream Subtitle Format), elas podem possuir cores e serem animadas, mas é preciso rasterizar e renderizar essa legenda gráfica no *streaming* de vídeo, por conta disso elas são pesadas e não devem ser utilizadas no Plex para que não ocorrá engasgos na transmissão.

Para configurar o Plex é necessário criar uma conta no [site principal](https://www.plex.tv/), clicando em `Sign Up` e seguindo as intruções, com a conta criada deverá ser acessado o [Claim Code](https://www.plex.tv/claim/) para ser passado pela variável `PLEX_CLAIM` do Docker Compose do Plex, caso o *container* tenha iniciado sem o *Claim Code* correto ele deverá ser recriado com o código obtido no *link*, essa configuração permite a conexão automaticamente do Plex Server a conta Plex. Caso tudo der certo, ao acessar o [App do Plex](https://app.plex.tv/) e ir no `Account Menu`, clicando na imagem da conta ou da primeira letra do nome, acessando a opção `Account Settings`, deverá ser visto no menu lateral esquerdo uma nova opção, o menu do servidor, esse menu informa o nome do servidor e um símbolo com cadeado (caso esteja verde a conexão está segura) e ao clicar em cima do nome do servidor irá informar que é uma conexão `Nearby`, indicando que está local, o Plex também funciona de forma remota mas com uma certa lentidão.

> Caso a aparência da conexão ao servidor esteja diferente, pode ser que o *Home Server* não esteja sendo reconhecido na rede local, portando deverá ser corrigido esse problema.

> Para poder realizar todas as configurações do Plex desse guia será preciso que as configurações avançadas estejam ativadas, algumas configurações só aparecem para os assinantes do Plex Pass.

> Todas as explicações das configurações feitas nesse guia estão no [guia do Plex no TRaSH Guides](https://trash-guides.info/Plex/Tips/Plex-media-server/).

Ainda em `Account Settings` > Submenu `Settings` > `Remote Access`.

* Marcar `Disable Remote Access`.

No menu `Account Settings` > Submenu `Settings` > `Library` deverá ser feito o seguinte procedimento:

* Marcar `Scan my library automatically`;
* Marcar `Run a partial scan when changes are detected`;
* Desmarcar `Allow media deletion`;
* Marcar `Run scanner tasks at a lower priority`;
* Selecionar em `Generate chapter thumbnails`: `as a scheduled task and when media is added`.

As próximas configurações serão de rede em `Account Settings` > Submenu `Settings` > `Network`:

* Desmarcar `Enable Relay`.

Para configurar o *Transcoder* será acessado `Account Settings` > Submenu `Settings` > `Transcoder`:

* Selecionar em `Transcoder quality`: `Automatic`.

Em `Account Settings` > Submenu `Settings` > `Languages`, terá as seguintes configurações:

* Marcar `Automatically select audio and subtitle tracks`.

Será configurado as bibliotecas dos filmes em `Account Settings` > Submenu `Manage` > `Libraries` > `Add Library` > `Movies`:

* Escrever em `Name`: `Filmes`;
* Selecionar em `Language`: `Portuguese`;
* Selecionar em `Add folder`: `/data/media/movies`;
* Marcar `Find extras`;
* Marcar `Only show trailers`;
* Marcar `Allow red band trailers`;
* Marcar `Localized subtitles`;
* Selecionar em `Minimum automatic collection size`: `2`;
* Selecionar em `Ratings Source`: `IMDb`;
* Desmarcar `Enable video preview thumbnails`;
* Selecionar em `Collections`: `Hide collections but show their items`.

Agora é a vez das séries em `Account Settings` > Submenu `Manage` > `Libraries` > `Add Library` > `TV Shows`:

* Escrever em `Name`: `Séries`;
* Selecionar em `Language`: `Portuguese`;
* Selecionar em `Add folder`: `/data/media/tv`;
* Marcar `Use season titles`;
* Marcar `Find extras`;
* Marcar `Only show trailers`;
* Marcar `Allow red band trailers`;
* Marcar `Localized subtitles`;
* Selecionar em `Ratings Source`: `IMDb`;
* Desmarcar `Enable video preview thumbnails`;
* Selecionar em `Collections`: `Hide collections but show their items`.

![Plex Settings](/images/plex-settings.gif)

Para gerenciar as solicitações e descoberta de filmes e séries pode ser utilizado o [Overseerr](https://overseerr.dev/), criado para funcionar no ecosistema Plex, possui integração com o Radarr e Sonarr, basta pesquisar no Overseerr e clicar em "Request". Com ele não será necessário acessar o Radarr e Sonarr, todas as solicitações de mídias serão feitas pela interface do Overseerr.

Ao abrir o Overseerr pela primeira vez é solicitado para que faça conexão com o Plex clicando em `Sign In` > `Entrar`, no próximo passo é solicitado as configurações do Plex que serão as seguintes:

* Escrever em `Hostname or IP Address`: <HOME-SERVER-IP>
* Clicar em `Save Changes`;
* Clicar em `Sync Libraries`;
* Marcar `Filmes`;
* Marcar `Séries`;
* Clicar em `Continue`.

Na última etapa, em `Configure Services` será solicitado para adicionar os servidores do Radarr e o Sonarr, em `+ Add Radarr Server` terá as seguintes configurações:

* Marcar `Default Server`;
* Marcar `4K Server`;
* Escrever em `Server Name`: `Radarr`;
* Escrever em `Hostname or IP Address`: `192.168.100.100`, anteriormente `radarr` mas alterado para uma URL que seja possível acessar fora do *container*, ao em clicar `Open in Radarr` em `Manage Movie`;
* Escrever em `API Key`: `<RADARR_API_KEY>`;
* Clicar em `Test`;
* Selecionar em `Quality Profile`: `UHD Bluray + Web`;
* Selecionar em `Root Folder`: `/data/media/movies`;
* Selecionar em `Minimum Availability`: `Released`;
* Marcar `Enable Scan`;
* Clicar em `Add Server`.

Já as configurações do Sonarr em `+ Add Sonarr Server` serão as seguintes:

* Marcar `Default Server`;
* Marcar `4K Server`;
* Escrever em `Server Name`: `Sonarr`;
* Escrever em `Hostname or IP Address`: `192.168.100.100`, anteriormente `sonarr` mas alterado para uma URL que seja possível acessar fora do *container*, ao em clicar `Open in Sonarr` em `Manage Series`;
* Escrever em `API Key`: `<SONARR_API_KEY>`;
* Clicar em `Test`;
* Selecionar em `Quality Profile`: `WEB-2160p`;
* Selecionar em `Root Folder`: `/data/media/tv`;
* Selecionar em `Language Profile`: `Deprecated`;
* Marcar `Season Folders`;
* Marcar `Enabled Scan`;
* Clicar em `Add Server`;
* Clicar em `Finish Setup`.

![Overseerr Config](/images/overseerr-config.gif)

> Após a configuração inicial feita, o idioma pode ser trocado em `Settings` > `Display Language`.

Para finalizar o guia será mostrado como adicionar o filme ou série para que seja baixado. Na interface do Overseer, basta pesquisar o nome do filme ou série e clicar em `Request`, no menu lateral em `Requests` poderá ser visto o status da pesquisa

## Outros Serviços

Existem alguns outros serviços/sites que podem ser utilizados nesse projeto, mas por não achar necessário não foram incluídos:

- [Letterboxd](https://letterboxd.com/) é uma rede social para discutir e descobrir sobre filmes de base. Pode ser utilizado a função de Listas para que o Radarr baixe automaticamente qualquer vídeo adicionado lá.

- [Lidaar](https://lidarr.audio/) é um gerenciador de coleção de música para usuários Usenet e BitTorrent, similar ao Radaar ou Sonaar.

- [LibreSpeed](https://librespeed.org/) é um teste de velocidade muito leve implementado em Javascript, usando XMLHttpRequest e Web Workers, a imagem do *container* pode ser encontrada [aqui](https://hub.docker.com/r/linuxserver/librespeed).

- [MakeMKV](https://www.makemkv.com/) é utilizado para "ripar" Blue-Ray e DVDs para o formato MKV, para isso é necessário modelos específicos de leitores de disco que pode ser encontrado no [fórum do MKV](https://forum.makemkv.com/forum/viewforum.php?f=16). Caso seja necessário "ripar" discos, o *container* [jlesage/makemkv](https://hub.docker.com/r/jlesage/makemkv) permite acesso pelo navegador a interface gráfica do aplicativo (como se fosse um cliente remoto, como VNC). Ele vai usar a biblioteca [LibreDrive](https://forum.makemkv.com/forum/viewtopic.php?t=18856) pra descriptografar os *streams* de dados do Blu-Ray e permitir copiar bit a bit. É o que se chama de REMUX (remuxing), quando não se recodificar ou "re-encodar" o vídeo.

- [Kavita](https://www.kavitareader.com/) é uma biblioteca digital, similar ao Plex, só que para mangás e quadrinhos, permite integração com metadados externos (para baixar capas e informações)e outras funcionalidades, pode ser utilizado pelo *container* [linuxserver/kavita](https://hub.docker.com/r/linuxserver/kavita). Pode ser utilizado em conjunto com o [Kaizoku](https://github.com/oae/kaizoku), para que seja feito o *download* automático dos mangás.

## Informações

Os vídeos possuem CODEC que faz o *encodings* dos *streams* de video e *streams* de audio. Por exemplo, H.265 ou AV1, são codecs de video. MP3 ou AAC são codecs de áudio. Para empacotar os dois streams (ou mais, caso tenha múltiplas dublagens, por exemplo) em um "arquivo", pode ser utilizado por exemplo o ".mp4", ".mov" ou ".mkv". Normalmente queremos o MKV ou Matroska, que é um formato de envelope de arquivo bem flexível, permite múltiplos *streams* de tudo, incluindo de legenda, por isso é muito usado pra compartilhar em torrent.

> Para ter os vídeos nas melhores qualidades possível pode ser baixado os BR-DISK que é backup de UHD e depois refazer o encoding para o perfil Matroska H.265.

Caso esteja o *Home Server* esteja sendo utilizado por WSL2 será necessário usar o comando `wsl --exec dbus-launch true` no Windows PowerShell para evitar que o WSL2 para automaticamente ao fechar o Terminal.

## Extra

Para entender o tempo que o `Reclycarr` diminui na configuração do Radaar e Sonaar será explicado as etapas manuais que deveriam ser realizadas, nesse exemplo será utilizando os procedimentos que seriam feitos no Radarr. Para começar, seria necessário a inclusão dos formatos personalizados, em `Settings` > `Custom Formats` > `+` > `Import` e importar todos os `Custom Formats` dos `Quality Profiles` `HD Bluray + WEB` e `UHD Bluray + WEB` do [How to set up Quality Profiles](https://trash-guides.info/Radarr/radarr-setup-quality-profiles/) do TRaSH Guides.

![Radaar Custom Formats](/images/radaar-custom-formats.gif)

Depois de adicionar os `Custom Formats`, seria preciso configurá-los no `Quality Profiles`, inicialmente seria criado dois *profiles* chamados de `HD Bluray + WEB` e `UHD Bluray + WEB`, eles deveriam possuir todas as configurações indicadas em [How to set up Quality Profiles](https://trash-guides.info/Radarr/radarr-setup-quality-profiles/), nessa etapa todos os *scores* dos formatos personalizados deveria ser passado a mão, pois não aparece de forma automática.

![Radaar Profiles](/images/radaar-profiles.gif)

## Atualizações Futuras

* Adicionar novos *container* de Radarr e Sonarr para baixar mídias 4K e Full HD em locais diferentes.

## Referência

[TRaSH-Guides](https://trash-guides.info/)

[Meu "Netflix Pessoal" com Docker Compose](https://www.akitaonrails.com/2024/04/03/meu-netflix-pessoal-com-docker-compose)

[Guia do Streaming Doméstico Automatizado (Sonarr, Radarr e Plex)](https://www.reddit.com/r/pirataria/comments/18ch7bt/guia_do_streaming_dom%C3%A9stico_automatizado_sonarr/)
