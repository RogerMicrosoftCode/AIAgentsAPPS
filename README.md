“””
🚀 IMPLEMENTACIÓN API - BING GROUNDING AZURE AI FOUNDRY
Convierte tu agente del playground en una API REST consumible

BASADO EN TU ENDPOINT:
https://amxfoundrywest.services.ai.azure.com/api/projects/chatasistente

INSTALACIÓN:
pip install azure-ai-projects azure-identity flask python-dotenv requests

VARIABLES DE ENTORNO (.env):
PROJECT_ENDPOINT=https://amxfoundrywest.services.ai.azure.com/api/projects/chatasistente
AGENT_ID=asst_jdrtyx2upZlf9tpnkOslKw0a
MODEL_DEPLOYMENT_NAME=gpt-4o
AZURE_CLIENT_ID=tu-client-id
AZURE_CLIENT_SECRET=tu-client-secret
AZURE_TENANT_ID=tu-tenant-id
“””

import os
import json
import time
from typing import Dict, List, Optional, Any
from flask import Flask, request, jsonify, make_response
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential, ClientSecretCredential
from dotenv import load_dotenv
import logging

# Configurar logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(**name**)

load_dotenv()

class BingGroundingAPIService:
“””
Servicio API que expone tu agente de Bing Grounding como REST API
“””

```
def __init__(self):
    self.project_endpoint = os.environ.get("PROJECT_ENDPOINT")
    self.agent_id = os.environ.get("AGENT_ID", "asst_jdrtyx2upZlf9tpnkOslKw0a")
    
    # Configurar autenticación
    self.credential = self._setup_authentication()
    
    # Crear cliente de AI Projects
    self.project_client = AIProjectClient(
        endpoint=self.project_endpoint,
        credential=self.credential
    )
    
    logger.info(f"✅ BingGroundingAPIService inicializado")
    logger.info(f"📍 Endpoint: {self.project_endpoint}")
    logger.info(f"🤖 Agent ID: {self.agent_id}")

def _setup_authentication(self):
    """
    Configurar autenticación para Azure
    """
    # Opción 1: Service Principal (recomendado para producción)
    if all([
        os.environ.get("AZURE_CLIENT_ID"),
        os.environ.get("AZURE_CLIENT_SECRET"), 
        os.environ.get("AZURE_TENANT_ID")
    ]):
        logger.info("🔐 Usando Service Principal authentication")
        return ClientSecretCredential(
            tenant_id=os.environ["AZURE_TENANT_ID"],
            client_id=os.environ["AZURE_CLIENT_ID"],
            client_secret=os.environ["AZURE_CLIENT_SECRET"]
        )
    else:
        # Opción 2: Default credential (para desarrollo local)
        logger.info("🔐 Usando Default Azure Credential")
        return DefaultAzureCredential()

def create_thread(self) -> str:
    """
    Crear un nuevo thread para conversación
    """
    try:
        thread = self.project_client.agents.threads.create()
        logger.info(f"📝 Thread creado: {thread.id}")
        return thread.id
    except Exception as e:
        logger.error(f"❌ Error creando thread: {e}")
        raise

def send_message_to_agent(self, thread_id: str, user_message: str) -> Dict[str, Any]:
    """
    Enviar mensaje al agente y obtener respuesta
    """
    try:
        # 1. Crear mensaje del usuario
        message = self.project_client.agents.messages.create(
            thread_id=thread_id,
            role="user",
            content=user_message
        )
        logger.info(f"📤 Mensaje enviado: {message.id}")
        
        # 2. Crear y ejecutar run
        run = self.project_client.agents.runs.create_and_process(
            thread_id=thread_id,
            agent_id=self.agent_id
        )
        
        logger.info(f"🔄 Run completado: {run.id} - Status: {run.status}")
        
        if run.status == "failed":
            error_msg = run.last_error.message if run.last_error else "Unknown error"
            logger.error(f"❌ Run falló: {error_msg}")
            return {
                "success": False,
                "error": error_msg,
                "run_id": run.id
            }
        
        # 3. Obtener respuesta del agente
        messages = list(self.project_client.agents.messages.list(thread_id=thread_id))
        
        # Buscar la última respuesta del agente
        agent_response = None
        citations = []
        
        for msg in messages:
            if msg.role == "assistant":
                # Obtener contenido de texto
                if msg.content:
                    for content_item in msg.content:
                        if hasattr(content_item, 'text') and hasattr(content_item.text, 'value'):
                            agent_response = content_item.text.value
                            
                            # Obtener anotaciones (citaciones)
                            if hasattr(content_item.text, 'annotations'):
                                for annotation in content_item.text.annotations:
                                    if hasattr(annotation, 'url_citation'):
                                        citations.append({
                                            "title": annotation.url_citation.title,
                                            "url": annotation.url_citation.url,
                                            "text": annotation.text
                                        })
                break
        
        # 4. Obtener detalles de búsqueda de Bing si están disponibles
        bing_queries = []
        try:
            run_steps = list(self.project_client.agents.run_steps.list(
                thread_id=thread_id,
                run_id=run.id
            ))
            
            for step in run_steps:
                if hasattr(step, 'step_details') and hasattr(step.step_details, 'tool_calls'):
                    for tool_call in step.step_details.tool_calls:
                        if hasattr(tool_call, 'bing_grounding'):
                            # Extraer query de Bing si está disponible
                            bing_data = tool_call.bing_grounding
                            if isinstance(bing_data, dict) and 'query' in bing_data:
                                query = bing_data['query']
                                bing_url = f"https://www.bing.com/search?q={query.replace(' ', '+')}&setlang=es&mkt=es-MX"
                                bing_queries.append(bing_url)
        except Exception as e:
            logger.warning(f"⚠️ No se pudieron obtener detalles de Bing: {e}")
        
        return {
            "success": True,
            "response": agent_response or "No se obtuvo respuesta del agente",
            "citations": citations,
            "bing_queries": bing_queries,
            "run_id": run.id,
            "thread_id": thread_id
        }
        
    except Exception as e:
        logger.error(f"❌ Error enviando mensaje: {e}")
        return {
            "success": False,
            "error": str(e)
        }

def cleanup_thread(self, thread_id: str):
    """
    Limpiar thread (opcional)
    """
    try:
        self.project_client.agents.threads.delete(thread_id)
        logger.info(f"🗑️ Thread eliminado: {thread_id}")
    except Exception as e:
        logger.warning(f"⚠️ Error eliminando thread: {e}")
```

