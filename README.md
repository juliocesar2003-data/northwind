# Northwind

**Contexto:** Projeto de análise exploratória e estratégica sobre a base de dados Northwind, com foco em quatro áreas de melhoria operacional e financeira.  
**Entregáveis:** Pipeline ETL (Python + Polars), relatório de análise (`Projeto.pdf`)

---

## Sobre o Projeto

A Northwind é uma empresa americana que distribui produtos alimentícios para clientes em todo o mundo. Este projeto propõe quatro análises com ações imediatas recomendadas, desenvolvidas inteiramente em Polars.

O detalhamento completo de cada análise, os critérios utilizados e as ações sugeridas estão no arquivo `Projeto.pdf`.

---

## Estrutura do Repositório

```
├── categories.csv              # Categorias de produtos
├── customers.csv               # Cadastro de clientes
├── customer_customer_demo.csv  # Relação clientes x demografias
├── customer_demographics.csv   # Demografias de clientes
├── employees.csv               # Cadastro de funcionários
├── employee_territories.csv    # Relação funcionários x territórios
├── orders.csv                  # Ordens de compra (830 registros)
├── order_details.csv           # Itens por ordem (2.155 registros)
├── products.csv                # Cadastro de produtos (77 produtos)
├── suppliers.csv               # Cadastro de fornecedores
├── shippers.csv                # Transportadoras
├── territories.csv             # Territórios de atuação
├── region.csv                  # Regiões
├── us_states.csv               # Estados dos EUA
├── Projet.ipynb                # Pipeline ETL e análises (Python + Polars)
└── Projeto.pdf                 # Relatório de análise
```

> Os arquivos CSV não estão disponíveis no repositório. Para reproduzir as análises, utilize a base de dados pública Northwind, disponível em: https://github.com/pthom/northwind_psql

---

## Pipeline ETL

**Arquivo:** `Projet.ipynb`  
**Stack:** Python, Polars

### Por que Polars

Polars foi utilizado no lugar do Pandas pela sua sintaxe de encadeamento de operações — próxima ao SQL — e pelo desempenho superior em transformações com múltiplos `group_by`, `join` e `filter` encadeados, que são o padrão em todas as análises deste projeto.

### Ingestão dos Dados

Todos os arquivos CSV são carregados em um único bloco via loop, utilizando `globals()` para nomear cada DataFrame dinamicamente:

```python
folder = 'caminho/para/os/arquivos/'

names = [
    "categories", "customer_customer_demo", "customer_demographics",
    "customers", "employee_territories", "employees", "order_details",
    "orders", "products", "region", "shippers", "suppliers",
    "territories", "us_states"
]

for name in names:
    globals()[name] = pl.read_csv(f"{folder}{name}.csv", truncate_ragged_lines=True, separator=';')
```

> **Atenção:** Atualize a variável `folder` para o caminho local onde os arquivos estiverem salvos.

---

### Análises Desenvolvidas

**1. Geografia e Logística**

Cruzamento entre a tabela de clientes (`customers`) e a tabela de fornecedores (`suppliers`) por país, utilizando um `anti join` — que retorna apenas os países de clientes sem fornecedor correspondente.

Em seguida, as ordens são agrupadas por país de destino (`ship_country`), filtradas pelo top 10 em volume e cruzadas novamente com os países de fornecedores para identificar os candidatos à expansão sem cobertura local.

Resultado: Venezuela (46 ordens), Áustria (40) e México (28) lideram em volume sem fornecedor local.

**2. Estoque e Produtos**

Agrupamento de `order_details` por `product_id` com soma de `quantity`, seguido do cálculo da representação percentual de cada produto em relação ao total. Os resultados são enriquecidos com o nome do produto via `join` com a tabela `products`.

O filtro de `Representação < 1%` isola os candidatos à descontinuação. A análise retorna os 10 piores ranqueados por `sort('Representação').head(10)`.

Recomendação: descontinuar os 10 produtos com menor volume — impacto de 3,95% nas vendas com redução significativa em custos de estoque e logística.

**3. Churn**

A análise é composta por três agregações independentes, unidas por `join` ao final:

- **`ordem_mais_recente`** — data da última ordem por cliente via `pl.col("order_date").max()`
- **`media_sem_compras`** — média de dias entre compras consecutivas, calculada com `shift(1).over("customer_id")` para obter a ordem anterior por cliente, subtração de datas e `mean()` por grupo
- **`order_counts`** — contagem de ordens únicas por cliente via `n_unique()`

Os três DataFrames são unidos e filtrados pelos critérios:
- Mínimo de 5 compras (`order_count >= 5`)
- Ausência de compras há mais de 90 dias em relação à data máxima da base
- Média de tempo entre compras inferior a 90 dias — garante que o período atual de ausência é anormal para aquele cliente

> A data máxima da base (`orders["order_date"].max()`) é usada como referência de "hoje". Em produção, isso seria substituído por `date.today()`.

Resultado: 8 clientes identificados em risco, distribuídos entre UK, França, Brasil, EUA, México e Canadá.

**4. Piores Transportadoras**

Filtro de ordens com `shipped_date` preenchido, conversão de `required_date` e `shipped_date` para tipo `Date` e cálculo de `dias_de_atraso = shipped_date - required_date`. Apenas atrasos positivos são considerados (`filter > 0`).

Agrupamento por `ship_via` com três métricas: número de ordens atrasadas (`len()`), média de dias de atraso (`mean()`) e pior caso (`max()`). O resultado é enriquecido com o nome da transportadora via `join` com a tabela `shippers`.

Resultado:

| Transportadora   | Ordens atrasadas | Média de atraso | Maior atraso |
|---|---|---|---|
| Speedy Express   | 12 | 8,08 dias | 18 dias |
| United Package   | 16 | 5,56 dias | 23 dias |
| Federal Shipping | 9  | 5,56 dias | 18 dias |

---

## Como Reproduzir

1. Baixe a base de dados Northwind no link indicado acima
2. Coloque os arquivos CSV em uma pasta local
3. Atualize a variável `folder` no segundo bloco do notebook com o caminho correto
4. Execute todas as células do `Projet.ipynb`
