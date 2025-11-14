# On Market - regras de negócio (modelo conceitual)

Documento de **regras de negócio** do estudo de caso **On Market DW**. Estas regras orientam a modelagem **conceitual** e a tradução para os modelos **dimensional, lógico e físico**. Cada regra possui um identificador (`RN-XX`) para rastreabilidade com DDL, ETL e testes.

> Escopo-alvo desta versão (v1): foco em **transações efetivadas** de e-commerce B2C, com **star schema** simples para vendas, operação (frete) e cortes por produto, cliente, data e distribuidor. Dados **sintéticos** via funções PL/pgSQL.

---

## 1) Objetivo e princípios

- **RN-01 (Propósito):** o DW deve responder perguntas sobre **vendas, operação e cliente** de forma consistente e reprodutível.
- **RN-02 (Simplicidade primeiro):** priorizar um **grão único** e dimensões essenciais; extensões ficam no backlog.
- **RN-03 (Reprodutibilidade):** todos os resultados devem ser reproduzíveis localmente com **Docker + PostgreSQL** e scripts versionados.

---

## 2) Grão analítico e evento de negócio

- **RN-04 (Grão da fato):** a tabela fato representa **uma linha por _produto_ vendido em uma _transação_ em uma _data_**.
- **RN-05 (Evento gatilho):** apenas **transações confirmadas** (pagas) geram linhas na fato (`pedido ≠ venda`).
- **RN-06 (Unicidade conceitual):** uma mesma combinação (`produto`, `transação`, `data`) não deve se repetir na fato.

> Observação: nesta v1 não modelamos `transacao_id` explicitamente; ver backlog em **RN-70**.

---

## 3) Escopo-in (v1) e fora-do-escopo

- **RN-07 (Escopo-in):** vendas B2C; métricas de **quantidade vendida, faturamento, custo de frete**; cortes por **produto, cliente, data, distribuidor**.
- **RN-08 (Fora v1):** multi-seller completo (comissão/repasse), impostos detalhados, múltiplas moedas, preço histórico, SLA logístico e avaliações; ver backlog.

---

## 4) Dimensões (conceitual)

### 4.1 Dimensão Cliente
- **RN-09 (Identificação):** cada cliente possui um identificador único interno `cliente_id`.
- **RN-10 (Atributos mínimos):** nome, endereço, cidade, país. **Sem PII real** (dados sintéticos).
- **RN-11 (Localização):** cidade e país devem ser consistentes (conjuntos válidos pré-definidos no ETL sintético).

### 4.2 Dimensão Produto
- **RN-12 (Identificação):** `produto_id` único.
- **RN-13 (Atributos mínimos):** nome, descrição, **preço atual**, categoria (como **atributo denormalizado** no star schema v1).
- **RN-14 (Categoria 1:N):** cada produto pertence a **exatamente uma** categoria no v1.

### 4.3 Dimensão Distribuidor
- **RN-15 (Identificação):** `distribuidor_id` único.
- **RN-16 (Atributos mínimos):** nome, cidade, país.
- **RN-17 (Relação com vendas):** distribuidor participa **apenas após confirmação** da transação.

### 4.4 Dimensão Data
- **RN-18 (Calendário):** `data_id` representa um dia; atributos: data, dia, mês, ano, dia_da_semana.
- **RN-19 (Janelas):** deve cobrir ao menos **2021-01-01 a 2025-12-31** (ajustável).

---

## 5) Fato Vendas (conceitual)

- **RN-20 (FKs obrigatórias):** `cliente_id`, `produto_id`, `distribuidor_id`, `data_id` devem apontar para dimensões válidas.
- **RN-21 (Medidas mínimas):**
  - `quantidade_vendida` > 0 (inteiro)
  - `faturamento` ≥ 0 (decimal, 2 casas)
  - `custo_frete` ≥ 0 **e** `custo_frete` ≤ `faturamento` × **limite_frete** (ver RN-33)
- **RN-22 (Sem nulos):** nenhuma medida ou FK pode ser nula.
- **RN-23 (Moeda):** faturamento e custo_frete estão em **BRL** (sem multi-moeda no v1).

---

## 6) Chaves, identificadores e integridade

- **RN-24 (PKs dimensões):** chaves substitutas **inteiras** (`serial` no físico v1) e únicas.
- **RN-25 (Integridade referencial):** 100% das FKs da fato devem existir nas dimensões correspondentes.
- **RN-26 (Unicidade semântica):** não pode haver duplicidade no triplo (`produto`, `transação`, `data`) por design conceitual.

---

## 7) Regras temporais e calendário

- **RN-27 (Data de negócio):** a data da linha na fato deve refletir a **data de confirmação** da transação (não a data do pedido).
- **RN-28 (Sazonalidade):** o calendário deve permitir cortes por **mês** e **dia_da_semana** para análises sazonais.
- **RN-29 (Consistência temporal):** `data_id` deve existir na dimensão data para todas as linhas da fato.

