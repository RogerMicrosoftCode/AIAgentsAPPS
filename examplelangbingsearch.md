"""
🦜 LANGCHAIN INTEGRATION CON TUS DATOS ESPECÍFICOS
Integración usando tu agente existente: asst_jdrtyx2upZlf9tpnkOslKw0a
Endpoint: https://amxfoundrywest.services.ai.azure.com/api/projects/chatasistente

INSTALACIÓN:
pip install langchain langchain-openai azure-ai-projects azure-identity python-dotenv

CONFIGURACIÓN (.env):
AZURE_CLIENT_ID=9cba7d58-03f7-4d4b-be36-d6913947fbd3
AZURE_CLIENT_SECRET=tu-nuevo-secreto-regenerado
AZURE_TENANT_ID=a08665d0-8775-4712-be9f-4e23bf1f6e93
PROJECT_ENDPOINT=https://amxfoundrywest.services.ai.azure.com/api/projects/chatasistente
AGENT_ID=asst_jdrtyx2upZlf9tpnkOslKw0a
"""

import os
import json
import time
from typing import Dict, List, Any, Optional
from dotenv import load_dotenv

# LangChain imports
from langchain.tools import BaseTool
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain.prompts import ChatPromptTemplate
from langchain_openai import AzureChatOpenAI
from langchain.schema import BaseMessage, HumanMessage, AIMessage
from langchain.callbacks.manager import CallbackManagerForToolRun

# Azure imports - usando TU código como base
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential, ClientSecretCredential
from azure.ai.agents.models import ListSortOrder

# Logging
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

load_dotenv()

class YourBingSearchAgentTool(BaseTool):
    """
    LangChain Tool que usa EXACTAMENTE tu agente existente de Bing Search
    Agent ID: asst_jdrtyx2upZlf9tpnkOslKw0a
    """
    
    name: str = "your_bing_search_agent"
    description: str = """
    Herramienta de búsqueda web que usa tu agente específico de Azure AI Foundry.
    Úsala para obtener información actualizada de internet, noticias, precios, clima, etc.
    
    Input: Una consulta de búsqueda (ej: "precio del dólar", "noticias de IA", "clima en CDMX")
    Output: Información actualizada con citaciones de fuentes
    """
    
    def __init__(self):
        super().__init__()
        self._setup_client()
        # Tu agente específico
        self.agent_id = "asst_jdrtyx2upZlf9tpnkOslKw0a"
    
    def _setup_client(self):
        """Configurar cliente usando TUS credenciales específicas"""
        try:
            # Usar Service Principal credentials
            credential = ClientSecretCredential(
                tenant_id=os.environ["AZURE_TENANT_ID"],
                client_id=os.environ["AZURE_CLIENT_ID"], 
                client_secret=os.environ["AZURE_CLIENT_SECRET"]
            )
            
            # Tu endpoint específico
            self.project = AIProjectClient(
                credential=credential,
                endpoint=os.environ["PROJECT_ENDPOINT"]
            )
            
            logger.info(f"✅ Cliente configurado con endpoint: {os.environ['PROJECT_ENDPOINT']}")
            logger.info(f"🤖 Usando Agent ID: {self.agent_id}")
            
        except Exception as e:
            logger.error(f"❌ Error configurando cliente: {e}")
            raise
    
    def _run(
        self, 
        query: str, 
        run_manager: Optional[CallbackManagerForToolRun] = None
    ) -> str:
        """
        Ejecutar búsqueda usando EXACTAMENTE tu lógica existente
        """
        try:
            logger.info(f"🔍 Ejecutando búsqueda con tu agente: {query}")
            
            # PASO 1: Crear thread (tu código)
            thread = self.project.agents.threads.create()
            logger.info(f"📝 Thread creado: {thread.id}")
            
            # PASO 2: Crear mensaje (tu código)
            message = self.project.agents.messages.create(
                thread_id=thread.id,
                role="user",
                content=query  # Usar el query de LangChain
            )
            logger.info(f"📤 Mensaje creado: {message.id}")
            
            # PASO 3: Ejecutar run (tu código)
            run = self.project.agents.runs.create_and_process(
                thread_id=thread.id,
                agent_id=self.agent_id  # Tu agente específico
            )
            logger.info(f"🔄 Run completado: {run.id} - Status: {run.status}")
            
            # PASO 4: Verificar si falló
            if run.status == "failed":
                error_msg = run.last_error.message if run.last_error else "Unknown error"
                logger.error(f"❌ Run falló: {error_msg}")
                return f"Error en búsqueda: {error_msg}"
            
            # PASO 5: Obtener mensajes (tu código con mejoras)
            messages = self.project.agents.messages.list(
                thread_id=thread.id, 
                order=ListSortOrder.ASCENDING
            )
            
            # PASO 6: Extraer respuesta del agente
            agent_response = None
            citations = []
            
            for message in messages:
                if message.role == "assistant":
                    # Obtener contenido de texto
                    if hasattr(message, 'text_messages'):
                        for text_message in message.text_messages:
                            agent_response = text_message.text.value
                            logger.info(f"✅ Respuesta obtenida: {len(agent_response)} caracteres")
                            break
                    
                    # Obtener citaciones URL
                    if hasattr(message, 'url_citation_annotations'):
                        for annotation in message.url_citation_annotations:
                            citations.append({
                                "title": annotation.url_citation.title,
                                "url": annotation.url_citation.url,
                                "text": annotation.text
                            })
                    break
            
            # PASO 7: Cleanup thread
            try:
                self.project.agents.threads.delete(thread.id)
                logger.info(f"🗑️ Thread limpiado: {thread.id}")
            except Exception as cleanup_error:
                logger.warning(f"⚠️ Error en cleanup: {cleanup_error}")
            
            # PASO 8: Formatear respuesta con citaciones
            if not agent_response:
                return "No se pudo obtener respuesta del agente Bing Search"
            
            # Agregar citaciones si existen
            if citations:
                citations_text = "\n\n📚 Fuentes consultadas:\n" + "\n".join([
                    f"• [{c['title']}]({c['url']})" for c in citations[:5]
                ])
                final_response = f"{agent_response}{citations_text}"
            else:
                final_response = agent_response
            
            logger.info(f"🎉 Búsqueda completada exitosamente")
            return final_response
            
        except Exception as e:
            logger.error(f"❌ Error ejecutando búsqueda: {e}")
            return f"Error en herramienta de búsqueda: {str(e)}"

