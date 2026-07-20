# Documentação Técnica — Pipeline Vulnerabilidade Social

## 1. Visão Geral

Este notebook é um pipeline de **ETL (Extração, Transformação e Carga)** executado no **Microsoft Fabric** (PySpark + Synapse), com objetivo de consolidar indicadores de vulnerabilidade social do estado de **Pernambuco (PE)** em um Lakehouse.

O pipeline baixa dados de fontes públicas (Ministério do Desenvolvimento e Assistência Social, Ministério dos Direitos Humanos, etc.), padroniza e limpa cada base, e grava o resultado como **tabelas Delta** dentro do Lakehouse `LAKEHOUSE_PEDRO`, prontas para consumo em relatórios/painéis (Power BI).

- **Desenvolvedor:** Pedro Advincula
- **Data:** 07/2026
- **Ambiente:** Fabric Notebook (kernel `synapse_pyspark`)

---

## 2. Estrutura Geral do Notebook

O notebook segue um padrão repetitivo por indicador (uma "mini-ETL" por seção):

1. Configuração do workspace/Lakehouse
2. Definição das URLs de origem
3. Imports
4. Download em lote de todos os arquivos
5. Uma sequência de blocos **ETL X**, um para cada indicador, cada um seguindo o padrão:
   - Ler CSV do Lakehouse (Files)
   - Renomear/selecionar/tratar colunas
   - Gravar como tabela Delta (Tables)

Cada indicador é independente dos demais (não há joins entre eles neste notebook — a consolidação final provavelmente acontece no modelo do Power BI ou em outro processo).

---

## 3. Configuração Inicial (Setup)

### 3.1 Workspace e Caminhos (Célula "GET WORKSPACE")

Descobre o workspace atual do Fabric e monta os caminhos-base:

- `lakehouse_name = 'LAKEHOUSE_PEDRO'`
- `tables_path` → onde as tabelas Delta finais são salvas (`.../Lakehouse/Tables`)
- `files_path` → onde os CSVs brutos ficam armazenados (`.../Lakehouse/Files/VULNERABILIDADE_SOCIAL`)

Todo o restante do notebook usa essas duas variáveis como referência.

### 3.2 URLs dos Projetos

Bloco com as URLs de origem de cada base (a maioria vindas do portal `aplicacoes.cidadania.gov.br` — Visualização de Dados Sociais do governo federal), cada uma com uma query codificada específica:

| Variável | Conteúdo |
|---|---|
| `URL_BOLSA` | Bolsa Família |
| `URL_CADUNICO` | Cadastro Único |
| `URL_PCD_BPC` | Pessoas com deficiência — BPC |
| `URL_TERREIRO` | Famílias de Comunidades de Terreiro |
| `URL_IDOSO_BPC` | Idosos — BPC |
| `URL_INDIGENAS` | Indígenas |
| `URL_QUILOMBOLAS` | Quilombolas |
| `URL_SITUACAO_RUA` | Situação de rua |

> ⚠️ Nota de manutenção: essas URLs são geradas pelo painel do governo e contêm parâmetros de consulta codificados (query string longa). Se o layout do portal mudar, as URLs precisarão ser regeradas manualmente lá e coladas aqui.

### 3.3 Imports

Bibliotecas usadas: `requests` (download HTTP), funções do `pyspark.sql` (`col`, `regexp_replace`, `to_date`, `split`, `F`), `datetime`.

### 3.4 Download dos Arquivos (em lote)

Percorre o dicionário `URLS` e, para cada item:
1. Faz `GET` na URL com streaming (arquivos grandes)
2. Salva temporariamente em `/tmp`
3. Remove a versão antiga no Lakehouse (se existir)
4. Copia o arquivo novo para `files_path` (pasta `Files` do Lakehouse)

Erros de download são capturados e apenas logados (não interrompem o processo dos demais arquivos).

---

## 4. Blocos de ETL por Indicador

Cada bloco abaixo é independente e gera **uma tabela Delta** na pasta `Tables/vul/` (exceto o Censo, que vai para `Tables/dbo/`).

### 4.1 Censo 2025 → `tbCenso2025` (schema `dbo`)
- Lê `Censo2025.csv` (separador `;`)
- Via SQL (`%%sql` e `spark.sql`), filtra apenas `UF = 'PE'` e seleciona/renomeia: UF, código do município, nome do município e população estimada.

### 4.2 CadÚnico → `tbCadUnico`
- Lê `cadunico.csv` (`,`, encoding ISO-8859-1)
- Renomeia colunas para nomes padronizados (Codigo, Unidade_Territorial, Data, Inscritos_cadunico)
- Converte quantidade de inscritos para inteiro

### 4.3 Bolsa Família → `tbBolsaFamilia`
- Lê `bolsa.csv`, mesmo padrão de renomeação
- Seleciona apenas as colunas de interesse
- Converte "Pessoas_beneficiarias_PBF" para inteiro

### 4.4 Nascidos Vivos → `tbNascimentos_Vivos`
- Lê `Nascimentos_Vivos.csv`
- Quebra a coluna `Município` (formato "código nome") em `cod_municipio` e `Municipio`
- Renomeia coluna de nascimentos e converte para inteiro

