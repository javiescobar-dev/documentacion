Un antipatrón es una **solución que parece razonable**, se repite mucho y acaba creando **más problemas de los que resuelve**. No es un error puntual: es una forma de trabajar o una estructura que se contagia. Casi siempre empieza como un atajo tomado bajo presión.

La diferencia con un patrón es la dirección del resultado: el patrón resuelve un problema recurrente; el antipatrón lo empeora. La línea es fina, porque **el mismo patrón aplicado sin necesidad** se convierte en sobreingeniería, que también es un antipatrón.

Tampoco son lo mismo un **antipatrón** y un **olor de código**. El olor es un síntoma local ("este método tiene 200 líneas"); el antipatrón es la práctica recurrente que lo produce ("objeto dios"). Los olores te ayudan a detectarlo, pero no arregles olores sueltos: busca la causa.

---

## Resumen en una frase

| Antipatrón | Nivel | Síntoma | Consecuencia | Remedio |
|------------|-------|---------|--------------|---------|
| **Objeto dios** (God Object) | Código | Una clase lo sabe y lo hace todo | Frágil, imposible de testear | SRP, extraer y delegar |
| **Código espagueti** | Código | Flujo enredado y anidado | Nadie se atreve a tocarlo | KISS, retornos tempranos, tabla |
| **Copiar y pegar** | Código | La misma regla en varios sitios | Cambios incompletos | DRY, extraer conocimiento |
| **Números mágicos** | Código | `0.21`, `30000`, `'05'` sueltos | Significado perdido | Constantes con nombre |
| **Lista larga de parámetros** | Código | 14 argumentos posicionales | Llamadas ilegibles | Objeto de parámetros o Builder |
| **Martillo dorado** | Diseño | La misma herramienta para todo | Fuerza el problema | Elegir según el contexto |
| **Sobreingeniería** | Diseño | Capas y abstracciones sin uso | Coste y lentitud | YAGNI, KISS, borrar |
| **Flujo de lava** | Código | Código muerto que nadie borra | Confusión y miedo | Borrar; Git lo recuerda |
| **Gran bola de barro** | Arquitectura | Todo depende de todo | Imposible escalar o testear | Módulos y dependencias claras |
| **Modelo de dominio anémico** | Diseño | Clases solo con datos; lógica fuera | Invariantes dispersas | Llevar comportamiento al dominio |
| **Culto al cargo** | Proceso | Copiar de la IA o de un blog sin entender | Errores ocultos | Revisar, probar, adaptar |
| **Optimización prematura** | Proceso | Complejidad por rendimiento no medido | Coste sin beneficio | Medir antes de optimizar |

---

## Antipatrones de código

### Objeto dios (God Object)

**Definición sencilla:** una clase que concentra demasiadas responsabilidades y demasiado conocimiento. Todos los cambios pasan por ella y todo el mundo tiene miedo de tocarla.

**Analogía:** un empleado que es recepcionista, contable, cocinero y seguridad. Si se pone enfermo, el hotel cierra.

### ❌ Malo: la clase que hace de todo

```ts
class CheckoutService {
  checkout(request: CheckoutRequest) {
    // 152 líneas: caché, logging, validaciones, precio,
    // cobro, pedido, email, analítica, stock, imprenta...
  }

  enviarEmail() { /* ... */ }
  actualizarStock() { /* ... */ }
  imprimirEntradas() { /* ... */ }
  calcularPrecio() { /* ... */ }
  // ... 250 líneas en total
}
```

Es el punto de partida de TicketBox. Cualquier cambio (un método de pago, un aviso nuevo, una regla de precio) obliga a abrir el mismo archivo. Y como todo está relacionado con todo, los tests necesitan montar medio sistema.

### ✅ Bueno: una clase que orquesta y delega

```ts
class CheckoutService {
  constructor(
    private readonly estrategias: Map<PaymentMethod, PaymentStrategy>,
    private readonly observers: PurchaseObserver[],
  ) {}

  checkout(request: CheckoutRequest): CheckoutResult {
    // Orquesta el caso de uso, pero no sabe cómo se cobra
    // ni cómo se envía un email. Delega.
  }
}
```

