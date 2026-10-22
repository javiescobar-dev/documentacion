Clean Architecture no es una carpeta llamada `domain` ni una lista de capas. Es una forma de organizar el código para que los **cambios importantes sean locales, comprensibles y verificables**. La idea central es sencilla: las **políticas** (qué debe ocurrir) no deberían depender de los **detalles** (cómo lo conectamos).

La arquitectura se nota cuando cambia el negocio. Si una regla nueva obliga a tocar el controlador, la consulta SQL, el componente de la interfaz y un proceso en segundo plano, el problema no es la regla: es que no hay límites claros. Si la regla vive en un caso de uso, se cambia ahí, se conserva el contrato y se ejecutan sus pruebas.

> **Definición operativa:** arquitectura es dar forma al sistema para que sus cambios importantes sean localizables y verificables. Clean Architecture es una herramienta para conseguirlo, no el objetivo final.

---

## Resumen en una frase

| Concepto | En una frase |
|----------|--------------|
| **Arquitectura** | Dar forma al sistema para que los cambios importantes sean locales y verificables. |
| **Política** | Regla de negocio, invariante o caso de uso: **qué** debe ocurrir. |
| **Detalle** | HTTP, SQL, framework, SDK: **cómo** lo conectamos. |
| **Límite** | Frontera explícita donde un conocimiento no se mezcla con otro. |
| **Acoplamiento** | Cuánto depende una pieza de otra. |
| **Cohesión** | Juntar lo que cambia por la misma razón. |
| **Dirección de dependencia** | Las dependencias del código apuntan hacia las políticas, no hacia los detalles. |
| **Puerto** | Contrato con nombre en un límite. |
| **Adaptador** | Traducción tecnológica de un puerto. |
| **Caso de uso** | Intención de la aplicación, no una operación técnica. |
| **Entidad** | Identidad + estado + comportamiento + invariantes. |
| **Objeto de valor** | Valor inmutable con reglas propias. |
| **Raíz de composición** | Único lugar donde se eligen las piezas concretas. |
| **Proporcionalidad** | La inversión arquitectónica debe compensar la complejidad del problema. |

---

## Mapa del documento

| Bloque | Qué responde |
|--------|--------------|
| **1. El problema** | Por qué una regla de una línea puede afectar a media aplicación. |
| **2. Políticas y detalles** | Qué debe vivir en el núcleo y qué en la periferia. |
| **3. Acoplamiento y cohesión** | Cómo detectar mezclas y cómo separarlas por razón de cambio. |
| **4. Los límites** | Dónde poner las fronteras y cómo saber si son reales. |
| **5. Clean, Hexagonal y Onion** | Tres mapas del mismo territorio. |
| **6. Vocabulario** | Dominio, caso de uso, puerto, adaptador, entidad y objeto de valor. |
| **7. Dirección de dependencias** | Por qué el flujo de ejecución y el flujo del código no son lo mismo. |
| **8. Dominio** | Entidades, identidad, invariantes y objetos de valor. |
| **9. Casos de uso** | Intenciones, coordinación y contratos de entrada/salida. |
| **10. Ports** | Contratos diseñados desde la necesidad interior. |
| **11. Adapters** | Traducción tecnológica y límites con el exterior. |
| **12. Composición** | Inyección de dependencias y raíz de composición. |
| **13. Errores y validación** | Entrada mal formada, resultado esperado y error inesperado. |
| **14. Tests** | Qué probar en cada zona y qué significa “comportamiento observable”. |
| **15. Proporcionalidad** | Cuándo simplificar y cuándo invertir en límites. |

---

## 1. El problema: una regla pequeña, un impacto grande

Imagina que el negocio dice:

> “Envío gratuito para clientes premium a partir de 50 €.”

La regla cabe en una línea. La pregunta interesante no es cómo escribirla, sino **cuántos lugares hay que tocar** para que funcione en toda la aplicación.

### El test del cambio

Antes de diseñar, hazte tres preguntas:

| Pregunta | Qué estás comprobando |
|----------|------------------------|
| **¿Cuántos lugares cambian si cambia el negocio?** | Si la regla está duplicada en controladores, consultas, componentes y tareas, el cambio es caro. |
| **¿Cambiar PostgreSQL altera las reglas?** | Si la respuesta es sí, la base de datos está demasiado cerca del negocio. |
| **¿Podemos probar sin levantar toda la infraestructura?** | Si necesitas HTTP, base de datos y proveedor de pago para probar una regla, el límite no existe. |

### Sin límites y con límites

**Sin límites:** la regla del envío gratuito aparece en el controlador, en una consulta SQL, en el componente de la interfaz y en un proceso nocturno. Cada cambio de negocio se convierte en una búsqueda por todo el proyecto.

**Con límites:** la regla vive en un caso de uso o en el dominio. Se modifica ahí, se conserva el contrato con el exterior y se ejecutan sus pruebas. El resto del sistema no necesita saber cómo se calcula.

### El código puede funcionar y estar mal preparado

El acoplamiento no siempre rompe el sistema hoy. Muchas veces el software funciona, pasa la demo y sale a producción. El problema aparece **mañana**, cuando hay que cambiar una regla y el cambio obliga a entender diez archivos.

