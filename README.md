# Color Palette Analyzer

Analiza los colores principales de una imagen usando Azure AI Foundry (GPT-4o-mini con visión),
genera una paleta visual con QuickChart y convierte el análisis a audio con ElevenLabs.
El resultado se entrega como un `.zip` con tres archivos: imagen de paleta, texto de análisis y audio.

---

## Origen: workflow de n8n

Este notebook es la traducción directa de un workflow de n8n compuesto por 9 nodos.
La siguiente tabla muestra la correspondencia entre cada nodo y su equivalente en Python.

| Nodo n8n | Tipo | Equivalente en Python |
|---|---|---|
| Telegram Trigger | Trigger | Parámetro `IMAGE_PATH` en la celda de ejecución |
| Azure OpenAI Chat Model | LLM | `FoundryChatClient` + `AzureOpenAI` (visión multimodal) |
| Simple Memory | Memoria | `agent.create_session()` — eliminado en la versión final |
| AI Agent | Agente | `color_agent = Agent(...)` + `analyze_image()` |
| Code in JavaScript | Transformación | `build_chart_url()` |
| HTTP Request | Acción | `download_chart_image()` |
| Convert text to speech | Acción | `elevenlabs_tts()` |
| Send a photo / text / audio | Salida | `build_zip()` + `files.download()` |

---

## Diferencias principales respecto al workflow de n8n

**Trigger de entrada**

En n8n el flujo arranca cuando el usuario envía una imagen por Telegram.
En el notebook, la imagen se lee directamente desde Google Drive especificando su ruta.

```python
# Notebook
IMAGE_PATH = "/content/drive/MyDrive/fotoanalisis.png"
```

**Autenticación con Azure**

n8n gestiona las credenciales internamente a través de su panel de conexiones.
En el notebook, `FoundryChatClient` requiere un objeto con la interfaz OAuth de Azure,
por lo que se usa un wrapper mínimo que adapta la API key a esa interfaz.

```python
# n8n: credencial configurada en el panel, sin código visible

# Notebook: wrapper necesario porque FoundryChatClient espera .get_token()
class ApiKeyCredential:
    def get_token(self, *scopes, **kwargs) -> AccessToken:
        return AccessToken(self._api_key, int(time.time()) + 3600)
```

**Envío de la imagen al modelo**

En n8n el nodo AI Agent tiene la opción `passthroughBinaryImages: true`,
que reenvía automáticamente el binario de la imagen al modelo.
En Python no existe ese mecanismo, por lo que la imagen se codifica en base64
y se incluye directamente en el array `content` del mensaje como `image_url`.

```python
# n8n: passthroughBinaryImages: true  (un toggle en la UI)

# Notebook: formato multimodal explícito
"content": [
    {"type": "image_url", "image_url": {"url": f"data:{media_type};base64,{b64_data}"}},
    {"type": "text", "text": "Analiza esta imagen y devuelve el JSON."}
]
```

**Generación de la paleta visual (nodo Code in JavaScript)**

Este es el nodo con más lógica en el workflow de n8n. Construye la configuración
de un gráfico Chart.js para QuickChart, inyecta una función JavaScript como string
(porque `json.dumps` no puede serializar funciones) y genera la URL final.
La traducción a Python es prácticamente línea por línea.

```javascript
// n8n (JavaScript)
const configStr = JSON.stringify(chartConfig);
configStr = configStr.replace('"INYECCION_DE_CODIGO"', 'function(value, context) { ... }');
const chartUrl = "https://quickchart.io/chart?..." + encodeURIComponent(configStr);
```

```python
# Python
config_str = json.dumps(chart_config)
config_str = config_str.replace('"INYECCION_DE_CODIGO"', 'function(value, context) { ... }')
return "https://quickchart.io/chart?..." + urllib.parse.quote(config_str)
```

**Memoria de sesión**

En n8n el nodo `memoryBufferWindow` mantiene el historial de conversación
usando el `chat.id` de Telegram como clave de sesión.
En el notebook esta funcionalidad se eliminó porque no hay canal de mensajería:
cada ejecución analiza una imagen de forma independiente.

**Salida**

n8n envía los resultados de vuelta al usuario por Telegram en tres mensajes separados:
foto, texto y audio. El notebook los empaqueta en un único `.zip` que se descarga
automáticamente al navegador desde Colab.

```
n8n:      Send photo → Send text → Send audio  (3 mensajes en Telegram)
Notebook: build_zip() → files.download()       (1 archivo .zip)
```

---

## Estructura del output

```
resultado_paleta.zip
├── palette_chart.png    # paleta visual de colores con códigos TCX
├── analisis.txt         # análisis del director de arte y propuestas alternativas
└── analisis_audio.mp3   # narración del análisis (voz Charlie, ElevenLabs)
```

---

## Requisitos

**APIs necesarias**

- Azure AI Foundry con un deployment de `gpt-4o-mini` con capacidades de visión
- ElevenLabs (opcional — si `generate_audio: False` se omite el MP3)

**Librerías**

```bash
pip install agent-framework-core
pip install agent-framework-foundry
pip install elevenlabs requests pillow
```

**Entorno**

El notebook está diseñado para ejecutarse en Google Colab.
La imagen de entrada debe estar en Google Drive.

---

## Uso

1. Abre el notebook en Colab
2. Rellena las credenciales en la celda 2
3. Ejecuta todas las celdas en orden
4. En la celda de ejecución, ajusta `IMAGE_PATH` con la ruta de tu imagen en Drive
5. El ZIP se descarga automáticamente al terminar
