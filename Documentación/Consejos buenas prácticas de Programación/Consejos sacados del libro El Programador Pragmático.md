
#### Diseño por contrato (DBC):

La idea de esta forma de diseño es comprobar al inicio si se cumples las precondiciones, ejecutar todo el código ya asumiendo que las precondiciones son correctas, evitando la necesidad de aplicar programación defensiva y al finalizar la lógica de negocio, comprobar que las postcondiciones se han cumplido para garantizar el correcto funcionamiento y que las Invariantes de clases (el estado general) sigue igual que al principio. En caso de incumplirse alguna de las reglas establecida, hacemos que el programa lo grite a los cuatro vientos.

Esto es también muy útil para verificar si todo esta funcionando como debería o en caso de haber un error, saber si viene de las precondiciones (cliente) o postcondiciones/Invariantes de clases al finalizar la lógica (anfitrión).

- **Precondición**: Requisitos que deben ser ciertos antes de llamar a una rutina o método. Es la parte del trato que le toca al cliente (quien llama a la función). Si el cliente no cumple la precondición, la función no tiene la obligación de hacer su trabajo.

- **Postcondición**: Es lo que el método garantiza que será cierto después de terminar su ejecución, siempre y cuando se hayan cumplido las precondiciones. Es la promesa que te hace el proveedor.

- **Invariantes de clases:** condiciones que siempre deben ser verdaderas para un objeto desde la perspectiva de alguien que lo observa desde fuera. Pueden cambiar mientras la función está haciendo su trabajo interno, pero deben ser ciertas antes de que el método empiece y justo cuando el método termina. Son las "leyes de la física" de ese objeto.

##### Ejemplo en Java:

```java
public class CuentaBancaria {
    private double saldo;

    // constructor
    public CuentaBancaria(double saldoInicial) {
        if (saldoInicial < 0) {
            throw new IllegalArgumentException("Precondición fallida: El saldo inicial no puede ser negativo.");
        }
        this.saldo = saldoInicial;
        comprobarInvariante(); // el objeto debe ser valido desde que nace
    }

    /**
     * Retira dinero de la cuenta.
     * * @param cantidad La cantidad a retirar.
     */
    public void retirarDinero(double cantidad) {

        // ---------------------------------------------------------
        // 1. PRECONDICIONES (El contrato que debe cumplir el cliente)
        // ---------------------------------------------------------
        // si el cliente no cumple, "explotamos" rapido y fallamos de forma ruidosa.
        if (cantidad <= 0) {
            throw new IllegalArgumentException("La cantidad a retirar debe ser mayor que cero.");
        }
        if (cantidad > this.saldo) {
            throw new IllegalStateException("No hay saldo suficiente para retirar esa cantidad.");
        }

        // Guardamos el estado previo para poder comprobar la postcondicion luego
        double saldoAnterior = this.saldo;

        // ---------------------------------------------------------
        // --- LOGICA PRINCIPAL DEL NEGOCIO ---
        // ---------------------------------------------------------
        this.saldo -= cantidad; // simulamos la entrega de los billetes y la resta

        // ---------------------------------------------------------
        // 2. POSTCONDICIONES (La promesa que cumple este metodo)
        // ---------------------------------------------------------
        // en caso de fallo a la hora de realizar la operacion.
        assert this.saldo == (saldoAnterior - cantidad) : "Postcondición fallida: El cálculo del saldo es incorrecto.";

        // ---------------------------------------------------------
        // 3. INVARIANTES DE CLASE (Las reglas del objeto)
        // ---------------------------------------------------------
        // antes de terminar el metodo y devolver el control, aseguramos que
        // las reglas universales de la cuenta siguen intactas.
        comprobarInvariante();
    }

    // metodo privado que contiene las "leyes de la física" de nuestra clase
    private void comprobarInvariante() {
        assert this.saldo >= 0 : "Invariante fallida: El saldo de la cuenta ha quedado en negativo.";
    }

    public double getSaldo() {
        return this.saldo;
    }
}

```