> Una arquitectura no se juzga por lo que permite escribir hoy, sino por lo que permite cambiar dentro de seis meses.

### Qué no es Clean Architecture

- No es tener carpetas llamadas `domain`, `application` e `infrastructure`.
- No es usar interfaces para todo.
- No es meter un contenedor de dependencias.
- No es copiar un diagrama de círculos.

Es una **dirección de dependencias** y una **separación de conocimientos**. Las carpetas son una consecuencia, no el objetivo.

---

## 2. Políticas y detalles

### Las políticas gobiernan; los detalles se adaptan

| Políticas | Detalles |
|-----------|----------|
| Qué debe ocurrir | Cómo lo conectamos |
| Reglas de negocio | HTTP |
| Invariantes | PostgreSQL |
| Casos de uso | Frameworks y proveedores |
| Decisiones estables | Decisiones reemplazables |

La política es lo que hace que el negocio sea ese negocio. El detalle es lo que puede cambiar sin cambiar la política: un framework web, una base de datos, un proveedor de pago, una interfaz de usuario.

### Cinco decisiones compiten dentro de una función

Mira este controlador. Parece inocente, pero dentro conviven cinco decisiones distintas:

```ts
app.post('/orders', async (req, res) => {
  const { customerId, items } = req.body

  if (!customerId || !items?.length) {
    return res.status(400).json({ error: 'Datos inválidos' })
  }

  let total = 0
  for (const item of items) {
    total += item.price * item.quantity
  }

  const charge = await stripe.charges.create({
    amount: Math.round(total * 100),
    currency: 'eur',
    customer: customerId,
  })

  await db.query(
    'INSERT INTO orders (customer_id, total, payment_id) VALUES ($1, $2, $3)',
    [customerId, total, charge.id],
  )

  res.status(201).json({ orderId: charge.id })
})
```

| Decisión | Pregunta | Dónde debería vivir |
|----------|----------|---------------------|
| **HTTP / transporte** | ¿Cómo entra y sale la petición? | Adaptador de entrada |
| **Validación** | ¿Los datos tienen la forma correcta? | Adaptador (estructura) y dominio (semántica) |
| **Precio** | ¿Cuánto cuesta el pedido? | Dominio |
| **Pago** | ¿Qué proveedor cobra y cómo? | Puerto + adaptador |
| **Persistencia** | ¿Qué tabla y qué columnas? | Adaptador de salida |

El problema no es que el código sea largo. El problema es que **cada cambio de negocio aterriza en la misma función**.

### Una separación razonable

```ts
// adapter/http/place-order-controller.ts
app.post('/orders', async (req, res) => {
  const input = parsePlaceOrderInput(req.body)
  const output = await placeOrder.execute(input)
  res.status(output.ok ? 201 : 422).json(toHttp(output))
})

// application/place-order.ts
class PlaceOrder {
  constructor(
    private readonly orders: OrderRepository,
    private readonly payments: PaymentGateway,
  ) {}

  async execute(input: PlaceOrderInput): Promise<PlaceOrderOutput> {
    const order = Order.create(input)
    const payment = await this.payments.charge(order.total(), input.customerId)

    if (!payment.approved) {
      return { ok: false, reason: 'payment_declined' }
    }

    order.markPaid()
    await this.orders.save(order)
    return { ok: true, orderId: order.id.value }
  }
}
```

El controlador sabe de HTTP. El caso de uso sabe de pedidos y pagos. El dominio sabe de reglas. El adaptador de pago sabe de Stripe. Cada pieza tiene un motivo de cambio distinto.

## 3. Acoplamiento y cohesión

Acoplamiento y cohesión son las dos caras de la misma pregunta: **¿qué piezas cambian juntas?** Si dos decisiones cambian por motivos distintos y viven juntas, cada cambio paga el coste de la otra.

### El acoplamiento adopta varias formas

No solo se acopla el código cuando importa una clase. Hay cuatro formas habituales:

| Tipo | Ejemplo | Cómo se filtra |
|------|---------|----------------|
| **Tipos** | `Request` y `Response` de Express | El adaptador construye la entrada y traduce la salida. |
| **Protocolos** | La respuesta rara del SDK de pago | El puerto devuelve un resultado propio del negocio. |
| **Datos** | Nombres de tablas y columnas | El repositorio mapea entre el dominio y la base de datos. |
| **Semántica** | ¿Euros o céntimos? | Un objeto de valor (`Money`) explicita la unidad. |

El acoplamiento semántico es el más traicionero: dos piezas pueden compilar y hasta funcionar, pero una habla de euros y la otra de céntimos. El tipo `number` no te salva.

### Cohesión: juntar lo que cambia por la misma razón

La cohesión no es “meter todo lo relacionado en la misma clase”. Es juntar las decisiones que **cambian por el mismo motivo**.

**Mezcla accidental:**

> Transporte + reglas + pago + SQL

Cada cambio de negocio, de proveedor o de base de datos aterriza en la misma función.

**Límites intencionales:**

