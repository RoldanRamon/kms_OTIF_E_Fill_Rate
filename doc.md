# Documentação das Colunas Calculadas - Dashboard OTIF e Fill Rate

## 📋 **Visão Geral**
Este documento detalha as colunas calculadas essenciais para análise de influenciadores no dashboard de OTIF e Fill Rate da Komatsu Forest, explicando sua lógica e importância estratégica.

---

## 🎯 **@Status Estoque**

### **Definição**
Classifica o nível de disponibilidade de estoque para atendimento da demanda em aberto, categorizando desde "Sem Estoque" até "Estoque Suficiente" com faixas percentuais específicas.

### **Lógica de Classificação**
```dax
VAR _EstoqueDisponivel = fPedido[@Qtde Estoque No Dia]
VAR _QtdeSolicitada = fPedido[QTDE_EM_ABERTO]
VAR _PercentualAtendimento = DIVIDE(_EstoqueDisponivel, _QtdeSolicitada, 0)

VAR _StatusEstoque = 
SWITCH(TRUE(),
    ISBLANK(_EstoqueDisponivel) || _EstoqueDisponivel = 0, "Sem Estoque",
    _PercentualAtendimento >= 1, "Estoque Suficiente",
    _PercentualAtendimento >= 0.8, "Estoque Parcial (80-99%)",
    _PercentualAtendimento >= 0.5, "Estoque Limitado (50-79%)",
    _PercentualAtendimento > 0, "Estoque Crítico (<50%)",
    _QtdeSolicitada = 0, "Linha Atendida",
    "Não Verificado"
)
```

### **Categorias e Critérios**
| Status | Critério | Impacto no OTIF/Fill Rate |
|--------|----------|---------------------------|
| **Sem Estoque** | Estoque = 0 ou nulo | Alto impacto negativo - impossível atender |
| **Estoque Crítico** | 0% < Cobertura < 50% | Alto risco de NO-FILL_RATE |
| **Estoque Limitado** | 50% ≤ Cobertura < 80% | Risco moderado - atendimento parcial |
| **Estoque Parcial** | 80% ≤ Cobertura < 100% | Baixo risco - quase total atendimento |
| **Estoque Suficiente** | Cobertura ≥ 100% | Capacidade plena de atendimento |
| **Linha Atendida** | Qtde em aberto = 0 | Já processada |

### **Por que essa Classificação?**
- **Granularidade Operacional**: Permite identificar diferentes níveis de urgência
- **Priorização Inteligente**: Foco em casos críticos e limitados
- **Planejamento de Compras**: Sinaliza necessidades de reposição
- **Gestão de Expectativas**: Comunicação clara com clientes sobre possibilidades de atendimento

---

## 👥 **@Tipo Cliente**

### **Definição**
Segmentação estratégica de clientes baseada em frequência de relacionamento comercial e valor monetário dos pedidos, criando categorias de priorização.

### **Lógica de Classificação**
```dax
VAR _TotalPedidosCliente = CALCULATE(
    COUNTROWS(fPedido),
    ALLEXCEPT(fPedido, fPedido[CLIENTE])
)
VAR _ValorTotal = fPedido[VLR_BRUTO]

VAR _TipoCliente = 
SWITCH(TRUE(),
    _TotalPedidosCliente >= 100 && _ValorTotal >= 5000, "Cliente VIP",
    _TotalPedidosCliente >= 50 && _ValorTotal >= 2000, "Cliente Premium",
    _TotalPedidosCliente >= 20, "Cliente Frequente",
    _TotalPedidosCliente >= 5, "Cliente Regular",
    _TotalPedidosCliente < 5, "Cliente Novo/Esporádico",
    "Não Classificado"
)
```

### **Categorias e Critérios**
| Tipo Cliente | Critérios | Tratamento Sugerido |
|-------------|-----------|---------------------|
| **Cliente VIP** | 100+ pedidos E valor ≥ R$5.000 | Máxima prioridade - SLA diferenciado |
| **Cliente Premium** | 50+ pedidos E valor ≥ R$2.000 | Alta prioridade - atendimento preferencial |
| **Cliente Frequente** | 20+ pedidos (qualquer valor) | Prioridade padrão - relacionamento consolidado |
| **Cliente Regular** | 5-19 pedidos | Tratamento padrão - potencial de crescimento |
| **Cliente Novo/Esporádico** | < 5 pedidos | Oportunidade de fidelização |

### **Por que essa Classificação?**
- **Valor Vitalício**: Clientes frequentes e de alto valor têm maior LTV (Lifetime Value)
- **Recursos Limitados**: Priorização baseada em impacto no negócio
- **Retenção Estratégica**: Foco em manter clientes valiosos satisfeitos
- **Crescimento de Conta**: Identificação de oportunidades de upgrade de categoria

---

## 📦 **@Faixa Qtde Itens Pedido**

### **Definição**
Classificação de complexidade operacional dos pedidos baseada no número de itens únicos, impactando diretamente nos processos logísticos e tempos de atendimento.

