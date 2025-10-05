# Gerenciador-de-Catalogos-da-Netflix-com-Azure-Functions-e-Banco-de-Dados

# Projeto Serverless de Ingestão e Gerenciamento de Dados no Azure

## Visão Geral do Projeto

Este repositório documenta o desenvolvimento de uma solução **Serverless** completa no Microsoft Azure, projetada para ingestão, processamento, armazenamento e consulta de dados. A arquitetura se baseia em **Azure Functions** como motor de execução escalável e sem servidor, utilizando o **Azure Storage Account** e o **Azure Cosmos DB** como camadas de persistência.

O objetivo é demonstrar a construção de microsserviços desacoplados e orientados a eventos (event-driven), garantindo alta disponibilidade, baixo custo operacional e escalabilidade horizontal.

### Arquitetura Principal

1.  **Ingestão de Arquivos:** Azure Function acionada por HTTP/eventos, salvando arquivos no Azure Storage.
2.  **Processamento:** Azure Function acionada por eventos do Storage, salvando metadados no Cosmos DB.
3.  **Consulta:** Azure Functions acionadas por HTTP para filtrar e listar registros no Cosmos DB.

---

## Tecnologias Utilizadas

| Componente | Serviço do Azure | Função |
| :--- | :--- | :--- |
| **Computação** | **Azure Functions** (Plano de Consumo) | Motor Serverless para todo o processamento e endpoints de API. |
| **Armazenamento de Arquivos** | **Azure Storage Account (Blob Storage)** | Armazenamento durável e econômico para arquivos de grande volume. |
| **Banco de Dados NoSQL** | **Azure Cosmos DB** | Banco de dados NoSQL globalmente distribuído, usado para armazenar dados processados e metadados. |
| **Segurança** | **Azure Key Vault** | Gerenciamento de chaves de conexão e segredos. |
| **Orquestração** | **Event Grid** (Implícito) | Roteamento de eventos (ex: quando um arquivo é salvo, um evento é disparado). |

## Próximos Passos

Os próximos módulos detalham a implementação de cada componente:

* **Criando a Infra Estrutura em Cloud:** Definição dos recursos.
* **Azure Function: Salvar Arquivos:** Implementação da ingestão inicial.
* **Azure Function: Salvar em CosmosDB:** Implementação do processamento de dados.
* **Azure Functions: Filtros e Listagem:** Implementação dos endpoints de consulta (API).


# Módulo 1 - Criação da Infraestrutura Serverless

## Objetivo

Este módulo detalha a criação dos recursos essenciais do Azure que servirão de base para a arquitetura Serverless. Todos os recursos devem ser provisionados com as configurações de segurança e escalabilidade adequadas.

## Recursos a Serem Criados

| Recurso | Tipo de Serviço | Propósito |
| :--- | :--- | :--- |
| **Conta de Armazenamento** | `Microsoft.Storage/storageAccounts` | Armazenar arquivos brutos (blobs). |
| **Azure Cosmos DB** | `Microsoft.DocumentDB/databaseAccounts` | Banco de dados para dados estruturados (NoSQL). |
| **Plano de Funções** | `Microsoft.Web/serverfarms` | Hospedagem Serverless (Plano de Consumo) para o código. |
| **Azure Function App** | `Microsoft.Web/sites` | A aplicação que executará todo o código. |
| **Azure Key Vault** | `Microsoft.KeyVault/vaults` | Armazenar strings de conexão do Cosmos DB e outros segredos. |

## 📝 Instruções de Provisionamento (Exemplo via Azure CLI)

Você pode usar o Azure Portal, Terraform, Bicep ou Azure CLI. Aqui está um exemplo simplificado:

1.  **Criar Grupo de Recursos:**
    ```bash
    az group create --name ServerlessRG --location eastus
    ```
2.  **Criar Azure Storage Account (Standard LRS):**
    ```bash
    az storage account create --name [StorageName] --resource-group ServerlessRG --sku Standard_LRS
    ```
3.  **Criar Cosmos DB Account (API Core - NoSQL):**
    ```bash
    az cosmosdb create --name [CosmosDBName] --resource-group ServerlessRG --kind GlobalDocumentDB
    ```
4.  **Criar Function App (Plano de Consumo):**
    ```bash
    az functionapp create --resource-group ServerlessRG --consumption-plan-location eastus --runtime node --name [FunctionName]
    ```

## Configuração de Segurança Pós-Provisionamento

* **Key Vault:** Armazene a **string de conexão do Cosmos DB** como um segredo.
* **Azure Function App:** Habilite a **Identidade Gerenciada** e conceda as permissões necessárias (via RBAC) para acessar o **Key Vault**.


# Módulo 2 - Ingestão de Arquivos (Azure Function)

## Objetivo

Desenvolver uma **Azure Function** acionada por HTTP que recebe um arquivo (upload) e o salva diretamente no **Azure Blob Storage**. Esta função atua como o ponto de entrada da ingestão de dados.

## Especificação da Azure Function

* **Nome:** `UploadFileFunction`
* **Gatilho:** HTTP Trigger (POST)
* **Lógica:** Recebe o corpo da requisição HTTP (o arquivo em si) e salva o conteúdo no Blob Storage.
* **Saída (Binding):** Nenhum. O salvamento é feito via código (SDK).
* **Próximo Passo:** Disparar um evento (via Event Grid) ou confiar no Event Grid do Storage para notificar a próxima função sobre o novo arquivo.

## Configuração de Acesso ao Storage

A função deve usar a **Identidade Gerenciada** configurada no módulo anterior para acessar o Storage Account.

1.  Conceda à Identidade Gerenciada do Function App a função **Storage Blob Data Contributor** no Storage Account.
2.  Use o SDK apropriado (ex: `Azure.Storage.Blobs` no .NET) no código para se autenticar automaticamente e realizar o upload.

## Exemplo de Resposta (Sucesso)

```json
{
  "status": "Success",
  "message": "Arquivo [nome_do_arquivo] salvo com sucesso.",
  "path": "https://[StorageName].blob.core.windows.net/[container]/[nome_do_arquivo]"
}


# Módulo 5 - Consulta de Dados (Listagem)

## Objetivo

Criar um endpoint de API simples (usando Azure Function) para **listar registros** do Azure Cosmos DB, geralmente utilizado para preencher tabelas em interfaces de usuário ou relatórios.

## Especificação da Azure Function

* **Nome:** `ListRecordsFunction`
* **Gatilho:** HTTP Trigger (GET)
* **Entrada:** Parâmetros opcionais para paginação (ex: `?page=1&pageSize=50`).
* **Lógica:** Executa uma consulta simples no Cosmos DB, aplicando paginação se necessário, para retornar um conjunto de documentos.
* **Saída:** Retorna uma lista JSON de documentos.

## Paginação para Desempenho

Em vez de tentar carregar todos os documentos de uma vez (o que pode ser lento e caro):

1.  Use o recurso de **Continuação Token** do Cosmos DB ou
2.  Use `LIMIT` e `OFFSET` na consulta SQL do Cosmos DB para implementar paginação baseada em deslocamento (`skip/take`).

## Exemplo de Consulta (Cosmos DB SQL)

```sql
SELECT c.id, c.nomeArquivo, c.dataProcessamento, c.status
FROM c
ORDER BY c.dataProcessamento DESC
OFFSET 0 LIMIT 50

