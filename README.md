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

### Parte 2: Instalação e Configuração no Ubuntu 24.04

Vamos preparar seu ambiente e instalar o Airflow.

#### Passo 1: Iniciar e Atualizar o Ubuntu 24.04

1.  Atualize os pacotes do seu sistema para garantir que tudo esteja em dia:

    ```bash
    sudo apt update && sudo apt upgrade -y

    ```

#### Passo 2: Instalar Pré-requisitos (Python)

O Ubuntu 24.04 já vem com o Python 3.12, que é perfeito para o Airflow. Precisamos apenas garantir que o `pip` e o `python3-venv` (para ambientes virtuais) estejam instalados.

1.  Instale os pacotes necessários:

    ```bash
    sudo apt install python3-pip python3-venv -y

    ```

2.  Verifique a versão do Python:

    ```bash
    python3 --version

    ```

    Você deve ver algo como `Python 3.12.3`.

#### Passo 3: Criar um Ambiente Virtual e Instalar o Airflow

É uma boa prática instalar pacotes Python em um ambiente virtual para evitar conflitos com pacotes do sistema.

1.  Crie um diretório para o seu projeto e navegue até ele:

    ```bash
    mkdir airflow-lab

    ```

    ```bash
    cd airflow-lab

    ```

2.  Crie um ambiente virtual:

    ```bash
    python3 -m venv venv

    ```

3.  Ative o ambiente virtual. Você precisará fazer isso toda vez que for trabalhar com o Airflow neste terminal.

    ```bash
    source venv/bin/activate

    ```

    Seu prompt de comando mudará, mostrando `(venv)` no início.

4.  Instale o Apache Airflow. Usaremos um arquivo de "constraints" para garantir que todas as dependências sejam compatíveis, o que é a maneira recomendada de instalar.

    ```bash
    pip install "apache-airflow[postgres,cncf.kubernetes]==2.9.2" --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.9.2/constraints-3.12.txt"

    ```

    *Nota: Substitua `3.12` pela sua versão do Python se for diferente.*

### Parte 3: Primeiros Passos com o Airflow

Com o Airflow instalado, a maneira mais fácil de começar um ambiente de desenvolvimento local é usando o comando `standalone`. Ele configura um banco de dados SQLite, cria um usuário para você e inicia todos os componentes necessários (servidor web e scheduler).

#### Passo 1: Iniciar o Ambiente Airflow

1.  No mesmo terminal com o ambiente virtual ativado, execute o seguinte comando:

    ```bash
    airflow standalone

    ```

2.  O Airflow começará a inicializar. Este processo pode levar alguns minutos na primeira vez. Ele irá:

      * Criar um diretório `~/airflow` para armazenar suas configurações e DAGs.
      * Inicializar o banco de dados.
      * Criar uma conta de administrador.
      * Iniciar o servidor web e o scheduler.

3.  Ao final do processo, você verá uma saída com o nome de usuário e a senha do administrador. **Guarde essa senha\!** Será algo como:

    ```
    Admin user created:
    Username: admin
    Password: <senha_gerada_aleatoriamente>
    ```

#### Passo 2: Acessar a Interface de Usuário (UI)

1.  Abra seu navegador no Windows e acesse:
    [**http://localhost:8080**](https://www.google.com/search?q=http://localhost:8080)

2.  Faça login com o usuário `admin` e a senha que você salvou no passo anterior.

3.  Explore a interface\! Você verá uma lista de DAGs de exemplo que vêm com o Airflow. Eles são ótimos para aprender, mas por enquanto, vamos criar o nosso.

### Parte 4: Criando seu Primeiro Pipeline de Dados (DAG)

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

Parabéns\! Você instalou o Apache Airflow em seu ambiente WSL2, iniciou os serviços, explorou a interface e criou, executou e monitorou seu primeiro pipeline de dados.

**Próximos Passos Sugeridos:**

  * Explore outros **Operadores**, como o `PythonOperator` para executar funções Python.
  * Aprenda sobre como passar dados entre tarefas usando **XComs**.
  * Configure um agendamento (`schedule`) para seu DAG em vez de executá-lo manualmente.
  * Tente criar um pipeline um pouco mais complexo, como baixar um arquivo da internet e depois processar seu conteúdo.

Este laboratório fornece a base sólida que você precisa para começar a construir pipelines de dados mais robustos e complexos com o Apache Airflow.