#### Maquina de Estados Finitos (FMS):

Modelo donde un sistema solo puede estar en **un estado a la vez**. Solo reacciona a ciertos eventos dependiendo del estado en el que se encuentre, y esos eventos determinan a qué nuevo estado cambiará.

Ejemplo: Torniquete del metro (máquina con barras giratorias). Un torniquete solo puede estar en un número limitado (finito) de situaciones o **"Estados"**:
1. **Bloqueado** (No te deja pasar).
2. **Desbloqueado** (Te deja pasar).

Para cambiar de un estado a otro, tiene que ocurrir un **"Evento"** (o transición):
- Si está **Bloqueado** y el evento es _Insertar Moneda_, pasa al estado **Desbloqueado**.
- Si está **Bloqueado** y el evento es _Empujar_, sigue **Bloqueado** (y no pasas).
- Si está **Desbloqueado** y el evento es _Empujar_, pasas y vuelve al estado **Bloqueado**.
- Si está **Desbloqueado** y el evento es _Insertar Moneda_, sigue **Desbloqueado** (quizás te devuelve la moneda extra).
##### Casos de Uso Ideales
- **Videojuegos:** Controlar el comportamiento de un personaje. Un personaje puede estar en estado `Reposo`, `Corriendo`, o `Saltando`. No puedes pasar de `Reposo` a `Saltando` sin presionar el botón de salto.
- **Procesamiento de Pedidos (E-commerce):** Un pedido pasa por estados estrictos: `Pendiente de Pago` -> `Pagado` -> `Enviado` -> `Entregado`. La FSM asegura que no puedas "Enviar" un pedido que no ha sido "Pagado".
- **Interfaces de Usuario complejas:** Asistentes paso a paso (Wizards) o validación de formularios dinámicos.
##### Ejemplo en Java:

```java
public class Torniquete {

    // 1. Definimos los Estados posibles
    public enum Estado {
        BLOQUEADO,
        DESBLOQUEADO
    }

    // 2. Definimos el estado inicial
    private Estado estadoActual = Estado.BLOQUEADO;

    // 3. Metodo para procesar eventos
    public void procesarEvento(String evento) {
        System.out.println("Evento recibido: " + evento);

        switch (estadoActual) {
            case BLOQUEADO:
                if (evento.equals("INSERTAR_MONEDA")) {
                    System.out.println("-> Moneda aceptada. Desbloqueando...");
                    estadoActual = Estado.DESBLOQUEADO;
                } else if (evento.equals("EMPUJAR")) {
                    System.out.println("-> El torniquete no se mueve. Introduce moneda.");
                }
                break;
            case DESBLOQUEADO:
                if (evento.equals("EMPUJAR")) {
                    System.out.println("-> Persona pasa. Bloqueando de nuevo...");
                    estadoActual = Estado.BLOQUEADO;
                } else if (evento.equals("INSERTAR_MONEDA")) {
                    System.out.println("-> Ya está desbloqueado. Moneda devuelta.");
                }
                break;
        }
        System.out.println("Estado actual: " + estadoActual + "\n");
    }

    // Uso
    public static void main(String[] args) {
        Torniquete miTorniquete = new Torniquete();

        miTorniquete.procesarEvento("EMPUJAR");         // Falla, esta bloqueado
        miTorniquete.procesarEvento("INSERTAR_MONEDA"); // Desbloquea
        miTorniquete.procesarEvento("INSERTAR_MONEDA"); // Devuelve la moneda
        miTorniquete.procesarEvento("EMPUJAR");         // Pasa y bloquea
    }
}
```
#### Patrón Observer y Sub/Pub:
##### El Patrón Observer:

Imagina que estás en una clase y le dices al profesor: _"Por favor, avíseme cuando entregue las notas"_. El profesor anota tu nombre en su lista. Cuando las notas están listas, el profesor lee su lista y le avisa directamente a cada alumno.

En este patrón, el objeto que cambia de estado (el **Sujeto**) tiene una **lista directa de los objetos que quieren ser notificados** (los **Observadores**). El Sujeto y los Observadores **se conocen directamente**.

