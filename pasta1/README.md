# 🎓 Data Mart de Acompanhamento de Egressos

![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=for-the-badge&logo=Databricks&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-000000?style=for-the-badge&logo=delta&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=for-the-badge&logo=postgresql&logoColor=white)
![Architecture](https://img.shields.io/badge/Architecture-Medallion-blue?style=for-the-badge)

## 📌 Visão Geral do Projeto

Este repositório contém a implementação do **Data Mart de Acompanhamento de Egressos**, desenvolvido utilizando a **Arquitetura Medallion** sobre a plataforma **Databricks Delta Lake**. O objetivo principal é estruturar, transformar e disponibilizar dados consolidados sobre a trajetória acadêmica, demográfica e profissional dos ex-alunos da instituição.

Com a estrutura dimensional em **Esquema Estrela (Star Schema)**, equipes de Business Intelligence e Analytics podem responder a perguntas estratégicas como:
- Qual é a taxa de empregabilidade e a renda estimada dos egressos por curso e modalidade?
- Qual o nível de satisfação (NPS) e engajamento da comunidade Alumni?
- Quais egressos apresentam alto potencial de captação para programas de Pós-Graduação?
- Qual é a distribuição geográfica e a inserção no mercado de trabalho (Setor Público, Privado, Terceiro Setor)?

---

## 🏗️ Arquitetura de Dados (Medallion Architecture)

O pipeline de dados segue o padrão Medallion em 3 camadas no banco de dados `mvp_eng_dados`:

```text
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────────────────┐
│  Camada BRONZE  │  ───> │  Camada SILVER  │  ───> │         Camada GOLD         │
│  (Raw Ingestion)│       │  (Cleansed)     │       │   (Data Mart Dimensional)   │
└─────────────────┘       └─────────────────┘       └─────────────────────────────┘
  • Ingestão raw            • Limpeza e validação     • Esquema Estrela (Star Schema)
  • Logs e pesquisas        • Padronizações           • Fato: fato_egressos
  • Dados de cursos         • Deduplicação            • Dimensões: dim_egresso,
                                                        dim_curso, dim_emprego
```

1. **Bronze (Raw):** Ingestão dos dados brutos capturados de formulários de acompanhamento e sistemas acadêmicos.
2. **Silver (Cleansed/Refined):** Tabelas higienizadas (`silver.dataset_egressos` e `silver.dataset_cursos`), com aplicação de regras de sanidade (ex: validação de idade entre 16 e 100 anos, eliminação de valores negativos em renda).
3. **Gold (Curated / Analytics):** Modelo dimensional em esquema estrela no database `mvp_eng_dados.gold`, otimizado para consumo por ferramentas analíticas.

---

## 📐 Modelo Dimensional (Esquema Estrela)

```text
                       ┌──────────────────────┐
                       │      dim_curso       │
                       ├──────────────────────┤
                       │ PK  id_curso         │
                       │     nome_curso       │
                       │     area_conhecimento│
                       │     duracao_semestres│
                       │     mensalidade_base │
                       │     modalidade_...   │
                       └──────────┬───────────┘
                                  │
                                  │ 1:N
┌──────────────────────┐          ▼          ┌──────────────────────┐
│     dim_egresso      │   ┌──────────────┐  │     dim_emprego      │
├──────────────────────┤   │ fato_egressos│  ├──────────────────────┤
│ PK  id_egresso       ├──>│──────────────│<──┤ PK  id_emprego       │
│     idade_egresso    │1:N│ FK id_egresso│1:N│     nivel_cargo      │
│     uf_residencia    │   │ FK id_curso  │  │     tipo_empresa     │
│     bolsista_grad... │   │ FK id_emprego│  └──────────────────────┘
└──────────────────────┘   │    renda_... │
                           │    satisf... │
                           └──────────────┘
```

---

## 📖 Dicionário Lógico de Dados (`mvp_eng_dados.gold`)

### 1. Tabela Fato: `fato_egressos`
Consolida as métricas operacionais, financeiras e comportamentais do acompanhamento de egressos.

| Nome da Coluna | Tipo de Dado | Chave | Nulo? | Origem (Silver) | Descrição / Regras de Negócio |
| :--- | :--- | :---: | :---: | :--- | :--- |
| **`id_egresso`** | `INT` | **FK** | Não | `silver.dataset_egressos` | Chave estrangeira para a dimensão `dim_egresso`. |
| **`id_curso`** | `INT` | **FK** | Não | `silver.dataset_egressos` | Chave estrangeira para a dimensão `dim_curso`. |
| **`id_emprego`** | `STRING` | **FK** | Não | Calculado (Hash) | Chave estrangeira (MD5) para a dimensão `dim_emprego`. |
| **`dt_ultimo_contato`** | `TIMESTAMP` | - | Sim | `silver.dataset_egressos` | Data e hora em que foi realizado o último contato de acompanhamento. |
| **`renda_mensal_estimada`** | `DECIMAL(10,2)` | - | Sim | `silver.dataset_egressos` | Renda mensal estimada do egresso em R$. Valores negativos são anulados. |
| **`satisfacao_graduacao_nps`** | `INT` | - | Sim | `silver.dataset_egressos` | Nota de satisfação NPS concedida pelo egresso (0 a 10). |
| **`categoria_nps`** | `STRING` | - | Sim | Derivado (Regra) | Classificação NPS: 'Promotor' (≥ 9), 'Neutro' (7 ou 8), 'Detrator' (≤ 6). |
| **`engajamento_alumni_score`** | `INT` | - | Sim | `silver.dataset_egressos` | Pontuação de engajamento do ex-aluno na rede Alumni (0 a 100). |
| **`potencial_matricula_pos`** | `INT` | - | Sim | `silver.dataset_egressos` | Flag binário (1 = Alto Potencial em Pós-graduação, 0 = Sem Interesse). |
| **`meses_desde_formacao`** | `INT` | - | Sim | `silver.dataset_egressos` | Quantidade de meses decorridos entre a colação de grau e o acompanhamento. |
| **`faixa_meses_formacao`** | `STRING` | - | Sim | Derivado (Regra) | Agrupamento temporal: '0-12m', '13-24m', '25-36m', '37-48m', '49-60m' ou '>60m'. |

---

### 2. Tabela Dimensão: `dim_egresso`
Contém o perfil demográfico individualizado do egresso.

| Nome da Coluna | Tipo de Dado | Chave | Nulo? | Origem (Silver) | Descrição / Regras de Negócio |
| :--- | :--- | :---: | :---: | :--- | :--- |
| **`id_egresso`** | `INT` | **PK** | Não | `silver.dataset_egressos` | Identificador único do egresso (Chave Primária de Negócio). |
| **`idade_egresso`** | `INT` | - | Sim | `silver.dataset_egressos` | Idade atual em anos (filtrado no intervalo de 16 a 100 anos). |
| **`uf_residencia`** | `STRING` | - | Sim | `silver.dataset_egressos` | Estado de residência do egresso (Sigla padronizada em maiúsculas: SP, RJ, MG...). |
| **`bolsista_graduacao`** | `INT` | - | Sim | `silver.dataset_egressos` | Indicador binário de bolsa de estudo durante o curso (1 = Bolsista, 0 = Não Bolsista). |

---

### 3. Tabela Dimensão: `dim_curso`
Catálogo de cursos de graduação e seus atributos acadêmicos e financeiros.

| Nome da Coluna | Tipo de Dado | Chave | Nulo? | Origem (Silver) | Descrição / Regras de Negócio |
| :--- | :--- | :---: | :---: | :--- | :--- |
| **`id_curso`** | `INT` | **PK** | Não | `silver.dataset_cursos` | Identificador único do curso (Chave Primária). |
| **`nome_curso`** | `STRING` | - | Não | `silver.dataset_cursos` | Nome do curso de graduação (padronizado em maiúsculas sem espaços extras). |
| **`area_conhecimento`** | `STRING` | - | Sim | `silver.dataset_cursos` | Área acadêmica (Tecnologia, Exatas, Humanas, Saúde, etc.). |
| **`duracao_semestres`** | `INT` | - | Sim | `silver.dataset_cursos` | Duração regular do curso em semestres (valores > 0). |
| **`mensalidade_base`** | `DECIMAL(10,2)` | - | Sim | `silver.dataset_cursos` | Mensalidade base de referência em reais (R$). |
| **`modalidade_graduacao`** | `STRING` | - | Sim | `silver.dataset_egressos` | Modalidade de ensino ofertada: 'PRESENCIAL', 'EAD' ou 'HÍBRIDO'. |

---

### 4. Tabela Dimensão: `dim_emprego`
Dimensão categorizada que mapeia o perfil profissional no mercado de trabalho.

| Nome da Coluna | Tipo de Dado | Chave | Nulo? | Origem (Silver) | Descrição / Regras de Negócio |
| :--- | :--- | :---: | :---: | :--- | :--- |
| **`id_emprego`** | `STRING` | **PK** | Não | Calculado (Hash) | Surrogate Key gerada via Hash MD5 de `nivel_cargo || '||' || tipo_empresa`. |
| **`nivel_cargo`** | `STRING` | - | Sim | `silver.dataset_egressos` | Nível hierárquico: 'ESTÁGIO', 'JÚNIOR', 'PLENO', 'SÊNIOR', 'GESTÃO', 'SEM VÍNCULO'. |
| **`tipo_empresa`** | `STRING` | - | Sim | `silver.dataset_egressos` | Segmento empregador: 'INICIATIVA PRIVADA', 'SETOR PÚBLICO', 'TERCEIRO SETOR', 'DESEMPREGADO'. |

---

## ⚡ Regras de Negócio & Transformações

1. **Geração de Surrogate Key (`dim_emprego`):**
   - Utilização de algoritmo Hash MD5 concatenando os atributos para garantir idempotência:
     ```sql
     md5(concat(coalesce(nivel_cargo, 'N/A'), '||', coalesce(tipo_empresa, 'N/A'))) AS id_emprego
     ```

2. **Categorização do NPS:**
   - `Promotor`: Notas 9 e 10
   - `Neutro`: Notas 7 e 8
   - `Detrator`: Notas 0 a 6

3. **Agrupamento Temporal (`faixa_meses_formacao`):**
   - Intervalos: `0-12m`, `13-24m`, `25-36m`, `37-48m`, `49-60m` e `>60m`.

4. **Tratamento de Anomalias:**
   - Rendas mensais negativas são anuladas na camada Silver.
   - Idades fora do intervalo biológico de 16 a 100 anos são desconsideradas.

---

## 🚀 Otimizações de Desempenho (Databricks Delta Lake)

Para garantir respostas rápidas nas consultas analíticas e nos dashboards de BI, o Data Mart aplica **Z-Ordering** nas tabelas físicas:

```sql
-- Otimização da Tabela Fato
OPTIMIZE mvp_eng_dados.gold.fato_egressos 
ZORDER BY (id_curso, id_egresso);

-- Otimização das Tabelas Dimensão
OPTIMIZE mvp_eng_dados.gold.dim_curso ZORDER BY (id_curso);
OPTIMIZE mvp_eng_dados.gold.dim_egresso ZORDER BY (id_egresso);
```

---

## 📂 Estrutura do Repositório

```text
├── config/
│   └── delta_config.json          # Configurações de ambiente e conexões Databricks
├── docs/
│   ├── dicionario_de_dados.md     # Documentação detalhada dos metadados
│   └── modelo_dimensional.png     # Diagrama ER em alta resolução
├── notebooks/
│   ├── 01_bronze_ingestion.py     # Carga dos dados brutos
│   ├── 02_silver_cleansing.py     # Tratamento, limpeza e filtros de sanidade
│   └── 03_gold_star_schema.py     # Carga do modelo dimensional e Z-Ordering
├── sql/
│   ├── ddl_gold_tables.sql        # Scripts DDL de criação das tabelas
│   └── analytical_queries.sql     # Queries analíticas de exemplo para BI
├── README.md                      # Documentação principal do repositório
└── requirements.txt               # Dependências do projeto PySpark / Databricks
```

---

## 🛠️ Como Executar

### Pré-requisitos
- **Databricks** (Runtime 11.3 LTS ou superior)
- **Delta Lake** ativado
- Permissões de escrita no catálogo/schema `mvp_eng_dados`

### Passo a Passo
1. Clone este repositório no seu Databricks Workspace / Repos:
   ```bash
   git clone https://github.com/usuario/data-mart-egressos.git
   ```
2. Execute o notebook `notebooks/01_bronze_ingestion.py` para realizar a ingestão.
3. Execute o notebook `notebooks/02_silver_cleansing.py` para higienização dos dados.
4. Execute o notebook `notebooks/03_gold_star_schema.py` para popular a camada Gold e executar a otimização **Z-Ordering**.
