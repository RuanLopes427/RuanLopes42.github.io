# Análise de Turnover & Risco de Retenção
### People Analytics | SQL Server · Power BI · DAX · Star Schema
**Ruan Estevão Barbosa Lopes** | 2026

---

## Visão Geral do Projeto

| Item | Detalhe |
|---|---|
| Tipo | End-to-end: Extração → Modelagem SQL → Dashboard Power BI |
| Dataset | IBM HR Analytics Employee Attrition & Performance (Kaggle) |
| Banco de dados | SQL Server — DW_PeopleAnalytics |
| Ferramenta de BI | Power BI Desktop + Power BI Service |
| Registros analisados | 1.470 colaboradores · 35 variáveis originais |
| Páginas do dashboard | 5 (Capa, Perfil, Desligamentos, Análise de Risco, Conclusões) |

---

## 1. Problema de Negócio

Turnover elevado é um dos maiores custos ocultos das organizações. Substituir um colaborador custa entre 1,5× e 2× o seu salário anual — incluindo recrutamento, treinamento e perda de produtividade. Apesar disso, a maioria das empresas só percebe o problema quando o colaborador já entregou o crachá.

**A pergunta central do projeto:**
> *"Por que nossos talentos estão saindo — e quem vai sair a seguir?"*

O projeto responde a essa pergunta em três camadas:

- **Descritiva:** quem saiu, quando saiu e de qual área
- **Diagnóstica:** quais fatores estão associados às saídas
- **Preditiva:** quais colaboradores ativos têm maior risco de desligamento

---

## 2. Fonte de Dados

**Dataset:** IBM HR Analytics Employee Attrition & Performance
**Origem:** Kaggle — `kaggle.com/datasets/pavansubhasht/ibm-hr-analytics-attrition-dataset`
**Formato original:** arquivo CSV único com 1.470 linhas e 35 colunas
**Natureza dos dados:** dados simulados replicando estrutura de ERP de RH (SAP/Workday)

**Principais campos do dataset original:**

| Campo | Tipo | Descrição |
|---|---|---|
| EmployeeNumber | INT | Identificador único do colaborador |
| Attrition | VARCHAR | Se o colaborador saiu (Yes/No) |
| Department | VARCHAR | Departamento |
| JobRole | VARCHAR | Cargo |
| JobLevel | INT | Nível hierárquico (1–5) |
| MonthlyIncome | INT | Salário mensal |
| Age | INT | Idade |
| YearsAtCompany | INT | Anos de casa |
| YearsSinceLastPromotion | INT | Anos desde a última promoção |
| OverTime | VARCHAR | Faz hora extra (Yes/No) |
| JobSatisfaction | INT | Satisfação no cargo (1–4) |
| WorkLifeBalance | INT | Equilíbrio vida-trabalho (1–4) |
| EnvironmentSatisfaction | INT | Satisfação com o ambiente (1–4) |
| RelationshipSatisfaction | INT | Satisfação com relacionamentos (1–4) |
| Education | INT | Grau de educação (1–5) |
| EducationField | VARCHAR | Área de formação |

---

## 3. Processo de Modelagem — SQL Server

### 3.1 Criação do Banco de Dados

```sql
CREATE DATABASE DW_PeopleAnalytics;
USE DW_PeopleAnalytics;
```

### 3.2 Staging Layer

O CSV foi importado via SQL Server Import Wizard (Tasks → Import Flat File) criando a tabela de staging `stg_HR_Attrition` com os dados brutos.

### 3.3 Modelo Estrela (Star Schema)

O dataset foi desnormalizado em **1 tabela fato + 5 dimensões**, seguindo boas práticas de Data Warehouse:

```
fColaboradores (FATO)
    ├── dDepartamento (dimensão)
    ├── dCargo (dimensão)
    ├── dFaixaEtaria (dimensão)
    ├── dEducacao (dimensão)
    └── dSatisfacao (dimensão)
```

---

### 3.4 Scripts de Criação das Dimensões

