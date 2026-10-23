# RabbitMQ

## 1. O que é

O [RabbitMQ](https://www.rabbitmq.com/) é um *message broker* (intermediário de mensagens) open source. Ele implementa o protocolo **AMQP 0-9-1** e também oferece suporte a AMQP 1.0, MQTT e STOMP (por plugins). Aplicações **produtoras** publicam mensagens em **exchanges**, e o broker as encaminha para **filas** de acordo com regras de **binding**. Aplicações **consumidoras** leem as mensagens dessas filas.

```
Produtor ──► Exchange ──(binding / routing key)──► Fila ──► Consumidor
```

| Conceito | Descrição |
|---|---|
| **Exchange** | Recebe as mensagens e decide para quais filas encaminhar. Tipos: `direct` (routing key exata), `topic` (padrões com `*` e `#`), `fanout` (todas as filas ligadas) e `headers`. |
| **Queue** | Armazena as mensagens até serem consumidas. |
| **Binding** | Regra que liga uma exchange a uma fila (opcionalmente com routing key). |
| **Routing key** | "Endereço" informado na publicação e usado pela exchange para rotear. |
| **Virtual host (vhost)** | Isolamento lógico de exchanges, filas e permissões dentro do mesmo broker. |

## 2. Para que serve

- Comunicação assíncrona entre serviços (desacoplamento).
- Filas de trabalho (*work queues*) para distribuir tarefas entre vários consumidores.
- Publicação de eventos para vários interessados (*pub/sub*).
- Absorver picos de carga e permitir novas tentativas (*retry*) com *dead letter exchanges*.

## 3. Quando utilizar

Use RabbitMQ quando precisar de **roteamento flexível**, **entrega por mensagem com confirmação (ack)** e filas de tarefas. Para **streaming de eventos** com retenção longa, releitura do histórico e alto volume, o Kafka costuma ser mais adequado (ver [`../kafka`](../kafka)).

## 4. Serviços inicializados

| Serviço | Imagem | Descrição |
|---|---|---|
| `rabbitmq` | `rabbitmq:4.3-management` (oficial) | Broker RabbitMQ com o **Management Plugin** (painel web + API HTTP) |

O `hostname` do container é fixo (`rabbitmq`). Isso é importante: o RabbitMQ grava os dados em um diretório com o nome do nó (`rabbit@<hostname>`). Sem um hostname fixo, cada novo container teria um nome diferente e "perderia" os dados do volume.

## 5. Portas

| Porta (host) | Porta (container) | Uso |
|---|---|---|
| `127.0.0.1:5672` | `5672` | AMQP (aplicações) |
| `127.0.0.1:15672` | `15672` | Painel de gerenciamento e API HTTP |

## 6. Variáveis de ambiente

| Variável | Padrão no `.env.example` | Descrição |
|---|---|---|
| `RABBITMQ_IMAGE_TAG` | `4.3-management` | Tag da imagem |
| `BIND_ADDRESS` | `127.0.0.1` | Interface do host onde as portas são publicadas |
| `RABBITMQ_AMQP_PORT` | `5672` | Porta AMQP no host |
| `RABBITMQ_MANAGEMENT_PORT` | `15672` | Porta do painel no host |
| `RABBITMQ_DEFAULT_USER` | `admin` | Usuário administrador (**obrigatória**) |
| `RABBITMQ_DEFAULT_PASS` | `troque-esta-senha` | Senha do administrador (**obrigatória**) |
| `RABBITMQ_DEFAULT_VHOST` | `/` | Virtual host padrão |

> **Atenção:** o usuário e a senha só são aplicados na **primeira** inicialização, quando o volume está vazio. Para trocar a senha depois, use o painel ou `rabbitmqctl change_password` (ver "Problemas comuns").

## 7. Iniciar e interromper

```bash
cd rabbitmq
cp .env.example .env        # edite a senha
docker compose up -d        # inicia em segundo plano
docker compose ps           # status (aguarde "healthy")
docker compose logs -f      # acompanha os logs (Ctrl+C para sair)
docker compose stop         # para sem remover os containers
docker compose down         # remove containers e rede (os dados ficam no volume)
docker compose down -v      # remove também o volume (APAGA filas e mensagens)
```

## 8. Verificar o funcionamento

```bash
# Status do healthcheck
docker compose ps

# Diagnóstico do nó
docker compose exec rabbitmq rabbitmq-diagnostics status
docker compose exec rabbitmq rabbitmq-diagnostics check_port_connectivity
```

Painel web: <http://localhost:15672>, com o usuário e a senha definidos no `.env`.

API HTTP:

```bash
curl -u admin:troque-esta-senha http://localhost:15672/api/overview
```

### Exemplo: exchange, fila, binding e publicação

A imagem inclui o `rabbitmqadmin` (v2), que usa a API HTTP do painel. Os comandos abaixo rodam dentro do container. Substitua a senha pela definida no `.env`.

```bash
# Linux/macOS (bash). No PowerShell, troque as "\" finais por crase (`).
docker compose exec rabbitmq rabbitmqadmin --username admin --password troque-esta-senha \
  declare exchange --name demo.exchange --type direct --durable true

