---
name: bnexia-signature-flow
description: Gestiona el envío, seguimiento y entrega de documentos para firma. Se activa cuando un cliente solicita firmar un contrato o quiere saber el estado de su firma.
---

# Overview
Automatiza el flujo de firma electrónica: Selección de plantilla -> Envío -> Seguimiento -> Entrega de PDF.

# Triggers
- "Quiero firmar el contrato de..."
- "¿Cómo va mi firma?"
- "Envíale este documento a [email]"

# Instructions
1. **Selección:** Primero usa `obtener_plantillas` para encontrar el ID correcto según el nombre que mencione el cliente.
2. **Envío:** Usa `enviar_documento`. Requiere obligatoriamente: `template_id`, `email` y `nombre_cliente`.
3. **Seguimiento:** Si el cliente pregunta por el estado, usa `consultar_estado_firma`. 
4. **Cierre:** Si el estado es `completed`, usa `obtener_descarga_pdf` y entrega el enlace de descarga de inmediato.

# Anti-patterns
- NO intentar enviar documentos sin haber consultado primero si la plantilla existe.
- NO pedir el PDF de descarga si el estado sigue siendo `pending`.

# Output Format
- Confirmación de envío: "Documento enviado a [email]. Puede revisar su bandeja de entrada."
- Entrega final: "¡Firma completada! Puede descargar su copia aquí: [URL]"