# CREAR FLASK API

app = Flask(**name**)
bing_service = BingGroundingAPIService()

@app.route(’/health’, methods=[‘GET’])
def health_check():
“””
Health check endpoint
“””
return jsonify({
“status”: “healthy”,
“service”: “bing_grounding_api”,
“timestamp”: time.strftime(”%Y-%m-%dT%H:%M:%SZ”),
“agent_id”: bing_service.agent_id
})

@app.route(’/chat’, methods=[‘POST’])
def chat_endpoint():
“””
Endpoint principal para chat con el agente

```
POST /chat
{
    "message": "¿Cuál es el precio del dólar hoy?",
    "session_id": "user123",
    "create_new_thread": true
}
"""
try:
    data = request.get_json()
    
    if not data or 'message' not in data:
        return jsonify({
            "success": False,
            "error": "Campo 'message' requerido"
        }), 400
    
    user_message = data['message']
    session_id = data.get('session_id', 'anonymous')
    create_new_thread = data.get('create_new_thread', True)
    thread_id = data.get('thread_id')
    
    # Crear nuevo thread si es necesario
    if create_new_thread or not thread_id:
        thread_id = bing_service.create_thread()
    
    logger.info(f"📨 Chat request - Session: {session_id}, Message: {user_message[:50]}...")
    
    # Enviar mensaje al agente
    result = bing_service.send_message_to_agent(thread_id, user_message)
    
    # Preparar respuesta
    response_data = {
        "success": result["success"],
        "response": result.get("response", ""),
        "citations": result.get("citations", []),
        "bing_queries": result.get("bing_queries", []),
        "session_id": session_id,
        "thread_id": thread_id,
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ"),
        "metadata": {
            "provider": "azure_bing_grounding",
            "agent_id": bing_service.agent_id,
            "run_id": result.get("run_id")
        }
    }
    
    if not result["success"]:
        response_data["error"] = result.get("error", "Unknown error")
        return jsonify(response_data), 500
    
    return jsonify(response_data)
    
except Exception as e:
    logger.error(f"❌ Error en chat endpoint: {e}")
    return jsonify({
        "success": False,
        "error": str(e),
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ")
    }), 500
```

