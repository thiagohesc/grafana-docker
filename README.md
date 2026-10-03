# Grafana com Zabbix e Docker Compose

Implantação do Grafana para visualizar dados e criar dashboards, com banco SQLite persistente, plugin Zabbix e acesso HTTPS por um Traefik externo.

Este projeto inicia apenas o Grafana. O Traefik e o Zabbix precisam estar disponíveis separadamente; a conexão com a API do Zabbix é configurada após a instalação.

## Sumário

- [Como funciona](#como-funciona)
- [Pré-requisitos](#pré-requisitos)
- [Instalação e configuração](#instalação-e-configuração)
- [Configurar o Zabbix](#configurar-o-zabbix)
- [Operação e manutenção](#operação-e-manutenção)
- [Solução de problemas](#solução-de-problemas)

## Como funciona

O navegador acessa o domínio do Grafana por HTTPS. O Traefik recebe a requisição e encaminha o tráfego para a porta interna `3000` do contêiner, pela rede Docker compartilhada. O Grafana guarda suas configurações no SQLite e consulta as fontes de dados configuradas, como a API do Zabbix.

| Recurso | Configuração neste projeto |
| --- | --- |
| Imagem | `grafana/grafana:13.2.3`. |
| Plugin Zabbix | `alexanderzobnin-zabbix-app@6.8.0`, instalado durante a inicialização. |
| Banco de dados | SQLite em `/var/lib/grafana/grafana.db`. |
| Armazenamento | Volume Docker `grafana_data`, montado em `/var/lib/grafana`. |
| HTTPS | Ponto de entrada `websecure` do Traefik, com TLS e resolvedor `cloudflare`. |
| Rede | Rede Docker externa, definida em `TRAEFIK_NET`. |
| Porta interna | HTTP na porta `3000`, sem publicação de porta no servidor. |
| Autenticação | Administrador inicial definido no `.env`; acesso anônimo e cadastro público desabilitados. |
| Logs | Logs da aplicação em JSON; rotação do Docker em até três arquivos de 10 MB cada. |
| Verificação de saúde | Verificação interna a cada 30 segundos, com até três tentativas antes de indicar falha. |
| Reinício | Política `unless-stopped`. |

### Arquivos do projeto

```text
.
├── docker-compose.yml       # Configuração do Grafana e regras do Traefik
├── env.example              # Modelo das variáveis de ambiente
├── .env                     # Configuração local, criada durante a instalação
├── .gitignore               # Arquivos que não devem ser versionados
└── README.md
```

O `.env` é ignorado pelo Git. Os dados persistentes ficam em um volume gerenciado pelo Docker, separado dos arquivos do projeto.

## Pré-requisitos

- Docker Engine em execução e Docker Compose com o comando `docker compose`.
- Permissão para executar comandos Docker.
- Domínio com DNS apontando para o endereço de entrada do Traefik.
- Traefik com provedor Docker habilitado, ponto de entrada `websecure` e resolvedor de certificados chamado `cloudflare` configurados.
- Rede Docker externa compartilhada entre Traefik e Grafana.
- Acesso à internet para baixar a imagem e instalar o plugin.
- Para usar Zabbix: API acessível a partir do contêiner do Grafana e credenciais com permissão de leitura dos hosts desejados.

As etiquetas (`labels`) deste projeto usam os nomes `websecure` e `cloudflare`. Se o seu Traefik usar outros nomes, ajuste as etiquetas em `docker-compose.yml`. As credenciais do Cloudflare e a configuração de emissão de certificados pertencem ao ambiente do Traefik. Consulte a [documentação do provedor Docker do Traefik](https://doc.traefik.io/traefik/reference/routing-configuration/other-providers/docker/).

Confira a instalação das ferramentas:

```bash
docker --version
docker compose version
```

## Instalação e configuração

Execute os comandos a seguir na pasta deste projeto. Os exemplos usam Bash.

### 1. Preencher o arquivo de ambiente

Copie o modelo somente se ainda não existir um `.env`, para preservar configurações existentes:

```bash
if [ ! -e .env ]; then
  cp env.example .env
fi
chmod 600 .env
```

Abra o `.env` no seu editor e substitua os valores de exemplo:

```dotenv
TRAEFIK_NET=traefik-net
GRAFANA_DOMAIN=grafana.seudominio.com.br
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=SUBSTITUA_POR_UMA_SENHA_FORTE
GRAFANA_SECRET_KEY=SUBSTITUA_POR_UMA_CHAVE_ALEATORIA
```

| Variável | Obrigatória | Descrição |
| --- | --- | --- |
| `TRAEFIK_NET` | Sim | Nome exato da rede externa usada pelo Traefik. |
| `GRAFANA_DOMAIN` | Sim | Domínio público, sem `https://`, porta ou caminho. |
| `GRAFANA_ADMIN_USER` | Não | Usuário administrador inicial; padrão `admin`. |
| `GRAFANA_ADMIN_PASSWORD` | Sim | Senha forte do administrador inicial. |
| `GRAFANA_SECRET_KEY` | Sim | Chave usada para proteger segredos armazenados pelo Grafana. |

Se tiver OpenSSL instalado, gere um valor aleatório para a chave e copie o resultado para `GRAFANA_SECRET_KEY`:

```bash
openssl rand -hex 32
```

Preserve essa chave junto aos backups. Alterá-la sem um procedimento de migração pode impedir o uso de credenciais criptografadas. O usuário e a senha administrativos são aplicados na criação inicial do banco; editar o `.env` não redefine a senha de um usuário já existente. Consulte a [configuração de segurança do Grafana](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#security).

### 2. Preparar a rede compartilhada

Para o nome usado no exemplo:

```bash
docker network inspect traefik-net
```

Se a rede ainda não existir, crie-a:

```bash
docker network create traefik-net
```

Use o nome definido em `TRAEFIK_NET` se ele for diferente. O contêiner do Traefik também deve participar dessa rede; criá-la não conecta automaticamente um Traefik existente. O Compose não cria essa rede porque ela está declarada como `external: true`.

### 3. Validar e iniciar o Grafana

```bash
docker compose config --quiet
docker compose pull grafana
docker compose up -d grafana
docker compose ps
docker compose logs --tail=100 grafana
```

O Docker Compose carrega o `.env` automaticamente e rejeita variáveis obrigatórias ausentes ou vazias. Os textos `SUBSTITUA_POR_...` passam nessa validação, por isso substitua-os antes de iniciar. A opção `--quiet` evita imprimir a configuração completa, que contém segredos.

Depois da inicialização, o serviço deve aparecer em execução e, após as verificações internas, com o estado `healthy`.

### Conferir o acesso

Acesse `https://grafana.seudominio.com.br`, substituindo pelo seu domínio, e entre com as credenciais configuradas no `.env`.

## Configurar o Zabbix

O plugin é instalado pelo `GF_PLUGINS_PREINSTALL`. A instalação ocorre em segundo plano e pode terminar após o Grafana começar a responder; acompanhe os logs se o plugin ainda não aparecer. Veja a [instalação de plugins na imagem Docker do Grafana](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/#install-plugins-in-the-docker-container).

1. Acesse **Administration → Plugins and data → Plugins**, procure **Zabbix** e habilite o plugin com **Enable**.
2. Em **Connections**, adicione uma fonte de dados **Zabbix**.
3. Informe a URL completa da API, por exemplo `https://zabbix.seudominio.com.br/api_jsonrpc.php`. Algumas instalações usam o caminho `/zabbix/api_jsonrpc.php`.
4. Configure usuário e senha ou token de API com acesso aos hosts que serão consultados.
5. Execute **Save & test** e crie ou importe os dashboards desejados.

O endereço precisa ser acessível pelo contêiner do Grafana. `localhost` dentro dele aponta para o próprio Grafana. A instalação do plugin não cria fontes de dados nem dashboards automaticamente. Consulte a [configuração oficial do plugin Zabbix](https://grafana.com/docs/plugins/alexanderzobnin-zabbix-app/latest/configure/).

## Operação e manutenção

Execute os comandos de administração do Grafana na pasta deste projeto.

| Ação | Comando |
| --- | --- |
| Validar a configuração | `docker compose config --quiet` |
| Consultar o estado | `docker compose ps` |
| Acompanhar os logs | `docker compose logs -f --tail=100 grafana` |
| Reiniciar o processo | `docker compose restart grafana` |
| Aplicar alterações no Compose ou `.env` | `docker compose up -d grafana` |
| Parar o serviço | `docker compose stop grafana` |
| Retomar o contêiner parado | `docker compose start grafana` |
| Remover o contêiner mantendo os dados | `docker compose down` |
| Conferir a saúde interna | `docker compose exec grafana wget -q -O - http://127.0.0.1:3000/api/health` |

A verificação de saúde consulta `/api/health` a cada 30 segundos, com timeout de 5 segundos, 3 tentativas e período inicial de 60 segundos. Ela verifica o Grafana internamente; confirme também o acesso público e a conexão com o Zabbix.

Um `restart` não aplica mudanças nas variáveis de ambiente. Após editar a configuração, use `up -d`, conforme a [documentação do Docker Compose](https://docs.docker.com/reference/cli/docker/compose/restart/).

### Preservar a configuração e os dados

O volume armazena o banco SQLite e os plugins. O banco contém usuários, dashboards, fontes de dados e outras configurações; o histórico das métricas continua na fonte de dados, como o Zabbix.

O nome físico do volume normalmente inclui o nome do projeto Compose, como `grafana-docker_grafana_data`. Mudar o diretório ou o nome do projeto pode fazer o Compose usar outro volume. Para identificar o volume atual:

```bash
docker inspect grafana --format '{{range .Mounts}}{{if eq .Destination "/var/lib/grafana"}}{{.Name}}{{end}}{{end}}'
```

`docker compose down` preserva o volume. **Não use `docker compose down -v` para uma parada normal: ele remove o volume e os dados persistidos.** Consulte o [comportamento de remoção do Compose](https://docs.docker.com/reference/cli/docker/compose/down/).

#### Criar um backup

Pare o Grafana durante a cópia para manter a integridade do SQLite, como orienta a [documentação de backup do Grafana](https://grafana.com/docs/grafana/latest/administration/back-up-grafana/). O procedimento causa uma breve indisponibilidade.

O exemplo usa a imagem do contêiner existente para executar apenas `tar`, com o volume em modo de leitura e sem rede. Salve os arquivos fora do repositório, pois o backup também contém o `.env`:

```bash
(
  set -e
  umask 077
  BACKUP_DIR="$HOME/backups/grafana/$(date +%Y%m%d-%H%M%S)"
  mkdir -p "$BACKUP_DIR"
  cp docker-compose.yml .env "$BACKUP_DIR/"
  GRAFANA_IMAGE="$(docker inspect grafana --format '{{.Config.Image}}')"

  docker compose stop grafana
  trap 'docker compose start grafana' EXIT
  docker run --rm --network none \
    --volumes-from grafana:ro \
    --entrypoint tar "$GRAFANA_IMAGE" \
    -czf - -C /var/lib/grafana . > "$BACKUP_DIR/grafana-data.tar.gz"

  tar -tzf "$BACKUP_DIR/grafana-data.tar.gz" > /dev/null
  printf 'Backup verificado em: %s\n' "$BACKUP_DIR"
)
```

O bloco interrompe a cópia se a parada falhar e tenta retomar o Grafana ao terminar, inclusive em caso de erro no backup. Confirme que a verificação e a retomada terminaram sem erro. Se necessário, retome manualmente com `docker compose start grafana`. Mantenha uma cópia protegida em outro local.

#### Restaurar um backup

Faça a restauração em um volume novo e vazio, preservando o volume anterior caso ele ainda exista. Copie o `docker-compose.yml` e o `.env` do backup para a pasta de recuperação, mantenha a mesma `GRAFANA_SECRET_KEY` e prepare a rede externa. Use a versão de Grafana registrada no backup.

No exemplo abaixo, substitua o caminho do backup. `docker compose create` prepara o contêiner e seu volume sem iniciar o Grafana:

```bash
(
  set -e
  BACKUP_DIR="$HOME/backups/grafana/AAAAmmdd-HHMMSS"
  tar -tzf "$BACKUP_DIR/grafana-data.tar.gz" > /dev/null
  docker compose config --quiet
  docker compose create grafana
  docker compose stop grafana
  GRAFANA_IMAGE="$(docker inspect grafana --format '{{.Config.Image}}')"
  test "$(docker inspect grafana --format '{{.State.Running}}')" = false

  docker run --rm -i --network none --user 0 \
    --volumes-from grafana \
    --entrypoint sh "$GRAFANA_IMAGE" \
    -c 'test ! -e /var/lib/grafana/grafana.db && exec tar -xzf - -C /var/lib/grafana' \
    < "$BACKUP_DIR/grafana-data.tar.gz"

  docker compose up -d grafana
  docker compose ps
  docker compose logs --tail=100 grafana
)
```

O bloco aborta em caso de erro, verifica que o contêiner está parado e recusa a extração se já existir um `grafana.db` no volume. O nome fixo impede criar outro contêiner chamado `grafana` no mesmo host; prepare o ambiente de recuperação antes desses comandos. Após restaurar, confira login, dashboards e fontes de dados.

### Atualizar o Grafana e o plugin Zabbix

1. Faça um backup do volume e da configuração.
2. Confira as notas da versão e a compatibilidade entre Grafana e plugin Zabbix.
3. Altere a tag de `image` e/ou a versão em `GF_PLUGINS_PREINSTALL` no `docker-compose.yml`.
4. Aplique e acompanhe a atualização:

```bash
docker compose config --quiet
docker compose pull grafana
docker compose up -d grafana
docker compose logs --tail=100 grafana
```

As versões estão fixadas; executar `pull` sozinho não seleciona uma versão mais nova. Para recuperar uma atualização que alterou o banco, use o backup e a versão correspondente. Consulte as [orientações de atualização do Grafana](https://grafana.com/docs/grafana/latest/upgrade-guide/).

### Segurança e logs

- Acesso anônimo, cadastro público, criação de organizações por usuários e incorporação em iframe estão desabilitados.
- Cookies usam `Secure` e `SameSite=lax`; utilize o endereço HTTPS configurado.
- O papel padrão para usuários adicionados à organização é `Viewer`.
- O contêiner usa `no-new-privileges:true`.
- Logs do Grafana vão para o console em JSON, no nível `info`. O Docker mantém até 3 arquivos de 10 MB por contêiner.
- Preserve o `.env`, a chave de criptografia e os backups em local protegido; não os adicione ao Git.

## Solução de problemas

Comece pela validação da configuração e pelos logs:

```bash
docker compose config --quiet
docker compose ps
docker compose logs --tail=100 grafana
```

| Sintoma | O que verificar |
| --- | --- |
| Erro `Defina ...` ao validar ou iniciar | Preencha a variável indicada no `.env` e execute `docker compose config --quiet`. |
| Rede externa não encontrada | Confira `TRAEFIK_NET` e a existência da rede; conecte também o Traefik. |
| Erro de nome de contêiner em uso | Confira `docker ps -a --filter name=grafana` e identifique a instância existente. |
| Resposta 404 pelo Traefik | Confira DNS, domínio da regra `Host`, provedor Docker e ponto de entrada `websecure`. |
| Resposta 502/504 pelo Traefik | Confira a rede compartilhada, o estado do Grafana e o encaminhamento interno para a porta `3000`. |
| Erro de certificado HTTPS | Confira o resolvedor `cloudflare`, suas credenciais e os logs do Traefik. |
| Contêiner `unhealthy` | Consulte os logs e teste `/api/health` dentro do contêiner. |
| Plugin Zabbix ausente | Aguarde a instalação, confira conectividade de saída, versão configurada e logs; depois habilite o plugin. |
| Teste da fonte Zabbix falha | Confira URL da API, conectividade a partir do Grafana e permissões das credenciais. |
| Senha antiga continua após editar `.env` | Altere a senha pela interface com um administrador; as variáveis definem apenas o administrador inicial. |
| Credenciais de fontes falham após restaurar | Confira se a `GRAFANA_SECRET_KEY` é a mesma usada no backup. |
| Instalação aparece vazia após mudança de pasta | Confira o nome do projeto Compose e o volume realmente montado. |
