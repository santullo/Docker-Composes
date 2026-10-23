# PostgreSQL

## 1. O que é

O [PostgreSQL](https://www.postgresql.org/) é um banco de dados **relacional** open source. Ele suporta SQL padrão, transações ACID, chaves estrangeiras, *views*, *functions*, índices avançados (B-tree, GIN, GiST, BRIN) e tipos como `JSONB`, arrays e intervalos.

## 2. Para que serve

- Banco principal de aplicações (por exemplo, Spring Boot com Spring Data JPA/Hibernate).
- Estudo de SQL, modelagem relacional, índices e planos de execução (`EXPLAIN ANALYZE`).
- Testar migrações de esquema com Flyway ou Liquibase.

## 3. Quando utilizar

Use PostgreSQL quando os dados forem **estruturados e relacionados entre si**, quando precisar de **integridade** (constraints, transações) ou de consultas complexas com `JOIN`s e agregações. Para cache ou dados efêmeros de acesso por chave, veja o [Redis](../redis).

## 4. Serviços inicializados

| Serviço | Imagem | Descrição |
|---|---|---|
| `postgres` | `postgres:18` (oficial, Debian) | Servidor PostgreSQL com healthcheck via `pg_isready` |

Os dados ficam no volume nomeado `postgres_data`, montado em `/var/lib/postgresql`. A partir da versão 18, a imagem oficial grava em `/var/lib/postgresql/18/docker`, e não mais em `/var/lib/postgresql/data`. Assim, o mesmo volume pode guardar versões principais diferentes, o que ajuda em atualizações com `pg_upgrade`.

## 5. Portas

| Porta (host) | Porta (container) | Uso |
|---|---|---|
| `127.0.0.1:5432` | `5432` | Protocolo PostgreSQL (JDBC, psql, IDEs) |

## 6. Variáveis de ambiente

| Variável | Padrão no `.env.example` | Descrição |
|---|---|---|
| `POSTGRES_IMAGE_TAG` | `18` | Tag da imagem. Mantenha a mesma versão principal dos dados existentes. |
| `BIND_ADDRESS` | `127.0.0.1` | Interface do host onde a porta é publicada |
| `POSTGRES_PORT` | `5432` | Porta no host |
| `POSTGRES_USER` | `postgres` | Superusuário criado na 1ª inicialização |
| `POSTGRES_PASSWORD` | `troque-esta-senha` | Senha do superusuário (**obrigatória**) |
| `POSTGRES_DB` | `postgres` | Banco criado na 1ª inicialização |
| `POSTGRES_DATA_SOURCE` | `data` | Origem dos dados: `data` (volume nomeado) ou um caminho de pasta no host |

> **Atenção:** `POSTGRES_USER`, `POSTGRES_PASSWORD` e `POSTGRES_DB` só são aplicados quando o diretório de dados está **vazio**. Com dados existentes, a senha válida continua sendo a que foi gravada na primeira inicialização.

## 7. Iniciar e interromper

```bash
cd postgres
cp .env.example .env        # edite a senha (e POSTGRES_DATA_SOURCE, se for migrar)
docker compose up -d
docker compose ps           # aguarde "healthy"
docker compose logs -f
docker compose stop
docker compose down         # mantém os dados
docker compose down -v      # APAGA o volume postgres_data (não afeta pastas do host)
```

## 8. Verificar o funcionamento

```bash
docker compose ps
docker compose exec postgres pg_isready -U postgres

# Abrir o psql dentro do container
docker compose exec postgres psql -U postgres
```

Comandos úteis no `psql`:

```sql
\l                 -- lista bancos
\c meu_banco       -- conecta a outro banco
\dt                -- lista tabelas
\d nome_tabela     -- estrutura de uma tabela
\du                -- lista usuários/roles
\q                 -- sai

CREATE DATABASE minha_api;
CREATE USER minha_api WITH PASSWORD 'senha-da-app';
GRANT ALL PRIVILEGES ON DATABASE minha_api TO minha_api;
ALTER DATABASE minha_api OWNER TO minha_api;   -- permite criar tabelas no schema public
```

Backup e restauração:

```bash
# Gera um dump do banco "postgres" no host
docker compose exec -T postgres pg_dump -U postgres -d postgres > backup.sql

# Restaura (no PowerShell, use: Get-Content backup.sql | docker compose exec -T postgres psql -U postgres -d postgres)
docker compose exec -T postgres psql -U postgres -d postgres < backup.sql
```

## 9. Conectar uma aplicação

| Parâmetro | Valor |
|---|---|
| Host | `localhost` |
| Porta | `5432` |
| Banco | `postgres` (ou o banco que você criar) |
| Usuário | `postgres` |
| Senha | a do `.env` |
| URL JDBC | `jdbc:postgresql://localhost:5432/postgres` |

### Spring Boot

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <scope>runtime</scope>
</dependency>
```

`application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/postgres
    username: postgres
    password: ${POSTGRES_PASSWORD}   # passe por variável de ambiente
  jpa:
    hibernate:
      ddl-auto: validate             # use Flyway/Liquibase para criar o esquema
    open-in-view: false
```

> Aplicação rodando **em outro container**: troque `localhost` por `host.docker.internal` (no Linux, adicione `extra_hosts: ["host.docker.internal:host-gateway"]` ao serviço da aplicação).

### IntelliJ IDEA (Ultimate) e DataGrip

1. Abra a janela **Database** (`View → Tool Windows → Database`).
2. Clique em **`+` → Data Source → PostgreSQL**.
3. Preencha **Host** `localhost`, **Port** `5432`, **Authentication** `User & Password`, **User** `postgres`, **Password** (a do `.env`) e **Database** `postgres`.
4. Se a IDE pedir, clique em **Download missing driver files**.
5. Clique em **Test Connection** e depois em **OK**.
6. Para ver todos os bancos e schemas, abra a aba **Schemas** do data source e marque **All databases** / **All schemas**.

O mesmo procedimento funciona no DBeaver e no pgAdmin (host `localhost`, porta `5432`).

## Migração da configuração anterior

A versão anterior deste ambiente (pasta `PostgreSQL/`, arquivo `docker-compose.yml`) usava:

- container `POSTGRES` e serviço `postgres-compose`;
- senha `admin` fixa no arquivo;
- dados em uma pasta do Windows (`C:/Meu_Banco_Postgres`), via bind mount.

Nenhum dado é migrado ou apagado automaticamente. Para continuar usando os dados antigos:

```powershell
# 1. Pare e remova o container antigo (os dados em C:/Meu_Banco_Postgres NÃO são apagados)
docker stop POSTGRES
docker rm POSTGRES

# 2. Configure o novo ambiente apontando para a pasta antiga
cd postgres
cp .env.example .env
#    edite o .env e defina:
#    POSTGRES_DATA_SOURCE=C:/Meu_Banco_Postgres

# 3. Suba e confira
docker compose up -d
docker compose exec postgres psql -U postgres -c "\l"
```

- A senha continua sendo **`admin`**, porque ela está gravada nos dados. Para trocá-la: `docker compose exec postgres psql -U postgres -c "ALTER USER postgres PASSWORD 'nova-senha';"`. Depois, atualize `POSTGRES_PASSWORD` no `.env`.
- Mantenha `POSTGRES_IMAGE_TAG=18`. A imagem anterior também era a versão 18 (Debian), então os dados são compatíveis.
- Para passar a usar o volume nomeado, faça um `pg_dump` com a pasta antiga, troque `POSTGRES_DATA_SOURCE` para `data`, suba o ambiente e restaure o dump. Só remova a pasta antiga depois de conferir os dados.

## 10. Problemas comuns

| Sintoma | Causa provável / solução |
|---|---|
| `password authentication failed for user "postgres"` | A senha do `.env` é diferente da gravada nos dados. Use a senha original ou altere com `ALTER USER` (seção de migração). |
| `Conflict. The container name "/postgres" is already in use` ou porta `5432` ocupada | Há outro PostgreSQL rodando: o container antigo `POSTGRES`, um PostgreSQL instalado no Windows ou outro Compose. Pare-o ou altere `POSTGRES_PORT`. |
| Banco "vazio" depois de atualizar o repositório | O ambiente passou a usar o volume `postgres_data`. Os dados antigos continuam em `C:/Meu_Banco_Postgres`: defina `POSTGRES_DATA_SOURCE` (seção de migração). |
| Container encerra logo após iniciar, citando dados em `/var/lib/postgresql/data` | A pasta montada tem dados de uma versão ≤ 17, que usava o ponto de montagem `/var/lib/postgresql/data`. Use a imagem da versão correspondente ou migre com `pg_dump`/restore. |
| `database files are incompatible with server` | Versão principal da imagem diferente da que criou os dados. Volte para a versão anterior e migre com `pg_dump`/`pg_restore` ou `pg_upgrade`. **Não** apague o volume sem backup. |
| `role "root" does not exist` ao rodar `psql` | Informe o usuário: `psql -U postgres`. |
| Erros de permissão com pasta do Windows | Prefira o volume nomeado (`POSTGRES_DATA_SOURCE=data`). Pastas do Windows montadas no WSL2 são mais lentas e podem ter problemas de permissão. |
