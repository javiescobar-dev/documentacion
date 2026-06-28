### El arquitecto:
**Fase: Planificación y Diseño del sistema**

Antes de escribir una sola línea de código, necesitas pensar la estructura. Este prompt convierte
una idea vaga en un plan técnico con stack, estructura de archivos, modelo de datos y decisiones
justificadas. Úsalo al inicio de cada proyecto o funcionalidad nueva.

```sh
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
**Fase: Generación de Código funcional**

Cuando ya sabes que construir, necesitas código que funcione a la primera. Este prompt genera
código limpio, modular y con manejo de errores incluido. No pide un ejemplo: pide producción.

```sh
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

### El Detective
**Fase: Debugging y Resolución de errores**

Los bugs son inevitables. Lo que no es inevitable es pasar 3 horas buscando la causa. Este
prompt usa razonamiento paso a paso (Chain of Thought) para forzar un análisis metódico:
hipótesis, análisis, causa raiz y solución.

```sh
Actua como un debugger experto. Necesito que analices un problema en mi código
de forma metódica.

## El problema
- Qué deberia pasar: [Comportamiento esperado. Ej: "El formulario deberia enviar
los datos y redirigir a /dashboard."]
- Qué pasa en realidad: [Comportamiento actual. Ej: "La página se recarga pero
no redirige. No aparece error en consola."]
- Mensaje de error (si hay): [Pega el error exacto o "No hay error visible"]
- Cuándo ocurre: [Siempre / solo a veces / solo en produccion / etc.]

## Código relevante
``` ```
[Pega aquí el código donde crees que esta el problema]
``` ```

## Contexto adicional
- Lenguaje/framework: [ej: React 19, Node 22]
- Que ya intente: [Lista lo que probaste. Esto evita sugerencias que ya descartaste]

## Cómo quiero que respondas
Sigue este proceso exacto:
1.Hipótesis inicial: Lista 3 causas posibles ordenadas por probabilidad.
2.Analisis linea por linea: Revisa el codigo y senala exactamente donde podría estar el fallo.
3.Causa raiz: Identifica la causa más probable y explica POR QUÉ provoca
el comportamiento observado.
4.Solución: Muestra el código corregido con los cambios resaltados.
5.Prevención: Sugiere una práctica o patron para evitar este tipo de error en el futuro.
```

**Consejo:** Incluir "que ya intente" es clave. Sin eso, la IA te sugerirá lo primero y más obvio
(que probablemente ya probaste). Con eso, va directo a causas menos evidentes.

### El Crítico
**Fase: Revisión de Código**

Tener un segundo par de ojos siempre mejora el código. Este prompt simula un code review
riguroso: seguridad, rendimiento, mantenibilidad y puntuación. Es como tener un compañero
senior disponible 24/7.

```sh
Actúa como un code reviewer senior exigente pero constructivo. Revisa el siguiente
código como si fuera un Pull Request en un equipo profesional.

## Código relevante
``` ```
[Pega aquí tu código]
``` ```

## Contexto
- Lenguaje/framework: [ej: Python con FastAPI]
- Que hace este código: [breve descripción de la funcionalidad]
- Es código de: [API / frontend / servicio / script / etc.]

## Analiza cada una de estas dimensiones
1.Seguridad: Hay vulnerabilidades? (inyección SQL, XSS, exposición de datos sensibles,
secrets hardcodeados, etc.)
2.Rendimiento: Hay cuellos de botella? (queries N+1, operaciones O(n2), cargas innecesarias...)
Al final, da una puntuación global de 1 a 10 con un resumen de una linea.
Cierra con los 3 cambios de mayor impacto que harías si solo pudieras cambiar 3 cosas.
3.Código limpio: Cumple con el principio de responsabilidad única? Los nombres
son descriptivos? Hay duplicación?
4.Patrones y estructura: Usa patrones adecuados? La estructura es coherente con el framework?
5.Manejo de errores: Gestiona los edge cases? Los errores se manejan o se tragan
silenciosamente?

## Formato de respuesta
Para cada dimensión:
- Estado: Bien / Mejorable / Problema
- Si hay problema: explica que, donde y muestra el código corregido.

Al final, da una puntuación global de 1 a 10 con un resumen de una linea.
Cierra con los 3 cambios de mayor impacto que harías si solo pudieras cambiar 3 cosas.
```

**Consejo:** Úsalo también con código que crees que "ya esta bien". Los problemas de seguridad
y rendimiento rara vez son obvios para quien escribió el código.