**Caso de uso ideal:** Interfaces gráficas de usuario (GUI). Por ejemplo, tienes un modelo de datos con la temperatura actual y un botón en la pantalla que muestra esa temperatura. El botón "observa" al modelo; cuando el modelo cambia, le avisa directamente al botón para que se actualice.

###### Ejemplo en Java:

```java
import java.util.ArrayList;
import java.util.List;

// 1. Interfaz del Observador
interface Observer {
    void update(String message);
}

// 2. El Sujeto (El que es observado)
class Subject {
    private List<Observer> observers = new ArrayList<>();

    // Añadir observadores a la lista
    public void addObserver(Observer o) {
        observers.add(o);
    }

    // Notificar a todos directamente
    public void notifyObservers(String message) {
        for (Observer o : observers) {
            o.update(message);
        }
    }
}

// 3. Uso
public class ObserverExample {
    public static void main(String[] args) {
        Subject sensorDeTemperatura = new Subject();

        // Creamos un observador (ej. una pantalla)
        Observer pantalla = message -> System.out.println("Pantalla actualizó: " + message);

        sensorDeTemperatura.addObserver(pantalla); // Se conocen directamente
        sensorDeTemperatura.notifyObservers("La temperatura es 25°C");
    }
}
```

##### El Patrón Publish-Subscribe (Pub/Sub):

Imagina ahora un tablón de anuncios en la universidad. El profesor no tiene una lista de alumnos; simplemente va y cuelga las notas en la sección "Física". Tú, por tu lado, te suscribes leyendo la sección "Física" del tablón. El profesor no sabe quién lee el tablón, y a ti no te importa quién colgó el papel, solo te importa el mensaje.

Aquí hay un intermediario llamado **Broker de Mensajes** o **Bus de Eventos**. Los **Publicadores** envían mensajes al broker, y los **Suscriptores** leen del broker. **Nunca se conocen entre sí**.

**Caso de uso ideal:** Sistemas desacoplados o microservicios. Por ejemplo, en una tienda online, cuando un usuario hace una compra, el servicio de pagos publica un evento "CompraRealizada". El servicio de emails y el servicio de inventario están suscritos a ese evento y reaccionan, sin que el servicio de pagos sepa que existen.

###### Ejemplo en Java:

```java
import java.util.*;

// 1. El Intermediario (Message Broker / Event Bus)
class EventBus {
    // Mapa de "Temas" (Topics) a listas de suscriptores
    private Map<String, List<Subscriber>> topics = new HashMap<>();

    public void subscribe(String topic, Subscriber sub) {
        topics.putIfAbsent(topic, new ArrayList<>());
        topics.get(topic).add(sub);
    }

    public void publish(String topic, String message) {
        if (topics.containsKey(topic)) {
            for (Subscriber sub : topics.get(topic)) {
                sub.receive(message);
            }
        }
    }
}

// 2. Interfaz del Suscriptor
interface Subscriber {
    void receive(String message);
}

// 3. Uso
public class PubSubExample {
    public static void main(String[] args) {
        EventBus bus = new EventBus(); // El intermediario

        // Creamos suscriptores
        Subscriber emailService = msg -> System.out.println("Enviando email: " + msg);
        Subscriber inventoryService = msg -> System.out.println("Actualizando inventario: " + msg);

        // Se suscriben a un tema en el bus (no al publicador)
        bus.subscribe("compra_realizada", emailService);
        bus.subscribe("compra_realizada", inventoryService);

        // Un publicador anónimo envía un mensaje al bus
        bus.publish("compra_realizada", "Orden #1234 pagada");
    }
}
```