> Cada decisión vive junto a las decisiones que comparten su motivo de cambio.

El controlador cambia si cambia HTTP. El caso de uso cambia si cambia el flujo de la aplicación. El dominio cambia si cambia el negocio. El adaptador cambia si cambia el proveedor.

---

## 4. Los límites: extraer por razón de cambio

### No extraer por tamaño, sino por razón de cambio

Una función de 300 líneas puede ser un problema, pero el tamaño no es el criterio principal. El criterio es: **¿qué cambia junto y por qué?**

La refactorización persigue cuatro límites:

```
[Adaptador HTTP] -> [Caso de uso PlaceOrder] -> [Dominio Order]
                              |
                         [Puertos] <- [Adaptadores]
```

| Límite | Qué protege |
|--------|-------------|
| **Adaptador HTTP** | El protocolo, el formato de entrada y los códigos de respuesta. |
| **Caso de uso PlaceOrder** | La intención de la aplicación y el orden de la operación. |
| **Dominio Order** | Las invariantes y las reglas del pedido. |
| **Puertos + adaptadores** | La colaboración con tecnologías externas. |

No extraemos una clase porque tenga muchas líneas. La extraemos porque tiene **otro motivo para cambiar**.

### Los límites explícitos reducen el contexto necesario

Sin límites, para cambiar la regla del envío gratuito hay que entender el controlador, la consulta, el componente y el proceso nocturno. Con límites, basta con entender el caso de uso y su contrato.

Un límite bien puesto hace que el cambio sea **local**. Un límite mal puesto solo añade una capa que hay que atravesar.

### No inspecciones el nombre: inspecciona la dependencia

Que una clase se llame `OrderRepository` no significa que sea un puerto. Que se llame `OrderHelper` no significa que no contenga dominio. Mira las dependencias:

1. **¿Dónde viven las reglas?** ¿Están en el dominio o repartidas por controladores y consultas?
2. **¿Quién conoce tecnologías concretas?** ¿Solo los adaptadores o también el núcleo?
3. **¿Hacia dónde apuntan los imports y los contratos?** ¿El núcleo importa Express, Stripe o SQL?

### Un test mental para cada dependencia

Cada vez que una pieza importa otra, pregúntate:

| Pregunta | Si la respuesta es “sí”... |
|----------|----------------------------|
| **¿Es una regla o un mecanismo?** | Si es un mecanismo, probablemente va en la periferia. |
| **¿Puede cambiar sin cambiar el negocio?** | Si puede, no debería arrastrar al núcleo. |
| **¿Filtra tipos externos al núcleo?** | Si el dominio importa tipos de HTTP, SQL o un SDK, hay fuga. |

> Un límite no es una carpeta. Es una dependencia que decides no permitir.

---

## 5. Clean, Hexagonal y Onion

Clean Architecture, Hexagonal y Onion son **tres mapas del mismo territorio**. Separar el núcleo de los mecanismos externos, y hacer que las dependencias apunten hacia dentro.

| Enfoque | Idea central | Vocabulario |
|---------|--------------|-------------|
| **Hexagonal** | Dentro y fuera; la aplicación conversa con el exterior mediante puertos y adaptadores. | Puerto de entrada, puerto de salida, adaptador. |
| **Onion** | El acoplamiento apunta al centro. | Modelo de dominio, servicios de aplicación, UI/infraestructura. |
| **Clean** | Distingue políticas y mecanismos. | Entidades, casos de uso, adaptadores, frameworks. |

### Hexagonal

```
[Adaptador HTTP] -> (Puerto de entrada) -> [Aplicación] -> (Puerto de salida) -> [Adaptador SQL]
```

El hexágono representa **dentro y fuera**. El número seis no es importante; es una forma de dibujar que hay varios puntos de entrada y salida.

- El adaptador HTTP llama a la aplicación.
- La aplicación pide colaboración a un puerto de salida.
- El adaptador SQL implementa ese puerto.

### Onion

```
UI · infraestructura · tests
        Servicios de aplicación
            Modelo de dominio
```

Las capas externas dependen de las internas. El dominio no conoce la interfaz, la base de datos ni los tests. El acoplamiento apunta al centro.

### Clean

```
Frameworks y drivers
    Adaptadores de interfaz
        Casos de uso
            Entidades
```

Clean distingue **políticas** (entidades y casos de uso) de **mecanismos** (adaptadores, frameworks, base de datos). Las dependencias apuntan hacia las políticas.

### La idea común

Las tres arquitecturas comparten lo importante:

- El núcleo no depende de la infraestructura.
- El exterior depende del núcleo, no al revés.
- Los límites se expresan con contratos.
- Los detalles son reemplazables.

Las diferencias son de vocabulario y de dibujo. No cambies de arquitectura por cambiar el nombre del diagrama.

---

## 6. Vocabulario mínimo

No hace falta un vocabulario enorme. Con estos términos se puede hablar de casi todo:

