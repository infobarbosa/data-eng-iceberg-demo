# Apache Iceberg com Spark e MinIO
Author: Prof. Barbosa  
Contact: infobarbosa@gmail.com  
Github: [infobarbosa](https://github.com/infobarbosa)

O objetivo deste projeto é aprender os conceitos fundamentais e os recursos avançados do Apache Iceberg de forma prática, utilizando um ambiente local com Spark e MinIO (um armazenamento de objetos compatível com S3).

## Pré-requisitos

Para seguir este roteiro, você precisará ter instalado em sua máquina:

  * **Docker:** [https://www.docker.com/get-started](https://www.docker.com/get-started)
  * **Docker Compose:** (geralmente já vem com o Docker Desktop)

## Estrutura do Roteiro

1.  Sessão 1: Configuração do Ambiente
2.  Sessão 2: Preparação dos Dados
3.  Sessão 3: Ingestão e Consulta de Dados
4.  Sessão 4: O Poder do Time Travel (Viagem no Tempo)
5.  Sessão 5: Evolução de Schema (Schema Evolution)
6.  Sessão 6: Particionamento Otimizado
7.  Sessão 7: Manutenção de Tabelas

-----

### Sessão 1: Configuração do Ambiente 🐳

Vamos começar criando nosso ambiente de desenvolvimento. Usaremos o Docker para orquestrar três serviços:

  * **Spark:** O motor de processamento que usaremos para interagir com as tabelas Iceberg.
  * **Iceberg REST Catalog:** Um serviço para gerenciar os metadados das nossas tabelas.
  * **MinIO:** Nosso Data Lake local, onde os dados serão armazenados.

**1. Crie uma pasta para o projeto:**

```bash
mkdir projeto-iceberg
cd projeto-iceberg
```

**2. Crie o arquivo `compose.yml`:**
Dentro da pasta `projeto-iceberg`, crie um arquivo com o nome `compose.yml` e cole o seguinte conteúdo:

```yaml
services:
  spark-iceberg:
    image: tabulario/spark-iceberg:3.5
    container_name: spark-iceberg
    depends_on:
      - rest-catalog
      - minio
    volumes:
      - ./data:/home/iceberg/data
      - ./warehouse:/home/iceberg/warehouse
    environment:
      - SPARK_HOME=/opt/spark
      - SPARK_CONF_DIR=/opt/spark/conf
      - SPARK_CATALOG_CATALOG-NAME: org.apache.iceberg.spark.SparkCatalog
      - SPARK_CATALOG_CATALOG-NAME_CATALOG-IMPL: org.apache.iceberg.rest.RESTCatalog
      - SPARK_CATALOG_CATALOG-NAME_URI: http://rest-catalog:8181
      - SPARK_CATALOG_CATALOG-NAME_S3_ENDPOINT: http://minio:9000
      - SPARK_CATALOG_CATALOG-NAME_S3_ACCESS-KEY-ID: admin
      - SPARK_CATALOG_CATALOG-NAME_S3_SECRET-ACCESS-KEY: password
      - SPARK_CATALOG_CATALOG-NAME_S3_PATH-STYLE-ACCESS: "true"
      - SPARK_SQL_EXTENSIONS: org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
    networks:
      - iceberg_net

  rest-catalog:
    image: tabulario/iceberg-rest:0.8.0
    container_name: rest-catalog
    ports:
      - "8181:8181"
    environment:
      - CATALOG_WAREHOUSE: s3a://warehouse/
      - CATALOG_IO__IMPL: org.apache.iceberg.aws.s3.S3FileIO
      - CATALOG_S3_ENDPOINT: http://minio:9000
      - CATALOG_S3_ACCESS__KEY__ID: admin
      - CATALOG_S3_SECRET__ACCESS__KEY: password
      - CATALOG_S3_PATH__STYLE__ACCESS: "true"
    networks:
      - iceberg_net

  minio:
    image: minio/minio:latest
    container_name: minio
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
    ports:
      - "9001:9001"
      - "9000:9000"
    command: server /data --console-address ":9001"
    networks:
      - iceberg_net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

networks:
  iceberg_net:
    driver: bridge

```

*Documentação:* Este arquivo define nossos três serviços e como eles se conectam. O Spark é configurado para usar o `rest-catalog` e para armazenar dados no `minio`.

**3. Inicie o ambiente:**
Execute o comando abaixo no seu terminal, dentro da pasta do projeto.

```bash
docker compose up -d

```

**4. Verifique se tudo está funcionando:**

```bash
docker compose ps

```

Você deve ver os três contêineres (`spark-iceberg`, `rest-catalog`, `minio`) com o status `Up` ou `running`.

-----

### Sessão 2: Preparação dos Dados 📂

Agora, vamos criar o arquivo de dados que você forneceu.

**1. Crie a pasta `data`:**
Esta pasta será mapeada para dentro do contêiner do Spark.

```bash
mkdir -p data

```

**2. Crie o arquivo `pedidos.csv` com os dados de exemplo:**
O comando abaixo cria o arquivo e insere o conteúdo de uma só vez.

```bash
cat <<EOF > data/pedidos.csv
id_pedido;produto;valor_unitario;quantidade;data_criacao;uf;id_cliente
fdd7933e-ce3a-4475-b29d-f239f491a0e7;MONITOR;600;3;2024-01-01T22:26:32;RO;12414
fe0f547a-69f3-4514-adee-8f4299f152af;MONITOR;600;2;2024-01-01T16:01:26;SP;11750
fe4f2b05-1150-43d8-b86a-606bd55bc72c;NOTEBOOK;1500;1;2024-01-01T06:49:21;RR;1195
fe8f5b08-160b-490b-bcb3-c86df6d2e53b;GELADEIRA;2000;1;2024-01-01T04:14:54;AC;8433
feaf3652-e1bd-4150-957e-ee6c3f62e11e;HOMETHEATER;500;2;2024-01-01T10:33:09;SP;12231
feb1efc5-9dd7-49a5-a9c7-626c2de3e029;CELULAR;1000;2;2024-01-01T13:48:39;SC;2340
ff181456-d587-4abd-a0ac-a8e6e33b87d5;TABLET;1100;1;2024-01-01T21:28:47;RS;12121
ff3bc5e0-c49a-46c5-a874-3eb6c8289fd1;HOMETHEATER;500;1;2024-01-01T22:31:35;SC;6907
ff4fcf5f-ca8a-4bc4-8d17-995ecaab3110;SOUNDBAR;900;3;2024-01-01T19:33:08;RJ;9773
ff703483-e564-4883-bdb5-0e25d8d9a006;NOTEBOOK;1500;3;2024-01-01T00:22:32;RN;2044
ffe4d6ad-3830-45af-a599-d09daaeb5f75;HOMETHEATER;500;3;2024-01-01T02:55:59;MS;3846
EOF
```

**3. Comprima o arquivo:**

```bash
gzip data/pedidos.csv
```

Agora você terá o arquivo `data/pedidos.csv.gz`, pronto para ser lido pelo Spark.

-----

### Sessão 3: Ingestão e Consulta de Dados 🚀

Com o ambiente de pé e os dados prontos, vamos criar nossa tabela Iceberg e carregar os dados.

**1. Acesse o terminal SQL do Spark:**

```bash
docker exec -it spark-iceberg spark-sql
```

*Documentação:* Este comando abre uma sessão interativa de SQL dentro do contêiner do Spark.

**2. Crie um Schema (Banco de Dados):**

```sql
CREATE SCHEMA IF NOT EXISTS db;
```

*Documentação:* Schemas ajudam a organizar as tabelas dentro do catálogo.

**3. Crie a tabela Iceberg `pedidos`:**

```sql
CREATE TABLE db.pedidos (
    id_pedido STRING,
    produto STRING,
    valor_unitario DECIMAL(10, 2),
    quantidade INT,
    data_criacao TIMESTAMP,
    uf STRING,
    id_cliente BIGINT
)
USING iceberg
TBLPROPERTIES ('format-version'='2');
```

*Documentação:* Criamos a estrutura da tabela, definindo nome e tipo de cada coluna. `USING iceberg` especifica o formato e `format-version='2'` habilita recursos modernos como `updates` e `deletes`.

**4. Crie uma visão temporária sobre o arquivo CSV:**
Para facilitar a leitura, criamos uma "view" que aponta para o nosso arquivo.

```sql
CREATE OR REPLACE TEMP VIEW pedidos_raw
USING csv
OPTIONS (
  path = '/home/iceberg/data/pedidos.csv.gz',
  header = 'true',
  delimiter = ';'
);
```

**5. Insira os dados na tabela Iceberg:**
Agora, copiamos os dados da `view` para a nossa tabela definitiva.

```sql
INSERT INTO db.pedidos
SELECT
  id_pedido,
  produto,
  CAST(valor_unitario AS DECIMAL(10, 2)),
  CAST(quantidade AS INT),
  CAST(data_criacao AS TIMESTAMP),
  uf,
  CAST(id_cliente AS BIGINT)
FROM pedidos_raw;
```

*Documentação:* O `CAST` garante que os dados do CSV (que são lidos como texto) sejam convertidos para os tipos corretos definidos na tabela Iceberg.

**6. Consulte os dados:**

```sql
SELECT * FROM db.pedidos LIMIT 5;
```

Você deverá ver os dados que acabamos de inserir\!

-----

### Sessão 4: O Poder do Time Travel (Viagem no Tempo) ⏳

Vamos ver um dos recursos mais incríveis do Iceberg. Toda alteração em uma tabela cria uma nova "foto" (snapshot) dos dados, e podemos consultar qualquer foto do passado.

**1. Faça uma alteração nos dados:**
Vamos remover todos os pedidos do estado de São Paulo (SP).

```sql
DELETE FROM db.pedidos WHERE uf = 'SP';
```

**2. Consulte o estado atual:**
Observe que os dois pedidos de SP sumiram.

```sql
SELECT uf, count(*) FROM db.pedidos GROUP BY uf;
```

**3. Visualize o histórico da tabela:**
O Iceberg mantém um log de todas as operações.

```sql
SELECT * FROM db.pedidos.history;
```

*Documentação:* Você verá duas linhas. A primeira é a da inserção (`INSERT`) e a segunda é a da exclusão (`DELETE`). Anote o `snapshot_id` da primeira linha (a da inserção).

**4. Viaje no tempo\!**
Use o `snapshot_id` que você anotou para consultar a tabela como ela era *antes* do `DELETE`.

```sql
-- Substitua <SEU_SNAPSHOT_ID_DA_INSERCAO> pelo ID correto
SELECT uf, count(*) FROM db.pedidos VERSION AS OF <SEU_SNAPSHOT_ID_DA_INSERCAO> GROUP BY uf;
```

*Resultado Mágico:* Você verá os pedidos de SP de volta, pois está consultando uma foto do passado\!

-----

### Sessão 5: Evolução de Schema (Schema Evolution) 🧬

Em data lakes tradicionais, alterar uma tabela é uma tarefa complexa. Com Iceberg, é trivial.

**1. Adicione uma nova coluna:**

```sql
ALTER TABLE db.pedidos ADD COLUMN status STRING;
```

**2. Consulte a tabela e veja a nova coluna:**
Para os registros antigos, o valor será `NULL`.

```sql
SELECT id_pedido, uf, status FROM db.pedidos LIMIT 5;
```

**3. Renomeie uma coluna existente:**

```sql
ALTER TABLE db.pedidos RENAME COLUMN uf TO estado;
```

**4. Consulte novamente para ver a mudança:**
A coluna `uf` não existe mais, agora se chama `estado`.

```sql
SELECT id_pedido, estado FROM db.pedidos LIMIT 5;
```

*Documentação:* O Iceberg gerencia essas mudanças nos metadados, sem precisar reescrever os arquivos de dados antigos, o que torna a operação instantânea.

-----

### Sessão 6: Particionamento Otimizado ⚡

O particionamento acelera consultas filtrando apenas os arquivos de dados relevantes. O Iceberg faz isso de forma "oculta", sem criar pastas extras no seu data lake.

**1. Adicione uma partição baseada no tempo:**
Vamos particionar nossos dados por dia, usando a coluna `data_criacao`.

```sql
ALTER TABLE db.pedidos ADD PARTITION FIELD days(data_criacao);
```

*Documentação:* A função `days()` transforma o timestamp em uma data. O Iceberg usará essa informação para otimizar filtros por `data_criacao`, mas a estrutura da tabela para o usuário continua a mesma. Isso é chamado de **Hidden Partitioning**.

-----

### Sessão 7: Manutenção de Tabelas 🧹

Com o tempo, muitas operações podem gerar arquivos pequenos ou snapshots antigos. É uma boa prática realizar manutenções periódicas.

**1. Expirar snapshots antigos:**
Vamos remover snapshots mais antigos que 1 hora, mantendo no mínimo 1.

```sql
CALL system.expire_snapshots('db.pedidos', older_than => NOW() - INTERVAL '1' HOUR, retain_last => 1);
```

*Documentação:* Isso remove os metadados dos snapshots antigos, e uma operação futura de "limpeza" removerá os arquivos de dados que não são mais referenciados.

**2. Compactar arquivos pequenos:**
Operações de `UPDATE` ou `DELETE` podem gerar arquivos pequenos. Podemos reescrevê-los em arquivos maiores e mais otimizados.

```sql
CALL system.rewrite_data_files(table => 'db.pedidos');
```

-----

### Como Parar o Ambiente

Quando terminar seus estudos, você pode desligar todo o ambiente com um único comando:

```bash
docker-compose down
```

Aproveite para explorar e experimentar\! Altere os dados, adicione mais colunas e crie novas tabelas. A prática é a melhor forma de aprender.