### 4.5 Óbitos < 5 anos → `tbObitos5anos`
- Mesmo padrão de quebra de `Município` em código + nome
- Renomeia coluna de óbitos e converte para inteiro

### 4.6 Quilombolas → `tbQuilombolas`
- Lê `quilombolas.csv`
- Renomeia e seleciona colunas (família quilombola inscrita no CadÚnico com renda até meio salário-mínimo)
- Converte quantidade para inteiro

### 4.7 Comunidades de Terreiro (CadÚnico) → `tbFamiliaTerreiro`
- Lê `terreiro.csv`
- Mesmo padrão: renomeia e converte quantidade de famílias para inteiro

### 4.8 BPC — Pessoas com Deficiência → `tbDeficiente_BPC`
- Lê `pcd_bpc.csv`
- Apenas normaliza nomes de colunas (espaço → underscore), sem seleção/renomeação específica

### 4.9 Situação de Rua → `tbSituacao_Rua`
- Lê `situacao_rua.csv`
- Normaliza nomes de colunas e depois renomeia campos-chave (código, unidade territorial, UF, data, quantidade)

### 4.10 Idosos PE → `tbIdososPE`
- Lê `IdososPE.csv` (encoding utf-8, diferente das demais que usam ISO-8859-1)
- Apenas normaliza nomes de colunas

### 4.11 Idosos — BPC → `tbIdoso_BPC`
- Lê `idoso_bpc.csv`
- Apenas normaliza nomes de colunas

### 4.12 Disque 100 → `tbDisque100_Terreiro`
Esse bloco é diferente dos demais — tem uma lógica de download própria e mais complexa:

1. **Download dinâmico:** monta o nome do arquivo com base na data atual (`disque100-primeiro-semestre-2026` ou `disque100-segundo-semestre-2026`), pois o governo publica esse arquivo por semestre. Se o arquivo do semestre vigente ainda não estiver disponível (HTTP 403/404), o notebook segue em frente sem quebrar.
2. **Leitura em lote:** lê **todos** os arquivos que casam com o padrão `disque100*.csv` já salvos no Lakehouse (histórico acumulado de semestres anteriores), não apenas o mais recente.
3. **Filtro:** mantém apenas registros de `UF = 'PE'` cuja `Etnia_da_vítima` contenha o termo "terreiro".
4. **Deduplicação:** remove duplicados por `hash` (identificador único do registro na base do Disque 100).
5. **Tratamento de município:** a coluna `Município` vem no formato `"código|nome"`; o notebook faz `split` por `|` e separa em `cod_municipio` e `municipio`.
6. Grava o resultado final em `tbDisque100_Terreiro`.

---

## 5. Padrão de Nomenclatura e Convenções

- **Pasta de arquivos brutos:** `Files/VULNERABILIDADE_SOCIAL` (definida uma única vez em `files_path`)
- **Pasta de tabelas:** `Tables/vul/` para a maioria dos indicadores; `Tables/dbo/` só para o Censo
- **Modo de escrita:** todas as tabelas usam `.mode("overwrite")` — ou seja, a cada execução o histórico da tabela é **substituído integralmente**, não é incremental
- **Encoding:** a maioria dos CSVs de origem usa `ISO-8859-1` (padrão do portal do governo); exceções conhecidas: Censo (default), Disque 100 e IdososPE (`utf-8`)
- **Separador:** varia entre `;` e `,` dependendo do arquivo — sempre checar antes de alterar algo

---

## 6. Pontos de Atenção para Manutenção

1. **URLs do portal `cidadania.gov.br`** são geradas com parâmetros de consulta específicos do painel; se pararem de funcionar (403/404), é necessário acessar o painel novamente e gerar uma nova URL de exportação.
2. **Disque 100** depende de nomenclatura semestral do arquivo — se o governo mudar o padrão de nome do arquivo, a lógica de montagem do nome (`nome_arquivo`) precisa ser ajustada.
3. **Encoding inconsistente** entre as fontes é uma fonte comum de erro (caracteres acentuados corrompidos) — sempre validar ao adicionar uma nova fonte.
4. **`overwrite`** em todas as tabelas significa que não há retenção de histórico entre execuções — se for necessário auditar mudanças ao longo do tempo, seria preciso mudar para `append` com uma coluna de data de carga.
5. Os blocos de ETL **não têm tratamento de erro** (diferente do bloco de download) — se um CSV estiver ausente ou com schema diferente do esperado, o notebook quebra naquele ponto.

---

## 7. Tabelas Finais Geradas

| Tabela | Schema | Fonte |
|---|---|---|
| tbCenso2025 | dbo | Censo 2025 |
| tbCadUnico | vul | CadÚnico |
| tbBolsaFamilia | vul | Bolsa Família |
| tbNascimentos_Vivos | vul | Nascidos Vivos |
| tbObitos5anos | vul | Óbitos < 5 anos |
| tbQuilombolas | vul | CadÚnico — Quilombolas |
| tbFamiliaTerreiro | vul | CadÚnico — Comunidades de Terreiro |
| tbDeficiente_BPC | vul | BPC — PCD |
| tbSituacao_Rua | vul | CadÚnico — Situação de Rua |
| tbIdososPE | vul | Idosos PE |
| tbIdoso_BPC | vul | BPC — Idosos |
| tbDisque100_Terreiro | vul | Disque 100 — Terreiro (PE) |