# Docker-Composes

Biblioteca pessoal de ambientes **Docker Compose** reutilizáveis para desenvolvimento local, estudos e consultas.

Cada tecnologia fica em uma pasta própria e funciona de forma **independente**: você sobe só o que precisa, sem depender dos demais ambientes.

## Pré-requisitos

| Ferramenta | Versão mínima | Observação |
|---|---|---|
| Docker Desktop (Windows/macOS) **ou** Docker Engine (Linux) | Engine 24+ | No Windows, use o backend **WSL2** |
| Docker Compose | v2 ou superior (`docker compose`, com espaço) | Já incluído no Docker Desktop. O antigo `docker-compose` (v1, com hífen) não é suportado |
| Memória para o Docker | 4 GB (8 GB se for usar o SonarQube junto com outros ambientes) | Docker Desktop → Settings → Resources |

Para conferir a instalação:

```bash
docker version
docker compose version
```

## Tecnologias disponíveis

| Tecnologia | Finalidade | Portas no host (padrão) | Diretório |
|---|---|---|---|
| PostgreSQL | Banco de dados relacional | `5432` | [`postgres/`](postgres/) |
| Apache Kafka (KRaft) + Kafka UI | Streaming de eventos / mensageria | `9092` (Kafka), `19000` (UI) | [`kafka/`](kafka/) |
| RabbitMQ | Message broker (AMQP) + painel web | `5672` (AMQP), `15672` (painel) | [`rabbitmq/`](rabbitmq/) |
| Redis | Banco chave-valor em memória / cache | `6379` | [`redis/`](redis/) |
| SonarQube | Análise estática de código (Community Build) | `9000` | [`sonarqube/`](sonarqube/) |