##### Diferencias clave entre el patrón Observer y Pub/Sub:
| **Característica** | **Observer**                                                                                                | **Publish-Subscribe**                                                              |
| ------------------ | ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Acoplamiento**   | Medio/Alto (El sujeto y el observador se conocen).                                                          | Muy Bajo (No se conocen, solo conocen al intermediario).                           |
| **Intermediario**  | No existe. Comunicación directa.                                                                            | Sí (Event Bus o Message Broker).                                                   |
| **Sincronía**      | Generalmente **síncrono** (el sujeto se bloquea hasta que todos los observadores terminan de actualizarse). | Generalmente **asíncrono** (el publicador dispara el mensaje y sigue con su vida). |
| **Escala**         | Aplicaciones en un mismo espacio de memoria (dentro del mismo programa).                                    | Sistemas grandes, aplicaciones distribuidas, microservicios.                       |

#### Alternativas a la Herencia:

##### Interfaces / Protocolos:

Las interfaces definen un **contrato**. Especifican _qué_ métodos o propiedades debe tener obligatoriamente una clase, pero no dicen _cómo_ deben funcionar (no proveen código real de forma predeterminada). La clase que "firma" este contrato está obligada a escribir la implementación (el código interno) de esos métodos.

**La analogía del Contrato:** Imagina un contrato para el puesto de "Repartidor". El contrato exige que el empleado sepa `entregarPaquete()`. Al sistema (el jefe) le da absolutamente igual si el repartidor es un humano en bicicleta o un dron volador de última generación; siempre que ambos firmen el contrato y tengan el método `entregarPaquete()`, el jefe puede ponerlos a trabajar sin preocuparse de sus detalles internos.

**Ejemplo en TypeScript/Java:**

```java
// Definimos el contrato (QUE hay que hacer, no el como)
interface Repartidor {
    entregarPaquete(destino: string): void;
}

// El Humano firma el contrato y define su propia forma de hacerlo
class Humano implements Repartidor {
    entregarPaquete(destino: string) {
        console.log(`Pedaleando en bicicleta y sudando hacia ${destino}`);
    }
}

// El Dron firma el mismo contrato, pero lo hace a su manera
class Dron implements Repartidor {
    entregarPaquete(destino: string) {
        console.log(`Volando en línea recta a 60km/h hacia ${destino}`);
    }
}

// POLIMORFISMO: Podemos agruparlos porque el lenguaje sabe que ambos cumplen el contrato
const misRepartidores: Repartidor[] = [new Humano(), new Dron()];

for (let repartidor of misRepartidores) {
    // Al sistema le da igual quién es quien. Sabe que el metodo existe.
    repartidor.entregarPaquete("Calle Mayor 123");
}
```

**¿Por qué es mejor?**
- **Polimorfismo seguro:** Puedes tratar objetos de clases completamente distintas (un Humano y un Dron, que no comparten ningún "árbol genealógico") de forma estandarizada.
- **Desacoplamiento brutal:** Tu código principal no depende de implementaciones concretas, sino de abstracciones. Si mañana inventas un `RepartidorRobotico`, el código que gestiona la lista de repartidores no tendrá que cambiar ni una sola línea.
- **Facilita el Testing:** Al depender de interfaces, es facilísimo crear objetos falsos (_Mocks_) que simulen cumplir el contrato para probar tu código sin tener que instanciar sistemas complejos (ej. base de datos reales).
##### Delegación:

La delegación se basa en el principio de **"Favorecer la composición sobre la herencia"** (un objeto _tiene un_ componente, en lugar de _ser un_ componente).

En lugar de heredar los métodos de otra clase, la clase guarda una instancia de esa otra clase y le "pasa la pelota" (delega) cuando le piden hacer algo.

**La analogía del Jefe:** Imagina que eres el gerente de un restaurante. Un cliente te pide que cocines una pizza. La clase `Gerente` no hereda de la clase `Cocinero`. En su lugar, **_tiene_ un empleado que es cocinero**, y cuando el cliente te pide la pizza, el gerente **delega** la tarea al cocinero.
- El Problema de la Herencia:
```python
# Un Coche NO es un Motor. Si heredas, el coche hereda cosas de motor que no tienen sentido.
class Coche(Motor):
    pass
```

