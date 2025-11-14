```mermaid
erDiagram
    %% RELACIONAMENTOS E CARDINALIDADES
    CLIENTE      ||--o{ VENDA       : "realiza"
    PRODUTO      ||--o{ VENDA       : "aparece_em"
    DATA         ||--o{ VENDA       : "ocorre_em"
    DISTRIBUIDOR ||--o{ VENDA       : "atende"
    TRANSACAO    ||--o{ VENDA       : "detalha"
    CATEGORIA    ||--o{ PRODUTO     : "agrupa"
    VENDA        |o--|| ENTREGA     : "tem"
    PRODUTO      ||--o{ AVALIACAO   : "recebe"
    CLIENTE      ||--o{ AVALIACAO   : "faz"
    PEDIDO       |o--|| TRANSACAO   : "gera"

    %% ENTIDADES E ATRIBUTOS (CONCEITUAL)
    CLIENTE {
        int    cliente_id PK
        string nome
        string endereco
        string cidade
        string pais
        date   data_cadastro
    }

    PRODUTO {
        int     produto_id PK
        string  nome
        string  descricao
        decimal preco
        string  nome_categoria
    }

    CATEGORIA {
        int    categoria_id PK
        string nome_categoria
        string descricao
    }

    DATA {
        int    data_id PK
        date   data
        int    dia
        int    mes
        int    ano
        string dia_da_semana
    }

    DISTRIBUIDOR {
        int    distribuidor_id PK
        string nome_distribuidor
        string cidade_distribuidor
        string pais_distribuidor
    }

    TRANSACAO {
        int     transacao_id PK
        date    data_transacao
        decimal valor_total
        string  meio_pagamento
        string  status
    }

    VENDA {
        int     venda_id PK
        int     quantidade_vendida
        decimal faturamento
        decimal custo_frete
    }

    ENTREGA {
        int     entrega_id PK
        date    data_envio
        date    data_entrega
        string  status_entrega
        decimal custo_entrega
    }

    AVALIACAO {
        int    avaliacao_id PK
        int    nota
        string comentario
        date   data_avaliacao
    }

    PEDIDO {
        int     pedido_id PK
        date    data_pedido
        string  status_pedido
        decimal valor_itens
        decimal frete_estimado
    }

    %% ESTILO POR ESCOPO
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