class YourLangChainBingAgent:
    """
    Agente LangChain que integra TU agente específico de Bing Search
    """
    
    def __init__(self, azure_openai_endpoint: str = None, azure_openai_key: str = None):
        self.azure_openai_endpoint = azure_openai_endpoint
        self.azure_openai_key = azure_openai_key
        
        logger.info("🚀 Inicializando YourLangChainBingAgent...")
        
        # Setup components
        self._setup_llm()
        self._setup_tools() 
        self._setup_agent()
        
        logger.info("✅ LangChain Agent listo con tu Bing Search integrado")
    
    def _setup_llm(self):
        """Configurar LLM (opcional si quieres usar otro LLM además de tu agente)"""
        if self.azure_openai_endpoint and self.azure_openai_key:
            try:
                self.llm = AzureChatOpenAI(
                    azure_endpoint=self.azure_openai_endpoint,
                    api_key=self.azure_openai_key,
                    api_version="2024-05-01-preview", 
                    azure_deployment="gpt-4o",
                    temperature=0.7,
                    max_tokens=2000
                )
                logger.info("✅ LLM adicional configurado")
            except Exception as e:
                logger.warning(f"⚠️ No se pudo configurar LLM adicional: {e}")
                self.llm = None
        else:
            self.llm = None
            logger.info("ℹ️ Solo usando tu agente de Bing Search (sin LLM adicional)")
    
    def _setup_tools(self):
        """Configurar herramientas - tu Bing Search Tool"""
        self.bing_tool = YourBingSearchAgentTool()
        self.tools = [self.bing_tool]
        logger.info(f"✅ {len(self.tools)} herramientas configuradas")
    
    def _setup_agent(self):
        """Configurar agente solo si tenemos LLM"""
        if self.llm:
            # Prompt optimizado para trabajar con tu agente específico
            system_prompt = """
            Tienes acceso a un agente especializado de búsqueda web con Bing Search de Azure.
            
            INSTRUCCIONES:
            1. Usa la herramienta your_bing_search_agent cuando necesites información actualizada:
               - Noticias recientes
               - Precios actuales 
               - Clima y condiciones meteorológicas
               - Datos que cambian frecuentemente
               - Eventos actuales
            
            2. El agente ya maneja la búsqueda y citaciones automáticamente
            
            3. Responde en español, de manera clara y precisa
            
            4. Si el agente incluye fuentes, asegúrate de mencionarlas
            
            EJEMPLOS de uso:
            - Usuario: "¿Cuál es el precio del Bitcoin?"
            - Tú: [usar herramienta] → responder con la información obtenida
            """
            
            prompt = ChatPromptTemplate.from_messages([
                ("system", system_prompt),
                ("human", "{input}"),
                ("placeholder", "{agent_scratchpad}")
            ])
            
            # Crear agente con herramientas
            agent = create_openai_tools_agent(
                llm=self.llm,
                tools=self.tools, 
                prompt=prompt
            )
            
            # Crear executor
            self.agent_executor = AgentExecutor(
                agent=agent,
                tools=self.tools,
                verbose=True,
                max_iterations=3,
                early_stopping_method="generate"
            )
            
            logger.info("✅ Agente LangChain configurado con LLM")
        else:
            self.agent_executor = None
            logger.info("ℹ️ Modo directo - solo usando tu agente Bing")
    
    def chat(self, message: str) -> str:
        """
        Chat principal - usa tu agente directamente o con LangChain
        """
        try:
            if self.agent_executor:
                # Modo LangChain con LLM adicional
                logger.info(f"💬 Procesando con LangChain: {message}")
                result = self.agent_executor.invoke({"input": message})
                return result.get("output", "No se pudo generar respuesta")
            else:
                # Modo directo - solo tu agente Bing
                logger.info(f"💬 Procesando directamente con tu agente: {message}")
                return self.bing_tool._run(message)
                
        except Exception as e:
            logger.error(f"❌ Error en chat: {e}")
            return f"Error procesando mensaje: {str(e)}"