- Solución con la Delegación:
```python
class Motor:
    def arrancar(self):
        print("Brum brum...")

class Coche:
    def __init__(self):
        # El coche "TIENE UN" motor (Composicion)
        self.motor = Motor()

    def arrancar_coche(self):
        # El coche "DELEGA" la accion de arrancar al motor
        self.motor.arrancar()
```

- **¿Por qué es mejor?** Es mucho más flexible. Si mañana quieres cambiar el `Motor` por un `MotorElectrico`, puedes hacerlo fácilmente pasándoselo al coche al crearlo. Con la herencia, estarías atrapado para siempre en la jerarquía familiar del motor original.
##### Mixins / Traits:

Si las interfaces te dicen _QUÉ_ métodos debe tener tu clase (pero tú tienes que escribir el código de esos métodos cada vez), los Mixins te dan el _CÓMO_. Un Mixin es **un paquete de métodos ya implementados** que puedes "inyectar" o "mezclar" (mix-in) en cualquier clase, sin que esas clases tengan que ser de la misma familia.

**La analogía del Superpoder:** Imagina que tienes una clase `Pajaro` y una clase `Avion`. Ambos pueden volar. Si usas herencia pura, tendrías que crear una clase padre `CosaQueVuela`. Pero un pájaro (animal) y un avión (vehículo) no tienen nada en común lógicamente. En lugar de eso, creas un **Mixin** llamado `HabilidadDeVolar` y se lo inyectas a ambos.

Ejemplo en Ruby:

```ruby
# Definimos el Mixin (un módulo con codigo real, no solo firmas vacias)
module HabilidadDeVolar
    def volar
        print("¡Estoy volando por los cielos!")
    end
end

class Pajaro
    # Inyectamos el Mixin
    include HabilidadDeVolar
end

class Avion
    # Inyectamos el mismo Mixin en una clase que no tiene nada que ver
    include HabilidadDeVolar
end

piolin = Pajaro.new()
piolin.volar() # Imprime: ¡Estoy volando por los cielos!
```

- **¿Por qué es mejor?**
- **Reutilización pura:** Tienes el código escrito una sola vez (a diferencia de las interfaces donde tienes que reimplementarlo).
- **Sin árboles genealógicos rígidos:** Evitas crear jerarquías de clases absurdas solo para compartir un método de utilidad (como guardar en base de datos, o convertir a formato JSON). Puedes "pegarle" el trait `ConvertibleAJson` a un `Usuario`, a una `Factura` o a un `Coche`.

**Información adicional (podría aprovechar también el beneficio de las interfaces):**
En muchos lenguajes modernos (como Scala con sus _Traits_, Dart con sus _Mixins_, o Swift con sus _Protocolos con implementación por defecto_), un Trait/Mixin **también actúa como una interfaz**.
Es decir, el lenguaje reconoce el Mixin como un "tipo de dato". Por lo tanto, **sí podrías** crear una lista y meter ahí cualquier objeto que use ese Mixin.

```java
// Lista que acepta cualquier objeto que tenga el Trait "HabilidadDeVolar"
List<HabilidadDeVolar> voladores = [Pajaro(), Avion(), Superman()];

for (volador in voladores) {
    volador.volar(); // Funciona perfectamente
}
```


##### Resumen comparativo:

- **Interfaz:** _"Te prometo que tendré estos métodos, pero yo mismo escribiré cómo funcionan"_.
    **Utilidad:** Necesito que varias clases distintas compartan un contrato para tratarlas igual (polimorfismo)

- **Delegación:** _"Yo no sé hacer esto, pero tengo guardado a un amigo (objeto) internamente que sí sabe, le diré que lo haga por mí"_.
    **Utilidad:** Necesito que mi clase haga algo complejo que ya sabe hacer otro objeto, pero no quiero heredar toda su basura.

- **Mixins/Traits:** _"Acabo de descargar un paquete de habilidades nuevas, me las voy a instalar para saber hacer cosas nuevas al instante"_.
    **Utilidad:** Tengo una funcionalidad genérica ("hacer log", "convertir a JSON", "volar") que quiero "enchufar" a muchas clases que no tienen relación entre sí para no repetir código.
