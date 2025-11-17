# On Market DW - modelo lógico (schema `dw`)

Documento do **modelo lógico** do **On Market DW**, descrevendo em nível de tabelas, colunas, tipos de dados lógicos e relacionamentos o schema `dw` usado para o Data Warehouse do On Market.

Este arquivo complementa:

* `docs/01_definicao_problema/01_contexto_on_market.md`
* `docs/01_definicao_problema/02_perguntas_negocio.md`
* `docs/02_modelo_conceitual/01_regras_negocio.md`
* `docs/02_modelo_conceitual/03_diagrama_conceitual.md`
* `docs/03_modelo_dimensional/01_fato_dimensoes.md`
* `docs/03_modelo_dimensional/02_kpis.md`
* `docs/03_modelo_dimensional/03_star_schema.md`

A ideia é sair do nível dimensional (fato + dimensões) e detalhar **tabelas, colunas e tipos lógicos**, preparando o caminho para o modelo físico em SQL.

---
## 1) Objetivo do documento

1. Descrever o **modelo lógico** do DW On Market no schema `dw`.
2. Especificar **tabelas**, **colunas**, **tipos lógicos**, **chaves primárias** e **chaves estrangeiras**.
3. Alinhar o modelo lógico ao **modelo dimensional v1** (fato `dw.vendas` e dimensões `dw.cliente`, `dw.produto`, `dw.distribuidor`, `dw.data`).
4. Servir de referência direta para a implementação do modelo físico em PostgreSQL (ou outro SGBD relacional).

---
## 2) Posição do modelo lógico na arquitetura

Linha do tempo de modelagem do On Market DW:

1. Definição do problema e perguntas de negócio
   `docs/01_definicao_problema/*.md`
2. Modelo conceitual (regras, glossário, entidades e relacionamentos)
   `docs/02_modelo_conceitual/*.md`
3. Modelo dimensional (fato, dimensões, KPIs, star schema)
   `docs/03_modelo_dimensional/*.md`
4. Modelo lógico (este documento)
   Tabelas, colunas, tipos lógicos e FKs.
5. Modelo físico
   Scripts SQL em `ddl/` (a serem criados), baseados neste arquivo.

O modelo lógico **não está preso a um SGBD específico**, mas já usa uma notação próxima de SQL (tipos como `INTEGER`, `DECIMAL`, `VARCHAR`, `DATE`) para facilitar a transição para o modelo físico.

---
## 3) Convenções de nomenclatura

1. **Schema lógico**: `dw`
2. **Tabelas de dimensão**:
   * `dw.cliente`
   * `dw.produto`
   * `dw.distribuidor`
   * `dw.data`
3. **Tabela fato**:
   * `dw.vendas`
4. **Padrões gerais**:
   * Nomes de tabelas em **minúsculas**, com underscore (`snake_case`).
   * Nomes de colunas em **minúsculas**, com underscore.
   * Chaves primárias de dimensões seguem o padrão `<entidade>_id`.
   * Chaves estrangeiras na fato usam exatamente o mesmo nome da PK da dimensão.
   * Tipos lógicos:
     * `INTEGER` para identificadores e contagens.
     * `DECIMAL(p, s)` para valores monetários.
     * `VARCHAR(n)` para textos de tamanho variável.
     * `DATE` para datas de calendário.

---
## 4) Visão geral das tabelas do schema `dw`

Resumo de alto nível (detalhes nas seções seguintes):

| Tabela         | Tipo     | Descrição                                                                                      |
|----------------|----------|------------------------------------------------------------------------------------------------|
| `dw.cliente`   | Dimensão | Clientes do portal que efetivamente realizaram compras (DW não armazena usuários sem compra). |
| `dw.produto`   | Dimensão | Produtos comercializados pelo On Market e sua categoria de negócio.                           |
| `dw.distribuidor` | Dimensão | Distribuidores responsáveis pela entrega/logística dos pedidos.                         |
| `dw.data`      | Dimensão | Calendário de datas para análise temporal (dia, mês, ano, dia da semana).                     |
| `dw.vendas`    | Fato     | Linhas de venda (produto x cliente x distribuidor x data), com quantidade, faturamento e frete.|