@app.route(’/chat/continue’, methods=[‘POST’])
def chat_continue_endpoint():
“””
Continuar conversación en thread existente

```
POST /chat/continue
{
    "message": "¿Y en pesos mexicanos?",
    "thread_id": "thread_abc123"
}
"""
try:
    data = request.get_json()
    
    if not data or 'message' not in data or 'thread_id' not in data:
        return jsonify({
            "success": False,
            "error": "Campos 'message' y 'thread_id' requeridos"
        }), 400
    
    user_message = data['message']
    thread_id = data['thread_id']
    
    logger.info(f"🔄 Continue chat - Thread: {thread_id}, Message: {user_message[:50]}...")
    
    # Enviar mensaje al agente
    result = bing_service.send_message_to_agent(thread_id, user_message)
    
    response_data = {
        "success": result["success"],
        "response": result.get("response", ""),
        "citations": result.get("citations", []),
        "bing_queries": result.get("bing_queries", []),
        "thread_id": thread_id,
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ")
    }
    
    if not result["success"]:
        response_data["error"] = result.get("error")
        return jsonify(response_data), 500
        
    return jsonify(response_data)
    
except Exception as e:
    logger.error(f"❌ Error en continue endpoint: {e}")
    return jsonify({
        "success": False,
        "error": str(e)
    }), 500
```

@app.route(’/thread/<thread_id>’, methods=[‘DELETE’])
def delete_thread(thread_id: str):
“””
Eliminar thread específico
“””
try:
bing_service.cleanup_thread(thread_id)
return jsonify({
“success”: True,
“message”: f”Thread {thread_id} eliminado”
})
except Exception as e:
return jsonify({
“success”: False,
“error”: str(e)
}), 500

@app.route(’/agent/info’, methods=[‘GET’])
def agent_info():
“””
Información del agente
“””
return jsonify({
“agent_id”: bing_service.agent_id,
“endpoint”: bing_service.project_endpoint,
“capabilities”: [
“bing_grounding_search”,
“real_time_web_data”,
“spanish_language_support”
],
“supported_markets”: [“es-MX”, “es-ES”, “es-AR”, “es-CL”],
“version”: “1.0.0”
})

# CORS support (si necesitas llamadas desde frontend)

@app.after_request
def after_request(response):
response.headers.add(‘Access-Control-Allow-Origin’, ‘*’)
response.headers.add(‘Access-Control-Allow-Headers’, ‘Content-Type,Authorization’)
response.headers.add(‘Access-Control-Allow-Methods’, ‘GET,PUT,POST,DELETE,OPTIONS’)
return response

# CLIENT DE PRUEBA

class BingGroundingAPIClient:
“””
Cliente Python para consumir tu API
“””

```
def __init__(self, api_base_url: str = "http://localhost:5000"):
    self.api_base_url = api_base_url
    
def send_message(self, message: str, session_id: str = None) -> Dict[str, Any]:
    """
    Enviar mensaje al agente
    """
    import requests
    
    payload = {
        "message": message,
        "session_id": session_id or "test_session",
        "create_new_thread": True
    }
    
    response = requests.post(f"{self.api_base_url}/chat", json=payload)
    return response.json()

def continue_conversation(self, message: str, thread_id: str) -> Dict[str, Any]:
    """
    Continuar conversación
    """
    import requests
    
    payload = {
        "message": message,
        "thread_id": thread_id
    }
    
    response = requests.post(f"{self.api_base_url}/chat/continue", json=payload)
    return response.json()
```

# SCRIPT DE TESTING

def test_api_locally():
“””
Test local de la API
“””
print(“🧪 TESTING BING GROUNDING API”)
print(”=” * 50)