---

## 8) Regras de qualidade de dados

- **RN-30 (Not null):** FKs e medidas da fato são obrigatórias.
- **RN-31 (Domínios):**
  - `quantidade_vendida` ∈ **[1, 1000]** (faixa v1; ajustável no ETL)
  - `faturamento` ≥ 0 e arredondado a **2 casas**
  - `custo_frete` ≥ 0
- **RN-32 (Coerência aritmética):** `faturamento` deve ser compatível com `quantidade_vendida × preço_do_produto` (tolerância de arredondamento v1).
- **RN-33 (Limite do frete):** `custo_frete` ≤ **30%** de `faturamento` por linha (limite_frete padrão v1 = 0,30).
- **RN-34 (Cardinalidade):** PKs das dimensões não podem se repetir; FKs devem respeitar cardinalidade 1:N com a fato.

---

## 9) Regras de cálculo (KPIs & métricas)

- **RN-35 (Básicas):**
  - `qtde_total` = Σ `quantidade_vendida`
  - `faturamento_total` = Σ `faturamento`
  - `custo_frete_total` = Σ `custo_frete`
- **RN-36 (Ticket médio):** `ticket_medio` = `faturamento_total` / **número_de_transacoes**
  > **Nota:** sem `transacao_id` no v1, o **proxy** pode ser `faturamento_total` / `n_notas` (ou agregação coerente a definir). Ver backlog RN-70.
- **RN-37 (Peso do frete):** `peso_do_frete` = `custo_frete_total` / `faturamento_total`
- **RN-38 (Participação por categoria):** `%_categoria` = `faturamento_da_categoria` / `faturamento_total`.
- **RN-39 (Top N):** ranking por `faturamento` ou `quantidade_vendida` dentro de uma janela (p.ex., mês).

---

## 10) Regras do ETL sintético (geração de dados)

- **RN-40 (Clientes):** gerar `N` clientes com **cidades/países válidos**; endereços fictícios.
- **RN-41 (Produtos):** gerar `M` produtos com **preço em [0, 1000]** (duas casas), descrição simples e **categoria** `Categoria 1..5`.
- **RN-42 (Distribuidores):** gerar `D` distribuidores com cidades/países válidos.
- **RN-43 (Calendário):** preencher datas dia a dia entre os limites (RN-19).
- **RN-44 (Vendas):** para cada linha:
  - escolher **aleatoriamente** cliente/produto/distribuidor/data válidos
  - `quantidade_vendida` em `[1, 100]`
  - `faturamento` coerente com `quantidade × preço` (com ruído controlado)
  - `custo_frete` ~ **U(0, 10% × faturamento)**, respeitando RN-33
- **RN-45 (Determinismo opcional):** permitir **seed** para reprodutibilidade exata, quando aplicável.
- **RN-46 (Sem PII):** **proibido** inserir dados pessoais reais.

---

## 11) Segurança e privacidade

- **RN-47 (Sem dados sensíveis):** somente dados **fictícios**.
- **RN-48 (Segredos):** `.env` e credenciais **não** devem ser versionados.
- **RN-49 (Exposição pública):** documentação e scripts são públicos; nenhum endpoint de produção é exposto.

---

## 12) Padrões e nomenclatura

- **RN-50 (Snake case):** nomes de arquivos/pastas/objetos em **snake_case**.
- **RN-51 (Tipos e EOL):** `.sql`, `.md`, `.yml` com **LF**; binários marcados; compatível com `.gitattributes`.

---

## 13) Regras de exceção e borda (edge cases)

- **RN-52 (Faturamento zero):** permitido **apenas** quando `quantidade_vendida > 0` e for caso especial (ex.: brinde); **padrão v1:** **não gerar** faturamento zero.
- **RN-53 (Frete zero):** permitido (promoção de frete grátis); ainda assim respeitar RN-31.
- **RN-54 (Preço zero):** **não permitido** para produtos (v1).
- **RN-55 (Categoria nula):** **não permitido** (v1 exige categoria por RN-14).
- **RN-56 (Datas fora do calendário):** **não permitido**; ETL deve garantir RN-29.

---

## 14) Rastreabilidade: onde cada regra é implementada

- **Dimensões / Fato (DDL):** `ddl/fisico/modelo_fisico.sql` → RN-09..RN-26, RN-29, RN-51
- **Geração ETL (PL/pgSQL):** `etl/sql/processo_etl_clientes.sql` → RN-40, RN-46
  `etl/sql/processo_etl_produtos.sql` → RN-41, RN-54, RN-55
  `etl/sql/processo_etl_distribuidor.sql` → RN-42
  `etl/sql/processo_etl_data.sql` → RN-19, RN-43
  `etl/sql/processo_etl_vendas.sql` → RN-04..RN-06, RN-20..RN-23, RN-31..RN-33, RN-44, RN-52..RN-56
