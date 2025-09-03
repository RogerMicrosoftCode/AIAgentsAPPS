## **🔐 CONFIGURACIÓN DE PERMISOS - TU SERVICE PRINCIPAL**

### **Tus datos específicos:**
- **App ID**: `9cba7d58-03f7-4d4b-be36-d6913947fbd3`
- **Display Name**: `aibingsearchprincipal`
- **Tenant ID**: `a08665d0-8775-4712-be9f-4e23bf1f6e93`

### **1. Obtener Object ID del Service Principal**

```bash
# Obtener el Object ID (necesario para asignar permisos)
az ad sp show --id "9cba7d58-03f7-4d4b-be36-d6913947fbd3" --query "id" -o tsv
```

Guarda este Object ID, lo necesitaremos para los permisos.

### **2. Configurar permisos específicos para Bing Grounding**

```bash
#!/bin/bash
# Configuración de permisos para aibingsearchprincipal

# TUS DATOS ESPECÍFICOS
APP_ID="9cba7d58-03f7-4d4b-be36-d6913947fbd3"
TENANT_ID="a08665d0-8775-4712-be9f-4e23bf1f6e93"
SUBSCRIPTION_ID="c310bc6f-4771-4684-8c18-9f6188293c62"  # Del screenshot anterior
RESOURCE_GROUP="poc_asistente_v2"                        # Tu resource group

# Obtener Object ID del Service Principal
SERVICE_PRINCIPAL_OBJECT_ID=$(az ad sp show --id $APP_ID --query "id" -o tsv)
echo "🔍 Service Principal Object ID: $SERVICE_PRINCIPAL_OBJECT_ID"

echo "🔐 Configurando permisos para Grounding with Bing Search..."

# 1. Cognitive Services User (OBLIGATORIO para Bing Grounding)
echo "📋 Asignando: Cognitive Services User"
az role assignment create \
  --assignee $SERVICE_PRINCIPAL_OBJECT_ID \
  --role "Cognitive Services User" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

# 2. Azure AI Developer (OBLIGATORIO para Azure AI Foundry)
echo "📋 Asignando: Azure AI Developer"
az role assignment create \
  --assignee $SERVICE_PRINCIPAL_OBJECT_ID \
  --role "Azure AI Developer" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

# 3. Reader (para acceso de lectura a recursos)
echo "📋 Asignando: Reader"
az role assignment create \
  --assignee $SERVICE_PRINCIPAL_OBJECT_ID \
  --role "Reader" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

echo "✅ Permisos configurados!"

# Verificar asignaciones
echo "🔍 Verificando permisos asignados:"
az role assignment list \
  --assignee $SERVICE_PRINCIPAL_OBJECT_ID \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP" \
  --query "[].{Role:roleDefinitionName, Scope:scope}" \
  -o table
```

### **3. Configurar tu archivo .env**

```bash
# .env file - TUS CREDENCIALES ESPECÍFICAS
PROJECT_ENDPOINT=https://amxfoundrywest.services.ai.azure.com/api/projects/chatasistente
AGENT_ID=asst_jdrtyx2upZlf9tpnkOslKw0a
MODEL_DEPLOYMENT_NAME=gpt-4o

# SERVICE PRINCIPAL CREDENTIALS
AZURE_CLIENT_ID=9cba7d58-03f7-4d4b-be36-d6913947fbd3
AZURE_CLIENT_SECRET=YXV8Q~13c0T2hn0S_wJBxVNwHljwTE5ow49SKbBJ
AZURE_TENANT_ID=a08665d0-8775-4712-be9f-4e23bf1f6e93

# AZURE RESOURCE INFO
AZURE_SUBSCRIPTION_ID=c310bc6f-4771-4684-8c18-9f6188293c62
AZURE_RESOURCE_GROUP=poc_asistente_v2
```

### **4. Obtener Connection ID de Bing Grounding**

```bash
# Encontrar tu recurso de Bing Grounding
az cognitiveservices account list \
  --resource-group "poc_asistente_v2" \
  --query "[?kind=='BingGrounding'].{Name:name, Id:id, Location:location}" \
  -o table

# Una vez que tengas el nombre, obtener el connection ID completo
BING_RESOURCE_NAME="tu-bing-grounding-resource-name"  # Reemplazar con el nombre real
echo "/subscriptions/c310bc6f-4771-4684-8c18-9f6188293c62/resourceGroups/poc_asistente_v2/providers/Microsoft.CognitiveServices/accounts/$BING_RESOURCE_NAME"
```

### **5. Test de conectividad**

```bash
# Script de testing con tus credenciales específicas
#!/bin/bash

export AZURE_CLIENT_ID="9cba7d58-03f7-4d4b-be36-d6913947fbd3"
export AZURE_CLIENT_SECRET="Y1287361289736198236719286312J"  
export AZURE_TENANT_ID="a08665d0-8775-4712-be9f-4e23bf1f6e93"

echo "🧪 Testing Service Principal authentication..."

# Test 1: Login con Service Principal
az login --service-principal \
  --username $AZURE_CLIENT_ID \
  --password $AZURE_CLIENT_SECRET \
  --tenant $AZURE_TENANT_ID

# Test 2: Verificar acceso a recursos
echo "🔍 Verificando acceso a Cognitive Services..."
az cognitiveservices account list \
  --resource-group "poc_asistente_v2" \
  --query "[].{Name:name, Kind:kind, Location:location}" \
  -o table

# Test 3: Verificar acceso a AI Foundry
echo "🤖 Verificando acceso a AI Foundry..."
az ml workspace list \
  --resource-group "poc_asistente_v2" \
  --query "[].{Name:name, Location:location}" \
  -o table 2>/dev/null || echo "No AI Foundry resources found or no access"

echo "✅ Testing completado!"
```

---

## **🎯 COMANDOS DIRECTOS PARA EJECUTAR**

Con tus datos específicos, ejecuta estos comandos:

```bash
# 1. Obtener Object ID
SERVICE_PRINCIPAL_OBJECT_ID=$(az ad sp show --id "9cba7d58-03f7-4d4b-be36-d6913947fbd3" --query "id" -o tsv)
echo "Object ID: $SERVICE_PRINCIPAL_OBJECT_ID"

# 2. Asignar permisos (reemplaza SUBSCRIPTION_ID con el correcto)
SUBSCRIPTION_ID="c310bc6f-4771-4684-8c18-9f6188293c62"  # Confirma este ID
RESOURCE_GROUP="poc_asistente_v2"

az role assignment create --assignee $SERVICE_PRINCIPAL_OBJECT_ID --role "Cognitive Services User" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
az role assignment create --assignee $SERVICE_PRINCIPAL_OBJECT_ID --role "Azure AI Developer" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
az role assignment create --assignee $SERVICE_PRINCIPAL_OBJECT_ID --role "Reader" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

# 3. Verificar permisos
az role assignment list --assignee $SERVICE_PRINCIPAL_OBJECT_ID --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP" -o table
```

---

## **⚠️ IMPORTANTE - SEGURIDAD**

Ya que compartiste el **Client Secret**, te recomiendo:

1. **Regenerar el secreto inmediatamente**:
```bash
az ad app credential reset --id "9cba7d58-03f7-4d4b-be36-d6913947fbd3"
```

2. **Copiar el nuevo secreto** y actualizarlo en tu configuración
---

**¿Quieres que te ayude a ejecutar estos comandos paso a paso, o tienes algún error específico?*
Una vez configurados los permisos, tu API funcionará perfectamente con tu Service Principal.
