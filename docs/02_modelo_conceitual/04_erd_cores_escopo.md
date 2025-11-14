# On Market - diagrama conceitual, escopo e cores

Documento que explica em detalhe o diagrama conceitual do **On Market DW** e o significado das **cores** usadas para cada entidade. Este arquivo complementa:

- `docs/01_definicao_problema/contexto_negocio.md`
- `docs/01_definicao_problema/02_perguntas_negocio.md`
- `docs/01_definicao_problema/03_escopo.md`
- `docs/02_modelo_conceitual/01_regras_negocio.md`
- `docs/02_modelo_conceitual/02_glossario.md`
- `docs/02_modelo_conceitual/03_diagrama_conceitual.md` (arquivo que contém apenas o diagrama Mermaid)

A ideia é deixar explícito como o diagrama traduz a jornada de e-commerce e como cada cor se conecta ao **escopo da v1**, às **opções futuras** e ao **backlog**.

---
## 1. Como ler o diagrama

O diagrama em `03_diagrama_conceitual.md` é um **ER (Entidade-Relacionamento)** com:

- Entidades em caixa alta (`CLIENTE`, `PRODUTO`, `VENDA` etc.).
- Atributos listados dentro de cada entidade (tipo e indicação de PK).
- Relacionamentos com cardinalidade usando a notação do Mermaid:
  - `||` lado obrigatório com cardinalidade 1.
  - `o{` lado opcional com cardinalidade N (zero ou muitos).
  - `|o` lado opcional com cardinalidade 0 ou 1.
- Estilo por cor para indicar escopo:
  - Verde: núcleo da v1.
  - Âmbar: opcional, para análises mais avançadas.
  - Preto: fora da v1 (backlog), mas importante para a jornada de negócio.

Este documento não altera o modelo, apenas explica o racional.

---
## 2. Núcleo analítico v1 (entidades em verde)

As entidades em **verde** (`CLIENTE`, `PRODUTO`, `DATA`, `DISTRIBUIDOR`, `TRANSACAO`, `VENDA`) representam o **coração do domínio analítico** da v1. São as peças que explicam:

- Quem compra.
- O que é comprado.
- Quando acontece.
- Quem entrega (operador logístico).
- Qual transação financeira confirma a venda.
- Qual é o registro analítico de venda usado no DW.

### 2.1 CLIENTE (verde)

Entidade de negócio que representa quem compra no e-commerce.

Atributos principais no diagrama:

- `cliente_id PK`: identificador único interno do cliente.
- `nome`: nome do cliente (dados sintéticos).
- `endereco`, `cidade`, `pais`: localização básica, usada para análises geográficas.
- `data_cadastro`: data em que o cliente entrou na base.

Relacionamento:

- `CLIENTE ||--o{ VENDA : "realiza"`
  Interpretação:
  - Um cliente pode **realizar muitas vendas**.
  - Cada venda está associada a **exatamente um cliente**.
- `CLIENTE ||--o{ AVALIACAO : "faz"`
  Quando avaliações forem usadas, um cliente pode fazer várias avaliações.

Conexão com o DW:

- No star schema v1, esta entidade corresponde à dimensão `dw.cliente`.
- Responde perguntas como:
  - “Qual o faturamento por cidade/país do cliente?”
  - “Quais cidades concentram maior receita?”

### 2.2 PRODUTO (verde)

Entidade que representa o item vendido.

Atributos principais:

- `produto_id PK`
- `nome`
- `descricao`
- `preco`: preço atual em BRL.
- `nome_categoria`: nome da categoria associada ao produto (atributo, não dimensão própria na v1).

Relacionamentos:

- `PRODUTO ||--o{ VENDA : "aparece_em"`
  Cada produto pode aparecer em muitas linhas de vendas; cada linha de venda referencia um produto.
- `CATEGORIA ||--o{ PRODUTO : "agrupa"`
  Uma categoria agrupa vários produtos (relação 1:N).
- `PRODUTO ||--o{ AVALIACAO : "recebe"`
  Um produto pode receber muitas avaliações.

Conexão com o DW:

- Vira `dw.produto` com preço e categoria como atributos.
- Permite análises:
  - “Quanto vendemos por produto e por categoria?”
  - “Quais produtos são top N em faturamento/quantidade?”

### 2.3 DATA (verde)

Entidade que representa a dimensão tempo.

Atributos:

- `data_id PK`
- `data`: data completa.
- `dia`, `mes`, `ano`
- `dia_da_semana`

Relacionamento:

