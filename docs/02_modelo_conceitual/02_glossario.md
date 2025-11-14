# On Market - glossário (modelo conceitual)

Glossário oficial do **On Market DW**. Define termos de negócio e técnicos usados na modelagem **conceitual** e nas camadas **dimensional, lógica e física**. Serve para alinhar linguagem entre documentação (`docs/`), DDL (`ddl/`) e ETL (`etl/`).

> Convenções: nomes em **snake_case** no texto; moeda **BRL**; dados **sintéticos**; foco em **transações confirmadas**.

---

## 1) Como usar este glossário

- **Termo** → definição curta, **escopo** (onde se aplica), **cálculo** (quando for métrica) e **mapeamento físico v1** (colunas/tabelas).
- Quando houver diferença entre o nome conceitual e o nome físico v1, o mapeamento fica explícito.
- Referências cruzadas apontam para arquivos do repositório (por ex.: `etl/sql/processo_etl_vendas.sql`).

---

## 2) Entidades e chaves (nível conceitual)

| Termo | Definição | Escopo | Mapeamento físico v1 (PostgreSQL) | Observações |
|---|---|---|---|---|
| **cliente** | Pessoa que compra no e-commerce. | Dimensão | `dw.cliente` (`clienteid`, `nome`, `endereco`, `cidadecliente`, `paiscliente`) | Sem PII real (dados sintéticos). |
| **produto** | Item vendido no e-commerce. | Dimensão | `dw.produto` (`produtoid`, `nome`, `descricao`, `preco`, `nomecategoria`) | Categoria como **atributo** no v1. |
| **categoria** | Classificação do produto (ex.: “Categoria 1..5”). | Atributo/Dimensão (backlog) | `dw.produto.nomecategoria` (v1) | Pode virar dimensão própria (snowflake) em versão futura. |
| **distribuidor** | Quem transporta/entrega. | Dimensão | `dw.distribuidor` (`distribuidorid`, `nomedistribuidor`, `cidadedistribuidor`, `paisdistribuidor`) | Relacionado apenas após transação confirmada. |
| **data** | Dia do calendário analítico. | Dimensão | `dw.data` (`dataid`, `data`, `dia`, `mes`, `ano`, `diadasemana`) | Cobertura 2021–2025 (ajustável). |
| **venda** | Registro analítico de **transação confirmada**. | Fato | `dw.vendas` (FKs e medidas) | Grão: **produto × transação × data**. |
| **transação** | Confirmação financeira do pedido. | Negócio | — | Gera a venda; **pedido ≠ venda**. |
| **pedido** | Intenção de compra antes do pagamento. | Negócio (fora v1) | — | Fora do escopo v1 (não está na fato). |

**Chaves (conceitual → físico v1):**
- `cliente_id` → `dw.vendas.clienteid` → `dw.cliente.clienteid`
- `produto_id` → `dw.vendas.produtoid` → `dw.produto.produtoid`
- `distribuidor_id` → `dw.vendas.distribuidorid` → `dw.distribuidor.distribuidorid`
- `data_id` → `dw.vendas.dataid` → `dw.data.dataid`

---

## 3) Atributos de dimensões (essenciais no v1)

### 3.1 Dimensão **cliente**
- **nome** → `dw.cliente.nome`
- **endereco** → `dw.cliente.endereco`
- **cidade** → `dw.cliente.cidadecliente`
- **pais** → `dw.cliente.paiscliente`

### 3.2 Dimensão **produto**
- **nome** → `dw.produto.nome`
- **descricao** → `dw.produto.descricao`
- **preco** (BRL, 2 casas) → `dw.produto.preco`
- **categoria** → `dw.produto.nomecategoria` (v1)

### 3.3 Dimensão **distribuidor**
- **nome** → `dw.distribuidor.nomedistribuidor`
- **cidade** → `dw.distribuidor.cidadedistribuidor`
- **pais** → `dw.distribuidor.paisdistribuidor`

### 3.4 Dimensão **data**
- **data** (YYYY-MM-DD) → `dw.data.data`
- **dia** (1–31) → `dw.data.dia`
- **mes** (1–12) → `dw.data.mes`
- **ano** (gregoriano) → `dw.data.ano`
- **dia_da_semana** (string) → `dw.data.diadasemana`

---

## 4) Medidas (fato vendas)

| Medida | Definição | Unidade / Tipo | Mapeamento físico v1 | Regras |
|---|---|---|---|---|
| **quantidade_vendida** | Qtde de unidades do produto na transação e data. | Inteiro | `dw.vendas.quantidadevendida` | **> 0** |
| **faturamento** | Valor total monetário da venda (produto × qtde; com arredondamento). | BRL (decimal, 2) | `dw.vendas.faturamento` | **≥ 0** |
| **custo_frete** | Valor monetário do frete associado à venda. | BRL (decimal, 2) | `dw.vendas.custofrete` | **≥ 0** e **≤ 30%** do faturamento (v1) |

**Observação:** faturamento deve guardar coerência com `quantidade_vendida × preço` (tolerância de arredondamento).

---

## 5) Métricas derivadas (negócio/relatório)

| Métrica | Fórmula (conceitual) | Observações |
|---|---|---|
| **ticket_medio** | `faturamento_total / numero_de_transacoes` | v1 pode usar **proxy** por ausência de `transacao_id`. |
| **peso_do_frete** | `custo_frete_total / faturamento_total` | Indicador percentual (0–1). |
| **participacao_categoria** | `faturamento_da_categoria / faturamento_total` | “Mix” de vendas por categoria. |
| **top_n_produtos** | ordenação por `faturamento` ou `quantidade_vendida` | Definir N (ex.: 10) por contexto de relatório. |

