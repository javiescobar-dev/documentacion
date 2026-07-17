
#### Configurar git:

```sh
git config --global user.name "Tu Nombre"
git config --global user.email "tu@email.com"
```

#### Ver configuración actual:

```sh
git config --global --list
```

#### Inicializar una carpeta como repositorio git:

```sh
git init
```

#### Renombrar una rama:

```sh
# renombra la rama actual al nombre establecido
git branch -m nombre_rama
```

#### Ver status del repositorio git:

```sh
# podemos ver que ficheros estan incluidos en git, cuales no, cuales estan pendientes de añadirse en un commit, etc
git status
```

#### Añadir cambios al repositorio git para commitearlos posteriormente:

```sh
# Esto añade los cambios al stage, pero hasta que no se realice el commit no estan añadidos el repositorio
git add nombre_archivo

# tambien podemos usar . para añadir todos
git add .
```

#### Crear un commit para añadir los cambios ya al repositorio en una rama:

```sh
# -m es message, para incluir el mensaje del commit directamente por comando
git commit -m "Descripcion del commit"
```

#### Ver el log de commits realizados en el repositorio:

```sh
# muestra los commits con la informacion por defecto
git log

# muestra los commits con la informacion, pero algo mas visual
git log --graph

# muestra los commits en una sola linea por commit, mas compacto
git log --graph --pretty=oneline

# similar al anterior, pero mas corto
git log --decorate --all --oneline
```

#### Navegar entre ramas y establecerte en un punto de los cambios del repositorio:

```sh
# nos vamos a los cambios de ese fichero en concreto en el punto del ultimo commit (por si lo hemos modificado y no lo hemos subido aun los cambios, volvemos a ver el fichero sin los cambios añadidos, como esta realmente en el repo en su ultimo commit)
git checkout nombre_fichero

# Tambien podemos indicar el nombre de un hash en concreto para ir a esos cambios:
git checkout hash_x

# por ejemplo
git checkout e7c20003d107baf06187307ae4b755504bb1d0e3

# Tambien podemos navegar a un tag especifico con:
git checkout tags/nombre_del_tag
```

#### Deshacer los cambios que no han sido commiteados (resetear):

```sh
# los cambios que no han sido commiteados los elimina
git reset

# tambien podemos usar un hash para retorceder a un cambio concreto pero sin perder los cambios posteriores
git reset hash_x

# si utilizamos el flag hard si que deshacemos todos los cambios al volver a un commit especifico (tambien podemos usarlo para volver a cambios posteriores)
git reset --hard hash_x
```

#### Crear un alias para ejecutar un comando git complejo usando un nombre sencillo:

```sh
# git config --global alias guarda un alias en el fichero global, en este caso el alias se llama tree y el comando es lo que hay entre comillas
git config --global alias.tree "log --graph --decorate --all --oneline"

# una vez almacenado, al utilizar git nombre_alias, en este caso git tree, ejecuta el comando que guardamos entre comillas, simplificando su uso
```

#### Ver los cambios pendientes de añadir a un commit o de diferentes ramas:

```sh
# para ver los cambios pendientes a añadir
git diff

# para ver diferencias entre la rama actual y otra
git diff otra_rama
```

#### Ver todo el historial de commits realizados en git:

```sh
# en caso de que hayamos hecho git reset --hard y queramos volver a donde estabamos previamente, podemos usar git reflog para mostrar todos los cambios y poder volver a ellos
git reflog
```

#### Usar etiquetas para referenciar un commit concreto:

```sh
# utilizamos tags para estblecer un punto importante en el historial del repositorio
git tag nombre_del_tag
```

#### Crear una rama en el repositorio:

```sh
git branch nombre_rama
```

#### Cambiar de rama:

```sh
git switch nombre_rama
```

#### Eliminar una rama:

```sh
git branch -d nombre-rama

# aunque la rama se borre, en git reflog persisten los commits, podemos rescatarlos aunque no esten en la rama como tal
```
#### Fusionar cambios de dos ramas:

```sh
# fusiona la rama actual con la rama que indiques en el comando
# al ejecutar el comando hay que añadir el nombre de un nuevo commit que se crea
git merge rama_a_fusionar
```

#### Almacenar o guardar cambios que no quieres añadir a un commit todavia para mantenerlos y poder cambiar de rama:

```sh
# esto almacenaria de forma temporal los cambios
git stash

# ahora podrias cambiar de rama
git switch main

# haces lo que tengas que haces y vuelves a la rama previa
git switch login

# y recuperamos los cambios almacenados temporalmente (los ultimos, como una pila de platos)
git stash pop

# para ver la lista de stash que tienes
git stash list

# quitar el ultimo stash almacenado en la lista (pero sin recuperar los cambios)
git stash drop

# quitar un stash especifico de la lista de stash almacenados (sin recuperar cambios) donde el valor 1 corresponderia al valor que te muestra git stash list
git stash drop stash@{1}

# para limpiar toda la lista de stash almacenados
git stash clear
```