**dDepartamento**
```sql
SELECT IDENTITY(INT, 1, 1) AS ID_Depto,
       Department AS Departamento
INTO dDepartamento
FROM (SELECT DISTINCT Department FROM stg_HR_Attrition) sub;
```

**dCargo**
```sql
SELECT IDENTITY(INT, 1, 1) AS ID_Cargo,
    JobRole  AS Cargo,
    JobLevel AS Nivel
INTO dCargo
FROM (SELECT DISTINCT JobRole, JobLevel FROM stg_HR_Attrition) sub;
```

**dFaixaEtaria**
```sql
SELECT IDENTITY(INT, 1, 1) AS ID_Idade,
    Age AS Idade,
    CASE
        WHEN Age BETWEEN 18 AND 25 THEN 'Jovem'
        WHEN Age BETWEEN 26 AND 35 THEN 'Adulto Jovem'
        WHEN Age BETWEEN 36 AND 45 THEN 'Adulto'
        WHEN Age BETWEEN 46 AND 55 THEN 'Senior'
        ELSE 'Maduro'
    END AS FaixaEtaria,
    CASE
        WHEN Age BETWEEN 14 AND 27 THEN 'Geração Z'
        WHEN Age BETWEEN 28 AND 43 THEN 'Millennial'
        WHEN Age BETWEEN 44 AND 59 THEN 'Geração X'
        ELSE 'Baby Boomer'
    END AS Geracao
INTO dFaixaEtaria
FROM (SELECT DISTINCT Age FROM stg_HR_Attrition) t;
```

**dEducacao**
```sql
SELECT IDENTITY(INT, 1, 1) AS ID_Educacao,
    EducationField AS AreaFormacao,
    Education      AS Grau
INTO dEducacao
FROM (SELECT DISTINCT EducationField, Education FROM stg_HR_Attrition) t;
```

**dSatisfacao**
```sql
SELECT IDENTITY(INT, 1, 1) AS ID_Sat,
    stg.JobSatisfaction,
    stg.EnvironmentSatisfaction,
    stg.WorkLifeBalance,
    stg.RelationshipSatisfaction,
    CASE stg.JobSatisfaction
        WHEN 1 THEN 'Baixa'  WHEN 2 THEN 'Média'
        WHEN 3 THEN 'Alta'   WHEN 4 THEN 'Muito Alta'
    END AS Satisfacao_Cargo,
    CASE stg.WorkLifeBalance
        WHEN 1 THEN 'Ruim'   WHEN 2 THEN 'Regular'
        WHEN 3 THEN 'Bom'    WHEN 4 THEN 'Excelente'
    END AS Qualidade_WLB
INTO dSatisfacao
FROM (
    SELECT DISTINCT JobSatisfaction, EnvironmentSatisfaction,
                    WorkLifeBalance, RelationshipSatisfaction
    FROM stg_HR_Attrition
) stg;
```

---

### 3.5 Tabela Fato — fColaboradores

```sql
SELECT
    stg.EmployeeNumber       AS ID_Colaborador,
    dep.ID_Depto,
    car.ID_Cargo,
    ida.ID_Idade,
    edu.ID_Educacao,
    sat.ID_Sat,
    stg.MonthlyIncome        AS Salario,
    stg.YearsAtCompany,
    stg.YearsSinceLastPromotion,
    stg.NumCompaniesWorked,
    stg.DistanceFromHome,
    stg.PerformanceRating,
    stg.OverTime,
    CASE WHEN stg.Attrition = 1 THEN 'Yes' ELSE 'No' END AS Attrition,
    stg.Attrition AS Attrition_Flag
INTO fColaboradores
FROM stg_HR_Attrition stg
JOIN dDepartamento dep ON stg.Department           = dep.Departamento
JOIN dCargo        car ON stg.JobRole              = car.Cargo
                      AND stg.JobLevel             = car.Nivel
JOIN dFaixaEtaria  ida ON stg.Age                  = ida.Idade
JOIN dEducacao     edu ON stg.Education            = edu.Grau
                      AND stg.EducationField       = edu.AreaFormacao
JOIN dSatisfacao   sat ON stg.JobSatisfaction      = sat.JobSatisfaction
                      AND stg.EnvironmentSatisfaction = sat.EnvironmentSatisfaction
                      AND stg.WorkLifeBalance      = sat.WorkLifeBalance
                      AND stg.RelationshipSatisfaction = sat.RelationshipSatisfaction;
```

