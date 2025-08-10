# Detecção e Monitoramento de Ameaças com Microsoft 365

Este repositório apresenta os principais requisitos para monitoramento do Microsoft 365 via Microsoft Graph API para melhorar a visibilidade e a resposta a incidentes em ambientes corporativos.

---

##  Visão Geral

Ref: https://www.logpoint.com/en/blog/threat-detection-and-monitoring-with-microsoft-365/

---

##  Principais Componentes & Integrações

### Microsoft Entra ID (anteriormente Azure AD)
- Gerencia identidades e acessos com suporte a autenticação multifator e sem senha (passwordless)

### Microsoft Defender XDR (anteriormente Microsoft 365 Defender)
- Fornece visibilidade, investigação e correlação de alertas entre diferentes produtos Defender

##  Integrações via Microsoft Graph API com URAF (Universal REST API Fetcher)



| Endpoint                        | Fonte Microsoft       | Permissões Necessárias                         | Licença Requerida                  |
|---------------------------------|------------------------|------------------------------------------------|------------------------------------|
| `directoryAudits` / `auditSignIns` | Entra ID               | AuditLog.Read.All                              | Entra ID P1 (alguns dados); P2 para mais detalhe  |
| `riskDetections`, `riskyUsers`     | Entra ID Identity Protection | IdentityRiskEvent.Read.All; IdentityRiskyUser.Read.All | Entra ID P2  |
| `alerts_v2`, `incidents`           | Defender XDR           | SecurityAlert.Read.All; SecurityIncident.Read.All;	SecurityEvents.Read.All | Microsoft 365 Defender (sem custo extra) : |

---

##  Permissões

Enabling API permissions
The application requires specific API permissions to request Office 365 activity events. In this case, we are looking for permissions related to the https://manage.office.com resource.

Perform the following steps to configure the application permissions:

Navigate to the API permissions menu and choose Add a permission.

Select the Office 365 Management APIs and click on Application permissions.

Add the following permissions under the ActivityFeed group:

ActivityFeed.Read: Read activity data for your organization.

ActivityFeed.ReadDlp: Read DLP policy events including detected sensitive data.

Click on the Add permissions button.




