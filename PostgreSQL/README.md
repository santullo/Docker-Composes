# PostgreSQL com Docker Compose

Projeto simples para subir um container PostgreSQL utilizando Docker Compose.

---

# Tecnologias utilizadas

- Docker
- Docker Compose
- PostgreSQL 18

---

# Como executar o projeto

## 1. Subir o container

```bash
docker-compose up -d
```

---

## 2. Verificar se o container está rodando

```bash
docker ps
```

---

## 3. Acessar o PostgreSQL

Host:

```text
localhost
```

Porta:

```text
5432
```

Usuário padrão:

```text
postgres
```

Senha:

```text
admin
```

---

# Persistência dos dados

Os dados do PostgreSQL ficarão armazenados localmente em:

```text
C:/Meu_Banco_Postgres
```

Isso garante que os dados não sejam perdidos ao remover o container.

---

# Comandos úteis

## Parar o container

```bash
docker-compose stop
```

---

## Iniciar novamente

```bash
docker-compose start
```

---

## Derrubar o ambiente

```bash
docker-compose down
```

---

## Derrubar removendo volumes

```bash
docker-compose down -v
```

---

# Observações

- A porta `5432` é a porta padrão do PostgreSQL.
- O container pode ser acessado por ferramentas como:
    - DBeaver
    - pgAdmin
    - IntelliJ
    - DataGrip

---

# Estrutura do projeto

```text
.
├── docker-compose.yml
└── README.md
```