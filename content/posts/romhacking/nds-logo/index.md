---
title: "Logo de Nintendo en GBA y NDS"
date: 2013-08-19T03:19:00+02:00
hero: images/pleonex_logo.png
description: >
  (Spanish) Análisis del logo en juegos de Nintendo DS.
theme: Toha
menu:
  sidebar:
    name: "Nintendo logo"
    identifier: nds-gba-logo
    parent: romhacking
    weight: 120
tags: ["rom-hacking", "nds", "gba"]
---

No hace mucho, [@Nitehack](http://twitter.com/nitehack) me preguntó si sabía
cómo podría extraer una imagen a partir de unos datos. No era la primera vez que
me enfrentaba a este reto en concreto y ahora, con más experiencia adquirida
quería volver a intentarlo.

## ¿Objetivo?

Todos los juegos para GameBoy Advance (GBA) y Nintendo DS tienen unos datos al
principio del archivo (_header_) que los usa la consola para la inicialización
del juego como el nombre, el código del desarrollador, las posiciones de memoria
donde va el código, el punto de inicio del sistema de archivos... Sorprende ver
entre estos datos un conjunto de bytes sin significado en ese contexto pero que,
mirando en GBATEK parece ser el "Logo de Nintendo". Teóricamente, a partir de
esos bytes se podría recrear esa imagen y es más, así lo hacen las consolas. En
en juego para GBA estos datos se encuentran a partir de la posición _0x04_
mientras que en uno de la DS en _0xC0_, en ambos dispositivos este logo debe
ocupar _0x9C_ bytes.

{{< img src="images/gba_logo.png" align="center" title="Logo de Nintendo en GBA" >}}
_Logo de Nintendo en GBA. En el archivo solo aparece el de Nintendo, no el de
GAME BOY._

Lo primero que intento es comprobar si puedo ver ya directamente el logo. Para
ello creo un archivo con los bytes `00 00 FF 7F` que al abrir como paleta serán
los colores negro y blanco respectivamente codificados en BGR555. A continuación
usando [Tinke](https://github.com/pleonex/tinke) abro los dos archivos, la
paleta seleccionándola y pulsando 'P' y luego los datos del logo, igual y
pulsando 'T'. El resultado final es este:

{{< img src="images/try0.png" align="center" title="Primer intento fallido" >}}
_Datos en bruto, nada visible_

La imagen está configurada como _1bpp y tiled (horizontal)_. Como se puede
apreciar tendremos que hacer operaciones con esos bytes para obtener la imagen
original y ese será el objetivo de la entrada, conseguir obtener la imagen y
analizar todo lo relacionado con ella. El porqué de hacer esto no tiene
respuesta, cuando a uno le gusta la ingeniería inversa y se le propone un reto
no se pregunta para qué quiere el resultado, solo quiere disfrutar investigando
;).

## Cómo obtener la imagen

Lo primero que hice fue buscar en el sitio donde sabía que algo habría
comentado, en [GBATEK](https://problemkaputt.de/gbatek.htm). Ahí está
prácticamente toda la documentación técnica pública que existe entorno a GBA y
NDS, todo investigado por Martin Korth en el desarrollo de su emulador no$gba,
un emulador que a pesar de los años que lleva sin ser actualizado funciona
perfectamente para juegos de la GBA y en bastantes de la NDS (a día de hoy, el
mejor es [DeSmuME](http://desmume.org/)). Concretamente, la información que
queremos está en la sección
[GBA Cartridge Header](https://problemkaputt.de/gbatek.htm#gbacartridgeheader)
donde encontramos lo que buscamos:

> **004h..09Fh - Nintendo Logo, 156 Bytes**  
> Contains the Nintendo logo which is displayed during the boot procedure.
> Cartridge won't work if this data is missing or modified.  
> In detail: This area contains **Huffman compression data (but excluding the
> compression header which is hardcoded in the BIOS**, so that it'd be probably
> not possible to hack the GBA by producing de-compression buffer overflows).  
> A copy of the compression data is stored in the BIOS, the GBA will compare
> this data and lock-up itself if the BIOS data isn't exactly the same as in the
> cartridge (or multiboot header). The only exception are the two entries below
> which are allowed to have variable settings in some bits.

Si vamos a la sección correspondiente de la DS vemos que nos dice que esos datos
son iguales a los de GBA, así que nos concentraremos en la GBA (más tarde
explicaré la diferencia entre las dos plataformas).

Bueno, ya al menos tenemos una pista que explica el porqué de tan pocos bytes
para una imagen, **están comprimidos con Huffman** (recomiendo leer este post de
CUE sobre esta codificación:
[Algoritmo de Huffman (no adaptativo)](http://romxhack.esforos.com/algoritmo-de-huffman-no-adaptativo-t30)
y usar sus
[herramientas](http://romxhack.esforos.com/compresiones-para-las-consolas-gba-ds-de-nintendo-t117)
para ello) y porqué no la hemos podido visualizar. Lo que se me ocurrió entonces
fue coger un descompresor de Huffman, añadirle yo a los datos del logo una
cabecera estándar como:

> `28 FF FF FF`
>
> - El 0x28 indica que los datos están comprimidos con Huffman 8bits, también
>   habría que probar con 0x24 (4bits).
> - Los bytes 0xFFFFFF indican el tamaño descomprimido, dado que este no lo
>   sabemos ponemos el más grande. El programa nos avisará cuando termine la
>   decodificación de que ese tamaño no corresponde al obtenido pero no es más
>   que una advertencia, si está bien programado no deberíamos tener problemas.

Convencido de que funcionaría, no fue así... Con ambas compresiones obtuve dos
archivos de 652 bytes que nada de sentido tenían. Bueno, como primer intento no
quedó mal... (Ahora que he estudiado a fondo esta compresión me doy cuenta de la
barbaridad que hice... Estaba claro que le faltaba el árbol (el segundo byte de
un árbol no puede ser _0xFF_))

No funcionó, así que lo siguiente que se me ocurrió fue ver como el juego leía
esos datos y operaba con ellos, es decir, a por **el código ensamblador**. Para
esta ocasión usaré _no$gba_, a pesar de que su interfaz es más sencilla, con
menos opciones y un poco más difícil de dislumbrar lo que buscamos, es el único
emulador para GBA con el que he trabajado y sé usarlo bien. Este programa es de
pago ya que contiene el depurador, creo que ya no es posible conseguirlo porque
según me dijeron el autor ya no contesta a los emails, está como desaparecido,
de todas formas es cosa de cada uno el conseguirlo.

{{< img src="images/nosgba.PNG" align="center" title="no$gba con el depurador" >}}
_no$gba con el depurador_

Como juego víctima escogí uno de Pokémon, porque ya lo tenía en el ordenador
pero dado que vamos a tratar con funciones de la BIOS da igual el que uséis. Lo
primero que hice fue poner un punto de interrupción (a partir de ahora PI) (se
añaden con Ctrl+B) en la posición **_0x08000004_**, ahí es donde comienzan los
datos del logo (la posición _0x08000000_ es donde se encuentra el juego), lo
hacemos escribiendo en la ventana de PI:

> `[08000004]?`
>
> Eso indica que cuando se lea (símbolo ?) en la posición de memoria encerrada
> entre corchetes, se pare el emulador.

Además, tenemos que indicarle al emulador que queremos que no se inicie el juego
directamente, si no que ejecute la BIOS primero, como en una GBA original (si no
nos saltamos lo que vamos buscando), eso se hace en: _Options -> Emulation Setup
-> Reset/Startup Entrypoint -> **GBA BIOS (Nintendo logo)**_. Preparado todo,
nos disponemos a comenzar nuestra sesión de depuración. La primera interrupción
se hace en **_0xB90_** con las instrucciones (están en THUMB):

```asm
0xB8A  cmp  r5,r4
0xB8C  bge  #0xB96
0xB8E  ldrh r3,[r0,r5]
0xB90  strh r3,[r1,r5]
0xB92  add  r5,r5,2
0xB94  b    #0xB8A
```

Ese fragmento es un bucle que copia una serie de datos (r4 bytes) de r0 a r1,
seguramente se estará ejecutando dentro de otro bucle mayor. Básicamente, ahí se
está copiando nuestro logo a r1 (para trabajar con los bytes, estos no pueden
estar en la zona donde está el cartucho, deben estar en _0x03yyyyyy_
_0x02yyyyyy_). En este caso los está copiando a **_0x03000088_** (el valor de
r1), luego ponemos otro PI ahí para saber que hace luego con ellos y continuamos
la ejecución (con F9).

Como veréis de nuevo llegamos a ese trozo de código, esta vez está copiando de
nuevo el logo de donde los había copiado antes a **_0x030000588_** luego ponemos
otro PI allí (esperando que este sea el último), continuamos y...

El emulador se para ahora en _0x1072_, delante nuestra tenemos la subrutina
(comienza en **_0x1014_**) que como veremos descomprime / **descodifica los
datos del logo** usando el algoritmo Huffman:

```plain
Rutina para descomprimir HUFFMAN
** Variables iniciales **
R0 = puntero de lectura, donde se encuentran los datos comprimidos
R1 = puntero de escritura, donde se van a escribir los datos descomprimidos
R13 = SP, es la pila donde se guardan variables temporales
** Variables de decodificación **
R0 = puntero de lectura
R1 = puntero de escritura
R2 = puntero de lectura del árbol
R3 = contiene los bytes decodificados (hasta 4 y escribe)
R4 = contiene el tipo de codificacin (4 u 8)
R5 = contiene el codeword actual
R6 = contiene el byte del nodo actual
R7 = puntero de lectura del rbol (no cambia)
R8 = bits procesados del codeword
R9 = bit actual del codeword
R10 = variable temporal
R11 = variable temporal
R12 = contiene los bytes que quedan por decodificar
R13 = pila
R14 = contador de escritura, se incrementa cada vez que se obtienen bits decodificado (un hijo)

----------------- Inicialización ------------------------------------
00001014 E92D4FF0 stmfd r13!,{r4-r11,r14} // Inicio de la subrutina, guarda algunos registros
00001018 E24DD008 sub r13,r13,#0x8 // Reserva 8 bytes para variables temporales, solo usa 4
0000101C E3B0C402 movs r12,#0x2000000 // Ver la siguiente instrucción
00001020 EBFFFEDF bl #0xBA4 // Más o menos allí hace lo mismo que MOV r12,r0
00001024 0A000030 beq #0x10EC (fin) // Si ha habido un fallo y R0 es 0 (puntero nulo) salir
00001028 E2802004 add r2,r0,#0x4 // R2 = R0 + 4 : para leer el número de nodos
0000102C E2827001 add r7,r2,#0x1 // R7 = R0 + 5 : inicio del rbol
00001030 E5D0A000 ldrb r10,[r0] // Lee la cabecera de compresión
00001034 E20A400F and r4,r10,#0xF // Obtiene el tipo de compresión 4 8.
00001038 E3A03000 mov r3,#0x0 // Inicializa algunos valores a 0
0000103C E3A0E000 mov r14,#0x0 // lo mismo
00001040 E204A007 and r10,r4,#0x7 // Obtiene el tipo de compresión de otra forma (4 or 0)
00001044 E28AB004 add r11,r10,#0x4 // Otra vez: 4->8, 0->4
00001048 E58DB004 str r11,[r13,#0x4] // Escribe ese valor en la pila
0000104C E590A000 ldr r10,[r0] // Lee otra vez la cabecera de compresión
00001050 E1A0C42A mov r12,r10,lsr #0x8 // Obtiene el tamaño del archivo descomprimido de ella
------------------Inicializa valores --------------------------------
00001054 E5D2A000 ldrb r10,[r2] // Lee el número de nodos
00001058 E28AA001 add r10,r10,#0x1 // Le suma uno
0000105C E082008A add r0,r2,r10,lsl #0x1 // y lo multiplica por 2 para obtener el puntero a los codewords
00001060 E1A02007 mov r2,r7 // Inicializa el puntero del árbol
------------------Inicio bucle (bucle1)------------------------------
00001064 E35C0000 cmp r12,#0x0 // Comprueba si ya tenemos todos los bytes decodificados
00001068 DA00001F ble #0x10EC (fin) // y en ese caso sale
0000106C E3A08020 mov r8,#0x20 // Inicializa los bits a procesar a 32 (4 bytes)
00001070 E4905004 ldr r5,[r0],#0x4 // Lee un codeword
------------------Inicio bucle (bucle2)------------------------------
00001074 E2588001 subs r8,r8,#0x1 // Actualiza los bits a procesar del codeword
00001078 BAFFFFF9 blt #0x1064 (bucle1) // Si son cero, sale de este bucle
0000107C E3A0A001 mov r10,#0x1 // Máscara para obtener el bit del codeword
00001080 E00A9FA5 and r9,r10,r5,lsr #0x1F // Obtiene el bit actual del codeword
00001084 E5D26000 ldrb r6,[r2] // Obtiene el byte del nodo para...
00001088 E1A06916 mov r6,r6,lsl r9 // activar el indicador de si es un hijo lo siguiente
0000108C E1A0A0A2 mov r10,r2,lsr #0x1 // Las siguientes operaciones son para actualizar
00001090 E1A0A08A mov r10,r10,lsl #0x1 // el puntero de lectura en el árbol
00001094 E5D2B000 ldrb r11,[r2] // a partir del valor que hay en el nodo que indica
00001098 E20BB03F and r11,r11,#0x3F // donde está el siguiente nodo / hijo
0000109C E28BB001 add r11,r11,#0x1 // haciendo el AND 0x3F
000010A0 E08AA08B add r10,r10,r11,lsl #0x1 // y tal
000010A4 E08A2009 add r2,r10,r9 // y aquí tenemos el valor final
000010A8 E3160080 tst r6,#0x80 // Comprueba el indicador de si es un hijo (byte decodificado)
000010AC 0A00000A beq #0x10DC (saltar escritura)// y salta
------------------Inicio condición (escritura)-----------------------
000010B0 E1A03433 mov r3,r3,lsr r4 // Actualiza el registro que contiene los bits decodificados
000010B4 E5D2A000 ldrb r10,[r2] // Obtiene el nuevo byte decodificado (hijo)
000010B8 E264B020 rsb r11,r4,#0x20 // Valor de desplazamiento
000010BC E1833B1A orr r3,r3,r10,lsl r11 // Añade el nuevo byte al registro
000010C0 E1A02007 mov r2,r7 // Resetea el puntero de lectura del árbol
000010C4 E28EE001 add r14,r14,#0x1 // Incrementa el registro de escrituras
000010C8 E59DB004 ldr r11,[r13,#0x4] // Obtiene el tipo de codificación para la comparación
000010CC E15E000B cmp r14,r11 // Comprueba si hemos decodificado ya 4 bytes
000010D0 04813004 streq r3,[r1],#0x4 // En ese caso escribimos esos 4 bytes
000010D4 024CC004 subeq r12,r12,#0x4 // Actualiza los bytes por decodificar
000010D8 03A0E000 moveq r14,#0x0 // Resetea el registro de escrituras
-----------------Fin condición---------------------------------------
000010DC E35C0000 cmp r12,#0x0 // Comprueba si ya tenemos todos los bytes decodificados
000010E0 C1A05085 movgt r5,r5,lsl #0x1 // Actualiza el codewords
000010E4 CAFFFFE2 bgt #0x1074 (bucle2) // Loop de bucle
------------------Fin bucle------------------------------------------
000010E8 EAFFFFDD b #0x1064 (blcue1) // Loop de bucle
------------------Fin bucle (fin)------------------------------------
000010EC E28DD008 add r13,r13,#0x8 // Borra la pila (variables temporales)
000010F0 E8BD4FF0 ldmfd r13!,{r4-r11,r14} // Recupera los registros
000010F4 E12FFF1E bx r14 // Sale
```

Lo mejor para saber si esa es la subrutina que buscamos es ir ejecutando las
instrucciones paso a paso (con F7). Obtenida la función hice un programilla en
C# que hiciese exactamente lo mismo (lo podéis descargar al final de la
entrada).

¡Ojo!, si reinicias el juego de forma que consigas ejecutar la subrutina desde
el principio comprobarás que esta no comienza a leer datos desde _0x03000588_,
si no desde **_0x03000564_**. ¿Qué sucede? Tal y como se comenta en GBATEK, lo
que viene en la cabecera de los juegos son **los datos comprimidos** de la
imagen, pero para descomprimirla, en Huffman **se necesitan datos extras** (los
que forman el árbol) por eso nuestro primer intento salió mal. Los datos en
cuestión que hay que añadir a los del logo para poder descomprimir son los
siguientes (se encuentra en la BIOS):

```plain
24 D4 00 00 0F 40 00 00 00 01 81 82 82 83 0F 83
0C C3 03 83 01 83 04 C3 08 0E 02 C2 0D C2 07 0B
06 0A 05 09
```

A pesar de todo esto si intentamos ver si tenemos la imagen...el resultado sigue
siendo malo.

{{< img src="images/try2.png" align="center" title="Segundo intento fallido" >}}
_Segundo intento._

Mmm...Los datos descomprimidos están bien ya que lo comprobé con los que hay en
la RAM al terminar la subrutina... Habrá que seguir investigando (¡¡bieeen!!).
Ponemos un primer PI al final de la subrutina, en _0x10EC_ (para poner PI en el
código haced doble-click sobre la línea que queréis), continuamos y cuando pare
ponemos otro PI en el comienzo de los datos descomprimidos: _0x03001568_. Deber
parar en **_0x13A6_** donde tenemos la siguiente subrutina:

```plain
** Variables iniciales **
r0 -> puntero de lectura
r1 -> puntero de escritura
r14 -> retorno

00001398 B510 push {r4,r14}
0000139A C810 ldmia r0!,{r4} // Lee el primer uint (4bytes)
0000139C 0A24 lsr r4,r4,#0x8 // Elimina el primer byte, el resultado indica el tamaño total (0xD0)
0000139E F7FFFBFD bl #0xB9C // r12 = r0 + r4, puntero al final de los datos
000013A2 D00B beq #0x13BC // Si es 0, salir
000013A4 8802 ldrh r2,[r0] // Lee los dos bytes siguientes (ushort), es el offset relativo a los datos
000013A6 1C80 add r0,r0,2 // Se los suma a r0
000013A8 800A strh r2,[r1] // Lo guarda en r1
000013AA 1C89 add r1,r1,2 // Y aumenta el puntero de escritura
----------------- Bucle 1 -----------------
000013AC 1EA4 sub r4,r4,2 // Actualiza los bytes que quedan por leer
000013AE DD05 ble #0x13BC // Si es 0, sale
000013B0 8803 ldrh r3,[r0] // Lee dos bytes
000013B2 189A add r2,r3,r2 // Se los suma a la cuenta anterior
000013B4 1C80 add r0,r0,2 // Actualiza el puntero de lectura
000013B6 800A strh r2,[r1] // Escribe los datos (dos bytes)
000013B8 1C89 add r1,r1,2 // Actualiza el puntero de escritura
000013BA E7F7 b #0x13AC // Salto del bucle
----------------- Fin ---------------------
000013BC BC10 pop {r4} // Recupera una variable
000013BE BC04 pop {r2} // Recupera la dirección de retorno
000013C0 4710 bx r2 // Sale
```

Tras analizar esa subrutina vemos que todavía no termina la cosa de
desencriptar, ahora resulta que **para obtener los bytes originales de la imagen
tenemos que sumar un byte detrás de otro**. Bueno, actualizo de nuevo el
programilla para que haga eso también... Y vemos si lo hemos conseguido:

{{< img src="images/try3.png" align="center" title="Tercer intento fallido" >}}
_Tercer intento, casiiiii_

Esta imagen está con las siguientes propiedades: _"1bpp, horizontal tiled de 8x8
pixels"_. Como vemos, estamos a un paso de conseguirlo, ¿qué sucede ahora? Esto
ya no tiene nada que ver con que está encriptada o no y aquí es donde nos ayuda
la experiencia y el sentido común. Si nos damos cuenta, vemos que la imagen está
**reflejada** sobre el eje X, pero no del todo. Fijaos sobre todo en el símbolo
®, adems en la 'N', en su primera mitad vemos claramente como está invertida en
el eje X. Recordad que esta imagen es _"tiled"_, es decir, la formamos mediante
bloques (_tiles_) de 8x8 pixels que unimos uno detrás de otro
([aquí](http://code.google.com/p/tinke/wiki/NCGR#Datos_de_la_imagen) pongo un
ejemplo de la diferencia en lo que llamo "Lineal" y "Horizontal (tiled)"). La
solución por tanto es que **al dibujar el _tile_ vayamos de derecha a
izquierda**, es decir empezamos con un valor 7 y vamos disminuyendo hasta 0, la
altura no la tocamos y es descendente.

{{< img src="images/try4.png" align="center" title="Logo con la palabra Nintendo" >}}
_Cuarto y último intento_

**ACTUALIZACIÓN:** Tras hacer una limpieza de código de la herramienta que tenía
hecha, vi que esto último no era cierto, y ese último paso de invertir los tiles
no es siempre necesario. Dependiendo de la forma de leer los bits de un byte es
o no necesario hacer la inversión. Para no realizarla habrá que leerlos de menos
significativo a más significativo.

## Cómo modificar el logo

Antes de pensar en cómo codificar de nuevo la imagen debemos saber un par de
cosas de porqué aparecen esos datos en la cabecera de todo juego. Para ello
continuaremos con la sesión de depuración en no$gba. Eliminamos todos los puntos
de interrupción y dejamos solo el _0x03000088_. Veremos que para muchas veces,
una por cada frame de la animación del logo, tenemos que tener paciencia y
esperar a que pase esa animación dándole a continuar, nos interesa ver lo que
pasa justo después... Pues en ese momento se para en _0x0702_. Analicemos esa
subrutina que merece la pena, os la dejo aquí ya comentada (comienza en _0x6E8_)

```plain
** Comprueba que los bytes del logo sean los originales **
-> Variables de inicio
R0 = Puntero a los datos copiados de la cabecera del juego

000006E8 B570     push    {r4-r6,r14}        // Guarda algunos registros
000006EA 49F6     ldr     r1,=#0x3290        // Puntero de lectura a los bytes originales en la BIOS
000006EC 2600     mov     r6,#0x0            // Inicializa a 0 el contador de bytes
----------------- Inicio del bucle ----------------------------------
----------------- Inicializa el valor de la máscara -----------------
000006EE 24FF     mov     r4,#0xFF            // Valor de la máscara para los datos del juego
000006F0 2E98     cmp     r6,#0x98            // Comprueba si estamos comparando los últimos 4 bytes
000006F2 D100     bne     #0x6F6            // Si no es así se salta la siguiente instrucción
000006F4 247B     mov     r4,#0x7B            // Valor de la máscara para los datos del juego (¡no comprueba el byte entero!)
000006F6 2E9A     cmp     r6,#0x9A            // Comprueba si estamos comparando los últimos 2 bytes
000006F8 D100     bne     #0x6FC            // Si no es así se salta la siguiente instrucción
000006FA 24FC     mov     r4,#0xFC            // Valor de la máscara para los datos del juego (¡no comprueba el byte entero!)
000006FC 2E9C     cmp     r6,#0x9C            // Comprueba si hemos llegado al final
000006FE DA06     bge     #0x70E            // En tal caso, saltar a la parte final
----------------- Comprobación --------------------------------------
00000700 5D82     ldrb    r2,[r0,r6]        // Lee un byte de los del juego
00000702 5D8B     ldrb    r3,[r1,r6]        // Lee un byte de los originales
00000704 4022     and     r2,r4                // Realiza un AND con el byte del juego
00000706 1C76     add     r6,r6,1            // Incrementa el contador
00000708 429A     cmp     r2,r3                // Comparan que sean iguales
0000070A D0F0     beq     #0x6EE            // Si son iguales
----------------- Fin del bucle -------------------------------------
0000070C E009     b       #0x722            // Salto a error
----------------- Inicio de la comprobación del nombre del juego ----
0000070E 2419     mov     r4,#0x19            // Valor inicial de la sumatoria
----------------- Inicio del bucle ----------------------------------
00000710 5D82     ldrb    r2,[r0,r6]        // A continuación lo que hay es el código del juego, lee un byte
00000712 18A4     add     r4,r4,r2            // Suma este byte
00000714 1C76     add     r6,r6,1            // Incrementa el contador
00000716 2EBA     cmp     r6,#0xBA            // El bucle se ejecuta hasta que vale 0xBA
00000718 DBFA     blt     #0x710            // Salto del bucle
----------------- Fin del bucle -------------------------------------
0000071A 0620     lsl     r0,r4,#0x18        // La suma total no puede superar 0xFFFF, los dos bytes...
0000071C D101     bne     #0x722            // Salto a error
0000071E 2000     mov     r0,#0x0            // Devuelve correcto
00000720 E000     b       #0x724            // Salto a la salida
00000722 2001     mov     r0,#0x1            // Devuelve error
00000724 BD70     pop     {r4-r6,r15}        // Recupera algunos registros
```

Resumamos las sorpresas:

1. Si queremos que la consola no se quede bloqueada (al devolver error entra en
   un bucle infinito) **no podemos cambiar el logo pero...** eso no es del todo
   cierto. Lo normal es que r4 valga siempre 0xFF para que se analicen todos los
   bits de todos los bytes, pero hay dos bytes del final, con los que deja que
   haya unos cuantos bits diferentes al original. De nuevo buscando en GBATEK
   damos con lo siguiente:

   > **09Ch Bit 2,7 - Debugging Enable**
   >
   > This is part of the above Nintendo Logo area, and must be commonly set to
   > 21h, however, Bit 2 and Bit 7 may be set to other values.
   >
   > When both bits are set (ie. A5h), the FIQ/Undefined Instruction handler in
   > the BIOS becomes unlocked, the handler then forwards these exceptions to
   > the user handler in cartridge ROM (entry point defined in 80000B4h, see
   > below). Other bit combinations currently do not seem to have special
   > functions.
   >
   > **09Eh Bit 0,1 - Cartridge Key Number MSBs**
   >
   > This is part of the above Nintendo Logo area, and must be commonly set to
   > F8h, however, Bit 0-1 may be set to other values.
   >
   > During startup, the BIOS performs some dummy-reads from a stream of
   > pre-defined addresses, even though these reads seem to be meaningless, they
   > might be intended to unlock a read-protection inside of commercial
   > cartridge. There are 16 pre-defined address streams - selected by a 4bit
   > key number - of which the upper two bits are gained from 800009Eh Bit 0-1,
   > and the lower two bits from a checksum across header bytes 09Dh..0B7h
   > (bytewise XORed, divided by 40h).

   Resumiendo, son bits que se usan para activar funciones especiales de
   depuración, de hecho si probáis a cambiarlos el logo final será siendo el
   mismo. Así que no hay forma de modificar un logo.

2. Otra curiosidad es que hace una segunda comprobación que no llego a entender:
   suma todos los bytes del nombre en clave del juego y si este resultado es
   mayor que _0xFFFF_ (65535, tamaño máximo de dos bytes), devuelve error. No le
   veo mucho sentido porque ese valor no se debería ni leer, para un usuario
   final ese nombre en clave no aparece nunca por ningún lado.

Conociendo esto aceptamos que no podemos modificar el logo si queremos que
funcione porque en el emulador si hacemos que se lo salte no vemos el logo
cambiado y si lo muestra no se ejecuta el juego (lo mismo en una consola)...
Pero digamos que queremos sorprender a nuestros amigos y dejar la consola
bloqueada con un logo propio, pues continuemos. Como sabréis en una codificacin
Huffman hay una tabla con cada caracter ordenada por su número de apariciones
con el objetivo de que los que más se repitan ocupen menos bits, veámoslo aquí
con el árbol de este caso.

**Árbol Huffman:**

```plain
                                   _____1E_____
                                  /            \
                           ______1D______       0
                          /              \
                      __1C__           __1B__
                     /      \         /      \
                    1A      19       18      17
                   /  \    /  \     /  \    /  \
                  F   16  C   15   3   14  1   13
                     /  \    /  \     /  \    /  \
                    4   12  8    E   2   11  D   10
                       /  \             /  \    /  \
                      7    B           6    A  5    9
```

De ahí podemos sacar los **codewords** (los números son bits):

```plain
    0: 1            1: 0110
    2: 01010        3: 0100
    4: 00010        5: 011110
    6: 010110       7: 000110
    8: 00110        9: 011111
    A: 010111       B: 000111
    C: 0010         D: 01110
    E: 00111        F: 0000
```

Codificar un archivo usando una tabla dada no es difícil, solo tenemos que
sustituir cada 4bits por el codeword correspondiente, es decir, que donde haya
un _5_ (_0101_ en binario) miramos en los codewords y lo sustituimos por
_011110_ (es binario) y así. **El problema es que el archivo final sea mayor que
el original**, entonces no podremos usar ese logo ya que solo se decodificaran
los _0xD4_ primeros bytes (los que indica la cabecera que está en la BIOS). Si
es más pequeño no hay problema, lo rellenamos con _0x00_. Dicho esto, me dispuse
a actualizar de nuevo mi programilla para que codificara usando la tabla
original.

De lo primero que me di cuenta es que los archivos no coincidían y no era
problema de mi codificador. Los dos últimos bytes que obtenía eran 00 (los bytes
en las posiciones 0x98 y 0x99, se escriben a la vez 4 bytes en low-endian) ya
que no había más datos para codificar pero en el original estos valores están
puestos a 0x21 y 0xD4. Lo que sucede es que la decodificación para cuando se
obtienen todos los bytes que indica la cabecera (0xD4), si nos fijamos en el
estado de cuando esto sucede vemos que **no todos los codewords se han usado**,
de hecho hay 0x12 (18) bits que se quedan sin ser procesados pero que no tiene
sentido procesarlos (hay como un codeword que no existe o que le faltan bits y
no son bits de relleno puesto que hay un 1). Sin embargo eso no es un error. A
partir del bit 18 de ese último uint (4 bytes juntos sin signo) comienzan los
bits especiales de depuración que no pertenecen al logo (en GBATEK solo viene el
significado de 4 de ellos, los que hemos dicho antes que son de depuración y no
se comprueban). Por ello, cuando los modificábamos el logo no cambiaba, por que
no se llegan a decodificar. Como esos bits no pertenecen a la codificación no
queda más remedio que ponerlos a mano. Solucionado dicho problema de
codificación solo me quedaba añadir la encriptación de suma siguiendo la
siguiente fórmula:

```plain
    encriptado_i = desencriptado_i - desencriptado_(i-1); // i=1,2,3...
```

esas tres variables son valores de 16bits, las variables que pertenecían al
desencriptado se van leyendo del logo, la imagen final, y el valor inicial de
`desencriptado_(i-1)` (para i = 1) es **0**.

{{< img src="images/imported.PNG" align="center" title="no$gba mostrando el logo inicial cambiado con la palabra pleonex en lugar de Nintendo" >}}

## Diferencias entre GBA y NDS

Hasta ahora todo lo comentado es referido a la GBA sin embargo, en la cabecera
de los juegos de la Nintendo DS encontramos esos mismos datos, concretamente
empiezan en _0xC0_. La forma de obtener la imagen es igual pero cambia bastante
la forma de cómo comprueba que sean los datos originales:

> For the Logo checksum, the BIOS verifies only [15Ch]=CF56h, it does NOT verify
> the actual data at [0C0h-15Bh] (nor it's checksum), however, the data is
> verified by the firmware.

Es decir, justo después en 0x15C hay un valor de 16bits que es el código CRC16
de los datos del logo. Ese código es el que comprueba la BIOS (no sé exactamente
si lo calcula de nuevo o lo comprueba con uno fijo que tiene). Pero además, no
contento con eso, el firmware comprueba que los datos sean los mismos. Para
resumir, **no hay forma de poder cambiarlo en la DS** y que se pueda jugar. De
nuevo, si lo cambias en los emuladores, tras una advertencia (en no$gba) se
puede jugar ya que no hacen esa comprobación. Sin embargo lo he probado en mi DS
con Wood (para M3) y no deja ejecutarlo, algo extraño porque si comprobáis el
logo de un **homebrew, por razones que comentaré abajo, es totalmente distinto**
al del Nintendo (y este no parece ser un logo porque lo intento extraer y me da
error por todos lados). Tras un segundo intento donde a ese mismo juego le he
puesto el mismo logo que viene en el homebrew ya he podido jugar, así que por lo
visto Wood admite dos logos: uno para juegos comerciales y otro para homebrews.

**ACTUALIZO:** He probado en otras flascard y por ejemplo con M3 Sakura se puede
ejecutar un juego con el logo cambiado sin problemas. Sin embargo con firmware
de _R4i Chrismas version_, el juego se queda en blanco pillado.

~~Segunda diferencia y más importante: **en la DS el logo no se muestra en
ningún momento** (si me equivoco que alguien me corrija pero yo nunca lo he
visto). Este punto nos introduce el siguiente, entonces ¿para qué querer
cambiarlo y que en los homebrew sea diferente?~~

Gracias al apunte de Nitehack en su
[comentario](http://pleonet.blogspot.com/2013/08/logo-de-nintendo-en-gba-y-nds.html?showComment=1376928684623#c2023428035078474522).
En la DS sí que aparece, justo nada más encenderla antes de que se cargue el
menú con opciones **siempre y cuando haya un juego introducido**. Ahí está la
clave de algo que seguramente habremos visto todo pero que nunca no hemos parado
a pensar porqué. Si no hay juego introducido, no aparece logo de Nintendo, ¡no
lo puede leer! Esto implica que las flashcard _deben de tener_ el logo de
Nintendo porque de otra manera veríamos otra cosa, al menos en la DSi, entre
otras porque no se puede arrancar el juego directamente, pero en las consolas
anteriores... se puede iniciar directamente sin pasar por el menú (incluso con
_M3 DS Real_ se salta hasta esa advertencia), y esa puede ser la clave para que
algunas flascard no lo hayan necesitado añadir. Finalmente os dejo una captura
de DeSmuME con este logo en la DS.

{{< img src="images/pleonex_logo.png" align="center" title="Logo en Nintendo DS con la palabra pleonex en lugar de Nintendo" >}}

## El motivo del logo

Esta pequeña historia tiene ya su tiempo. Las primeras consolas portátiles de
Nintendo (GB, GBC...) también traen un logo de Nintendo en la cabecera de sus
juegos, ligeramente diferente y que analizaré en otra entrada (esto no tiene
fin). El motivo es muy simple: **evitar que se distribuyan juegos sin permiso de
Nintendo**. La verdad es que está muy bien y no tan bien pensado, el logo de
Nintendo tiene copyright, pertenece a Nintendo y cualquiera no puede
distribuirlo cuando quiera ni como quiera, por eso los homebrew y toda
modificación de un juego original licenciado por Nintendo, no puede contenerlo
porque **estaría distribuyendo material con copyright sin permiso** de Nintendo.

La consecuencias son varias, como hemos comentado antes, cualquier homebrew o
parche no puede contenerlo, a parte ningún programa que sirva para modificar
juegos puede contener esos bytes en su código. Ejemplo: _fugbar_, programa para
arreglar cabeceras de juegos de la GBA

> fuGBAr does NOT include this logo to correct rom headers, as it is illegal to
> store the logo within fuGBAr. It is copyrighted, and belongs to Nintendo. **It
> can be argued that it's ok to include this copyrighted logo in your GBA works,
> since its required for the rom to actually work**, but this tool is not a gba
> rom and we'd rather not deal with any legal harrassment.

Otro, Turrican 3.5 un homebrew:

> He is aware of the issue on real hardware. No header was put because of
> copyright reason (Nintendo logo in the header). So, officially :D, T2002 for
> GBA is for emulators only.

¿Y esta acción por parte de Nintendo? Respuesta: ¿y cómo es que las tarjetas
para backups funcionan en una GBA/NDS?, si la consola las confunde con un juego
es porque tienen ese logo. Yo no lo sé, no sé si el chip que lleva la tarjeta
hace que se salte ese y otros bloqueos _antipiratería_, pero en caso de que lo
tuviera ¿no sería una buena estrategia para retirarlas del mercado?

Finalmente, ampliando la frase en negrita de la cita de _fugbar_ os recomiendo
leer
[esta historia](http://compgroups.net/comp.lang.forth/dtc-address-problem/1257397).
Y aquí os dejo un resumen:

> El usuario Jeff Massung creador, según deja a entender, de un compilador de
> BASIC para la GBA le llegó un email de Nintendo pidiéndole que retirara de su
> compilador los bytes del logo porque era material protegido por copyright que
> ellos no querían que estuviesen ahí. Él preguntaba en el post que cómo podía
> hacer para ejecutar los juegos sin ese logo. El usuario Albert van der Horst
> le responde que no debe asustarse, lo que Jeff había hecho era legal porque no
> usaba el logo para obtener beneficios a partir de él y porque está obligado a
> poner esa cabecera para asegurar la interoperatividad y le recomienda luchar.

Y así es, en Europa el
[artículo 6 de 1991](http://es.wikipedia.org/wiki/Decompilador#Aspectos_legales)
permite la decompilación con motivo de la interoperatividad algo que implica en
algunos casos añadir datos como los datos del logo. Casos frecuentes es el de
[WhatsAPI](http://www.securitybydefault.com/2012/09/whatsapp-coacciona-los-creadores-de.html),
que al final no fue a ningún lado, pero el más conocido fue el de
[Sega y Accolade](http://en.wikipedia.org/wiki/Sega_v._Accolade), un caso muy
muy parecido al anterior:

> Accolade, una empresa, puso en venta un videojuego para el cual tuvo que hacer
> ingeniería inversa y determinar que para que este funcionase tenía que
> contener el archivo del juego en una parte la palabra "SEGA" y si esa palabra
> no estaba el juego no se ejecutaba, exactamente como el logo de Nintendo, SEGA
> lo denunció por eso. Finalmente, el juez determinó que no había cometido
> ninguna ilegalidad, no había copiado código de SEGA y solo había puesto esa
> palabra para poder ejecutar su propio código.

Y así es como debe ser, para evitar los monopolios en el mercado pero claro,
quién ante una imponente carte de "cese y desista" de una gran compañía al menos
no se plantea si merece la pena seguir (con tu proyecto y puede que ir a juicio)
o parar.

Y hasta aquí la primera entrada de verdad, ha salido muuuy larga pero había
mucho que explicar en este gran misterio que me perseguía desde que me inicié en
este mundillo y que hasta ahora no he sido capaz de resolver. Espero que lo
hayáis disfrutado tanto como yo ;).

Código de programa para extraer e importar logos de NDS y GBA:
[LogoGBA.cs](https://gist.github.com/pleonex/6265017)

{{< gist pleonex 6265017 >}}
