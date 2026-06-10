# Sección 3: Implementar soluciones de Computer Vision (10-15%)

---

## 3.1 Generación de imágenes y videos

### Conceptos clave

- **DALL-E 3**: modelo de generación de imágenes a partir de texto, disponible en Azure AI Foundry. Soporta control de estilo (vivid / natural), calidad (standard / hd) y tamaño (1024×1024, 1792×1024, 1024×1792).
- **gpt-image-1**: modelo de generación y edición de imágenes más reciente de OpenAI, accesible desde Azure AI Foundry. Admite inpainting nativo mediante máscaras.
- **Inpainting**: modificación selectiva de regiones de una imagen usando una máscara (imagen PNG con áreas transparentes que indican qué regenerar).
- **Generación de video**: modelos como Sora generan clips de video a partir de texto o imágenes de referencia. Azure AI Foundry actúa como punto de orquestación.
- **Controles de generación**:
  - `size`: resolución de la imagen de salida.
  - `quality`: `standard` vs `hd` (mayor detalle, mayor coste).
  - `style`: `vivid` (hiperrealista) vs `natural` (más fotorrealista suave).
  - `n`: número de variantes a generar.
  - `response_format`: `url` (enlace temporal) o `b64_json` (base64 inline).

### Código Python de referencia

```python
# Generación de imagen con DALL-E 3
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://<resource>.openai.azure.com/",
    api_key="<api-key>",
    api_version="2024-02-01",
)

response = client.images.generate(
    model="dall-e-3",          # nombre del deployment en Azure
    prompt="Un paisaje andino al atardecer, estilo acuarela",
    size="1792x1024",
    quality="hd",
    style="natural",
    n=1,
)

image_url = response.data[0].url
print(image_url)
```

```python
# Inpainting / edición con gpt-image-1 (imagen + máscara)
import base64, pathlib

with open("original.png", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

with open("mask.png", "rb") as f:
    mask_b64 = base64.b64encode(f.read()).decode()

response = client.images.edit(
    model="gpt-image-1",
    image=open("original.png", "rb"),
    mask=open("mask.png", "rb"),   # áreas transparentes = zonas a rellenar
    prompt="Reemplaza el fondo con un cielo estrellado",
    size="1024x1024",
)
print(response.data[0].url)
```

### Puntos clave para el examen

| Aspecto | Detalle |
|---|---|
| Modelo para generación de imágenes en Azure | DALL-E 3 o gpt-image-1 |
| Parámetro de calidad superior | `quality="hd"` |
| Inpainting requiere | imagen original + máscara PNG (zonas transparentes) |
| `response_format="b64_json"` | devuelve imagen como string base64 (sin URL expirable) |
| Generación de video | modelos de video (ej. Sora) vía Azure AI Foundry |
| Edición controlada por prompts | se especifica qué modificar en el `prompt` |

---

## 3.2 Flujos de comprensión multimodal

### Conceptos clave

- **GPT-4o vision**: acepta imágenes (URL pública o base64) junto con texto en el array `messages`. Permite analizar, describir y razonar sobre contenido visual.
- **Image captioning**: generar descripciones cortas (una frase) o detalladas (párrafo) de una imagen pasada como entrada al modelo.
- **Visual Question Answering (VQA)**: el usuario adjunta una imagen y hace una pregunta; el modelo responde basándose en el contenido visual.
- **Alt text / Accesibilidad**: GPT-4o puede generar texto alternativo descriptivo para imágenes (importante para WCAG y cumplimiento de accesibilidad).
- **Azure AI Video Indexer**: servicio que extrae automáticamente transcripciones, caras, objetos, escenas, emociones, marcas y palabras clave de vídeos. Genera índice consultable.
- **Azure Content Understanding** (nuevo en Foundry): define *analizadores* que extraen campos estructurados o markdown de imágenes y documentos usando modelos multimodales. Útil para pipelines RAG.
- **Análisis de objetos/regiones**: GPT-4o puede identificar objetos, leer texto (OCR visual) y razonar sobre regiones específicas de una imagen.

### Código Python de referencia

```python
# Análisis de imagen con GPT-4o (URL pública)
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Describe esta imagen en detalle y genera un alt text accesible."},
                {
                    "type": "image_url",
                    "image_url": {"url": "https://ejemplo.com/imagen.jpg", "detail": "high"},
                },
            ],
        }
    ],
    max_tokens=500,
)
print(response.choices[0].message.content)
```

```python
# Imagen como base64 (archivo local)
import base64

with open("foto.jpg", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "¿Qué objetos identificas en la imagen?"},
                {
                    "type": "image_url",
                    "image_url": {"url": f"data:image/jpeg;base64,{img_b64}"},
                },
            ],
        }
    ],
)
print(response.choices[0].message.content)
```