En TicketBox, `checkout()` pasó de 152 a 31 líneas: el servicio sigue existiendo, pero el **conocimiento** se repartió en estrategias, observadores, adaptador y tabla de transiciones.

**🚩 Señal de alarma:** una clase de cientos de líneas; muchos `import` que no tienen que ver entre sí; la palabra "y" aparece al describirla; cambiar cualquier cosa obliga a volver a ella.

**Pregunta clave:** ¿cuántas razones distintas tiene esta clase para cambiar? Si son más de una, sobra responsabilidad.

**Remedio:** escribe tests que capturen el comportamiento actual, extrae métodos con nombres de intención y mueve cada responsabilidad a su clase. No lo hagas todo de golpe: un paso, tests, siguiente paso.

---

### Código espagueti

**Definición sencilla:** un flujo sin estructura clara, lleno de saltos, anidamientos y banderas. Para entender una rama hay que reconstruir mentalmente todo el recorrido.

**Analogía:** una carretera sin señales ni salidas. Quieres ir a la panadería y acabas en otra provincia.

### ❌ Malo: anidar en vez de decidir

```ts
function precio(pedido: Pedido): number {
  let total = 0
  if (pedido) {
    if (pedido.pais === 'ES') {
      if (pedido.vip) {
        if (pedido.cantidad > 10) {
          total = pedido.base * 0.8
        } else {
          total = pedido.base * 0.9
        }
      } else {
        total = pedido.base
      }
    } else {
      total = pedido.base * 1.2
    }
  }
  return total
}
```

Cada regla nueva añade otro nivel. El lector tiene que mantener en la cabeza todas las condiciones anteriores para saber qué pasa en la última rama.

### ✅ Bueno: guardas, nombres y una tabla

```ts
const DESCUENTO_VIP = 0.1
const RECARGO_INTERNACIONAL = 0.2

function precio(pedido: Pedido): number {
  if (!pedido) throw new Error('El pedido es obligatorio')
  if (pedido.cantidad <= 0) throw new Error('La cantidad debe ser positiva')

  const descuento = pedido.vip ? DESCUENTO_VIP : 0
  const recargo = pedido.pais === 'ES' ? 0 : RECARGO_INTERNACIONAL

  return pedido.base * (1 - descuento) * (1 + recargo)
}
```

La lógica queda a la vista: validar, calcular descuento, calcular recargo, combinar. Si el cálculo se complica, se convierte en una **tabla** (como en KISS) en lugar de en más `if`.

**🚩 Señal de alarma:** funciones de más de 50 líneas; más de tres niveles de anidamiento; banderas booleanas que cambian el flujo; comentarios para explicar "a qué rama estamos entrando".

**Pregunta clave:** ¿puedo explicar el flujo en voz alta sin dibujar un árbol?

**Remedio:** retornos tempranos, nombres claros, extraer funciones y sustituir ramas por datos. Si el flujo es un pipeline, un patrón Chain puede ayudar; si no, no lo fuerces.

---

### Copiar y pegar (Copy-Paste Programming)

**Definición sencilla:** duplicar código o, peor, duplicar **conocimiento**. La misma regla vive en varios sitios y nadie recuerda cuál es la copia buena.

**Analogía:** tener tres relojes en casa y que cada uno marque una hora distinta. Para saber la hora real tienes que decidir a cuál creer.

### ❌ Malo: la misma regla en tres sitios

```ts
function precioConIva(base: number): number {
  return base * 1.21
}

function resumenFactura(base: number): number {
  return Math.round(base * 1.21 * 100) / 100
}

function totalCarrito(base: number): number {
  return base + base * 0.21
}
```

El día que el IVA cambie, hay que localizar las tres copias. Basta olvidar una para introducir un fallo silencioso. Y fíjate: una de las copias ya está mal (redondea distinto).

### ✅ Bueno: el conocimiento vive en un solo sitio

```ts
const IVA = 0.21

function conIva(base: number): number {
  return Math.round(base * (1 + IVA) * 100) / 100
}

function precioConIva(base: number): number { return conIva(base) }
function resumenFactura(base: number): number { return conIva(base) }
function totalCarrito(base: number): number { return conIva(base) }
```