# WRAPPER SIMPLIFICADO para integración rápida en TU aplicación
class YourBingSearchWrapper:
    """
    Wrapper que encapsula TU agente de Bing Search para uso directo
    """
    
    def __init__(self):
        logger.info("🔧 Inicializando wrapper de tu agente Bing Search...")
        self.agent = YourLangChainBingAgent()  # Sin LLM adicional
        logger.info("✅ Wrapper listo")
    
    def search(self, query: str) -> Dict[str, Any]:
        """
        REEMPLAZO DIRECTO de tu función de búsqueda anterior
        
        ANTES (Bing API deprecated):
        def search(query):
            return bing_api.search(query)  # Ya no funciona
        
        DESPUÉS (con tu agente):
        wrapper = YourBingSearchWrapper()
        return wrapper.search(query)
        """
        try:
            response = self.agent.chat(query)
            
            # Extraer fuentes de la respuesta (simple regex)
            import re
            sources = re.findall(r'https?://[^\s\)]+', response)
            
            # Detectar si se obtuvieron resultados válidos
            success = len(response) > 50 and "Error" not in response
            
            return {
                "success": success,
                "response": response,
                "sources": sources,
                "query": query,
                "agent_id": self.agent.bing_tool.agent_id,
                "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ")
            }
            
        except Exception as e:
            return {
                "success": False,
                "response": f"Error: {str(e)}",
                "sources": [],
                "query": query,
                "error": str(e)
            }

# EJEMPLO DE MIGRACIÓN DESDE TU CÓDIGO ACTUAL
def ejemplo_migracion():
    """
    Ejemplo de cómo migrar desde tu implementación actual
    """
    print("🔄 EJEMPLO DE MIGRACIÓN")
    print("=" * 50)
    
    # ANTES - Tu código actual (simplificado)
    def tu_codigo_anterior(query):
        """
        # Tu código que ya no funciona por Bing API deprecated
        from azure.ai.projects import AIProjectClient
        from azure.identity import DefaultAzureCredential
        
        project = AIProjectClient(
            credential=DefaultAzureCredential(),
            endpoint="https://amxfoundrywest.services.ai.azure.com/api/projects/chatasistente"
        )
        
        agent = project.agents.get_agent("asst_jdrtyx2upZlf9tpnkOslKw0a")
        thread = project.agents.threads.create()
        
        message = project.agents.messages.create(
            thread_id=thread.id,
            role="user", 
            content=query
        )
        
        run = project.agents.runs.create_and_process(
            thread_id=thread.id,
            agent_id=agent.id
        )
        
        # Procesar resultados...
        return "resultado"
        """
        pass
    
    # DESPUÉS - Con LangChain integrado
    def tu_codigo_nuevo(query):
        """
        Nuevo código que integra todo con LangChain
        """
        wrapper = YourBingSearchWrapper()
        result = wrapper.search(query)
        return result["response"]
    
    # Test de comparación
    test_query = "¿Cuál es el precio del dólar hoy en México?"
    
    print(f"📝 Query de prueba: {test_query}")
    print("\n🆕 Resultado con nuevo wrapper:")
    
    try:
        nuevo_resultado = tu_codigo_nuevo(test_query)
        print(f"✅ {nuevo_resultado[:200]}...")
    except Exception as e:
        print(f"❌ Error: {e}")

# TESTING FUNCTIONS
def test_your_bing_tool():
    """Test directo de tu herramienta Bing"""
    print("🧪 TESTING TU HERRAMIENTA BING SEARCH")
    print("=" * 50)
    
    try:
        tool = YourBingSearchAgentTool()
        
        test_queries = [
            "¿Cuál es el precio del Bitcoin hoy?",
            "Últimas noticias sobre inteligencia artificial",
            "¿Qué tiempo hace en Ciudad de México?"
        ]
        
        for i, query in enumerate(test_queries, 1):
            print(f"\n{i}. 🔍 Probando: {query}")
            print("-" * 40)
            
            start_time = time.time()
            result = tool._run(query)
            elapsed = time.time() - start_time
            
            print(f"⏱️ Tiempo: {elapsed:.2f}s")
            print(f"📋 Resultado: {result[:150]}...")
            
            # Detectar si hay fuentes
            sources_count = result.count("http")
            print(f"📚 Fuentes detectadas: {sources_count}")
            print("=" * 50)
            
    except Exception as e:
        print(f"❌ Error en test: {e}")