Por padrão, todas as portas são publicadas apenas em `127.0.0.1`, ou seja, ficam acessíveis só a partir da sua máquina. Veja [Acesso a partir de outras máquinas](#acesso-a-partir-de-outras-máquinas).

## Atualizando a partir da versão anterior do repositório

As pastas `PostgreSQL/` e `Kafka/` foram renomeadas para `postgres/` e `kafka/`, e os arquivos `docker-compose.yml` para `compose.yaml`. Nenhum dado é apagado ou migrado automaticamente:

- **PostgreSQL:** por padrão, o ambiente passa a usar o volume nomeado `postgres_data`. Para continuar usando os dados em `C:/Meu_Banco_Postgres`, defina `POSTGRES_DATA_SOURCE=C:/Meu_Banco_Postgres` no `.env`. A senha desses dados continua `admin`. Passo a passo em [`postgres/README.md`](postgres/README.md#migração-da-configuração-anterior).
- **Kafka:** os containers agora se chamam `kafka` e `kafka-ui`. Remova os antigos com `docker rm -f KAFKA KAFKA-UI`. Detalhes em [`kafka/README.md`](kafka/README.md#mudanças-em-relação-à-configuração-anterior).

> No Windows, o sistema de arquivos não diferencia maiúsculas de minúsculas. Depois do `git pull`, a pasta pode continuar aparecendo como `Kafka/` no Explorer, mas `cd kafka` funciona normalmente. Se o Git indicar arquivos inconsistentes, clone o repositório novamente.

## Estrutura

```
Docker-Composes/
├── postgres/
├── kafka/
├── rabbitmq/
│   ├── compose.yaml      # definição dos serviços
│   ├── .env.example      # variáveis documentadas (copie para .env)
│   └── README.md         # documentação do ambiente
├── redis/
├── sonarqube/
├── .gitignore
└── README.md
```

### Convenções

Todas as pastas seguem o mesmo padrão:

- O arquivo se chama `compose.yaml` (nome padrão do Compose moderno) e não tem a chave obsoleta `version:`.
- A chave `name:` fixa o nome do projeto. Assim, containers, redes e volumes têm nomes previsíveis, como `redis_data` e `sonarqube_db-data`, mesmo se a pasta for renomeada.
- As imagens são oficiais e usam **tags explícitas** (nunca `latest`). A tag pode ser trocada no `.env`.
- Portas, credenciais e tags vêm de variáveis de ambiente, documentadas no `.env.example`.
- Senhas obrigatórias usam a sintaxe `${VAR:?mensagem}`. Sem o `.env`, o Compose falha com uma mensagem clara em vez de subir com uma senha padrão.
- Volumes nomeados guardam os dados persistentes.
- Todo serviço tem um `healthcheck`, e as dependências usam `depends_on: condition: service_healthy`.
- Os serviços usam `restart: unless-stopped`: voltam com o Docker, mas respeitam um `stop` manual.

### Adicionar uma nova tecnologia

1. Crie a pasta `<tecnologia>/` com `compose.yaml`, `.env.example` e `README.md`, seguindo as convenções acima.
2. Escolha portas que não conflitem com a tabela de tecnologias.
3. Adicione uma linha na tabela deste README.

## Configuração do `.env`

Cada ambiente lê as variáveis do arquivo `.env` **da própria pasta**. Esse arquivo é ignorado pelo Git. Só o `.env.example` é versionado.

```bash
cd redis
cp .env.example .env     # funciona em bash e PowerShell
# edite o .env e troque as senhas
```

- Os valores do `.env.example` são **exemplos para desenvolvimento**, não credenciais reais.
- Para conferir o resultado final, com as variáveis já substituídas, use `docker compose config`.
- Em geral, alterar o `.env` exige recriar o container: `docker compose up -d`. O Compose detecta a mudança sozinho.
- Credenciais de bancos e brokers (PostgreSQL, RabbitMQ) são aplicadas só na **primeira inicialização**, com o volume vazio. Mudar a senha no `.env` depois **não** altera o usuário já criado. Veja o README de cada ambiente.

## Comandos básicos do Docker Compose

Execute os comandos dentro da pasta do ambiente:

```bash
docker compose up -d                 # cria e inicia em segundo plano
docker compose up -d --wait          # idem, mas aguarda os healthchecks ficarem "healthy"
docker compose ps                    # status dos serviços (inclui saúde)
docker compose logs -f [serviço]     # acompanha os logs
docker compose exec <serviço> sh     # abre um shell no container
docker compose stop                  # para os containers (sem removê-los)
docker compose start                 # inicia containers parados
docker compose restart [serviço]     # reinicia
docker compose down                  # remove containers e rede (MANTÉM os volumes)
docker compose pull                  # baixa as imagens das tags configuradas
docker compose config                # valida e exibe a configuração final
```

Também é possível rodar de qualquer diretório:

```bash
docker compose -f redis/compose.yaml --env-file redis/.env up -d
```

## Persistência e remoção de volumes

Os dados ficam em **volumes nomeados**, gerenciados pelo Docker. Eles sobrevivem a `docker compose down` e a atualizações de imagem.

```bash
docker volume ls                          # lista os volumes
docker volume inspect redis_data          # detalhes de um volume
docker compose down -v                    # remove containers E volumes do ambiente (APAGA os dados)
docker volume rm redis_data               # remove um volume específico
docker system df -v                       # espaço usado por imagens, containers e volumes
```

Backup simples de um volume (exemplo com `redis_data`):

```bash
# Linux/macOS/Git Bash
docker run --rm -v redis_data:/data -v "$PWD":/backup alpine:3.22 tar czf /backup/redis_data.tar.gz -C /data .
```

```powershell
# PowerShell
docker run --rm -v redis_data:/data -v "${PWD}:/backup" alpine:3.22 tar czf /backup/redis_data.tar.gz -C /data .
```

> ⚠️ `down -v` e `docker volume rm` são **irreversíveis**. Antes de atualizar a **versão principal** de um banco de dados (por exemplo, PostgreSQL 16 → 17), faça backup: o formato dos arquivos muda entre versões principais e a nova imagem não abre o volume antigo.

## Acesso a partir de outras máquinas

Por segurança, as portas são publicadas em `127.0.0.1`. Para acessar a partir de outra máquina da rede (um celular ou uma VM, por exemplo), defina no `.env` do ambiente:

```dotenv
BIND_ADDRESS=0.0.0.0
```

Faça isso apenas em redes confiáveis e com senhas fortes.

Para **containers** de outros projetos acessarem estes serviços:

- Use `host.docker.internal:<porta>`. Funciona no Docker Desktop. No Linux, adicione ao serviço cliente:

  ```yaml
  extra_hosts:
    - "host.docker.internal:host-gateway"
  ```

- Ou conecte os dois projetos a uma mesma rede Docker externa.

## Solução de problemas frequentes

| Problema | Solução |
|---|---|
| `Bind for 127.0.0.1:XXXX failed: port is already allocated` | Outro processo ou container usa a porta. Descubra qual com `docker ps` ou `netstat -ano \| findstr :XXXX` (Windows) / `ss -ltnp \| grep XXXX` (Linux). Depois, altere a porta no `.env`. |
| `required variable ... is missing a value` / `Defina ... no arquivo .env` | O `.env` não existe. Rode `cp .env.example .env`. |
| Container em `unhealthy` ou reiniciando | Veja os logs com `docker compose logs <serviço>`. Muitas vezes é falta de memória ou senha incompatível com o volume existente. |
| Autenticação falha depois de trocar a senha no `.env` | A senha antiga está gravada no volume. Altere-a pela ferramenta da tecnologia ou recrie o volume (`down -v`, que apaga os dados). |
| `docker: command not found` / `docker compose` não existe | Instale o Docker Desktop ou o plugin `docker-compose-plugin` (Linux). |
| SonarQube não sobe (`vm.max_map_count`) | Veja os requisitos do host em [`sonarqube/README.md`](sonarqube/README.md#7-requisitos-do-host). |
| Lentidão ou containers encerrados com código `137` | Falta memória. Aumente os recursos do Docker Desktop ou pare os ambientes que não estão em uso. |
| No Windows, `localhost` não responde | Teste com `127.0.0.1`. Confira se o Docker Desktop está usando WSL2 e se a porta está publicada (`docker compose ps`). |
