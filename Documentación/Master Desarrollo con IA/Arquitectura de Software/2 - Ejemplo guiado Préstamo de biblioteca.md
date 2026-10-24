
Clean Architecture no se aprende memorizando capas, se aprende aplicando límites a un problema concreto. En este bloque veremos un **corte vertical**: `BorrowBook` (prestar un libro) llevado desde el enunciado hasta la composición, los adaptadores, los tests y la regla de direcciones.

## Resumen en una frase

| Concepto | En una frase |
|----------|--------------|
| **Corte vertical** | Un único caso de uso resuelto de principio a fin, cruzando dominio, aplicación, puertos, adaptadores y composición. |
| **Dominio** | Concentrar las reglas estables (disponibilidad, suspensión, límite de préstamos, duración). No reproduce tablas. |
| **Caso de uso** | Coordina la secuencia, pregunta al dominio, orquesta puertos externos (repositorio, reloj, generador de ids) y devuelve un resultado explícito. |
| **Puerto (Driven)** | Contrato que la aplicación necesita para hablar con el exterior. Se diseña desde dentro hacia fuera. |
| **Adaptador (Driven)** | Traducción tecnológica que implementa ese contrato (InMemory, PostgreSQL, etc.). |
| **Adaptador (Driving)** | Traducción desde el exterior hacia el caso de uso (HTTP, CLI, cola). |
| **`LocalDate` vs `Date`** | Para reglas de "día natural" (fecha de vencimiento) es mejor trabajar con `LocalDate`, no con `Date` con zona horaria. |
| **Unidad lógica de persistencia** | Todo lo que debe cambiar junto se confirma en una única operación (`saveBorrowing`). |
| **Raíz de composición** | Único lugar donde se eligen las implementaciones concretas y se conectan todas las piezas. |

## Mapa del documento

| Bloque | Qué responde |
|--------|--------------|
| **1. El punto de partida** | Por qué empezar por el enunciado y clasificar decisiones, no archivos. |
| **2. Un corte vertical** | El camino: modelar → contratar → coordinar → adaptar → verificar. |
| **3. El dominio** | Entidades, objetos de valor, transiciones y rechazos esperados vs invariantes. |
| **4. El caso de uso `BorrowBook`** | Entrada/salida explícitas, flujo en 5 pasos y coordinación. |
| **5. Puertos: describir necesidades** | ¿Qué necesita el caso de uso? Repositorio, reloj, generador de IDs. |
| **6. Adaptadores** | InMemory, traducción y el compromiso de atomicidad. |
| **7. HTTP como adaptador driving** | Validación, mapeo de resultados a códigos HTTP y separación de reglas. |
| **8. Raíz de composición** | Conectar piezas sin contaminar dominio/aplicación. |
| **9. Tests por zona** | Dominio puro, caso de uso con spies/stubs, controlador HTTP, composición. |
| **10. Grafo de dependencias** | El verdadero contrato: imports + constructores. |
| **11. Lecciones clave** | Qué queda limpio, qué es proporcional y qué recordar. |

---

## 1. El punto de partida

Todo empieza con un enunciado claro. El objetivo no es "crear entidades", sino **saber quién toma cada decisión**.

> Disponibilidad, suspensión, límites, tiempo, persistencia y transporte HTTP **no pertenecen al mismo lugar**.

Para eliminar ambigüedad, los tests de aceptación ayudan a fijar el comportamiento:

```text
Given a book with one available copy
And an active standard member
When the member borrows it on 2026-09-01
Then the due date is 2026-09-15
```

La clasificación debe ser por **decisiones de negocio/técnicas**, no por "carpetas por defecto":

| Zona | Responsabilidad |
|------|-----------------|
| **Dominio** | Reglas estables: disponibilidad, suspensión, límite de préstamos, duración del préstamo. |
| **Aplicación (Casos de uso)** | Secuencia, participantes, coordinación y resultado. |
| **Adaptadores** | Persistencia, HTTP, reloj, identidad (generador de IDs). |
| **Composición (Composition Root)** | Elegir implementaciones concretas y conectarlas. |

