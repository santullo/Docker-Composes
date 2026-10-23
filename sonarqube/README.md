# SonarQube

## 1. O que é

O [SonarQube](https://www.sonarsource.com/products/sonarqube/) é uma plataforma de **análise estática de código**. Ele inspeciona o código-fonte sem executá-lo e aponta:

- **Bugs**: código que provavelmente se comporta de forma errada.
- **Vulnerabilidades** e **Security Hotspots**: trechos sensíveis à segurança que precisam de revisão.
- **Code smells**: problemas de manutenibilidade.
- **Duplicação** e **cobertura de testes** (importada de relatórios como o do JaCoCo).

Este ambiente usa o **SonarQube Community Build**, a edição gratuita e open source (`sonarqube:<versão>-community`). Ela tem algumas limitações em relação às edições pagas: analisa apenas a branch principal (sem análise de branches e PRs) e não inclui algumas linguagens e regras de segurança avançadas.

## 2. Para que serve

- Medir a qualidade do código antes de abrir um PR ou durante o estudo de boas práticas.
- Aplicar um **Quality Gate**: critérios mínimos, como zero bugs novos e cobertura ≥ 80% no código novo.
- Conectar a IDE (SonarQube for IDE) para usar as mesmas regras do servidor.

## 3. Quando utilizar

Use quando quiser um **histórico** de métricas por projeto, um painel central de issues ou testar a integração do SonarQube com Maven/CI. Se precisar só de feedback enquanto escreve o código, o plugin **SonarQube for IDE** funciona sozinho, sem servidor.

## 4. Serviços inicializados

| Serviço | Imagem | Descrição |
|---|---|---|
| `sonarqube` | `sonarqube:26.10.0.133078-community` (oficial) | Servidor web, Compute Engine e Elasticsearch embutido |
| `sonarqube-db` | `postgres:17-alpine` (oficial) | PostgreSQL **dedicado** ao SonarQube |

- O SonarQube não suporta mais o banco H2 embutido em uso real. Por isso, a configuração inclui um PostgreSQL próprio.
- O `sonarqube-db` **não publica portas** no host e fica só na rede interna deste Compose (`sonarqube_default`). Ele não conflita com o ambiente [`../postgres`](../postgres) e não compartilha dados com ele.
- O `sonarqube` só inicia depois que o banco fica `healthy` (`depends_on: condition: service_healthy`).
- O PostgreSQL 17 foi escolhido por estar dentro da faixa suportada pelas versões atuais do SonarQube. Antes de mudar a versão principal do banco, consulte a [documentação de requisitos](https://docs.sonarsource.com/sonarqube-community-build/server-installation/server-host-requirements/).

**Volumes:**

| Volume | Montagem | Conteúdo |
|---|---|---|
| `sonarqube_data` | `/opt/sonarqube/data` | Índices do Elasticsearch |
| `sonarqube_extensions` | `/opt/sonarqube/extensions` | Plugins instalados |
| `sonarqube_logs` | `/opt/sonarqube/logs` | Logs |
| `sonarqube_db-data` | `/var/lib/postgresql/data` | Banco de dados (projetos, issues, usuários, tokens) |

## 5. Portas

| Porta (host) | Porta (container) | Uso |
|---|---|---|
| `127.0.0.1:9000` | `9000` | Interface web e API |

## 6. Variáveis de ambiente

| Variável | Padrão no `.env.example` | Descrição |
|---|---|---|
| `SONARQUBE_IMAGE_TAG` | `26.10.0.133078-community` | Tag da imagem do SonarQube |
| `BIND_ADDRESS` | `127.0.0.1` | Interface do host onde a porta é publicada |
| `SONARQUBE_PORT` | `9000` | Porta da interface no host |
| `SONAR_DB_IMAGE_TAG` | `17-alpine` | Tag do PostgreSQL dedicado |
| `SONAR_DB_NAME` | `sonar` | Nome do banco |
| `SONAR_DB_USER` | `sonar` | Usuário do banco |
| `SONAR_DB_PASSWORD` | `troque-esta-senha` | Senha do banco (**obrigatória**) |

## 7. Requisitos do host

O SonarQube usa um Elasticsearch embutido, que valida alguns parâmetros do kernel na inicialização. Se eles não estiverem corretos, o container reinicia em loop.

| Requisito | Valor mínimo | Onde configurar |
|---|---|---|
| Memória disponível para o Docker | **4 GB** (recomendado; o SonarQube sozinho usa ~2 GB) | Docker Desktop → Settings → Resources |
| `vm.max_map_count` | `524288` | Kernel do host (Linux) ou da VM do WSL2 |
| `fs.file-max` | `131072` | Kernel (o padrão das distribuições atuais já atende) |
| `ulimit nofile` / `nproc` | `131072` / `8192` | Já definidos no `compose.yaml` (`ulimits`) |

O SonarQube oficial roda em processadores x86_64 e ARM64 (incluindo Apple Silicon).

### Linux (Docker Engine)

```bash
# Verificar
sysctl vm.max_map_count fs.file-max

# Ajustar até o próximo reboot
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072

# Tornar permanente
echo "vm.max_map_count=524288" | sudo tee /etc/sysctl.d/99-sonarqube.conf
echo "fs.file-max=131072"      | sudo tee -a /etc/sysctl.d/99-sonarqube.conf
sudo sysctl --system
```

### Windows (Docker Desktop com WSL2)

Opção permanente: crie ou edite o arquivo `%UserProfile%\.wslconfig`:

```ini
[wsl2]
kernelCommandLine = "sysctl.vm.max_map_count=524288"
```

Depois, no PowerShell, reinicie o WSL e o Docker Desktop:

```powershell
wsl --shutdown
```

Opção temporária (vale até o próximo reinício do WSL):

```powershell
wsl -d docker-desktop sh -c "sysctl -w vm.max_map_count=524288"
```

## 8. Iniciar e interromper

```bash
cd sonarqube
cp .env.example .env        # edite SONAR_DB_PASSWORD
docker compose up -d
docker compose ps           # aguarde "healthy" (1 a 3 minutos na primeira vez)
docker compose logs -f sonarqube
docker compose stop
docker compose down         # mantém dados
docker compose down -v      # APAGA projetos, histórico, usuários e tokens
```

## 9. Verificar o funcionamento

```bash
docker compose ps
curl http://localhost:9000/api/system/status
# {"id":"...","version":"26.10.0.133078","status":"UP"}
```

Os estados possíveis são `STARTING`, `DB_MIGRATION_NEEDED`, `DB_MIGRATION_RUNNING` e `UP`.

Abra <http://localhost:9000> e entre com **admin / admin**. No primeiro acesso, o SonarQube obriga a troca da senha.

## 10. Criar projetos e tokens

### Criar um projeto

1. **Projects → Create project → Local project**.
2. Informe o **Project display name** e o **Project key** (exemplo: `minha-api`). Use a mesma chave em `sonar.projectKey`.
3. Defina a branch principal (`main`) e escolha a regra de "código novo". Para estudos, use **Previous version**.
4. Em **Analysis method**, escolha **Locally**. O assistente gera um token e mostra o comando Maven pronto.

### Tipos de token

Os tokens são gerados em **My Account → Security** (ícone do usuário, canto superior direito).

| Tipo | Uso |
|---|---|
| **Project Analysis Token** | Analisar **um** projeto específico. É o mais restrito e o recomendado para CI e uso local. |
| **Global Analysis Token** | Analisar qualquer projeto. |
| **User Token** | Chamadas à API com as permissões do usuário. É o tipo usado pelo **SonarQube for IDE**. |

O token só é exibido uma vez. Guarde-o em uma variável de ambiente e **nunca** o versione:

```bash
# Linux/macOS
export SONAR_TOKEN=sqp_xxxxxxxxxxxxxxxxx
```

```powershell
# Windows PowerShell
$env:SONAR_TOKEN = "sqp_xxxxxxxxxxxxxxxxx"
```

Também é possível criar projetos e tokens pela API:

```bash
curl -u admin:SUA_SENHA -X POST "http://localhost:9000/api/projects/create?project=minha-api&name=minha-api"
curl -u admin:SUA_SENHA -X POST "http://localhost:9000/api/user_tokens/generate?name=minha-api-token&type=PROJECT_ANALYSIS_TOKEN&projectKey=minha-api"
```

## 11. Integração com Maven e Spring Boot

### Análise rápida (sem alterar o `pom.xml`)

Na raiz do projeto Maven:

```bash
mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=$SONAR_TOKEN \
  -Dsonar.projectKey=minha-api
```

```powershell
# PowerShell (coloque os -D entre aspas)
mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar `
  "-Dsonar.host.url=http://localhost:9000" `
  "-Dsonar.token=$env:SONAR_TOKEN" `
  "-Dsonar.projectKey=minha-api"
```

> O atalho `mvn sonar:sonar` só funciona se o plugin estiver declarado no `pom.xml` (configuração abaixo) ou se `org.sonarsource.scanner.maven` estiver em `<pluginGroups>` no `~/.m2/settings.xml`. Sem isso, o Maven retorna `No plugin found for prefix 'sonar'`.

### Configuração permanente no `pom.xml` (com cobertura via JaCoCo)

```xml
<properties>
  <sonar.host.url>http://localhost:9000</sonar.host.url>
  <sonar.projectKey>minha-api</sonar.projectKey>
  <!-- NÃO coloque sonar.token aqui; passe via -Dsonar.token ou variável SONAR_TOKEN -->
</properties>

<build>
  <plugins>
    <plugin>
      <groupId>org.sonarsource.scanner.maven</groupId>
      <artifactId>sonar-maven-plugin</artifactId>
      <version>5.8.0.7211</version>
    </plugin>
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.15</version>
      <executions>
        <execution>
          <id>prepare-agent</id>
          <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
          <id>report</id>
          <phase>verify</phase>
          <goals><goal>report</goal></goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

Com essa configuração:

```bash
mvn clean verify sonar:sonar -Dsonar.token=$SONAR_TOKEN
```

- O `verify` roda os testes e gera `target/site/jacoco/jacoco.xml`. O scanner encontra esse relatório automaticamente e mostra a cobertura no SonarQube.
- O scanner também lê a variável de ambiente `SONAR_TOKEN`. Se ela estiver definida, `-Dsonar.token` é opcional.
- Projetos Spring Boot não precisam de nenhuma configuração extra: o plugin usa as informações do `pom.xml` (fontes, testes, bytecode compilado).
- Em projetos com vários módulos, rode o comando na **raiz** (POM pai).

Ao final, o log mostra `ANALYSIS SUCCESSFUL, you can find the results at: http://localhost:9000/dashboard?id=minha-api`.

> **Aplicação ou Maven dentro de um container:** use `-Dsonar.host.url=http://host.docker.internal:9000` em vez de `localhost`.

## 12. SonarQube for IDE (IntelliJ IDEA e WebStorm)

O **SonarQube for IDE** (antigo SonarLint) analisa o código enquanto você digita. No **modo conectado**, ele sincroniza as regras e o Quality Profile do servidor e marca como resolvidas as issues já tratadas no servidor.

O procedimento é o mesmo no IntelliJ IDEA e no WebStorm (e nas demais IDEs JetBrains):

1. **Settings → Plugins → Marketplace**, procure **SonarQube for IDE**, instale e reinicie a IDE.
2. No SonarQube web, gere um **User Token** em **My Account → Security**.
3. Na IDE: **Settings → Tools → SonarQube for IDE → Connections → `+`**.
   - Tipo: **SonarQube Server**.
   - URL: `http://localhost:9000`.
   - Autenticação: **Token** (cole o User Token).
4. Associe o projeto: **Settings → Tools → SonarQube for IDE → Project Settings → Bind project to SonarQube**. Escolha a conexão e a **Project key** criada no servidor.
5. Use a aba **SonarQube for IDE** (janela inferior) para ver as issues do arquivo atual ou do projeto inteiro (**Report**).

No WebStorm, a análise cobre JavaScript, TypeScript, CSS e HTML. Para projetos front-end, o servidor analisa os mesmos arquivos com o [SonarScanner CLI](https://docs.sonarsource.com/sonarqube-community-build/analyzing-source-code/scanners/sonarscanner/) ou pelo pacote npm `@sonar/scan`:

```bash
npx @sonar/scan -Dsonar.host.url=http://localhost:9000 -Dsonar.token=$SONAR_TOKEN -Dsonar.projectKey=meu-front
```

## 13. Problemas comuns

| Sintoma | Causa provável / solução |
|---|---|
| Container reinicia em loop; log `max virtual memory areas vm.max_map_count [65530] is too low` | Ajuste `vm.max_map_count` (seção 7). |
| Log `max file descriptors [...] is too low` | O limite `nofile` não foi aplicado. Confira os `ulimits` do `compose.yaml` e, no Linux, os limites do serviço `docker` (`LimitNOFILE`). |
| `Elasticsearch did not exit normally` / container encerra com código 137 | Falta memória. Aumente a memória do Docker Desktop para 4 GB ou mais. |
| Status `DB_MIGRATION_NEEDED` após atualizar a imagem | Abra <http://localhost:9000/setup> e inicie a migração. Faça backup do volume antes de qualquer atualização. |
| `Not authorized` / `401` na análise | Token inválido, expirado ou de outro projeto. Gere um novo token. |
| `No plugin found for prefix 'sonar'` | Use a coordenada completa do plugin (seção 11) ou declare-o no `pom.xml`. |
| Cobertura aparece como 0% | O relatório do JaCoCo não foi gerado. Verifique se o `jacoco-maven-plugin` está configurado e se você rodou `verify` (e não só `compile`). |
| Esqueci a senha do `admin` | Recrie o ambiente com `docker compose down -v` (apaga tudo) ou siga o procedimento de reset da documentação oficial. |
| Porta 9000 ocupada | Altere `SONARQUBE_PORT` no `.env` e use a nova porta em `sonar.host.url`. |

> **Para usar em último caso** (por exemplo, quando o ambiente não deixa ajustar o kernel): defina `SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true` no serviço `sonarqube`. Isso desliga as verificações de inicialização do Elasticsearch. Pode servir para um teste rápido, mas mascara problemas reais e não é recomendado.
