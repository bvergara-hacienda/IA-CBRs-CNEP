# Piloto de Consolidación de Inscripciones de DAA mediante IA

**Comisión Nacional de Evaluación y Productividad (CNEP)**  
Equipo: Benjamín Vergara, Constanza Rosales, Daniel Stipo y José Contreras

---

## Descripción

Este repositorio contiene el código del piloto que automatiza la consolidación de inscripciones de **Derechos de Aprovechamiento de Aguas (DAA)** provenientes de los Conservadores de Bienes Raíces (CBR), utilizando Inteligencia Artificial.

El piloto demuestra que es posible extraer y estructurar la información jurídica de estos documentos con una **precisión promedio del 95%** sobre 34 variables, a un **costo ~10 veces menor** que el proceso tradicional (≈$267 vs. $2.500 por inscripción).

---

## Flujo de trabajo

El proceso se divide en tres etapas secuenciales:

```
PDFs de inscripciones CBR
        │
        ▼
1. Document Intelligence (OCR cognitivo con Gemini 2.5 Pro)
        │  → Texto etiquetado por secciones
        ▼
2. Extracción estructurada (GPT-5-mini, 7 fases)
        │  → JSON con 47 variables por inscripción
        ▼
3. Validación asistida por IA (GPT-5-mini, 7 fases)
           → Excel con status OK/PARCIAL/WRONG por campo
           → Reporte de precisión por variable
```

---

## Estructura del repositorio

| Notebook | Descripción |
|---|---|
| `Document_Intelligence.ipynb` | **Etapa 1 - OCR.** Convierte cada página del PDF en imagen y usa `gemini-2.5-pro` para extraer el texto clasificado por etiquetas (`[Cuerpo central:]`, `[Nota al margen izquierdo:]`, etc.). Guarda los resultados en archivos `.txt` y un Excel consolidado. |
| `Instrucciones.ipynb` | **Prompts de extracción.** Define las instrucciones detalladas, ejemplos few-shot y esquemas de clasificación para cada una de las 7 fases de extracción. Es importado por `Extracción.ipynb` y `Validación.ipynb` mediante `%run`. |
| `Modelo_Datos.ipynb` | **Esquemas Pydantic.** Define los modelos de datos estructurados (`Fase1Registro`, `Fase2Registro`, etc.) que garantizan que las respuestas del modelo siguen el formato correcto. También es importado vía `%run`. |
| `Ejecución.ipynb` | **Funciones de llamada a la API.** Contiene `procesar_fase_gpt()`, que construye el prompt, llama a la API de OpenAI con structured outputs y retorna el JSON validado por Pydantic. Importado por `Extracción.ipynb`. |
| `Extracción.ipynb` | **Etapa 2 - Pipeline principal.** Orquesta las 7 fases de extracción sobre cada inscripción, acumulando el contexto entre fases. Incluye lógica de dependencias condicionales (ej. la Fase 5 de montos se omite en herencias) y reintentos automáticos. Guarda resultados en Excel. |
| `Validación.ipynb` | **Etapa 3 - Auditoría.** Para cada campo extraído, compara la respuesta con el texto fuente usando `gpt-5-mini` como auditor. Clasifica cada campo como `OK`, `PARCIAL` o `WRONG`, genera un comentario justificativo y calcula la tasa de precisión por variable. |

---

## Las 7 fases de extracción

| Fase | Contenido extraído |
|---|---|
| 1 - Datos basales | Tipo de transacción, fojas, número CBR, fecha |
| 2 - Caracterización del derecho | Tipo (consuntivo/no consuntivo), naturaleza del agua, régimen de ejercicio, uso |
| 3 - Partes | Emisores y receptores: nombre, RUT, representante legal, domicilio |
| 4 - Caudal | Caudal promedio, distribución mensual, tipo de fuente, cuenca |
| 5 - Monto de transacción | Valor total, valor del agua, moneda (omitida en actos gratuitos) |
| 6 - Títulos anteriores | Cadena registral histórica: fojas, número, CBR y año de inscripciones anteriores |
| 7 - Puntos geográficos | Coordenadas UTM o geográficas, datum, huso de captación y restitución |

---

## Requisitos

### APIs necesarias

- **Google Gemini API** (`GOOGLE_API_KEY`) — para Document Intelligence
- **OpenAI API** (`OPENAI_API_KEY`) — para extracción y validación

Las claves deben estar en un archivo `.env` en la carpeta de códigos (no incluido en el repositorio).

### Librerías principales

```
google-generativeai
openai
langchain
langchain-openai
pydantic
pandas
openpyxl
PyMuPDF (fitz)
python-dotenv
Pillow
```

---

## Configuración de rutas

Antes de ejecutar, ajusta las rutas en cada notebook:

```python
# En Document_Intelligence.ipynb
path = os.path.join(os.environ['USERPROFILE'], "TU_RUTA")
region = "Metropolitana"
cbr = "SANTIAGO"

# En Extracción.ipynb y Validación.ipynb
path = os.path.join(os.environ['USERPROFILE'], "TU_RUTA")
```

### Estructura de carpetas esperada

```
TU_RUTA/
├── Códigos/
│   └── .env                         ← API keys
├── Archivos Auxiliares/
│   └── Listados Auxiliares.xlsx     ← Catálogo de CBRs, tipos de transacción y variables
├── {Region}/{CBR}/
│   ├── *.pdf                        ← Inscripciones de entrada
│   └── Resultados/                  ← Textos OCR (.txt) y Excel
├── Resultados/
│   ├── OCR/Resultados_OCR.xlsx
│   ├── Extracción/resultados.xlsx
│   └── Validación/Resultados_Validacion.xlsx
```

---

## Orden de ejecución

1. **`Document_Intelligence.ipynb`** — procesa los PDFs y genera los textos OCR
2. **`Extracción.ipynb`** — extrae las 47 variables por inscripción (llama internamente a `Instrucciones`, `Modelo_Datos` y `Ejecución`)
3. **`Validación.ipynb`** — audita los resultados y calcula la precisión (llama internamente a `Instrucciones` y `Modelo_Datos`)

---

## Resultados del piloto

Sobre 517 inscripciones de los CBR de Santiago, Puente Alto, Melipilla y La Ligua:

| Métrica | IA | Proceso tradicional |
|---|---|---|
| Costo por inscripción | ~$267 CLP | ~$2.500 CLP |
| Tiempo total (12.000 inscripciones, 3 API keys) | ~26 días | ~88 días |
| Precisión promedio global | 95% | n.d. |

Variables con mayor precisión: `tipo_transaccion`, `fojas_cbr_actual`, `huso`, `unidad_transaccion` (100%).  
Variable con menor precisión: `cuenca` (44%) — requiere análisis adicional.

---

## Referencia

Vergara, B., Rosales, C., Stipo, D. y Contreras, J. (2026). *Piloto de Consolidación de Inscripciones de Derechos de Aprovechamiento de Aguas en los CBR*. Comisión Nacional de Evaluación y Productividad (CNEP).