---

## 6) Dimensão tempo: hierarquias e cortes

- **Níveis**: `dia` → `mes` → `ano`; adicional: `dia_da_semana`.
- **Cortes comuns**: mensal, anual, por dia da semana, janela móvel.
- **Janelas sugeridas**: últimos 12 meses; YTD (year-to-date).

---

## 7) Termos de negócio (desambiguação)

| Termo | Significado neste projeto | Não confundir com |
|---|---|---|
| **pedido** | intenção de compra **antes** do pagamento | **venda/transação** |
| **transação** | confirmação financeira; gatilho de registro na fato | **pedido** |
| **venda** | linha analítica na fato (`produto × transação × data`) | “pedido enviado” sem pagamento |
| **categoria** | classificação do produto (atributo v1) | dimensão normalizada (somente em versões futuras) |
| **distribuidor** | quem entrega após confirmação | fornecedor, fabricante |

---

## 8) ETL e qualidade de dados (termos oficiais)

| Termo | Definição | Referência |
|---|---|---|
| **dados sintéticos** | Dados de teste gerados por funções PL/pgSQL (sem PII). | `etl/sql/*` |
| **idempotência controlada** | Reprocessar após limpar/zerar tabelas para repetir resultados. | pipeline manual v1 |
| **integridade referencial** | FKs da fato **sempre** existem nas dimensões. | DDL + checagens em `tests/sql` |
| **faixas/ranges** | Domínios válidos para medidas (ex.: `custo_frete ≤ 0.3 × faturamento`). | regras v1 |
| **seed aleatório** | Semente opcional para reprodutibilidade da geração. | (backlog v1) |

---

## 9) Modelagem dimensional (terminologia)

| Termo | Definição | Observação |
|---|---|---|
| **star schema** | Fato central com dimensões denormalizadas ao redor. | Escolha v1 pela simplicidade. |
| **fato** | Tabela de métricas com FKs para dimensões. | `dw.vendas`. |
| **dimensão** | Tabela de contexto (quem, o quê, onde, quando). | `cliente`, `produto`, `distribuidor`, `data`. |
| **grão (grain)** | Nível de detalhe da linha da fato. | **produto × transação × data**. |
| **SCD** | Slowly Changing Dimensions (histórico de atributos). | Não aplicado em v1 (exceto calendário). |
| **snowflake** | Normalização adicional de dimensões. | Possível para **categoria** em v2+. |

---

## 10) Performance e físico (vocabulário)

| Termo | Definição | Nota v1 |
|---|---|---|
| **índice em FK** | Índice nas chaves estrangeiras da fato para acelerar joins. | Recomendado. |
| **índice composto** | Índice em (`data_id`, `produto_id`) para agregações por mês/produto. | Opcional. |
| **particionamento** | Divisão física por período (mês/ano) na fato. | Backlog. |
| **ANALYZE/VACUUM** | Estatísticas e manutenção do PostgreSQL. | Conforme volumetria. |

---

## 11) Mapa termo → tabelas/colunas (físico v1)

- **Dimensões**
  - `cliente` → `dw.cliente (clienteid, nome, endereco, cidadecliente, paiscliente)`
  - `produto` → `dw.produto (produtoid, nome, descricao, preco, nomecategoria)`
  - `distribuidor` → `dw.distribuidor (distribuidorid, nomedistribuidor, cidadedistribuidor, paisdistribuidor)`
  - `data` → `dw.data (dataid, data, dia, mes, ano, diadasemana)`

- **Fato**
  - `vendas` → `dw.vendas (clienteid, produtoid, distribuidorid, dataid, quantidadevendida, faturamento, custofrete)`

> **Nota de nomenclatura:** apesar de o físico v1 usar nomes com capitalização mista, a documentação do repositório padroniza **snake_case**. Em uma revisão futura, o DDL poderá ser migrado para snake_case completo.

---

## 12) Exemplos de uso (expressões/metadados)

- **Ticket médio (mensal):**
  - `ticket_medio_mensal = sum(faturamento) / count(distinct transacao_id)` *(v2 com transação)*
  - **Proxy v1:** `sum(faturamento) / count(*)` em um nível coerente (ex.: por `produto_id` + `data_id`), com ressalva na documentação.

- **Peso do frete (mensal):**
  `peso_do_frete = sum(custo_frete) / sum(faturamento)`

- **Top N produtos por faturamento (mês):**
  ordenar `sum(faturamento)` desc e selecionar N.

---

## 13) Sinônimos e anti-definições

| Preferir | Evitar | Por quê |
|---|---|---|
| **transação** | “compra” (ambíguo) | “Compra” pode incluir pedidos não pagos. |
| **venda (confirmada)** | “pedido” | Pedido é intenção; não entra na fato v1. |
| **categoria do produto** | “departamento” (sem mapeamento) | Usar termos com mapeamento físico. |
| **custo de frete** | “frete” (sem valor monetário claro) | Deixar explícito que é **valor em BRL**. |

---

## 14) Referências cruzadas

- **Definição do problema:** `docs/01_definicao_problema/`
- **Regras de negócio:** `docs/02_modelo_conceitual/01_regras_negocio.md`
- **Modelo físico:** `ddl/fisico/modelo_fisico.sql`
- **ETL sintético:** `etl/sql/*.sql`
- **Testes (smoke/performance):** `tests/sql/*.sql`

---

## 15) Histórico do documento

- **v0.2.0** — primeira versão do glossário conceitual (entidades, medidas, métricas, desambiguação, mapeamento físico v1).

