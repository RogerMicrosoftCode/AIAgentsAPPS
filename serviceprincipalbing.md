
#!/bin/bash
# Script para configurar roles en un Service Principal para Bing Search y Azure AI

# ===================== CONFIGURACIÓN =====================
# Reemplaza estos valores por los de tu entorno
APP_ID="9cba7d58-03f7-4d4b-be36-d6913947fbd3"
TENANT_ID="a08665d0-8775-4712-be9f-4e23bf1f6e93"
SUBSCRIPTION_ID="c310bc6f-4771-4684-8c18-9f6188293c62"
RESOURCE_GROUP="poc_asistente_virtual"
SERVICE_PRINCIPAL_OBJECT_ID="d367515b-216b-44f2-8f46-58a8f2b33f0d"
# Si prefieres obtener el Object ID automáticamente, descomenta la siguiente línea:
# SERVICE_PRINCIPAL_OBJECT_ID=$(az ad sp show --id $APP_ID --query "id" -o tsv)

# ===================== VALIDACIÓN DE VARIABLES =====================
for var in APP_ID TENANT_ID SUBSCRIPTION_ID RESOURCE_GROUP SERVICE_PRINCIPAL_OBJECT_ID; do
	if [ -z "${!var}" ]; then
		echo "❌ La variable $var no está definida. Revisa la configuración."
		exit 1
	fi
done

echo "🔐 Configurando permisos para el Service Principal..."

# ===================== ASIGNACIÓN DE ROLES =====================
# 1. Cognitive Services User (necesario para Bing Search)
echo "📋 Asignando rol: Cognitive Services User"
az role assignment create --assignee "$SERVICE_PRINCIPAL_OBJECT_ID" --role "Cognitive Services User" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
if [ $? -ne 0 ]; then echo "❌ Error asignando Cognitive Services User"; exit 1; fi

# 2. Azure AI Developer (necesario para Azure AI Foundry)
echo "📋 Asignando rol: Azure AI Developer"
az role assignment create --assignee "$SERVICE_PRINCIPAL_OBJECT_ID" --role "Azure AI Developer" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
if [ $? -ne 0 ]; then echo "❌ Error asignando Azure AI Developer"; exit 1; fi

# 3. Reader (acceso de solo lectura a recursos)
echo "📋 Asignando rol: Reader"
az role assignment create --assignee "$SERVICE_PRINCIPAL_OBJECT_ID" --role "Reader" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
if [ $? -ne 0 ]; then echo "❌ Error asignando Reader"; exit 1; fi

echo "✅ Permisos configurados correctamente!"

# ===================== VERIFICACIÓN DE ROLES =====================
echo "🔍 Verificando roles asignados:"
az role assignment list --assignee "$SERVICE_PRINCIPAL_OBJECT_ID" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP" --query "[].{Role:roleDefinitionName, Scope:scope}" -o table