En TicketBox, la traducción a la pasarela legacy estaba copiada en tarjeta y monedero. El **Adapter** la dejó en un único sitio.

**🚩 Señal de alarma:** el mismo número, la misma regla o el mismo cálculo en varios archivos; un cambio de requisito te obliga a editar en varios sitios; una copia ya está desincronizada.

**Pregunta clave:** si esta regla cambia, ¿cuántos archivos tengo que tocar? Si es más de uno, está duplicada.

**Remedio:** extrae el conocimiento a una función, constante, clase o adaptador. Pero cuidado: **DRY es sobre conocimiento, no sobre texto**. Dos trozos que se parecen pero cambian por motivos distintos **no** son duplicación; unificarlos crea un acoplamiento falso.

---

### Números mágicos

**Definición sencilla:** valores literales sin nombre que esconden una regla de negocio. El lector no sabe qué significan ni por qué están ahí.

**Analogía:** una receta que dice "añade 37". ¿37 qué? ¿gramos, mililitros, minutos? Sin unidades ni contexto, no se puede seguir.

### ❌ Malo: literales por todas partes

```ts
if (importe > 30000) {
  throw new Error('Importe demasiado alto')
}

if (codigo === '05') {
  return 'Tarjeta rechazada'
}

const total = base * 1.21

if (evento.amountCharged > 200) {
  // alerta antifraude
}
```

¿Qué es `30000`? ¿céntimos o euros? ¿Qué significa `'05'`? ¿Por qué `200`? El código funciona hasta que alguien cambia una regla y no encuentra todas las apariciones.

### ✅ Bueno: cada número tiene nombre y contexto

```ts
const MAXIMO_PASARELA_CENTIMOS = 30_000
const CODIGO_TARJETA_RECHAZADA = '05'
const IVA = 0.21
const UMBRAL_ANTIFRAUDE_EUR = 200

if (importe > MAXIMO_PASARELA_CENTIMOS) {
  throw new Error('Importe por encima del máximo de la pasarela')
}

if (codigo === CODIGO_TARJETA_RECHAZADA) {
  return 'Tarjeta rechazada'
}

const total = base * (1 + IVA)

if (evento.amountCharged > UMBRAL_ANTIFRAUDE_EUR) {
  // alerta antifraude
}
```

**🚩 Señal de alarma:** números y cadenas repetidos; constantes que solo se entienden mirando el resto del método; reglas de negocio escritas a mano en varios sitios.

**Pregunta clave:** si mañana cambia este valor, ¿sabría qué significa y dónde están todas sus apariciones?

**Remedio:** constantes con nombre, enums o **objetos de valor** (`Importe`, `Porcentaje`, `CodigoRespuesta`). No conviertas cada número en una constante: solo los que representan una **decisión de negocio**.

---

### Lista larga de parámetros

**Definición sencilla:** una función o constructor con tantos argumentos posicionales que ya no se puede leer ni llamar sin equivocarse.

**Analogía:** dar instrucciones a un taxista con catorce datos seguidos: "a la izquierda, luego a la derecha, luego...". Es más fácil darle una dirección completa.

### ❌ Malo: catorce posiciones que nadie recuerda

```ts
const peticion = new CheckoutRequest(
  'ORD-2045', 'EVT-9', 'ana@example.com',
  2, 'vip', 10, false,
  'card', 'token-123', null, null, null, null, null,
)
```

¿El `10` es descuento o cantidad? ¿El `false` es regalo o VIP? Si añades un campo, todos los llamadores se enteran. Y los `null` no dicen nada.

### ✅ Bueno: un objeto con nombres

```ts
const peticion: CheckoutRequest = {
  orderId: 'ORD-2045',
  eventId: 'EVT-9',
  customerEmail: 'ana@example.com',
  quantity: 2,
  ticketType: 'vip',
  discountPercentage: 10,
  isGift: false,
  paymentMethod: 'card',
  cardToken: 'token-123',
}
```

Si además hay muchos campos opcionales y reglas de construcción, un **Builder** puede ayudar. Pero para un solo punto de llamada, un literal con nombres ya es legible.