```
# Verificar que las variables estén configuradas
required_vars = ["PROJECT_ENDPOINT", "AGENT_ID"]
missing_vars = [var for var in required_vars if not os.getenv(var)]

if missing_vars:
    print(f"❌ Variables faltantes: {', '.join(missing_vars)}")
    print("\nConfigura tu .env:")
    print("PROJECT_ENDPOINT=https://amxfoundrywest.services.ai.azure.com/api/projects/chatasistente")
    print("AGENT_ID=asst_jdrtyx2upZlf9tpnkOslKw0a")
    return

# Test del servicio
try:
    service = BingGroundingAPIService()
    
    # Test 1: Crear thread
    print("\n1️⃣ Creando thread...")
    thread_id = service.create_thread()
    print(f"✅ Thread creado: {thread_id}")
    
    # Test 2: Enviar mensaje
    print("\n2️⃣ Enviando mensaje...")
    test_message = "¿Cuál es el precio del dólar hoy en México?"
    result = service.send_message_to_agent(thread_id, test_message)
    
    if result["success"]:
        print(f"✅ Respuesta recibida: {result['response'][:100]}...")
        print(f"📚 Citaciones: {len(result['citations'])}")
        print(f"🔍 Queries Bing: {len(result['bing_queries'])}")
    else:
        print(f"❌ Error: {result['error']}")
    
    # Test 3: Cleanup
    print("\n3️⃣ Limpiando recursos...")
    service.cleanup_thread(thread_id)
    print("✅ Cleanup completado")
    
except Exception as e:
    print(f"❌ Error en testing: {e}")
```

if **name** == “**main**”:
import sys

```
if len(sys.argv) > 1 and sys.argv[1] == "test":
    test_api_locally()
else:
    # Ejecutar Flask API
    print("🚀 INICIANDO BING GROUNDING API SERVER")
    print("=" * 50)
    print(f"📍 Endpoint: http://localhost:5000")
    print(f"🤖 Agent ID: {os.getenv('AGENT_ID', 'not_configured')}")
    print(f"🔗 Project: {os.getenv('PROJECT_ENDPOINT', 'not_configured')}")
    print("\n📚 Endpoints disponibles:")
    print("  GET  /health          - Health check")
    print("  POST /chat            - Nuevo chat")
    print("  POST /chat/continue   - Continuar chat")
    print("  GET  /agent/info      - Info del agente")
    print("  DELETE /thread/<id>   - Eliminar thread")
    print("\n🧪 Para testing: python script.py test")
    print("=" * 50)
    
    app.run(debug=True, host='0.0.0.0', port=5000)
```

“””
EJEMPLOS DE USO DE LA API:

1. NUEVO CHAT:
   curl -X POST http://localhost:5000/chat   
     -H “Content-Type: application/json”   
     -d ‘{
   “message”: “¿Cuál es el precio del petróleo hoy?”,
   “session_id”: “user123”
   }’
1. CONTINUAR CONVERSACIÓN:
   curl -X POST http://localhost:5000/chat/continue   
     -H “Content-Type: application/json”   
     -d ‘{
   “message”: “¿Y en los últimos 30 días?”,
   “thread_id”: “thread_abc123”
   }’
1. HEALTH CHECK:
   curl http://localhost:5000/health
1. INFO DEL AGENTE:
   curl http://localhost:5000/agent/info

RESPUESTA TÍPICA:
{
“success”: true,
“response”: “El precio del petróleo WTI hoy es de $73.45 por barril…”,
“citations”: [
{
“title”: “Precio del Petróleo Hoy - Reuters”,
“url”: “https://reuters.com/…”,
“text”: “[1]”
}
],
“bing_queries”: [“https://www.bing.com/search?q=precio+petroleo+hoy&setlang=es&mkt=es-MX”],
“thread_id”: “thread_abc123”,
“timestamp”: “2025-09-03T10:30:00Z”
}

CONFIGURACIÓN PARA PRODUCCIÓN:

- Usar Service Principal (CLIENT_ID, CLIENT_SECRET, TENANT_ID)
- Configurar HTTPS
- Agregar rate limiting
- Implementar logging a Azure Monitor
- Usar Azure App Service o Container Apps
  “””