---

### 3.6 Score de Risco Preditivo (SQL)

Após a criação da fato, foi implementado um modelo de score preditivo com 4 fatores de risco:

```sql
ALTER TABLE fColaboradores
ADD Score_Risco INT,
    Nivel_Risco NVARCHAR(20);

UPDATE fColaboradores
SET Score_Risco =
    CASE WHEN OverTime = 'Yes' THEN 25 ELSE 0 END +
    CASE WHEN JobSatisfaction <= 2 THEN 25
         WHEN JobSatisfaction = 3  THEN 10 ELSE 0 END +
    CASE WHEN WorkLifeBalance <= 2 THEN 25
         WHEN WorkLifeBalance = 3  THEN 10 ELSE 0 END +
    CASE WHEN YearsAtCompany <= 2  THEN 25
         WHEN YearsAtCompany <= 5  THEN 10 ELSE 0 END
FROM fColaboradores f
JOIN dSatisfacao s ON f.ID_Sat = s.ID_Sat;

UPDATE fColaboradores
SET Nivel_Risco =
    CASE
        WHEN Score_Risco >= 75 THEN 'Critico'
        WHEN Score_Risco >= 50 THEN 'Alto'
        WHEN Score_Risco >= 25 THEN 'Médio'
        ELSE 'Baixo'
    END;
```

**Lógica dos 4 fatores (25 pontos cada):**

| Fator | Condição de risco alto | Pontos |
|---|---|---|
| Hora extra | Faz overtime | 25 |
| Satisfação no cargo | Nível 1 ou 2 | 25 |
| Work-life balance | Nível 1 ou 2 | 25 |
| Tempo de casa | 2 anos ou menos | 25 |

---

### 3.7 Consultas SQL Analíticas Avançadas

**Análise de attrition por tempo de casa (com CTE)**
```sql
-- Pareto de saídas por departamento com percentual acumulado
SELECT
    d.Departamento,
    Saidas,
    Total,
    ROUND(100.0 * Saidas / Total, 1) AS Attrition_Pct,
    ROUND(
        100.0 * SUM(Saidas) OVER (ORDER BY Saidas DESC ROWS UNBOUNDED PRECEDING)
        / SUM(Saidas) OVER (), 1
    ) AS Pct_Acumulado
FROM (
    SELECT f.ID_Depto,
           COUNT(*) AS Total,
           SUM(CASE WHEN f.Attrition = 'Yes' THEN 1 ELSE 0 END) AS Saidas
    FROM fColaboradores f
    GROUP BY f.ID_Depto
) sub
JOIN dDepartamento d ON sub.ID_Depto = d.ID_Depto
ORDER BY Attrition_Pct DESC;
```

---

## 4. Modelagem no Power BI

### 4.1 Conexão e Importação

Conexão via **SQL Server (Import mode)** importando as 6 tabelas do DW_PeopleAnalytics.

### 4.2 Colunas Calculadas (DAX)

```dax
// Faixa de tempo de casa — para análise de curva de retenção
Faixa Tempo Casa =
SWITCH(TRUE(),
    fColaboradores[YearsAtCompany] = 0,   "Menos de 1 Ano",
    fColaboradores[YearsAtCompany] <= 2,  "1-2 Anos",
    fColaboradores[YearsAtCompany] <= 5,  "3-5 Anos",
    fColaboradores[YearsAtCompany] <= 10, "6-10 Anos",
    "Mais de 10 Anos"
)

// Ordenação lógica de risco — evita ordem alfabética no gráfico
Ordem Risco =
SWITCH(TRUE(),
    fColaboradores[Score_Risco] >= 75, 1,
    fColaboradores[Score_Risco] >= 50, 2,
    fColaboradores[Score_Risco] >= 25, 3,
    4
)
```

---

