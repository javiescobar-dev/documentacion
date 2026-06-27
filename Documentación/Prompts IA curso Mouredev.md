### El arquitecto:
**Fase:** Planificación y Diseño del sistema

Antes de escribir una sola línea de código, necesitas pensar la estructura. Este prompt convierte
una idea vaga en un plan técnico con stack, estructura de archivos, modelo de datos y decisiones
justificadas. Úsalo al inicio de cada proyecto o funcionalidad nueva.

```
Actua como un arquitecto de software senior con mas de 15 anos de experiencia disenando
sistemas escalables.

Necesito que diseñes la arquitectura tecnica completa para el siguiente proyecto:

## Proyecto
[Describe tu proyecto en 2-3 frases. Ej: "Una app web donde los usuarios pueden crear
y compartir listas de lectura personalizadas, con recomendaciones basadas en sus gustos."]

## Requisitos clave
- Usuarios esperados: [numero estimado]
- Tipo de aplicacion: [web / movil / API / CLI / etc.]
- Restricciones tecnicas: [lenguaje o framework obligatorio, si aplica, o "Ninguna, recomienda tu"]
  
## Lo que necesito que entregues
1.Stack tecnologico recomendado: frontend, backend, base de datos, infraestructura.
Justifica cada eleccion en una linea.
2.Estructura de carpetas del proyecto: muestra el arbol de archivos inicial.
3.Modelo de datos: entidades principales, sus campos clave y relaciones.
6.Riesgos tecnicos: identifica 2-3 posibles problemas y como mitigarlos.
4.Diagrama de flujo: describe paso a paso el flujo principal del usuario (de la forma: Paso 1 ->
Paso 2 -> ...).
5.Decisiones de diseño: lista las 3-5 decisiones arquitectónicas más importantes
que has tomado y por que.
```

**Consejo:** Cuanto mas contexto des en la descripción del proyecto, mejor será la arquitectura.
Incluye el tipo de usuario, el problema que resuelves, y cualquier integración externa que necesites.

### El Constructor
**Fase:** Generación de Código funcional

Cuando ya sabes que construir, necesitas código que funcione a la primera. Este prompt genera
código limpio, modular y con manejo de errores incluido. No pide un ejemplo: pide producción.

```
Actua como un desarrollador senior especializado en [tu lenguaje/framework. Ej: "TypeScript
con Next.js y Prisma"].

Necesito que implementes lo siguiente:

## Funcionalidad
[Describe exactamente que debe hacer el codigo. Ej: "Un endpoint POST /api/auth/register
que reciba email y password, valide los campos, hashee la contraseña con bcrypt, guarde
el usuario en PostgreSQL y devuelva un JWT."]

## Contexto técnico
- Stack: [lenguaje, framework, ORM, etc.]
- Base de datos: [tipo y si ya existe el schema]
- Convenciones del proyecto: [nombrado de archivos, estilo de codigo, o "Usa
convenciones estandar"]

## Formato de entrega
Entrega el código en bloques separados por archivo. Indica la ruta de cada archivo como
encabezado. Al final, incluye una sección breve "Como probarlo" con los pasos exactos.

## Requisitos del código
1.Código listo para produccion, no ejemplos simplificados.
2.Incluir validacion de inputs y manejo de errores completo.
3.Aplicar el principio de responsabilidad única.
6.Si necesitas alguna dependencia externa, indícala al inicio con el comando de instalación.
4.Anadir tipos/interfaces cuando el lenguaje lo permita.
5.Incluir comentarios SOLO donde la logica no sea obvia.
```

**Consejo:** La clave es la especificidad. "Haz un login" produce código genérico. "Endpoint POST
que reciba email/password, valide formato, hashee con bcrypt..." produce código que puedes
usar directamente.