---

## 2. Un corte vertical

En lugar de "hacer todas las entidades primero", se construye **un único flujo completo**:

| Paso | Acción | Objetivo |
|------|--------|-----------|
| **01. Modelar** | Reglas ejecutables en el dominio | Decidir sin infraestructura. |
| **02. Contratar** | Definir `Input`, `Output` y puertos (driven) | Fijar el contrato antes de la implementación. |
| **03. Coordinar** | Implementar el caso de uso legible | Orquestar sin detalles. |
| **04. Adaptar** | Implementar adaptadores (memoria/HTTP) | Traducir al exterior. |
| **05. Verificar** | Tests por zona + revisar dependencias | Asegurar que los límites se mantienen. |

---

## 3. El dominio: reglas que deciden

El dominio responde a preguntas de negocio: *¿Puede prestarse este libro? ¿Puede este socio? ¿Cuándo vence?*

### 3.1. Entidades, objetos de valor e identidad

| Elemento | Tipo | Explicación |
|----------|------|-------------|
| **`Book`** | Entidad | Tiene identidad (`bookId`). Gestiona `availableCopies`. Su comportamiento cambia disponibilidad al prestarse. |
| **`Member`** | Entidad | Tiene identidad (`memberId`). Sabe si está activo/suspendido y lleva el contador de préstamos activos (`activeLoanCount`). |
| **`Loan`** | Entidad | Representa el préstamo aceptado. Nace cuando todas las reglas se cumplen. |
| **`LoanPeriod`** | Objeto de Valor | `startDate` + `dueDate`. Inmutable, con reglas propias para calcular la duración según el tipo de socio. |

### 3.2. Transiciones con resultado explícito

Una operación de dominio debe **producir un nuevo estado** o **rechazar con motivo explícito**. El objeto original permanece inalterado.

```typescript
// Ejemplo conceptual
const result = book.borrowCopy();

if (!result.ok) {
  return { ok: false, reason: "book_unavailable" };
}

// result.book tiene availableCopies = 0
```

### 3.3. Rechazos esperados vs invariantes

| Caso | Tratamiento | Explicación |
|------|-------------|-------------|
| **Rechazos esperados del negocio** | Retornar `ok: false, reason: ...` | Forman parte de la **tabla de decisión**. Ej.: `member_suspended`, `loan_limit_reached`, `book_unavailable`. |
| **Invariantes rotas** | Lanzar `DomainRuleViolation` | Estados imposibles que indican un **bug**, no un flujo de negocio. Ej.: `activeLoanCount < 0`. |

### 3.4. Trabajar con fechas

Para este caso de uso, la regla es "días naturales". Por eso se prefiere un tipo como `LocalDate`:

- **`Date`**: representa un instante; con zonas horarias es fácil equivocarse.
- **`LocalDate`**: representa un día válido (`YYYY-MM-DD`). Permite `parse`, `plusDays`, `toString` y comparaciones claras.

### 3.5. Fijar reglas con tests de dominio

El dominio debe poder probarse **sin red, sin base de datos, sin reloj global, sin framework**.

```typescript
expect(book.borrowCopy()).toEqual({ ok: false, reason: "book_unavailable" });
expect(period.dueOn.toString()).toBe("2026-09-15");
```

---

## 4. El caso de uso `BorrowBook`

El caso de uso es una **frontera**. Tiene entrada pequeña, resultados explícitos y **ningún detalle de transporte**.

### 4.1. Contrato de entrada/salida

```typescript
// Input
type BorrowBookInput = {
  bookId: string;
  memberId: string;
};

// Output (tabla de decisión)
type BorrowBookOutput =
  | { ok: true; loanId: string; dueDate: string }
  | { ok: false; reason: "book_not_found" | "member_not_found" | "book_unavailable" | "member_suspended" | "loan_limit_reached" };
```