### 4.3 Medidas DAX — Hierarquia de dependência

**Medidas base:**
```dax
Headcount =
COUNTROWS(fColaboradores)

Total Saidas =
CALCULATE(COUNTROWS(fColaboradores), fColaboradores[Attrition] = "Yes")

Taxa de Turnover% =
DIVIDE([Total Saidas], [Headcount])

Qtd Funcionários Ativos =
CALCULATE(COUNTROWS(fColaboradores), fColaboradores[Attrition] = "No")

Média Salarial =
AVERAGE(fColaboradores[Salario])
```

**Medidas de risco:**
```dax
Qtd Funcionários Risco Crítico =
CALCULATE(
    COUNTROWS(fColaboradores),
    fColaboradores[Nivel_Risco] = "Critico",
    fColaboradores[Attrition] = "No"
)

% Risco Crítico func em atividade =
VAR TotalFunc =
    CALCULATE([Qtd Funcionários Ativos])
VAR FuncCritico =
    CALCULATE([Qtd Funcionários Risco Crítico],
              fColaboradores[Attrition] = "No")
RETURN DIVIDE(FuncCritico, TotalFunc, 0)
```

**Medida financeira:**
```dax
Custo colaboradores desligados =
VAR media_salarial = [Media salarial funcionarios desligados]
VAR custo_anual    = (media_salarial * 12) * 1.5
VAR total_desligamentos = [Qtd. Funcionários Desligados]
RETURN total_desligamentos * custo_anual
-- Benchmark: custo de substituição = 1,5× o salário anual
-- Inclui recrutamento, treinamento e perda de produtividade
```

**Análise de defasagem salarial:**
```dax
% diferença Média Salarial critico baixo risco =
VAR critico =
    CALCULATE(
        Medidas[Média Salarial],
        ALL(fColaboradores),
        fColaboradores[Nivel_Risco] = "Critico",
        fColaboradores[Attrition] = "No"
    )
VAR baixo =
    CALCULATE(
        Medidas[Média Salarial],
        ALL(fColaboradores),
        fColaboradores[Nivel_Risco] = "Baixo",
        fColaboradores[Attrition] = "No"
    )
RETURN DIVIDE(baixo - critico, baixo, 0)
-- Resultado: 13,01% de defasagem salarial
```

**Medidas de cor dinâmica (preattentive attributes):**
```dax
// Destaca dinamicamente a barra de maior valor
Cor Barra Departamento =
VAR MaxValor =
    MAXX(ALL(dDepartamento[Departamento]),
         CALCULATE([Qtd. Funcionários]))
VAR ValorAtual = [Qtd. Funcionários]
RETURN IF(ValorAtual = MaxValor, "#094D37", "#5CAD8A")

// Semáforo de risco — encodes urgency visually
Cor Barra Risco =
IF(HASONEVALUE(fColaboradores[Nivel_Risco]),
    SWITCH(VALUES(fColaboradores[Nivel_Risco]),
        "Critico", "#DC3545",
        "Alto",    "#FD7E14",
        "Médio",   "#FFC107",
        "Baixo",   "#0F9D58"
    ), "#5CAD8A"
)
```

**Insights dinâmicos (texto gerado por DAX):**
```dax
Insight Desligamentos =
VAR TopDepto =
    TOPN(1, VALUES(dDepartamento[Departamento]), [Taxa de Turnover%])
VAR Taxa =
    CALCULATE([Taxa de Turnover%],
    TOPN(1, VALUES(dDepartamento[Departamento]), [Taxa de Turnover%]))
VAR Diferenca = FORMAT(Taxa - [Taxa de Turnover%], "0,00%")
RETURN
"⚠ " & CONCATENATEX(TopDepto, dDepartamento[Departamento])
& " tem a taxa de turnover " & Diferenca
& " acima da média da empresa"
-- Resultado: "⚠ Sales tem a taxa de turnover 4,51% acima da média"

Insight Geração =
"👥 " & CONCATENATEX(GeracaoDom, dFaixaEtaria[Geracao])
& " representa " & FORMAT(Pct, "0,00%")
& " da força de trabalho ativa"
-- Resultado: "👥 Millennial representa 63,02% da força de trabalho ativa"
```