| Término | Qué es | Qué no es |
|---------|--------|-----------|
| **Dominio** | Conceptos e invariantes del negocio. | Una tabla de la base de datos. |
| **Caso de uso** | Intención de la aplicación. | Un controlador HTTP. |
| **Puerto** | Contrato en un límite. | La API concreta del proveedor. |
| **Adaptador** | Traducción tecnológica. | Una regla de negocio. |
| **Entidad** | Identidad, estado, comportamiento e invariantes. | Una fila de una tabla. |
| **Objeto de valor** | Valor inmutable con reglas propias. | Un `string` con nombre bonito. |
| **Dato / primitivo** | Un valor sin reglas propias. | Todo necesita una clase. |

La frase clave: **el centro no es una base de datos disfrazada**. Una tabla almacena datos; una entidad protege comportamiento e invariantes.

---

## 7. La dirección de las dependencias

### Las dependencias apuntan hacia las políticas

El dominio no conoce la aplicación. La aplicación no conoce los adaptadores concretos. Los adaptadores conocen el dominio y la aplicación, pero el núcleo no los conoce a ellos.

```
Dominio  <-  Casos de uso  <-  Adaptadores  <-  Frameworks
```

La flecha significa “depende de”. El dominio no importa Express, Stripe ni `pg`. El caso de uso importa contratos del dominio o de la aplicación. El adaptador importa el caso de uso y la tecnología concreta.

### Flujo de ejecución y flujo de dependencia son mapas distintos

En tiempo de ejecución, una petición entra por HTTP, pasa por el caso de uso y termina en PostgreSQL:

```
HTTP -> Controlador -> Caso de uso -> Repositorio -> PostgreSQL
```

Pero en el código, la dependencia va en otra dirección:

```
Controlador -> Caso de uso -> OrderRepository <- PostgresOrderRepository
```

El caso de uso no depende de PostgreSQL. PostgreSQL depende del contrato `OrderRepository`. Eso es la **inversión de dependencias**: la implementación concreta depende del contrato interior, no al revés.

### La necesidad interior define la interfaz

No diseñes el puerto copiando la API del proveedor. Diseña el puerto desde lo que el caso de uso necesita.

**Contrato del proveedor:**

```ts
query(sql, values)
insert(table, row)
transaction(client)
```

**Contrato de aplicación:**

```ts
save(order)
findById(orderId)
```

El primer contrato habla de SQL. El segundo habla de pedidos. El adaptador se encarga de traducir.

### Un puerto genérico no siempre es mejor

Un `Repository<T>` con `insert`, `update`, `delete` y `query` parece reutilizable, pero está dictado por la infraestructura. Obliga al dominio a pensar en tablas y SQL.

```ts
// ❌ El puerto copia la forma de la base de datos
interface Repository<T> {
  insert(row: T): Promise<void>
  update(row: T): Promise<void>
  delete(id: string): Promise<void>
  query(sql: string): Promise<T[]>
}

// ✅ El puerto expresa una capacidad del caso de uso
interface OrderRepository {
  save(order: Order): Promise<void>
  findById(id: OrderId): Promise<Order | null>
}
```

El segundo puerto es más pequeño, pero dice mucho más: el caso de uso necesita guardar y buscar pedidos, no ejecutar SQL arbitrario.

### TypeScript ayuda, pero no decide la arquitectura

El compilador comprueba:

- Compatibilidad estructural.
- Firmas de funciones.
- `narrowing` y tipos.

El equipo decide:

- La dirección de los imports.
- Los límites entre módulos.
- Qué es política y qué es mecanismo.

TypeScript no te impide importar `express` dentro del dominio. Una revisión de código, una regla de lint o una prueba de arquitectura sí pueden hacerlo.

### Señales de fuga

- El dominio importa `Request`, `Response`, `Pool`, `Stripe` o `PrismaClient`.
- El caso de uso devuelve códigos HTTP.
- El repositorio devuelve filas de la base de datos en lugar de entidades.
- El adaptador contiene reglas de negocio.
- Para probar una regla hay que levantar PostgreSQL.

> **Pregunta clave:** si cambio la base de datos, ¿tengo que tocar las reglas del negocio? Si la respuesta es sí, la dependencia está al revés.

---

## 8. El dominio: entidades y objetos de valor

### Una entidad combina identidad, estado y comportamiento

| Componente | Qué significa |
|------------|---------------|
| **Identidad** | El pedido sigue siendo el mismo aunque cambien sus datos. |
| **Estado** | Creado, pagado, enviado, cancelado. |
| **Comportamiento** | Solo permite transiciones válidas. |

Una entidad no es una fila de la tabla. Es un objeto que protege sus invariantes: no se puede pagar un pedido cancelado, no se puede añadir una línea con cantidad negativa, no se puede enviar un pedido sin pagar.

### Evita los dos extremos del modelado

**Modelo anémico:** la entidad solo tiene datos y las reglas viven en servicios externos.

```ts
class Order {
  id: string
  status: string
  total: number
}

class OrderService {
  pay(order: Order): void {
    if (order.status !== 'created') throw new Error('No se puede pagar')
    order.status = 'paid'
  }
}
```

Cualquier servicio puede cambiar `order.status` sin pasar por las reglas. Las invariantes se dispersan.

**Entidad sobrecargada:** la entidad coordina emails, pagos, persistencia y cualquier integración.

