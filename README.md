# Caso de Estudio: Manejo de Consistencia en Sistemas de Pagos Distribuidos

Este repositorio presenta un análisis técnico sobre el manejo de **consistencia, fallas parciales e integridad de datos** en arquitecturas de microservicios orientadas al procesamiento de pagos.

El objetivo es analizar las principales alternativas arquitectónicas y sus trade-offs, considerando escenarios como fallas de red, reintentos, procesamiento duplicado y divergencia temporal de estados.

## Tabla de Contenidos

1. [Introducción](#introducción)
2. [El Problema: Fallas en Sistemas Distribuidos](#el-problema-fallas-en-sistemas-distribuidos)
3. [Estrategias de Solución](#estrategias-de-solución)
4. [Monitoreo y Conciliación](#monitoreo-y-conciliación)
5. [Consideraciones Arquitectónicas](#consideraciones-arquitectónicas)
6. [Conclusiones](#conclusiones)

---

## 1. Introducción

El procesamiento de pagos en una arquitectura distribuida presenta un desafío fundamental: una operación de negocio puede involucrar múltiples servicios y fuentes de datos que no comparten una única transacción local.

En este contexto, mantener consistencia fuerte entre todos los componentes puede introducir costos importantes en términos de latencia, disponibilidad y complejidad operativa.

Este caso de estudio explora estrategias basadas en **consistencia eventual, coordinación mediante Sagas, idempotencia y Transactional Outbox**, complementadas con mecanismos de observabilidad y conciliación.

---

## 2. El Problema: Fallas en Sistemas Distribuidos

En sistemas basados en microservicios, las fallas parciales forman parte del comportamiento normal del sistema.

Algunos escenarios relevantes son:

### Timeouts de red

Un servicio puede procesar correctamente una operación, pero la respuesta puede perderse antes de llegar al consumidor.

Esto genera una situación ambigua:

> ¿La operación falló o simplemente no recibimos la respuesta?

### Condiciones de carrera y solicitudes duplicadas

Los clientes o componentes intermedios pueden realizar reintentos ante un timeout.

Sin mecanismos de idempotencia, una misma operación podría procesarse más de una vez.

### Divergencia temporal de estados

Cuando una operación involucra varios servicios, los estados pueden no actualizarse simultáneamente.

Por ejemplo:

```text
Order
  |
  v
Payment
  |
  v
External Gateway
```

Un pago puede haber sido aceptado por la pasarela mientras el servicio de órdenes todavía mantiene el estado `PENDING`.

Esta divergencia no necesariamente representa una inconsistencia permanente, pero requiere mecanismos para converger hacia un estado correcto.

---

## 3. Estrategias de Solución

### 3.1 Saga — Orquestación

El patrón Saga permite representar una transacción distribuida como una secuencia de transacciones locales.

Cada paso modifica el estado local de un servicio y, cuando es necesario, puede existir una operación compensatoria.

Un ejemplo simplificado:

```text
Create Order
     |
     v
Reserve Payment
     |
     v
Confirm Payment
     |
     v
Complete Order
```

Si una operación posterior falla, el proceso puede ejecutar las compensaciones correspondientes.

La principal ventaja es evitar la necesidad de mantener una única transacción distribuida durante todo el flujo.

El costo es una mayor complejidad en la gestión del estado y de los escenarios de error.

---

### 3.2 Idempotencia

Las operaciones que pueden recibir reintentos deben diseñarse para que procesar la misma solicitud más de una vez no produzca efectos duplicados.

Una estrategia habitual consiste en utilizar una **Idempotency Key** asociada a cada operación.

Por ejemplo:

```text
POST /payments

Idempotency-Key:
7f4d8e2a-...
```

El servicio puede almacenar el resultado asociado a esa clave y devolver el mismo resultado ante un reintento.

La idempotencia es especialmente importante cuando existen:

* timeouts;
* reintentos automáticos;
* mensajes duplicados;
* fallas de comunicación;
* procesamiento asíncrono.

---

### 3.3 Transactional Outbox

El patrón Transactional Outbox permite resolver un problema frecuente:

> ¿Cómo garantizamos que un cambio realizado en una base de datos local y el evento correspondiente no queden en estados diferentes?

En lugar de actualizar la entidad y publicar directamente un evento, el servicio registra ambos cambios dentro de su transacción local:

```text
Database Transaction
       |
       +---- Business Data
       |
       +---- Outbox Event
```

Posteriormente, un proceso de publicación recupera los eventos pendientes y los envía al sistema de mensajería.

La publicación puede implementarse mediante diferentes mecanismos, incluyendo polling o Change Data Capture (CDC).

El patrón no elimina todos los problemas de entrega, por lo que el consumidor también debe considerar idempotencia y procesamiento repetible.

---

## 4. Monitoreo y Conciliación

La consistencia eventual requiere mecanismos que permitan detectar y resolver divergencias.

### 4.1 Trazabilidad Distribuida

Un `Trace ID` permite seguir una operación a través de múltiples servicios.

Por ejemplo:

```text
Client
  |
  +-- Order Service
        |
        +-- Payment Service
              |
              +-- Payment Gateway
```

El mismo identificador permite reconstruir el flujo completo y facilita el diagnóstico de fallas.

---

### 4.2 Conciliación

En sistemas de pagos, la conciliación constituye una segunda línea de control.

Un proceso de conciliación puede comparar:

```text
Estado interno
      VS
Estado de proveedor externo
```

Por ejemplo:

```text
Internal:  PAYMENT_PENDING
External:  APPROVED
```

La divergencia puede generar un proceso de recuperación o revisión.

Esto permite que la arquitectura no dependa exclusivamente de que todos los mensajes y respuestas sean procesados correctamente en tiempo real.

---

## 5. Consideraciones Arquitectónicas

No existe una única estrategia correcta para todos los escenarios.

La elección depende de factores como:

* necesidad de consistencia;
* latencia aceptable;
* disponibilidad requerida;
* duración de la operación;
* capacidad de compensación;
* comportamiento de los sistemas externos;
* volumen transaccional;
* complejidad operativa.

Por ejemplo, una operación crítica que requiere consistencia inmediata puede justificar mecanismos de coordinación más fuertes, mientras que otros procesos pueden beneficiarse de una arquitectura basada en eventos y consistencia eventual.

El punto central es entender que **la consistencia es una decisión arquitectónica y no simplemente una propiedad técnica de la base de datos**.

---

## 6. Conclusiones

Los sistemas de pagos distribuidos deben asumir que pueden producirse fallas parciales, mensajes duplicados, timeouts y divergencias temporales de estado.

Patrones como **Saga, Idempotency y Transactional Outbox**, combinados con observabilidad y conciliación, permiten construir mecanismos para manejar estos escenarios sin depender necesariamente de una única transacción distribuida.

La principal consecuencia es un desplazamiento de la complejidad:

> La complejidad que antes podía estar concentrada en una transacción distribuida pasa a distribuirse entre el modelo de estados, los mecanismos de compensación, la mensajería, la observabilidad y los procesos de reconciliación.

Por ello, la decisión arquitectónica no debería ser simplemente:

**“¿Usamos consistencia fuerte o eventual?”**

sino:

**“¿Qué nivel de consistencia necesita cada parte del negocio y qué complejidad estamos dispuestos a asumir para conseguirlo?”**

---

*Este documento es un caso de estudio técnico orientado a arquitectos y desarrolladores interesados en sistemas distribuidos, procesamiento de pagos y consistencia de datos.*
