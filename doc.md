# Documentação OTIF e Fill Rate - Dashboard Komatsu

## 1. Conceito de OTIF (On-Time In-Full)

### O que é OTIF?

OTIF é um indicador de desempenho logístico que mede a capacidade de uma empresa entregar pedidos **no prazo (On-Time)** e **completo (In-Full)** simultaneamente. É uma métrica crítica para avaliar a eficiência operacional e a satisfação do cliente.

### Dimensões do OTIF

O sistema analisa o OTIF em **dois níveis**:

#### **1. OTIF por Linha**
Avalia cada linha de item dentro de um pedido individualmente, considerando:
- **Data Prometida (DT_ENTREGA)**: Data acordada com o cliente para entrega
- **Data da Nota Fiscal (NF_DATA)**: Data em que o produto foi efetivamente faturado/emitido
- **Data Atual (TODAY)**: Data de análise

#### **2. OTIF por Pedido**
Consolida o status de todas as linhas de um pedido em uma única classificação. Um pedido é considerado:
- **OTIF**: Quando TODAS as suas linhas estão em situação OTIF
- **NO-OTIF**: Quando pelo menos uma linha está com atraso na entrega
- **Ainda No Prazo**: Quando nenhuma linha foi entregue, mas ainda há tempo até a data prometida

---

## 2. Lógica de Classificação OTIF - Nível Linha

### Fórmula de Cálculo

A coluna calculada `@OTIF Linha` da tabela `fPedido` utiliza a seguinte lógica:

```
VAR _DiferencaPrevistoHoje = DATEDIFF(DT_ENTREGA, TODAY(), DAY)
VAR _DiferencaPrevistoNota = DATEDIFF(DT_ENTREGA, NF_DATA, DAY)
VAR _NotaEmBranco = IF(ISBLANK(NF_DATA), "Sim", "Não")

SWITCH(TRUE(),
    _DiferencaPrevistoHoje <= 0 AND _NotaEmBranco = "Sim" → "Ainda No Prazo",
    _DiferencaPrevistoHoje > 0 AND _NotaEmBranco = "Sim" → "NO-OTIF",
    _DiferencaPrevistoNota > 0 → "NO-OTIF",
    _DiferencaPrevistoNota <= 0 → "OTIF",
    BLANK()
)
```

### Estados Possíveis por Linha

| Status | Condição | Significado |
|--------|----------|-------------|
| **OTIF** | Nota fiscal emitida com data igual ou anterior à data prometida | Entrega no prazo |
| **NO-OTIF** | Nota fiscal emitida com data posterior à data prometida, OU data de hoje já passou da prometida sem nota | Entrega atrasada |
| **Ainda No Prazo** | Sem nota fiscal emitida E data de hoje ainda está antes da data prometida | Aguardando execução |

---

## 3. Lógica de Classificação OTIF - Nível Pedido

### Agregação de Linhas para Pedido

A tabela `Analise_OTIF_Pedido` consolida o status das linhas usando a seguinte lógica:

```
SWITCH(TRUE(),
    [NO-OTIF] > 0 → "NO-OTIF",
    [Ainda No Prazo] > 0 → "Ainda No Prazo",
    → "OTIF"
)
```

### Hierarquia de Classificação

1. Se **qualquer linha tem NO-OTIF** → Pedido é **NO-OTIF**
2. Senão, se **qualquer linha está Ainda No Prazo** → Pedido está **Ainda No Prazo**
3. Senão → Pedido é **OTIF**

---

## 4. Fill Rate (Taxa de Preenchimento)

### O que é Fill Rate?

Fill Rate mede a capacidade de atender à quantidade solicitada no pedido com base no **estoque disponível no dia da emissão do pedido**.

### Estados de Fill Rate - Nível Linha

A coluna calculada `@FILL RATE Linha` utiliza:

```
IF(Qtde Estoque No Dia > 0 AND Qtde Estoque No Dia >= QTDE_EM_ABERTO, 
   "FILL_RATE", 
   "NO-FILL_RATE")
```

| Status | Condição | Significado |
|--------|----------|-------------|
| **FILL_RATE** | Estoque disponível >= Quantidade em aberto | Pedido pode ser 100% atendido |
| **NO-FILL_RATE** | Estoque disponível < Quantidade em aberto | Faltam produtos para atender completamente |