- **Qualidade / Testes:** `tests/sql/smoke_queries.sql` e `tests/sql/performance_queries.sql` → RN-30..RN-39

---

## 15) Critérios de aceite (checks mínimos)

- **RN-57 (Integridade):** 100% das FKs válidas (fato → dimensões).
- **RN-58 (Domínios):** 0 linhas fora de faixa (`quantidade`, `faturamento`, `custo_frete`).
- **RN-59 (Coerência frete):** 0 linhas com `custo_frete` > 30% de `faturamento`.
- **RN-60 (Calendário):** 0 linhas com `data_id` inexistente.
- **RN-61 (Soma mensal):** somatórios por mês batem com o total do período.
- **RN-62 (Ranking estável):** top N reprodutível nos mesmos dados.

> Exemplos de queries para `tests/sql/smoke_queries.sql`:
```sql
-- FK válidas
SELECT COUNT(*) AS fks_invalidas
FROM dw.vendas v
LEFT JOIN dw.cliente c   ON c.clienteid = v.clienteid
LEFT JOIN dw.produto p   ON p.produtoid = v.produtoid
LEFT JOIN dw.distribuidor d ON d.distribuidorid = v.distribuidorid
LEFT JOIN dw.data dt     ON dt.dataid = v.dataid
WHERE c.clienteid IS NULL
   OR p.produtoid IS NULL
   OR d.distribuidorid IS NULL
   OR dt.dataid IS NULL;

-- Faixas e frete
SELECT
  SUM(CASE WHEN quantidadevendida <= 0 THEN 1 ELSE 0 END) AS qtde_invalida,
  SUM(CASE WHEN faturamento < 0 THEN 1 ELSE 0 END) AS faturamento_invalido,
  SUM(CASE WHEN custofrete < 0 OR custofrete > faturamento * 0.30 THEN 1 ELSE 0 END) AS frete_invalido
FROM dw.vendas;

-- Soma mensal
SELECT dt.ano, dt.mes,
       SUM(v.quantidadevendida) AS qtde,
       SUM(v.faturamento) AS faturamento,
       SUM(v.custofrete) AS frete
FROM dw.vendas v
JOIN dw.data dt ON dt.dataid = v.dataid
GROUP BY dt.ano, dt.mes
ORDER BY dt.ano, dt.mes;
```

---
## 16) Desempenho (conceitual → físico)

**RN-63 (Índices FK):** prever índices nas FKs da fato para joins (físico).

**RN-64 (Índice composto):** opcional em `(data_id, produto_id)` para agregações comuns.

**RN-65 (Particionamento):** opcional por mês/ano em `dw.vendas` (backlog físico).

---
## 17) Governança e versionamento

**RN-66 (Commits):** mensagens curtas com prefixos (`docs`, `feat`, `chore`, `fix`).

**RN-67 (Tags):** marcar marcos (ex.: `v0.1.0` para definição do problema).

**RN-68 (Reprodutibilidade):** `.gitattributes` e `.gitignore` padronizam EOL e evitam arquivos sensíveis.

---
## 18) Limitações assumidas (v1)

**RN-69 (Preço estático):** preço do produto não histórico; análises temporais de preço ficam limitadas.

**RN-70 (Sem transacao_id):** ausência deste identificador limita métricas “por transação” (ver backlog).

**RN-71 (Sem impostos e custo do produto):** margem não é calculada nesta fase.

---
## 19) Backlog de evolução das regras (v2+)

**RN-72 (transacao_id):** incluir na fato para métricas por transação (ticket real, itens/transação).

**RN-73 (Categoria normalizada):** promover categoria a dimensão própria (snowflake leve/SCD).

**RN-74 (Preço histórico):** tabela de vigência de preço (SCD/temporal) para elasticidade.

**RN-75 (Entrega/OTIF):** modelar expedição/entrega e KPIs logísticos.

**RN-76 (Avaliações):** ativar dimensão/ponte para correlação com vendas.

**RN-77 (Marketplace):** vendedor, oferta, preço-oferta, comissionamento/repasse.

---
## 20) Glossário rápido (conceitual)

**Transação:** confirmação financeira do pedido (evento que gera a venda).

**Pedido:** intenção; só vira venda quando confirmado (fora da fato v1).

**Grão:** nível de detalhe da linha na fato (produto × transação × data).

**Star schema:** fato central com dimensões ao redor.

**SCD:** Slowly Changing Dimension; não aplicado nesta v1 (exceto calendário natural).

**Ticket médio:** faturamento dividido por número de transações (proxy no v1).

---
## 21) Referências cruzadas (onde encontrar)

**Definição do problema:** `docs/01_definicao_problema/`

**Este arquivo (regras):** `docs/02_modelo_conceitual/01_regras_negocio.md`

**DDL físico:** `ddl/fisico/modelo_fisico.sql`

**ETL sintético:** `etl/sql/*.sql`

**Testes:** `tests/sql/*.sql`
---