---

## 5. Estrutura do Dashboard

### Página 1 — Capa Executiva

**Objetivo:** primeiro impacto e navegação

Elementos:
- Título: *"Por que nossos talentos estão saindo?"*
- Subtítulo: *"Análise de Turnover & Risco de Retenção"*
- 4 KPIs: Qtd. Funcionários · % Desligamentos · Média Salarial · Qtd Risco Crítico
- Frase de impacto (DAX): *"1 em cada 3 colaboradores deixa a empresa antes de completar o primeiro ano."*
- 4 cards de navegação com ícones
- Assinatura: *"Desenvolvido por Ruan Lopes · 2026"*

---

### Página 2 — Perfil Força de Trabalho

**Objetivo:** responder *"quem são nossos colaboradores?"*

Elementos:
- Botão de alternância: Funcionários em Atividade / Total Funcionários
- 4 KPIs: Total Funcionários · Média Salarial · % Desligamentos · Tempo Médio na Empresa
- Gráfico de barras: Funcionários por Departamento *(destaque dinâmico na maior barra)*
- Gráfico de barras: Funcionários por Cargo *(destaque dinâmico)*
- Gráfico de barras: Funcionários por Área de Formação *(destaque dinâmico)*
- Gráfico de barras: Funcionários por Geração
- Insight dinâmico: geração dominante com percentual

---

### Página 3 — Desligamentos

**Objetivo:** responder *"quem está saindo e por quê?"*

Elementos:
- 2 cards de comparação: Turnover com Overtime (30,53%) vs sem Overtime (10,44%)
- Gráfico de barras: % Desligamentos por Departamento *(destaque dinâmico)*
- Insight dinâmico: departamento com maior desvio da média
- Gráfico de linha: % Desligamentos por Tempo de Casa *(curva descendente)*
- Gráfico de barras: % Desligamentos por Cargo *(destaque dinâmico)*
- Gráfico de barras: % Desligamentos por Qualidade de Vida no Trabalho

---

### Página 4 — Análise de Risco

**Objetivo:** responder *"quem vai sair se não agirmos?"*

Elementos:
- 3 KPIs: Func. Risco Crítico · % Risco Crítico · Média Salarial Críticos
- Gráfico de barras semafórico: distribuição por Nível de Risco *(vermelho/laranja/amarelo/verde)*
- Matriz: Departamento × Nível de Risco com formatação condicional
- Gráfico de barras: Qtd Risco Crítico por Departamento
- Scatter plot: Relação entre Salário e Risco de Saída *(4 bolhas coloridas)*
- Insight narrativo: *"Quanto menor o salário, maior o risco de saída..."*

---

### Página 5 — Conclusões & Recomendações

**Objetivo:** transformar análise em decisão executiva

Elementos:
- Card de impacto financeiro: $20,42 Mi custo estimado de turnover
- Subtexto metodológico com benchmark de mercado
- 3 blocos Problema → Evidência → Recomendação:
  - 🟡 Onboarding Crítico
  - 🔴 Defasagem Salarial
  - 🟠 Impacto do Overtime
- Frase de encerramento com projeção de economia

---

## 6. KPIs e Métricas Principais

| KPI | Valor | Significado |
|---|---|---|
| Total de colaboradores | 1.470 | Base da análise |
| Taxa de Turnover | 16,12% | Acima da média do setor (10–12%) |
| Média Salarial | $6,50 Mil | Referência da força de trabalho |
| Tempo Médio de Casa | 7 anos | Índice de senioridade |
| Colaboradores em Risco Crítico | 60 (4,87%) | Prioridade de ação imediata |
| Média Salarial dos Críticos | $5,14 Mil | 13% abaixo dos de baixo risco |
| Custo estimado de turnover | $20,42 Mi | Impacto financeiro do problema |
| Turnover com Overtime | 30,53% | 3× acima de quem não faz hora extra |
| Turnover sem Overtime | 10,44% | Referência de saúde organizacional |
| Saídas no 1º ano | 34,88% | Onboarding crítico |
| Defasagem salarial Crítico vs Baixo | 13,01% | Driver de risco confirmado |