### 4.2. Flujo en 5 movimientos

| Orden | Movimiento | Acción |
|-------|------------|--------|
| **01** | **Localizar** | Buscar `Book` y `Member` por ID. Si no existen → `*_not_found`. |
| **02** | **Decidir** | Aplicar reglas de dominio. `book.borrowCopy()` y `member.registerLoan()`. Si fallan → rechazos esperados. |
| **03** | **Fechar** | Calcular periodo con `LoanPeriod.forMembership(clock.today(), member.membershipType)`. |
| **04** | **Crear** | Crear `Loan` con `ids.nextLoanId()`, libro, socio, fechas. |
| **05** | **Confirmar** | Persistir **todo junto** (`saveBorrowing`) y devolver `ok: true` con `loanId` y `dueDate`. |

### 4.3. Implementación esquemática

```typescript
// Esquema del caso de uso
const book = await repository.findBookById(bookId(input.bookId));
if (!book) return { ok: false, reason: "book_not_found" };

const member = await repository.findMemberById(memberId(input.memberId));
if (!member) return { ok: false, reason: "member_not_found" };

// Dominio decide
const borrowedBook = book.borrowCopy();
if (!borrowedBook.ok) return borrowedBook;

const borrowingMember = member.registerLoan();
if (!borrowingMember.ok) return borrowingMember;

// Tiempo e identidad via ports
const period = LoanPeriod.forMembership(clock.today(), member.membershipType);
const loan = Loan.create({
  id: ids.nextLoanId(),
  bookId: book.id,
  memberId: member.id,
  startDate: period.startOn,
  dueDate: period.dueOn,
});

// Unidad lógica completa
await repository.saveBorrowing({ book: borrowedBook.book, member: borrowingMember.member, loan });
return { ok: true, loanId: loan.id.value, dueDate: loan.dueDate.toString() };
```

> **Clave:** El caso de uso define **qué** debe persistirse junto. El adaptador define **cómo** garantizar esa atomicidad (transacción, versión, locking, etc.).

---

## 5. Puertos: contratos diseñados desde dentro

Los **puertos driven** nacen de una necesidad del caso de uso, no de "guardar entidades genéricas".

```typescript
interface BorrowingRepository {
  findBookById(id: BookId): Promise<Book | null>;
  findMemberById(id: MemberId): Promise<Member | null>;
  saveBorrowing(changes: BorrowingChanges): Promise<void>;
}
```

Se necesitan tres dependencias por razones distintas:

| Puerto | Razón |
|--------|-------|
| **`BorrowingRepository`** | Cargar estado y **confirmar el cambio completo** de forma atómica a nivel lógico. |
| **`Clock`** | Fuente externa de fecha. Permite tests deterministas con un reloj fijo. |
| **`IdGenerator`** | Fuente externa de identidad. Permite generar IDs controlados en tests. |

**Regla:** Los puertos pertenecen al **núcleo** (aplicación/dominio). Los adaptadores viven en la **periferia**.

---

## 6. Adaptadores: traducir sin ensuciar el núcleo

### 6.1. Adaptador driven en memoria

Para aprender, `InMemoryBorrowingRepository` es perfecto. Traduce el contrato a un `Map<BookId, Book>`, `Map<MemberId, Member>`, etc.

- **Es infraestructura**, no lógica de aplicación.
- Puede aceptar un `seed` para demos/tests sin ensanchar el puerto.
- Debe garantizar que `saveBorrowing` persiste **los tres cambios juntos** (libro, socio, préstamo).

```typescript
// Conceptual
class InMemoryBorrowingRepository implements BorrowingRepository {
  constructor(private books = new Map(), private members = new Map(), private loans = new Map()) {}
  async saveBorrowing(changes: BorrowingChanges) {
    // Unidad lógica: actualizar book, member y crear loan
    this.books.set(changes.book.id.value, changes.book);
    this.members.set(changes.member.id.value, changes.member);
    this.loans.set(changes.loan.id.value, changes.loan);
  }
}
```

