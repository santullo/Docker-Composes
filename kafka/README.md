# Apache Kafka

## 1. O que é

O [Apache Kafka](https://kafka.apache.org/) é uma plataforma distribuída de **event streaming**. Ele funciona como um **log de eventos** particionado, replicado e persistente. Produtores gravam eventos em **tópicos**, e consumidores leem esses eventos no próprio ritmo, controlando a posição de leitura (**offset**). Diferente de uma fila tradicional, uma mensagem **não é removida ao ser consumida**: ela fica disponível até expirar pela política de retenção.

| Conceito | Descrição |
|---|---|
| **Tópico** | Canal nomeado de eventos (exemplo: `pedidos`) |
| **Partição** | Divisão de um tópico. A ordem é garantida **dentro** de cada partição. Mensagens com a mesma *key* vão para a mesma partição. |
| **Offset** | Posição de uma mensagem na partição |
| **Consumer group** | Grupo de consumidores que divide as partições entre si. Cada grupo recebe **todas** as mensagens. |
| **Broker** | Servidor Kafka. Aqui há apenas um. |
| **KRaft** | Modo em que o próprio Kafka gerencia os metadados do cluster, sem ZooKeeper. É o único modo a partir do Kafka 4.0. |

## 2. Para que serve

- Comunicação assíncrona orientada a eventos entre microsserviços.
- *Event sourcing*, CDC (*change data capture*) e integração entre sistemas.
- Processamento de fluxos em tempo real (Kafka Streams, Flink).
- Reprocessar eventos antigos ao "voltar" o offset de um consumer group.

## 3. Quando utilizar

Use Kafka quando precisar de **alto volume**, **retenção de eventos**, **releitura** por vários consumidores independentes ou garantia de ordem por chave. Para filas de tarefas com roteamento flexível e confirmação mensagem a mensagem, o [RabbitMQ](../rabbitmq) costuma ser mais simples.

## 4. Serviços inicializados

| Serviço | Imagem | Descrição |
|---|---|---|
| `kafka` | `confluentinc/cp-kafka:8.3.2` (Kafka 4.3) | Broker único em modo **KRaft** (broker + controller no mesmo nó) |
| `kafka-ui` | `kafbat/kafka-ui:v1.5.0` | Interface web para tópicos, mensagens, consumer groups e brokers |

A UI só inicia depois que o broker fica `healthy`. Os dados do Kafka (tópicos, mensagens, offsets e metadados do KRaft) ficam no volume `kafka_data`.

### Listeners: como o Kafka é acessado de dentro e de fora do Docker

Um cliente Kafka faz duas etapas: conecta no *bootstrap server* e depois usa o **endereço anunciado** (`advertised.listeners`) que o broker devolve. Por isso, existe um listener para cada "lado":

```
Seu PC (host)                         Rede Docker (kafka_default)
 Spring Boot ─► localhost:9092 ──┐     kafka-ui ─► kafka:29092
                                 ▼                     │
                     ┌───────────────────────────────────────┐
                     │ kafka                                 │
                     │  PLAINTEXT          :9092  → anuncia localhost:9092
                     │  PLAINTEXT_INTERNAL :29092 → anuncia kafka:29092
                     │  CONTROLLER         :9093  (KRaft, uso interno)
                     └───────────────────────────────────────┘
```

Se o endereço anunciado estiver errado, o cliente conecta no bootstrap, mas falha em seguida (veja "Problemas comuns").

## 5. Portas

| Porta (host) | Porta (container) | Uso |
|---|---|---|
| `127.0.0.1:9092` | `9092` | Kafka para aplicações no host |
| — | `29092` | Kafka para outros containers (rede Docker) |
| — | `9093` | Controller KRaft (interno) |
| `127.0.0.1:19000` | `8080` | Kafka UI (<http://localhost:19000>) |

## 6. Variáveis de ambiente

| Variável | Padrão no `.env.example` | Descrição |
|---|---|---|
| `KAFKA_IMAGE_TAG` | `8.3.2` | Tag da imagem `confluentinc/cp-kafka` |
| `KAFKA_UI_IMAGE_TAG` | `v1.5.0` | Tag da imagem `kafbat/kafka-ui` |
| `BIND_ADDRESS` | `127.0.0.1` | Interface do host onde as portas são publicadas |
| `KAFKA_PORT` | `9092` | Porta do Kafka no host. Também é usada no endereço anunciado `localhost:<porta>`. |
| `KAFKA_UI_PORT` | `19000` | Porta da Kafka UI no host |

Este ambiente não usa autenticação (protocolo `PLAINTEXT`) e é destinado apenas a uso local.

## 7. Iniciar e interromper

```bash
cd kafka
cp .env.example .env
docker compose up -d
docker compose ps            # aguarde "healthy" nos dois serviços
docker compose logs -f kafka
docker compose stop
docker compose down          # mantém tópicos e mensagens
docker compose down -v       # APAGA tópicos, mensagens e offsets
```

## 8. Verificar o funcionamento e operações básicas

```bash
docker compose ps
docker compose exec kafka kafka-broker-api-versions --bootstrap-server kafka:29092 | head -1
```

Interface web: <http://localhost:19000>. As estatísticas da tela inicial (como a contagem de tópicos) são atualizadas periodicamente e podem levar até ~30 s para refletir mudanças.

Os comandos abaixo rodam **dentro** do container. Na imagem da Confluent, as ferramentas não têm o sufixo `.sh`.

```bash
# Criar tópico
docker compose exec kafka kafka-topics --bootstrap-server kafka:29092 \
  --create --topic pedidos --partitions 3 --replication-factor 1

# Listar e descrever tópicos
docker compose exec kafka kafka-topics --bootstrap-server kafka:29092 --list
docker compose exec kafka kafka-topics --bootstrap-server kafka:29092 --describe --topic pedidos

# Produzir mensagens (digite uma por linha; Ctrl+C para sair)
docker compose exec kafka kafka-console-producer --bootstrap-server kafka:29092 --topic pedidos

# Produzir mensagens com chave (formato chave:valor)
docker compose exec kafka kafka-console-producer --bootstrap-server kafka:29092 --topic pedidos \
  --reader-property parse.key=true --reader-property key.separator=:

# Consumir desde o início (Ctrl+C para sair)
docker compose exec kafka kafka-console-consumer --bootstrap-server kafka:29092 --topic pedidos \
  --from-beginning --formatter-property print.key=true --group grupo-teste

# Consumer groups: offsets e lag
docker compose exec kafka kafka-consumer-groups --bootstrap-server kafka:29092 --list
docker compose exec kafka kafka-consumer-groups --bootstrap-server kafka:29092 --describe --group grupo-teste

# Voltar os offsets de um grupo para o início (o grupo precisa estar sem consumidores ativos)
docker compose exec kafka kafka-consumer-groups --bootstrap-server kafka:29092 --group grupo-teste \
  --topic pedidos --reset-offsets --to-earliest --execute

# Aumentar partições (não é possível diminuir)
docker compose exec kafka kafka-topics --bootstrap-server kafka:29092 --alter --topic pedidos --partitions 6

# Excluir tópico
docker compose exec kafka kafka-topics --bootstrap-server kafka:29092 --delete --topic pedidos
```

> No PowerShell, troque as `\` do final das linhas por crase (`` ` ``) ou escreva o comando em uma linha só.

Na Kafka UI, as mesmas operações estão em **Topics → Add a Topic**, **Topics → \<tópico\> → Produce Message / Messages** e **Consumers**.

> Por padrão, o broker cria tópicos automaticamente na primeira publicação (`auto.create.topics.enable=true`), com 1 partição. Para controlar partições, crie o tópico antes.

## 9. Conectar uma aplicação

| Onde a aplicação roda | `bootstrap-servers` |
|---|---|
| No host (IDE, `mvn spring-boot:run`) | `localhost:9092` |
| Em um container **neste** Compose | `kafka:29092` |
| Em um container de **outro** Compose | Conecte-o à rede `kafka_default` e use `kafka:29092` |

Exemplo para conectar um serviço de outro Compose à rede deste ambiente:

```yaml
services:
  minha-api:
    networks: [kafka]
networks:
  kafka:
    name: kafka_default
    external: true
```

### Spring Boot (Spring for Apache Kafka)

```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>
```

`application.yml`:

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: minha-api
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
```

```java
@Configuration
class KafkaTopics {
    // Criado automaticamente na inicialização, se não existir
    @Bean NewTopic pedidos() { return TopicBuilder.name("pedidos").partitions(3).replicas(1).build(); }
}

@Service
class PedidoProducer {
    private final KafkaTemplate<String, String> kafka;
    PedidoProducer(KafkaTemplate<String, String> kafka) { this.kafka = kafka; }

    void publicar(String pedidoId, String json) {
        kafka.send("pedidos", pedidoId, json);   // a key define a partição
    }
}

@Component
class PedidoConsumer {
    @KafkaListener(topics = "pedidos")
    void consumir(ConsumerRecord<String, String> record) {
        System.out.printf("partição=%d offset=%d key=%s valor=%s%n",
            record.partition(), record.offset(), record.key(), record.value());
    }
}
```

## Mudanças em relação à configuração anterior

- **Imagens com versão fixa:** `cp-kafka:latest` passou para `cp-kafka:8.3.2`, e `provectuslabs/kafka-ui:latest` foi trocado por `kafbat/kafka-ui:v1.5.0`. O projeto da Provectus não recebe atualizações desde 2024. O Kafbat UI é a continuação mantida dele e usa as mesmas variáveis de configuração.
- **Persistência:** a configuração anterior não tinha volume, então os tópicos eram perdidos a cada `docker compose down`. Agora os dados ficam no volume `kafka_data`.
- **Healthchecks:** os dois serviços têm healthcheck, e a UI aguarda o broker ficar saudável.
- **Portas:** publicadas em `127.0.0.1` e configuráveis. A porta do host é usada também no endereço anunciado.
- **Containers:** renomeados para minúsculas (`kafka`, `kafka-ui`). Antes de subir a nova versão, remova os containers antigos: `docker rm -f KAFKA KAFKA-UI`.
- **Sem migração de ZooKeeper:** a configuração anterior já usava KRaft, então não havia nada a migrar. O `CLUSTER_ID` foi mantido.
- **README:** a versão anterior descrevia o Kafdrop, mas o Compose já usava o Kafka UI. A documentação agora corresponde ao que está configurado.

## 10. Problemas comuns

| Sintoma | Causa provável / solução |
|---|---|
| A aplicação conecta, mas fica em `Connection to node 1 (kafka/...:29092) could not be established` | A aplicação está no host e recebeu o endereço interno. Use `localhost:9092` (listener `PLAINTEXT`), não `29092`. |
| Container da aplicação não alcança `localhost:9092` | Dentro de um container, `localhost` é o próprio container. Use `kafka:29092` na mesma rede (seção 9). |
| Mudei `KAFKA_PORT` e nada conecta | Recrie o broker para anunciar a nova porta: `docker compose up -d --force-recreate kafka`. |
| `Invalid cluster.id in: /var/lib/kafka/data/meta.properties` | O `CLUSTER_ID` foi alterado com dados existentes. Restaure o valor original ou apague os dados com `docker compose down -v`. |
| Kafka UI mostra o cluster `offline` | O broker ainda está iniciando ou foi reiniciado. Confira com `docker compose ps` e `docker compose logs kafka`. |
| `Conflict. The container name "/kafka" is already in use` / porta 9092 ocupada | Há um Kafka antigo rodando. Pare-o (`docker rm -f KAFKA KAFKA-UI`) ou altere `KAFKA_PORT`. |
| Acesso de outra máquina com `BIND_ADDRESS=0.0.0.0` não funciona | O broker anuncia `localhost`. Para acesso pela rede, troque `localhost` em `KAFKA_ADVERTISED_LISTENERS` pelo IP ou nome da máquina. |
