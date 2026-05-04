---
name: planka-master
description: >
  Skill maestra para gestión completa de Planka (Kanban). Úsala SIEMPRE que el usuario mencione:
  proyectos Planka, boards, listas, cards, etiquetas, expedientes, consultas de clientes, adjuntos,
  asignaciones, configurar cliente nuevo, "b-project", montar estructura Kanban, registrar consulta,
  asignar abogado/responsable, subir documento, o cualquier operación sobre Planka.
  Cubre dos modos: (1) ARQUITECTO — diseña e implementa la infraestructura (proyectos, boards,
  listas, etiquetas) para un cliente nuevo; (2) OPERADOR — gestión diaria de cards (registro de
  consultas, asignación de usuarios, adjuntos, comentarios, tareas). Si hay ambigüedad, pregunta
  al usuario cuál modo necesita antes de proceder.
---

# Planka Master Skill

Skill unificada para el ecosistema b-project sobre Planka. Contiene dos agentes especializados
y la documentación completa de la API REST.

## Cómo Usar Esta Skill

Lee primero esta sección y luego carga **solo** el archivo de referencia relevante al modo solicitado.

### Árbol de Decisión de Modo

```
¿Qué quiere el usuario?
│
├─ Configurar/crear nuevo cliente, proyecto, board, lista, etiqueta
│   → Leer: references/architect.md
│
├─ Registrar consulta, asignar, subir adjunto, comentar, gestionar card existente
│   → Leer: references/operator.md
│
└─ Consultar endpoints, parámetros, cómo llamar a la API
    → Leer: references/api-reference.md
```

> **Regla de oro:** Nunca inventes IDs. Si no tienes el ID en contexto, usa el endpoint de listado
> apropiado o pide el JSON de mapeo al usuario.

---

## Configuración de Entorno

Antes de cualquier llamada, asegúrate de tener estas variables:

```
PLANKA_BASE_URL  → URL base de la instancia (ej: https://planka.miempresa.com)
PLANKA_TOKEN     → Token de sesión obtenido en POST /api/access-tokens
                   (o API Key si el admin la proporcionó)
```

### Autenticación Rápida

```http
POST /api/access-tokens
Content-Type: application/json

{ "emailOrUsername": "admin@ejemplo.com", "password": "contraseña" }
```

Respuesta: `{ "item": "<TOKEN_STRING>" }`

Usar en todas las peticiones como header:
```
Authorization: Bearer <TOKEN_STRING>
```

---

## Jerarquía de Objetos en Planka

```
Planka Instance
└── Project  (projectId)
    └── Board  (boardId)
        ├── List   (listId)
        │    └── Card  (cardId)
        │         ├── Label       (via CardLabel)
        │         ├── Member      (via CardMembership)
        │         ├── Task / TaskList
        │         ├── Comment     (Action)
        │         └── Attachment
        └── Label  (labelId — pertenece al Board)
```

---

## Colores de Etiquetas Válidos

| Nombre | Color visual |
|--------|-------------|
| `berry-red` | Rojo urgente |
| `pumpkin-orange` | Naranja medio |
| `wet-moss` | Verde bajo |
| `lagoon-blue` | Azul |
| `pink-tulip` | Rosa |
| `light-mud` | Marrón |
| `orange-peel` | Naranja brillante |
| `bright-moss` | Verde brillante |
| `antique-blue` | Azul oscuro |
| `dark-granite` | Gris |
| `lagune-blue` | Azul verdoso |
| `sunny-grass` | Amarillo |
| `morning-sky` | Celeste |
| `light-orange` | Naranja pastel |
| `midnight-blue` | Azul noche |
| `tank-green` | Verde militar |
| `gun-metal` | Gris oscuro |
| `wet-moss` | Verde musgo |
| `red-burgundy` | Burdeos |
| `light-concrete` | Gris claro |
| `apricot-red` | Rojo salmón |
| `desert-sand` | Arena |
| `navy-blue` | Azul marino |
| `egg-yellow` | Amarillo huevo |
| `coral-green` | Verde coral |
| `light-cocoa` | Cacao |

**Para b-project usar prioritariamente:** `berry-red` (ALTA), `pumpkin-orange` (MEDIA), `wet-moss` (BAJA)

---

## Referencia Rápida de Endpoints

Ver `references/api-reference.md` para documentación completa.

| Acción | Método | Endpoint |
|--------|--------|----------|
| Login | POST | `/api/access-tokens` |
| Listar proyectos | GET | `/api/projects` |
| Crear proyecto | POST | `/api/projects` |
| Ver board (con listas/cards) | GET | `/api/boards/:boardId` |
| Crear board | POST | `/api/projects/:projectId/boards` |
| Crear lista | POST | `/api/boards/:boardId/lists` |
| Crear card | POST | `/api/lists/:listId/cards` |
| Actualizar card | PATCH | `/api/cards/:cardId` |
| Crear etiqueta | POST | `/api/boards/:boardId/labels` |
| Asignar etiqueta a card | POST | `/api/cards/:cardId/labels` |
| Asignar usuario a card | POST | `/api/cards/:cardId/memberships` |
| Subir adjunto | POST | `/api/cards/:cardId/attachments` |
| Crear comentario | POST | `/api/cards/:cardId/actions` |
| Listar usuarios | GET | `/api/users` |

---

## Error Handling Universal

| Código | Significado | Acción |
|--------|-------------|--------|
| 400 | Bad Request — parámetro inválido | Revisar body/parámetros |
| 401 | No autenticado | Re-autenticar, renovar token |
| 403 | Sin permisos | Verificar rol del usuario |
| 404 | Recurso no encontrado | Verificar que el ID existe |
| 409 | Conflicto (ya existe) | Listar primero, usar el existente |
| 422 | Datos semánticamente inválidos | Revisar tipos y formatos |

---

## Archivos de Referencia

- **`references/architect.md`** — Workflow completo del Agente Arquitecto (setup de cliente nuevo)
- **`references/operator.md`** — Workflow completo del Agente Operador (gestión diaria)
- **`references/api-reference.md`** — Documentación exhaustiva de todos los endpoints REST
