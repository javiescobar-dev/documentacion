
SOLID son cinco principios de diseño orientado a objetos que sirven para escribir código **fácil de mantener, de extender y de testear**. No son reglas mágicas ni obligatorias: son señales para detectar cuándo un diseño se va a volver un problema con el tiempo.

La idea de fondo es simple: el código se lee y se cambia muchísimo más de lo que se escribe la primera vez, así que conviene diseñarlo pensando en **quien vendrá después** (que probablemente seas tú en unos meses).

---

## Resumen en una frase

| Letra | Principio | En una frase |
|-------|-----------|--------------|
| **S** | Single Responsibility (SRP) | Una clase debe tener **una sola razón para cambiar**. |
| **O** | Open/Closed (OCP) | Abierta a **extensión**, cerrada a **modificación**. |
| **L** | Liskov Substitution (LSP) | Un subtipo debe poder **sustituir** a su tipo base sin romper nada. |
| **I** | Interface Segregation (ISP) | Mejor **varias interfaces pequeñas** que una gorda que obliga a implementar de más. |
| **D** | Dependency Inversion (DIP) | Depende de **abstracciones**, no de implementaciones concretas. |

---

## S — Single Responsibility (SRP)

**Definición sencilla:** una clase (o módulo, o función) debería tener una única razón para cambiar, es decir, una única responsabilidad.

**Analogía:** un cuchillo corta y un sacacorchos saca corchos. Si juntas las dos cosas en una "navaja-multiusos-que-también-hace-café", cada vez que cambies una parte puedes romper otra.

### ❌ Malo: la clase hace de todo

```ts
class Factura {
  calcularTotal(): number { /* ... */ }

  guardarEnBaseDeDatos(): void { /* ... */ }   // responsabilidad de persistencia
  enviarPorEmail(): void { /* ... */ }          // responsabilidad de notificación
  generarPdf(): void { /* ... */ }              // responsabilidad de formato
}
```

Cambia el esquema de la base de datos, cambia el formato del email y cambia el diseño del PDF... ¡la misma clase tiene tres motivos distintos para cambiar!

### ✅ Bueno: cada clase, una responsabilidad

```ts
class Factura {
  calcularTotal(): number { /* ... */ }
}

class FacturaRepository {
  guardar(factura: Factura): void { /* ... */ }
}

class EmailNotifier {
  enviarFactura(factura: Factura): void { /* ... */ }
}

class PdfGenerator {
  generar(factura: Factura): string { /* ... */ }
}
```

**🚩 Señal de alarma:** nombrar la clase te cuesta porque "hace muchas cosas"; la palabra "**and**" aparece en la descripción; tienes que saltar entre imports de base de datos, HTTP y formato dentro del mismo archivo.

**Pregunta clave:** si solo pudiera cambiar **un** requisito, ¿cuál sería el motivo por el que tocaría esta clase? Si te salen dos o más, probablemente sobra responsabilidad.

---

## O — Open/Closed (OCP)

**Definición sencilla:** deberías poder **añadir** comportamiento nuevo **sin modificar** el código que ya funciona.

**Analogía:** una regleta eléctrica. Para enchufar un aparato nuevo no abres la pared ni reescribes la instalación: enchufas. El sistema estaba *abierto a extensión* y *cerrado a modificación*.

### ❌ Malo: cada tipo nuevo obliga a tocar la función

```ts
class CalculadoraDescuentos {
  calcular(tipoCliente: string, importe: number): number {
    if (tipoCliente === 'normal') return importe
    if (tipoCliente === 'vip') return importe * 0.9
    if (tipoCliente === 'empleado') return importe * 0.8
    // mañana toca añadir 'socio'... y volvemos a editar aquí
    return importe
  }
}
```

Cada cliente nuevo = abrir esta función y añadir otro `if`. Riesgo de romper los casos que ya funcionaban.

### ✅ Bueno: extendemos sin tocar lo existente

```ts
interface Cliente {
  aplicarDescuento(importe: number): number
}

class ClienteNormal implements Cliente {
  aplicarDescuento(importe: number) { return importe }
}

class ClienteVip implements Cliente {
  aplicarDescuento(importe: number) { return importe * 0.9 }
}

class ClienteEmpleado implements Cliente {
  aplicarDescuento(importe: number) { return importe * 0.8 }
}

function calcularTotal(cliente: Cliente, importe: number): number {
  return cliente.aplicarDescuento(importe)
}
```

