Un patrón de diseño es una **solución con nombre** para un problema que aparece una y otra vez al diseñar software. No es una receta que se copia tal cual ni una librería: es una idea probada que te ayuda a **nombrar**, **comunicar** y **resolver** ese problema.

Los patrones no son el objetivo. El objetivo es un código **fácil de leer y de cambiar**. Un patrón añade clases, interfaces e indirección; ese coste solo se paga si el cambio que evita es real. Aplicarlos todos es la forma más rápida de violar KISS y YAGNI.

La idea de fondo es la misma que en SOLID y en DRY/KISS/YAGNI: el código se lee y se cambia muchísimo más de lo que se escribe la primera vez, así que conviene diseñarlo pensando en **por dónde va a cambiar**, no en el catálogo de patrones.

---

## Resumen en una frase

| Patrón | Familia | Problema que resuelve | En una frase |
|--------|---------|-----------------------|--------------|
| **Strategy** (Estrategia) | Comportamiento | Varias formas de hacer lo mismo | Una interfaz y una clase por variante; el cliente delega. |
| **Observer** (Observador) | Comportamiento | Avisar a varios sin conocerlos | El emisor publica un evento; los suscriptores escuchan. |
| **Adapter** (Adaptador) | Estructura | Encajar una API que no controlas | Traduce una interfaz a otra en un único sitio. |
| **State** (Estado) | Comportamiento | El comportamiento cambia según el estado | Cada estado sabe qué puede hacer y a dónde puede ir. |
| **Decorator** (Decorador) | Estructura | Añadir responsabilidades sin tocar el objeto | Envuelve el objeto y añade una capa. |
| **Builder** (Constructor) | Creación | Construir objetos con muchos datos | Construye paso a paso, con nombres. |
| **Chain of Responsibility** (Cadena) | Comportamiento | Varios manejadores en orden | Cada eslabón procesa o pasa la petición al siguiente. |
| **Proxy** (Apoderado) | Estructura | Controlar el acceso a un objeto | Un intermediario añade caché, permisos o carga diferida. |
| **Factory Method** (Fábrica) | Creación | Crear sin acoplar el cliente a la clase concreta | La fábrica o el registro decide qué instancia crear. |
| **Facade** (Fachada) | Estructura | Simplificar un subsistema complejo | Una puerta de entrada simple para muchas piezas. |

> Los nombres clásicos están en inglés porque así los vas a encontrar en libros, documentación y código. Entre paréntesis tienes la traducción que usamos al hablar.

---

## Los tres grupos clásicos

El catálogo original (Gang of Four, 1994) agrupa los patrones en tres familias. No hace falta memorizarlas, pero ayudan a ordenar la cabeza:

| Familia | Responde a... | Ejemplos |
|---------|---------------|----------|
| **Creación** | ¿Cómo creo objetos sin acoplarme a sus clases concretas? | Factory Method, Abstract Factory, Builder, Prototype, Singleton |
| **Estructura** | ¿Cómo compongo objetos para que colaboren sin romperse? | Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight |
| **Comportamiento** | ¿Cómo se reparten el trabajo y se comunican los objetos? | Strategy, Observer, State, Command, Template Method, Chain, Iterator, Mediator, Memento, Visitor |

---

## Los patrones, uno a uno

### Strategy (Estrategia) — comportamiento

**Definición sencilla:** define una familia de algoritmos, encapsula cada uno y los hace intercambiables. El cliente usa una interfaz común y no sabe qué implementación hay detrás.

**Analogía:** pagar en una tienda. El cajero siempre hace el mismo gesto ("cobrar"), pero tarjeta, monedero y transferencia lo resuelven de forma distinta.

### ❌ Malo: un `switch` que crece con cada método de pago

```ts
class Checkout {
  pagar(metodo: string, importe: number): void {
    if (metodo === 'tarjeta') {
      // validar tarjeta, llamar a la pasarela...
    } else if (metodo === 'monedero') {
      // validar cuenta, llamar a la pasarela...
    } else if (metodo === 'transferencia') {
      // generar datos bancarios...
    }
    // mañana: Apple Pay... y volvemos a abrir este método
  }
}
```

