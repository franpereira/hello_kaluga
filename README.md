| Supported Targets | ESP32-S2 |
| ----------------- | -------- |

# Hello Kaluga — App 1 (Clase 1)

Firmware base de la **App 1** de la Clase 1. Hace **parpadear/ciclar el LED
RGB direccionable** (WS2812) de la placa **ESP32-S2-Kaluga-1** y **publica el
estado por el monitor serie**. Es el primer contacto con un periférico real
de la placa.

Este proyecto implementa el ejemplo
[clase-1/ejemplos/hello_kaluga](../../clase-1/ejemplos/hello_kaluga) y toma
como referencia la configuración del proyecto que ya funciona
[projects/hello_world](../hello_world) (VS Code, ESP-IDF, clangd,
devcontainer).

## Hardware

- **LED RGB direccionable** en **GPIO45** de la Kaluga-1.
- ⚠️ **Coloca el jumper RGB** en la placa: sin él, el LED no recibe señal.
- Target ESP-IDF: **`esp32s2`**.

## Estructura

```
hello_kaluga/
├── CMakeLists.txt
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml      Declara la dependencia espressif/led_strip
│   └── hello_kaluga_main.c
└── README.md   (este archivo)
```

El componente `espressif/led_strip` se descarga **automáticamente** al
compilar por primera vez (queda en la carpeta `managed_components/`).

## Cómo compilar, flashear y monitorear

Con la extensión **ESP-IDF para VS Code** usa los botones de la barra
inferior, o desde la terminal de ESP-IDF:

```bash
idf.py set-target esp32s2
idf.py build
idf.py -p /dev/cu.usbserial-XXXX flash monitor
```

- Sustituye `/dev/cu.usbserial-XXXX` por tu puerto (ver `ls /dev/cu.*` en macOS).
- Conecta el cable al puerto **UART** de la placa.
- Salir del monitor: `Ctrl + ]`.

## Qué hace

1. Configura el LED RGB (GPIO45) mediante el componente `led_strip` (RMT).
2. Recorre en bucle una secuencia de colores (rojo, verde, azul, amarillo,
   apagado). Cada color dura un **ciclo** de 2000 ms, dividido en 20
   actualizaciones de 100 ms para poder animar el color y el brillo dentro
   del ciclo.
3. Con los dos efectos activados, en cada actualización el color se interpola
   hacia el siguiente de la secuencia mientras el brillo sube y vuelve a
   bajar: el LED da un "latido" por ciclo y el color transiciona durante ese latido.
4. Imprime por el monitor serie una línea por ciclo con el número de
   iteración y el color base (el color de partida del ciclo).

## Salida esperada

```
I (xxx) hello_kaluga: Hola desde la ESP32-S2-Kaluga-1
I (xxx) hello_kaluga: Controlando el LED RGB en GPIO45 (jumper RGB requerido)
I (xxx) hello_kaluga: [0] LED -> ROJO     (R= 32 G=  0 B=  0)
I (xxx) hello_kaluga: [1] LED -> VERDE    (R=  0 G= 32 B=  0)
I (xxx) hello_kaluga: [2] LED -> AZUL     (R=  0 G=  0 B= 32)
...
```

...y el LED de la placa cambiando de color en sincronía.

## Extensiones implementadas

Sobre el ejemplo base agregué dos efectos, que se activan mediante las siguientes flags:

```c
#define ENABLE_MIXING    1   /* interpolación entre colores consecutivos */
#define ENABLE_BREATHING 1   /* efecto de respiración (brillo suave)     */
```

### 1. Temporización por sub-pasos

El original cambiaba de color de golpe cada 500 ms. Para animar dentro de cada
color, el ciclo se subdivide:

```c
#define COLOR_CYCLE_MS    2000                            /* duración de un color  */
#define UPDATES_PER_CYCLE 20                              /* pasos por ciclo       */
#define UPDATE_DELAY_MS   (COLOR_CYCLE_MS / UPDATES_PER_CYCLE)  /* 100 ms por paso */
```

`COLOR_CYCLE_MS` controla la duración de cada color, y
`UPDATES_PER_CYCLE` controla que tan suave es la animación: a más pasos, más suave, a costa de mas uso de CPU.

### 2. Efecto de respiración (`ENABLE_BREATHING`)

`escalar_intensidad()` multiplica los tres componentes por un factor entre
0.0 y 1.0. Dentro del ciclo, `t` sube de 0 -> 1 en la primera mitad y baja de
1 -> 0 en la segunda, así que el brillo del color sube y se apaga de forma gradual en
lugar de encender de golpe.

### 3. Mezcla de colores (`ENABLE_MIXING`)

`mezclar_colores()` hace una interpolación lineal entre el color actual y el
siguiente de la secuencia a lo largo del ciclo. El resultado es una transición
continua (rojo -> naranja -> verde -> ...).

### Resultado observado

Con los dos efectos activados, el LED da un latido por ciclo: arranca apagado, sube
hasta el brillo máximo a mitad de ciclo y vuelve a apagarse, mientras se desplaza desde un color hasta el siguiente de la secuencia.

Con la mezcla activada, el ciclo `APAGADO` deja de ser una pausa completamente oscura, ya que interpola hasta el siguiente color (`ROJO`).

En el monitor serie aparece una línea cada 2 s con el contador y el color base con el cual arranca el ciclo.

## Solución de problemas

| Síntoma | Solución |
|---|---|
| El LED no enciende | Colocar el **jumper RGB**; verificar GPIO45 |
| No compila (`led_strip.h`) | Comprobar conexión a Internet en el primer *build* |
| `Failed to connect` al flashear | Secuencia **BOOT + RST** (modo bootloader) |
| Monitor con caracteres raros | Verificar **115200** baudios y puerto correcto |

## Referencias

- Guía de conexión e instalación:
  [../../bibliografia/Guia-Kaluga-1-macOS-ESP-IDF-VSCode.md](../../bibliografia/Guia-Kaluga-1-macOS-ESP-IDF-VSCode.md)
- Espressif, *ESP32-S2-Kaluga-1 User Guide* (LED RGB en GPIO45, jumper RGB).
- Componente `espressif/led_strip` en el ESP Component Registry.
