# On Market - modelo dimensional: fato e dimensões

Documento do **modelo dimensional** do **On Market DW**, focado na definição da **tabela fato de vendas** e das **dimensões principais**. Este arquivo assume como base:

1. Contexto e escopo em `docs/01_definicao_problema/*.md`
2. Regras de negócio em `docs/02_modelo_conceitual/01_regras_negocio.md`
3. Glossário conceitual em `docs/02_modelo_conceitual/02_glossario.md`
4. Diagrama conceitual em `docs/02_modelo_conceitual/03_diagrama_conceitual.md`
5. Explicação de escopo e cores do ERD em `docs/02_modelo_conceitual/04_erd_cores_escopo.md`

A ideia é sair do nível conceitual e consolidar a visão em **star schema**, preparando o terreno para o modelo físico em PostgreSQL.

---
## 1) Objetivo do documento

1. Descrever o **star schema** da v1, com foco em uma única tabela fato de vendas.
2. Detalhar o **grão da fato** e as colunas de medidas e chaves.
3. Especificar as dimensões necessárias para responder às **perguntas de negócio prioritárias**.
4. Registrar decisões de modelagem que serão usadas no DDL e no ETL físico.

---
## 2) Visão geral do star schema v1

O modelo dimensional v1 é centrado na tabela fato `dw.vendas`, com grão:

1. Uma linha por `produto` vendido
2. Em uma `transacao` confirmada
3. Em uma `data` de negócio

As dimensões principais são:

1. `dw.cliente` contexto de quem compra e onde está
2. `dw.produto` contexto do item vendido e sua categoria
3. `dw.distribuidor` contexto de quem atende a venda logisticamente
4. `dw.data` contexto temporal para análises sazonais e de série histórica

Dimensões futuras que permanecem no backlog:

1. `dw.categoria` como dimensão própria
2. Estrutura de avaliações, associando clientes, produtos e notas

---
## 3) Tabela fato `dw.vendas`

### 3.1 Grão da fato

O grão lógico da fato é:

1. Uma linha por combinação de `produto` e `transacao` em uma `data` de confirmação da transacao
2. Medidas armazenadas em nível de linha para permitir agregações por tempo, produto, cliente, distribuidor e categoria

Na v1, o identificador `transacao_id` ainda não aparece como coluna explícita na fato, mas o grão continua definido conceitualmente dessa forma.

### 3.2 Colunas chave

Chaves estrangeiras previstas na fato `dw.vendas`:

1. `cliente_id` referencia `dw.cliente`
2. `produto_id` referencia `dw.produto`
3. `distribuidor_id` referencia `dw.distribuidor`
4. `data_id` referencia `dw.data`

Regras associadas:

1. Todas as chaves devem ser **not null**
2. Todas as chaves devem ter correspondência em suas dimensões, garantindo integridade referencial
3. Cada linha da fato deve representar exatamente um cliente, um produto, um distribuidor e uma data

### 3.3 Medidas da fato

Medidas mínimas para a v1:

1. `quantidade_vendida`
   1. Inteiro maior que zero
   2. Representa a quantidade de unidades daquele produto naquela venda

2. `faturamento`
   1. Valor monetário em BRL
   2. Decimal com duas casas
   3. Coerente com `quantidade_vendida` multiplicada pelo preço do produto, admitindo pequenas diferenças por arredondamento ou regras simples do ETL

3. `custo_frete`
   1. Valor monetário em BRL
   2. Decimal com duas casas
   3. Maior ou igual a zero e limitado a uma fração máxima do faturamento, com limite padrão de trinta por cento na v1

Todas as medidas são **not null** na v1.

### 3.4 Regras de uso da fato

Algumas regras práticas para uso da fato `dw.vendas` em consultas:

1. Qualquer cálculo de receita, quantidade e custo de frete parte de agregações sobre `dw.vendas`
2. Métricas derivadas como `ticket_medio` e `peso_do_frete` são calculadas a partir das três medidas básicas
3. Para cortes por categoria, a v1 utiliza o atributo de categoria armazenado na dimensão de produto
4. Consultas de sazonalidade sempre devem usar `dw.data` como fonte de atributos temporais

---
## 4) Dimensão `dw.cliente`

### 4.1 Grão e chave

1. Um registro por cliente
2. Chave substituta `cliente_id` inteira
3. Dados totalmente sintéticos, sem PII real

### 4.2 Atributos

Atributos mínimos na v1:

1. `nome` nome do cliente
2. `endereco` endereço textual
3. `cidade` cidade do cliente
4. `pais` país do cliente
5. Opcionalmente, `data_cadastro` se for útil para recência, mesmo que a primeira versão das perguntas não explore isso em detalhe

### 4.3 Uso analítico

Consultas típicas:

1. Distribuição de faturamento por cidade e país
2. Mapas de calor por localidade
3. Segmentações simples por localização

A dimensão não é historizada na v1, ou seja, não há SCD, apenas o estado atual dos atributos.