Cada método nuevo obliga a editar el mismo `switch` y arrastra sus validaciones y sus datos. Es el ejemplo clásico de código que **crece por un eje de variación**.

### ✅ Bueno: cada método de pago es una estrategia

```ts
interface MetodoPago {
  validar(): void
  cobrar(importe: number): void
}

class Tarjeta implements MetodoPago {
  validar(): void { /* ... */ }
  cobrar(importe: number): void { /* ... */ }
}

class Monedero implements MetodoPago {
  validar(): void { /* ... */ }
  cobrar(importe: number): void { /* ... */ }
}

class Checkout {
  constructor(private readonly metodo: MetodoPago) {}

  pagar(importe: number): void {
    this.metodo.validar()
    this.metodo.cobrar(importe)
  }
}
```

En TicketBox, el `switch` de ~80 líneas se convirtió en `PaymentStrategy` + una clase por método + un registro. Añadir Apple Pay fue **un archivo nuevo** y una línea en el registro.

**🚩 Señal de alarma:** un `switch` o una cadena de `if` que compara un "tipo" y que crece cada vez que aparece un caso nuevo.

**Pregunta clave:** ¿cuántas variantes hay hoy y cuántas espero en los próximos seis meses? Si son dos y no van a cambiar, un `if` corto se lee mejor que cinco clases.

**Coste:** una interfaz, una clase por variante y un registro. Un nivel de indirección.

---

### Observer (Observador) — comportamiento

**Definición sencilla:** define una dependencia uno-a-muchos. Cuando el objeto principal cambia, avisa a todos sus suscriptores **sin conocerlos**.

**Analogía:** una emisora de radio y muchos receptores. La emisora no sabe quién la escucha; solo emite. Cada receptor decide qué hacer con la señal.

### ❌ Malo: el servicio conoce a todos los sistemas por su nombre

```ts
class Checkout {
  pagar(): void {
    // ...
    this.email.enviar('Tus entradas...')
    this.analitica.registrar('ticket_purchased', 164.43)
    this.inventario.actualizar('EVT-9', 2)
    this.fidelizacion.sumar('ana@example.com', 164.43)
    this.imprenta.encolar('ORD-2045', 2)
  }
}
```

Añadir antifraude obliga a editar el servicio. Y el servicio, que debería ocuparse de cobrar, termina sabiendo cómo funciona el email, la analítica y la imprenta.

### ✅ Bueno: el servicio publica un evento y los observadores reaccionan

```ts
interface PurchaseObserver {
  onPurchase(evento: PurchaseEvent): void
}

class Checkout {
  constructor(private readonly observers: PurchaseObserver[]) {}

  pagar(): void {
    // ... cobrar y actualizar el pedido
    const evento: PurchaseEvent = {
      orderId: 'ORD-2045',
      eventId: 'EVT-9',
      amount: 164.43,
      method: 'card',
    }
    for (const observer of this.observers) {
      observer.onPurchase(evento)
    }
  }
}
```

Antifraude es un observador nuevo. El servicio no cambia. En TicketBox, el registro de observadores es el "composition root": ahí se decide quién escucha.

**🚩 Señal de alarma:** cinco llamadas seguidas a sistemas distintos después de la lógica principal; el servicio importa diez módulos que no tienen que ver con su responsabilidad.

**Pregunta clave:** ¿el emisor necesita conocer a los receptores, o solo necesita anunciar que ha pasado algo?

**Ojo:** Observer es "dispara y olvida". Si un observador lanza una excepción, los siguientes no se ejecutan. Decide si cada aviso debe ser independiente; si el orden y la transacción son críticos, un pipeline explícito se lee mejor.

**Coste:** un evento, una interfaz, un observador por receptor y un registro.

---

### Adapter (Adaptador) — estructura

**Definición sencilla:** convierte la interfaz de una clase en otra que el cliente espera. Sirve para integrar código que **no controlas** sin contaminar el dominio con sus nombres y sus rarezas.

