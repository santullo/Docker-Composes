# Redis

## 1. O que é

O [Redis](https://redis.io/) é um banco de dados **em memória** do tipo chave-valor. Ele oferece estruturas de dados nativas (strings, hashes, listas, sets, sorted sets, streams) e operações atômicas com latência abaixo de 1 ms. A partir da versão 8, a imagem oficial também inclui recursos antes distribuídos como módulos, como JSON, busca/consulta, séries temporais e filtros probabilísticos.

## 2. Para que serve

- **Cache** de consultas, respostas de APIs e resultados caros de calcular.
- Armazenamento de **sessões HTTP** (por exemplo, com Spring Session).
- **Rate limiting**, contadores e rankings (sorted sets).
- **Locks distribuídos**, filas leves (listas e streams) e pub/sub.

## 3. Quando utilizar

Use Redis quando a prioridade for **velocidade de acesso** a dados pequenos e frequentemente lidos, acessados por chave. Ele não substitui um banco relacional quando há consultas complexas, relacionamentos ou transações ACID entre muitas entidades.

### Redis como cache × como armazenamento de dados

| | **Cache** | **Armazenamento de dados** |
|---|---|---|
| Fonte da verdade | Outro banco (PostgreSQL etc.). O Redis guarda só uma cópia. | O próprio Redis |
| Perda de dados | Aceitável: o dado é recalculado ou buscado de novo | Não aceitável |
| Expiração (TTL) | Quase sempre definida | Normalmente sem TTL |
| Quando a memória acaba | Remover chaves antigas: `maxmemory-policy allkeys-lru` | Recusar escritas: `noeviction` |
| Persistência em disco | Opcional | Necessária (AOF e/ou RDB) |

Este ambiente vem configurado de forma **conservadora** (`noeviction` + AOF + RDB), o que serve aos dois casos em desenvolvimento. Para simular um cache puro, defina `REDIS_MAXMEMORY_POLICY=allkeys-lru` no `.env`.

**Persistência configurada:**

- **AOF** (`appendonly yes`, `appendfsync everysec`): registra cada escrita em log. No pior caso, perde cerca de 1 segundo de dados.
- **RDB** (`save 3600 1 300 100 60 10000`): snapshots periódicos do conjunto de dados.

Os arquivos ficam no volume `redis_data`, montado em `/data`.

## 4. Serviços inicializados

| Serviço | Imagem | Descrição |
|---|---|---|
| `redis` | `redis:8.8-alpine` (oficial) | Servidor Redis com senha, AOF e RDB |
| `redisinsight` | `redis/redisinsight:3.8.0` (oficial) | Painel web para navegar por chaves, rodar comandos e ver métricas |

## 5. Portas

| Porta (host) | Porta (container) | Uso |
|---|---|---|
| `127.0.0.1:6379` | `6379` | Protocolo Redis (RESP) |
| `127.0.0.1:5540` | `5540` | Painel RedisInsight: http://localhost:5540 |

## 6. Variáveis de ambiente

| Variável | Padrão no `.env.example` | Descrição |
|---|---|---|
| `REDIS_IMAGE_TAG` | `8.8-alpine` | Tag da imagem |
| `REDISINSIGHT_IMAGE_TAG` | `3.8.0` | Tag da imagem do RedisInsight |
| `BIND_ADDRESS` | `127.0.0.1` | Interface do host onde a porta é publicada |
| `REDIS_PORT` | `6379` | Porta no host |
| `REDISINSIGHT_PORT` | `5540` | Porta do painel RedisInsight no host |
| `REDIS_PASSWORD` | `troque-esta-senha` | Senha do usuário `default` (**obrigatória**) |
| `REDIS_MAXMEMORY` | `256mb` | Limite de memória do Redis |
| `REDIS_MAXMEMORY_POLICY` | `noeviction` | Política quando o limite é atingido |

A senha é passada ao `redis-server` pela linha de comando a partir de uma variável de ambiente, então não aparece em nenhum arquivo versionado. Quem tem acesso ao Docker do host consegue vê-la com `docker inspect`, o que é aceitável para desenvolvimento local. O container também define `REDISCLI_AUTH`, então o `redis-cli` executado **dentro** dele já autentica sem precisar digitar a senha.

## 7. Iniciar e interromper

```bash
cd redis
cp .env.example .env        # edite a senha
docker compose up -d
docker compose ps           # aguarde "healthy"
docker compose logs -f
docker compose stop
docker compose down         # mantém os dados no volume
docker compose down -v      # APAGA os dados
```

## 8. Verificar o funcionamento

```bash
docker compose exec redis redis-cli ping            # PONG
docker compose exec redis redis-cli INFO server     # versão e uptime
docker compose exec redis redis-cli INFO persistence
```

### Comandos básicos

Abra um terminal interativo com `docker compose exec redis redis-cli` e experimente:

```
SET usuario:1:nome "Maria"           # grava uma string
GET usuario:1:nome                   # lê
SET sessao:abc "dados" EX 60         # grava com expiração de 60 s
TTL sessao:abc                       # segundos restantes
INCR contador:visitas                # incremento atômico
HSET produto:10 nome "Teclado" preco 199.90
HGETALL produto:10
LPUSH fila:emails "a@ex.com" "b@ex.com"
RPOP fila:emails
ZADD ranking 100 "ana" 80 "bruno"
ZREVRANGE ranking 0 -1 WITHSCORES
KEYS usuario:*                       # só em desenvolvimento! Em produção use SCAN
SCAN 0 MATCH usuario:* COUNT 100
DEL usuario:1:nome
DBSIZE
FLUSHDB                              # APAGA todas as chaves do banco atual
```

Pelo host, se você tiver o `redis-cli` instalado:

```bash
redis-cli -h 127.0.0.1 -p 6379 -a troque-esta-senha ping
```

### Painel RedisInsight

Abra http://localhost:5540. A conexão **`redis-local`** já aparece cadastrada, com a senha do `.env`, então basta clicar nela. No painel você pode:

- navegar pelas chaves, ver e editar valores, TTL e tipo (**Browser**);
- rodar comandos com autocompletar (**Workbench** e **CLI**);
- acompanhar memória, clientes conectados e comandos lentos (**Analysis Tools** e **Slow Log**).

Dentro do Docker, o painel acessa o Redis pelo nome do serviço (`redis:6379`), e não por `localhost`. Para cadastrar outro Redis manualmente, use esse mesmo endereço se ele estiver na mesma rede do Compose. As preferências do painel ficam no volume `redis_insight-data`.

> Se você trocar `REDIS_PASSWORD` no `.env`, recrie o painel junto com o Redis: `docker compose up -d --force-recreate`.

Também dá para usar o suporte a Redis do IntelliJ IDEA Ultimate / DataGrip: conecte em `localhost:6379`, usuário `default` e a senha do `.env`.

## 9. Conectar uma aplicação

**URI:** `redis://default:troque-esta-senha@localhost:6379/0`

### Spring Boot com Spring Data Redis

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!-- Para usar Redis como cache com @Cacheable -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

`application.yml`:

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD}     # passe por variável de ambiente
      database: 0
      timeout: 2s
  cache:
    type: redis
    redis:
      time-to-live: 10m               # TTL padrão das entradas de cache
