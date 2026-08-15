# ARQUITECTURA — RPG Battle (CS50x Final Project)

> Documento de referencia del proyecto. Actualizar cuando haya cambios de diseño.
> Nota: este archivo esta en `.gitignore` (no se sube al repo) junto con `AGENTS.md`.

## 0. Reglas de colaboracion

- El desarrollo se hace CONJUNTAMENTE con el usuario, paso a paso.
- Nunca implementar un modulo completo por cuenta propia: explicar cada parte
  primero y dejar que el usuario la escriba con nuestra ayuda.
- Tras cada funcionalidad terminada, actualizar la seccion "Features" de
  `README.md` (en ingles). Documentacion incremental, no al final.
> Repo: `https://github.com/javiescobar-dev/rpg-battle-cs50x` (público, rama `main`).
> Deadline CS50x: 2026-12-31.

## 1. Stack y decisiones técnicas

- Python 3.12.x
- Juego: **pygame-ce**
- Launcher: **customTkinter 5.x**
- Rutas por SO: **platformdirs**
- Comparar versiones: **packaging** (launcher)
- Descargas HTTP: **urllib** (stdlib)
- Distribución del juego: assets por SO (1 zip por SO) en GitHub Releases, mismo tag.
- Distribución del launcher: binario por SO, generado por CI.
- Código portable siempre (pathlib, sin `LOCALAPPDATA` fijo ni `\`).

## 2. URLs canónicas (constantes en el launcher)

- API releases: `https://api.github.com/repos/javiescobar-dev/rpg-battle-cs50x/releases/latest`
- Noticias: `https://raw.githubusercontent.com/javiescobar-dev/rpg-battle-cs50x/main/news/news.json`

## 3. Estructura del repo

```
game/            # Pygame: main.py, config.py, entities.py, battle.py, score.py, requirements.txt + ui.py (Fase 2) + assets.py y assets/ (Fase 3)
launcher/        # customTkinter: main.py, updater.py, news.py, paths.py, requirements.txt
news/news.json   # Feed de noticias
docs/ARQUITECTURA.md
README.md        # El de CS50x (título + video + ~750 palabras)
.github/workflows/build.yml   # CI multi-SO
```

## 4. Juego (pygame)

- Batalla por turnos Heroe vs Enemigo.
- Acciones del heroe: **Attack** (golpe fisico, sin mana) y tres habilidades tipo
  *rune* (inspiradas en Suikoden II) que consumen mana: **Spell** (daño),
  **Guard** (mitiga daño el proximo turno) y **Heal** (cura HP). Ademas:
  **Potion** (item limitado) y **Flee** (probabilidad segun velocidad).
- `Spell` es una mecanica generica; el nombre visible depende de quien la lanza
  (Heroe: "Fireball"; Enemigo: "Shadow Bolt") via `spell_name` en cada personaje.
- `Guard` es una mecanica comun: vive en la clase base `Character` (`self.guard`),
  asi que tanto Heroe como Enemigo pueden mitigar; `take_damage` lo aplica, lo consume
  y **devuelve el daño real aplicado** (tras mitigacion), que es lo que usa el log.
- `cast_skill` (Hero) devuelve `False` si falla (mana insuficiente) y el valor del
  efecto si tiene exito; `use_potion` devuelve `False` si no quedan pociones.
- Orden de turnos dentro de una ronda: el **jugador actua primero siempre** y luego
  el enemigo (la velocidad solo influye en huir).
- Si el jugador lanza un skill sin mana suficiente, **el turno se consume** igualmente
  (le penaliza: el enemigo responde).
- `Character` (base) -> `Hero` / `Enemy`. Daño con fórmula + random. IA enemiga ponderada.
- Estados UI pygame (Fase 2): MENU -> BATTLE -> ANIM -> (BATTLE | END) -> MENU, con STATS accesible desde MENU. Detalle en 4.3.
- File I/O: `score.py` guarda resultados en JSON local.

### 4.1 Balance (Fase 1, ajustable en `config.py`)

- Heroe: HP 100 / MP 30 / ataque 15 / defensa 10 / velocidad 12.
- Enemigo: HP 80 / MP 20 / ataque 13 / defensa 8 / velocidad 10.
- Spell: coste 8 MP, daño = ataque x 1.8 + random.
- Guard: coste 4 MP, -50% daño recibido el proximo turno (se consume al recibir daño).
- Heal: coste 6 MP, cura 30 HP.
- Potion: cura 25 HP; se empieza con 3.
- Formula de daño: `max(1, ataque_atacante - defensa_defensor x 0.5) + random(0..5)`.
- Golpe critico: 10% de probabilidad (`CRIT_CHANCE`) de multiplicar el daño final x1.5
  (`CRIT_MULTIPLIER`); `calculate_damage` devuelve `(damage, is_crit)`.
- Flee: 50% + (vel_heroe - vel_enemigo) x 0.05, clamp 25%..90%; si falla, el enemigo ataca.
  (Con las stats base, huida ~60%.)
- IA enemiga: pesos 0.5 Attack / 0.3 Spell / 0.2 Guard (suman 1.0); sin mana suficiente cae a Attack.

### 4.2 `scores.json`

- En Fase 1 vive en `game/scores.json` (ignorado en git: es dato de usuario generado en runtime);
  se migrara a platformdirs en la fase del launcher.
- API de `score.py`: `load_scores()` (defensiva: `[]` si falta o esta corrupto),
  `add_result(result, turns, hero_hp_left, hero_hp_max, enemy_name)` y
  `summary()` (devuelve `total_battles`, `victories`, `defeats`, `flees`, `most_common_enemy`).
- Cada registro: fecha/hora ISO, resultado (victory/defeat/fled), turnos,
  HP restante y maximo del heroe, y nombre del enemigo.

### 4.3 Fase 2 — UI pygame (pasos 1-6 completados; 3 mejoras cerradas abajo)

Objetivo: que se parezca a Suikoden II (personajes visibles + pequena animacion)
reutilizando el motor de batalla de la Fase 1 tal cual. Decisiones cerradas con el usuario:

- Ventana **960x640**, **FPS 60**, interaccion por **teclado (1-4, como la consola) y clic de raton**.
- Entrada: **pygame-ce** (ya en `game/requirements.txt`). PASO 0 completado:
  `pygame-ce 2.5.7` instalado (wheel nativo para CPython 3.14) junto con `platformdirs 4.11.1`.
- Alcance visual Fase 2: personajes **placeholder** dibujados con primitivas
  (rectangulo/poligono + cara) y animaciones **basicas** (lunge, flash, recoil,
  numero de daño flotante). Sprites reales, frames y efectos spell/pocion en Fase 3.
- Disposicion con **profundidad** estilo Suikoden II (no un espejo simple): el enemigo
  queda arriba a la izquierda y **mas lejos** (`ENEMY_SCALE=0.8`, mas pequeno); el heroe
  abajo a la derecha y **mas cerca** (`HERO_SCALE=1.0`).
- **Tarjeta de estado fija del heroe** arriba a la derecha (`HERO_CARD_RECT`, 150x54 con
  marco: nombre + barras HP/MP). La **vida/mana del enemigo se oculta** para mantener la
  tension del combate (como Suikoden II); solo se muestra su **placa de nombre**
  (`ENEMY_NAME_RECT`) encima de su sprite.
- Numeros flotantes **coloreados por tipo**: daño amarillo, critico rojo (mas grande),
  curacion/pocion verde (`+N`), fallos/guard gris.
- Arquitectura de animacion disenada ya desde ahora para ser extensible en Fase 3.

**Refactor del motor (`battle.py`) — eventos estructurados**
- `Battle.log` pasa de strings a **dicts** (eventos). La logica del motor NO cambia
  (misma formula, turnos, mana, guard, resultado). Solo cambia lo que se guarda.
- Esquema de eventos:
  - `{"action": "attack"|"spell", "attacker", "defender", "damage", "is_crit", "was_guarding"}`
  - `{"action": "guard"|"heal"|"potion", "actor", "amount"}` (amount solo en heal/potion)
  - `{"action": "mana_fail", "actor", "skill"}`
  - `{"action": "potion_fail", "actor"}`
  - `{"action": "flee_success"|"flee_fail", "actor"}`
  - `{"action": "defeated", "defender"}`
- `format_event(event) -> str` (en `battle.py`): genera los mismos textos que la Fase 1
  para que el log visual y la semantica no cambien.
- `attack` y `spell` son eventos distintos a proposito: la UI los distingue
  (en Fase 3 `SpellAnimation` hereda de `AttackAnimation` y anade brillo sin tocar el resto).

**`ui.py` — capa de render y animacion (nuevo)**
- `Menu`: widget reutilizable para listas de opciones. Un solo `cursor` compartido
  por teclado y raton (al mover el raton sobre una opcion, el cursor la sigue);
  `move_cursor(delta)` con modulo (wrap around), `index_at(pos)` para hit-testing
  de filas (`(y - rect.top) // row_height`), `handle_click(pos)` y `draw(surface)`
  que resalta la fila del cursor con `COLOR_ACCENT`. El submenu se implementa
  sustituyendo `options` y reiniciando `cursor` a 0.
- `CharacterSprite`: envuelve un `Character`; posicion base + offset animado + `scale`
  (cuerpo y ojos proporcionales: `HERO_SCALE=1.0`, `ENEMY_SCALE=0.8` para la
  profundidad). En Fase 2 dibuja el placeholder procedural; en Fase 3 solo cambia el
  interior de su `draw()` para cargar imagenes desde `assets.py` — el resto de la UI no se toca.
- Clase base `Animation` con `update(dt)` / `is_done()` / `draw(surface)`:
  - `AttackAnimation`: lunge (el atacante recorre casi toda la distancia hasta el
    objetivo, se detiene a `LUNGE_GAP=30px` del borde, ~0.35s) -> flash de impacto
    sobre el defensor (~0.15s) -> recoil (el defensor retrocede, ~0.15s) -> numero
    de daño flotante (~0.8s, sube y se desvanece) -> el atacante vuelve.
  - `SpellAnimation`: subclase de `AttackAnimation` (Fase 2 identica; Fase 3 hereda y
    anade el brillo de lanzamiento).
  - Clases simples para guard / heal / potion / flee_success / flee_fail /
    mana_fail / potion_fail / defeated.
- `EventPlayer`: recibe la lista de eventos nuevos, programa la `Animation` de cada uno
  y avanza con `dt` (nunca `sleep`; el bucle del juego nunca se bloquea).
- Rendering: **tarjeta fija del heroe** arriba-derecha (`draw_hud` con `rect` fijo:
  nombre + barras HP/MP con relleno proporcional y marco), **placa de nombre del
  enemigo** encima de su sprite (`draw_enemy_name`; sin vida/mana: tension), panel de
  **log** (texto de `format_event`), menus dibujados.
- Layout por zonas: **arena arriba** (enemigo arriba-izquierda y menor escala = mas
  lejos; heroe abajo-derecha y escala 1.0 = mas cerca), **panel de menu abajo**,
  **log** a su derecha.
- Constantes de UI en `config.py`: tamano de ventana, FPS, colores, fuentes,
  posiciones del layout (incluidos `HERO_X/Y`, `ENEMY_X/Y`, `HERO/ENEMY_SCALE`,
  `HERO_CARD_RECT`, `ENEMY_NAME_RECT`) y duraciones de animacion
  (LUNGE/FLASH/RECOIL/FLOAT en segundos).

**`main.py` — bucle pygame + maquina de estados (reescrito)**
- Estados: **MENU** (Play / Statistics / Quit), **STATS** (muestra `summary()`),
  **BATTLE** (espera la accion del jugador), **ANIM** (`EventPlayer` reproduciendo
  eventos), **END** (Victory / Defeat / Fled + summary).
- Flujo: MENU -> (Play) BATTLE -> (accion) `battle.player_turn(...)` resuelve el turno
  al instante -> nuevos eventos = `battle.log[prev_len:]` -> ANIM -> si
  `battle.is_finished`: `add_result(...)` y END; si no, vuelve a BATTLE.
- Bucle principal: `while running: handle_events -> update(dt) -> draw` con `clock.tick(FPS)`.
- **Menu de combate estilo Suikoden II** (dos niveles en el panel `MENU_RECT`):
  nivel superior `[1] Attack [2] Skill [3] Potion [4] Flee`; al elegir Skill el panel
  se sustituye por el submenu de runas `[1] Fireball [2] Guard [3] Heal [4] Back`
  (Back/Esc vuelve al nivel superior). Navegacion: flechas + Enter, teclas 1-4 y clic.
- El turno se resuelve al instante y los eventos nuevos se guardan en `pending_events`,
  que es el "mensajero" hacia ANIM. Desde el paso 5, ANIM los entrega a un `EventPlayer`
  que los reproduce como animaciones en secuencia (el paso 4 era un passthrough que los
  anadia al log de golpe; ahora el log fluye en vivo via callback `on_event`).
- MENU/STATS/END/ANIM se dibujan con helpers por pantalla (`draw_menu_screen`,
  `draw_battle_screen`, `draw_stats_screen`, `draw_end_screen`). El flujo esta
  cableado de punta a punta (MENU -> BATTLE -> ANIM -> END/BATTLE -> MENU) y al terminar
  la batalla se registra `add_result(...)`. END y STATS son pantallas reales que
  muestran `summary()` (formateado con `format_summary_lines`, apilado con
  `draw_lines_centered`, pista `draw_hint`) y vuelven a MENU con cualquier tecla o clic;
  END anade el titulo de resultado (dict RESULT_* -> texto) y una linea de sabor
  (`{hero} vs {enemy} - {turns} turns`).

**Orden de implementacion (paso a paso con el usuario, segun AGENTS.md)**
1. [OK] Instalar pygame-ce + esqueleto de ventana (constantes UI en config + main minimo).
   Ventana 960x640 a 60 FPS, loop basico (`set_mode`, `Clock.tick(FPS)/1000`,
   QUIT event, `fill` + `flip`). Constantes de UI en `config.py` (colores, fuentes,
   layout, duraciones de animacion, dimensiones de barras).
2. [OK] Refactor de `battle.py` a eventos + `format_event` (verificado: los textos
   quedan iguales con un script golden-code de 200 seeds contra la Fase 1).
   `Battle.log` ahora guarda dicts; el motor no cambia. Detalles del esquema arriba.
   Decisiones cerradas: mensaje de guard unificado para heroe y enemigo; eliminado el
   branch "did nothing" de `player_turn`; `was_guarding` se captura ANTES de `take_damage`.
3. [OK] Layout + HUD / barras de vida + log.
   `ui.py` con `CharacterSprite` (placeholder procedural + cara, con `offset` para
   animaciones), `draw_bar`/`draw_hud` (nombre + barras MP/HP sobre cada personaje) y
   `LogPanel` (deque maxlen=6, wrap greedy con `font.size()`, anclado abajo con un
   solo `reversed(lines)`). Layout final: `MENU_RECT=(40,450,300,180)`,
   `LOG_RECT=(360,450,560,180)`. `main.py` tenia un preview temporal que simulaba 3
   turnos (attack, spell, potion) para ajustar el diseno; el paso 4 lo elimino al
   reescribir `main.py` como maquina de estados.
   Nota: tras el paso 4, `draw_hud` dejo de dibujar sobre cada personaje y paso a ser la
   **tarjeta fija del heroe** (`HERO_CARD_RECT`) arriba-derecha, con `draw_enemy_name`
   para el rotulo del enemigo y `scale` en `CharacterSprite` (profundidad).
4. [OK] Menus + maquina de estados completa.
   Hecho: widget `Menu` en `ui.py` (cursor unico compartido por teclado y raton;
   submenu sustituyendo `options` y reseteando `cursor`); `main.py` reescrito con
   estados MENU/STATS/BATTLE/ANIM/END y bucle `handle_events -> update -> draw` con
   helpers por pantalla; pantalla MENU funcional (Play / Statistics / Quit); flujo de
   combate completo: Play -> `start_battle()` -> menu de combate de dos niveles
   (Suikoden II): `[1]Attack [2]Skill [3]Potion [4]Flee`, Skill abre el submenu
   `[1]Fireball [2]Guard [3]Heal [4]Back` (Back/Esc vuelve); las acciones resuelven
   el turno al instante (`resolve_turn`) y guardan los eventos nuevos en
   `pending_events` para ANIM (passthrough: los anade al log y decide BATTLE/END);
   al terminar la batalla se registra `add_result(...)`. Pantallas END (titulo de
   resultado + linea de sabor con turnos + resumen) y STATS (`summary()`) reales,
   ambas con `draw_lines_centered`/`draw_hint` y vuelta a MENU con tecla/clic.
   Leccion anotada: el menu muestra el nombre visible ("Fireball") pero el motor usa
   la clave (`"spell"`) — un bug de "not enough mana" sin gastar MP venia de pasar
   "fireball" a `player_turn("skill", ...)`.
5. [OK] Animaciones (`EventPlayer` + `Animation`/`AttackAnimation`/`SpellAnimation`/`TextAnimation`).
   - `Animation` base: `update(dt)` / `is_done()` / `draw(surface)`, con `duration` + `elapsed`.
   - `AttackAnimation`: fases derivadas de `elapsed` (lunge -> flash -> recoil -> float) sin
     maquina de estados; flash con `SRCALPHA` (alpha desciende) y numero de daño flotante que
     sube y se desvanece (`set_alpha`), coloreado por tipo (critico rojo y mas grande con `!`).
   - `SpellAnimation(AttackAnimation)`: subclase vacia en Fase 2; en Fase 3 hereda y anade el
     brillo de lanzamiento sin tocar el resto.
   - `TextAnimation`: generica para guard/heal/potion/flee/mana_fail/potion_fail/defeated
     (tabla texto+color: guard azul, heal/potion verde `+N`, fallos blanquecinos, KO rojo).
   - `EventPlayer`: cola FIFO, reproduce en secuencia con `dt`, `is_idle()`, callback
     `on_event(event)` para el log en vivo; mapea Character->sprite con un dict.
   - `main.py`: ANIM vuelca `pending_events` al player una vez y avanza con `update(dt)`;
     solo al quedar `is_idle()` decide BATTLE/END; `draw_battle_screen` recibe
     `event_player` y lo dibuja encima de los sprites.
   - Correccion de diseno: el recoil empuja al defensor EN LA DIRECCION DEL GOLPE
     (`+sign`, igual que el lunge), no en la contraria (fisica de transferencia de impulso).
6. [OK] Verificacion y documentacion (smoke test + README Features + sync ARQUITECTURA).

**Mejoras post-paso 6 (cerradas con el usuario, implementadas y verificadas)**
1. **Textos flotantes con contorno y fade tardio** — helper compartido
   `render_outlined_text(font, text, color)` (pre-renderiza el texto con contorno
   negro 1px sobre superficie SRCALPHA) usado por `_draw_float` y `TextAnimation`;
   ambos permanecen opacos hasta `FLOAT_FADE_START=0.5` de su duracion y solo
   despues se desvanecen (antes se desvanecia desde el inicio). Fuentes 24->26
   (flotante) y 34->36 (critico).
2. **HUD de vida/mana progresivo (snapshots por evento)** — el motor sigue intacto:
   nuevo helper `Battle._append(event)` que centraliza todos los `log.append` y anade
   al dict las claves `hero_hp`/`hero_mp`/`enemy_hp`/`enemy_mp` en el momento exacto
   del evento. `format_event` ignora esas claves (textos identicos). En la UI,
   `CharacterSprite` expone `display_hp`/`display_mp` (inicializados al valor real)
   que `draw_hud` dibuja; `EventPlayer` los sincroniza al empezar cada animacion:
   el MP del actor se gasta al iniciarse su ataque/hechizo, el HP del defensor cae
   EXACTAMENTE en el impacto via callback `on_impact` (disparado al entrar en la fase
   flash de `AttackAnimation`), heal/potion sube el HP del actor, defeated vacia la
   barra. Resultado: cada ronda se revela progresivamente, no toda de golpe.
3. **Lunge de recorrido completo (estilo Suikoden II)** — antes el atacante avanzaba
   un tramo fijo (`LUNGE_DISTANCE=40 * scale`); ahora recorre casi toda la separacion
   entre ambos (`max(1, sign*(defender.x-attacker.x) - LUNGE_GAP)`), deteniendose a
   `LUNGE_GAP=30px` del objetivo, y la duracion sube a `LUNGE_DURATION=0.35s`.
   Constantes viejas eliminadas de `config.py`.

**Verificacion de las mejoras**: smoke test de regresion repetido con SDL dummy
(partida completa real via teclado sintetico, BATTLE->ANIM->END->add_result): sin
crash y resultado registrado. Revision visual pendiente con la ventana real.

**Mejoras 2 (PLAN propuesto, pendiente de implementar punto a punto con el usuario)**

Objetivo: reparto mas centrado estilo Suikoden II, ataque en diagonal con salto
de vuelta, y hechizos lanzados a distancia (no con el movimiento del ataque fisico).
Decisiones cerradas con el usuario: proyectil volando para hechizos, arco simple
(sin rotacion) para la vuelta, y valores de layout los propuestos abajo.

1. **Layout centrado** (`config.py` + orden de dibujo)
   - `ENEMY_X/Y` (260,210) -> (400,180) y `ENEMY_SCALE` 0.8 -> 0.7 (mas lejos).
   - `HERO_X/Y` (700,300) -> (590,360) y `HERO_SCALE` 1.0 -> 1.1 (mas cerca).
   - `ENEMY_NAME_RECT` se deriva solo de las constantes, no requiere cambios.
   - `main.py` `draw_battle_screen`: dibujar enemigo PRIMERO y heroe DESPUES (el
     mas cercano a camara va encima cuando se superponen durante el lunge).
2. **Ataque en diagonal + salto de vuelta** (`ui.py` + `config.py`)
   - `CharacterSprite`: sustituir `offset` (escalar X) por `offset_x` + `offset_y`;
     `draw()` aplica ambos a `x` e `y` (cuerpo y ojos).
   - `AttackAnimation.__init__`: vector unitario `(ux, uy)` hacia el defensor y
     punto de aterrizaje `land = defender - ux*LUNGE_GAP` (se elimina `sign`).
   - `update()`: lunge/flash avanzan en diagonal hasta `land` (X e Y); el recoil
     empuja al defensor en diagonal siguiendo el vector del golpe
     (`ux*recoil*p`, `uy*recoil*p`); en float la vuelta es un SALTO DE ESPALDAS
     con arco: `offset_y = land_y*(1-r) - RETURN_JUMP_HEIGHT*sin(pi*r)`, ojos
     siempre hacia el objetivo (sin rotacion). Nueva constante
     `RETURN_JUMP_HEIGHT = 30`.
   - Refactor: guardar los instantes de fase (`t_flash`, `t_float_start`) como
     atributos del anim para reutilizar `_draw_flash`/`_draw_float` en hechizos
     (hoy se calculan desde constantes fijas). Flash y numero flotante siguen al
     defensor con `defender.x + offset_x`.
3. **Hechizos a distancia** (`SpellAnimation` reescrita)
   - Subclase de `AttackAnimation` reutilizando flash, numero flotante y `on_impact`,
     pero con fases `fly -> flash -> float`: sin lunge ni recoil, el lanzador no
     se mueve. Nueva `FLY_DURATION = 0.40`.
   - Proyectil (circulo con brillo) que cruza del lanzador al objetivo; color por
     bando: `COLOR_FIREBALL` (naranja) para el heroe, `COLOR_SHADOW` (morado) para
     el enemigo.
   - `EventPlayer._make_animation`: `attack` -> `AttackAnimation`, `spell` ->
     `SpellAnimation(..., color segun bando)`; ambos conservan `on_impact`
     (el HP cae al impacto, HUD progresivo intacto).
4. **Verificacion y docs**: smoke test de regresion (solo cambia la UI, el motor
   no), README Features, y este documento. Pendiente: revision visual en ventana real.

**Verificacion**: smoke test ejecutado con SDL dummy (fuera del repo, en el dir temporal):
driver que publica eventos sinteticos (Play -> Attack -> Attack -> ...) y detiene el bucle
en cuanto `scores.json` gana un registro. Resultado: sin crash y una batalla completa
registrada (BATTLE -> ANIM -> END -> add_result -> QUIT). El `format_event` ya se verifico
identico a la Fase 1 en el paso 2 (script golden-code de 200 seeds). Queda la revision
visual de las animaciones abriendo la ventana real (`python -m game.main`).

## 5. Launcher (customTkinter)

- Panel de noticias (fetch + cache), versión instalada, botones Actualizar / Jugar.
- `updater.py`: API releases -> selecciona asset según `platform.system()` -> descarga zip -> extrae -> guarda versión.
- `paths.py`: dir de datos por SO (platformdirs); lanza el juego con `subprocess.Popen`.

## 6. CI + Releases (build.yml)

- Matrix `[windows-latest, macos-latest, ubuntu-latest]`, Python 3.12, PyInstaller `--onefile --windowed`.
- Trigger al push de tags `v*`: genera `rpg-battle-{tag}-{platform}.zip` y los sube al Release vía `softprops/action-gh-release`.

## 7. Convenciones de código

- Cabecera de atribución AI en cada `.py` (obligatorio CS50x):
  `# Developed with the assistance of an AI coding assistant (opencode).`
- Identificadores y docstrings en inglés; README en inglés.
- Sin librerías innecesarias; sin comentarios superfluos.

## 8. Roadmap por fases

1. **[COMPLETA]** Logica del juego en consola — `config.py`, `entities.py`, `battle.py`, `score.py` (testeable sin pygame).
   - `config.py`: constantes de balance (stats, habilidades, pocion, formula, huida, estados de resultado).
   - `entities.py`: `Character` (base) -> `Hero` / `Enemy` + factories `make_hero()` / `make_enemy()`.
   - `battle.py`: clase `Battle` (jugador primero siempre, formula de daño con
     criticos, `player_turn` como orquestador, fin de batalla).
   - `score.py`: API JSON (`load_scores`, `add_result`, `summary`).
   - `main.py`: loop de consola MENU -> BATTLE -> VICTORY/DEFEAT -> JUEGO NUEVO.
   - Las animaciones de ataque estilo Suikoden II se desarrollan en la Fase 2
     (basicas) y Fase 3 (sprites reales + efectos); ver 4.3.
   - Nota: el motor de batalla (entities/battle) es independiente de la UI; la Fase 2
     lo reutilizara tal cual y `main.py` pasara a ser el bucle pygame.
2. **[EN CURSO]** UI pygame — plan cerrado en `4.3`. Hecho: pygame-ce 2.5.7 instalado,
   `config.py` con constantes de UI, refactor de `battle.py` a eventos + `format_event`
   (textos verificados identicos), `ui.py` (CharacterSprite placeholder con `scale` +
   tarjeta fija del heroe + placa de nombre del enemigo + LogPanel + widget `Menu`),
   y `main.py` con la maquina de estados completa (MENU/STATS/BATTLE/ANIM/END, menu de
   combate de dos niveles estilo Suikoden II, disposicion con profundidad, pantallas END
    y STATS con `summary()`, y animaciones completas (`EventPlayer` + `AttackAnimation` +
    `SpellAnimation` + `TextAnimation`, log en vivo via callback). Ademas, 3 mejoras
    cerradas con el usuario: textos flotantes con contorno y fade tardio, HUD de vida/mana
    progresivo por snapshots (`Battle._append` + `display_hp/mp` + `on_impact`) y lunge de
    recorrido completo (`LUNGE_GAP`). Verificacion (smoke test SDL dummy) hecha. Pendiente:
    revision visual final de las mejoras con la ventana real.
3. Pulido — sprites reales y frames (solo se cambia el draw() de CharacterSprite via
   `assets.py`), efectos de hechizo y pocion (subclases de Animation), sonidos,
   pantallas win/lose pulidas.
4. Launcher — `paths.py`, `news.py`, `updater.py`, UI customTkinter.
5. Integracion + CI + Release — PyInstaller specs, `build.yml`, pruebas end-to-end.
6. CS50x — README final, video <=3 min (titulo, nombre, GH+edX usernames, ciudad/pais, fecha), `submit50 cs50/problems/2026/x/project`, gradebook.

## 9. Checklist CS50x

- [ ] Codigo en multiples archivos con clases y funciones
- [ ] Librerias externas (pygame, customTkinter)
- [ ] File I/O (news.json, score.json, version local)
- [ ] Atribucion de uso de AI en comentarios
- [ ] README.md + video <=3 min + submit50 antes de 2026-12-31
