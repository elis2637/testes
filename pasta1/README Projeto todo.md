# 🚀 MVP de Engenharia de Dados: Plataforma Integrada de Dados & Data Marts Institucionais

![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=for-the-badge&logo=Databricks&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-000000?style=for-the-badge&logo=delta&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache_Spark-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=for-the-badge&logo=postgresql&logoColor=white)
![Medallion Architecture](https://img.shields.io/badge/Architecture-Medallion-blue?style=for-the-badge)
![Data Governance](https://img.shields.io/badge/Governance-Unity_Catalog-green?style=for-the-badge)

---

## 📌 1. Visão Geral do Projeto Completo

Este repositório contém a solução completa de **Engenharia de Dados e Business Intelligence** desenvolvida para estruturar o ecossistema de dados institucionais da organização. O projeto abrange desde a **ingestão brutos de sistemas heterogêneos** até a disponibilização de **Data Marts analíticos de alta performance** estruturados em **Esquema Estrela (Star Schema)**.

O principal objetivo do projeto é consolidar e tratar dados acadêmicos, demográficos e profissionais, fornecendo à alta gestão e equipes analíticas uma **Plataforma de Dados Única (Single Source of Truth - SSOT)** para tomada de decisão estratégica baseada em evidências.

### 🎯 Principais Objetivos da Plataforma:
- **Acompanhamento contínuo de Egressos:** Mensurar a empregabilidade, faixas salariais e aderência dos cursos ao mercado de trabalho.
- **Gestão da Satisfação (NPS & Alumni):** Monitorar os índices de satisfação dos ex-alunos e o nível de engajamento com a instituição.
- **Identificação de Oportunidades de Receita:** Mapear egressos com alto potencial de rematrícula em programas de Pós-Graduação, MBA e Cursos de Extensão.
- **Governança e Qualidade de Dados:** Garantir rastreabilidade, padronização, limpeza de dados corrompidos e segurança em conformidade com as diretrizes da LGPD.

---

## 🏗️ 2. Arquitetura da Plataforma (Medallion Architecture)

A esteira de dados foi projetada utilizando o padrão de **Arquitetura Medallion** sobre a plataforma **Databricks Delta Lake**, dividida no banco de dados operacional `mvp_eng_dados`:

```text
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                 FONTES DE DADOS ORIGEM                                  │
│   • Formulários de Pesquisa   • Sistema Acadêmico (ERP)   • Base de Cursos & Mensalidades │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  CAMADA BRONZE (RAW)                                    │
│   • Ingestão em formato nativo / Append-only                                            │
│   • Metadados de auditoria (dt_ingestao, arquivo_origem)                                │
│   • Tabelas: bronze.raw_pesquisas_egressos, bronze.raw_sistema_academico               │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                CAMADA SILVER (CLEANSED)                                 │
│   • Sanitização de limites (Idade entre 16 e 100 anos; Renda >= 0)                      │
│   • Padronização de datas, siglas de UF e textos em maiúsculo                           │
│   • Deduplicação de registros e tratamento de nulos                                     │
│   • Tabelas: silver.dataset_egressos, silver.dataset_cursos                             │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              CAMADA GOLD (CURATED / DATA MARTS)                         │
│   • Modelagem Dimensional em Esquema Estrela (Star Schema)                              │
│   • Cálculo de Métricas (Categorização NPS, Faixa Temporal, Surrogate Keys MD5)          │
│   • Otimização física via Z-Ordering                                                    │
│   • Tabelas: fato_egressos, dim_egresso, dim_curso, dim_emprego, dim_tempo              │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                             CAMADA DE CONSUMO & ANALYTICS                               │
│   • Dashboards Executivos (Power BI / Tableau / Databricks SQL)                         │
│   • Modelos de Machine Learning (Propensão a Pós-Graduação)                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📐 3. Modelagem Dimensional Completa do Ecossistema

O Data Mart institucional foi construído com foco em **performance analítica** e **facilidade de consulta** por ferramentas de self-service BI:

```text
                             ┌──────────────────────────────────┐
                             │           dim_curso              │
                             ├──────────────────────────────────┤
                             │ PK  id_curso                     │
                             │     nome_curso                   │
                             │     area_conhecimento            │
                             │     duracao_semestres            │
                             │     mensalidade_base             │
                             │     modalidade_graduacao         │
                             └────────────────┬─────────────────┘
                                              │
                                              │ 1:N
┌──────────────────────────────────┐          ▼          ┌──────────────────────────────────┐
│           dim_egresso            │   ┌──────────────┐  │           dim_emprego            │
├──────────────────────────────────┤   │ fato_egressos│  ├──────────────────────────────────┤
│ PK  id_egresso                   ├──>│──────────────│<──┤ PK  id_emprego                   │
│     idade_egresso                │1:N│ FK id_egresso│1:N│     nivel_cargo                  │
│     uf_residencia                │   │ FK id_curso  │  │     tipo_empresa                 │
│     bolsista_graduacao           │   │ FK id_emprego│  └──────────────────────────────────┘
└──────────────────────────────────┘   │ FK id_tempo  │
                                       │    renda_... │  ┌──────────────────────────────────┐
                                       │    satisf... │  │            dim_tempo             │
                                       │    engaja... │  ├──────────────────────────────────┤
                                       │    potenc... │<─┤ PK  id_tempo (YYYYMMDD)           │
                                       └──────────────┘1:N│    data_completa, ano, mes, dia  │
                                                         └──────────────────────────────────┘
```

---

## 📖 4. Dicionário Lógico de Dados Unificado (`mvp_eng_dados`)

### 4.1. Camada Gold (Data Mart de Consumo Analítico)

#### Tabela Fato: `gold.fato_egressos`
Armazena os eventos e métricas de acompanhamento dos egressos.

| Nome da Coluna | Tipo | Chave | Nulo? | Origem | Regra de Negócio / Descrição |
| :--- | :--- | :---: | :---: | :--- | :--- |
| **`id_egresso`** | `INT` | **FK** | Não | `silver.dataset_egressos` | Chave de ligação com `dim_egresso`. |
| **`id_curso`** | `INT` | **FK** | Não | `silver.dataset_egressos` | Chave de ligação com `dim_curso`. |
| **`id_emprego`** | `STRING` | **FK** | Não | Hash MD5 | Chave surrogate de ligação com `dim_emprego`. |
| **`id_tempo`** | `INT` | **FK** | Não | Derivado | Data do contato formatada como `YYYYMMDD`. |
| **`dt_ultimo_contato`** | `TIMESTAMP` | - | Sim | `silver.dataset_egressos` | Data e hora exata da pesquisa de acompanhamento. |
| **`renda_mensal_estimada`** | `DECIMAL(10,2)` | - | Sim | `silver.dataset_egressos` | Renda mensal (R$). Rendas < 0 convertidas em `NULL`. |
| **`satisfacao_graduacao_nps`** | `INT` | - | Sim | `silver.dataset_egressos` | Nota atribuída ao curso de 0 a 10. |
| **`categoria_nps`** | `STRING` | - | Sim | Derivado | `Promotor` (≥9), `Neutro` (7-8), `Detrator` (≤6). |
| **`engajamento_alumni_score`** | `INT` | - | Sim | `silver.dataset_egressos` | Pontuação de participação na comunidade Alumni (0 a 100). |
| **`potencial_matricula_pos`** | `INT` | - | Sim | `silver.dataset_egressos` | Flag (1 = Alto Potencial para Pós, 0 = Baixo). |
| **`meses_desde_formacao`** | `INT` | - | Sim | `silver.dataset_egressos` | Tempo transcorrido desde a formatura (em meses). |
| **`faixa_meses_formacao`** | `STRING` | - | Sim | Derivado | Faixas: `0-12m`, `13-24m`, `25-36m`, `37-48m`, `49-60m`, `>60m`. |

---

#### Tabelas de Dimensão

- **`gold.dim_egresso`**: Perfil demográfico (`id_egresso`, `idade_egresso`, `uf_residencia`, `bolsista_graduacao`).
- **`gold.dim_curso`**: Cadastro acadêmico (`id_curso`, `nome_curso`, `area_conhecimento`, `duracao_semestres`, `mensalidade_base`, `modalidade_graduacao`).
- **`gold.dim_emprego`**: Mercado de trabalho (`id_emprego` [MD5], `nivel_cargo`, `tipo_empresa`).
- **`gold.dim_tempo`**: Calendário corporativo (`id_tempo`, `data_completa`, `ano`, `trimestre`, `mes`, `nome_mes`, `dia`).

---

## 🛡️ 5. Qualidade de Dados & Regras de Governança

Para assegurar a confiabilidade dos relatórios executivos, o pipeline aplica validações automáticas em cada etapa:

1. **Higienização Biológica e Financeira (Camada Silver):**
   - Idades fora da faixa `[16, 100]` anos são marcadas como nulas para evitar distorção de médias.
   - Valores negativos de renda salarial são invalidados.
2. **Idempotência e Deduplicação:**
   - Registros duplicados provenientes de re-submissões de formulários são filtrados mantendo-se a última atualização (`dt_ultimo_contato`).
3. **Padronização Enunciativa:**
   - Nomes de cursos e cargos são convertidos para maiúsculas (`UPPER`), removendo espaços extras e acentuação inconsistente.
   - Siglas de UF são validadas contra a lista oficial do IBGE.

---

## 🚀 6. Otimização & Alta Performance (Delta Lake)

O ambiente foi otimizado para responder a queries complexas em segundos, utilizando os recursos nativos do Databricks Delta Lake:

- **Z-Ordering Multi-Dimensional:**
  Organização física dos arquivos Parquet para acelerar o *data skipping*:
  ```sql
  OPTIMIZE mvp_eng_dados.gold.fato_egressos ZORDER BY (id_curso, id_egresso);
  OPTIMIZE mvp_eng_dados.gold.dim_curso ZORDER BY (id_curso);
  OPTIMIZE mvp_eng_dados.gold.dim_egresso ZORDER BY (id_egresso);
  ```
- **Formato Delta Lake:** Suporte a transações ACID, time travel e compilação automática de pequenos arquivos (*auto-compact*).

---

## 📂 7. Estrutura Completa do Repositório

```text
.
├── README.md                           # Documentação global do projeto
├── requirements.txt                    # Dependências do ambiente Python / Databricks
├── config/
│   ├── databricks_cluster_config.json  # Configurações recomendadas de cluster
│   └── delta_tables_config.json        # Parâmetros de retenção e otimização Delta
├── docs/
│   ├── arquitetura_medallion.png       # Diagrama de arquitetura da plataforma
│   ├── modelo_dimensional_ert.png      # Diagrama Entidade-Relacionamento do Data Mart
│   └── dicionario_de_dados_completo.md # Detalhamento dos campos e tipos
├── src/
│   ├── bronze/
│   │   ├── 01_ingest_pesquisas.py      # Ingestão raw de pesquisas de egressos
│   │   └── 02_ingest_academico.py      # Ingestão de dados de matrículas e cursos
│   ├── silver/
│   │   ├── 01_clean_egressos.py        # Limpeza, deduplicação e filtros de qualidade
│   │   └── 02_clean_cursos.py          # Harmonização do catálogo de cursos
│   └── gold/
│       ├── 01_build_dim_egresso.py     # Carga da dimensão egresso
│       ├── 02_build_dim_curso.py       # Carga da dimensão curso
│       ├── 03_build_dim_emprego.py     # Carga da dimensão emprego (MD5)
│       ├── 04_build_dim_tempo.py       # Carga da dimensão tempo
│       └── 05_build_fato_egressos.py   # Carga da fato e execução do Z-Ordering
├── sql/
│   ├── ddl/
│   │   ├── create_bronze_tables.sql    # DDL da camada Bronze
│   │   ├── create_silver_tables.sql    # DDL da camada Silver
│   │   └── create_gold_tables.sql      # DDL da camada Gold
│   └── queries/
│       ├── bi_indicadores_empregabilidade.sql # Queries prontas para dashboards
│       ├── bi_analise_nps_cursos.sql
│       └── bi_funil_propensao_pos.sql
└── workflows/
    └── databricks_job_pipeline.json    # Definição da DAG de execução dos workflows
```

---

## ⚙️ 8. Guia de Instalação e Execução

### 8.1. Pré-requisitos
- **Databricks Workspace** (Runtime 11.3 LTS PySpark / Delta Lake).
- Permissões de administrador ou criação de esquemas no Catálogo.
- **Python 3.10+** (para desenvolvimentos locais / testes unitários).

### 8.2. Passo a Passo de Implantação

1. **Clonar o repositório no Databricks Repos:**
   ```bash
   git clone https://github.com/seu-usuario/mvp-engenharia-dados.git
   ```

2. **Criar a Estrutura de Schemas no Databricks SQL:**
   ```sql
   CREATE DATABASE IF NOT EXISTS mvp_eng_dados;
   CREATE DATABASE IF NOT EXISTS mvp_eng_dados_bronze;
   CREATE DATABASE IF NOT EXISTS mvp_eng_dados_silver;
   ```

3. **Executar a Esteira de Dados (Workflows):**
   - **Método 1 (Automatizado):** Importe o arquivo `workflows/databricks_job_pipeline.json` no **Databricks Workflows** e clique em **Run Now**.
   - **Método 2 (Manual por Notebooks):**
     1. Execute `src/bronze/01_ingest_pesquisas.py`
     2. Execute `src/silver/01_clean_egressos.py`
     3. Execute `src/gold/05_build_fato_egressos.py`

---

## 📊 9. Exemplos de Consultas Analíticas (SQL para BI)

### 9.1. Média de Renda e Empregabilidade por Curso e Modalidade
```sql
SELECT 
    c.nome_curso,
    c.modalidade_graduacao,
    COUNT(DISTINCT f.id_egresso) AS total_egressos,
    ROUND(AVG(f.renda_mensal_estimada), 2) AS renda_media,
    SUM(CASE WHEN e.nivel_cargo != 'SEM VÍNCULO' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS taxa_empregabilidade_pct
FROM mvp_eng_dados.gold.fato_egressos f
JOIN mvp_eng_dados.gold.dim_curso c ON f.id_curso = c.id_curso
JOIN mvp_eng_dados.gold.dim_emprego e ON f.id_emprego = e.id_emprego
GROUP BY c.nome_curso, c.modalidade_graduacao
ORDER BY renda_media DESC;
```

### 9.2. Distribuição do NPS e Egressos com Alto Potencial de Pós-Graduação
```sql
SELECT 
    c.area_conhecimento,
    f.categoria_nps,
    COUNT(f.id_egresso) AS qtd_egressos,
    SUM(f.potencial_matricula_pos) AS alvo_campanha_pos
FROM mvp_eng_dados.gold.fato_egressos f
JOIN mvp_eng_dados.gold.dim_curso c ON f.id_curso = c.id_curso
GROUP BY c.area_conhecimento, f.categoria_nps
ORDER BY c.area_conhecimento, qtd_egressos DESC;
```

---

## 🤝 10. Contribuição e Manutenção

Para contribuir com novas funcionalidades ou melhorias nos pipelines:
1. Crie uma *Feature Branch* (`git checkout -b feature/nova-dimensao`).
2. Adicione as validações de qualidade necessárias na camada Silver.
3. Submeta um *Pull Request* detalhando os impactos no modelo dimensional.

---
*Projeto desenvolvido como MVP de Engenharia de Dados sobre Databricks e Delta Lake.*
