# Laboratório Airflow
Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

---

## Objetivo
Este laboratório visa capacitar engenheiros de dados no uso do Apache Airflow para orquestração de workflows. O foco é aumentar a produtividade, governança e confiabilidade na construção de fluxos de dados.

---

## Etapas do laboratório

Este laboratório irá guiá-lo através dos seguintes estágios:

1.  **O que é o Apache Airflow?** Uma breve introdução conceitual.
2.  **Instalação e Configuração:** Preparando seu ambiente Ubuntu 24.04 e instalando o Airflow.
3.  **Primeiros Passos:** Iniciando os serviços do Airflow e explorando a interface de usuário.
4.  **Criando seu Primeiro Pipeline de Dados (DAG):** Escrevendo, executando e monitorando um workflow simples.

---

## Ambiente de laboratório
Este curso foi desenvolvido para execução principalmente em ambiente **Linux**.<br>
Caso você não tenha um à disposição então recomendo utilizar o AWS Cloud9.<br>
As instruções de criação estão [aqui](https://github.com/infobarbosa/data-engineering-cloud9).

### Abertura de porta AWS
Execute o script a seguir para abrir a porta `8080` do seu ambiente Cloud9.
```bash
curl -sS https://raw.githubusercontent.com/infobarbosa/airflow-lab/main/assets/scripts/cloud9.sh | bash

```

---    

## Atenção aos custos!
Este curso tem como objetivo a criação de um pipeline de dados completo na AWS. É fundamental que os participantes estejam cientes dos custos associados aos serviços da AWS. Embora nos esforcemos para utilizar os serviços de forma otimizada e, sempre que possível, dentro do nível gratuito (Free Tier), a execução contínua de recursos pode gerar cobranças.

**Recomendações:**
*   **Monitore seus gastos:** Utilize o AWS Billing Dashboard para acompanhar seus custos regularmente.
*   **Desligue recursos não utilizados:** Certifique-se de encerrar instâncias EC2, clusters EMR, bancos de dados RDS e outros serviços quando não estiverem em uso.
*   **Exclua recursos ao final do laboratório:** Após a conclusão do curso, revise e exclua todos os recursos criados para evitar cobranças inesperadas.

O autor e a organização deste laboratório não se responsabilizam por quaisquer custos incorridos na sua conta AWS.

---

## Parte 1: O que é o Apache Airflow?

Apache Airflow é uma plataforma de código aberto para orquestração de workflows e pipelines de dados. Em vez de usar scripts cron espalhados por várias máquinas, o Airflow permite que você defina, agende e monitore workflows complexos como código, de forma centralizada.

O conceito central do Airflow é o **DAG (Directed Acyclic Graph)**, ou Grafo Acíclico Dirigido. Pense em um DAG como uma coleção de todas as tarefas que você deseja executar, organizadas de uma forma que reflete seus relacionamentos e dependências.

  * **Directed (Dirigido):** As tarefas têm uma ordem de execução. A Tarefa A deve ser concluída antes que a Tarefa B possa começar.
  * **Acyclic (Acíclico):** As tarefas não podem criar loops. A Tarefa B não pode depender da Tarefa A se a Tarefa A já depende da Tarefa B.
  * **Graph (Grafo):** As tarefas e suas dependências são representadas como um grafo.

Neste laboratório, vamos instalar o Airflow e criar um DAG simples.

## Parte 2: Inicializando o Airflow
Usar Docker é uma das maneiras mais eficientes e limpas de começar a aprender Airflow, pois encapsula toda a complexidade da instalação.

Para aprendizado, a melhor imagem Docker do Airflow é, sem dúvida, a **imagem oficial do Apache Airflow (`apache/airflow`)**. O motivo principal é que a equipe do Airflow fornece um arquivo `docker-compose.yaml` pronto que facilita enormemente a inicialização de um ambiente completo e funcional.

Aqui está o processo para colocar o ambiente de aprendizado no ar usando a imagem oficial.

**Pré-requisito:** Ter o [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado e rodando em seu sistema (ele já se integra perfeitamente com o WSL2 no Windows).

#### **Passo 1: Crie uma Pasta para seu Projeto**
Crie uma pasta onde seus arquivos de configuração e DAGs ficarão.

```bash
mkdir airflow-lab

```

```bash
cd airflow-lab

```

#### **Passo 2: Baixe o Arquivo `docker-compose.yaml`**
Este é o arquivo que descreve todos os serviços que o Docker irá criar. Você pode baixá-lo diretamente do site do Airflow com o seguinte comando:

```bash
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/3.1.0/docker-compose.yaml'

```

*(Verifique no [site oficial do Airflow](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html) a versão mais recente se necessário.)*

#### **Passo 3: Prepare o Ambiente**
O arquivo `docker-compose.yaml` precisa de algumas pastas e de um arquivo `.env` para funcionar corretamente.

1.  Crie as pastas necessárias:

    ```bash
    mkdir -p ./dags ./logs ./plugins ./config

    ```

2.  Crie um arquivo `.env` para definir o UID (User ID) do Airflow, evitando problemas de permissão de arquivos:

    ```bash
    echo -e "AIRFLOW_UID=$(id -u)" > .env

    ```

#### **Passo 4: Suba o Ambiente Airflow**
Agora, com um único comando, o Docker Compose irá baixar as imagens e iniciar todos os contêineres.

1.  Inicialize o banco de dados e crie o usuário admin padrão (`airflow`/`airflow`):

    ```bash
    docker compose up airflow-init

    ```

2.  Após a conclusão do `airflow-init`, suba todo o ambiente em modo "detached" (-d), para que ele continue rodando em segundo plano:

    ```bash
    docker compose up -d

    ```

#### **Passo 5: Acesse o Airflow**
Pronto\! O ambiente está no ar.

  * Abra seu navegador e acesse: [**http://localhost:8080**](https://www.google.com/search?q=http://localhost:8080)
  * Use o login e senha padrão: `airflow` / `airflow`

Agora você tem um ambiente Airflow completo e funcional. Qualquer arquivo `.py` que você colocar na pasta `dags` que você criou localmente aparecerá automaticamente na interface do Airflow.

---

## Parte 3: Criando seu Primeiro Pipeline de Dados (DAG)

Nosso pipeline será simples:

  * **Tarefa 1:** Imprimir a data e hora atuais.
  * **Tarefa 2:** Listar os arquivos no diretório de DAGs.
  * **Dependência:** A Tarefa 2 só pode começar depois que a Tarefa 1 for concluída com sucesso.

#### Passo 1: Criar o Arquivo do DAG

1.  O Airflow procura por arquivos Python na sua pasta de DAGs. Por padrão, ela está localizada em `~/airflow/dags`.

2.  Abra um **novo terminal** WSL (deixe o `airflow standalone` rodando no primeiro). Crie um novo arquivo Python para o nosso DAG:

    ```bash
    nano ~/airflow/dags/meu_primeiro_dag.py
    ```

    *Você pode usar qualquer editor de texto, como `vim` ou até mesmo o VS Code com a extensão WSL.*

3.  Cole o seguinte código no arquivo:

    ```python
    from __future__ import annotations

    import pendulum

    from airflow.models.dag import DAG
    from airflow.operators.bash import BashOperator

    # Define o DAG
    with DAG(
        dag_id="meu_primeiro_dag",  # O ID único do seu DAG
        start_date=pendulum.datetime(2023, 10, 26, tz="UTC"), # A data de início
        schedule=None,  # Não agendaremos, vamos disparar manualmente
        catchup=False,  # Não executa execuções passadas
        tags=["laboratorio", "introducao"], # Etiquetas para organizar seus DAGs
    ) as dag:
        # Define a primeira tarefa usando o BashOperator
        tarefa_imprimir_data = BashOperator(
            task_id="imprimir_data", # O ID único da tarefa
            bash_command="date", # O comando a ser executado
        )

        # Define a segunda tarefa
        tarefa_listar_arquivos = BashOperator(
            task_id="listar_arquivos",
            # O comando `sleep 5` é só para simular uma tarefa que leva mais tempo
            bash_command="sleep 5 && ls -l ~/airflow/dags",
        )

        # Define a dependência entre as tarefas
        # A tarefa_listar_arquivos só começará depois que a tarefa_imprimir_data for concluída
        tarefa_imprimir_data >> tarefa_listar_arquivos

    ```

4.  Salve e feche o arquivo (no `nano`, pressione `Ctrl+X`, depois `Y` e `Enter`).

#### Passo 2: Ver, Ativar e Executar o DAG

1.  Volte para a interface do Airflow no seu navegador (`http://localhost:8080`). Em um ou dois minutos, o Airflow detectará seu novo arquivo e o `meu_primeiro_dag` aparecerá na lista de DAGs.

2.  Por padrão, novos DAGs estão pausados. Ative-o clicando no botão de alternância à esquerda do nome do DAG.

3.  Clique no nome do DAG (`meu_primeiro_dag`) para ver mais detalhes. A visualização "Grid" é a mais útil para começar.

4.  Para executar o DAG manualmente, clique no botão "Play" (▶️) no canto superior direito e selecione "Trigger DAG".

#### Passo 3: Monitorar a Execução

1.  Na visualização "Grid", você verá uma nova execução do seu DAG aparecer. Os quadrados representarão as tarefas. Eles mudarão de cor à medida que a execução progride:

      * **Cinza:** Pendente
      * **Verde claro:** Em execução
      * **Verde escuro:** Sucesso
      * **Vermelho:** Falha

2.  Você pode clicar em qualquer uma das tarefas (quadrados) e selecionar "Log" para ver a saída do comando que foi executado.

      * No log da tarefa `imprimir_data`, você verá a data e hora.
      * No log da tarefa `listar_arquivos`, você verá a lista de arquivos no seu diretório de DAGs.

-----

### Conclusão

Parabéns\! Você ininializou o Apache Airflow em seu ambiente Linux, explorou a interface e criou, executou e monitorou seu primeiro pipeline de dados.

**Próximos Passos Sugeridos:**

  * Explore outros **Operadores**, como o `PythonOperator` para executar funções Python.
  * Aprenda sobre como passar dados entre tarefas usando **XComs**.
  * Configure um agendamento (`schedule`) para seu DAG em vez de executá-lo manualmente.
  * Tente criar um pipeline um pouco mais complexo, como baixar um arquivo da internet e depois processar seu conteúdo.
