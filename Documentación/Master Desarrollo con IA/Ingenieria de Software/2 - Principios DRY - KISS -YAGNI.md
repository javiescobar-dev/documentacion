
DRY, KISS y YAGNI son tres principios de diseño que sirven para escribir código **fácil de leer, de mantener y de cambiar**. Junto con SOLID forman el sentido común del buen diseño: mientras SOLID se centra en la **estructura** (clases, interfaces, dependencias), estos tres se centran sobre todo en la **cantidad y la forma** del código que escribes.

No son reglas mágicas ni obligatorias. Son **señales** para darte cuenta de cuándo un diseño se está volviendo un problema: cuándo repites de más, cuándo complicas de más y cuándo construyes de más.

La idea de fondo es la misma que en SOLID: el código se lee y se cambia muchísimo más de lo que se escribe la primera vez, así que conviene escribir **lo mínimo, lo más simple y lo más claro** que resuelva lo que de verdad se necesita hoy.

---

## Resumen en una frase

| Sigla | Principio | En una frase |
|-------|-----------|--------------|
| **DRY** | Don't Repeat Yourself | No repitas el mismo **conocimiento** en dos sitios. |
| **KISS** | Keep It Simple, Stupid | La solución más **simple** que funcione es la mejor. |
| **YAGNI** | You Aren't Gonna Need It | No construyas hoy lo que **quizá** necesites mañana. |

---

## DRY — Don't Repeat Yourself

**Definición sencilla:** cada pieza de **conocimiento** del sistema debería tener una única representación autorizada. Si una regla cambia, deberías cambiarla en **un solo sitio**.

**Analogía:** una única emisora de radio. Si cada habitación tuviera su propio altavoz con su propio disco, tendrías que ir cambiando el disco habitación por habitación. Con una sola emisora, cambias la canción una vez y suena en toda la casa.

### ❌ Malo: la misma regla escrita varias veces

```ts
function precioConIva(base: number): number {
  return base * 1.21          // el 21% aparece aquí...
}

function resumenFactura(base: number): number {
  return Math.round(base * 1.21 * 100) / 100   // ...y otra vez aquí...
}

function totalCarrito(base: number): number {
  return base + base * 0.21   // ...y de otra forma distinta aquí
}
```

El día que el IVA cambie al 22 %, hay que localizar **todos** los sitios donde se escribió el 21 %. Basta olvidar uno para introducir un fallo silencioso (y fíjate: incluso aquí, una de las copias ya está mal).

### ✅ Bueno: el conocimiento vive en un único sitio

```ts
const IVA = 0.21

function conIva(base: number): number {
  return Math.round(base * (1 + IVA) * 100) / 100
}

function precioConIva(base: number): number { return conIva(base) }
function resumenFactura(base: number): number { return conIva(base) }
function totalCarrito(base: number): number { return conIva(base) }
```

Ahora el tipo de IVA está en **un solo lugar**. Cambia `IVA` y cambia todo el sistema a la vez.

### El matiz más importante: DRY es sobre conocimiento, no sobre texto

Dos trozos de código que **se parecen** no son necesariamente duplicación. Lo son solo si representan **la misma decisión** y, por tanto, cambian **por el mismo motivo**.

```ts
// ❌ NO es duplicación: se parecen, pero cambiarían por motivos distintos.
function precioVenta(base: number): number { return base * 0.9 }   // política comercial
function precioCoste(base: number): number { return base * 0.9 }   // margen de proveedor
```

Si unificas esto solo porque "el resultado es el mismo número", creas un acoplamiento falso: el día que una cambie, romperás la otra. **Unir lo que no es lo mismo es peor que duplicar.**

**🚩 Señal de alarma:** el mismo número, la misma regla o el mismo cálculo copiados por varios ficheros; un cambio de requisito te obliga a editar en varios sitios a la vez; una copia ya está desincronizada de las demás (síntoma clásico de que el "original" se olvidó en algún sitio).

**Pregunta clave:** si esta regla cambia, ¿cuántos ficheros tengo que tocar? Si es más de uno, está duplicada.