**🚩 Señal de alarma:** constructores con 10+ parámetros; llamadas con `null, null, null`; booleanos seguidos que nadie distingue; el orden importa y no está documentado.

**Pregunta clave:** ¿el problema es construir el objeto o solo que la lista es larga? Si es lo segundo, un objeto de parámetros basta.

**Remedio:** objeto de parámetros, Builder o dividir la operación en pasos más pequeños.

---

## Antipatrones de diseño y arquitectura

### Martillo dorado (Golden Hammer)

**Definición sencilla:** cuando tu única herramienta es un martillo, todo te parece un clavo. Se aplica la misma tecnología, patrón o solución a **todos** los problemas, aunque no encaje.

**Analogía:** usar un martillo industrial para colgar un cuadro. Funciona, pero rompe la pared y cuesta más que un clavo.

### ❌ Malo: la solución de moda para todo

- "Tenemos microservicios, pues partimos también el formulario de contacto en tres servicios."
- "Aprendí Strategy, pues meto una estrategia hasta para sumar dos números."
- "La IA genera código, pues le pido que escriba el sistema entero y lo pego."

El problema no es la herramienta: es **no preguntar si encaja**. La solución se elige por contexto, no por costumbre.

### ✅ Bueno: elegir según el problema

- ¿Tres pantallas y un equipo pequeño? Un monolito modular es más simple que microservicios.
- ¿Dos variantes estables? Un `if` claro es mejor que cinco clases.
- ¿Una regla de negocio? Una función con tests, no un modelo de IA.

**🚩 Señal de alarma:** la misma palabra ("microservicios", "patrón", "IA") aparece en todas las soluciones; no se comparan alternativas; el coste no se mide; el problema se adapta a la herramienta y no al revés.

**Pregunta clave:** ¿elegí esta solución porque encaja o porque es la que sé usar?

**Remedio:** escribe el problema, compara al menos dos alternativas y elige la más simple que lo resuelva. La experiencia vale, pero el contexto manda.

---

### Sobreingeniería (Arquitectura de astronauta)

**Definición sencilla:** construir para un futuro hipotético: capas, abstracciones, configuraciones y patrones que nadie usa todavía. El sistema se vuelve más caro de entender y de cambiar.

**Analogía:** preparar la maleta para un viaje de tres días con ropa "por si nieva, por si hay boda". Pesa más y no encuentras lo que sí necesitas.

### ❌ Malo: infraestructura para un futuro que no llegó

```ts
class ReportEngine {
  private cache: ReportCache
  private plugins: PluginRegistry
  private factories: RendererFactoryRegistry
  private retryPolicy: ExponentialBackoffPolicy

  constructor(options: ReportOptions = DEFAULT_OPTIONS) {
    // ...
  }
}
```

Cientos de líneas para escenarios que nadie ha pedido: varios formatos, cachés configurables, plugins que nadie registra. Es mantenimiento que pagas desde el primer día.

### ✅ Bueno: solo lo que hoy se necesita

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

Cuando aparezca una necesidad real (un segundo formato, una caché de verdad), se añade con la información que entonces se tendrá.

**🚩 Señal de alarma:** interfaces con una sola implementación y ningún doble de test; parámetros "por si acaso"; capas que solo pasan datos; configuraciones que nadie configura; código comentado "para más adelante".

**Pregunta clave:** ¿hay **alguien** que use esto hoy? Si la respuesta es "todavía no, pero puede que sí", probablemente sobra.

**Remedio:** YAGNI y KISS. Borra la capa, el patrón o el parámetro. Si de verdad hace falta, volverá con una necesidad concreta. **No es lo mismo pensar en el futuro que pagarlo por adelantado.**

---

### Flujo de lava (Lava Flow)

**Definición sencilla:** código muerto, comentado o antiguo que se queda en el repositorio porque nadie se atreve a borrarlo. Se solidifica y forma parte del paisaje.

**Analogía:** lava que se enfrió y nadie limpió. Ya no arde, pero ocupa la carretera.

### ❌ Malo: comentarios y ramas que ya no significan nada

```ts
// function antiguoCalcularPrecio() {
//   return base * 1.16
// }

// TODO: revisar esto en 2019
if (false) {
  enviarFax()
}

function nuevoCheckout() {
  // ...
}
```