**Analogía:** un adaptador de enchufe de viaje. El enchufe de la pared no cambia; tu portátil tampoco. El adaptador traduce entre los dos.

### ❌ Malo: la lógica habla el idioma de la pasarela de 2011

```ts
const legacy = new PaymentsGatewayV1()

const respuesta = legacy.execute({
  AMOUNT_IN_CENTS: Math.round(total * 100),
  CURRENCY_CODE: 'EUR',
  OPERATION: 'SALE',
  CARD_TOKEN: token,
  META: { /* ... */ },
})
```

Los nombres en mayúsculas, los códigos numéricos y los campos raros se filtran por todo el servicio. Si mañana cambia el proveedor, hay que buscar cada uso.

### ✅ Bueno: una sola traducción, en un adaptador

```ts
interface ChargeGateway {
  charge(request: ChargeRequest): ChargeResult
}

class LegacyGatewayAdapter implements ChargeGateway {
  constructor(private readonly legacy: PaymentsGatewayV1) {}

  charge(request: ChargeRequest): ChargeResult {
    return this.legacy.execute({
      AMOUNT_IN_CENTS: request.amountInCents,
      CURRENCY_CODE: request.currency,
      OPERATION: request.operation,
      CARD_TOKEN: request.cardToken,
      ACCOUNT_ID: request.accountId,
      META: request.meta,
    })
  }
}
```

El servicio habla de `ChargeRequest` y `ChargeResult`. El adaptador es el único que sabe que la pasarela usa `AMOUNT_IN_CENTS` y que el código `99` significa "no responde".

**🚩 Señal de alarma:** nombres en mayúsculas, códigos numéricos o campos de una librería externa dentro de tu lógica de negocio; la misma traducción copiada en dos sitios.

**Pregunta clave:** ¿mi dominio habla su idioma o el del proveedor?

**Coste:** una interfaz (el puerto) y un adaptador. Es de los patrones más baratos y más rentables cuando hay una API externa.

---

### State (Estado) — comportamiento

**Definición sencilla:** permite que un objeto cambie su comportamiento cuando cambia su estado. Cada estado sabe qué operaciones permite y a qué estado se pasa.

**Analogía:** un semáforo. El mismo botón de "avanzar" produce una acción distinta según esté en rojo, ámbar o verde.

### ❌ Malo: las mismas comprobaciones de estado en todos los métodos

```ts
class Pedido {
  estado: 'created' | 'reserved' | 'paid' | 'issued' = 'created'

  reservar(): void {
    if (this.estado !== 'created') throw new Error('No se puede reservar')
    this.estado = 'reserved'
  }

  pagar(): void {
    if (this.estado !== 'reserved') throw new Error('No se puede pagar')
    this.estado = 'paid'
  }

  emitir(): void {
    if (this.estado !== 'paid') throw new Error('No se pueden emitir')
    this.estado = 'issued'
  }
}
```

Cada método repite la misma idea ("¿en qué estado estoy?") y el conocimiento de las transiciones está repartido por toda la clase. Un estado nuevo obliga a tocar varios métodos.