> Nota: el antónimo de DRY es **WET** (*Write Everything Twice*). Y el famoso consejo de "la regla de tres" va en esta línea: la primera vez escribe, la segunda duplica, y solo a la **tercera** extrae lo común (y hazlo solo si de verdad es el mismo conocimiento).

---

## KISS — Keep It Simple, Stupid

**Definición sencilla:** entre dos soluciones que funcionan, elige **la más simple**. La complejidad hay que ganársela: solo se añade si resuelve un problema real.

**Analogía:** para colgar un cuadro no hace falta un robot industrial; un clavo y un martillo. Cuanta más maquinaria metes, más cosas se pueden romper, más difícil es de entender y más caro es de cambiar.

### ❌ Malo: el mismo resultado, con muchísimo ruido

```ts
function costeEnvio(pedido: Pedido): number {
  let coste = 0
  if (pedido.pais === 'ES') {
    if (pedido.exprés) {
      if (pedido.pesoKg > 10) coste = 24.95
      else coste = 9.95
    } else {
      if (pedido.pesoKg > 10) coste = 8.95
      else if (pedido.subtotal >= 50) coste = 0
      else coste = 4.95
    }
  } else if (pedido.pais === 'PT' || pedido.pais === 'FR' || pedido.pais === 'IT') {
    // ...y así hasta cinco niveles de if anidados
  }
  // ...
  return coste
}
```

Para saber "cuánto cuesta enviar a Francia" hay que recorrer mentalmente toda la maraña de ramas. El código no cabe en la cabeza.

### ✅ Bueno: la complejidad se convierte en una tabla

```ts
const TARIFAS = {
  ES:    { express: { light: 9.95,  heavy: 24.95 }, standard: { light: 4.95,  heavy: 8.95  } },
  EU:    { express: { light: 19.95, heavy: 39.95 }, standard: { light: 9.95,  heavy: 22.95 } },
  WORLD: { express: { light: 59.95, heavy: 59.95 }, standard: { light: 24.95, heavy: 24.95 } },
}

function costeEnvio(pedido: Pedido): number {
  const zona = zonaDe(pedido.pais)                         // ES | EU | WORLD
  const tramo = pedido.pesoKg > 10 ? 'heavy' : 'light'
  const tipo = pedido.exprés ? 'express' : 'standard'
  return TARIFAS[zona][tipo][tramo]
}
```

El dato va en datos y la lógica queda a la vista. Se lee de un vistazo y añadir un país es añadir una fila.

**🚩 Señal de alarma:** funciones larguísimas o muy anidadas; números "mágicos" sueltos por el código; soluciones "listas" que cuesta explicar en voz alta; comentarios largos para justificar algo que debería ser evidente por sí solo.

**Pregunta clave:** ¿puedo explicarle este código a otra persona sin dibujar un diagrama? Si me cuesta más de un minuto, probablemente se puede simplificar.

> Nota: **simple no es lo mismo que corto**. Un nombre claro y una función de tres líneas es más simple que una de una sola línea imposible de leer. Se simplifica para que el lector **entienda antes**, no para ocupar menos.

---

## YAGNI — You Aren't Gonna Need It

**Definición sencilla:** no construyas hoy la funcionalidad, la flexibilidad o la abstracción que **crees** que vas a necesitar mañana. Impleméntalo **cuando** de verdad se necesite.

**Analogía:** preparar la maleta para un viaje de tres días sin cargar también la ropa "por si acaso nieva, por si acaso hay boda". Cuanto más metes de más, más pesa todo y menos encuentras lo que sí necesitas.

### ❌ Malo: infraestructura para un futuro que no llegó

```ts
// Un "motor de informes" con cachés, plugins y fábricas...
class ReportEngine {
  private cache: ReportCache
  private plugins: PluginRegistry
  constructor(options: ReportOptions = DEFAULT_OPTIONS) { /* ... */ }
}

const cache = createCache('lru', { maxSize: 100 })   // nadie la usa
const registry = new PluginRegistry()                 // nadie registra nada
class AbstractReportRendererFactory { /* ... */ }     // una sola subclase real
```

