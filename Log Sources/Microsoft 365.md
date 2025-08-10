# Detecção e Monitoramento de Ameaças com Microsoft 365

Este repositório apresenta os principais requisitos para monitoramento do Microsoft 365 via Microsoft Graph API e Office 365 Management Activity API, visando melhorar a visibilidade e a resposta a incidentes em ambientes corporativos.

---

## 📌 Referências
- [Threat Detection and Monitoring with Microsoft 365 – Logpoint](https://www.logpoint.com/en/blog/threat-detection-and-monitoring-with-microsoft-365/)
- [Microsoft Office 365 Audit – Fortinet FortiSIEM](https://docs.fortinet.com/document/fortisiem/7.4.0/external-systems-configuration-guide/514932/microsoft-office365-audit)
- [Audit log activities https://learn.microsoft.com/en-us/purview/audit-log-activities)]

---

## 🔍 Visão Geral dos Componentes

### **Microsoft Entra ID** (Azure AD)
- Gerencia identidades e acessos com suporte a autenticação multifator e passwordless.

### **Microsoft Defender XDR**
- Oferece visibilidade, investigação e correlação de alertas entre produtos Defender.

---

## 🔗 Integrações via Microsoft Graph API (URAF)

| Endpoint                           | Fonte Microsoft                | Permissões Necessárias                                                                 | Licença Requerida                                    |
|------------------------------------|---------------------------------|----------------------------------------------------------------------------------------|------------------------------------------------------|
| `directoryAudits` / `auditSignIns` | Entra ID                        | `AuditLog.Read.All`, `Directory.Read.All`                                              | Entra ID P1 (alguns dados); P2 para mais detalhes    |
| `riskDetections`, `riskyUsers`     | Entra ID Identity Protection    | `IdentityRiskEvent.Read.All`, `IdentityRiskyUser.Read.All`                             | Entra ID P2                                          |
| `alerts_v2`, `incidents`           | Defender XDR                    | `SecurityAlert.Read.All`, `SecurityIncident.Read.All`, `SecurityEvents.Read.All`       | Microsoft 365 Defender (sem custo extra)            |

---

### 🔑 Permissões Adicionais
Algumas permissões adicionais podem ser necessárias, dependendo do nível de detalhamento desejado:
- `Reports.Read.All` *(principal)*
- `IdentityRiskyServicePrincipal.Read.All`
- `ThreatIndicators.Read.All`
- `ThreatIntelligence.Read.All`

---

## 🛠️ Configuração de Permissões na API

Para habilitar permissões para coleta de eventos de atividade do Office 365:

1. Acesse **API permissions** no portal do Azure e clique em **Add a permission**.
2. Escolha **Office 365 Management APIs** → **Application permissions**.
3. Adicione as seguintes permissões no grupo **ActivityFeed**:
   - `ActivityFeed.Read` – Ler dados de atividade da organização.
   - `ActivityFeed.ReadDlp` – Ler eventos de políticas DLP, incluindo dados sensíveis detectados.
4. Adicione também:
   - `User.Read`
5. Clique em **Add permissions**.

---

## 📊 O que é Monitorado

A API do Office 365 Management Activity agrupa eventos em blobs de conteúdo por tipo:

- **Audit.AzureActiveDirectory**
- **Audit.Exchange**
- **Audit.SharePoint**
- **Audit.General** (outros workloads)
- **DLP.All** (eventos DLP)

Atividades monitoradas:
- Ações em arquivos e pastas
- Solicitações de compartilhamento e acesso
- Administração de sites
- Atividades em caixas de correio Exchange
- Administração de usuários, grupos, aplicativos e funções

---

## ⚙️ Configuração de Auditoria do Office 365

A configuração da auditoria no Office 365 é composta por **três etapas principais**:

### **Etapa 1 – Configurar caixas de correio do Office 365 para auditoria**
1. **Acesse o Exchange Online**
   > Observação: as instruções podem variar para ambientes **GCC High** e governamentais.
   
2. **Conecte-se via PowerShell**:
   ```powershell
   Connect-ExchangeOnline -UserPrincipalName navin@contoso.onmicrosoft.com
2. **Habilite o rastreamento no Exchange Online**:
Para ativar auditoria em massa em todas as caixas de correio de usuários, execute no PowerShell:  
   ```powershell
   Get-Mailbox -ResultSize Unlimited -Filter {RecipientTypeDetails -eq "UserMailbox"} | Set-Mailbox -AuditEnabled $true