```ts
class Order {
  async pay(): Promise<void> {
    await this.email.send('Pedido pagado')
    await this.payments.charge(this.total)
    await this.db.save(this)
    await this.analytics.track('order_paid')
  }
}
```

La entidad termina sabiendo de infraestructura. Se vuelve difícil de probar y de reutilizar.

**Punto medio:** la entidad protege sus reglas; el caso de uso coordina; los adaptadores hablan con la tecnología.

```ts
class Order {
  private constructor(
    readonly id: OrderId,
    private status: OrderStatus,
    private readonly lines: OrderLine[],
  ) {}

  markPaid(): void {
    if (this.status !== 'created') {
      throw new Error('Solo se puede pagar un pedido creado')
    }
    this.status = 'paid'
  }

  total(): Money {
    return this.lines.reduce(
      (total, line) => total.add(line.subtotal()),
      Money.zero('EUR'),
    )
  }
}
```

### No todo sustantivo necesita una clase

| Tipo | Cuándo usarlo | Ejemplo |
|------|---------------|---------|
| **Entidad** | Tiene identidad y ciclo de vida. | `Order`, `Customer`, `Invoice`. |
| **Objeto de valor** | El valor importa más que la identidad y tiene reglas. | `Money`, `Email`, `OrderId`. |
| **Dato / primitivo** | No tiene reglas propias. | Un `string` para un nombre interno, un `number` para una cantidad simple. |

No conviertas cada `string` en una clase. El coste de un objeto de valor debe estar justificado por las reglas que protege.

### El dominio tiene el tamaño de sus reglas

| Tipo de sistema | Dominio | Inversión |
|-----------------|---------|-----------|
| **CRUD sencillo** | Pocas invariantes. Un dominio mínimo puede ser suficiente. | Baja. No fuerces entidades ricas. |
| **Negocio complejo** | Reservas, facturación, aprobaciones, cálculos. | Alta. Un modelo expresivo compensa. |

La arquitectura debe ser proporcional. Un formulario de contacto no necesita la misma estructura que un sistema de reservas.

### Un `number` no explica su significado

```ts
charge({ amount: 50 })
// ¿50 euros?
// ¿50 céntimos?
// ¿otra moneda?
```

El tipo `number` compila, pero no dice nada. Un objeto de valor explicita la semántica:

```ts
class Money {
  private constructor(
    readonly cents: number,
    readonly currency: 'EUR' | 'USD',
  ) {}

  static fromEuros(euros: number, currency: 'EUR' | 'USD' = 'EUR'): Money {
    if (!Number.isFinite(euros) || euros < 0) {
      throw new Error('Importe inválido')
    }
    return new Money(Math.round(euros * 100), currency)
  }

  add(other: Money): Money {
    if (other.currency !== this.currency) {
      throw new Error('No se pueden sumar monedas distintas')
    }
    return new Money(this.cents + other.cents, this.currency)
  }

  static zero(currency: 'EUR' | 'USD'): Money {
    return new Money(0, currency)
  }
}
```

### Identidad y valor responden a preguntas diferentes

| Entidad | Objeto de valor |
|---------|-----------------|
| Dos pedidos con los mismos datos pueden ser pedidos distintos. | Dos importes de 50 € representan el mismo valor. |
| Se compara por identidad. | Se comparan sus componentes. |
| Tiene ciclo de vida. | Es inmutable. |
| Puede cambiar de estado. | No cambia: se sustituye por otro valor. |

### Validar una vez permite confiar después

```
unknown / primitive -> Factory -> ValueObject -> Operaciones seguras
```

La fábrica valida una vez y construye un valor que ya es válido. A partir de ahí, el núcleo puede confiar en el tipo y no repetir comprobaciones.

```ts
const amount = Money.fromEuros(input.amount)
charge(amount) // dentro, amount siempre tiene una unidad y un valor válidos
```

### Más seguridad también introduce traducción

| Ventaja | Coste |
|---------|-------|
| Semántica explícita | Mapeo en los bordes |
| Inmutabilidad | Serialización |
| Invariantes locales | Ruido si se abusa |
| Menos estados inválidos | Más tipos y conversiones |

Un objeto de valor no es gratis. Si solo envuelve un `string` sin reglas, probablemente no compensa.

---

## 9. Casos de uso

### Una intención merece un nombre

`PlaceOrder`, `ApproveExpense`, `ReserveRoom`, `RenewLoan`.

Los nombres expresan **objetivos de la aplicación**, no tecnologías. `OrderController` o `OrderService` describen una pieza técnica; `PlaceOrder` describe lo que el usuario quiere conseguir.

### El caso de uso coordina; el dominio decide sus reglas

Un caso de uso `PlaceOrder` puede:

1. Crear el pedido.
2. Añadir las líneas.
3. Cobrar.
4. Marcar el pedido como pagado.
5. Guardarlo.

Pero las reglas de qué es un pedido válido, cuándo se puede pagar o cómo se calcula el total viven en el dominio. El caso de uso **orquesta**, no reimplementa las reglas.

