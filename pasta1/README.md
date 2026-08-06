# consultoria_driva-tech_powerbi (Em Construção)
<br>

# Sumário
1. Contexto
2. Preview dashboard
3. Origem dos dados
4. ETL
5. Criação da tabela Calendário
6. Relacionamentos (Star Schema)
7. Hipóteses
8. Clusterização com Machine Learning (K-Means)
9. Páginas do Dashboard + Explicação dos gráficos
10. Algumas Fórmulas utilizadas nas medidas criadas
11. Validação de hipóteses
12. Insights e Recomendações

# 1. Contexto
A varejista de moda, Arfex, busca entender melhor o desempenho de suas lojas físicas para otimizar estratégias de distribuição de produtos e promoções, visando aumento das vendas e melhoria na satisfação do cliente. O cliente forneceu dados das vendas das lojas e dados demográficos dos consumidores.<br>
<br>

**Objetivo**:
1. Examinar os dados para identificar padrões de vendas e diferenças regionais;<br>
2. Aplicar técnicas básicas de segmentação para identificar diferentes perfis de consumidores. Analisar suas preferências de compra para entender quais produtos ou promoções são mais eficazes para cada segmento;<br>
3. Elaborar recomendações práticas sobre como ajustar a distribuição de produtos e as estratégias de marketing para melhor atender aos diferentes segmentos de clientes, baseando-se nos insights extraídos das análises anteriores.<br>

# 2. Preview Dashboard
<img width="1245" height="694" alt="image" src="https://github.com/user-attachments/assets/6b232908-9a7a-47a5-b52e-c6a80475e1f5" />


# 3. ETL
Carregamento das planilhas normalizadas em Power BI e Transformação em Power Query para limpeza, criação de novas colunas, medidas e criação da tabela calendário. Na sessão 9 compartilho algumas das fórmulas DAX que utilizei o objetivo:<br>

## Origem dos Dados: 
Fictícios. Gerados usando IA, porém seguindo modelo de consultoria para aplicação de conceitos e estudo.<br>

## Esquema de Dados das Tabelas Originais:
<img width="1074" height="431" alt="image" src="https://github.com/user-attachments/assets/c5a95d55-59ca-4095-9d50-0f7f7ccb9f46" />
<br>

# 4. Criação da tabela Calendário
Esta fórmula DAX para a criação da tabela calendário é uma melhor opção do que o Auto Date/Time do Power BI, pois prioriza a performance e a capacidade de análise avançada (Time Intelligence) no projeto, evitando o inchaço (model bloat) que a fórmula automática Auto Date/Time geraria ao criar diversas tabelas ocultas de data para cada coluna de data no projeto.

DIM_Calendario = 
ADDCOLUMNS (
    CALENDAR (MIN(FATO_Vendas[VENDAS.DATA_VENDA]), MAX(FATO_Vendas[VENDAS.DATA_VENDA])),
    "Ano", YEAR([Date]),
    "Trimestre Num", QUARTER([Date]),
    "Trimestre", "T" & QUARTER([Date]),
    "Mês Nome", FORMAT([Date], "MMMM"),
    "Mês Num/Ano", FORMAT([Date], "YYYY-MM"), // Importante para ordenação
    "Dia da Semana Nome", FORMAT([Date], "dddd"),
    "Dia da Semana Num", WEEKDAY([Date], 2) // 1=Segunda, 7=Domingo
)

**Objetivo na Análise**:<br>
O principal objetivo é permitir realizar Análises Temporais Avançadas (Time Intelligence) de forma correta e flexível, pois possibilita:<br>
- Agrupamento: Permite agrupar vendas por Mês, Dia da Semana, Trimestre, ou Dia Útil/Fim de Semana.<br>
- Comparação: É o motor para métricas complexas como "Vendas Mês Anterior" ou "Vendas Ano Passado", que exigem uma sequência de datas ininterrupta.<br>
- Filtro: Permite filtrar todas as suas métricas (Vendas, Ticket Médio, etc.) usando atributos de tempo (como "Mês", "Dia da Semana") em vez de usar apenas a data bruta.<br>

# 5. Relacionamentos
<img width="1129" height="720" alt="image" src="https://github.com/user-attachments/assets/f3026842-e92a-457e-82d9-cdb4787b5f71" />

# 6. Clusterização com Machine Learning (K-Means)
import pandas as pd