def test_langchain_integration():
    """Test de integración completa con LangChain"""
    print("🧪 TESTING INTEGRACIÓN LANGCHAIN COMPLETA")
    print("=" * 50)
    
    try:
        # Crear wrapper
        wrapper = YourBingSearchWrapper()
        
        # Test queries
        queries = [
            "¿Cuál es el precio del petróleo WTI hoy?",
            "¿Cuáles son las últimas noticias sobre Microsoft Azure?",
            "¿Cómo está el clima en Madrid hoy?"
        ]
        
        for query in queries:
            print(f"\n💬 Usuario: {query}")
            print("-" * 40)
            
            result = wrapper.search(query)
            
            if result["success"]:
                print(f"✅ Éxito: {result['response'][:200]}...")
                print(f"📚 Fuentes: {len(result['sources'])}")
                print(f"🕐 Timestamp: {result['timestamp']}")
            else:
                print(f"❌ Error: {result['response']}")
            
            print("=" * 60)
            
    except Exception as e:
        print(f"❌ Error en integración: {e}")

if __name__ == "__main__":
    print("🦜 LANGCHAIN + TU AGENTE BING SEARCH ESPECÍFICO")
    print("Agent ID: asst_jdrtyx2upZlf9tpnkOslKw0a")
    print("=" * 60)
    
    # Verificar configuración
    required_vars = [
        "AZURE_CLIENT_ID", "AZURE_CLIENT_SECRET", "AZURE_TENANT_ID",
        "PROJECT_ENDPOINT", "AGENT_ID"
    ]
    
    missing_vars = [var for var in required_vars if not os.getenv(var)]
    
    if missing_vars:
        print(f"❌ Variables faltantes: {', '.join(missing_vars)}")
        print("\n📝 Configura tu .env:")
        print("AZURE_CLIENT_ID=9cba7d58-03f7-4d4b-be36-d6913947fbd3")
        print("AZURE_CLIENT_SECRET=tu-nuevo-secreto")
        print("AZURE_TENANT_ID=a08665d0-8775-4712-be9f-4e23bf1f6e93")
        print("PROJECT_ENDPOINT=https://amxfoundrywest.services.ai.azure.com/api/projects/chatasistente")
        print("AGENT_ID=asst_jdrtyx2upZlf9tpnkOslKw0a")
    else:
        print("✅ Configuración encontrada")
        print("\n🧪 Opciones:")
        print("python script.py tool      - Test tu herramienta Bing")
        print("python script.py langchain - Test integración LangChain") 
        print("python script.py migrate   - Ejemplo de migración")
        
        import sys
        if len(sys.argv) > 1:
            command = sys.argv[1]
            if command == "tool":
                test_your_bing_tool()
            elif command == "langchain":
                test_langchain_integration()
            elif command == "migrate":
                ejemplo_migracion()
        else:
            print("\n💡 Ejecuta: python script.py [tool|langchain|migrate]")

"""
📚 DOCUMENTACIÓN DE TU INTEGRACIÓN ESPECÍFICA:

1. TU AGENTE ESPECÍFICO:
   - Agent ID: asst_jdrtyx2upZlf9tpnkOslKw0a
   - Endpoint: https://amxfoundrywest.services.ai.azure.com/api/projects/chatasistente
   - Service Principal: 9cba7d58-03f7-4d4b-be36-d6913947fbd3

2. MIGRACIÓN RÁPIDA:
   # Antes
   def buscar(query):
       # Tu código actual con threads/runs
   
   # Después  
   wrapper = YourBingSearchWrapper()
   result = wrapper.search(query)

3. CARACTERÍSTICAS:
   ✅ Usa EXACTAMENTE tu agente existente
   ✅ Mantiene tu lógica de threads/runs
   ✅ Integra citaciones automáticamente  
   ✅ Compatible con LangChain
   ✅ Wrapper simple para migración rápida

4. TESTING:
   python script.py tool      # Test solo tu agente
   python script.py langchain # Test con LangChain
   python script.py migrate   # Ejemplo de migración

5. USO EN PRODUCCIÓN:
   from your_integration import YourBingSearchWrapper
   
   wrapper = YourBingSearchWrapper()
   result = wrapper.search("tu consulta")
   print(result["response"])
"""
