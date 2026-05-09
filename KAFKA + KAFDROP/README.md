# Kafka + Kafdrop com Docker Compose

Projeto simples para estudos utilizando:

- Apache Kafka
- KRaft Mode (sem Zookeeper)
- Kafdrop
- Docker Compose

---

# Tecnologias utilizadas

- Docker
- Docker Compose
- Apache Kafka
- Kafdrop
- Confluent Platform

---

# Arquitetura do projeto

```text
Seu PC
 ├── localhost:9092  -> Kafka
 └── localhost:19000 -> Kafdrop

Docker
 ├── Kafka
 └── Kafdrop
```

---

# O que é o Kafka?

O Apache Kafka é uma plataforma distribuída de event streaming utilizada para:

- comunicação assíncrona
- troca de mensagens
- processamento de eventos em tempo real
- integração entre sistemas

---

# O que é o Kafdrop?

O Kafdrop é uma interface web para visualizar:

- tópicos
- mensagens
- brokers
- consumers
- partitions

---

# Como executar o projeto

## 1. Subir os containers

```bash
docker-compose up -d
```

---

## 2. Verificar containers ativos

```bash
docker ps
```

---

# Acessos

## Kafka

```text
localhost:9092
```

---

## Kafdrop

```text
http://localhost:19000
```

---

# Comandos úteis

## Ver logs do Kafka

```bash
docker logs KAFKA
```

---

## Ver logs do Kafdrop

```bash
docker logs KAFDROP
```

---

## Parar os containers

```bash
docker-compose stop
```

---

## Iniciar novamente

```bash
docker-compose start
```

---

## Derrubar ambiente

```bash
docker-compose down
```

---

## Derrubar removendo volumes

```bash
docker-compose down -v
```

---

# Explicação das portas

| Porta | Uso |
|---|---|
| 9092 | Acesso externo ao Kafka |
| 29092 | Comunicação interna Docker |
| 9093 | Controller KRaft |
| 19000 | Interface web do Kafdrop |

---

# O que é KRaft?

KRaft é o novo modo do Kafka que substitui o uso do Zookeeper.

Agora o próprio Kafka gerencia:
- metadata
- controllers
- coordenação do cluster

---

# Estrutura do projeto

```text
.
├── docker-compose.yml
└── README.md
```

---

# Observações

Este projeto é destinado para:
- estudos
- desenvolvimento local
- testes

Não é recomendado para produção sem:
- múltiplos brokers
- replicação adequada
- autenticação
- monitoramento
- persistência robusta