### ✅ Bueno: una tabla de transiciones (o clases de estado)

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

  reservar(): void { this.mover('reserve') }
  pagar(): void { this.mover('markPaid') }

  private mover(accion: Accion): void {
    const siguiente = TRANSICIONES[this.estado][accion]
    if (!siguiente) throw new Error(`Transición no permitida: ${accion}`)
    this.estado = siguiente
  }
}
```

**State frente a tabla:** si cada estado solo decide **qué transiciones son legales**, una tabla es más simple. Si cada estado tiene **comportamiento propio** (cálculos, permisos, pasos distintos) y el conjunto crece, entonces sí compensa una clase por estado.

En TicketBox se usó la tabla: seis estados estables y transiciones que se leen de un vistazo. No se añadieron seis clases con métodos que lanzan excepciones.

**🚩 Señal de alarma:** `if (estado === ...)` repetido en varios métodos; métodos que existen solo para lanzar "no se puede en este estado".

**Pregunta clave:** ¿cada estado hace cosas distintas o solo permite transiciones distintas?

**Coste:** una tabla (barato) o una interfaz y una clase por estado (más caro, pero más expresivo si hay comportamiento real).

---

### Decorator (Decorador) — estructura

**Definición sencilla:** añade responsabilidades a un objeto **envolviéndolo** con otro objeto que implementa la misma interfaz. Las capas se pueden combinar.

**Analogía:** vestirse por capas. La persona es la misma, pero cada capa añade algo (camiseta, abrigo, chubasquero). Puedes quitarte una sin cambiar la persona.

### ❌ Malo: el método hace logging, reintentos, caché y métricas

```ts
class Gateway {
  charge(request: ChargeRequest): ChargeResult {
    if (this.withLogging) console.log('cobrando...')

    let result = this.legacy.execute(/* ... */)

    if (this.withRetry) {
      // reintentar...
    }

    if (this.withCache) {
      // cachear...
    }

    if (this.withMetrics) {
      // medir...
    }

    return result
  }
}
```

Cada preocupación nueva añade un `if` y hace el método más difícil de leer. Además, todas las combinaciones posibles terminan dentro de la misma clase.

### ✅ Bueno: cada preocupación es una capa que envuelve a la anterior

```ts
interface Gateway {
  charge(request: ChargeRequest): ChargeResult
}

class LoggingGateway implements Gateway {
  constructor(private readonly inner: Gateway) {}

  charge(request: ChargeRequest): ChargeResult {
    console.log(`[gateway] ${request.operation} ${request.amountInCents}`)
    return this.inner.charge(request)
  }
}

class RetryGateway implements Gateway {
  constructor(private readonly inner: Gateway, private readonly attempts: number) {}

  charge(request: ChargeRequest): ChargeResult {
    let result = this.inner.charge(request)
    let attempt = 1
    while (!result.ok && result.retryable && attempt < this.attempts) {
      attempt += 1
      result = this.inner.charge(request)
    }
    return result
  }
}

const gateway = new RetryGateway(
  new LoggingGateway(new LegacyGatewayAdapter(legacy)),
  3,
)
```

En TicketBox **no** se usó: solo había logging y reintento, y un método privado `callGateway()` era suficiente. Decorator compensa cuando hay **tres o más** preocupaciones que se combinan de formas distintas.

**🚩 Señal de alarma:** un método que hace logging + reintentos + caché + métricas + permisos; clases que crecen con cada "y también".

**Pregunta clave:** ¿necesito combinar estas capas de formas distintas, o siempre van juntas?

**Coste:** una interfaz y un decorador por preocupación. Si solo hay dos preocupaciones fijas, un helper es más simple.

---

### Builder (Constructor) — creación

**Definición sencilla:** separa la construcción de un objeto complejo de su representación. Permite construir paso a paso, con nombres y sin depender de una lista de parámetros posicionales.

**Analogía:** pedir un menú. Eliges primero, segundo, bebida y postre; no das catorce números en un orden que nadie recuerda.

### ❌ Malo: catorce parámetros posicionales

```ts
const peticion = new CheckoutRequest(
  'ORD-2045', 'EVT-9', 'ana@example.com',
  2, 'vip', 10, false,
  'card', 'token-123', null, null, null, null, null,
)
```

En la llamada no se sabe qué es `10`, ni `false`, ni el tercer `null`. Y si mañana hay un campo opcional más, todos los llamadores se enteran.

### ✅ Bueno: un objeto con nombres (o un Builder si de verdad hace falta)

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

En TicketBox se eliminó `createCheckoutRequest()`: un literal con nombres era suficiente. Un Builder sería útil si hubiera **muchos campos opcionales**, **varios puntos de llamada** y **reglas de construcción** (por ejemplo, validar al final).

**🚩 Señal de alarma:** constructores con 10+ parámetros; llamadas llenas de `null, null, null`; el orden de los argumentos importa y nadie lo recuerda.

**Pregunta clave:** ¿el problema es construir el objeto o solo que la lista de parámetros es larga? Si es lo segundo, un objeto de parámetros basta.

**Coste:** un Builder y su interfaz. Es fácil pasarse: para un solo punto de llamada, un literal es más claro.

---

### Chain of Responsibility (Cadena de responsabilidad) — comportamiento

**Definición sencilla:** pasa una petición por una cadena de manejadores. Cada uno decide si la procesa, si la rechaza o si la pasa al siguiente.

**Analogía:** los controles de un aeropuerto: documentación, seguridad y puerta. Si fallas en uno, no sigues; si pasas, avanzas al siguiente.

### ❌ Malo: una función de validación con veinte `if`

```ts
function validar(request: CheckoutRequest): void {
  if (!request.customerEmail.includes('@')) throw new Error('Email inválido')
  if (request.quantity <= 0) throw new Error('Cantidad inválida')
  if (request.quantity > 10) throw new Error('Máximo 10 entradas')
  if (request.discountPercentage < 0) throw new Error('Descuento inválido')
  if (request.discountPercentage > 100) throw new Error('Descuento inválido')
  if (request.ticketType === 'vip' && request.quantity > 4) throw new Error('Máximo 4 VIP')
  if (request.isGift && request.quantity > 1) throw new Error('Regalo de una entrada')
  // ...
}
```

Se lee de arriba abajo, pero crece sin control y mezcla reglas de negocio, mensajes y orden.

### ✅ Bueno: cada regla es un eslabón

```ts
interface Validator {
  validate(request: CheckoutRequest): void
}

