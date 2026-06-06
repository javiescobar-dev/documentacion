
#### Diseño por contrato (DBC):

La idea de esta forma de diseño es comprobar al inicio si se cumples las precondiciones, ejecutar todo el código ya asumiendo que las precondiciones son correctas, evitando la necesidad de aplicar programación defensiva y al finalizar la lógica de negocio, comprobar que las postcondiciones se han cumplido para garantizar el correcto funcionamiento y que las Invariantes de clases (el estado general) sigue igual que al principio. En caso de incumplirse alguna de las reglas establecida, hacemos que el programa lo grite a los cuatro vientos.

Esto es también muy útil para verificar si todo esta funcionando como debería o en caso de haber un error, saber si viene de las precondiciones (cliente) o postcondiciones/Invariantes de clases al finalizar la lógica (anfitrión).

- **Precondición**: Requisitos que deben ser ciertos antes de llamar a una rutina o método. Es la parte del trato que le toca al cliente (quien llama a la función). Si el cliente no cumple la precondición, la función no tiene la obligación de hacer su trabajo.

- **Postcondición**: Es lo que el método garantiza que será cierto después de terminar su ejecución, siempre y cuando se hayan cumplido las precondiciones. Es la promesa que te hace el proveedor.

- **Invariantes de clases:** condiciones que siempre deben ser verdaderas para un objeto desde la perspectiva de alguien que lo observa desde fuera. Pueden cambiar mientras la función está haciendo su trabajo interno, pero deben ser ciertas antes de que el método empiece y justo cuando el método termina. Son las "leyes de la física" de ese objeto.

Ejemplo en Java:

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