try:
    df = dataset.copy()

    # 1. LIMPEZA DE CABEÇALHOS (Remove espaços extras e coloca tudo em maiúsculo)
    # Isso resolve se estiver " NOME_FILIAL" ou "nome_filial"
    df.columns = [col.strip().upper() for col in df.columns]

    # Verifica se a coluna de filial existe (ajuste 'NOME_FILIAL' se o seu nome for diferente, ex: 'LOJA')
    col_filial = 'NOME_FILIAL' 
    
    if col_filial not in df.columns:
        # Se não achar, tenta procurar uma coluna que contenha "FILIAL" no nome
        possiveis = [c for c in df.columns if 'FILIAL' in c]
        if possiveis:
            col_filial = possiveis[0] # Pega a primeira que achar
        else:
            raise KeyError(f"Coluna de Filial não encontrada. Colunas disponíveis: {list(df.columns)}")

    # 2. AUTO-CRUZAMENTO
    # Usa a variável col_filial que encontramos acima
    df_merge = pd.merge(df, df, on=['ID_VENDA', 'PERFIL_CLIENTE', col_filial])

    # 3. FILTRO E RANKING
    df_pares = df_merge[df_merge['NOME_PRODUTO_x'] < df_merge['NOME_PRODUTO_y']]

    ranking = df_pares.groupby(['PERFIL_CLIENTE', col_filial, 'NOME_PRODUTO_x', 'NOME_PRODUTO_y']).size().reset_index(name='Qtd_Vendas_Juntas')

    ranking.rename(columns={
        'NOME_PRODUTO_x': 'Produto_A',
        'NOME_PRODUTO_y': 'Produto_B'
    }, inplace=True)

    ranking.sort_values('Qtd_Vendas_Juntas', ascending=False, inplace=True)

    df_final = ranking

except Exception as e:
    # Isso cria uma tabela de erro legível no Power BI se algo der errado
    df_final = pd.DataFrame({'Erro': [str(e)]})
    
# 7. Hipóteses
1. O ticket médio dos clusters é expressivamente diferente, e clientes ouro gastam mais que os outros por pedido.
2. Cada cluster compra produtos e combos diferentes, portanto, a estratégia de mix e marketing deve ser diferente para cada cluster
3. Clientes Ouro compram mais vezes que clientes Prata e Bronze
4. Clientes Bronze são clientes inativos (com mais de 30 dias sem comprar em média)
5. Clientes Bronze compram mais produtos em promoção do que os outros clusters
6. Cada cluster compra essencialmente o mesmo mix de produtos independente da filial
7. Algumas filiais possuem mais clientes ouro do que outros, podendo se beneficiar de ter produtos mais voltados para este público
8. Os melhores dias de vendas para todas as filiais são: Sex, Sáb e Domingo, e estes são os dias mais importantes de se ter mais funcionários e estoque garantido em loja
9. As filiais que faturam mais, vendem maior quantidade e não produtos mais caros, portanto, o foco deve ser recorrência e fidelidade
10. Não há variação expressiva entre a origem dos clientes entre as filiais, portanto, a empresa não precisa ter estratégias diferentes com relação à isso
11. *****NÃO TESTADA***** O mix de produtos comprados por clientes de diferentes origens é muito parecido, portanto, é indiferente para estratégia de mix do PDV

# 8. Páginas do Dashboard + Explicação dos gráficos (em construção)
<img width="1245" height="692" alt="image" src="https://github.com/user-attachments/assets/8d97cfb7-805f-4946-bf72-6d6927b7ecb5" /><br>
<br>
<br>
<img width="1250" height="697" alt="image" src="https://github.com/user-attachments/assets/8d34858e-b2ad-44dc-b789-4ddaa1951a0d" /><br>
<br>
<br>
<img width="1263" height="703" alt="image" src="https://github.com/user-attachments/assets/f29ea16b-e47c-467b-9f5a-2c1f11c6b20b" /><br>
<br>
<br>
<img width="1268" height="703" alt="image" src="https://github.com/user-attachments/assets/458d87db-76dc-4dfb-a7f1-55f1e9bb15a8" /><br>
<br>
<br>