class EmailValidator implements Validator {
  validate(request: CheckoutRequest): void {
    if (!request.customerEmail.includes('@')) throw new Error('Email inválido')
  }
}

class QuantityValidator implements Validator {
  validate(request: CheckoutRequest): void {
    if (request.quantity <= 0) throw new Error('Cantidad inválida')
    if (request.quantity > 10) throw new Error('Máximo 10 entradas')
  }
}

class ValidatorChain {
  constructor(private readonly validators: Validator[]) {}

  validate(request: CheckoutRequest): void {
    for (const validator of this.validators) {
      validator.validate(request)
    }
  }
}
```

En TicketBox **no** se usó: seis reglas simples y el orden importa (los tests fijan el primer mensaje). Una cadena añadía clases y escondía ese orden. Compensa cuando las reglas se configuran desde fuera, cambian por cliente o forman un pipeline de verdad.

**🚩 Señal de alarma:** una validación de 100 líneas; reglas que se activan o desactivan según el cliente; el orden de los `if` es parte del comportamiento y nadie lo documenta.

**Pregunta clave:** ¿las reglas cambian por cliente o solo son seis y estables?

**Coste:** una interfaz, un manejador por regla y una cadena. Es bastante ceremonia para seis `if`.

---

### Proxy (Apoderado) — estructura

**Definición sencilla:** sustituye un objeto por otro que controla el acceso al original. Puede añadir caché, permisos, carga diferida o registro **sin que el cliente cambie**.

**Analogía:** el portero de un edificio. El edificio es el mismo, pero el portero decide quién entra, cuándo y qué queda registrado.

### ❌ Malo: la caché y los permisos dentro del objeto real

```ts
class Checkout {
  private cache = new Map<string, CheckoutResult>()

  ejecutar(request: CheckoutRequest): CheckoutResult {
    const cached = this.cache.get(request.orderId)
    if (cached) return cached

    if (!this.permisos.puede(request.customerEmail)) {
      throw new Error('Sin permiso')
    }

    // ... lógica de cobro
    const result = this.cobrar(request)
    this.cache.set(request.orderId, result)
    return result
  }
}
```

El caso de uso mezcla negocio con infraestructura. Y si otro cliente quiere el mismo comportamiento, hay que repetirlo.

### ✅ Bueno: un proxy que envuelve el caso de uso

```ts
interface CheckoutUseCase {
  execute(request: CheckoutRequest): CheckoutResult
}

