#!/bin/bash
# Configuración de permisos para aibingsearchprincipal

# TUS DATOS ESPECÍFICOS
APP_ID="9cba7d58-03f7-4d4b-be36-d6913947fbd3"
TENANT_ID="a08665d0-8775-4712-be9f-4e23bf1f6e93"
SUBSCRIPTION_ID="c310bc6f-4771-4684-8c18-9f6188293c62"  # Del screenshot anterior
RESOURCE_GROUP="poc_asistente_virtual"                        # Tu resource group
SERVICE_PRINCIPAL_OBJECT_ID="d367515b-216b-44f2-8f46-58a8f2b33f0d" #Service principal object id
# Obtener Object ID del Service Principal
#SERVICE_PRINCIPAL_OBJECT_ID=$(az ad sp show --id $APP_ID --query "id" -o tsv)
#echo "🔍 Service Principal Object ID: $SERVICE_PRINCIPAL_OBJECT_ID"

echo "🔐 Configurando permisos para Grounding with Bing Search..."

# 1. Cognitive Services User (OBLIGATORIO para Bing Grounding)
echo "📋 Asignando: Cognitive Services User"
az role assignment create --assignee $SERVICE_PRINCIPAL_OBJECT_ID --role "Cognitive Services User" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

# 2. Azure AI Developer (OBLIGATORIO para Azure AI Foundry)
echo "📋 Asignando: Azure AI Developer"
az role assignment create --assignee $SERVICE_PRINCIPAL_OBJECT_ID --role "Azure AI Developer" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

# 3. Reader (para acceso de lectura a recursos)
echo "📋 Asignando: Reader"
az role assignment create --assignee $SERVICE_PRINCIPAL_OBJECT_ID --role "Reader" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

echo "✅ Permisos configurados!"

# Verificar asignaciones
echo "🔍 Verificando permisos asignados:"
az role assignment list --assignee $SERVICE_PRINCIPAL_OBJECT_ID --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP" --query "[].{Role:roleDefinitionName, Scope:scope}" -o table
