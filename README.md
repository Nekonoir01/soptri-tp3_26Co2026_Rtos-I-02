# CESE - Sistemas Operativos de Tiempo Real
## Trabajo Práctico N°: 3 - Sincronización de Tareas de FreeRTOS
### CE2026-gr2

### Integrantes del grupo:
| N° SIU | Apellidos, Nombres                   |
| :----- | :----------------------------------- |
| e2610  | Nuñez Cuji, Marcos Neptali           |
| e2607  | Morales Gariglio, Matías             |
| e2623  | Berrezueta Guerrero, Rodrigo Antonio |

### Responsable de la entrega:
| N° SIU | Apellidos, Nombres         | Fecha      | Deadline  |
| :----- | :------------------------- | :--------: | :-------: |
| e2610  | Berrezueta Guerrero, Rodrigo Antonio | 2026-06-13 | Semana 08 |

---

## Actividades

| Actividad | Aplicación | Problema de sincronización | Mecanismos |
| :-------- | :--------- | :------------------------- | :--------- |
| TP3-01 | `sotri-tp3_01-application` | Producer - Consumer (Mutual exclusion) | Mutex + semáforo binario |
| TP3-02 | `sotri-tp3_02-application` | Readers - Writers | Mutex + semáforo binario |
| TP3-03 | `sotri-tp3_03-application` | Vehicular crossing | Binarios (señalización) + contador + mutex |
| TP3-04 | `sotri-tp3_04-application` | Security airlock | Binarios (señalización) + mutex (esclusa) |

Cada aplicación incluye su archivo `sotri-tp3_0X-application.md` con el análisis
del código y el comportamiento observado.