```
PlaceOrder -> Order.create()
           -> Order.addLine()
           -> PaymentGateway.charge()
           -> Order.markPaid()
           -> OrderRepository.save()
```

### Input y output son contratos de aplicación

La entrada debe ser independiente del framework. El adaptador puede construirla desde un JSON, un formulario o un comando de consola.

```ts
interface PlaceOrderInput {
  customerId: string
  items: Array<{
    productId: string
    quantity: number
  }>
}

type PlaceOrderOutput =
  | { ok: true; orderId: string }
  | { ok: false; reason: 'payment_declined' | 'out_of_stock' }
```

La salida tampoco debe tener códigos HTTP ni detalles de la interfaz. El caso de uso dice “pago rechazado”; el adaptador decide que eso es un `422` con un JSON concreto.

### Separa por intención, no por método privado

**Evita:**

```ts
class OrderService {
  create() {}
  pay() {}
  send() {}
  cancel() {}
  list() {}
  export() {}
}
```

**Prefiere:** un contrato reconocible para cada intención relevante.

```ts
interface PlaceOrder {
  execute(input: PlaceOrderInput): Promise<PlaceOrderOutput>
}

interface CancelOrder {
  execute(input: CancelOrderInput): Promise<CancelOrderOutput>
}

interface ExportOrders {
  execute(input: ExportOrdersInput): Promise<ExportOrdersOutput>
}
```

No significa que cada método necesite una clase. Significa que no mezcles intenciones distintas en un servicio gigante solo porque comparten el sustantivo “Order”.

---

## 10. Ports: contratos en los límites

### Un puerto es un límite con nombre

La aplicación **ofrece capacidades** y **solicita colaboraciones** mediante contratos.

- **Puerto de entrada:** lo que la aplicación ofrece al exterior. Normalmente, la interfaz del caso de uso.
- **Puerto de salida:** lo que la aplicación necesita de otros sistemas. Por ejemplo, `OrderRepository` o `PaymentGateway`.

```ts
// Puerto de entrada
interface PlaceOrderUseCase {
  execute(input: PlaceOrderInput): Promise<PlaceOrderOutput>
}

// Puerto de salida
interface OrderRepository {
  save(order: Order): Promise<void>
  findById(id: OrderId): Promise<Order | null>
}
```

### Diseña el puerto desde la necesidad interior

No diseñes el puerto mirando la API del proveedor. Pregúntate qué necesita el caso de uso.

| Puerto dictado por la infraestructura | Puerto dictado por la aplicación |
|---------------------------------------|----------------------------------|
| `Repository<T>` con `insert`, `update`, `delete`, `query` | `OrderRepository` con `save`, `findById` |
| Habla de tablas y SQL | Habla de pedidos |
| Obliga al dominio a conocer la base de datos | Permite probar con un repositorio en memoria |

### El puerto no copia el idioma del proveedor

```
“Cobrar 50 €” -> PaymentGateway -> Adaptador -> SDK del proveedor
```

El adaptador realiza una **traducción semántica**: euros a céntimos, `approved` a `paid`, códigos del proveedor a resultados del negocio.

```ts
interface PaymentGateway {
  charge(amount: Money, customerId: CustomerId): Promise<PaymentResult>
}

type PaymentResult =
  | { approved: true; reference: string }
  | { approved: false; reason: 'declined' | 'provider_error' }
```

El caso de uso no sabe si el proveedor devuelve `paid`, `succeeded` o un código `200`. El adaptador lo traduce.

### Una abstracción debe proteger una decisión

| Coste | Valor |
|-------|-------|
| Un contrato más | Aísla la variación |
| Conversión en los bordes | Mejora las pruebas |
| Una implementación más | Evita que los tipos externos entren al núcleo |

No abstraigas por moda. Abstrae cuando hay una **decisión que proteger**: un proveedor que puede cambiar, una base de datos que puede sustituirse, una regla que necesita pruebas sin infraestructura.

### Los ports documentan las necesidades del caso de uso

El constructor de `PlaceOrder` es un mapa reducido de colaboraciones externas:

```ts
class PlaceOrder {
  constructor(
    private readonly orders: OrderRepository,
    private readonly payments: PaymentGateway,
    private readonly ids: IdGenerator,
  ) {}
}
```

Si el constructor tiene diez dependencias, el caso de uso probablemente está haciendo demasiado. Los puertos también ayudan a ver ese exceso.

### El exterior y el núcleo hablan idiomas diferentes

| Exterior | Núcleo |
|----------|--------|
| JSON, HTTP | Pedidos |
| SQL, SDKs | Dinero |
| Variables de entorno | Clientes |
| Códigos de estado | Reglas |
| Formatos de proveedor | Resultados |

El adaptador es el traductor. El núcleo no debería tener que aprender el idioma de cada proveedor.

---

## 11. Adapters: traducción tecnológica

### Los adapters conducen o son conducidos

```
[Adaptador HTTP] -> [Caso de uso] -> [Puerto] -> [Adaptador SQL]
```

- **Adaptador de entrada (conducido por el exterior):** HTTP, CLI, cola, test. Llama a la aplicación.
- **Adaptador de salida (conducido por la aplicación):** SQL, API de pago, email, almacenamiento. La aplicación lo llama a través del puerto.

