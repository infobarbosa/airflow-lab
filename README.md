# Apache Airflow + Apache Spark Lab
Author: Prof. Barbosa<br>
Contact: infobarbosa@gmail.com<br>
Github: [infobarbosa](https://github.com/infobarbosa)

## Objetivo
O objetivo deste laboratório é estudar o Apache Airflow e o Apache Spark.

## Stack
| Componente | Versão |
|---|---|
| Apache Airflow | 3.2.1 |
| Python | 3.12 |
| Apache Spark | 4.1.1 |
| PySpark | 4.1.1 |
| PostgreSQL | 17 |
| Redis | 7.2 |
| Java | 17 (OpenJDK, Debian Bookworm) |

## Ambiente
Este laboratório pode ser executado em qualquer estação de trabalho com Docker disponível.<br>
Recomendo a execução em Linux. Caso não tenha um à disposição, utilize o serviço **AWS Cloud9** — instruções [aqui](https://github.com/infobarbosa/data-engineering-cloud9).

---

## Parte 1 — Entendendo o Dockerfile

O `Dockerfile` descreve como construir uma **imagem Docker customizada** para o Airflow. Precisamos customizá-la porque a imagem oficial do Airflow não vem com Java nem com PySpark, que são necessários para se comunicar com o cluster Spark.

```dockerfile
FROM apache/airflow:3.2.1-python3.12
```

**`FROM`** define a imagem base — o ponto de partida da nossa imagem. Estamos usando a imagem oficial do Apache Airflow na versão `3.2.1` com Python `3.12`. Tudo que já está nessa imagem (o próprio Airflow, suas dependências, o sistema Debian Bookworm) é herdado automaticamente. Só precisamos acrescentar o que falta.

```dockerfile
USER root
```

Por padrão, a imagem oficial do Airflow opera com um usuário sem privilégios chamado `airflow` (UID 50000). Para instalar pacotes do sistema operacional com o `apt-get`, precisamos temporariamente ser `root`. A diretiva `USER root` faz essa troca.

> **Boas práticas de segurança:** nunca deixe o container rodando como `root` em produção. Voltamos para o usuário `airflow` logo após a instalação do sistema.

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends openjdk-17-jre-headless && \
    apt-get autoremove -yqq --purge && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Por que instalar Java?** O PySpark precisa da JVM (Java Virtual Machine) para funcionar. Sem Java, ao tentar submeter um job Spark, o processo falha imediatamente.

Detalhes de cada linha:
- `apt-get update` — atualiza o índice de pacotes disponíveis no repositório Debian.
- `apt-get install -y --no-install-recommends openjdk-17-jre-headless` — instala o Java Runtime (JRE) sem a interface gráfica e sem pacotes recomendados que não precisamos. A flag `--no-install-recommends` mantém a imagem enxuta.
- `apt-get autoremove && apt-get clean && rm -rf /var/lib/apt/lists/*` — remove pacotes temporários e o cache do apt. Em Docker cada `RUN` vira uma camada (layer). Limpar ao final da mesma camada garante que o tamanho da imagem não cresça desnecessariamente.

> **Java 17 vs 21:** O Spark 4.x exige Java 17 no mínimo. O Java 21 (LTS mais recente) não está disponível nos repositórios padrão do Debian Bookworm — precisaria de backports. Java 17 é suficiente e está disponível por padrão.

```dockerfile
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
ENV PATH="/home/airflow/.local/bin:${JAVA_HOME}/bin:${PATH}"
```

**`ENV`** define variáveis de ambiente que persistem dentro do container em tempo de execução.

- `JAVA_HOME` — aponta para o diretório de instalação do Java. Diversas ferramentas (incluindo o PySpark) consultam essa variável para encontrar a JVM.
- `PATH` — lista de diretórios onde o sistema operacional procura por executáveis. Acrescentamos dois caminhos na frente:
  - `/home/airflow/.local/bin` — onde o `pip install --user` coloca os binários do Airflow (como o comando `airflow`).
  - `${JAVA_HOME}/bin` — onde ficam os executáveis do Java (`java`, `javac` etc.).

```dockerfile
USER airflow
```

Voltamos para o usuário `airflow` (sem privilégios de root) antes de instalar pacotes Python. Isso garante que os pacotes sejam instalados no contexto correto do usuário e que o container não execute com permissões desnecessárias.

```dockerfile
RUN pip install --no-cache-dir \
    apache-airflow-providers-apache-spark==6.0.1 \
    apache-airflow-providers-standard \
    pyspark==4.1.1 \
    --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-3.2.1/constraints-3.12.txt"
```

Instalamos três pacotes Python:

- **`apache-airflow-providers-apache-spark==6.0.1`** — o "provider" oficial do Airflow para o Spark. Ele adiciona o `SparkSubmitOperator` e a configuração de conexão do tipo "Spark" na UI. Providers são plugins que estendem o Airflow com suporte a sistemas externos.
- **`apache-airflow-providers-standard`** — pacote obrigatório no Airflow 3 que contém operadores que antes vinham no núcleo, como `BashOperator` e `PythonOperator`.
- **`pyspark==4.1.1`** — a biblioteca Python do Spark. Permite que o Airflow monte e submeta jobs usando a API Python. A versão deve ser idêntica à do cluster Spark (4.1.1).

O flag `--no-cache-dir` evita que o pip guarde cache de pacotes, mantendo a imagem menor.

O `--constraint` aponta para o arquivo de restrições de versão mantido pelo próprio projeto Airflow. Esse arquivo especifica as versões exatas de cada dependência que foram testadas e aprovadas para funcionar juntas com o Airflow 3.2.1 + Python 3.12. Sem ele, o pip poderia instalar uma versão nova de alguma dependência que quebrasse o Airflow.

---

## Parte 2 — Entendendo o compose.yaml

O `compose.yaml` orquestra **9 containers** que trabalham juntos para formar o ambiente completo: infraestrutura (banco de dados e message broker), componentes do Airflow, e o cluster Spark.

### 2.1 YAML Anchors — evitando repetição

```yaml
x-airflow-common:
  &airflow-common
  build: .
  environment:
    &airflow-common-env
    ...
  volumes:
    ...
  depends_on:
    &airflow-common-depends-on
    ...
```

Esta seção usa um recurso do YAML chamado **âncoras** (`&`) e **referências** (`*`) para evitar repetir configurações idênticas em todos os serviços do Airflow.

- `x-airflow-common` — prefixo `x-` indica uma extensão (extension field) do Docker Compose. Ela não cria nenhum container; serve apenas como bloco de configuração reutilizável.
- `&airflow-common` — define a âncora com o nome `airflow-common`.
- `<<: *airflow-common` (usado nos serviços) — "mescle tudo que está em `airflow-common` aqui". É o equivalente de herança em orientação a objetos.
- `&airflow-common-env` e `&airflow-common-depends-on` — sub-âncoras para que serviços individuais possam herdar apenas o bloco de environment ou de depends_on, acrescentando campos extras sem duplicar tudo.

### 2.2 Variáveis de ambiente compartilhadas

As variáveis de ambiente do Airflow seguem um padrão de nomenclatura: `AIRFLOW__{SEÇÃO}__{CHAVE}`. Cada variável sobrescreve a configuração equivalente no `airflow.cfg`. Isso permite configurar o Airflow completamente via ambiente, sem editar arquivos de configuração.

```yaml
AIRFLOW__CORE__EXECUTOR: CeleryExecutor
```
Define o **executor**, que é o mecanismo pelo qual o Airflow distribui e executa as tasks. O `CeleryExecutor` envia cada task como uma mensagem para uma fila (Redis), onde um ou mais workers a consomem e executam. É a arquitetura correta para ambientes com múltiplos workers e alta disponibilidade.

> Outros executores existem: `LocalExecutor` (executa tudo no processo do scheduler, sem fila) e `KubernetesExecutor` (cria um Pod por task). Para este lab usamos Celery por ser o mais próximo de um ambiente real de produção.

```yaml
AIRFLOW__CORE__AUTH_MANAGER: airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
```
No Airflow 3, o gerenciamento de autenticação foi desacoplado do núcleo e transformado em um componente plugável. Esta variável declara explicitamente que usaremos o **FAB Auth Manager** (Flask-AppBuilder), que é o gerenciador padrão com suporte a usuários, roles e permissões pela UI.

```yaml
AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
```
String de conexão com o banco de dados de metadados do Airflow. O Airflow armazena tudo aqui: DAGs, runs, task instances, conexões, variáveis, logs de auditoria. O formato é `driver://usuario:senha@host/banco`. O hostname `postgres` é resolvido pelo DNS interno do Docker Compose pelo nome do serviço.

```yaml
AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql+psycopg2://airflow:airflow@postgres/airflow
```
Quando um Celery worker termina uma task, ele precisa gravar o resultado (status de sucesso/falha) em algum lugar. Esta variável aponta para o backend de resultados — aqui, o mesmo PostgreSQL. O prefixo `db+` é a notação do Celery para backends baseados em banco de dados.

```yaml
AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
```
O **broker** é a fila de mensagens do Celery. O Scheduler publica tasks nessa fila; os Workers consomem. Usamos Redis como broker. O formato é `redis://:senha@host:porta/numero_do_banco`. A senha está vazia (`:@`) pois o Redis deste lab não tem autenticação. O `/0` indica o banco de dados 0 do Redis (Redis suporta múltiplos bancos lógicos).

```yaml
AIRFLOW__CORE__FERNET_KEY: 'IZ-mcMBkRg5e41OB59SlEWjsim6nOGyvT8lWVuuM1y0='
```
O Airflow usa criptografia simétrica (Fernet) para proteger dados sensíveis armazenados no banco, como senhas de conexões. Esta chave de 32 bytes (em base64 URL-safe) deve ser **idêntica em todos os containers** — se cada um gerasse a sua, não conseguiriam descriptografar os dados uns dos outros. Para produção, gere uma chave única com:
```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

```yaml
AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
```
Quando um novo DAG é detectado, ele nasce pausado (não agenda runs automaticamente). Isso evita execuções acidentais logo após um deploy. O operador deve ativar manualmente o DAG quando estiver pronto.

```yaml
AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
```
A imagem oficial vem com DAGs de exemplo que poluem a interface. Desativamos para manter o ambiente limpo para o laboratório.

```yaml
AIRFLOW__CORE__EXECUTION_API_SERVER_URL: 'http://airflow-apiserver:8080/execution/'
```
Novidade do Airflow 3. O Scheduler e os Workers se comunicam com o API Server via HTTP para reportar e buscar o estado de execução das tasks. Este endpoint interno usa o hostname do serviço Docker (`airflow-apiserver`).

```yaml
AIRFLOW__API_AUTH__JWT_SECRET: 'airflow_jwt_secret_lab'
AIRFLOW__API_AUTH__JWT_ISSUER: 'airflow'
```
No Airflow 3, a comunicação interna entre componentes (Scheduler → API Server, Worker → API Server) é autenticada via **JWT** (JSON Web Token). O `JWT_SECRET` é a chave usada para assinar os tokens. Para produção, use um valor longo e aleatório.

```yaml
AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'
```
Ativa um servidor HTTP simples no Scheduler (porta 8974) que responde a requisições de health check. Isso permite que o Docker Compose verifique se o Scheduler está vivo.

### 2.3 Volumes compartilhados

```yaml
volumes:
  - ./dags:/opt/airflow/dags
  - ./logs:/opt/airflow/logs
  - ./plugins:/opt/airflow/plugins
  - ./config:/opt/airflow/config
```

Todos os containers do Airflow montam os mesmos diretórios locais. O formato é `caminho_no_host:caminho_no_container`.

- `./dags` — onde você escreve seus arquivos Python de DAG. O `dag-processor` lê daqui e registra os DAGs no banco. O `worker` também precisa acessar para executar as tasks.
- `./logs` — logs de execução de cada task instance. Montado em todos os serviços para que a UI consiga exibir os logs independente de qual worker executou a task.
- `./plugins` — operadores, hooks e macros customizados que estendem o Airflow.
- `./config` — arquivo `airflow.cfg` gerado na inicialização. Permite persistir configurações entre reinicializações.

### 2.4 Dependências entre serviços

```yaml
depends_on:
  &airflow-common-depends-on
  redis:
    condition: service_healthy
  postgres:
    condition: service_healthy
```

`depends_on` controla a **ordem de inicialização**. O Docker Compose não sobe um serviço até que suas dependências estejam prontas. A `condition: service_healthy` significa "aguarde até o health check passar" — muito mais confiável do que simplesmente esperar o container iniciar.

---

### 2.5 Os serviços em detalhe

#### postgres

```yaml
postgres:
  image: postgres:17
  environment:
    POSTGRES_USER: airflow
    POSTGRES_PASSWORD: airflow
    POSTGRES_DB: airflow
  volumes:
    - postgres-db-volume:/var/lib/postgresql/data
  healthcheck:
    test: ["CMD", "pg_isready", "-U", "airflow"]
    interval: 10s
    retries: 5
    start_period: 5s
  restart: always
```

O **PostgreSQL** é o banco de metadados do Airflow — armazena tudo: definições de DAGs, histórico de execuções, conexões, variáveis e usuários.

- `image: postgres:17` — usamos a imagem oficial sem customização.
- `POSTGRES_USER/PASSWORD/DB` — usuário, senha e nome do banco criados na primeira inicialização.
- `volumes: postgres-db-volume` — volume **nomeado** (gerenciado pelo Docker). Diferente dos bind mounts (`./dags`), volumes nomeados persistem os dados mesmo após `docker compose down`. Os dados do banco sobrevivem a reinicializações. Só são apagados com `docker compose down --volumes`.
- `healthcheck` — o comando `pg_isready` verifica se o PostgreSQL está aceitando conexões. O Docker testa a cada `10s`, espera até `5s` para cada resposta, tenta `5` vezes antes de declarar o serviço como unhealthy. O `start_period: 5s` é um período de tolerância inicial para o banco terminar de inicializar.
- `restart: always` — se o container cair por qualquer motivo, o Docker reinicia automaticamente.

#### redis

```yaml
redis:
  image: redis:7.2-bookworm
  expose:
    - 6379
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 30s
    retries: 50
    start_period: 30s
  restart: always
```

O **Redis** é o **message broker** do Celery. O Scheduler publica mensagens do tipo "execute a task X" nesta fila; os Workers ficam escutando e consomem essas mensagens.

- `image: redis:7.2-bookworm` — versão específica conforme recomendação oficial do Airflow (Redis mudou de licença na versão 7.4+, então a comunidade fixou em 7.2).
- `expose: 6379` — torna a porta 6379 acessível apenas para outros containers na mesma rede Docker. Diferente de `ports`, o `expose` **não** publica a porta no host — por segurança, o Redis não precisa ser acessível de fora do Docker.
- `healthcheck` — o comando `redis-cli ping` retorna `PONG` quando o Redis está pronto. O timeout alto (`30s`) e muitas retries (`50`) são necessários porque o Redis pode levar alguns segundos para responder durante inicialização sob carga.

#### airflow-init

```yaml
airflow-init:
  <<: *airflow-common
  entrypoint: /bin/bash
  command:
    - -c
    - |
      mkdir -p /opt/airflow/{logs,dags,plugins,config}
      chown -R "50000:0" /opt/airflow/
      exec /entrypoint airflow version
  environment:
    <<: *airflow-common-env
    _AIRFLOW_DB_MIGRATE: 'true'
    _AIRFLOW_WWW_USER_CREATE: 'true'
    _AIRFLOW_WWW_USER_USERNAME: 'admin'
    _AIRFLOW_WWW_USER_PASSWORD: 'admin'
  user: "0:0"
```

Este é o **serviço de inicialização** — executa uma única vez e termina. Ele é o pré-requisito de todos os outros serviços Airflow (`condition: service_completed_successfully`).

- `user: "0:0"` — roda como root (UID 0, GID 0) para poder criar diretórios e ajustar permissões.
- `entrypoint: /bin/bash` + `command` — substitui o entrypoint padrão da imagem por um script bash que:
  1. Cria os diretórios necessários dentro do container.
  2. Define o dono dos arquivos como UID `50000` (o usuário `airflow`), garantindo que os outros containers consigam ler e escrever.
  3. Executa `/entrypoint airflow version`, que é o entrypoint original da imagem do Airflow. Este binário lê as variáveis abaixo e age sobre elas antes de executar o comando `airflow version`.
- `_AIRFLOW_DB_MIGRATE: 'true'` — sinaliza para o entrypoint que ele deve rodar `airflow db migrate` automaticamente, criando ou atualizando todas as tabelas no PostgreSQL.
- `_AIRFLOW_WWW_USER_CREATE: 'true'` — sinaliza para criar o usuário administrador na UI.
- `_AIRFLOW_WWW_USER_USERNAME/PASSWORD` — credenciais do usuário admin criado (login: `admin`, senha: `admin`).

> Esta é a abordagem oficial documentada em https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html. O usuário admin é criado pelo próprio entrypoint da imagem — não via `airflow users create`.

#### airflow-apiserver

```yaml
airflow-apiserver:
  <<: *airflow-common
  command: api-server
  ports:
    - "8080:8080"
  healthcheck:
    test: ["CMD", "curl", "--fail", "http://localhost:8080/api/v2/monitor/health"]
  depends_on:
    <<: *airflow-common-depends-on
    airflow-init:
      condition: service_completed_successfully
```

**Novidade do Airflow 3.** O que era o `webserver` no Airflow 2 foi dividido: a interface web e a API REST agora vivem no **API Server**, construído com FastAPI (antes era Flask/FAB).

- `command: api-server` — inicia o componente de API e UI.
- `ports: "8080:8080"` — publica a porta no host. Formato: `porta_no_host:porta_no_container`. Acesse pelo navegador em `http://localhost:8080`.
- `healthcheck` — verifica o endpoint `/api/v2/monitor/health` da própria API REST do Airflow. Retorna JSON com status de todos os componentes.
- `depends_on: airflow-init: condition: service_completed_successfully` — só sobe depois que o `airflow-init` terminar com código 0 (sucesso), garantindo que o banco já foi migrado e o usuário admin já existe.

#### airflow-scheduler

```yaml
airflow-scheduler:
  <<: *airflow-common
  command: scheduler
  healthcheck:
    test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
```

O **Scheduler** é o coração do Airflow. Ele:
1. Monitora todos os DAGs e verifica se chegou a hora de criar novas runs (baseado em `schedule`).
2. Determina quais tasks de uma run estão prontas para executar (dependências satisfeitas).
3. Envia as tasks prontas para a fila do Celery (Redis).

- `healthcheck` na porta `8974` — servidor HTTP interno do Scheduler (ativado por `AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: 'true'`). Retorna `200 OK` enquanto o Scheduler estiver vivo.

#### airflow-dag-processor

```yaml
airflow-dag-processor:
  <<: *airflow-common
  command: dag-processor
  healthcheck:
    test: ["CMD-SHELL", 'airflow jobs check --job-type DagProcessorJob --hostname "$${HOSTNAME}"']
```

**Outro componente novo do Airflow 3.** No Airflow 2, o Scheduler também processava (parseava) os arquivos de DAG. No Airflow 3 essa responsabilidade foi separada em um processo dedicado.

O **DAG Processor** varre continuamente o diretório `./dags`, importa os arquivos Python, extrai as definições de DAG e as serializa no banco de dados. Isso melhora o isolamento: se um arquivo de DAG tiver um erro de sintaxe ou importação pesada, ele não afeta o Scheduler.

- `healthcheck` — usa o comando `airflow jobs check` para verificar se o processo está registrado como vivo no banco de dados.
- `$${HOSTNAME}` — o `$$` é a forma de escapar o `$` dentro de strings YAML usadas como comandos shell no Docker Compose (evita que o Compose tente interpolar a variável).

#### airflow-worker

```yaml
airflow-worker:
  <<: *airflow-common
  command: celery worker
  environment:
    <<: *airflow-common-env
    DUMB_INIT_SETSID: "0"
  depends_on:
    <<: *airflow-common-depends-on
    airflow-apiserver:
      condition: service_healthy
    airflow-init:
      condition: service_completed_successfully
```

O **Worker** é quem de fato executa as tasks. Ele:
1. Fica escutando a fila do Redis (broker Celery).
2. Quando recebe uma mensagem de task, executa o código Python do operador (ex.: `SparkSubmitOperator`).
3. Reporta o resultado (sucesso/falha) de volta para o Airflow via API Server.

- `command: celery worker` — inicia o processo Celery worker dentro do container Airflow.
- `DUMB_INIT_SETSID: "0"` — o `dumb-init` (init process do container) normalmente cria uma nova sessão de processo. Este flag desativa esse comportamento para que o Celery receba sinais de shutdown corretamente, permitindo que tasks em andamento terminem antes do container parar (graceful shutdown).
- `depends_on: airflow-apiserver: condition: service_healthy` — o Worker precisa que o API Server esteja de pé para poder reportar o estado das tasks. Não basta o init ter terminado.

#### airflow-triggerer

```yaml
airflow-triggerer:
  <<: *airflow-common
  command: triggerer
  healthcheck:
    test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']
```

O **Triggerer** suporta **operadores deferríveis** — uma categoria de operadores que pausam sua execução enquanto aguardam um evento externo (ex.: um arquivo chegar no S3, uma query terminar no BigQuery), liberando o worker para executar outras tasks nesse intervalo.

Sem o Triggerer, um sensor tradicional ocupa um slot de worker o tempo todo que fica fazendo polling. Com operadores deferríveis e o Triggerer, o worker é liberado imediatamente e o Triggerer fica monitorando o evento de forma assíncrona e eficiente.

---

### 2.6 O cluster Spark

#### spark-master

```yaml
spark-master:
  image: apache/spark:4.1.1-java21-python3
  command: /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
  ports:
    - "7077:7077"
    - "9090:8080"
```

O **Master** é o nó coordenador do cluster Spark no modo Standalone. Ele:
- Recebe requisições de jobs dos clientes (ex.: `spark-submit` disparado pelo Airflow).
- Aloca recursos nos Workers disponíveis.
- Monitora a saúde dos Workers.

- `image: apache/spark:4.1.1-java21-python3` — imagem oficial mantida pela Apache Software Foundation. A tag `java21-python3` inclui Java 21 e suporte a Python.
- `command` — inicia explicitamente a classe Java do Master. A imagem oficial não usa variáveis de ambiente como `SPARK_MODE` (padrão Bitnami) — o papel do container é definido pelo comando passado.
- `porta 7077` — porta de comunicação interna do cluster. Workers e clientes conectam aqui para se registrar/submeter jobs.
- `porta 9090:8080` — a UI web do Master roda na porta 8080 do container. Publicamos como 9090 no host para não colidir com a porta 8080 do Airflow. Acesse em `http://localhost:9090`.

#### spark-worker

```yaml
spark-worker:
  image: apache/spark:4.1.1-java21-python3
  command: /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
  depends_on:
    - spark-master
```

O **Worker** é o nó executor do cluster. Ele:
- Registra-se no Master ao inicializar.
- Recebe tarefas de processamento (executors) do Master.
- Executa as transformações Spark e retorna os resultados.

- `command` — inicia a classe Java do Worker passando o endereço do Master como argumento (`spark://spark-master:7077`). O hostname `spark-master` é resolvido pelo DNS interno do Docker Compose.
- `depends_on: spark-master` — o Worker precisa que o Master esteja rodando antes de tentar se registrar.

> Para escalar horizontalmente, adicione mais replicas do Worker com `docker compose up --scale spark-worker=3`.

---

### 2.7 Volumes nomeados

```yaml
volumes:
  postgres-db-volume:
```

Declara o volume nomeado `postgres-db-volume` gerenciado pelo Docker. Diferentemente dos bind mounts (`./dags:/opt/airflow/dags`), volumes nomeados:
- São gerenciados pelo Docker Engine, não pelo sistema de arquivos do host.
- **Persistem os dados após `docker compose down`** (são apagados apenas com `--volumes`).
- Têm melhor performance em Linux e são mais portáveis entre sistemas operacionais.

---

## Parte 3 — Arquitetura completa

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker Network                           │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────────┐  │
│  │ postgres │    │  redis   │    │     airflow-init          │  │
│  │  :5432   │    │  :6379   │    │  (executa e termina)      │  │
│  └────┬─────┘    └────┬─────┘    └──────────────────────────┘  │
│       │               │                        │                │
│       └───────────────┴────────────────────────┘                │
│                         depende de ↑                            │
│                                                                 │
│  ┌─────────────────┐   ┌──────────────────┐                    │
│  │ airflow-        │   │ airflow-         │                    │
│  │ apiserver :8080 │◄──│ dag-processor    │                    │
│  └────────┬────────┘   └──────────────────┘                    │
│           │                                                     │
│  ┌────────┴────────┐   ┌──────────────────┐                    │
│  │ airflow-        │   │ airflow-         │                    │
│  │ scheduler       │   │ triggerer        │                    │
│  └────────┬────────┘   └──────────────────┘                    │
│           │ publica tasks                                       │
│           ▼                                                     │
│  ┌────────────────┐   ┌────────────────────────────────────┐   │
│  │ airflow-worker │──►│ spark-master :7077  UI :9090       │   │
│  │ (spark-submit) │   │   └── spark-worker                 │   │
│  └────────────────┘   └────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Fluxo de execução de um DAG com SparkSubmitOperator:**
1. O **DAG Processor** detecta o arquivo `.py` em `./dags` e registra o DAG no banco.
2. O **Scheduler** verifica que o DAG deve ser executado e cria uma Task Instance.
3. O **Scheduler** publica a task na fila do **Redis**.
4. O **Worker** consome a mensagem da fila e executa o `SparkSubmitOperator`.
5. O `SparkSubmitOperator` chama `spark-submit` apontando para o **spark-master**.
6. O **Spark Master** distribui o job para os **Spark Workers**.
7. O resultado volta para o Worker, que reporta sucesso/falha ao **API Server**.
8. A UI do **API Server** exibe o status final.

---

## Parte 4 — Executando o laboratório

> **Atenção:** todos os comandos presumem que você está no diretório raiz do projeto.

### 4.1 Build da imagem customizada

```bash
docker compose build --no-cache
```

### 4.2 Crie os diretórios necessários

```bash
mkdir -p ./dags ./logs ./plugins ./config
```

### 4.3 Inicialize o banco e crie o usuário admin

```bash
docker compose up airflow-init
```

Aguarde a mensagem: `airflow-init-1 exited with code 0`

### 4.4 Suba todos os serviços

```bash
docker compose up -d
```

### 4.5 Verifique os logs

```bash
docker compose logs -f
```
> Para sair, pressione `Control+C`

---

## Parte 5 — Usando o Airflow

Abra o navegador e acesse `http://localhost:8080`<br>
Faça login com usuário `admin` e senha `admin`.

### 5.A. Configure a conexão com o Spark

- Vá em **Admin → Connections**
- Clique em `+` para adicionar
- **Connection Id**: `spark_default`
- **Connection Type**: `Spark`
- **Host**: `spark://spark-master`
- **Port**: `7077`

### 5.B. Crie o DAG de teste

#### `dags/test_spark_dag.py`

```python
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from datetime import datetime

with DAG(
    dag_id='teste_spark_orchestration',
    start_date=datetime(2023, 1, 1),
    schedule=None,
    catchup=False
) as dag:

    submit_job = SparkSubmitOperator(
        task_id='submit_pyspark_job',
        application='/opt/airflow/dags/scripts/hello_spark.py',
        conn_id='spark_default',
        verbose=True
    )
```

#### `dags/scripts/hello_spark.py`

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("AirflowLab").getOrCreate()
data = [("Marcelo", 45), ("Airflow", 10), ("Spark", 12)]
df = spark.createDataFrame(data, ["Name", "Age"])
df.show()
spark.stop()
```

---

## Parte 6 — Limpeza

### 6.1 Containers, redes e volumes nomeados

Para todos os containers, remove as redes criadas pelo Compose e apaga o volume `postgres-db-volume` (dados do banco):

```bash
docker compose down --volumes --remove-orphans
```

### 6.2 Arquivos de DAG criados no laboratório

```bash
rm -rf ./dags/test_spark_dag.py ./dags/scripts/
```

### 6.3 Diretórios locais criados no laboratório

Remove os diretórios de logs, plugins e config montados como bind mount nos containers:

```bash
rm -rf ./logs ./plugins ./config ./dags
```

### 6.4 Imagens construídas (build local)

Imagens geradas a partir do `Dockerfile` deste projeto:

```bash
docker rmi airflow-lab-airflow-apiserver \
           airflow-lab-airflow-scheduler \
           airflow-lab-airflow-dag-processor \
           airflow-lab-airflow-worker \
           airflow-lab-airflow-triggerer \
           airflow-lab-airflow-init
```

### 6.5 Imagens baixadas do Docker Hub (opcional)

> **Atenção:** execute este passo apenas se não utilizar essas imagens em outros projetos.

```bash
docker rmi apache/spark:4.1.1-java21-python3
```

```bash
docker rmi apache/airflow:3.2.1-python3.12
```

```bash
docker rmi postgres:17
```

```bash
docker rmi redis:7.2-bookworm
```

---

## Parabéns
Neste laboratório nós:
1. Construímos uma imagem Docker customizada estendendo o Airflow 3 oficial com Java e PySpark.
2. Orquestramos 9 containers com Docker Compose seguindo o template oficial do Airflow 3.
3. Criamos um cluster Apache Spark 4 standalone com master e worker.
4. Configuramos a integração Airflow → Spark via `SparkSubmitOperator`.