Nadie sabe si el código comentado se puede recuperar, si la rama `false` era una prueba o si `antiguoCalcularPrecio` sigue siendo la fuente de verdad. El lector pierde tiempo y el equipo pierde confianza.

### ✅ Bueno: borrar sin miedo

```ts
function nuevoCheckout() {
  // ...
}
```

El historial de Git ya guarda la versión anterior. Si de verdad hace falta, se recupera. El código vivo debe ser el que se usa.

**🚩 Señal de alarma:** bloques comentados; funciones que nadie importa; `if (false)`; banderas de funcionalidad que ya no se activan; `TODO` con años; carpetas "old", "backup" o "copy".

**Pregunta clave:** ¿alguien usa esto hoy? ¿Hay un test que lo cubra? Si no, ¿por qué está aquí?

**Remedio:** borra. Si da miedo, primero escribe un test de caracterización y comprueba que no se usa. El control de versiones es la máquina del tiempo; no necesitas el museo en la rama principal.

---

### Gran bola de barro (Big Ball of Mud)

**Definición sencilla:** una arquitectura sin límites claros donde todo depende de todo. Cada cambio puede romper cualquier cosa y no hay una dirección de dependencias.

**Analogía:** una habitación donde todo está tirado. Para encontrar un calcetín levantas la cama, y al levantar la cama rompes la lámpara.

### ❌ Malo: dependencias en todas las direcciones

```ts
// ui.ts
import { db } from './database'

// database.ts
import { ui } from './ui'

// checkout.ts
import { ui } from './ui'
import { db } from './database'
import { email } from './email'
import { analytics } from './analytics'
import { stock } from './stock'
```

La interfaz conoce la base de datos, la base de datos conoce la interfaz y todos conocen a todos. No se puede testear una pieza sin montar el sistema entero.

### ✅ Bueno: direcciones claras y contratos

```ts
// domain/OrderRepository.ts
export interface OrderRepository {
  save(order: Order): void
}

// application/CheckoutUseCase.ts
import type { OrderRepository } from '../domain/OrderRepository'

export class CheckoutUseCase {
  constructor(private readonly orders: OrderRepository) {}
  // ...
}

// infrastructure/PostgresOrderRepository.ts
import type { OrderRepository } from '../domain/OrderRepository'

export class PostgresOrderRepository implements OrderRepository {
  // ...
}
```

La interfaz vive en el dominio; la infraestructura depende de ella, no al revés (**DIP**). Los módulos tienen una dirección y un contrato.

**🚩 Señal de alarma:** dependencias circulares; un cambio en la interfaz obliga a tocar la base de datos; tests que necesitan levantar medio sistema; carpetas `utils` de las que depende todo; build lento y frágil.

**Pregunta clave:** ¿puedo cambiar una pieza sin tocar las demás? ¿Sé en qué dirección van las dependencias?

**Remedio:** módulos con límites claros, interfaces en el dominio e inversión de dependencias. Si el sistema ya es una bola, no lo reescribas de golpe: usa el **patrón estrangulador** (strangler): envuelve una parte, migra poco a poco y borra lo viejo.

---

### Modelo de dominio anémico

**Definición sencilla:** las clases del dominio solo tienen datos (getters y setters), y toda la lógica vive en servicios externos. Las reglas se dispersan y las invariantes se pueden romper desde cualquier sitio.

**Analogía:** un formulario que cualquiera puede rellenar con valores imposibles. No hay nadie en la puerta comprobando que la fecha de salida sea posterior a la de entrada.

### ❌ Malo: el pedido es solo un saco de datos

```ts
class Pedido {
  id: string
  estado: string
  total: number
}

class PedidoService {
  pagar(pedido: Pedido): void {
    if (pedido.estado !== 'reserved') throw new Error('No se puede pagar')
    pedido.estado = 'paid'
  }

  cancelar(pedido: Pedido): void {
    if (pedido.estado === 'issued') throw new Error('Ya no se puede cancelar')
    pedido.estado = 'cancelled'
  }
}
```

Cualquier servicio puede cambiar `pedido.estado` sin pasar por las reglas. Las transiciones están repartidas y nadie garantiza que se respeten.

