# sotri-tp3_04-application — Security airlock

Implementación del problema **Security airlock** (puerta esclusa, *Other problems*)
sobre FreeRTOS: un sistema de control de acceso que monitorea y controla el
ingreso/egreso de personas por **cuatro puntos** (gate_a, gate_b, gate_c, gate_d),
permitiendo el paso de **una sola persona a la vez** — normalmente las puertas
están cerradas y el sistema habilita **abrir una única puerta a la vez**.

## Arquitectura de tareas

| Tarea | Prioridad | Rol |
|---|---|---|
| `task_gate_a` / `task_gate_c` | 3 | Monitoreo y control de ingreso/egreso de personas |
| `task_gate_b` / `task_gate_d` | 2 | Monitoreo y control de ingreso/egreso de personas |
| `task_test` | 1 (se auto-eleva a 3) | Recupera estímulos de un array y excita a las demás tareas |

`task_test` recorre `e_task_test_array[]` y, por cada estímulo
(`OPEN_REQUEST_X`, `DOOR_CLOSED_X`), señaliza a la gate correspondiente. El array
a usar se selecciona con `E_TASK_TEST_X` (la prueba documentada usa `== 6`).

## Mecanismos de sincronización

| Primitiva | Tipo | Uso |
|---|---|---|
| `h_open_a/b/c/d_bin_sem` | Semáforo binario | Señalización `task_test → task_gate_X`: pedido de apertura |
| `h_closed_a/b/c/d_bin_sem` | Semáforo binario | Señalización `task_test → task_gate_X`: evento de puerta cerrada |
| `h_airlock_mut_sem` | **Mutex** | La esclusa: garantiza que **una sola puerta** esté abierta a la vez |

La idea central: **la esclusa es un mutex de capacidad 1**. Cada gate, al recibir
el pedido de apertura, **toma** el mutex (se bloquea si otra puerta está abierta),
abre su puerta, espera el evento de cierre, cierra y **libera** el mutex para la
siguiente. Como la misma tarea toma y libera el mutex, se respeta la propiedad de
*ownership* del mutex de FreeRTOS.

**Lógica de cada `task_gate_X`:**
```
Take(h_open_X)          // espera pedido de apertura (binario, arranca vacío)
Take(h_airlock_mut_sem) // entra a la esclusa: se bloquea si otra puerta está abierta
LOG "Gate X: OPEN  (puerta abierta)"
Take(h_closed_X)        // espera el evento de puerta cerrada
LOG "Gate X: CLOSED (puerta cerrada)"
Give(h_airlock_mut_sem) // libera la esclusa -> la próxima puerta puede abrir
```

## Mapa de la consigna (pasos recomendados)

- [x] Sincronización `task_test`↔`task_gate_a` para pedido de apertura y puerta cerrada (binarios).
- [x] Acceso a recursos compartidos por las gates → mutex compartido (`h_airlock_mut_sem`).
- [x] Control de puerta (Close/Open) → los `LOGGER_INFO` OPEN/CLOSED.
- [x] Repetición del procedimiento para `task_gate_b`, `task_gate_c` y `task_gate_d`.

## Comportamiento observado (`E_TASK_TEST_X == 6`)

Array de estímulos: `{OPEN_REQUEST_A, OPEN_REQUEST_B, DOOR_CLOSED_A, DOOR_CLOSED_B}`.

```
index 0: Signal OPEN_REQUEST_A  ->  Gate A - OPEN   - puerta abierta
index 1: Signal OPEN_REQUEST_B  ->  (Gate B NO abre: bloqueada en la esclusa)
index 2: Signal DOOR_CLOSED_A   ->  Gate A - CLOSED - puerta cerrada
                                    Gate B - OPEN   - puerta abierta
index 3: Signal DOOR_CLOSED_B   ->  Gate B - CLOSED - puerta cerrada
```

**Conclusión:** cuando llega el pedido de apertura de B (index 1) mientras la
puerta A está abierta, `task_gate_b` queda **bloqueada** tomando el mutex de la
esclusa — no se imprime "Gate B OPEN". Recién cuando A cierra (index 2) y libera
el mutex, B logra abrir. Se cumple el invariante de la esclusa: **una sola puerta
abierta a la vez**, y las personas pasan de a una por orden de llegada.

## Nota

El template de la cátedra creaba `task_gate_c` con el nombre `"Task Gate B"`
(bug cosmético en el string de `xTaskCreate`); se corrigió a `"Task Gate C"` para
que el log de arranque identifique correctamente cada tarea.