### El controller traduce resultados al protocolo

```
payment_declined -> Adaptador HTTP -> 422 + JSON
```

El caso de uso no conoce códigos de estado. El adaptador decide que `payment_declined` se representa como `422`, `out_of_stock` como `409` y un error inesperado como `500`.

### Un adapter puede contener lógica técnica

No todo lo que hay en un adaptador es “sucio”. Hay decisiones que pertenecen al adaptador:

| Decisión | Dónde vive |
|----------|------------|
| “Un gasto de más de 1.000 € requiere finanzas” | Dominio o caso de uso (regla de negocio). |
| “Un timeout del SDK activa una política de reintento” | Adaptador (decisión técnica). |
| “Un JSON inválido responde 400” | Adaptador (traducción de protocolo). |
| “No se puede pagar un pedido cancelado” | Dominio (invariante). |

La diferencia: una regla de negocio cambia porque cambia el negocio; una decisión técnica cambia porque cambia la tecnología.

### Cambia el mecanismo sin reescribir la política

```
HTTP / CLI -> PlaceOrder <- Memoria / SQL / Fake / PSP
```

El mismo caso de uso puede ejecutarse desde HTTP o desde la consola, con un repositorio en memoria o con PostgreSQL, con un pago falso o con un proveedor real.

La sustitución no es gratis: hay que implementar el adaptador y probarlo. Pero el límite hace **visible** el impacto. Sin límite, cambiar el mecanismo obliga a buscar sus detalles por todo el núcleo.

### Ejemplo de adaptador de salida

```ts
class StripePaymentGateway implements PaymentGateway {
  constructor(private readonly stripe: Stripe) {}

  async charge(amount: Money, customerId: CustomerId): Promise<PaymentResult> {
    const result = await this.stripe.charges.create({
      amount: amount.cents,
      currency: amount.currency.toLowerCase(),
      customer: customerId.value,
    })

    return result.paid
      ? { approved: true, reference: result.id }
      : { approved: false, reason: 'declined' }
  }
}
```

El adaptador conoce Stripe. El puerto solo conoce `Money`, `CustomerId` y `PaymentResult`.

---

## 12. Composición e inyección de dependencias

### Alguien tiene que elegir las piezas concretas

En algún lugar hay que decidir:

- ¿Qué repositorio se usa?
- ¿Qué pasarela de pago?
- ¿Qué ciclo de vida tienen los objetos?
- ¿Qué configuración se carga?

Ese lugar es la **raíz de composición** (composition root). Normalmente está en el arranque de la aplicación, no en el dominio.

### Inyección de dependencias sin magia

Las dependencias obligatorias entran por el constructor:

```ts
const orders = new PostgresOrderRepository(pool)
const payments = new StripePaymentGateway(stripe)
const placeOrder = new PlaceOrder(orders, payments)
const controller = new PlaceOrderController(placeOrder)
```

No hace falta un contenedor para esto. La composición manual es explícita, legible y suficiente en muchos proyectos.

### Un contenedor automatiza; no crea la inversión

| Composición manual | Contenedor de dependencias |
|--------------------|----------------------------|
| Explícita | Útil para grafos grandes |
| Legible | Puede ocultar el grafo |
| Suficiente en muchos proyectos | Debe confinarse a la raíz de composición |

El contenedor no arregla un diseño acoplado. Si el dominio depende de la infraestructura, inyectar esas dependencias no invierte nada.

### La configuración se resuelve en la periferia

```
Variables de entorno -> Clientes y pools -> Adaptadores -> Casos de uso
```

El núcleo recibe colaboradores ya preparados. No lee `process.env`, no abre conexiones y no sabe qué base de datos está detrás.

### La aplicación es un grafo de objetos

```
Adaptadores -> Casos de uso -> Dominio
      ^             |
      |             v
    Puertos <- Adaptadores
```

La raíz de composición construye un grafo distinto para cada entorno:

- **Producción:** PostgreSQL, pasarela real, email real.
- **Tests:** repositorio en memoria, pasarela falsa, reloj controlado.
- **Desarrollo:** base de datos local, proveedor simulado.

El núcleo es el mismo; cambian las piezas de la periferia.

---

## 13. Errores, validación y fronteras

### No todas las situaciones excepcionales significan lo mismo

| Tipo | Ejemplo | Quién lo maneja | Cómo |
|------|---------|-----------------|------|
| **Entrada mal formada** | JSON inválido, falta un campo | Adaptador | Responde `400` antes de ejecutar el caso de uso. |
| **Resultado esperado** | Pago rechazado, recurso no disponible | Caso de uso | Devuelve un resultado con un motivo; el adaptador lo traduce. |
| **Error inesperado** | Bug, caída, contrato incumplido | Infraestructura / borde | Se registra y se responde `500`. |

No mezcles los tres. Un pago rechazado no es un `throw` genérico; es un resultado de negocio. Un JSON inválido no debería llegar al dominio. Un `NullPointerException` no es “datos inválidos”.