### Fill Rate - Nível Pedido

Similar ao OTIF de pedido:
- **FILL_RATE**: Todas as linhas têm estoque suficiente
- **NO-FILL_RATE**: Pelo menos uma linha não tem estoque suficiente

---

## 5. Filtros da Dashboard

### 5.1 Filtros a Nível de Linha

Os seguintes filtros podem ser aplicados para filtrar linhas de pedido individuais:

| Filtro | Campo Base | Valores | Descrição |
|--------|-----------|--------|-----------|
| **OTIF Linha** | `fPedido[@OTIF Linha]` | OTIF, NO-OTIF, Ainda No Prazo | Status de conformidade da linha com prazo |
| **FILL RATE Linha** | `fPedido[@FILL RATE Linha]` | FILL_RATE, NO-FILL_RATE | Disponibilidade de estoque para a linha |
| **Data de Emissão** | `fPedido[DT_EMISSAO]` | Intervalo de datas | Data de criação do pedido |
| **Data de Entrega** | `fPedido[DT_ENTREGA]` | Intervalo de datas | Data prometida de entrega |

### 5.2 Filtros a Nível de Pedido

Os seguintes filtros operam agregando dados por pedido:

| Filtro | Tabela Base | Valores | Descrição |
|--------|-----------|--------|-----------|
| **OTIF Pedido** | `Analise_OTIF_Pedido[@OTIF Pedido]` | OTIF, NO-OTIF, Ainda No Prazo | Status consolidado do pedido |
| **FILL RATE Pedido** | `Analise_FILL_RATE_Pedido[@FILL_RATE Pedido]` | FILL_RATE, NO-FILL_RATE | Atendimento total do pedido |

### 5.3 Filtros Dimensionais (por Contexto de Negócio)

Através da tabela `fPedido` e relacionamentos, é possível filtrar por:

| Campo | Descrição |
|-------|-----------|
| `NR_PEDIDO` | Número do pedido |
| `DIM_CLIENTE_ID` | Identificação do cliente |
| `DIM_EMPRESA_ID` | Empresa/Filial responsável |
| `DIM_ITEM_ID` | Produto solicitado |
| `DIM_REPRESENTANTE_ID` | Representante de vendas |
| `DIM_DIVISAO_VENDA_ID` / `TP_DIVISAO_VENDA` | Divisão de vendas (PEÇAS, PEÇAS ATIVAS, PEÇAS KDB, PEÇAS CONTRATO) |
| `DIM_UNID_NEGOC_ID` / `NM_UNID_NEGOC` | Unidade de negócio |
| `DIM_COND_PAGTO_ID` | Condição de pagamento |

### 5.4 Filtros de Calendario

Através do relacionamento com `dCalendario`:

| Campo | Descrição |
|-------|-----------|
| `dCalendario[Date]` | Análise por data de emissão |
| Anos, Trimestres, Meses, Semanas, Dias | Períodos de análise |

---

## 6. Estrutura de Dados - Relacionamentos

### Relacionamentos Principais

```
fPedido (Tabela Fato Principal)
├── DT_EMISSAO → dCalendario[Date]
├── NR_PEDIDO → Analise_OTIF_Pedido[NR_PEDIDO] (Bi-direcional)
├── NR_PEDIDO → Analise_FILL_RATE_Pedido[NR_PEDIDO] (Bi-direcional)
├── IdPedidoSaldoEstoque → dSaldoEstoqueDiario[IdPedidoSaldoEstoque]
└── ID_PEDIDO_SEQ_ITEM → fBaseCarteira[ID_PEDIDO_SEQ_ITEM]
```

### Tabelas Auxiliares

- **Analise_OTIF_Pedido**: Tabela calculada que consolida OTIF por número de pedido
- **Analise_FILL_RATE_Pedido**: Tabela calculada que consolida Fill Rate por número de pedido
- **dCalendario**: Dimensão de datas para análises temporais
- **dSaldoEstoqueDiario**: Estoque disponível por item e data
- **fBaseCarteira**: Informações adicionais de carteira de pedidos

---

## 7. Medidas Principais

### OTIF - Nível Linha