- `DATA ||--o{ VENDA : "ocorre_em"`
  Cada venda é registrada em **uma data** (data da confirmação da transação). Uma mesma data pode ter várias vendas.

Conexão com o DW:

- Vira a dimensão `dw.data`.
- Atende perguntas como:
  - “Qual o faturamento por mês/ano?”
  - “Como variam as vendas por dia da semana?”

### 2.4 DISTRIBUIDOR (verde)

Entidade que representa quem realiza a entrega (transportadora, parceiro logístico etc.).

Atributos:

- `distribuidor_id PK`
- `nome_distribuidor`
- `cidade_distribuidor`
- `pais_distribuidor`

Relacionamento:

- `DISTRIBUIDOR ||--o{ VENDA : "atende"`
  Cada venda é atendida por **um distribuidor**; um distribuidor pode atender muitas vendas.

Conexão com o DW:

- Vira a dimensão `dw.distribuidor`.
- Permite análises como:
  - “Qual o custo de frete por distribuidor?”
  - “Qual distribuidor atende qual região/cidade?”

### 2.5 TRANSACAO (verde)

Entidade de negócio que representa a **confirmação financeira** de um pedido (pagamento aprovado).

Atributos:

- `transacao_id PK`
- `data_transacao`: data de confirmação do pagamento.
- `valor_total`: valor total da transação.
- `meio_pagamento`: cartão, boleto, pix etc.
- `status`: aprovado, cancelado, rejeitado etc.

Relacionamentos:

- `PEDIDO |o--|| TRANSACAO : "gera"`
  Um pedido pode gerar **zero ou uma** transação; cada transação está ligada a **um pedido**.
  Isso reforça a regra de negócio: pedido é intenção, transação é confirmação.
- `TRANSACAO ||--o{ VENDA : "detalha"`
  Uma transação pode ser detalhada em várias linhas de venda (uma por produto na cesta).

Conexão com o DW:

- Conceitualmente, `TRANSACAO` define o **evento gatilho** da fato:
  - Apenas transações confirmadas geram vendas (regra “pedido ≠ venda”).
- Na v1 física, o DW não possui ainda uma dimensão ou coluna explícita `transacao_id` na fato (está no backlog).
- Mesmo assim, a entidade aparece em verde porque:
  - Define o **grão** da fato: produto × transação × data.
  - É base para futuras métricas “por transação” (ticket real, itens por transação).

### 2.6 VENDA (verde)

Entidade que representa o **registro analítico** da venda, em nível de produto.

Atributos:

- `venda_id PK`: identificador interno da linha de venda.
- `quantidade_vendida`: unidades do produto naquela venda.
- `faturamento`: valor monetário da venda do produto.
- `custo_frete`: valor de frete imputado àquela linha.

Relacionamentos:

- `CLIENTE`, `PRODUTO`, `DATA`, `DISTRIBUIDOR`, `TRANSACAO` todos apontam para `VENDA` com `||--o{`, compondo o grão:
  - Cada venda é:
    - de um cliente,
    - de um produto,
    - em uma data,
    - atendida por um distribuidor,
    - vinculada a uma transação.
  - Cada entidade dimensional pode estar ligada a muitas vendas.
- `VENDA |o--|| ENTREGA : "tem"`
  Uma venda pode ter zero ou uma entrega associada (por exemplo, venda cancelada antes de expedir). Cada entrega está ligada obrigatoriamente a uma venda.

Conexão com o DW:

- `VENDA` é o conceito da **tabela fato** `dw.vendas`:
  - FKs: cliente, produto, distribuidor, data (e futuramente transação).
  - Medidas: quantidade_vendida, faturamento, custo_frete.
- Todas as perguntas de negócio principais saem desta entidade no contexto do DW.

---
## 3. Entidades opcionais para análises avançadas (âmbar)

As entidades em **âmbar** (`CATEGORIA`, `AVALIACAO`) representam informações que não são estritamente necessárias para o **MVP analítico**, mas agregam valor em análises mais avançadas.

Na v1:

- Elas estão modeladas conceitualmente.
- A **implementação física** pode ser:
  - parcialmente embutida em outras tabelas, ou
  - deixada para versões futuras, dependendo do esforço x benefício.

### 3.1 CATEGORIA (âmbar)

Entidade que agrupa produtos.

Atributos:

- `categoria_id PK`
- `nome_categoria`
- `descricao`

Relacionamento:

- `CATEGORIA ||--o{ PRODUTO : "agrupa"`
  Uma categoria pode ter muitos produtos; cada produto pertence a uma categoria no v1.