Añadir `ClienteSocio` es crear una clase nueva; `calcularTotal` ni se entera y no se toca.

**🚩 Señal de alarma:** un `switch` o una cadena de `if/else` que comparan un "tipo" y que **crecen cada vez que aparece un caso nuevo**.

**Pregunta clave:** ¿para añadir una variante nueva tengo que **editar** código que ya estaba bien, o solo **añadir** código?

---

## L — Liskov Substitution (LSP)

**Definición sencilla:** si tienes código que usa un tipo base, debe funcionar igual si le pasas cualquier subtipo, **sin sorpresas**.

**Analogía:** si un contrato dice "aquí se entrega un coche que arranca y frena", no vale entregar un coche sin frenos solo porque "es un coche". El subtipo debe cumplir lo que promete el tipo base.

### ❌ Malo: el subtipo rompe la promesa

```ts
class Ave {
  volar(): void { /* vuela */ }
}

class Pinguino extends Ave {
  volar(): void {
    throw new Error('¡Los pingüinos no vuelan!')  // rompe el contrato de Ave
  }
}

function hacerVolarLasAves(aves: Ave[]): void {
  aves.forEach(a => a.volar())  // explota si hay un Pinguino
}
```

`Pinguino` **es** un `Ave`, pero no se comporta como el código espera. El cliente tiene que defenderse con comprobaciones raras.

### ✅ Bueno: el contrato refleja la realidad

```ts
interface Ave { /* cosas que TODAS las aves pueden hacer */ }

interface Voladora {
  volar(): void
}

class Aguila implements Ave, Voladora {
  volar(): void { /* vuela */ }
}

class Pinguino implements Ave { /* solo lo que de verdad puede hacer */ }
```

Ahora solo pides `Voladora` cuando de verdad vas a volar.

**🚩 Señal de alarma:** métodos del subtipo que hacen `throw new Error('no soportado')`, o que quedan **vacíos**, o que **cambian el significado** de lo que hace la clase base.

**Pregunta clave:** si cambio un objeto por un subtipo, ¿el código que lo usa sigue teniendo sentido **sin cambiar una sola línea**?

---

## I — Interface Segregation (ISP)

**Definición sencilla:** ninguna clase debería verse obligada a implementar métodos que no usa. Mejor muchas interfaces pequeñas y específicas que una enorme y genérica.

**Analogía:** menús de restaurante. Un menú único gigantesco obliga a todos a leer platos que no van a pedir. Mejor una carta de desayunos, una de comidas y una de postres.

### ❌ Malo: interfaz gorda

```ts
interface AparatoOficina {
  imprimir(): void
  escanear(): void
  enviarFax(): void
}

class ImpresoraBasica implements AparatoOficina {
  imprimir(): void { /* ok */ }
  escanear(): void { throw new Error('no soportado') }
  enviarFax(): void { throw new Error('no soportado') }
}
```

La impresora tiene que "implementar" cosas que ni existen en ella (código muerto + errores en tiempo de ejecución).

### ✅ Bueno: interfaces pequeñas

```ts
interface Impresora { imprimir(): void }
interface Escaner { escanear(): void }
interface Fax { enviarFax(): void }

class ImpresoraBasica implements Impresora {
  imprimir(): void { /* ok */ }
}

class Multifuncion implements Impresora, Escaner, Fax {
  imprimir(): void { /* ok */ }
  escanear(): void { /* ok */ }
  enviarFax(): void { /* ok */ }
}
```

**🚩 Señal de alarma:** implementaciones con `throw new Error('no soportado')` o métodos vacíos; interfaces con muchísimos métodos donde cada cliente usa solo 2 o 3.

**Pregunta clave:** ¿todos los clientes de esta interfaz usan **todos** sus métodos? Si no, divídela.

> Nota: LSP e ISP a menudo aparecen juntos. La diferencia está en **dónde** miras: ISP mira la **interfaz** (¿es demasiado grande?), LSP mira la **jerarquía** (¿el subtipo sustituye bien al padre?).

---

## D — Dependency Inversion (DIP)

**Definición sencilla:** el código de alto nivel (tu lógica de negocio) no debe depender de detalles (base de datos, email, sistema de ficheros). Ambos deben depender de **abstracciones** (interfaces), y **las abstracciones no deben depender de los detalles**.