| Medida | Fórmula | Resultado |
|--------|---------|-----------|
| `Qtde Linhas OTIF` | COUNT onde `@OTIF Linha = "OTIF"` | Quantidade de linhas entregues no prazo |
| `Qtde Linhas NO-OTIF` | COUNT onde `@OTIF Linha = "NO-OTIF"` | Quantidade de linhas atrasadas |
| `Qtde Linhas Ainda No Prazo` | COUNT onde `@OTIF Linha = "Ainda No Prazo"` | Quantidade de linhas pendentes |
| `% OTIF` | `Qtde Linhas OTIF / Total de Linhas` | Percentual de linhas em dia |

### Fill Rate - Nível Linha

| Medida | Fórmula | Resultado |
|--------|---------|-----------|
| `Qtde Linhas FILL_RATE` | COUNT onde `@FILL RATE Linha = "FILL_RATE"` | Linhas com estoque suficiente |
| `Qtde Linhas NO-FILL_RATE` | COUNT onde `@FILL RATE Linha = "NO-FILL_RATE"` | Linhas com estoque insuficiente |
| `% FILL_RATE` | `Qtde Linhas FILL_RATE / Total de Linhas` | Taxa de atendimento de estoque |

### OTIF - Nível Pedido

| Medida | Fórmula | Resultado |
|--------|---------|-----------|
| `Qtde Pedidos OTIF` | DISTINCTCOUNT(NR_PEDIDO) onde `@OTIF Pedido = "OTIF"` | Pedidos entregues no prazo |
| `% OTIF PEDIDO` | `Qtde Pedidos OTIF / Total de Pedidos` | Taxa de pedidos em dia |

### Fill Rate - Nível Pedido

| Medida | Fórmula | Resultado |
|--------|---------|-----------|
| `Qtde Pedidos FILL_RATE` | DISTINCTCOUNT(NR_PEDIDO) onde `@FILL_RATE Pedido = "FILL_RATE"` | Pedidos 100% atendidos |
| `% FILL_RATE PEDIDO` | `Qtde Pedidos FILL_RATE / Total de Pedidos` | Taxa de pedidos 100% atendidos |

---

## 8. Fluxo de Análise Recomendado

1. **Página OTIF**: Analisa o desempenho de prazos de entrega
   - Visualize o percentual de pedidos entregues no prazo
   - Identifique NO-OTIFs por divisão, representante ou cliente
   - Use filtros de período para tendências

2. **Página Fill Rate**: Analisa a disponibilidade de estoque
   - Verifique taxa de atendimento total de pedidos
   - Identifique produtos com falta de estoque
   - Correlacione com OTIF para entender impacto

3. **Cruzamento de Análises**:
   - Um pedido pode ser OTIF mas não ser FILL_RATE (entrega no prazo, mas parcial)
   - Um pedido pode ter FILL_RATE mas ser NO-OTIF (estoque disponível, mas entregue atrasado)

---

## 9. Filtros Implementados na Dashboard

A dashboard implementa os seguintes filtros interativos:

### Página OTIF (Principal)
- **Slicer - OTIF Linha** (Dropdown): Filtra linhas por status OTIF
- Filtros dimensionais por cliente, empresa, divisão de vendas, período

### Página Fill Rate
- **Slicer - FILL RATE Linha** (Dropdown): Filtra linhas por status de Fill Rate
- Mesmos filtros dimensionais para consistência

### Comportamento de Filtro
- Os slicers são **bidirecionais** entre tabelas fPedido e análises consolidadas
- Seleção de filtro em uma visualização afeta todas as outras
- "Select All" habilitado para facilitar análises comparativas

---

## 10. Fonte de Dados

- **Banco**: `idba_komatsu` (SQL Server: BRPIN-DBS008\PowerBI)
- **Tabela Principal**: `FATO_PEDIDO_BI`
- **Filtro de Dados**: 
  - Tipo Divisão de Venda: PEÇAS, PEÇAS ATIVAS, PEÇAS KDB, PEÇAS CONTRATO
  - Posição (Status): Diferente de "Cancelado"
  - Período: A partir de 01/01/2022

---

## Conclusão

A dashboard OTIF e Fill Rate fornece visibilidade completa sobre dois KPIs críticos de desempenho logístico, permitindo análises detalhadas tanto por linha quanto por pedido, com filtros dimensionais para contexto de negócio específico.