class CachedCheckout implements CheckoutUseCase {
  constructor(
    private readonly inner: CheckoutUseCase,
    private readonly cache: Map<string, CheckoutResult>,
  ) {}

  execute(request: CheckoutRequest): CheckoutResult {
    const cached = this.cache.get(request.orderId)
    if (cached) return cached

    const result = this.inner.execute(request)
    this.cache.set(request.orderId, result)
    return result
  }
}

const checkout = new CachedCheckout(new CheckoutService(), cache)
```

En TicketBox se descartó: la caché son tres líneas y **cachear el cobro** sería peligroso (riesgo de doble cobro). Se mantuvo a nivel de **resultado del checkout**, no de la pasarela.

**🚩 Señal de alarma:** caché, permisos o registro metidos dentro del objeto real; el cliente tiene que saber si usa la versión "con caché" o "sin caché".

**Pregunta clave:** ¿quiero controlar el acceso sin cambiar al cliente?

**Coste:** una interfaz y un proxy. Cuidado con los efectos secundarios: no todo se puede cachear.

---

### Factory Method (Método fábrica) — creación

**Definición sencilla:** define una interfaz para crear un objeto, pero deja que la fábrica (o un registro) decida qué clase concreta instanciar. El cliente pide "un método de pago", no `new Tarjeta()`.

**Analogía:** una fábrica de coches. Pides un coche; no atornillas el motor ni eliges la cadena de montaje. La fábrica decide el modelo según el pedido.

### ❌ Malo: el cliente conoce todas las clases concretas

```ts
class Checkout {
  pagar(tipo: string, importe: number): void {
    if (tipo === 'tarjeta') {
      new Tarjeta().cobrar(importe)
    } else if (tipo === 'monedero') {
      new Monedero().cobrar(importe)
    } else if (tipo === 'transferencia') {
      new Transferencia().cobrar(importe)
    }
  }
}
```

Cada método nuevo vuelve a tocar el cliente y el `switch`. La creación está repartida.

### ✅ Bueno: un registro (fábrica simple) decide qué instancia usar

```ts
const metodos = new Map<string, MetodoPago>([
  ['tarjeta', new Tarjeta()],
  ['monedero', new Monedero()],
  ['transferencia', new Transferencia()],
])

class Checkout {
  pagar(tipo: string, importe: number): void {
    const metodo = metodos.get(tipo)
    if (!metodo) throw new Error(`Método no soportado: ${tipo}`)
    metodo.cobrar(importe)
  }
}
```

En TicketBox, el registro de estrategias ya hace esta función. No se añadió una jerarquía de fábricas: habría sido una capa de más.

**No confundas:** una **fábrica simple** (una función o un registro que crea) no es lo mismo que **Factory Method** (las subclases deciden qué crear) ni **Abstract Factory** (familias de objetos relacionados). Para este proyecto, la fábrica simple era suficiente.

**🚩 Señal de alarma:** `new` repartidos por el código; un `switch` que solo sirve para instanciar clases; el cliente importa diez implementaciones concretas.

**Pregunta clave:** ¿quién debe decidir la clase concreta: el cliente o la configuración?

**Coste:** una fábrica o un registro. Si solo hay un `new` en un sitio, no hace falta.

---

### Facade (Fachada) — estructura

**Definición sencilla:** ofrece una interfaz simple para un conjunto de clases o un subsistema complejo. El cliente pide una operación y la fachada coordina las piezas.

**Analogía:** la recepción de un hotel. Pides "una habitación para dos noches" y no hablas con limpieza, cocina y mantenimiento.

### ❌ Malo: el cliente coordina cinco servicios

```ts
class Cliente {
  comprar(): void {
    const stock = new StockService()
    const pago = new PaymentService()
    const factura = new InvoiceService()
    const email = new EmailService()
    const analitica = new AnalyticsService()

    stock.reservar('EVT-9', 2)
    pago.cobrar(164.43)
    factura.generar('ORD-2045')
    email.enviar('ana@example.com')
    analitica.registrar('ticket_purchased')
  }
}
```

Cada cliente tiene que saber el orden, los servicios y sus detalles. Si cambia el flujo, se cambia en todos los clientes.

### ✅ Bueno: una fachada que coordina el caso de uso

```ts
class CheckoutFacade {
  constructor(
    private readonly stock: StockService,
    private readonly pago: PaymentService,
    private readonly email: EmailService,
  ) {}