---

## 7. Principais Insights

### Insight 1 — Crise no onboarding
**34,88%** dos colaboradores saem antes de completar o primeiro ano. Colaboradores no 1º ano saem **178% mais** do que os de 1 a 2 anos de casa. A taxa cai progressivamente: 34,88% → 21,26% → 13,82% → 12,28% → 8,13%. O risco se concentra nos 24 primeiros meses de empresa.

### Insight 2 — Overtime tóxico
Colaboradores que fazem hora extra têm taxa de turnover de **30,53%** — quase **3× mais** do que os que não fazem (10,44%). Sales e Research & Development são os departamentos com maior incidência de overtime e maior risco.

### Insight 3 — Defasagem salarial como driver de risco
Colaboradores em risco crítico ganham em média **13% menos** do que os de baixo risco ($5,14 Mil vs $6,99 Mil). O scatter confirma: quanto menor o salário, maior o score de risco — o Crítico está isolado no canto superior esquerdo do gráfico.

### Insight 4 — Sales lidera as saídas
O departamento Sales tem **20,63% de attrition** — 4,51 pontos percentuais acima da média da empresa. O cargo Sales Representative registra **39,76%** de turnover — o mais alto de todos os cargos.

### Insight 5 — Concentração geracional
**Millennials representam 63,02%** da força de trabalho ativa. Isso significa que políticas de retenção precisam ser desenhadas para as expectativas dessa geração: propósito, crescimento e equilíbrio vida-trabalho.

### Insight 6 — Custo financeiro mensurável
O turnover no período gerou um custo estimado de **$20,42 Milhões** (benchmark: 1,5× o salário anual por colaborador). Reduzir o turnover em 30% com as ações recomendadas representaria uma economia estimada de ~$6 Milhões.

---

## 8. Desafios Enfrentados e Soluções

| Desafio | Solução implementada |
|---|---|
| Coluna `Attrition` importada como BIT pelo Import Wizard | `CASE WHEN stg.Attrition = 1 THEN 'Yes' ELSE 'No'` na criação da fato |
| Nomes de colunas com acento gerando erro nos JOINs | Padronização sem acento nos nomes de campo SQL; renomeação no Power BI |
| Dependência circular ao criar coluna `Ordem Risco` | Referenciar `Score_Risco` numérico diretamente em vez de `Nivel_Risco` |
| MAXX recebendo argumentos demais | Refatoração com VAR para separar a expressão da tabela |
| Contexto de filtro distorcendo cálculo de defasagem salarial | Uso de `ALL(fColaboradores)` para remover filtros externos nas VARs |
| SELECTEDVALUE não funcionando em título dinâmico | Substituição por título fixo neutro; lógica dinâmica mantida em cards de texto |
| Cores do semáforo propagando para outras páginas | Medidas de cor específicas por página com lógica independente |
| Tabela com 1.200 linhas sem valor executivo | Substituída por Matriz Departamento × Nível de Risco + Top 10 via filtro nativo |

---

## 9. Tecnologias e Ferramentas

| Tecnologia | Aplicação |
|---|---|
| SQL Server Express | Banco de dados relacional — armazenamento do DW |
| SQL Server Management Studio (SSMS) | Desenvolvimento das queries e modelagem |
| SQL (T-SQL) | DDL, DML, Window Functions, CTEs, CASE WHEN |
| Power BI Desktop | Desenvolvimento do dashboard |
| Power BI Service | Publicação e compartilhamento |
| DAX | Medidas, colunas calculadas, inteligência de tempo |
| Power Query (M) | Transformações e tipagem de dados |
| Star Schema | Padrão de modelagem dimensional |
| Kaggle | Fonte do dataset público |

---

## 10. Boas Práticas Aplicadas

**Modelagem de dados:**
- Star Schema com chaves surrogate (IDENTITY) em todas as dimensões
- Staging layer separada do DW final
- Validação de integridade referencial (COUNT(*) = 1.470, zero NULLs nas FKs)