```python
# Azure AI Video Indexer — obtener insights de un video
import requests

ACCOUNT_ID = "<account-id>"
LOCATION   = "trial"
API_KEY    = "<api-key>"
VIDEO_ID   = "<video-id>"

url = (
    f"https://api.videoindexer.ai/{LOCATION}/Accounts/{ACCOUNT_ID}"
    f"/Videos/{VIDEO_ID}/Index"
)
headers = {"Ocp-Apim-Subscription-Key": API_KEY}
insights = requests.get(url, headers=headers).json()

# Extraer transcripción
for block in insights.get("videos", [{}])[0].get("insights", {}).get("transcript", []):
    print(block["text"])
```

### Puntos clave para el examen

| Aspecto | Detalle |
|---|---|
| Parámetro `detail` en image_url | `low` (rápido, menos tokens) vs `high` (mayor precisión) |
| Formato base64 en mensaje | `data:image/jpeg;base64,<datos>` |
| Video Indexer extrae | transcripciones, caras, objetos, escenas, emociones, marcas |
| Content Understanding | analizadores configurables → salida JSON o markdown |
| VQA con GPT-4o | imagen + pregunta en el mismo mensaje multipart |
| Alt text | generado con instrucción explícita; útil para WCAG |

---

## 3.3 IA Responsable para contenido multimodal

### Conceptos clave

- **Content Safety — análisis de imágenes**: la API `ImageAnalysis` de Azure AI Content Safety detecta contenido violento, sexual, de odio o autolesivo en imágenes, devolviendo niveles de severidad (0–6).
- **Filtros de contenido en DALL-E / gpt-image-1**: Azure aplica filtros de contenido automáticos en los modelos de generación; los prompts que violan las políticas son rechazados.
- **Prompt injection indirecta (visual)**: texto malicioso incrustado en una imagen (p. ej. instrucciones ocultas en una captura) puede manipular al modelo multimodal. Mitigación: analizar el texto extraído de imágenes con Content Safety antes de procesarlo.
- **Marcas de agua (watermarking)**: Azure puede añadir marcas de agua imperceptibles (C2PA / provenance) a imágenes generadas para indicar origen IA.
- **Símbolos prohibidos / brand compliance**: los filtros visuales bloquean la generación de marcas registradas, logos reales, figuras públicas o contenido con derechos de autor.
- **Moderación en flujos de análisis**: al usar GPT-4o para analizar imágenes de usuarios, se recomienda pasar la imagen por Content Safety antes de enviarla al modelo.

### Código Python de referencia

```python
# Azure AI Content Safety — moderación de imagen
from azure.ai.contentsafety import ContentSafetyClient
from azure.ai.contentsafety.models import AnalyzeImageOptions, ImageData
from azure.core.credentials import AzureKeyCredential
import base64

client_cs = ContentSafetyClient(
    endpoint="https://<resource>.cognitiveservices.azure.com/",
    credential=AzureKeyCredential("<api-key>"),
)

with open("imagen_usuario.jpg", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

request = AnalyzeImageOptions(image=ImageData(content=img_b64))
result  = client_cs.analyze_image(request)

for cat in result.categories_analysis:
    print(f"{cat.category}: severidad {cat.severity}")
    if cat.severity >= 4:
        print("  ⚠ Contenido bloqueado")
```

```python
# Detección de prompt injection en texto extraído de imagen
# 1. Extraer texto de la imagen con GPT-4o
extracted_text = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role":"user","content":[
        {"type":"text","text":"Extrae todo el texto visible en la imagen."},
        {"type":"image_url","image_url":{"url": f"data:image/png;base64,{img_b64}"}},
    ]}],
).choices[0].message.content

# 2. Analizar el texto con Content Safety
from azure.ai.contentsafety.models import AnalyzeTextOptions
text_result = client_cs.analyze_text(AnalyzeTextOptions(text=extracted_text))
for cat in text_result.categories_analysis:
    if cat.severity >= 2:
        print(f"Posible injection detectada: {cat.category} ({cat.severity})")
```

### Puntos clave para el examen

| Aspecto | Detalle |
|---|---|
| API de moderación de imágenes | Azure AI Content Safety → `analyze_image` |
| Severidad de Content Safety | 0 = seguro … 6 = muy grave; umbral típico de bloqueo ≥ 4 |
| Prompt injection visual | texto oculto en imagen → extraer → moderar antes de procesar |
| Marcas de agua en imágenes generadas | C2PA / provenance watermarking (Azure) |
| Filtros automáticos en DALL-E | activos por defecto; no se pueden deshabilitar en Azure |
| Política de figuras públicas | DALL-E rechaza generar imágenes realistas de personas reales |

---

## Recursos

- [Azure OpenAI Images — Documentación oficial](https://learn.microsoft.com/azure/ai-services/openai/how-to/dall-e)
- [GPT-4o Vision — Image inputs](https://learn.microsoft.com/azure/ai-services/openai/how-to/gpt-with-vision)
- [Azure AI Video Indexer](https://learn.microsoft.com/azure/azure-video-indexer/video-indexer-overview)
- [Azure AI Content Understanding](https://learn.microsoft.com/azure/ai-services/content-understanding/overview)
- [Azure AI Content Safety — Image moderation](https://learn.microsoft.com/azure/ai-services/content-safety/quickstart-image)
- [Responsible AI — Content filters](https://learn.microsoft.com/azure/ai-services/openai/concepts/content-filter)