Cientos de líneas pensadas para escenarios hipotéticos (varios formatos de salida, cachés configurables, reintentos con espera aleatoria) que **ningún cliente** usa todavía. Es mantenimiento que pagas desde el primer día por un beneficio que quizá nunca llega.

### ✅ Bueno: solo lo que hoy se usa

```ts
interface ReportRow {
  label: string
  value: number
}

function generateReport(rows: ReportRow[], title: string): string {
  const lines = [title, '='.repeat(title.length)]
  for (const row of rows) {
    lines.push(`${row.label.padEnd(20, '.')} ${formatMoney(row.value)}`)
  }
  return lines.join('\n')
}
```

Cuando aparezca una necesidad real (un segundo formato, una caché de verdad), se añade **con la información que entonces se tendrá**, que casi nunca es la que imaginaste.

**🚩 Señal de alarma:** clases o parámetros "por si acaso"; abstracciones con una sola implementación y ningún cliente; capas configurables que nadie configura; código comentado "para más adelante".

**Pregunta clave:** ¿hay **alguien** que use esto hoy? Si la respuesta es "todavía no, pero puede que sí", probablemente sobra.

> Nota: YAGNI **no** es excusa para no pensar en el diseño. Es no **construir** lo que no hace falta. Pensar en el futuro está bien; pagar por él cuando aún no ha llegado es lo que no.

---

## Cómo diferenciarlos

| Si el problema es... | Suele ser... | Cómo reconocerlo |
|----------------------|--------------|------------------|
| La misma regla copiada en varios sitios | **DRY** | un cambio obliga a editar en varios ficheros |
| Una solución enrevesada para un problema sencillo | **KISS** | cuesta mucho entenderla o explicarla |
| Código/flexibilidad para algo que nadie usa | **YAGNI** | no tiene **ni un solo** cliente real |

---

## Cómo se relacionan entre sí (y con SOLID)

Los tres tiran del mismo carro, pero a veces **chocan** entre ellos. Ahí está lo interesante:

- **DRY vs KISS:** extraer una abstracción para no repetir puede complicar el código. Si unificar hace el resultado **menos** legible, quizá merezca la pena duplicar. Repetir un poco suele ser más barato que una mala abstracción.
- **DRY vs YAGNI:** la abstracción "que servirá para todo" es a la vez prematura (YAGNI) y frágil. Extrae solo cuando el patrón **ya se ha repetido** de verdad (la regla de tres).
- **YAGNI vs OCP (SOLID):** aquí está el matiz más fino. OCP te empuja a **abstraer** para poder añadir casos sin tocar código; YAGNI te frena si ese caso nuevo no existe aún. La clave: **OCP se aplica cuando hay un eje de variación real** (varios formatos que de verdad se van a usar); si el eje es imaginario, abstraer es YAGNI y sobra.
- **KISS vs SOLID en general:** SOLID añade clases, interfaces y capas para desacoplar. KISS recuerda que todo eso **tiene un coste**: hay que aplicarlo donde reduzca el dolor, no por protocolo.

> Regla común: los tres son **heurísticas**, no leyes. La pregunta que los une es siempre *"¿esto hace que el código sea más fácil de cambiar dentro de seis meses?"*.

---

## Cómo localizarlos (checklist rápida)

Cuando revises código, pregúntate en este orden:

1. **La misma regla, número o cálculo en varios sitios** → probable **DRY**.
2. **Funciones largas, muy anidadas o con números mágicos** → probable **KISS**.
3. **Clases, parámetros o abstracciones que nadie usa** → probable **YAGNI**.
4. **Un `switch`/`if` comparando un tipo** que hay que tocar al añadir casos → piensa en **OCP** (y valora si el caso nuevo **ya** existe: entonces YAGNI no aplica).
5. **Una abstracción con una sola implementación y sin clientes** → sospecha de **YAGNI** (o de DRY llevado demasiado lejos).

> Regla de oro: escribe lo **mínimo** (YAGNI), lo más **simple** (KISS) y **sin repetir conocimiento** (DRY). Cuando estos tres se contradigan, gana la **legibilidad**.