### ✅ Bueno: el pedido protege sus invariantes

```ts
type Estado = 'created' | 'reserved' | 'paid' | 'issued' | 'cancelled' | 'refunded'
type Accion = 'reserve' | 'markPaid' | 'issueTickets' | 'cancel' | 'refund'

const TRANSICIONES: Record<Estado, Partial<Record<Accion, Estado>>> = {
  created:   { reserve: 'reserved', cancel: 'cancelled' },
  reserved:  { markPaid: 'paid', cancel: 'cancelled' },
  paid:      { issueTickets: 'issued', refund: 'refunded', cancel: 'cancelled' },
  issued:    { refund: 'refunded' },
  cancelled: {},
  refunded:  {},
}

class Pedido {
  private estado: Estado = 'created'

  pagar(): void { this.mover('markPaid') }
  cancelar(): void { this.mover('cancel') }

  private mover(accion: Accion): void {
    const siguiente = TRANSICIONES[this.estado][accion]
    if (!siguiente) throw new Error(`Transición no permitida: ${accion}`)
    this.estado = siguiente
  }
}
```

Ahora el estado solo cambia por los métodos del pedido, y las reglas están en un sitio. No hace falta un modelo de dominio rico para todo: si la aplicación es solo un formulario CRUD, un modelo anémico puede ser suficiente. El antipatrón aparece cuando hay **reglas de negocio reales** y se dejan fuera.

**🚩 Señal de alarma:** clases con solo getters/setters; servicios que hacen `if (objeto.estado === ...)`; setters públicos para todo; invariantes comprobadas en varios servicios.

**Pregunta clave:** ¿quién garantiza que este objeto sea siempre válido?

**Remedio:** mover el comportamiento y las invariantes al dominio; usar métodos con nombres de negocio (`pagar`, `cancelar`, `emitir`), no setters. Si el conjunto de estados es estable, una tabla puede bastar; no fuerces el patrón State.

---

## Antipatrones de proceso

### Culto al cargo (Cargo Cult) y copiar de la IA sin entender

**Definición sencilla:** copiar una práctica, un patrón o un código porque "funciona" o porque lo hace alguien con éxito, **sin entender por qué**. La IA hace que esto sea más fácil y más peligroso: genera código plausible en segundos.

**Analogía:** imitar el ritual de un culto sin saber qué significa. Quizá llueva, quizá no; pero nadie sabe por qué.

### ❌ Malo: pegar sin leer

- Copiar un patrón de un blog y meterlo aunque el problema sea otro.
- Pegar 200 líneas de la IA y decir "funciona" porque la demo no explota.
- Añadir una dependencia porque el tutorial la usa.
- Copiar una configuración sin saber qué hace.

### ✅ Bueno: usar la IA como tutora, no como oráculo

- Pide que te explique el código y **qué sobra**.
- Escribe tests que capturen el comportamiento.
- Reduce el código a lo que de verdad necesitas.
- Revisa nombres, dependencias y casos límite.
- Pregunta: "¿qué problema resuelve esto y cuándo **no** debería usarlo?".

**🚩 Señal de alarma:** no puedes explicar una línea; el código tiene nombres que no encajan con el proyecto; hay dependencias que nadie sabe por qué están; el comentario dice una cosa y el código hace otra.

**Pregunta clave:** ¿entiendo esto lo suficiente como para mantenerlo dentro de seis meses?

**Remedio:** leer, probar, simplificar y revisar. La IA acelera la escritura, pero **no sustituye el criterio**. Un código generado sin revisión es deuda técnica con buena letra.

---

### Optimización prematura

**Definición sencilla:** añadir complejidad para mejorar el rendimiento **antes de medir**. Se optimiza lo que parece lento, no lo que es lento.

**Analogía:** comprar un motor de carreras para un coche que aún no ha salido del garaje. Quizá el problema era la puerta.

### ❌ Malo: caché y complejidad sin datos

```ts
class Checkout {
  private cache = new Map<string, CheckoutResult>()

  checkout(request: CheckoutRequest): CheckoutResult {
    // caché, índices, workers, colas...
    // ...pero nadie ha medido cuánto tarda
  }
}
```

