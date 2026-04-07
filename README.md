# Unidad 3: Manejo del DEBUG — Post-Contenido 2

---

## Descripción del Laboratorio

En este laboratorio se ensamblaron programas directamente en memoria usando el comando `A` del DEBUG dentro de DOSBox. Se ejecutaron instrucciones paso a paso con el comando `T`, registrando el estado de los registros y banderas después de cada instrucción en tablas de traza. Además, se analizó el comportamiento de un programa con bucle utilizando el registro `CX` y la instrucción `LOOP`.

### Comandos principales utilizados

| Comando | Descripción |
|---|---|
| `A 100` | Ensamblar instrucciones a partir de la dirección 0x0100 |
| `U 100 10E` | Desensamblar (verificar instrucciones ensambladas) |
| `R IP` | Ver y modificar el registro IP|
| `T` | Ejecutar una instrucción paso a paso|
| `D CS:100 L0C` | Volcar bytes de código máquina en memoria |

---

## Parte A — Programa de Suma con Traza Completa

```
-A 100
0482:0100 MOV AX, 000A   ; AX = 10 decimal
0482:0103 MOV BX, 0005   ; BX = 5
0482:0106 MOV CX, 0003   ; CX = 3
0482:0109 ADD AX, BX     ; AX = AX + BX = 15
0482:010B ADD AX, CX     ; AX = AX + CX = 18
0482:010D INT 20        
```

### Descripción
El programa carga tres valores en los registros `AX`, `BX` y `CX`, luego realiza dos sumas acumulando el resultado en `AX`. Al finalizar, `AX` debe contener `0x0012`.

---

## Checkpoint 1 - Tabla de Traza

| # | Instrucción ejecutada | AX | BX | CX | IP siguiente | ZF | CF | SF |
|---|---|---|---|---|---|---|---|---|
| 1 | `MOV AX, 000A` | 000A | 0000 | 0000 | 0103 | 0 (NZ) | 0 (NC) | 0 (PL) |
| 2 | `MOV BX, 0005` | 000A | 0005 | 0000 | 0106 | 0 (NZ) | 0 (NC) | 0 (PL) |
| 3 | `MOV CX, 0003` | 000A | 0005 | 0003 | 0109 | 0 (NZ) | 0 (NC) | 0 (PL) |
| 4 | `ADD AX, BX`   | 000F | 0005 | 0003 | 010B | 0 (NZ) | 0 (NC) | 0 (PL) |
| 5 | `ADD AX, CX`   | 0012 | 0005 | 0003 | 010D | 0 (NZ) | 0 (NC) | 0 (PL) |

---

## Parte B — Programa con Bucle usando LOOP y CX

### Programa ensamblado

```
-A 100
0482:0100 MOV AX, 0000   ; Acumulador = 0
0482:0103 MOV CX, 0004   ; Contador de iteraciones = 4
0482:0106 ADD AX, 0002   ; Suma 2 al acumulador ← inicio del bucle
0482:010A LOOP 0106       ; Decrementa CX, salta a 0106 si CX ≠ 0
0482:010C INT 20          ; Terminar (AX debe ser 0x0008 = 8)
```

### Descripción
El programa usa la instrucción `LOOP` para repetir la suma de `0x0002` exactamente 4 veces. `CX` actúa como contador: `LOOP` lo decrementa en cada iteración y salta de vuelta al inicio del bucle mientras `CX ≠ 0`. Cuando `CX` llega a `0`, la ejecución continúa hacia `INT 20`.

---

## Checkpoint 2: Tabla de Traza — Programa con Bucle LOOP

| # | Iteración | Instrucción ejecutada | AX después | CX después | IP siguiente | ¿LOOP salta? |
|---|---|---|---|---|---|---|
| 1  | Inicio   | `MOV AX, 0000` | 0000 | 0003 | 0103 | — |
| 2  | Inicio   | `MOV CX, 0004` | 0000 | 0004 | 0106 | — |
| 3  | Iter. 1  | `ADD AX, 0002` | 0002 | 0004 | 0109 | — |
| 4  | Iter. 1  | `LOOP 0106`    | 0002 | 0003 | 0106 | Sí |
| 5  | Iter. 2  | `ADD AX, 0002` | 0004 | 0003 | 0109 | — |
| 6  | Iter. 2  | `LOOP 0106`    | 0004 | 0002 | 0106 | Sí |
| 7  | Iter. 3  | `ADD AX, 0002` | 0006 | 0002 | 0109 | — |
| 8  | Iter. 3  | `LOOP 0106`    | 0006 | 0001 | 0106 | Sí |
| 9  | Iter. 4  | `ADD AX, 0002` | 0008 | 0001 | 0109 | — |
| 10 | Iter. 4  | `LOOP 0106`    | 0008 | 0000 | 010B | No salta |
| 11 | Fin      | `INT 20`       | 0008 | 0000 | —    | Programa termina |

---

## Parte C — Análisis del Código Máquina con D

### Paso 8: Volcado de memoria

```
-D CS:100 L0C
0482:0100  B8 00 00 B9 04 00 05 02-00 E2 FB CD 20
```
### Tabla de análisis del código máquina

| Bytes en memoria | Instrucción | Tamaño | Explicación de la codificación |
|---|---|---|---|
| `B8 00 00` | `MOV AX, 0000` | 3 bytes | `B8` es el opcode de MOV AX, imm16. Los 2 bytes siguientes `00 00` son el valor inmediato en formato little-endian. |
| `B9 04 00` | `MOV CX, 0004` | 3 bytes | `B9` es el opcode de MOV CX, imm16. Los bytes `04 00` representan el valor 0x0004 en little-endian. |
| `05 02 00` | `ADD AX, 0002` | 3 bytes | `05` es el opcode de ADD AX, imm16. Los bytes `02 00` representan el valor 0x0002 en little-endian. |
| `E2 FB`     | `LOOP 0106`    | 2 bytes | `E2` es el opcode de LOOP. `FB` es el desplazamiento relativo firmado: FB = -5 en decimal, calculado como 0x0106 − 0x010B = −5. Los saltos cortos del 8086 usan 8 bits con signo (rango ±127 bytes). |
| `CD 20`     | `INT 20`       | 2 bytes | `CD` es el opcode de INT (interrupción de software). `20` (32 decimal) es el número de interrupción de DOS para terminar el programa. |

**Total del programa: 13 bytes**

### Observaciones Parte C
- El formato little-endian del 8086 almacena el byte menos significativo primero. Por eso `0x0004` se guarda como `04 00` y no como `00 04`.
- `LOOP` usa un desplazamiento relativo de 8 bits con signo, lo que permite saltos dentro de un rango de ±127 bytes desde la siguiente instrucción.
- El cálculo del desplazamiento de LOOP: dirección destino (0x0106) − dirección de la instrucción siguiente (0x010B) = −5 = `0xFB` en complemento a dos.
- Un programa que realiza 4 sumas de forma iterativa ocupa solo 13 bytes, lo que demuestra la eficiencia del uso de bucles con `LOOP` frente a repetir instrucciones `ADD` individualmente.

---