Conexão com o DW:

- No modelo físico v1, a categoria está como **atributo** de `dw.produto` (`nome_categoria`).
- A entidade separada em amarelo indica a possibilidade de:
  - transformar em dimensão normalizada (`dw.categoria`),
  - aplicar SCD,
  - permitir drill-down mais fino nos relatórios.

### 3.2 AVALIACAO (âmbar)

Entidade que representa avaliações de produtos pelos clientes.

Atributos:

- `avaliacao_id PK`
- `nota`
- `comentario`
- `data_avaliacao`

Relacionamentos:

- `CLIENTE ||--o{ AVALIACAO : "faz"`
  Um cliente pode fazer várias avaliações.
- `PRODUTO ||--o{ AVALIACAO : "recebe"`
  Um produto pode receber várias avaliações.

Conexão com o DW:

- Permite KPIs como:
  - “Produtos com nota alta vendem mais?”
  - “Existe correlação entre avaliação e quantidade vendida?”
- Na v1, avaliações estão marcadas como opcionais:
  - podem ser adicionadas como dimensão/ponte em versões futuras,
  - não são necessárias para os KPIs básicos de vendas e frete.

---
## 4. Entidades de jornada e backlog (preto)

As entidades em **preto** (`PEDIDO`, `ENTREGA`) aparecem no diagrama para representar a **jornada completa de e-commerce**, mas estão fora da v1 em termos de implementação analítica.

Objetivo de mostrá-las:

- Tornar explícitas as extensões naturais do DW.
- Facilitar o desenho de backlog (v2+) sem perder o contexto de negócio.

### 4.1 PEDIDO (preto)

Entidade que representa a **intenção de compra** antes da confirmação do pagamento.

Atributos:

- `pedido_id PK`
- `data_pedido`
- `status_pedido`
- `valor_itens`
- `frete_estimado`

Relacionamento:

- `PEDIDO |o--|| TRANSACAO : "gera"`
  Um pedido pode gerar zero ou uma transação:
  - zero: pedido cancelado, abandonado ou não pago.
  - uma: pedido que vira transação confirmada.

Conexão com o DW:

- Reforça a regra: **pedido ≠ venda**.
- Fica fora da v1 do star schema para manter foco em:
  - vendas efetivas,
  - métricas de receita e frete com menor ambiguidade.
- É candidato natural para extensões:
  - funil pedido → transação → entrega,
  - conversão de carrinho,
  - análise de abandono.

### 4.2 ENTREGA (preto)

Entidade que representa o processo logístico após a venda.

Atributos:

- `entrega_id PK`
- `data_envio`
- `data_entrega`
- `status_entrega`
- `custo_entrega`

Relacionamento:

- `VENDA |o--|| ENTREGA : "tem"`
  Uma venda pode ter zero ou uma entrega. Cada entrega está ligada a uma venda.

Conexão com o DW:

- Abre espaço para KPIs logísticos:
  - lead time de entrega,
  - OTIF,
  - análise de SLA por distribuidor e região.
- Fica em backlog na v1, mas já está desenhada para simplificar a evolução do modelo.

---
## 5. Significado das cores e mapeamento para o DW

O bloco de estilos no Mermaid define a paleta e a semântica:

```mermaid
%% v1 (verde) - texto branco
style CLIENTE      fill:#06402B,stroke:#15803d,stroke-width:2px,color:#ffffff
style PRODUTO      fill:#06402B,stroke:#15803d,stroke-width:2px,color:#ffffff
style DATA         fill:#06402B,stroke:#15803d,stroke-width:2px,color:#ffffff
style DISTRIBUIDOR fill:#06402B,stroke:#15803d,stroke-width:2px,color:#ffffff
style TRANSACAO    fill:#06402B,stroke:#15803d,stroke-width:2px,color:#ffffff
style VENDA        fill:#06402B,stroke:#15803d,stroke-width:2px,color:#ffffff

%% opcional (âmbar) - texto branco
style CATEGORIA    fill:#eead2d,stroke:#92400e,stroke-width:2px,color:#ffffff
style AVALIACAO    fill:#eead2d,stroke:#92400e,stroke-width:2px,color:#ffffff

%% backlog fora v1 (preto) - texto branco
style PEDIDO       fill:#000000,stroke:#000000,stroke-width:2px,color:#ffffff
style ENTREGA      fill:#000000,stroke:#000000,stroke-width:2px,color:#ffffff
---