La caché puede estar bien, pero también puede introducir datos viejos, doble cobro o bugs difíciles. Si no hay una medición, no sabes si el problema era ese.

### ✅ Bueno: medir y atacar el cuello de botella

1. Define la métrica: tiempo de respuesta, memoria, consultas, coste.
2. Mide el flujo real con datos representativos.
3. Encuentra el cuello de botella (normalmente el 20 % que causa el 80 %).
4. Optimiza **eso**, vuelve a medir y compara.
5. Solo entonces añade caché, índices o concurrencia.

**🚩 Señal de alarma:** "esto es más rápido" sin benchmark; complejidad para un tráfico que no existe; optimizaciones que hacen el código ilegible; caché sin tests ni invalidación.

**Pregunta clave:** ¿qué número ha empeorado y cuánto mejora esto?

**Remedio:** primero medir, luego optimizar. La optimización prematura es una forma de YAGNI: pagas hoy por un problema que quizá no llegue.

---

## Otros antipatrones de proceso que oirás

| Antipatrón | Qué pasa | Cómo salir |
|------------|----------|------------|
| **Análisis parálisis** | Se investiga y documenta sin decidir | Timebox, prototipo, decide con la información disponible |
| **Bala de plata** | Se cree que una herramienta resuelve todo | Contexto, alternativas, prueba pequeña |
| **No inventado aquí** (NIH) | Se reinventa lo que ya existe | Evaluar librerías, coste de mantener |
| **Bicicleta de la casa** (Bikeshedding) | Se discute lo trivial y se ignora lo importante | Priorizar por impacto |
| **Deuda técnica sin pagar** | Se acumulan atajos y nadie los revisa | Presupuesto periódico, registrar y priorizar |
| **Funciona en mi máquina** | No hay entorno reproducible | Contenedores, CI, dependencias fijadas |
| **Falta de tests** | Cada cambio da miedo | Tests de caracterización antes de refactorizar |

---

## Antipatrones en TicketBox (clase 5)

| Antipatrón | Dónde se veía | Cómo se corrigió |
|------------|---------------|------------------|
| **Objeto dios** | `CheckoutService` de 250 líneas y un `checkout()` de 152 | SRP + Strategy/Observer/Adapter; `checkout()` en 31 líneas |
| **Lista larga de parámetros** | `createCheckoutRequest()` con 14 argumentos | Se eliminó; un literal con nombres |
| **Números mágicos** | Umbrales, códigos y límites (`30000`, `200`, `'05'`, `10`, `4`) | El umbral de antifraude ya es una constante (`ANTIFRAUD_THRESHOLD`); el resto sigue pendiente de extraer a constantes |
| **Copiar y pegar** | Traducción a la pasarela legacy en tarjeta y monedero | Un único `LegacyGatewayAdapter` |
| **Sobreingeniería** | Riesgo de aplicar todos los patrones del catálogo | Solo 3 patrones + una tabla; el resto se descartó con motivo |

---

## Olores de código que anuncian antipatrones

| Olor | Qué huele | Antipatrón probable |
|------|-----------|---------------------|
| **Método largo** | 100 líneas y muchos `if` | Espagueti u objeto dios |
| **Clase grande** | 1.000 líneas y muchos imports | Objeto dios |
| **Lista larga de parámetros** | 14 argumentos posicionales | Lista larga |
| **Código duplicado** | La misma regla en varios sitios | Copiar y pegar |
| **`switch` por tipo** | Crece con cada caso | Falta Strategy/State |
| **Envidia de funcionalidad** | Un método usa más datos de otra clase | Mal reparto de responsabilidades |
| **Grupos de datos** | Los mismos tres campos viajan juntos | Objeto de valor |
| **Obsesión por primitivos** | `string` para todo | Objeto de valor |
| **Cirugía de escopeta** | Un cambio toca 20 archivos | Acoplamiento / DRY |
| **Cambio divergente** | Una clase cambia por muchos motivos | SRP |
| **Comentarios como desodorante** | Explican código que debería ser claro | Refactor |
| **Código muerto** | Nadie lo usa y nadie lo borra | Flujo de lava |

