# GenieBot – Microsoft Teams com Databricks Genie

## 📖 Introdução
Este guia é um **manual completo e consolidado** para implantar um chatbot no **Microsoft Teams** que interage com a **API Genie do Databricks**, utilizando um fluxo de autenticação seguro com **Single Sign-On (SSO)**.  

Ele combina as melhores práticas de guias de referência com lições aprendidas em cenários reais, seguindo uma ordem comprovada de operações.  

---

## 🗂 Sumário
- [Arquitetura Final](#-arquitetura-final)  
- [Etapa 1: Preparação do Ambiente](#-etapa-1-preparação-do-ambiente-e-recolha-de-informações)  
- [Etapa 2: Criação da Infraestrutura no Azure](#-etapa-2-criação-da-infraestrutura-base-no-azure)  
- [Etapa 3: Configuração e Primeiro Deploy](#-etapa-3-configuração-pós-criação-e-primeiro-deploy)  
- [Etapa 4: Autenticação Avançada (SSO)](#-etapa-4-configuração-avançada-de-autenticação-sso)  
- [Etapa 5: Validação e Instalação no Teams](#-etapa-5-validação-final-e-instalação-no-teams)  

---

## 🏗 Arquitetura Final
- **Código-Fonte**: [Repositório base](https://github.com/carrosSoni/DatabricksGenieBOT), clonado para máquina local.  
- **Hospedagem**: App Service no Azure.  
- **Identidade & SSO**: App Registration no Microsoft Entra ID.  
- **Deploy**: Azure CLI.  
- **Conector**: Azure Bot.  
- **Instalador**: Pacote `.zip` do Teams criado manualmente.  

---

## 🛠 Etapa 1: Preparação do Ambiente e Recolha de Informações
1. **Limpeza Total**: exclua grupos de recursos, apps e pacotes antigos no Azure/Teams.  
2. **Repositório GitHub**:  
   - Faça um *fork* de: [DatabricksGenieBOT](https://github.com/carrosSoni/DatabricksGenieBOT)  
   - Clone o fork localmente.  
   - Confirme a existência de `requirements.txt`.  
3. **Pré-requisitos**:  
   - Acesso de administrador ao **Azure** e **Teams**.  
   - Colete informações do Databricks:  
     ```bash
     DATABRICKS_HOST   = https://adb-<workspace>.azuredatabricks.net
     DATABRICKS_TOKEN  = <seu-token-pat>
     DATABRICKS_SPACE_ID = <id-space-genie>
     ```

---

## ☁ Etapa 2: Criação da Infraestrutura Base no Azure
### 2.1 Grupo de Recursos
```text
Nome sugerido: geniebot-resource-group
```

### 2.2 Plano de Serviço de Aplicativo
- Nome: `geniebot-app-plan-service`  
- SO: Linux  
- SKU: B1 (Básico)  

### 2.3 Aplicativo Web
- Nome: `geniebot-web-app` (único globalmente)  
- Runtime: Python 3.10  
- Plano: `geniebot-app-plan-service`  

### 2.4 Azure Bot
- Nome: `geniebot-bot`  
- Dados: Global  
- Plano: Gratuito  
- Tipo: Multilocatário  
- Crie uma nova **App Registration** automaticamente  

> ⚠️ **Guarde as credenciais:**  
> - Microsoft **App ID**  
> - **Client Secret** (copiar imediatamente após gerar)

---

## 🚀 Etapa 3: Configuração Pós-Criação e Primeiro Deploy
### 3.1 Configurar o App Service
Adicione variáveis:  
```text
SCM_DO_BUILD_DURING_DEPLOYMENT = True
WEBSITE_HTTPLOGGING_RETENTION_DAYS = 3
```

Comando de inicialização:
```bash
gunicorn --bind 0.0.0.0 --worker-class aiohttp.worker.GunicornWebWorker --timeout 1200 app:app
```

Configure o **ponto de extremidade do Azure Bot**:  
```
https://<SEU_APP_SERVICE>.azurewebsites.net/api/messages
```

### 3.2 Deploy Inicial
```bash
az login
az webapp up --name "geniebot-web-app"              --resource-group "geniebot-resource-group"              --plan "geniebot-app-plan-service"
```

---

## 🔐 Etapa 4: Configuração Avançada de Autenticação (SSO)
1. Configure **Autenticação** → `https://token.botframework.com/.auth/web/redirect`  
2. Exponha a API → Crie escopo `scope_as_user`  
3. Autorize IDs de clientes do Teams:
   - `1fec8e78-bce4-4aaf-ab1b-5451cc387264` (Desktop/Mobile)  
   - `5e3ce6c0-2b1f-4285-8d4b-75ee78787346` (Web)  
4. Adicione permissões:  
   - **Databricks** → `user_impersonation`  
   - **Graph** → `email`, `openid`, `profile`, `offline_access`, `User.Read`  
5. Configure **OAuth Connection** no Azure Bot:  
   - Nome: `TeamsAuth`  
   - Provider: `Azure Active Directory v2`  
   - Scope:  
     ```
     2ff814a6-3304-4ab8-85cb-cd0e6f879c1d/.default
     ```

---

## ✅ Etapa 5: Validação Final e Instalação no Teams
### 5.1 Deploy Final
```bash
az webapp up --name "geniebot-web-app"              --resource-group "geniebot-resource-group"              --plan "geniebot-app-plan-service"
```

### 5.2 Teste no Azure
- Use **"Testar no Webchat"** no portal do Azure Bot.  
- Confirme a resposta do bot.  

### 5.3 Criação do Pacote do Teams
Arquivos necessários:
```
manifest.json
color.png (192x192)
outline.png (32x32)
```

Exemplo `manifest.json`:
```json
{
  "manifestVersion": "1.20",
  "id": "SEU_MICROSOFT_APP_ID_AQUI",
  "name": { "short": "Genie", "full": "Assistente Databricks" },
  "developer": {
    "name": "Sua Empresa",
    "websiteUrl": "https://www.suaempresa.com",
    "privacyUrl": "https://www.suaempresa.com/privacidade",
    "termsOfUseUrl": "https://www.suaempresa.com/termos"
  },
  "description": {
    "short": "Ajuda com insights referente a dados da empresa.",
    "full": "Permite usar linguagem natural para perguntar sobre dados da empresa de forma rápida e fácil."
  },
  "bots": [
    {
      "botId": "SEU_MICROSOFT_APP_ID_AQUI",
      "scopes": ["personal", "team"]
    }
  ],
  "webApplicationInfo": {
    "id": "SEU_MICROSOFT_APP_ID_AQUI",
    "resource": "SEU_URI_DA_ID_DO_APLICATIVO_AQUI"
  }
}
```

### 5.4 Instalação no Teams
- Vá até **Admin Center do Teams** → `Gerenciar aplicativos`.  
- **Carregue o .zip** e aprove.  
- Após propagação (~20 min), reinicie o Teams e adicione o bot.  

---

## 📌 Observações
- Sempre armazene segredos em **Key Vault** ou variáveis de ambiente (não no código).  
- Use logs do **App Service** para depuração.  
- Para produção, considere **Plano Premium** para maior resiliência.  