Modelo lógico v1 segue o star schema definido em `03_star_schema.md`, sem dimensões adicionais (categoria e avaliação permanecem como extensões futuras).

---
## 5) Dimensão `dw.cliente`

### 5.1 Descrição e grão

* Tabela: `dw.cliente`
* Grão: **um registro por cliente** que tenha pelo menos uma venda registrada na fato `dw.vendas`.
* Origem conceitual: entidade **Cliente** do modelo conceitual, filtrando apenas clientes com transação efetivada.

### 5.2 Colunas e tipos lógicos

| Coluna         | Tipo lógico   | Nulo?     | Chave | Descrição                                                                                 |
|----------------|---------------|-----------|--------|-------------------------------------------------------------------------------------------|
| `cliente_id`   | INTEGER       | NOT NULL  | PK     | Identificador técnico do cliente no DW.                                                  |
| `nome`         | VARCHAR(150)  | NOT NULL  |        | Nome do cliente (fictício na base sintética).                                            |
| `endereco`     | VARCHAR(255)  | NULL      |        | Endereço textual. Não é usado diretamente nos KPIs da v1.                                |
| `cidade`       | VARCHAR(100)  | NOT NULL  |        | Cidade do cliente, usada para análises geográficas.                                      |
| `pais`         | VARCHAR(100)  | NOT NULL  |        | País do cliente, usada para agregações mais amplas (ex.: BR x exterior).                |
| `data_cadastro`| DATE          | NULL      |        | Data de cadastro do cliente (opcional na v1, útil para análises de recência futura).     |

### 5.3 Regras específicas

1. `cliente_id` é um identificador **sem significado de negócio**, podendo ser gerado automaticamente no modelo físico.
2. A combinação (`nome`, `cidade`, `pais`) não é necessariamente única; a unicidade é garantida por `cliente_id`.
3. Em v1 não há histórico de alterações (sem SCD); o registro representa o estado atual do cliente.
4. `data_cadastro` é opcional – se ausente, o cliente ainda pode participar normalmente das análises de venda.

---
## 6) Dimensão `dw.produto`

### 6.1 Descrição e grão

* Tabela: `dw.produto`
* Grão: **um registro por produto** comercializado no On Market.
* Categoria é representada por um atributo de texto (`nome_categoria`), não como dimensão própria na v1.

### 6.2 Colunas e tipos lógicos

| Coluna          | Tipo lógico    | Nulo?     | Chave | Descrição                                                                                                  |
|-----------------|----------------|-----------|--------|------------------------------------------------------------------------------------------------------------|
| `produto_id`    | INTEGER        | NOT NULL  | PK     | Identificador técnico do produto no DW.                                                                    |
| `nome`          | VARCHAR(150)   | NOT NULL  |        | Nome comercial do produto.                                                                                |
| `descricao`     | VARCHAR(255)   | NULL      |        | Descrição resumida do produto (opcional na v1).                                                           |
| `preco`         | DECIMAL(10, 2) | NOT NULL  |        | Preço atual do produto, em BRL, usado como referência nas análises (não necessariamente preço histórico). |
| `nome_categoria`| VARCHAR(100)   | NOT NULL  |        | Categoria do produto (eletrodomésticos, eletrônicos etc.), usada para KPIs por categoria.                 |

### 6.3 Regras específicas

1. `produto_id` é chave técnica; o identificador de origem (se houver) pode ser mapeado via tabela de staging ou ETL.
2. `nome_categoria` é tratado como texto controlado, mas **não** se torna dimensão própria na v1.
3. Não há histórico de preço; `preco` representa um valor de referência e não é utilizado para reconstruir faturamento (o faturamento vem da fato).
4. Um produto participa de múltiplas vendas na fato `dw.vendas`.