### Estructura en el borde; semántica en el núcleo

| Validación estructural | Validación semántica |
|------------------------|----------------------|
| ¿Los datos tienen la forma necesaria? | ¿La operación respeta las reglas? |
| `unknown -> input` | `input -> dominio` |
| La hace el adaptador | La hace el dominio |

```ts
// Adaptador: validación estructural
const input = PlaceOrderInput.parse(req.body)

// Dominio: validación semántica
order.addLine(productId, quantity) // rechaza cantidades no válidas
```

El adaptador comprueba que el JSON tiene los campos y los tipos. El dominio comprueba que la operación tiene sentido.

### Cada frontera responde por el conocimiento que posee

| Frontera | Conocimiento |
|----------|--------------|
| **Adaptador** | Formatos, protocolos, SDKs, códigos de respuesta. |
| **Dominio** | Invariantes, cálculos, transiciones válidas. |
| **Caso de uso** | Orden de la operación, resultados esperados. |

Cuando un error aparece en el lugar equivocado, suele ser porque una frontera está asumiendo un conocimiento que no le corresponde.

---

## 14. Tests por zona

### Cada zona concentra un riesgo diferente

| Zona | Qué probar | Ejemplo |
|------|------------|---------|
| **Dominio** | Invariantes y cálculos | Un pedido cancelado no se puede pagar. |
| **Caso de uso** | Orquestación y resultados | Si el pago se rechaza, el pedido no se guarda. |
| **Adaptadores** | Traducciones y contratos | Un JSON inválido responde `400`; un `payment_declined` responde `422`. |
| **E2E** | Cableado e infraestructura real | El flujo completo con PostgreSQL y un proveedor simulado. |

### Prueba comportamiento observable, no capas

No pruebes que se llamó a un método privado. Prueba lo que el sistema hace:

1. **Pago rechazado:** el pedido no se guarda.
2. **Cantidad inválida:** la entidad no acepta la línea.
3. **JSON inválido:** el adaptador responde antes de ejecutar el caso de uso.

Eso te permite cambiar la implementación sin romper las pruebas. Si una prueba se rompe cada vez que refactorizas, quizá está acoplada a detalles internos.

### Si necesitas infraestructura para probar una regla, hay una fuga

Una prueba de dominio no debería necesitar PostgreSQL, HTTP ni un proveedor de pago. Una prueba de caso de uso debería usar dobles de prueba para los puertos. Las pruebas E2E existen, pero son pocas y lentas: sirven para comprobar el cableado, no para cubrir cada regla.

---

## 15. Proporcionalidad: cuándo invertir

### La inversión arquitectónica debe ser proporcional

| Simplifica | Invierte |
|------------|----------|
| CRUD | Reglas complejas |
| Pocas reglas | Proveedores variables |
| Bajo riesgo de cambio | Varios canales |
| Vida útil corta | Vida útil larga |

No todos los proyectos necesitan entidades ricas, puertos y adaptadores. Un CRUD sencillo puede vivir con un controlador, un servicio y un repositorio. La arquitectura es una herramienta, no un examen de pureza.

### Cuatro preguntas para diseñar un límite

| Pregunta | Qué decides |
|----------|-------------|
| **¿Qué protegemos?** | La regla o invariante que no debe mezclarse con detalles. |
| **¿Quién puede decidir?** | Si la decisión es del dominio, del caso de uso o del adaptador. |
| **¿Qué necesita?** | El contrato mínimo para colaborar. |
| **¿Qué puede cambiar?** | El detalle que conviene aislar. |

Si no puedes responder a estas cuatro preguntas, probablemente el límite todavía no está claro.

### Señales de sobreingeniería

- Una interfaz con una sola implementación y ningún doble de prueba.
- Un puerto que copia la API del proveedor.
- Una entidad que no tiene ninguna regla.
- Un contenedor de dependencias para cuatro objetos.
- Capas que solo pasan datos.
- Un caso de uso por cada método de un servicio.
- Más tiempo dibujando el diagrama que resolviendo el problema.

Clean Architecture no consiste en aplicar todas las piezas. Consiste en aplicar **las que el problema justifica**.

---

## Checklist final

1. ¿Puedo cambiar PostgreSQL sin tocar las reglas del negocio?
2. ¿Puedo probar el caso de uso sin levantar HTTP ni base de datos?
3. ¿El dominio importa Express, SQL, Stripe o algún SDK?
4. ¿El caso de uso tiene un nombre de intención (`PlaceOrder`, no `OrderService`)?
5. ¿El puerto habla el idioma del negocio y no el del proveedor?
6. ¿El adaptador es el único que conoce la tecnología concreta?
7. ¿La raíz de composición es el único lugar donde se eligen las piezas reales?
8. ¿Distingo entrada mal formada, resultado esperado y error inesperado?
9. ¿Las pruebas comprueban comportamiento observable y no detalles internos?
10. ¿La inversión arquitectónica es proporcional a la complejidad del problema?

> **Regla de oro:** si un cambio importante obliga a tocar HTTP, SQL y reglas a la vez, el límite no existe. La arquitectura se nota en lo que **no** hay que tocar.
