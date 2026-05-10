# Versión 2.4 — Corrección lógica de Pagos por Módulo

Fecha: 09/05/2026

## Resumen

Cuatro arreglos en la pantalla **`/pagos/por-modulo/`** y sus
exportaciones a Excel y PDF. La lógica visible en pantalla ahora
coincide con la regla que ya prometía el subtítulo y con el modelo
mental binario del negocio (un módulo está pagado o no — punto).

> *"La reserva se distribuye automáticamente al primer módulo: si el
> estudiante pagó reserva + módulo 1 por el total del curso, ese
> módulo aparece como Pagado."*

---

## Bug 1 — La reserva no se distribuía (el síntoma visible original)

**Antes:** un estudiante con reserva $20 + Módulo 1 $50 (curso $120 / 2
módulos de $60 c/u) aparecía así:

| | Mód. 1 | Mód. 2 |
|---|---|---|
| Pantalla anterior (incorrecta) | ◐ $50 de $60 | ○ Sin pagar $60 |

**Ahora:**

| | Mód. 1 | Mód. 2 |
|---|---|---|
| Pantalla actual (correcta) | ✓ Pagado $60 | ◐ $10 de $60 |

El total pagado ($70) se distribuye así: $60 llenan el módulo 1 (con la
reserva $20 + los primeros $40 del módulo 1) y los $10 restantes
*derraman* al módulo 2 como abono parcial. El saldo pendiente ($50)
queda visible donde corresponde: en el módulo 2.

### Causa raíz

`academia/views_pagos.py:1516` llamaba a `m.pagos_por_modulo()`, que
**solo cuenta los abonos asignados explícitamente a un módulo** y no
incluye la reserva ni los abonos libres. Existía un método correcto en
el modelo (`pagos_por_modulo_efectivo()`) que sí distribuye, pero la
matriz no lo usaba.

### Solución

Se introdujo un método nuevo en `Matricula` que devuelve TODO en una
sola llamada (cantidad + estado + fecha) usando un algoritmo de "río
con derrame" que preserva la fecha del abono al pasar de un módulo al
siguiente:

```python
Matricula.desglose_pagos_por_modulo()
# → [{'numero', 'pagado', 'esperado', 'estado', 'fecha_ultimo_pago'}, ...]
```

`_construir_matriz_pagos` ahora consume directamente este desglose, lo
que también arregla el Bug 3 abajo.

---

## Bug 2 — Excel y PDF reventaban con filtro de estado por módulo

`pagos_por_modulo_export_excel` y `pagos_por_modulo_export_pdf`
pasaban un `kwarg` que la función receptora no aceptaba:

```python
# ANTES (TypeError)
_construir_matriz_pagos(..., estado_modulo_filtro=filtros['estado_modulo'])

# AHORA
_construir_matriz_pagos(..., filtro_modulo_estado=filtros['filtro_modulo_estado'])
```

Adicionalmente, el helper `_export_pagos_modulo_filtros` leía la clave
GET equivocada (`estado_modulo`) cuando el formulario manda
`filtro_modulo_estado`. Resultado neto: hacer click en 📊 Excel o 📄
PDF con cualquier filtro de estado activo crasheaba.

Ahora ambos endpoints devuelven 200 OK y respetan el filtro.

---

## Bug 3 — Fechas perdidas tras el derrame

Antes se mostraba la fecha del módulo solo si había un abono
explícitamente asignado a él. Si el módulo 2 recibía dinero por
derrame del módulo 1, la celda se mostraba sin fecha.

`desglose_pagos_por_modulo` rastrea el origen y conserva la fecha del
abono cuya plata aterrizó en cada módulo. Si un mismo abono aporta a
dos módulos, ambos muestran su fecha.

---

## Bug 4 — Las celdas de "Parcial" confundían al equipo y al cliente

**Antes:** un abono de $20 sobre un módulo de $25 (curso $100 / 4
módulos) se renderizaba como un cuadro **amarillo** con `◐ $20,00 de
$25,00`. Visualmente parecía que ese módulo "ya tenía avance" cuando
la realidad financiera es que el estudiante apenas tenía $20
abonados al curso entero — ningún módulo estaba cubierto todavía.

**Ahora:** la matriz muestra solo **2 estados visuales** —
✓ Pagado (verde) o ○ Sin pagar (rojo). Cuando hay un abono parcial
que no cubre el módulo, la celda se muestra como Sin pagar pero con
una nota pequeña en gris debajo:

```
○ Sin pagar
$25,00
abonó $20,00
09/05/2026
```

Así el equipo no se confunde ("¿está pagado o no?") y al mismo tiempo
no se pierde la trazabilidad del dinero que sí entró.

### Lo que NO cambió (a propósito)

A petición expresa, los siguientes lugares **siguen mostrando los 3
estados** porque siguen siendo útiles para reportes y filtros:

- Las **tarjetas resumen del top** (`✓ X · ◐ Y · ○ Z` por módulo).
- El **filtro "Estado por Módulo"** del formulario, que permite
  filtrar por Pagado, Parcial o Pendiente.
- Las exportaciones a Excel y PDF mantienen los 3 estados detallados
  para la contabilidad.
- El método del modelo `desglose_pagos_por_modulo()` sigue devolviendo
  los 3 estados — solo el render del template colapsa "Parcial" a
  "Sin pagar + nota".

---

## Archivos tocados

```
academia/
├── models.py                # +desglose_pagos_por_modulo() en Matricula
└── views_pagos.py           # Reemplaza la lógica manual por desglose_pagos_por_modulo
                            # Corrige los kwargs de los exports Excel/PDF
                            # Corrige el nombre de la GET-key en _export_pagos_modulo_filtros

templates/pagos/
└── por_modulo.html          # Visual binario en la matriz: Parcial → Sin pagar + nota
```

Sin migraciones nuevas.

---

## Verificación manual

```python
# Caso del usuario reportado
m = Matricula.objects.get(estudiante__cedula='1207342716')
for d in m.desglose_pagos_por_modulo():
    print(d)
# {'numero': 1, 'pagado': Decimal('60.00'), 'esperado': Decimal('60.00'),
#  'estado': 'Pagado', 'fecha_ultimo_pago': date(2026, 5, 9)}
# {'numero': 2, 'pagado': Decimal('10.00'), 'esperado': Decimal('60.00'),
#  'estado': 'Parcial', 'fecha_ultimo_pago': date(2026, 5, 9)}
```

Y los tres endpoints (`/pagos/por-modulo/`,
`/pagos/por-modulo/export/excel/`, `/pagos/por-modulo/export/pdf/`)
devuelven 200 con y sin el filtro `filtro_modulo_estado` activo.
