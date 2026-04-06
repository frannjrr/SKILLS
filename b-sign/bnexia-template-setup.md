---
name: bnexia-template-setup
description: Configura la infraestructura de plantillas para nuevos clientes o proyectos. Se activa cuando se recibe un PDF nuevo o se requiere duplicar una base legal existente.
---

# Overview
Esta skill permite la creación, clonación y validación de plantillas legales en el sistema BNEXIA.

# Triggers
- Recepción de un archivo PDF para digitalizar.
- Solicitud de duplicar una plantilla existente para un nuevo cliente.
- Necesidad de verificar si una plantilla recién creada es correcta.

# Instructions
1. **Para nuevos documentos:** Usa `crear_plantilla_pdf`. Debes solicitar al usuario el nombre deseado y convertir el archivo a base64.
2. **Para estandarización:** Si el cliente pide "lo mismo que el cliente X", usa `clonar_plantilla` usando el ID de origen.
3. **Validación Obligatoria:** Tras cualquier creación o clonación, usa `obtener_plantilla_detalle` para confirmar que los campos y documentos se cargaron correctamente. No des por finalizado el proceso sin este paso.

# Best Practices
- Siempre verifica el `template_id` antes de proceder a una clonación.
- Si la creación de PDF falla, pide al usuario que confirme el formato del array JSON.

# Output Format
Reporte de éxito: "Plantilla [Nombre] configurada con ID: [ID]. Estructura verificada."