---
## 5) Dimensão `dw.produto`

### 5.1 Grão e chave

1. Um registro por produto
2. Chave substituta `produto_id` inteira
3. Produto pertence a exatamente uma categoria na v1

### 5.2 Atributos

Atributos mínimos na v1:

1. `nome` nome do produto
2. `descricao` descrição resumida
3. `preco` preço atual em BRL com duas casas
4. `nome_categoria` nome da categoria do produto, armazenado como atributo da dimensão e não como dimensão própria

### 5.3 Uso analítico

Consultas típicas:

1. Faturamento por produto
2. Top produtos por quantidade e faturamento
3. Mix de vendas por categoria, usando `nome_categoria`
4. Perguntas de curva ABC por produto ou por categoria

A dimensão de produto também não é historizada na v1; o preço é considerado estático para o recorte de análise.

---
## 6) Dimensão `dw.distribuidor`

### 6.1 Grão e chave

1. Um registro por distribuidor
2. Chave substituta `distribuidor_id` inteira

### 6.2 Atributos

Atributos mínimos na v1:

1. `nome_distribuidor` nome do parceiro logístico
2. `cidade_distribuidor`
3. `pais_distribuidor`

### 6.3 Uso analítico

Consultas típicas:

1. Custo de frete por distribuidor
2. Participação de cada distribuidor por região
3. Comparação de eficiência de frete entre distribuidores, cruzando com dimensões de produto e data

---
## 7) Dimensão `dw.data`

### 7.1 Grão e chave

1. Um registro por dia do calendário
2. Chave substituta `data_id` inteira, usada como chave artificial da dimensão tempo
3. Cobertura planejada, por exemplo, de dois mil e vinte e um a dois mil e vinte e cinco

### 7.2 Atributos

Atributos mínimos na v1:

1. `data` data completa
2. `dia` componente do dia
3. `mes` componente do mês
4. `ano`
5. `dia_da_semana` representação textual do dia da semana

### 7.3 Uso analítico

Consultas típicas:

1. Faturamento por mês e por ano
2. Análises por dia da semana para identificar picos
3. Janela de tempo para métricas como últimos doze meses ou ano corrente

A dimensão de data é naturalmente estável e faz o papel de calendário padrão do DW.

---
## 8) Dimensões opcionais e extensões futuras

Embora a v1 use apenas as quatro dimensões principais, já existem decisões conceituais para extensões.

### 8.1 Dimensão `dw.categoria` futura

1. Grão previsto: uma linha por categoria de produto
2. Chave substituta `categoria_id`
3. Atributos candidatos: nome, descrição, hierarquias adicionais quando houver

Na v1, esta informação está embutida em `dw.produto` por meio do atributo `nome_categoria`. Se relatórios exigirem drill down ou histórico de categoria, a dimensão própria pode ser criada em versões futuras.

### 8.2 Estrutura de avaliações futura

1. Entidade de negócio: avaliação de produto feita por cliente
2. Pode ser implementada como uma dimensão adicional ou como uma tabela de ponte entre produto, cliente e fato de vendas
3. Permite análises de correlação entre nota média e desempenho de vendas

Esta estrutura permanece no backlog da v1 e será explicitada em documentos futuros quando o escopo incluir satisfação do cliente.

---
## 9) Rastreabilidade com o modelo conceitual

Alguns vínculos diretos entre este modelo dimensional e as regras conceituais:

1. O grão da fato `dw.vendas` segue o definido em RN zero quatro
2. As chaves estrangeiras para cliente, produto, distribuidor e data refletem RN vinte e RN vinte e quatro
3. As medidas `quantidade_vendida`, `faturamento` e `custo_frete` seguem os domínios e limites descritos em RN vinte e um, RN trinta e um e RN trinta e três
4. A decisão de manter categoria como atributo de produto na v1 está alinhada a RN quatorze e RN setenta e três
5. A ausência de `transacao_id` explícito na fato está registrada em RN setenta, com previsão de inclusão em versões posteriores

---
## 10) Critérios de aceite do modelo dimensional

O modelo dimensional descrito neste arquivo é considerado aceito quando:

1. A tabela fato `dw.vendas` implementa o grão produto por transacao por data com as três medidas básicas e quatro chaves estrangeiras
2. As dimensões `dw.cliente`, `dw.produto`, `dw.distribuidor` e `dw.data` estão criadas com, no mínimo, os atributos listados neste documento
3. Todas as perguntas de negócio essenciais mapeadas em `docs/01_definicao_problema/02_perguntas_negocio.md` podem ser respondidas exclusivamente com combinações entre `dw.vendas` e essas quatro dimensões
4. O DDL físico em `ddl/fisico/modelo_fisico.sql` respeita o desenho deste modelo dimensional, permitindo a implementação de ETL sintético e testes definidos em `tests/sql`

---
## 11) Histórico do documento

1. v0.3.0 primeira versão do modelo dimensional, com foco em fato de vendas e dimensões base
---
