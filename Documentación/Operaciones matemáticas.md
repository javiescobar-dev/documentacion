##### Obtener el último dígito de un número:

```c
    // obtenemos el ultimo digito de un numero
    // ya que al dividir entre 10 el resto es el ultimo digito
    int digit = number % 10;
```

##### Quitar el último dígito a un número:

```c
    // quitamos el ultimo digito a un numero
    // ya que al dividir entre 10 el cociente es el numero sin el ultimo digito
    int digit = number / 10;
```

##### Recorrer los dígitos de un número de derecha a izquierda

```c
    // iteramos mientras el numero sea mayor a 0, cada iteracion dividimos entre 10, por lo que vamos quitando un digito en cada iteracion
    while (number > 0)
    {
        // obtenemos el ultimo digito de un numero
        int digit = number % 10;

        // hacer lo que sea con el digito obtenido...

        // quitamos el ultimo digito y pasamos al siguiente
        numer /= 10;
```
