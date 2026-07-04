# Microsoft Azure Event Hub
## Envio de Logs para o SIEM via Azure Event Hub

**Versão:** 1.1
**Última atualização:** Julho de 2026

---

## Sumário

- [1. Introdução](#1-introdução)
- [2. Visão Geral da Arquitetura](#2-visão-geral-da-arquitetura)
- [3. Pré-requisitos](#3-pré-requisitos)
- [4. Parte 1 — Configuração no Azure](#4-parte-1--configuração-no-azure)
- [5. Parte 2 — Configuração no SIEM (Splunk)](#5-parte-2--configuração-no-siem-splunk)
- [6. Validação](#6-validação)
- [7. Requisitos de Rede e Firewall](#7-requisitos-de-rede-e-firewall)
- [8. Troubleshooting](#8-troubleshooting)
- [9. Referências](#9-referências)

---

## 1. Introdução

O Azure Event Hubs oferece uma plataforma escalável e resiliente para transmissão de dados em tempo real, sendo adequada para diversos casos de uso, como a coleta de telemetria e de logs sensíveis. A implementação de uma autenticação robusta para a ingestão desses dados não é apenas uma boa prática — é uma necessidade, especialmente ao lidar com informações que exigem gerenciamento centralizado e acesso seguro.

Este documento descreve o processo de configuração ponta a ponta para enviar logs do Azure a um SIEM (com foco em Splunk) via Event Hub, cobrindo duas fontes de log de auditoria:

- **Azure Activity Log** — operações de plano de controle na assinatura (criação, alteração e remoção de recursos).
- **Microsoft Entra ID Logs** — eventos de autenticação e identidade (sign-in, auditoria, risco de usuário).

### 1.1 Escopo

- Transporte: Azure Event Hub.
- Destino: SIEM (exemplos de configuração detalhados para Splunk via Splunk Add-on for Microsoft Cloud Services).
- Autenticação: duas opções documentadas — Service Principal (recomendado) ou Shared Access Policy (SAS Key).
- Este documento não cobre Diagnostic Logs de recursos individuais (ex.: NSG, App Service).

---

## 2. Visão Geral da Arquitetura

O fluxo de dados segue as seguintes etapas:

- O Azure Monitor (ou o Microsoft Entra ID, no caso dos logs de identidade) captura os eventos na origem.
- Uma Diagnostic Setting encaminha esses eventos para um Event Hub dentro de um Namespace.
- Uma credencial com permissão de leitura no Event Hub — Service Principal ou SAS Key — é usada pelo SIEM para autenticação.
- O SIEM, através do conector apropriado, consome os eventos do Event Hub via protocolo AMQP e os indexa.

**Fluxo resumido:**

```
Azure Activity Log / Entra ID Logs  →  Diagnostic Setting  →  Event Hub (Namespace)  →  Conector do SIEM  →  Índice no SIEM
```

---

## 3. Pré-requisitos

- Assinatura do Azure ativa, com permissão de **Security Administrator** (ou superior) para criar Diagnostic Settings.
- Permissão para criar recursos: Resource Group, Event Hubs Namespace, Event Hub e, se aplicável, App Registration no Microsoft Entra ID.
- Acesso administrativo à instância do Splunk (Splunk Enterprise ou Splunk Cloud com Inputs Data Manager habilitado), caso o SIEM utilizado seja o Splunk.
- Splunk Add-on for Microsoft Cloud Services instalado na instância responsável pela coleta (normalmente um heavy forwarder).
- Conectividade de rede liberada entre o coletor do SIEM e o Azure Event Hub (ver detalhes na [Seção 7](#7-requisitos-de-rede-e-firewall)).

> **Observação:** Se a instância do Splunk estiver na Splunk Cloud usando o Inputs Data Manager (IDM), a liberação de portas deve ser feita via Admin Configuration Service (ACS).

---

## 4. Parte 1 — Configuração no Azure

### 4.1 Criar o Resource Group

**Passo 1. Acessar o Azure Portal**
Faça login em [portal.azure.com](https://portal.azure.com). No menu lateral esquerdo, clique em "Resource Groups".

**Passo 2. Criar um novo Resource Group**
Clique em "Create" e preencha:

| Campo | Descrição |
|---|---|
| Subscription | Assinatura do Azure onde os recursos serão criados |
| Resource Group Name | Nome exclusivo para o Resource Group (o Azure valida a disponibilidade automaticamente) |
| Region | Recomenda-se usar a mesma região dos demais recursos, para reduzir latência e simplificar o gerenciamento |

Clique em "Review + Create" e, após validar, em "Create".

### 4.2 Criar o Event Hubs Namespace

**Passo 3. Criar o Namespace**
Dentro do Resource Group criado, clique em "Criar" e busque "Event Hubs". Selecione a assinatura e o grupo de recursos, defina um nome globalmente único (ex.: `log-integration-ns`), a região e o nível de pricing (Standard ou superior, para suportar maior throughput). Confirme a criação.

> **Observação:** Um Namespace é apenas um contêiner lógico; nenhum Event Hub é criado automaticamente nesta etapa.

### 4.3 Criar o Event Hub

**Passo 4. Criar o Event Hub dentro do Namespace**
Acesse "Event Hubs" no menu lateral do Namespace e clique em "+ Event Hub". Configure:

| Campo | Recomendação |
|---|---|
| Nome | Ex.: `insights-activity-logs` (ou um Event Hub dedicado por fonte de log — ver observação abaixo) |
| Partition Count | O padrão é 1, equivalente a ~1 MB/s de capacidade de ingestão. Ajuste conforme o volume esperado |
| Cleanup Policy | Delete |
| Retention Time | 24 horas (os dados só precisam durar até o SIEM realizar o poll) |

> **Observação:** Como este documento cobre duas fontes de log (Activity Log e Entra ID Logs), recomenda-se criar **dois Event Hubs separados** dentro do mesmo Namespace (ex.: `activity-log-eh` e `entraid-log-eh`), facilitando a segregação de índices/sourcetypes no SIEM.

### 4.4 Configurar autenticação de acesso ao Event Hub

Existem duas formas de conceder acesso de leitura ao Event Hub. Escolha a opção compatível com o conector do seu SIEM — as duas podem coexistir no mesmo ambiente, para ferramentas diferentes.

#### Opção A — Service Principal (App Registration no Microsoft Entra ID)

Método usado pelo **Splunk Add-on for Microsoft Cloud Services** (documentado na Parte 2). Recomendado por se basear em RBAC do Azure AD, sem chaves compartilhadas de longa duração.

**Passo 5. Registrar uma aplicação no Microsoft Entra ID**
Pesquise por "Microsoft Entra ID" > "App registrations" > "+ New registration". Dê um nome (ex.: `siem-log-collector`), selecione "Single tenant" e não é necessário preencher o Redirect URI. Clique em "Register".

**Passo 6. Coletar as credenciais**
Na página de visão geral, copie o **Application (client) ID** e o **Directory (tenant) ID**. Em "Certificates & secrets", crie um novo Client Secret e copie o valor imediatamente (ele não será exibido novamente).

**Passo 7. Atribuir a role Azure Event Hubs Data Receiver**
Acesse "Subscriptions" > assinatura correspondente > "Access control (IAM)" > "Add" > "Add role assignment". Pesquise pela role "Azure Event Hubs Data Receiver".

**Passo 8. Selecionar o Service Principal**
Em "Members", clique em "Select members", pesquise pelo App Registration criado no Passo 5, selecione-o e conclua com "Review + assign".

> **Observação:** Atribuir a role no nível da assinatura garante acesso a qualquer Namespace/Event Hub criado futuramente nela. Para um escopo mais restrito, atribua a role diretamente no Namespace do Event Hub.

#### Opção B — Shared Access Policy (SAS Key)

Método mais simples, baseado em chave compartilhada. Usado por conectores/SIEMs que autenticam via connection string.

**Passo 9. Criar a Shared Access Policy**
Dentro do Namespace, acesse "Shared Access Policies" > "+ Add". Nomeie a política (ex.: `SIEMPolicy`) e marque apenas a permissão **Listen** (não é necessário Send nem Manage). Após criar, copie a **Connection String – Primary Key**; ela será usada pelo conector do SIEM.

> **Observação de segurança:** a SAS Key não expira automaticamente e concede acesso a quem a possuir. Trate-a como um segredo e faça rotação periódica.

### 4.5 Configurar as Diagnostic Settings

#### 4.5.1 Azure Activity Log

**Passo 10. Acessar as configurações do Activity Log**
No Azure Portal, pesquise por "Monitor" > "Activity Log" > "Export Activity Logs" (ou "Diagnostic settings").

**Passo 11. Criar a Diagnostic Setting**
Selecione a assinatura desejada, clique em "+ Add diagnostic setting", nomeie a configuração (ex.: `activity-log-to-eventhub`) e marque as categorias: **Administrative, Security, Alert, Policy** (no mínimo).

**Passo 12. Direcionar para o Event Hub**
Marque "Stream to an event hub", selecione o Namespace e o Event Hub dedicado ao Activity Log (ex.: `activity-log-eh`) e a política de acesso, se aplicável. Salve a configuração.

> **Observação:** O Activity Log é único por assinatura — não é necessário repetir por recurso, apenas por assinatura monitorada.

#### 4.5.2 Microsoft Entra ID Logs

**Passo 13. Acessar as configurações de diagnóstico do Entra ID**
Pesquise por "Microsoft Entra ID" > "Monitoring" > "Diagnostic settings" > "Add diagnostic setting".

**Passo 14. Nomear e selecionar as categorias**
Nomeie a configuração (ex.: `entraid-logs-to-eventhub`) e selecione as categorias desejadas:

- `AuditLogs`
- `SignInLogs`
- `NonInteractiveUserSignInLogs`
- `ServicePrincipalSignInLogs`
- `ManagedIdentitySignInLogs`
- `ProvisioningLogs`
- `RiskyUsers`
- `UserRiskEvents`

> **Observação:** `NonInteractiveUserSignInLogs` e `ServicePrincipalSignInLogs` costumam gerar volume significativamente maior que as demais categorias. Avalie o custo de ingestão antes de habilitá-las em produção.

**Passo 15. Direcionar para o Event Hub**
Marque "Stream to an event hub" e selecione o Namespace e o Event Hub dedicado aos logs do Entra ID (ex.: `entraid-log-eh`). Salve a configuração.

---

## 5. Parte 2 — Configuração no SIEM (Splunk)

### 5.1 Instalar o Splunk Add-on for Microsoft Cloud Services

**Passo 16. Instalar o add-on**
No Splunk Web, acesse "Apps" > "Find More Apps", pesquise por "Splunk Add-on for Microsoft Cloud Services" e instale-o na instância responsável pela coleta (heavy forwarder).

> **Observação:** Esse add-on é distinto do "Splunk Add-on for Microsoft Azure". Se o ambiente já usa o add-on legado para o mesmo Event Hub, desabilite-o para evitar conflito de consumidores. Este add-on autentica via **Service Principal (Opção A)** — se o seu conector exigir SAS Key (Opção B), consulte a tabela de parâmetros genéricos abaixo.

### 5.2 Configurar a conta de aplicação do Azure no add-on

**Passo 17. Adicionar a conta do Azure App**
Em "Configuration" > "Account" > "Add", informe o Application (client) ID e o Client Secret coletados no Passo 6, além do Tenant ID.

### 5.3 Criar o input do Azure Event Hub

**Passo 18. Criar um novo input**
Acesse "Inputs" > "Create New Input" > "Azure Event Hub".

**Passo 19. Preencher os parâmetros de conexão**
Informe o nome do input, a conta configurada no Passo 17, o Event Hubs Namespace (FQDN, ex.: `log-integration-ns.servicebus.windows.net`), o Event Hub (ex.: `activity-log-eh` ou `entraid-log-eh`) e um Consumer Group dedicado (diferente do `$Default`).

**Passo 20. Definir índice e sourcetype**
Crie um input por Event Hub, com índices separados (ex.: `azure-activity` e `azure-entraid`), mantendo o sourcetype padrão `mscs:azure:eventhub`.

**Passo 21. Salvar e habilitar o input**
Confirme a criação. O Splunk iniciará a leitura via protocolo AMQP.

> **Observação:** Caso o Splunk esteja atrás de proxy ou seja Splunk Cloud sem suporte a add-ons customizados, encaminhe os eventos via Azure Function acionada pelo Event Hub, enviando os dados ao Splunk HTTP Event Collector (HEC).

#### Parâmetros genéricos para conectores baseados em SAS Key (Opção B)

Para SIEMs/conectores que autenticam via Shared Access Policy em vez de Service Principal:

| Parâmetro | Descrição |
|---|---|
| Event Hub Namespace Name | Nome curto do Namespace (ex.: `log-integration-ns`), não o FQDN |
| Event Hub Name | Nome do Event Hub criado (ex.: `activity-log-eh`) |
| SAS Policy Name | Nome da Shared Access Policy criada no Passo 9 (ex.: `SIEMPolicy`) |
| Primary Key | Chave primária gerada para a Shared Access Policy |
| Consumer Group Name | Consumer Group dedicado ao conector; use `$Default` apenas se nenhum outro tiver sido criado |
| Account Environment | Normalmente `AzureCloud`; ambientes governamentais usam endpoints distintos |

---

## 6. Validação

Após a configuração, valide o recebimento dos dados no Splunk:

**Passo 22. Verificar o Data Summary**
Em "Settings" > "Data Summary" > aba "Sourcetypes", confirme a presença do sourcetype `mscs:azure:eventhub` com eventos recentes em ambos os índices (`azure-activity` e `azure-entraid`).

**Passo 23. Executar buscas de validação**

Para o Activity Log:

```spl
index=azure-activity sourcetype="mscs:azure:eventhub" category=Administrative
```

Para os logs do Entra ID:

```spl
index=azure-entraid sourcetype="mscs:azure:eventhub"
```

---

## 7. Requisitos de Rede e Firewall

### 7.1 Acesso ao Azure Event Hub

O coletor do SIEM precisa de conexão de saída (outbound) para:

| Destino | Porta | Protocolo |
|---|---|---|
| `*.servicebus.windows.net` | TCP 5671 | AMQP sobre TLS |
| `*.servicebus.windows.net` | TCP 443 | AMQP via WebSockets (alternativa) |

### 7.2 Proxy corporativo

Se houver proxy entre o coletor e a internet, verifique se ele permite tráfego para `*.servicebus.windows.net`. Proxies com **SSL Inspection** podem interromper a negociação TLS/AMQP — recomenda-se criar uma exceção (bypass) para esse domínio.

### 7.3 Regras de rede do próprio Event Hub

No Azure Portal, em Event Hub Namespace > "Networking", verifique se o acesso está configurado como:

- Public access (all networks)
- Selected networks — se marcado, adicione o IP público do coletor à lista de permitidos
- Private Endpoint

### 7.4 Network Security Groups (NSG) e Azure Firewall

Se o acesso for via Private Endpoint, confirme que os NSGs permitem tráfego entre o coletor, o Private Endpoint e a Virtual Network. Se houver Azure Firewall, libere as mesmas regras da tabela da Seção 7.1.

### 7.5 DNS e teste de conectividade

Confirme a resolução de nome do Namespace:

```bash
nslookup <namespace>.servicebus.windows.net
```

Teste de conectividade (Linux):

```bash
nc -vz <namespace>.servicebus.windows.net 5671
nc -vz <namespace>.servicebus.windows.net 443
```

Teste de conectividade (Windows PowerShell):

```powershell
Test-NetConnection <namespace>.servicebus.windows.net -Port 5671
Test-NetConnection <namespace>.servicebus.windows.net -Port 443
```

### 7.6 Checklist de liberação

| Item | Status |
|---|---|
| Liberação de saída TCP 5671 | ☐ |
| Liberação de saída TCP 443 | ☐ |
| Resolução DNS para `*.servicebus.windows.net` | ☐ |
| Proxy permite acesso ao Event Hub | ☐ |
| Exceção criada para SSL Inspection (se aplicável) | ☐ |
| Event Hub configurado para permitir acesso do coletor | ☐ |
| IP público do coletor liberado (se "Selected networks") | ☐ |
| Teste de conectividade realizado com sucesso | ☐ |

---

## 8. Troubleshooting

| Sintoma | Causa provável | Ação recomendada |
|---|---|---|
| Nenhum evento chega ao SIEM | Portas AMQP (5671/443) bloqueadas no firewall | Revisar checklist da Seção 7.6 |
| Erro de autenticação no input (Opção A) | Role "Azure Event Hubs Data Receiver" não atribuída ou Client Secret expirado | Revisar o IAM da assinatura/Namespace e gerar novo Client Secret |
| Erro de autenticação no conector (Opção B) | SAS Key incorreta, expirada ou rotacionada | Gerar nova chave na Shared Access Policy e atualizar o conector |
| Eventos duplicados ou ausentes | Mais de um consumidor no mesmo Consumer Group | Criar um Consumer Group dedicado por ferramenta |
| Input não inicia (erro de checkpoint) | Checkpoint local corrompido ou permissão de diretório | Verificar permissões da pasta do add-on ou habilitar Blob Checkpoint Store |
| Alto volume de dados / custo elevado | Categorias de log muito abrangentes (ex.: `NonInteractiveUserSignInLogs`) | Revisar e restringir as categorias exportadas ao necessário |

---

## 9. Referências

- [Microsoft Learn — Stream Microsoft Entra logs to an event hub](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-stream-logs-to-event-hub)
- [Splunk Lantern — Getting started with Microsoft Azure Event Hub data](https://lantern.splunk.com/Platform_Data_Management/Unlock_Insights/Getting_started_with_Microsoft_Azure_Event_Hub_data)
- [Splunk Docs — Configure Azure Event Hub input (Splunk Add-on for Microsoft Cloud Services)](https://splunk.github.io/splunk-add-on-for-microsoft-cloud-services/Configureeventhubs/)
- Documentação oficial do Azure Monitor — Diagnostic settings