### 6.2. ¿Qué cambia al pasar a PostgreSQL?

| Aspecto | InMemory | PostgreSQL |
|---------|----------|------------|
| **Acceso** | `Map` directo | `hydration` (SQL → entidades) |
| **Persistencia** | Inmediato | **Transacción** (BEGIN/COMMIT/ROLLBACK) |
| **Concurrencia** | No aplica | `SELECT ... FOR UPDATE`, control de versiones/optimistic locking |
| **Semántica** | Suficiente para aprender | Debe preservar la **unidad lógica** definida por el caso de uso |

**Importante:** Cambiar de adaptador **no cambia la historia del caso de uso ni las reglas de dominio**.

---

## 7. HTTP como adaptador driving

HTTP es solo **una interpretación** del caso de uso. Validar, invocar, traducir. **Las reglas del préstamo no se reescriben en el controlador**.

### 7.1. Validación del borde

En el borde se trabaja con `unknown`. Se valida **forma**, no negocio.

```typescript
function parseInput(body: unknown): BorrowBookInput | null {
  if (!isRecord(body) || typeof body.bookId !== "string" || typeof body.memberId !== "string") {
    return null;
  }
  return { bookId: body.bookId, memberId: body.memberId };
}
```

### 7.2. Mapeo de resultados a códigos HTTP

El controlador traduce el resultado del caso de uso a la respuesta HTTP. Los códigos reflejan **naturaleza**, no detalles internos.

| Código | Cuándo | Significado |
|--------|--------|-------------|
| **`400 Bad Request`** | `parseInput` falla / entrada inválida | Forma incorrecta. |
| **`404 Not Found`** | `book_not_found` / `member_not_found` | Recurso no encontrado. |
| **`409 Conflict`** | `book_unavailable`, `member_suspended`, `loan_limit_reached` | Regla de negocio rechaza la operación. |
| **`201 Created`** | `ok: true` | Préstamo aceptado. |

```typescript
// Esquema del controlador
const input = parseInput(req.body);
if (!input) return res.status(400).json({ error: "invalid_input" });

const result = await borrowBook.execute(input);

if (!result.ok) {
  const map = { book_not_found: 404, member_not_found: 404, book_unavailable: 409, member_suspended: 409, loan_limit_reached: 409 };
  return res.status(map[result.reason]).json({ error: result.reason });
}

return res.status(201).json({ ok: true, loanId: result.loanId, dueDate: result.dueDate });
```

Mañana este **mismo caso de uso** podría ser invocado por Hono, Next.js, un CLI o una cola. **No cambia una regla**.

---

## 8. Raíz de composición (Composition Root)

Es el **único lugar** donde se eligen las piezas concretas. Aquí se construye el grafo de objetos.

| Paso | Acción |
|------|--------|
| **01** | Instanciar adaptadores: `InMemoryBorrowingRepository` (con seed), `SystemClock`, `UuidIdGenerator`. |
| **02** | Instanciar el caso de uso: `new BorrowBook(repository, clock, ids)`. Recibe sus **puertos** (abstracciones). |
| **03** | Instanciar el adaptador driving: `new BorrowBookController(borrowBook)`. |
| **04** | Levantar el servidor/CLI con esas instancias. |

```typescript
// Composition root (esquemático)
const repo = new InMemoryBorrowingRepository(seed);
const clock = new SystemClock();
const ids = new UuidIdGenerator();

const borrowBook = new BorrowBook(repo, clock, ids);
const controller = new BorrowBookController(borrowBook);

app.post("/loans", (req,res) => controller.borrow(req,res));
app.listen(3000);
```

**Regla:** Dominio y aplicación **no conocen** `app.listen`, `express`, `Map`, SQL ni imports de infraestructura.