---
## 7) Dimensão `dw.distribuidor`

### 7.1 Descrição e grão

* Tabela: `dw.distribuidor`
* Grão: **um registro por distribuidor** responsável por atender vendas/logística.
* Alinha-se à entidade Distribuidor do modelo conceitual.

### 7.2 Colunas e tipos lógicos

| Coluna               | Tipo lógico   | Nulo?     | Chave | Descrição                                                   |
|----------------------|---------------|-----------|--------|-------------------------------------------------------------|
| `distribuidor_id`    | INTEGER       | NOT NULL  | PK     | Identificador técnico do distribuidor no DW.               |
| `nome_distribuidor`  | VARCHAR(150)  | NOT NULL  |        | Nome do parceiro logístico.                                |
| `cidade_distribuidor`| VARCHAR(100)  | NOT NULL  |        | Cidade base do distribuidor (para análises regionais).     |
| `pais_distribuidor`  | VARCHAR(100)  | NOT NULL  |        | País do distribuidor.                                      |

### 7.3 Regras específicas

1. Em v1 não há modelagem de SLA ou nível de serviço; a dimensão é usada principalmente para análise de custo de frete por distribuidor.
2. A combinação (`nome_distribuidor`, `cidade_distribuidor`, `pais_distribuidor`) não precisa ser única.
3. Distribuidores sem vendas não aparecem em `dw.distribuidor` (o DW é orientado a fatos de vendas).

---
## 8) Dimensão `dw.data`

### 8.1 Descrição e grão

* Tabela: `dw.data`
* Grão: **um registro por dia de calendário**.
* Usada como referência temporal em todos os relatórios de vendas, sazonalidade e KPIs de calendário.

### 8.2 Colunas e tipos lógicos

| Coluna         | Tipo lógico | Nulo?     | Chave | Descrição                                            |
|----------------|------------|-----------|--------|------------------------------------------------------|
| `data_id`      | INTEGER    | NOT NULL  | PK     | Identificador técnico da data no DW.                |
| `data`         | DATE       | NOT NULL  |        | Data completa (YYYY-MM-DD).                         |
| `dia`          | INTEGER    | NOT NULL  |        | Dia do mês (1 a 31).                                |
| `mes`          | INTEGER    | NOT NULL  |        | Mês (1 a 12).                                       |
| `ano`          | INTEGER    | NOT NULL  |        | Ano calendário (ex.: 2023).                         |
| `dia_da_semana`| VARCHAR(15)| NOT NULL  |        | Nome do dia da semana (ex.: 'segunda', 'monday').   |

### 8.3 Regras específicas

1. Intervalo de datas mínimo recomendado: de 2021 a 2025 (ajustável conforme necessidade de histórico e projeções).
2. `data_id` pode ser um inteiro sequencial ou codificação baseada em `YYYYMMDD`; a escolha é feita no modelo físico.
3. A dimensão de data é estável; não há necessidade de histórico ou rastreamento de alterações.
4. Todas as consultas de tempo devem usar `dw.data`, evitando datas “cruas” em outras tabelas.

---
## 9) Tabela fato `dw.vendas`

### 9.1 Descrição e grão

* Tabela: `dw.vendas`
* Grão: **uma linha por combinação de cliente, produto, distribuidor e data em uma venda efetivada**, seguindo as regras:

  1. Venda já consolidada (pedido convertido em transação paga).
  2. Uma linha representa a venda de um produto em determinada quantidade.
  3. Não há coluna explícita de `transacao_id` na v1; o conceito de transação é incorporado no processo de carga.

### 9.2 Colunas e tipos lógicos

