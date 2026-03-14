# ☁️ Cloud Fundamentals — Python + GCP

Projeto desenvolvido como parte do curso de **Fundamentos de Nuvem** da Alura, adaptado para explorar na prática o uso do Python como ferramenta de acesso e manipulação de dados dentro do ecossistema **Google Cloud Platform (GCP)**.

A ideia aqui foi sair do "só teoria" e colocar a mão na massa: conectar no GCP pelo Python, buscar dados de um bucket no Cloud Storage, rodar consultas SQL direto no BigQuery e ainda salvar os resultados de volta na nuvem. Tudo isso dentro de um Jupyter Notebook no Google Colab.

---

## 🧰 Tecnologias e Serviços Utilizados

- **Python 3** — linguagem principal
- **Google Colab** — ambiente de execução do notebook
- **Google Cloud Storage (GCS)** — armazenamento de arquivos na nuvem
- **Google BigQuery (BQ)** — data warehouse para consultas SQL em grande escala
- **Bibliotecas:** `google-cloud-storage`, `google-cloud-bigquery`, `json`, `pandas`

---

## 📁 Estrutura do Projeto

```
Cloud_Fundaments.ipynb   ← notebook principal com todo o código
```

---

## 🔄 Fluxo do Projeto

O notebook está organizado em etapas sequenciais:

### 1. Autenticação no GCP
```python
from google.colab import auth
auth.authenticate_user()
```
Primeiro passo sempre: autenticar o usuário no Google Cloud direto pelo Colab. Sem isso, nenhum serviço GCP vai deixar a gente passar.

---

### 2. Leitura de arquivo no Cloud Storage
```python
from google.cloud import storage

client_gcs = storage.Client(project=project_id)
bucket = client_gcs.bucket(bucket_name)
blob = bucket.blob(file_name)

file_content_str = blob.download_as_text()
```
Aqui conectamos ao **GCS**, acessamos o bucket `cloudgc_fundaments` e fizemos o download do arquivo `BR.json`, que contém os **feriados nacionais do Brasil** (dados de 2016), em formato de texto.

Depois o conteúdo foi convertido de string JSON para uma lista Python usando `json.loads()`, tornando os dados manipuláveis.

---

### 3. Consulta no BigQuery — Todos os pedidos
```python
from google.cloud import bigquery

client_bq = bigquery.Client(project=project_id)

consulta_pedido = """
SELECT
    order_id, order_status, order_purchase_timestamp,
    order_estimated_delivery_date, order_delivered_customer_date
FROM `alura-studies.olist_dataset.orders`
"""

query_job = client_bq.query(consulta_pedido)
pedidos = query_job.to_dataframe()
```
Conectamos ao **BigQuery** e rodamos uma consulta SQL para trazer todos os pedidos da base de dados do dataset público da [Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce). O resultado veio como um DataFrame do Pandas — **99.441 pedidos**.

---

### 4. Consulta no BigQuery — Pedidos com atraso
```python
consulta_atrasos = """
SELECT
    order_id,
    order_estimated_delivery_date,
    order_delivered_customer_date,
    DATE_DIFF(order_delivered_customer_date, order_estimated_delivery_date, DAY) AS atraso_dias
FROM `alura-studies.olist_dataset.orders`
WHERE
    order_delivered_customer_date IS NOT NULL
    AND order_estimated_delivery_date IS NOT NULL
    AND order_delivered_customer_date > order_estimated_delivery_date
    AND DATE_DIFF(order_delivered_customer_date, order_estimated_delivery_date, DAY) > 0
ORDER BY atraso_dias DESC
"""
```
Segunda consulta, mais focada: filtramos apenas os pedidos que chegaram **depois da data estimada** e calculamos quantos dias de atraso cada um teve, usando a função `DATE_DIFF` do BigQuery. O resultado trouxe **7.827 pedidos atrasados**, ordenados do maior para o menor atraso.

> ⚠️ **Nota sobre a coluna `atraso_dias`:** a condição `order_delivered_customer_date > order_estimated_delivery_date` compara data e hora, então é possível um pedido chegar no mesmo dia (mas em hora posterior) e o `DATE_DIFF` retornar 0. Por isso, adicionamos `AND DATE_DIFF(...) > 0` para garantir que só entram pedidos com atraso real de pelo menos 1 dia.

---

### 5. Escrita de tabela no BigQuery
```python
caminho = 'alura-studies.olist_ecommerce.pedidos_atrasados'

job_config = bigquery.LoadJobConfig(
    schema=schema,
    write_disposition="WRITE_APPEND"
)

job = client_bq.load_table_from_dataframe(df_atraso, caminho, job_config=job_config)
job.result()
```
Após ter o DataFrame com os atrasos, criamos um dataset novo (`olist_ecommerce`) no BigQuery e carregamos os dados como uma nova tabela (`pedidos_atrasados`). A opção `WRITE_APPEND` faz com que, se a tabela já existir, os dados sejam adicionados ao invés de sobrescrever tudo.

---

### 6. Upload do resultado de volta pro Cloud Storage
```python
df_atraso.to_csv('dados.csv')

bucket = client_gcs.bucket(bucket_name)
blob = bucket.blob('dados/dados.csv')
blob.upload_from_filename('dados.csv')
```
Por fim, salvamos o resultado da análise como CSV localmente e fizemos o upload de volta pro bucket no GCS, armazenando no caminho `dados/dados.csv`. Fechando o ciclo: buscamos dado da nuvem, processamos, e devolvemos pra nuvem.

---

## 🐛 Correções Aplicadas

Durante a revisão do código original, foram identificados e corrigidos os seguintes pontos:

| # | Problema | Correção |
|---|----------|----------|
| 1 | `client_bq` sozinho numa célula (linha sem efeito) | Linha removida |
| 2 | `client_gcs` sozinho numa célula (linha sem efeito) | Linha removida |
| 3 | `client_gcs.get_bucket()` — método depreciado | Substituído por `client_gcs.bucket()` (consistente com o início do notebook) |
| 4 | `DATE_DIFF` podia retornar `0` com pedidos entregues no mesmo dia em hora diferente | Adicionado filtro `AND DATE_DIFF(...) > 0` na query SQL |
| 5 | Nome da coluna `atraso_medio_dias` semanticamente incorreto (não é uma média, é o atraso por pedido) | Renomeado para `atraso_dias` |

---

## ▶️ Como Executar

1. Abra o notebook no **Google Colab**
2. Certifique-se de ter acesso a um projeto GCP com BigQuery e Cloud Storage habilitados
3. Ajuste as variáveis de configuração no início:
   ```python
   project_id = 'seu-projeto-gcp'
   bucket_name = 'nome-do-seu-bucket'
   file_name = 'BR.json'
   ```
4. Execute as células em ordem — a autenticação na primeira célula vai abrir um pop-up do Google para login
5. O notebook vai ler, processar e salvar os dados automaticamente

---

## 📊 Sobre os Dados

- **BR.json** — feriados nacionais do Brasil (2016), obtido de uma API pública de feriados
- **olist_dataset.orders** — dataset de pedidos do e-commerce brasileiro [Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), disponível publicamente no BigQuery

---