---

## 9. Tests por zona

Cada zona responde una pregunta distinta.

| Zona | Qué probar | Cómo |
|------|------------|------|
| **Dominio** | ¿Decide bien? | Datos concretos → transición → resultado observable. Sin dobles. Sin framework. |
| **Aplicación (Caso de uso)** | ¿Coordina bien? | Controlar puertos con **spies/stubs/fakes**. Verificar: resultado, **cambios completos** guardados, que se usan reloj/IDs correctos. |
| **Adaptador driving (HTTP)** | ¿Traduce bien? | Inyectar un **stub** del caso de uso. Comprobar mapeo a status/body. Validación de entrada. |
| **Adaptador driven / Composición** | ¿Traduce/conecta bien? | Verificar que el adaptador cumple el contrato. Comprobar que la composición cablea correctamente. |

### 9.1. Caso de uso con puertos controlados

```typescript
const repo = new BorrowingRepoSpy();
const clock = new FixedClock("2026-09-01");
const ids = new FixedIdGenerator(["loan-1"]);
const borrowBook = new BorrowBook(repo, clock, ids);

const result = await borrowBook.execute({ bookId: "b1", memberId: "m1" });

expect(result).toEqual({ ok: true, loanId: "loan-1", dueDate: "2026-09-15" });
expect(repo.saved).toHaveLength(1); // unidad lógica completa
```

### 9.2. Controlador con stub

```typescript
const controller = new BorrowBookController(
  new BorrowBookStub({ ok: false, reason: "member_suspended" })
);

const res = controller.borrow(req, res);
expect(res.status).toBe(409);
expect(res.body).toEqual({ error: "member_suspended" });
```

---

## 10. El grafo de dependencias cuenta la verdad

Las carpetas orientan, pero **los imports y los constructores demuestran la dirección real**.

```text
HTTP adapter  →  Application  ←  Persistence adapter
                    ↓
                   Domain
```

- **Dependencias hacia dentro:** Los adaptadores dependen de puertos definidos en el núcleo. El núcleo **no depende** de los adaptadores.
- **Flujo de ejecución vs flujo de dependencias:** El HTTP llama al caso de uso (flujo hacia dentro), pero las **dependencias en código** apuntan hacia las abstracciones del núcleo.
- **Puertos:** `Driving port` (lo que el exterior quiere hacer) e `Driven port` (lo que el núcleo necesita del exterior).

> Si para ejecutar un test del caso de uso tienes que levantar Express, conectar a Postgres o importar un framework, **los límites no son reales**.

---

## 11. Lecciones clave

| Lección | Explicación |
|---------|-------------|
| **Empezar por el caso de uso** | No por la entidad más genérica. Un corte vertical revela límites con mucho menos esfuerzo. |
| **Resultados explícitos** | `ok: true/false` con `reason` deja la tabla de decisión clara y testable. |
| **Dominio puro** | Las reglas viven en entidades/VOs. Si se pueden probar sin infraestructura, vas por buen camino. |
| **Puertos desde la necesidad** | `BorrowingRepository` existe porque **`BorrowBook` lo necesita**, no porque "necesitemos un repositorio". |
| **Unidad lógica** | `saveBorrowing` refleja que libro + socio + préstamo **cambian juntos**. Ahí empieza la conversación sobre transacciones. |
| **Borde estricto** | Validar forma en HTTP, decidir negocio en dominio, coordinar en aplicación. |
| **Composición explícita** | El wiring va en la raíz de composición. Ahí sí está permitido conocer todos los detalles. |
| **Proporcionalidad** | Completo para **aprender**; preparado para **crecer**. No añadas capas por moda: añádelas cuando el cambio lo justifique. |

> **Regla de oro:** *Transporte decide sobre transporte. Aplicación decide sobre secuencia. Dominio decide sobre reglas.*  
> **Puertos describen necesidades; adaptadores las hacen reales; composición las conecta.**