| Coluna             | Tipo lógico    | Nulo?     | Chave | Descrição                                                                                      |
|--------------------|----------------|-----------|--------|------------------------------------------------------------------------------------------------|
| `cliente_id`       | INTEGER        | NOT NULL  | FK     | Referência a `dw.cliente.cliente_id`.                                                          |
| `produto_id`       | INTEGER        | NOT NULL  | FK     | Referência a `dw.produto.produto_id`.                                                          |
| `distribuidor_id`  | INTEGER        | NOT NULL  | FK     | Referência a `dw.distribuidor.distribuidor_id`.                                                |
| `data_id`          | INTEGER        | NOT NULL  | FK     | Referência a `dw.data.data_id`.                                                                |
| `quantidade_vendida`| INTEGER       | NOT NULL  |        | Quantidade de unidades do produto nesta linha de venda. Deve ser estritamente maior que zero.  |
| `faturamento`      | DECIMAL(12, 2) | NOT NULL  |        | Valor monetário da venda para esta linha, em BRL.                                              |
| `custo_frete`      | DECIMAL(12, 2) | NOT NULL  |        | Custo de frete associado a esta linha; não negativo e limitado a uma fração do faturamento.    |

### 9.3 Restrições e regras de integridade

1. `cliente_id` deve existir em `dw.cliente`.
2. `produto_id` deve existir em `dw.produto`.
3. `distribuidor_id` deve existir em `dw.distribuidor`.
4. `data_id` deve existir em `dw.data`.
5. `quantidade_vendida` > 0.
6. `faturamento` ≥ 0.
7. `custo_frete` ≥ 0 e, conceitualmente, `custo_frete` ≤ 0,30 × `faturamento` na v1.
8. Em v1 **não é definida uma chave primária explícita na fato**. A identificação de linha é técnica (linha física da tabela).
   * Em versões futuras, recomenda-se introduzir uma coluna `venda_id` como chave substituta.

### 9.4 Considerações de indexação lógica

Em nível lógico, são recomendados:

1. Índices nas colunas FK:
   * `cliente_id`
   * `produto_id`
   * `distribuidor_id`
   * `data_id`
2. Índices compostos opcionais focados em uso típico de consultas:
   * (`data_id`, `produto_id`)
   * (`data_id`, `cliente_id`)
   * (`data_id`, `nome_categoria` via join com `dw.produto`)

A definição exata de índices será feita no modelo físico, considerando volume de dados e padrões reais de consulta.

---
## 10) Relacionamentos e cardinalidades

Visão lógica dos relacionamentos (espelhando o star schema):

1. `dw.cliente` 1 → N `dw.vendas`
   Um cliente participa de muitas linhas de venda.
2. `dw.produto` 1 → N `dw.vendas`
   Um produto aparece em muitas vendas.
3. `dw.distribuidor` 1 → N `dw.vendas`
   Um distribuidor atende muitas vendas.
4. `dw.data` 1 → N `dw.vendas`
   Uma data está associada a muitas vendas.

Formalmente, chaves estrangeiras lógicas:

* `dw.vendas.cliente_id` → `dw.cliente.cliente_id`
* `dw.vendas.produto_id` → `dw.produto.produto_id`
* `dw.vendas.distribuidor_id` → `dw.distribuidor.distribuidor_id`
* `dw.vendas.data_id` → `dw.data.data_id`

Não há relacionamentos diretos entre dimensões (por exemplo, `cliente` com `produto`); todos os cruzamentos passam pela tabela fato `dw.vendas`.

---
## 11) Tratamento de nulos e valores padrão

Diretrizes lógicas para nulos:

1. Chaves estrangeiras na fato (`cliente_id`, `produto_id`, `distribuidor_id`, `data_id`) são **sempre not null**.
2. Medidas (`quantidade_vendida`, `faturamento`, `custo_frete`) são **sempre not null**.
3. Dimensões:
   * Campos de identificação (`*_id`) são not null.
   * Campos usados diretamente em filtros ou agrupamentos principais (`cidade`, `pais`, `nome_categoria`) devem ser preenchidos; o ETL deve garantir valores padrão quando o atributo de origem estiver ausente.
   * Atributos opcionais (como `descricao`, `data_cadastro`) podem ser nulos, sem impacto nas métricas principais.