**DAX:**
- Uso de VARs para legibilidade e manutenção
- ALL() para quebra intencional de contexto de filtro
- DIVIDE() com fallback zero para evitar erros de divisão
- Medidas base → medidas compostas (hierarquia de dependência)

**Visualização:**
- Preattentive attributes: cor dinâmica para destacar o valor mais relevante
- Semáforo de risco: vermelho/laranja/amarelo/verde com encoding semântico
- Insights dinâmicos em DAX no lugar de textos estáticos
- Estrutura Problema → Evidência → Recomendação na página executiva
- Princípios do *Storytelling with Data* (Cole Nussbaumer Knaflic)

---

## 11. Competências Demonstradas

**SQL & Modelagem de Dados**
- Modelagem dimensional (Star Schema) do zero
- Criação de staging layer e processo ETL manual
- DDL e DML avançados: ALTER TABLE, UPDATE com JOIN, SELECT INTO
- Window Functions: RANKX, SUM OVER, NTILE
- CTEs para queries analíticas complexas
- Integridade referencial e validação de dados

**Power BI & DAX**
- Conexão com SQL Server e configuração de relacionamentos
- Colunas calculadas vs medidas — aplicação correta de cada uma
- Context transition e uso de ALL() / CALCULATE()
- Formatação condicional via medidas DAX (cor dinâmica)
- Parâmetros de campo para alternância de visão
- Tooltip pages para informações contextuais
- Publicação no Power BI Service

**Análise e Storytelling**
- Transformação de dado em insight acionável
- Construção de score preditivo com múltiplos fatores
- Quantificação de impacto financeiro com benchmark de mercado
- Comunicação executiva: estrutura Problema→Evidência→Recomendação
- Aplicação de princípios de Data Visualization (Storytelling with Data)

**Domínio de negócio**
- Métricas de People Analytics: Attrition Rate, Turnover, Custo de Substituição
- Análise de risco de retenção com drivers comportamentais
- Segmentação geracional e suas implicações para RH
- Benchmark de mercado para contextualização de KPIs

---

## 12. Texto para LinkedIn / Portfólio

**Título do projeto:**
`People Analytics: Análise de Turnover & Risco de Retenção`

**Descrição resumida (para card do portfólio):**
> Dashboard end-to-end de People Analytics desenvolvido do zero: extração de dados, modelagem Star Schema em SQL Server, e dashboard em Power BI com análise descritiva, diagnóstica e preditiva de turnover. Identifica os 60 colaboradores ativos com maior risco de saída e quantifica o custo financeiro de $20,42 Milhões gerado pelo turnover no período.

**Descrição detalhada:**
> Este projeto responde uma das perguntas mais críticas para qualquer organização: por que os talentos estão saindo — e quem vai sair a seguir?
>
> Partindo de um dataset público com 1.470 colaboradores, estruturei um Data Warehouse completo em SQL Server com modelagem Star Schema (1 tabela fato + 5 dimensões), desenvolvi queries analíticas com Window Functions e CTEs, e construí um dashboard de 5 páginas no Power BI com análises descritiva, diagnóstica e preditiva.
>
> O diferencial técnico está no modelo de score preditivo — um índice composto por 4 fatores de risco que classifica os colaboradores ativos em Crítico, Alto, Médio e Baixo, identificando os 60 casos que demandam ação imediata. A página de Conclusões traduz os dados em 3 recomendações executivas com evidência quantitativa e estimativa de impacto financeiro.

**Competências para destacar no LinkedIn:**
`Power BI` · `DAX` · `SQL Server` · `T-SQL` · `Star Schema` · `Data Modeling` · `People Analytics` · `ETL` · `Data Visualization` · `Storytelling with Data` · `Business Intelligence` · `HR Analytics`

---

*Documentação gerada com base no histórico completo de desenvolvimento do projeto.*
*Ruan Estevão Barbosa Lopes — ruanestevao51@gmail.com — linkedin.com/in/ruan-lopes-8556573b0*