docker compose exec rabbitmq rabbitmqadmin --username admin --password troque-esta-senha \
  declare queue --name demo.queue --durable true

docker compose exec rabbitmq rabbitmqadmin --username admin --password troque-esta-senha \
  declare binding --source demo.exchange --destination-type queue \
  --destination demo.queue --routing-key demo.key

docker compose exec rabbitmq rabbitmqadmin --username admin --password troque-esta-senha \
  publish message --exchange demo.exchange --routing-key demo.key --payload "Olá, RabbitMQ"

# Lê uma mensagem e a devolve para a fila
docker compose exec rabbitmq rabbitmqadmin --username admin --password troque-esta-senha \
  get messages --queue demo.queue --count 1 --ack-mode ack_requeue_true

# Lista as filas
docker compose exec rabbitmq rabbitmqadmin --username admin --password troque-esta-senha list queues
```

O mesmo pode ser feito no painel web: **Exchanges → Add a new exchange**, **Queues and Streams → Add a new queue**, depois abra a exchange e use **Bindings** e **Publish message**.

## 9. Conectar uma aplicação

**URI AMQP:** `amqp://admin:troque-esta-senha@localhost:5672/`

Para aplicações em **outro Compose/container**, `localhost` aponta para o próprio container. Use `host.docker.internal` (Docker Desktop; no Linux, adicione `extra_hosts: ["host.docker.internal:host-gateway"]`) ou coloque os serviços na mesma rede Docker.

### Spring Boot

Dependência (Maven):

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

`application.yml`:

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: ${RABBITMQ_PASSWORD}   # passe por variável de ambiente
    virtual-host: /
```

Declaração da topologia, publicação e consumo:

```java
@Configuration
class RabbitConfig {
    static final String EXCHANGE = "demo.exchange";
    static final String QUEUE = "demo.queue";
    static final String ROUTING_KEY = "demo.key";

    @Bean DirectExchange demoExchange() { return new DirectExchange(EXCHANGE); }
    @Bean Queue demoQueue() { return QueueBuilder.durable(QUEUE).build(); }
    @Bean Binding demoBinding(Queue demoQueue, DirectExchange demoExchange) {
        return BindingBuilder.bind(demoQueue).to(demoExchange).with(ROUTING_KEY);
    }
}

@Service
class Producer {
    private final RabbitTemplate rabbitTemplate;
    Producer(RabbitTemplate rabbitTemplate) { this.rabbitTemplate = rabbitTemplate; }

    void send(String message) {
        rabbitTemplate.convertAndSend(RabbitConfig.EXCHANGE, RabbitConfig.ROUTING_KEY, message);
    }
}

@Component
class Consumer {
    @RabbitListener(queues = RabbitConfig.QUEUE)
    void receive(String message) {
        System.out.println("Recebido: " + message);
    }
}
```

O Spring declara automaticamente no broker os beans `Exchange`, `Queue` e `Binding` na primeira conexão.

## 10. Problemas comuns

| Sintoma | Causa provável / solução |
|---|---|
| `ACCESS_REFUSED - Login was refused` | Senha diferente da gravada no volume. Os valores do `.env` só valem na 1ª inicialização. Troque com `docker compose exec rabbitmq rabbitmqctl change_password admin NOVA_SENHA` ou recrie o volume com `docker compose down -v`. |
| Login `guest/guest` não funciona | O usuário `guest` só é criado quando `RABBITMQ_DEFAULT_USER` não é definido e só aceita conexões de `localhost` **dentro** do container. Use o usuário do `.env`. |
| Filas e mensagens somem após recriar o container | Faltava hostname fixo ou o volume foi removido (`down -v`). Mensagens publicadas sem `delivery_mode=2` (persistentes) ou em filas não duráveis não sobrevivem a reinícios. |
| `port is already allocated` | Outra instância usa a porta. Altere `RABBITMQ_AMQP_PORT` / `RABBITMQ_MANAGEMENT_PORT` no `.env`. |
| Container fica em `starting` | O RabbitMQ leva ~20 a 30 s para subir. Acompanhe com `docker compose logs -f`. |
| Alarme de disco ou memória (publicações bloqueadas) | Pouco espaço em disco ou memória no Docker. Libere espaço ou aumente os recursos do Docker Desktop. |