```

Acesso direto (armazenamento de dados):

```java
@Service
class ContadorService {
    private final StringRedisTemplate redis;
    ContadorService(StringRedisTemplate redis) { this.redis = redis; }

    long registrarVisita(String pagina) {
        return redis.opsForValue().increment("visitas:" + pagina);
    }
}
```

Uso como cache:

```java
@SpringBootApplication
@EnableCaching
public class Application { /* ... */ }

@Service
class ProdutoService {
    @Cacheable(cacheNames = "produtos", key = "#id")
    public ProdutoDto buscar(Long id) {
        // só executa quando a chave não está no cache
        return repository.findById(id).map(ProdutoDto::from).orElseThrow();
    }

    @CacheEvict(cacheNames = "produtos", key = "#id")
    public void atualizar(Long id, ProdutoDto dto) { /* ... */ }
}
```

> O serializador padrão do cache é o do JDK, então o DTO precisa implementar `Serializable`. Para gravar em JSON, configure um `RedisCacheConfiguration` com um serializador JSON do Spring Data Redis (`GenericJackson2JsonRedisSerializer` no Spring Boot 3; no Boot 4, a variante baseada em Jackson 3).

## 10. Problemas comuns

| Sintoma | Causa provável / solução |
|---|---|
| `NOAUTH Authentication required` | O cliente não enviou a senha. Configure `spring.data.redis.password` ou use `redis-cli -a`. |
| `WRONGPASS invalid username-password pair` | A senha está errada. Confira o `.env`. Se você alterou o `.env`, recrie o container: `docker compose up -d --force-recreate`. |
| `Defina REDIS_PASSWORD no arquivo .env` | O arquivo `.env` não existe. Rode `cp .env.example .env`. |
| `OOM command not allowed when used memory > 'maxmemory'` | O limite foi atingido com `noeviction`. Aumente `REDIS_MAXMEMORY`, apague chaves ou use `allkeys-lru` para cache. |
| `port is already allocated` | Já existe um Redis na porta 6379 (ou outro serviço na 5540). Altere `REDIS_PORT` ou `REDISINSIGHT_PORT`. |
| Painel mostra a conexão `redis-local` com erro de autenticação | A senha mudou no `.env`, mas o painel não foi recriado. Rode `docker compose up -d --force-recreate`. |
| Aviso `vm.overcommit_memory` nos logs | É apenas um aviso em ambiente local. Em Linux, pode ser resolvido com `sudo sysctl vm.overcommit_memory=1`. |
