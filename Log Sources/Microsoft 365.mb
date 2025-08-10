# Detecção e Monitoramento de Ameaças com Microsoft 365

Este repositório apresenta a integração entre o SIEM da Logpoint e o Microsoft 365 via Microsoft Graph API para melhorar a visibilidade e a resposta a incidentes em ambientes corporativos.

---

##  Visão Geral

- **Autor do blog**: Priyanka Shrestha, Solutions Engineer :contentReference[oaicite:0]{index=0}
- **Publicação**: 7 de agosto de 2024 :contentReference[oaicite:1]{index=1}
- **Objetivo**: Demonstrar como integrar fontes de log do Microsoft 365 à plataforma Logpoint para correlação de eventos, detecção de anomalias e resposta rápida :contentReference[oaicite:2]{index=2}

---

##  Principais Componentes & Integrações

### Microsoft Entra ID (anteriormente Azure AD)
- Gerencia identidades e acessos com suporte a autenticação multifator e sem senha (passwordless) :contentReference[oaicite:3]{index=3}

### Microsoft Defender XDR (anteriormente Microsoft 365 Defender)
- Fornece visibilidade, investigação e correlação de alertas entre diferentes produtos Defender :contentReference[oaicite:4]{index=4}

---

##  Integrações via Microsoft Graph API com URAF (Universal REST API Fetcher)

A plataforma Logpoint utiliza URAF para integrar diretamente com os seguintes endpoints:

| Endpoint                        | Fonte Microsoft       | Permissões Necessárias                         | Licença Requerida                  |
|---------------------------------|------------------------|------------------------------------------------|------------------------------------|
| `directoryAudits` / `auditSignIns` | Entra ID               | AuditLog.Read.All                              | Entra ID P1 (alguns dados); P2 para mais detalhe :contentReference[oaicite:5]{index=5} |
| `riskDetections`, `riskyUsers`     | Entra ID Identity Protection | IdentityRiskEvent.Read.All; IdentityRiskyUser.Read.All | Entra ID P2 :contentReference[oaicite:6]{index=6} |
| `alerts_v2`, `incidents`           | Defender XDR           | SecurityAlert.Read.All; SecurityIncident.Read.All | Microsoft 365 Defender (sem custo extra) :contentReference[oaicite:7]{index=7} |

---

##  Configuração no Logpoint

1. Registrar um app no Microsoft Entra ID com permissões via OAuth 2.0  
2. Importar o pacote `.pak` do Log Source MicrosoftGraph no Logpoint  
3. Informar Tenant ID, Client ID e Client Secret  
4. Criar ou usar um repositório para armazenar os logs  
5. Normalização e enriquecimento são pré-configurados :contentReference[oaicite:8]{index=8}

---

##  Criação de Regras e Dashboards

Exemplos de query (em linguagem de busca da Logpoint) para monitoramento:

- **Alertas críticos no Defender 365**:
    ```text
    norm_id=MicrosoftGraph api_endpoint="security/alerts_v2" risk_level=high status=new | chart count() by log_ts,...
    ```
- **Detecção de risco elevado no Entra ID Identity Protection**:
    ```text
    norm_id=MicrosoftGraph api_endpoint="identityProtection/riskDetections" risk_level=high | chart count() by log_ts, user,...
    ```

Estas queries são usadas para construir painéis, alertas e relatórios que auxiliam na resposta a incidentes em tempo real :contentReference[oaicite:9]{index=9}

---

##  Como Usar Este Repositório

1. Atualize o arquivo README com informações específicas do seu ambiente (Tenant, IDs, licenças)
2. Inclua exemplos práticos de queries e screenshots dos dashboards criados
3. Adicione passos de configuração com imagens (se possível)

---

Se quiser, posso ajudar a refinar ainda mais o conteúdo ou traduzir partes para português técnico. Como você gostaria de continuar?
::contentReference[oaicite:10]{index=10}

