---
name: save-session
description: Guarda un resumen de la sesión actual en Documents/sessions, organizado por proyecto e indexado para consultas posteriores. Usar cuando el usuario pida save-session, guardar sesión, guardar esta conversación, registrar lo trabajado o buscar una sesión guardada.
---

# Save Session

Guarda un registro Markdown compacto y útil para retomar el trabajo en `~/Documents/sessions/<proyecto>/`. Resuelve `~` al directorio personal del usuario actual.

Escribe en el idioma de la conversación, con frases claras y concretas. Este skill es autónomo: no necesita otros skills, servicios ni paquetes adicionales.

## Elegir el proyecto y la carpeta

1. Usa el proyecto indicado por el usuario. Por ejemplo, «save-session de clasificar» corresponde al proyecto `clasificar`.
2. Si no lo indica, infiérelo del proyecto trabajado en la conversación. Confirma con las rutas afectadas o la raíz del repositorio cuando haga falta; el directorio de trabajo puede ser simplemente el directorio personal del usuario o una subcarpeta del proyecto. Para worktrees, usa el nombre del proyecto original, no el de la rama o carpeta temporal.
3. Si se trabajó en varios proyectos, usa el principal y menciona los demás en el documento. Pregunta solo si hay varios destinos igualmente plausibles. Para conversaciones sin proyecto, usa `general`.
4. Revisa las carpetas existentes directamente dentro de `~/Documents/sessions/` antes de crear una. Reutiliza la correspondiente al mismo proyecto, aunque difiera en mayúsculas, tildes, espacios, guiones o guiones bajos. No unas proyectos distintos solo porque compartan parte del nombre. Si hay varias carpetas equivalentes y no hay una coincidencia exacta, aclara cuál usar.
5. Si no hay una carpeta correspondiente, crea `~/Documents/sessions/<proyecto>/`, incluyendo la raíz si falta. Para carpetas nuevas, convierte el nombre en un slug: minúsculas, sin tildes, palabras separadas por un guion y solo caracteres `a-z`, `0-9` y `-`. El nombre debe ser un único componente de ruta, sin separadores ni `..`.

Ejemplo de destino: `~/Documents/sessions/clasificar/2026-09-18-ajustes-en-clasificacion.md`. Las siguientes sesiones de Clasificar van en esa misma carpeta.

## Preparar el resumen

Revisa el contexto disponible de la conversación y extrae:

- Un título descriptivo y la fecha local real al guardar. Puedes obtenerla con `date +%F`; si el usuario indica una zona horaria distinta a la del sistema, úsala con `TZ=<zona> date +%F`. No dependas de una variable `$CURRENT_DATE`.
- Un resumen breve, temas principales y resultados concretos.
- Decisiones y sus motivos; archivos o rutas afectados; herramientas y tecnologías relevantes.
- Validaciones ejecutadas y sus resultados, aprendizajes útiles y trabajo pendiente con el próximo paso.
- Entre 5 y 10 palabras clave específicas en minúsculas, usando guiones para términos compuestos. Incluye el proyecto.
- Una frase de hasta 280 caracteres para el índice.

Usa solo hechos presentes en el contexto o comprobados con herramientas. Distingue lo completado de lo propuesto; no inventes cambios, pruebas ni partes de la conversación que ya no estén disponibles. No copies credenciales ni secretos al registro.

## Guardar el documento

Nombre: `YYYY-MM-DD-titulo-en-slug.md`, usando las mismas reglas de slug que para carpetas nuevas. Si ese nombre ya existe dentro del proyecto, prueba `YYYY-MM-DD-titulo-en-slug-2.md`, luego `-3.md`, hasta encontrar uno libre. No sobrescribas sesiones anteriores; otras sesiones con distinta denominación en la misma fecha no necesitan contador.

Guarda el archivo dentro de la carpeta del proyecto elegida. Usa esta plantilla y omite las secciones vacías; traduce los encabezados si la conversación está en otro idioma. En el frontmatter, escribe texto entre comillas y escapa las comillas internas para mantener YAML válido.

```markdown
---
title: "Título de la sesión"
date: YYYY-MM-DD
project: "Nombre del proyecto"
keywords: [proyecto, palabra-clave, tecnologia]
---

## Resumen
Dos a cuatro frases sobre lo trabajado, su resultado y por qué importa.

## Trabajo realizado
- Resultado concreto, con las herramientas o tecnologías relevantes.

## Decisiones
- Decisión y motivo.

## Archivos y rutas
- `ruta/al/archivo` — qué cambió; indica el repositorio o ruta base si es necesario.

## Validación
- Comprobación ejecutada y resultado, o limitación relevante.

## Aprendizajes
- Hallazgo útil para futuras sesiones.

## Pendientes
- Trabajo abierto y próximo paso concreto para retomarlo.
```

## Actualizar el índice global

Lee `~/Documents/sessions/INDEX.md`. Si falta, créalo con esta cabecera:

```markdown
# Índice de sesiones

| Fecha | Proyecto | Sesión | Palabras clave | Resumen |
|-------|----------|--------|----------------|---------|
```

Añade una fila con el enlace relativo desde la raíz de sesiones, incluyendo la carpeta real del proyecto:

```markdown
| YYYY-MM-DD | Clasificar | [Título](clasificar/YYYY-MM-DD-titulo.md) | clasificar, palabra-clave, tecnologia | Frase de hasta 280 caracteres. |
```

Mantén las sesiones más recientes primero y conserva las filas existentes. Evita duplicar filas para una misma ruta. Escapa `|` y saltos de línea dentro de las celdas, y codifica espacios u otros caracteres especiales en los enlaces cuando reutilices una carpeta existente que los contenga.

Comprueba que el documento quedó guardado y que su enlace en el índice apunta al archivo correcto. Si el índice no pudo actualizarse, conserva el documento y comunica qué quedó pendiente.

Confirma brevemente al usuario con un enlace al archivo guardado, el proyecto, la frase de resumen y las palabras clave.

## Buscar o recordar sesiones

Cuando el usuario pida encontrar o recordar trabajo anterior:

1. Consulta `~/Documents/sessions/INDEX.md` y filtra por proyecto, fecha, título, palabras clave o resumen.
2. Para ampliar detalles o si falta el índice, busca con `rg` en los Markdown de la carpeta del proyecto o, si no se conoce, en toda la raíz de sesiones. Por ejemplo: `rg -n -i -F -g '*.md' -- 'autenticación' "$HOME/Documents/sessions"`.
3. Abre los documentos relevantes y responde citando su fecha y un enlace al archivo local.

La búsqueda no crea un nuevo registro. Si no hay sesiones o coincidencias, indícalo sin inventar resultados.