  comprar(request: CheckoutRequest): CheckoutResult {
    this.stock.reservar(request.eventId, request.quantity)
    const resultado = this.pago.cobrar(request)
    this.email.enviar(request.customerEmail)
    return resultado
  }
}
```

En TicketBox, `CheckoutService` **ya es** la fachada/orquestador. Añadir otra capa encima sería indirección gratuita. Una fachada no elimina la complejidad: la esconde detrás de una puerta de entrada.

**🚩 Señal de alarma:** el cliente instancia y coordina cinco servicios para una sola operación; el mismo flujo copiado en varios clientes.

**Pregunta clave:** ¿el cliente necesita conocer todo el subsistema o solo una operación?

**Coste:** una clase fachada. Si ya tienes un servicio que orquesta, no añadas otra.

---

## Otros patrones que oirás

No hace falta aplicarlos, pero conviene reconocerlos:

| Patrón | Familia | En una frase | Cuándo mirarlo |
|--------|---------|--------------|----------------|
| **Template Method** | Comportamiento | El esqueleto del algoritmo está en la base; los pasos cambian en las subclases. | Varios procesos casi iguales. |
| **Command** | Comportamiento | Encapsula una petición como objeto. | Deshacer, colas, auditoría. |
| **Composite** | Estructura | Trata un árbol y una hoja de la misma forma. | Jerarquías parte-todo. |
| **Iterator** | Comportamiento | Recorre una colección sin conocer su estructura interna. | Colecciones propias. |
| **Mediator** | Comportamiento | Centraliza la comunicación entre muchos objetos. | Objetos que se hablan entre sí. |
| **Memento** | Comportamiento | Guarda y restaura el estado de un objeto. | Deshacer, instantáneas. |
| **Visitor** | Comportamiento | Añade operaciones a una jerarquía sin tocarla. | Muchas operaciones sobre tipos estables. |
| **Singleton** | Creación | Una única instancia compartida. | Configuración global; cuidado con los tests. |
| **Prototype** | Creación | Clona un objeto existente. | Crear es caro o complejo. |
| **Abstract Factory** | Creación | Crea familias de objetos relacionados. | Varias familias (tema claro/oscuro). |
| **Bridge** | Estructura | Separa la abstracción de su implementación. | Dos ejes de variación. |
| **Flyweight** | Estructura | Comparte estado inmutable entre muchos objetos. | Muchos objetos con datos repetidos. |

> No todo lo que llamamos "patrón" es del catálogo GoF. **MVC**, **capas**, **hexagonal**, **microservicios** o **Repository** son patrones arquitectónicos o de empresa. Son otra escala: no compiten con Strategy u Observer.

---

## Qué se aplicó en TicketBox (clase 5)

| Patrón | Decisión | Por qué |
|--------|----------|---------|
| **Strategy** | ✅ Aplicado | Cuatro métodos de pago y Apple Pay confirmado; el `switch` crecía con cada caso. |
| **Observer** | ✅ Aplicado | Cinco sistemas de aviso y antifraude; el servicio no debía conocerlos. |
| **Adapter** | ✅ Aplicado | La traducción a la pasarela legacy estaba duplicada en tarjeta y monedero. |
| **State** | ❌ Descartado | Seis estados estables; una tabla de transiciones era más simple. |
| **Decorator** | ❌ Descartado | Solo logging y reintento; un método privado bastaba. |
| **Builder** | ❌ Descartado | Un solo punto de llamada; un literal con nombres era suficiente. |
| **Chain of Responsibility** | ❌ Descartado | Seis validaciones simples y el orden importa. |
| **Proxy** | ❌ Descartado | La caché son tres líneas y cachear el cobro era peligroso. |
| **Factory Method** | ❌ Descartado | El registro de estrategias ya hace la selección. |
| **Facade** | ❌ Descartado | `CheckoutService` ya era la fachada/orquestador. |

> Esta tabla es el resumen de la decisión; el detalle está en el `ANALISIS.md` del proyecto TicketBox.

---

## Cómo elegir un patrón

1. **Escribe el problema en lenguaje llano**, sin nombrar patrones. "Para añadir un método de pago hay que editar un `switch` de 80 líneas."
2. **Pregunta cuántos casos hay hoy y cuántos esperas en seis meses.** El patrón depende de por dónde va a cambiar el código, no de lo feo que sea.
3. **Pregunta qué se toca al añadir un caso nuevo.** Si la respuesta es "un archivo nuevo", vas bien; si es "cinco archivos", hay acoplamiento.
4. **Empieza por la solución más simple:** una función, una tabla, un objeto con nombres. Si eso basta, no metas una jerarquía.
5. **Aplica un patrón cada vez**, con los tests en verde. Un paso pequeño, tests, siguiente paso.
6. **Cuenta el coste:** archivos nuevos, interfaces, nombres, saltos para leer el flujo. Si no puedes explicar qué problema evita, no lo pongas.
7. **Pregunta qué descartas y por qué.** Un "aquí no compensa porque solo hay dos casos estables" es una respuesta de senior.

---

## Señales de que estás metiendo demasiados patrones

- Hay una clase por método, y cada clase tiene una sola línea.
- Hay interfaces con una única implementación y ningún doble de test.
- Hay fábricas que solo hacen `new` y no deciden nada.
- Hay capas que solo pasan datos de una a otra.
- Para entender un cobro hay que abrir ocho archivos.
- Los nombres suenan a catálogo ("AbstractStrategyFactoryProvider") y no a negocio.
- Pasas más tiempo dibujando el diagrama que resolviendo el problema.

Eso no es diseño: es **patternitis**. Un patrón aplicado sin necesidad es una violación de YAGNI y de KISS.

---

## Cómo se relacionan con SOLID, DRY, KISS y YAGNI

| Principio | Qué aporta a los patrones |
|-----------|---------------------------|
| **SRP** | Cada estrategia, observador o adaptador tiene una responsabilidad pequeña. |
| **OCP** | Strategy, Observer y Factory permiten añadir casos sin tocar el núcleo. |
| **LSP** | Un patrón con herencia solo funciona si los subtipos respetan el contrato. |
| **ISP** | Mejor interfaces pequeñas (`PaymentStrategy`, `PurchaseObserver`) que una gorda. |
| **DIP** | El dominio depende de `ChargeGateway`, no de la pasarela concreta. |
| **DRY** | El adaptador evita repetir la traducción; el registro evita repetir la selección. |
| **KISS** | Una tabla de transiciones puede sustituir a seis clases de estado. |
| **YAGNI** | No apliques un patrón para un caso que todavía no existe. |

> La pregunta que los une: *"¿esto hace que el código sea más fácil de cambiar dentro de seis meses?"*. Si la respuesta es no, el patrón sobra.

---

## Checklist rápida

Cuando veas un problema, pregúntate:

1. **Un `switch`/`if` por tipo que crece** → **Strategy** (si son algoritmos) o **State** (si son estados).
2. **Muchos sistemas a los que avisar** → **Observer**.
3. **Una API externa con nombres raros** → **Adapter** (y **Facade** si además quieres simplificar).
4. **Un objeto con demasiados parámetros** → **Builder** u objeto de parámetros.
5. **Añadir comportamiento alrededor de un objeto** → **Decorator** o **Proxy**.
6. **Un pipeline de validaciones o manejadores** → **Chain of Responsibility**.
7. **Creación que varía** → **Factory Method** o un registro.
8. **Un cliente que coordina cinco servicios** → **Facade**.

> Regla de oro: un patrón no es un punto extra; es una **deuda de indirección** que contraes a cambio de flexibilidad. Si no vas a cobrar esa flexibilidad, no la contrates.