### **Lógica de Classificação**
```dax
VAR _QtdItens = CALCULATE(
    DISTINCTCOUNT(fPedido[DIM_ITEM_ID]),
    ALLEXCEPT(fPedido, fPedido[NR_PEDIDO])
)

VAR _Faixa = SWITCH(TRUE(),
    _QtdItens = 1, "Item Único",
    _QtdItens <= 8, "Pedido Pequeno (2-8 itens)",
    _QtdItens <= 19, "Pedido Médio (9-19 itens)",
    _QtdItens <= 550, "Pedido Grande (20-550 itens)",
    _QtdItens > 551, "Pedido Complexo (+551 itens)",
    "Não Classificado"
)
```

### **Categorias e Implicações Operacionais**
| Faixa | Qtde Itens | Complexidade | Tempo Processamento | Risco OTIF |
|-------|------------|--------------|--------------------|-----------:|
| **Item Único** | 1 | Mínima | Muito rápido | Baixo |
| **Pedido Pequeno** | 2-8 | Baixa | Rápido | Baixo |
| **Pedido Médio** | 9-19 | Moderada | Normal | Médio |
| **Pedido Grande** | 20-550 | Alta | Lento | Alto |
| **Pedido Complexo** | 551+ | Crítica | Muito lento | Muito Alto |

### **Por que essa Classificação?**
- **Planejamento Operacional**: Pedidos complexos demandam mais recursos e tempo
- **Gestão de SLA**: Diferentes expectativas de prazo por complexidade
- **Alocação de Recursos**: Priorização baseada na dificuldade de execução
- **Prevenção de Gargalos**: Identificação antecipada de pedidos desafiadores

---

## 💰 **@Categoria Valor Pedido**

### **Definição**
Estratificação financeira dos pedidos para priorização por impacto econômico, permitindo foco em transações de maior valor agregado.

### **Lógica de Classificação**
```dax
VAR _CategoriaValor = 
SWITCH(TRUE(),
    fPedido[VLR_BRUTO] <= 500, "Baixo Valor (≤R$500)",
    fPedido[VLR_BRUTO] <= 2000, "Valor Médio (R$501-2.000)",
    fPedido[VLR_BRUTO] <= 5000, "Alto Valor (R$2.001-5.000)",
    fPedido[VLR_BRUTO] <= 15000, "Valor Premium (R$5.001-15.000)",
    fPedido[VLR_BRUTO] > 15000, "Valor Crítico (>R$15.000)",
    "Valor Não Informado"
)
```

### **Categorias e Estratégia de Priorização**
| Categoria | Faixa Valor | % Revenue Típico | Prioridade | Ação Recomendada |
|-----------|-------------|------------------|------------|-------------------|
| **Valor Crítico** | > R$15.000 | ~40-50% | Máxima | Acompanhamento individual |
| **Valor Premium** | R$5.001-15.000 | ~25-30% | Alta | Monitoramento próximo |
| **Alto Valor** | R$2.001-5.000 | ~15-20% | Média-Alta | Processo padrão acelerado |
| **Valor Médio** | R$501-2.000 | ~8-12% | Média | Processo padrão |
| **Baixo Valor** | ≤ R$500 | ~3-7% | Baixa | Processamento em lote |

### **Por que essa Classificação?**
- **Impacto Financeiro**: Foco no Princípio de Pareto (80/20)
- **Gestão de Risco**: Pedidos de alto valor requerem atenção especial
- **Eficiência Operacional**: Processamento otimizado por categoria
- **Satisfação do Cliente**: Clientes de alto valor esperam serviço premium

---

## 🔄 **Integração com Visual de Principais Influenciadores**

### **Como Usar no Visual**
1. **Medida Analisada**: % OTIF Linha ou % FILL_RATE Linha
2. **Dimensões de Influência**: Usar as 4 colunas calculadas como "Explicar por"
3. **Interpretação**: Identifica quais combinações de categorias mais impactam performance

### **Exemplos de Insights Esperados**
- *"Clientes VIP com Pedidos Complexos têm 15% menor OTIF"*
- *"Pedidos de Valor Crítico com Estoque Limitado apresentam maior risco"*
- *"Itens Únicos de Clientes Regulares têm melhor Fill Rate"*

### **Ações Baseadas nos Influenciadores**
- **Status Estoque Crítico** → Acelerar reposição ou comunicar prazo
- **Cliente VIP** → Priorizar na fila de processamento
- **Pedido Complexo** → Alocar equipe especializada
- **Valor Crítico** → Acompanhamento gerencial dedicado

---

## ⚡ **Otimizações Implementadas**

### **Performance**
- Uso de variáveis (VAR) para evitar recálculos
- SWITCH(TRUE()) otimizado vs múltiplos IF
- CALCULATE com ALLEXCEPT para contexto eficiente

### **Manutenibilidade**
- Lógica centralizada em colunas calculadas
- Nomenclatura clara e consistente
- Documentação inline nos códigos DAX

### **Escalabilidade**
- Faixas de valores baseadas em distribuição real dos dados
- Flexibilidade para ajustes futuros de critérios
- Compatibilidade com filtros e slicers

---

*Documento criado em: {{data_atual}}*  
*Última atualização: {{data_atualizacao}}*  
*Responsável: Equipe Analytics Komatsu Forest*