> Un olor no es una condena. Es una pista. Antes de "arreglarlo", pregúntate qué antipatrón lo está generando.

---

## Cómo detectarlos (checklist rápida)

Cuando revises un proyecto, fíjate en estas señales:

1. **Miedo a cambiar**: nadie quiere tocar un archivo porque "siempre rompe algo".
2. **Un cambio toca muchos archivos**: cirugía de escopeta.
3. **La misma regla en varios sitios**: copiar y pegar.
4. **Tests difíciles**: hay que montar base de datos, red o relojes reales para probar una regla.
5. **Nadie sabe por qué existe una clase**: código muerto o sobreingeniería.
6. **"Lo dijo la IA / el blog"**: culto al cargo.
7. **Complejidad sin medición**: optimización prematura.
8. **El flujo no se explica en dos minutos**: espagueti.
9. **Comentarios que piden perdón**: el código no es claro.
10. **El equipo evita ciertos módulos**: objeto dios o bola de barro.

---

## Cómo salir de un antipatrón

No se sale reescribiendo todo de golpe. Se sale con pasos pequeños y red:

1. **Escribe tests de caracterización** que capturen lo que hace hoy (aunque esté mal).
2. **Haz un cambio pequeño** y vuelve a pasar los tests.
3. **Pon nombres de intención** antes de extraer: si no sabes nombrarlo, aún no lo entiendes.
4. **Extrae una responsabilidad** a su clase o función.
5. **Borra lo que ya no se usa**; Git guarda el historial.
6. **Mide** antes y después si el cambio es de rendimiento.
7. **Pide una revisión** (a una persona o a la IA como crítica) y decide tú.
8. **No arregles todo a la vez**: un antipatrón grande se desmonta en varios pasos.

---

## Cómo se relacionan con los principios y los patrones

| Antipatrón | Principio que se rompe | Herramienta que ayuda |
|------------|------------------------|-----------------------|
| Objeto dios | SRP | Strategy, Observer, Adapter, Facade |
| Código espagueti | KISS | Retornos tempranos, tablas, funciones pequeñas |
| Copiar y pegar | DRY | Funciones, constantes, Adapter, Template Method |
| Números mágicos | KISS / DRY | Constantes, objetos de valor |
| Lista larga de parámetros | KISS | Objeto de parámetros, Builder |
| Martillo dorado | YAGNI | Contexto, alternativas, prueba pequeña |
| Sobreingeniería | YAGNI / KISS | Borrar capas, medir coste |
| Flujo de lava | YAGNI | Git, tests, borrar |
| Gran bola de barro | DIP / SRP | Módulos, interfaces, patrón estrangulador |
| Modelo de dominio anémico | SRP / OCP | Métodos de dominio, State o tabla |
| Culto al cargo | Todos | Leer, probar, revisar, simplificar |
| Optimización prematura | KISS / YAGNI | Medir, perfilar, comparar |

> Los patrones no son la cura de todos los antipatrones. A veces la cura es **borrar código**, **una tabla** o **una función de tres líneas**. El patrón solo entra cuando el problema recurrente lo justifica.

---

## Checklist rápida

1. **¿Una clase lo hace todo?** → Objeto dios.
2. **¿El flujo es un laberinto?** → Espagueti.
3. **¿La misma regla está en varios sitios?** → Copiar y pegar.
4. **¿Hay números o cadenas sin nombre?** → Números mágicos.
5. **¿La llamada tiene 14 argumentos?** → Lista larga.
6. **¿La misma herramienta para todo?** → Martillo dorado.
7. **¿Capas y abstracciones que nadie usa?** → Sobreingeniería.
8. **¿Código que nadie borra?** → Flujo de lava.
9. **¿Todo depende de todo?** → Bola de barro.
10. **¿Clases solo con datos y lógica fuera?** → Dominio anémico.
11. **¿Copiado sin entender?** → Culto al cargo.
12. **¿Complejidad sin medición?** → Optimización prematura.

> Regla de oro: si no puedes explicar **por qué** existe una clase, bórrala. Si no puedes cambiar una línea sin miedo, faltan tests. Si añadir un caso te obliga a tocar cinco archivos, falta un patrón o sobra acoplamiento.
