# Planka API — Referencia Completa REST

**Base URL:** `https://<tu-instancia>/api`  
**Auth:** `Authorization: Bearer <TOKEN>`  
**Content-Type:** `application/json` (salvo adjuntos: `multipart/form-data`)

---

## Tabla de Contenidos

1. [Autenticación](#1-autenticación)
2. [Proyectos](#2-proyectos)
3. [Boards](#3-boards)
4. [Listas](#4-listas)
5. [Cards](#5-cards)
6. [Etiquetas (Labels)](#6-etiquetas-labels)
7. [Membresías de Card](#7-membresías-de-card)
8. [Adjuntos](#8-adjuntos)
9. [Comentarios y Acciones](#9-comentarios-y-acciones)
10. [Tareas (Tasks)](#10-tareas-tasks)
11. [Usuarios](#11-usuarios)
12. [Membresías de Board](#12-membresías-de-board)
13. [Gestores de Proyecto](#13-gestores-de-proyecto)
14. [Notificaciones](#14-notificaciones)
15. [Webhooks](#15-webhooks)
16. [Campos Personalizados](#16-campos-personalizados)

---

## 1. Autenticación

### POST /api/access-tokens — Iniciar Sesión

Obtiene un token de sesión.

```http
POST /api/access-tokens
Content-Type: application/json

{
  "emailOrUsername": "admin@ejemplo.com",
  "password": "mi_contraseña"
}
```

**Respuesta exitosa (200):**
```json
{
  "item": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### DELETE /api/access-tokens/me — Cerrar Sesión

```http
DELETE /api/access-tokens/me
Authorization: Bearer <TOKEN>
```

---

## 2. Proyectos

### GET /api/projects — Listar Proyectos

```http
GET /api/projects
Authorization: Bearer <TOKEN>
```

**Respuesta:**
```json
{
  "items": [
    {
      "id": "proj_abc123",
      "name": "Cliente XYZ",
      "type": "project",
      "createdAt": "2024-01-01T00:00:00.000Z",
      "updatedAt": "2024-01-01T00:00:00.000Z"
    }
  ]
}
```

### POST /api/projects — Crear Proyecto

```http
POST /api/projects
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "name": "Nombre del Proyecto",
  "type": "project",
  "description": "Descripción opcional"
}
```

Campos:
- `name` (string, requerido): Nombre del proyecto
- `type` (string, requerido): Siempre `"project"`
- `description` (string, opcional): Descripción

**Respuesta (201):**
```json
{
  "item": {
    "id": "proj_nuevoid",
    "name": "Nombre del Proyecto",
    "type": "project"
  }
}
```

### GET /api/projects/:projectId — Ver Proyecto

```http
GET /api/projects/:projectId
```

### PATCH /api/projects/:projectId — Actualizar Proyecto

```http
PATCH /api/projects/:projectId
Content-Type: application/json

{
  "name": "Nuevo Nombre",
  "description": "Nueva descripción"
}
```

### DELETE /api/projects/:projectId — Eliminar Proyecto

```http
DELETE /api/projects/:projectId
```

> ⚠️ Elimina todos los boards, listas, cards y datos asociados. Irreversible.

---

## 3. Boards

### GET /api/boards/:boardId — Ver Board Completo

Retorna el board con todas sus listas, cards, etiquetas y miembros.

```http
GET /api/boards/:boardId
Authorization: Bearer <TOKEN>
```

**Respuesta:** Incluye `item` (el board) e `included` con:
- `included.lists[]` — todas las listas del board
- `included.cards[]` — todos los cards
- `included.labels[]` — todas las etiquetas
- `included.users[]` — usuarios con acceso
- `included.boardMemberships[]` — membresías

```json
{
  "item": {
    "id": "brd_xxx",
    "name": "Legal",
    "position": 65535,
    "projectId": "proj_xxx"
  },
  "included": {
    "lists": [...],
    "cards": [...],
    "labels": [...],
    "users": [...],
    "boardMemberships": [...]
  }
}
```

### POST /api/projects/:projectId/boards — Crear Board

```http
POST /api/projects/:projectId/boards
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "name": "Nombre del Board",
  "position": 65535
}
```

Campos:
- `name` (string, requerido)
- `position` (number, requerido): Posición para ordenación. Usar 65535, 131070, 196605...

**Respuesta (201):**
```json
{
  "item": {
    "id": "brd_nuevoid",
    "name": "Legal",
    "position": 65535,
    "projectId": "proj_xxx"
  }
}
```

### PATCH /api/boards/:boardId — Actualizar Board

```http
PATCH /api/boards/:boardId
Content-Type: application/json

{
  "name": "Nuevo Nombre",
  "position": 131070
}
```

### DELETE /api/boards/:boardId — Eliminar Board

```http
DELETE /api/boards/:boardId
```

---

## 4. Listas

### POST /api/boards/:boardId/lists — Crear Lista

```http
POST /api/boards/:boardId/lists
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "name": "Pendiente",
  "position": 65535
}
```

Campos:
- `name` (string, requerido)
- `position` (number, requerido)

**Respuesta (201):**
```json
{
  "item": {
    "id": "lst_nuevoid",
    "name": "Pendiente",
    "position": 65535,
    "boardId": "brd_xxx"
  }
}
```

### PATCH /api/lists/:listId — Actualizar Lista

```http
PATCH /api/lists/:listId
Content-Type: application/json

{
  "name": "Nuevo Nombre",
  "position": 131070
}
```

### DELETE /api/lists/:listId — Eliminar Lista

```http
DELETE /api/lists/:listId
```

> ⚠️ Elimina todos los cards dentro de la lista.

### POST /api/lists/:listId/sort — Ordenar Cards de una Lista

```http
POST /api/lists/:listId/sort
Content-Type: application/json

{ "sortBy": "name" }
```

Valores de `sortBy`: `"name"`, `"dueDate"`, `"createdAt"`

---

## 5. Cards

### POST /api/lists/:listId/cards — Crear Card

```http
POST /api/lists/:listId/cards
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "boardId": "<boardId>",
  "listId": "<listId>",
  "name": "Título de la Card",
  "description": "Descripción detallada en **Markdown**",
  "position": 65535,
  "dueDate": "2024-01-15T23:59:59.000Z",
  "isCompleted": false
}
```

Campos:
- `boardId` (string, requerido): ID del board padre
- `listId` (string, requerido): ID de la lista (también va en la URL)
- `name` (string, requerido): Título de la card
- `description` (string, opcional): Soporta Markdown
- `position` (number, requerido): Posición de ordenación
- `dueDate` (string ISO 8601, opcional): Fecha límite
- `isCompleted` (boolean, opcional): Estado de completado

**Respuesta (201):**
```json
{
  "item": {
    "id": "crd_nuevoid",
    "name": "Título",
    "description": "...",
    "position": 65535,
    "dueDate": null,
    "isCompleted": false,
    "boardId": "brd_xxx",
    "listId": "lst_xxx",
    "createdAt": "2024-01-01T00:00:00.000Z"
  }
}
```

### GET /api/cards/:cardId — Ver Card

```http
GET /api/cards/:cardId
```

Incluye en `included`: labels, members, tasks, attachments, actions.

### PATCH /api/cards/:cardId — Actualizar Card

```http
PATCH /api/cards/:cardId
Content-Type: application/json

{
  "name": "Nuevo título",
  "description": "Nueva descripción",
  "listId": "<nuevo_listId>",
  "position": 65535,
  "dueDate": "2024-01-20T23:59:59.000Z",
  "isCompleted": true
}
```

Todos los campos son opcionales. Envía solo los que quieres modificar.

Para **mover una card** a otra lista: incluir `listId` (y opcionalmente `boardId` si es otro board).

### DELETE /api/cards/:cardId — Eliminar Card

```http
DELETE /api/cards/:cardId
```

### POST /api/cards/:cardId/duplicate — Duplicar Card

```http
POST /api/cards/:cardId/duplicate
Content-Type: application/json

{
  "position": 131070
}
```

---

## 6. Etiquetas (Labels)

### POST /api/boards/:boardId/labels — Crear Etiqueta

```http
POST /api/boards/:boardId/labels
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "name": "ALTA",
  "color": "berry-red",
  "position": 65535
}
```

Campos:
- `name` (string, requerido): Nombre visible
- `color` (string, requerido): Ver lista de colores en SKILL.md
- `position` (number, requerido)

**Respuesta (201):**
```json
{
  "item": {
    "id": "lbl_nuevoid",
    "name": "ALTA",
    "color": "berry-red",
    "position": 65535,
    "boardId": "brd_xxx"
  }
}
```

### PATCH /api/labels/:labelId — Actualizar Etiqueta

```http
PATCH /api/labels/:labelId
Content-Type: application/json

{
  "name": "Nuevo nombre",
  "color": "pumpkin-orange"
}
```

### DELETE /api/labels/:labelId — Eliminar Etiqueta

```http
DELETE /api/labels/:labelId
```

### POST /api/cards/:cardId/labels — Asignar Etiqueta a Card

```http
POST /api/cards/:cardId/labels
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "labelId": "<labelId>"
}
```

### DELETE /api/cards/:cardId/labels/:labelId — Quitar Etiqueta de Card

```http
DELETE /api/cards/:cardId/labels/:labelId
```

---

## 7. Membresías de Card

### POST /api/cards/:cardId/memberships — Asignar Usuario a Card

```http
POST /api/cards/:cardId/memberships
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "userId": "<userId>"
}
```

### DELETE /api/cards/:cardId/memberships/:userId — Quitar Usuario de Card

```http
DELETE /api/cards/:cardId/memberships/:userId
```

---

## 8. Adjuntos

### POST /api/cards/:cardId/attachments — Subir Adjunto

```http
POST /api/cards/:cardId/attachments
Authorization: Bearer <TOKEN>
Content-Type: multipart/form-data

file: <archivo binario>
```

> ⚠️ Usar `multipart/form-data`. El campo se llama exactamente `file`.

**Respuesta (201):**
```json
{
  "item": {
    "id": "att_nuevoid",
    "name": "documento.pdf",
    "url": "/attachments/proj_xxx/brd_xxx/documento.pdf",
    "coverUrl": null,
    "creatorUserId": "usr_xxx",
    "cardId": "crd_xxx",
    "createdAt": "2024-01-01T00:00:00.000Z"
  }
}
```

### PATCH /api/attachments/:attachmentId — Renombrar Adjunto

```http
PATCH /api/attachments/:attachmentId
Content-Type: application/json

{
  "name": "nuevo-nombre.pdf",
  "isCover": false
}
```

### DELETE /api/attachments/:attachmentId — Eliminar Adjunto

```http
DELETE /api/attachments/:attachmentId
```

---

## 9. Comentarios y Acciones

Las acciones son el historial de actividad de una card. Los comentarios son un tipo de acción.

### POST /api/cards/:cardId/actions — Añadir Comentario

```http
POST /api/cards/:cardId/actions
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "text": "Texto del comentario (soporta **Markdown**)",
  "type": "commentCard"
}
```

Campos:
- `text` (string, requerido): Contenido del comentario
- `type` (string, requerido): Siempre `"commentCard"` para comentarios

**Respuesta (201):**
```json
{
  "item": {
    "id": "act_nuevoid",
    "type": "commentCard",
    "data": { "text": "Texto del comentario" },
    "cardId": "crd_xxx",
    "userId": "usr_xxx",
    "createdAt": "2024-01-01T00:00:00.000Z"
  }
}
```

### GET /api/cards/:cardId/actions — Listar Acciones de una Card

```http
GET /api/cards/:cardId/actions
```

### PATCH /api/actions/:actionId — Editar Comentario

```http
PATCH /api/actions/:actionId
Content-Type: application/json

{
  "text": "Texto corregido"
}
```

### DELETE /api/actions/:actionId — Eliminar Comentario

```http
DELETE /api/actions/:actionId
```

---

## 10. Tareas (Tasks)

Las tareas son checklists dentro de una card. Se organizan en TaskLists.

### POST /api/cards/:cardId/task-lists — Crear Grupo de Tareas

```http
POST /api/cards/:cardId/task-lists
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "name": "Documentación requerida",
  "position": 65535
}
```

**Respuesta:** `item` con `id` → `taskListId`

### PATCH /api/task-lists/:taskListId — Actualizar TaskList

```http
PATCH /api/task-lists/:taskListId
Content-Type: application/json

{
  "name": "Nuevo nombre",
  "position": 131070
}
```

### DELETE /api/task-lists/:taskListId — Eliminar TaskList

```http
DELETE /api/task-lists/:taskListId
```

### POST /api/task-lists/:taskListId/tasks — Crear Tarea

```http
POST /api/task-lists/:taskListId/tasks
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "name": "Descripción de la tarea",
  "position": 65535,
  "isCompleted": false,
  "assigneeUserId": "<userId>",
  "dueDate": "2024-01-15T23:59:59.000Z"
}
```

Campos opcionales: `assigneeUserId`, `dueDate`, `isCompleted`

### PATCH /api/tasks/:taskId — Actualizar / Completar Tarea

```http
PATCH /api/tasks/:taskId
Content-Type: application/json

{
  "isCompleted": true,
  "name": "Tarea actualizada"
}
```

### DELETE /api/tasks/:taskId — Eliminar Tarea

```http
DELETE /api/tasks/:taskId
```

---

## 11. Usuarios

### GET /api/users — Listar Usuarios

```http
GET /api/users
Authorization: Bearer <TOKEN>
```

Requiere rol `admin` o `projectOwner`.

**Respuesta:**
```json
{
  "items": [
    {
      "id": "usr_abc123",
      "name": "Juan García",
      "username": "jgarcia",
      "email": "juan@ejemplo.com",
      "role": "editor",
      "createdAt": "2024-01-01T00:00:00.000Z"
    }
  ]
}
```

Roles posibles: `"admin"`, `"projectOwner"`, `"editor"`, `"viewer"`

### GET /api/users/:userId — Ver Usuario

```http
GET /api/users/:userId
```

### POST /api/users — Crear Usuario (Admin)

```http
POST /api/users
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "email": "nuevo@ejemplo.com",
  "password": "contraseña_segura",
  "name": "Nombre Completo",
  "username": "usuario",
  "role": "editor",
  "organization": "Empresa S.L.",
  "phone": "+34 600 000 000"
}
```

Campos requeridos: `email`, `password`, `name`, `role`
Roles válidos: `"admin"`, `"projectOwner"`, `"editor"`, `"viewer"`

### PATCH /api/users/:userId — Actualizar Usuario

```http
PATCH /api/users/:userId
Content-Type: application/json

{
  "name": "Nuevo Nombre",
  "email": "nuevo@email.com",
  "role": "projectOwner"
}
```

### DELETE /api/users/:userId — Eliminar Usuario

```http
DELETE /api/users/:userId
```

### GET /api/users/me — Perfil del Usuario Actual

```http
GET /api/users/me
```

### PATCH /api/users/me — Actualizar Perfil Propio

```http
PATCH /api/users/me
Content-Type: application/json

{
  "name": "Mi Nombre",
  "username": "mi_usuario",
  "language": "es-ES"
}
```

### PUT /api/users/:userId/password — Cambiar Contraseña

```http
PUT /api/users/:userId/password
Content-Type: application/json

{
  "currentPassword": "actual",
  "newPassword": "nueva"
}
```

### POST /api/users/:userId/avatar — Subir Avatar

```http
POST /api/users/:userId/avatar
Content-Type: multipart/form-data

file: <imagen>
```

---

## 12. Membresías de Board

### POST /api/boards/:boardId/memberships — Añadir Miembro al Board

```http
POST /api/boards/:boardId/memberships
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "userId": "<userId>",
  "role": "editor"
}
```

Roles: `"editor"`, `"viewer"`

**Respuesta:** `item` con `id` → `boardMembershipId`

### PATCH /api/board-memberships/:boardMembershipId — Cambiar Rol

```http
PATCH /api/board-memberships/:boardMembershipId
Content-Type: application/json

{
  "role": "viewer"
}
```

### DELETE /api/board-memberships/:boardMembershipId — Quitar Miembro del Board

```http
DELETE /api/board-memberships/:boardMembershipId
```

---

## 13. Gestores de Proyecto

### POST /api/projects/:projectId/managers — Añadir Gestor

```http
POST /api/projects/:projectId/managers
Authorization: Bearer <TOKEN>
Content-Type: application/json

{
  "userId": "<userId>"
}
```

**Respuesta:** `item` con `id` → `projectManagerId`

### DELETE /api/project-managers/:projectManagerId — Quitar Gestor

```http
DELETE /api/project-managers/:projectManagerId
```

---

## 14. Notificaciones

### GET /api/notifications — Listar Notificaciones

```http
GET /api/notifications
Authorization: Bearer <TOKEN>
```

### PUT /api/notifications/:notificationId — Marcar como Leída

```http
PUT /api/notifications/:notificationId
Content-Type: application/json

{ "isRead": true }
```

---

## 15. Webhooks

### GET /api/webhooks — Listar Webhooks (Admin)

```http
GET /api/webhooks
Authorization: Bearer <TOKEN>
```

### POST /api/webhooks — Crear Webhook

```http
POST /api/webhooks
Content-Type: application/json

{
  "name": "Mi Webhook",
  "url": "https://mi-servidor.com/webhook",
  "accessToken": "token_secreto",
  "events": ["cardCreated", "cardUpdated", "cardDeleted"]
}
```

Eventos disponibles (selección): `cardCreated`, `cardUpdated`, `cardDeleted`,
`cardMembershipCreated`, `commentCardActionCreated`, `listCreated`, `boardCreated`

### PATCH /api/webhooks/:webhookId — Actualizar Webhook

```http
PATCH /api/webhooks/:webhookId
Content-Type: application/json

{
  "isActive": false
}
```

### DELETE /api/webhooks/:webhookId — Eliminar Webhook

```http
DELETE /api/webhooks/:webhookId
```

---

## 16. Campos Personalizados

### POST /api/boards/:boardId/custom-field-groups — Crear Grupo de Campos

```http
POST /api/boards/:boardId/custom-field-groups
Content-Type: application/json

{
  "name": "Datos del Cliente",
  "position": 65535
}
```

### POST /api/custom-field-groups/:groupId/custom-fields — Crear Campo

```http
POST /api/custom-field-groups/:groupId/custom-fields
Content-Type: application/json

{
  "name": "NIF",
  "type": "text",
  "position": 65535,
  "showOnFrontOfCard": true
}
```

Tipos de campo (`type`): `"text"`, `"number"`, `"date"`, `"dropdown"`

### PUT /api/cards/:cardId/custom-field-values/:customFieldId — Establecer Valor

```http
PUT /api/cards/:cardId/custom-field-values/:customFieldId
Content-Type: application/json

{
  "value": "12345678A"
}
```

---

## Notas de Implementación

### Posicionamiento
Planka usa un sistema de posición flotante (LexoRank simplificado). Para insertar elementos:
- **Al final de la lista:** usa `(max_position_existente) + 65535`
- **Al inicio:** usa `(min_position_existente) / 2`
- **En medio:** usa `(pos_anterior + pos_siguiente) / 2`
- **Primera vez:** usar `65535`

### Paginación
La mayoría de endpoints de listado no pagina actualmente, pero la estructura de respuesta
usa `items[]` (plural) para listas e `item` (singular) para recursos individuales.

### Formato de Fechas
Siempre ISO 8601 en UTC: `"2024-01-15T23:59:59.000Z"`

### Markdown en Descripciones
Los campos `description` (card) y `text` (comentarios) soportan Markdown completo:
`**negrita**`, `*cursiva*`, `## títulos`, `- listas`, `[enlace](url)`, `` `código` ``

### IDs
Los IDs en Planka son strings de 32 caracteres hexadecimales (UUID v4 sin guiones).
Ejemplo: `"6572da3f0a9f123456789abc12345678"`