# 9. Algumas Fórmulas utilizadas nas medidas criadas
#### B. Agregação por Cliente (Proxy LTV)
Esta fórmula é uma Coluna Calculada na tabela DIM_Clientes e é a base para a segmentação de clientes por Lifetime Value (LTV). Ela Resolve o problema de agregar dados transacionais (muitas linhas na FATO) em uma tabela de dimensão (uma linha por cliente), usando a técnica de Context Transition (CALCULATE) e ignora/aplica filtros (FILTER(ALL())) para garantir que o gasto total de cada cliente seja preciso.
```
VALOR_GASTO_CLIENTE =
CALCULATE(
    [VALOR_VENDA_TOTAL],
    FILTER(
        ALL('FATO_Vendas'),
        'FATO_Vendas'[VENDAS.ID_CLIENTE] = 'DIM_Clientes'[ID_CLIENTE]
    )
)
```
**Objetivo na Análise**:<br>
Esta fórmula permite medir o valor monetário total que cada cliente contribuiu para o negócio, desde o início dos dados, sem interferências de filtros gerais.<br>
<br>

#### C. Benchmark Dinâmico (Média Geral Ajustável)
Esta métrica é essencial para criar a linha de referência nos gráficos de Faturamento por Filial e Ticket Médio. Ela calcula a média da rede, mas tem a inteligência de se ajustar a filtros de data ou cliente (tornando-a dinâmica).<br>
```
MEDIA_FATURAMENTO_GERAL_FILIAL_DINAMICA =
VAR FaturamentoGeral = CALCULATE([VALOR_VENDA_TOTAL], ALL(DIM_Filiais))
VAR NumFiliais = COUNTROWS(ALL(DIM_Filiais))
RETURN
    DIVIDE(FaturamentoGeral, NumFiliais)
```
**Objetivo na Análise**:<br>
Esta métrica permite avaliar o desempenho das filiais em relação à média de faturamento que cada filial deveria atingir no período e contexto filtrado (se o usuário filtrou por "Novembro", a média de novembro é calculada).<br>
<br>

# 10. Validação de hipóteses (em construção)

# 11. Insights e Recomendações (em construção)

# Trecho do projeto antigo (em construção, pode ignorar daqui pra baixo)

---



---



---

## Dicas de Ferramenta:



# Técnica de Análise:
Análise Exploratória no Dashboard em Power BI, utilizando recursos estatísticos para observar padrões e tendências para geração de insights<br>

# Insights extraídos:
[Imagens da apresentação] (Em construção)<br>
<br>

# Análises possíveis caso eu tivesse mais tempo:

### 1. Segmentação RFM (Recência, Frequência e Valor Monetário):
**1.1. Monetário   (M)**: Permitiria uma classificação mais precisa de LTV (Lifetime Value);<br>
**1.2. Frequência  (F)**: Ajudaria a distinguir o "campeão" (muitas compras, alto gasto) do "cliente premium" (poucas compras, altíssimo gasto);<br>
**1.3. Recência    (R)**: Nesse caso não seria útil, pois temos dados transacionais somente de um período de 3 meses.<br>
<br>

### 2. Análises Estatísticas:

#### 2.1. Análise de Cesta de Mercado (Market Basket Analysis - MBA)
**Método**: Análise de Regras de Associação (Conceitualmente, Algoritmo Apriori).<br>
**Objetivo**: Identificar grupos de produtos que são frequentemente comprados juntos.<br>
**Ação**: Calcular a Apoio (frequência de ocorrência conjunta) e a Confiança (probabilidade de comprar o item B se o item A foi comprado).<br>
**Insight**: "Se um cliente compra uma Calça Jeans, ele compra Meias Coloridas em 60% das vezes." Isso informa o cross-selling no PDV e o bundling (pacotes) de produtos.<br>
<br>

#### 2.2. Análise de Elasticidade-Preço da Demanda (PED)
**Método**: Regressão (ou análise de correlação segmentada);<br>
**Objetivo**: Medir a sensibilidade do cliente ao desconto;<br>
**Ação**: Analisar como o volume de venda (QUANTIDADE) de um produto específico muda em relação ao desconto concedido (VALOR_UNITARIO_VENDA_PRODUTO vs PRECO_TABELA);<br>
**Insight**: Identificar produtos que são elásticos (respondem muito bem ao desconto) e inelásticos (vendem a mesma quantidade, mesmo sem desconto, permitindo maior margem de lucro).<br>
<br>