**Analogía:** un enchufe estándar. Tu televisor no viene soldado a la central eléctrica: ambos cumplen el "contrato del enchufe". Puedes cambiar de compañía eléctrica sin cambiar la tele.

### ❌ Malo: la lógica crea sus propias dependencias

```ts
class ServicioDeAlta {
  private repo = new PostgresUserRepository()
  private email = new SmtpEmailSender()

  registrar(emailUsuario: string): void {
    const usuario = { email: emailUsuario }
    this.repo.guardar(usuario)      // atado a PostgreSQL
    this.email.enviar(emailUsuario) // atado a SMTP concreto
  }
}
```

El servicio **decide** qué base de datos y qué email usar. No puedes testearlo sin una base de datos real, ni cambiar de SMTP sin editar esta clase.

### ✅ Bueno: depende de interfaces e inyectamos las implementaciones

```ts
interface UserRepository {
  guardar(usuario: Usuario): void
}

interface NotificationSender {
  enviar(destino: string): void
}

class ServicioDeAlta {
  constructor(
    private readonly repo: UserRepository,
    private readonly notificador: NotificationSender
  ) {}

  registrar(email: string): void {
    this.repo.guardar({ email })
    this.notificador.enviar(email)
  }
}

// En el "composition root" (arranque de la app) unimos las piezas reales:
const servicio = new ServicioDeAlta(
  new PostgresUserRepository(),
  new SmtpEmailSender()
)
```

Ahora en los tests pasas un doble (fake) y en producción las clases reales. La flecha de dependencia se **invierte**: tanto la lógica como la infraestructura apuntan a la interfaz.

**🚩 Señal de alarma:** `new AlgoConcreto()` dentro de la lógica de negocio; uso directo de `Date`, `Math.random`, `process.env`, `fetch` o `fs` escondido en medio del dominio; imposible testear sin montar servicios reales.

**Pregunta clave:** ¿mi lógica **nombra** clases concretas de infraestructura, o solo habla de interfaces que yo defino?

---

## Cómo diferenciarlos (los pares que se confunden)

| Si el problema es... | Suele ser... | Cómo reconocerlo |
|----------------------|--------------|------------------|
| Una clase cambia por muchos motivos distintos | **SRP** | "más de una razón para cambiar" |
| Un `switch`/`if` por tipo crece al añadir casos | **OCP** | añadir variante obliga a **editar** código existente |
| Un subtipo rompe lo que prometía el padre | **LSP** | `throw`/métodos vacíos en la herencia |
| Una interfaz obliga a implementar de más | **ISP** | interfaces gordas, métodos ignorados |
| La lógica crea/usa clases concretas de infra | **DIP** | `new Concreto()`, `process.env`, `fetch` en el dominio |

**Confusiones típicas:**

- **SRP vs ISP:** SRP habla de una **clase** con demasiadas responsabilidades; ISP habla de una **interfaz** con demasiados métodos. Mira si el problema está en la implementación o en el contrato.
- **OCP vs DIP:** OCP es *"no tocar el código al añadir algo nuevo"*; DIP es *"depender de abstracciones"*. A menudo se resuelven juntos (polimorfismo + interfaces), pero son ideas distintas: puedes violar DIP sin violar OCP y al revés.
- **LSP vs OCP:** LSP es sobre **sustituir** (herencia que respeta el contrato); OCP es sobre **extender** sin modificar. LSP suele ser el pegamento que hace funcionar el OCP.
- **SRP vs DIP:** SRP pregunta *"¿de qué se encarga?"*; DIP pregunta *"¿de quién depende?"*.

---

## Cómo localizarlos (checklist rápida)

Cuando revises código, pregúntate en este orden:

1. **Clases o funciones enormes** → probable **SRP**.
2. **`switch`/`if` comparando un tipo** que hay que tocar al añadir casos → **OCP**.
3. **Herencia con `throw 'no soportado'` o métodos vacíos** → **LSP** (o **ISP** si el origen es una interfaz gorda).
4. **Interfaces con métodos que algunos clientes ignoran** → **ISP**.
5. **`new Concreto()` / acceso a `process.env`, `Date`, red o disco dentro de la lógica** → **DIP**.

> Regla de oro: los tres primeros suelen ser problemas de **responsabilidad y estructura**; los dos últimos, de **contratos y dependencias**.