Valores padrão sugeridos (tratados no ETL, não no modelo lógico):

* País desconhecido: `'Desconhecido'`
* Cidade desconhecida: `'Nao_informado'`
* Categoria desconhecida: `'Sem_categoria'`

---
## 12) Mapeamento com o modelo dimensional e KPIs

Relação entre modelo lógico e elementos dimensionais:

1. A fato `dw.vendas` implementa o grão e as medidas documentadas em:
   * `03_modelo_dimensional/01_fato_dimensoes.md`
2. As dimensões `dw.cliente`, `dw.produto`, `dw.distribuidor`, `dw.data` implementam os atributos descritos em:
   * `03_modelo_dimensional/01_fato_dimensoes.md`
3. Os KPIs descritos em `03_modelo_dimensional/02_kpis.md` são calculados a partir de:
   * Somatórios e agregações sobre `dw.vendas` (`quantidade_vendida`, `faturamento`, `custo_frete`).
   * Junções com as dimensões para cortes por cliente, produto, distribuidor, data, categoria e geografia.

Exemplos de ligação:

* `faturamento_mensal`:
  Usa `dw.vendas.faturamento` + `dw.data.ano`, `dw.data.mes`.
* `participacao_categoria`:
  Usa `dw.vendas.faturamento` + `dw.produto.nome_categoria`.
* `faturamento_por_localidade`:
  Usa `dw.vendas.faturamento` + `dw.cliente.cidade`, `dw.cliente.pais`.

---
## 13) Extensões futuras (não implementadas na v1)

Itens planejados como evolução natural do modelo lógico:

1. **Dimensão de categoria própria** (`dw.categoria`):
   * Grão: uma linha por categoria.
   * Ligação: `dw.produto` passaria a referenciar `categoria_id`.
   * Benefício: facilita hierarquias de categoria e relatórios mais complexos.

2. **Dimensão de avaliação** (`dw.avaliacao` ou fato separada):
   * Permite análises de correlação entre nota média e desempenho de vendas.
   * Poderia ser modelada como fato de avaliação ou dimensão ligada a produto e cliente.

3. **Chave substituta da fato** (`venda_id`):
   * Facilita rastreabilidade de linhas e auditoria de ETL.
   * Permite implementar relacionamentos adicionais se forem necessários no futuro.

Essas extensões devem ser documentadas em versões futuras dos arquivos de modelo dimensional e lógico antes de entrarem no modelo físico.

---
## 14) Critérios de aceite do modelo lógico

O modelo lógico descrito neste documento é considerado aceito quando:

1. Todas as tabelas lógicas (`dw.cliente`, `dw.produto`, `dw.distribuidor`, `dw.data`, `dw.vendas`) estão definidas com:
   * Nome da tabela.
   * Lista de colunas.
   * Tipo lógico.
   * Indicadores de PK/FK e nulidade.
2. Os relacionamentos entre fato e dimensões são suficientes para calcular todos os KPIs descritos em `03_modelo_dimensional/02_kpis.md`.
3. Não há divergência entre:
   * Grão da fato descrito no modelo dimensional.
   * Grão da fato assumido neste modelo lógico.
4. O modelo físico em SQL pode ser derivado deste documento sem decisões adicionais de modelagem (apenas escolhas de tipos físicos e índices).
5. O time de dados valida que as principais consultas de referência (faturamento mensal, top produtos, frete por categoria, ticket médio aproximado) podem ser expressas diretamente contra este schema.

---
## 15) Histórico do documento

1. v0.4.0 primeira versão do modelo lógico do On Market DW (fato `dw.vendas` e dimensões `dw.cliente`, `dw.produto`, `dw.distribuidor`, `dw.data`).
---
