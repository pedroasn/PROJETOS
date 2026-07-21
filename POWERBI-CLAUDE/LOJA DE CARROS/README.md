# Loja de Carros

Projeto Power BI para análise de vendas de veículos, composto por um relatório (`LojaCarros.Report`) e um modelo semântico (`LojaCarros.SemanticModel`).

## Visão Geral

- `LojaCarros.Report/`
  - `definition.pbir`: definição do relatório Power BI em formato `.pbir`
  - `report.json`: configurações e seções do relatório
  - `StaticResources/SharedResources/BaseThemes/Fluent2-CY26SU05.json`: tema customizado usado no relatório

- `LojaCarros.SemanticModel/`
  - `definition.pbism`: definição do modelo semântico Power BI
  - `diagramLayout.json`: layout do diagrama do modelo
  - `definition/database.tmdl`: metadados de compatibilidade e configuração do modelo
  - `definition/model.tmdl`: definição do modelo e referências às tabelas
  - `definition/tables/`: definições das tabelas de dados e suas consultas de origem

## Estrutura do Modelo

O modelo usa um esquema estrela simples com as seguintes tabelas:

- `dim_cliente`
  - id_cliente, nome, estado, idade, sexo
  - origem: `D:\Cursos\Python\Loja Carros\bases\dim_cliente.csv`

- `dim_tempo`
  - id_tempo, data, ano, mes, dia, feriado
  - origem: `D:\Cursos\Python\Loja Carros\bases\dim_tempo.csv`

- `dim_veiculo`
  - id_veiculo, marca, modelo, ano, categoria
  - origem: `D:\Cursos\Python\Loja Carros\bases\dim_veiculo.csv`

- `fato_vendas`
  - id_venda, id_cliente, id_veiculo, id_tempo, valor_venda, quantidade, desconto
  - origem: `D:\Cursos\Python\Loja Carros\bases\fato_vendas.csv`

## Observações Importantes

- Todas as tabelas são carregadas por consulta M em modo de importação.
- As consultas usam caminhos absolutos para arquivos CSV na máquina local. Se for abrir o relatório em outro computador, atualize os caminhos para os arquivos de origem ou use outra fonte de dados compatível.
- O projeto está em cultura `pt-BR` e o modelo usa compatibilidade de banco de dados `1600`.

## Como Abrir

1. Abra o Power BI Desktop.
2. Abra o arquivo `LojaCarros.Report/definition.pbir` ou carregue o modelo semântico em `LojaCarros.SemanticModel/definition.pbism`.
3. Verifique e ajuste os caminhos de origem dos dados no editor de consultas, se necessário.

## Pontos de Extensão

- Adicionar medidas DAX para análise de receita, desconto médio e ticket médio.
- Criar relacionamentos entre `fato_vendas` e as dimensões, se ainda não definidos no modelo.
- Construir páginas de relatório com filtros de data, marca de veículo e segmento de cliente.

## Contato

- Projeto criado para análise de vendas na loja de carros.
- Ajuste o modelo conforme a base de dados local ou ambiente de implantação.
