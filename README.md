# Automatización ETL con el Editor Avanzado de Power Query

Este repositorio contiene el script en Lenguaje M desarrollado para limpiar, estandarizar y tipar la tabla `ventas_raw` de TechStore (7 registros originales reducidos a 5 ventas productivas válidas tras depurar espacios, mayúsculas inconsistentes y registros de prueba).

## Estructura del repositorio
- `script_limpieza.md`: Código M completo y comentado paso a paso.
- `README.md`: Análisis técnico sobre el comportamiento de las funciones de texto y el orden de ejecución en Power Query.

---

## Respuestas técnicas

### 1. Si en lugar de `Text.Proper` hubieras usado `Text.Upper` para estandarizar la categoría, ¿qué tendrías que cambiar en el filtro de los registros de prueba? ¿Por qué?
Tendría que modificar la condición del paso `EliminarPruebas` para comparar contra `"PRUEBA"` todo en mayúsculas:

```powerquery
EliminarPruebas = Table.SelectRows(EstandarizarCategoria, each [categoria] <> "PRUEBA")
```

**Motivo:** El lenguaje M es *Case Sensitive* (distingue entre mayúsculas y minúsculas). Si `Text.Upper` transforma la columna a mayúsculas (`"COMPUTACIÓN"`, `"ACCESORIOS"`, `"PRUEBA"`), dejar la condición `each [categoria] <> "Prueba"` haría que Power Query no encuentre ninguna coincidencia exacta y deje pasar los registros de prueba (`id_venta` 3 y 6) hacia el modelo final.

---

### 2. Si mañana llega un registro con la categoría `"prueba "` (con un espacio al final), ¿tu script lo elimina? Si no, ¿qué paso agregarías y en qué lugar del script?
**No, el script actual no lo eliminaría.** Al pasar por `Text.Proper`, el valor `"prueba "` se convertiría en `"Prueba "` (conservando el espacio final). Como la comparación exacta `"Prueba " <> "Prueba"` evalúa como verdadera, ese registro pasaría el filtro como si fuera una venta real.

**Solución:** Aplicaría `Text.Trim` sobre la columna `categoria` **antes** de ejecutar el filtro `EliminarPruebas` (por ejemplo, integrándolo dentro del paso `LimpiarEspacios` junto con `nombre_producto`):

```powerquery
LimpiarEspacios = Table.TransformColumns(
    Origen,
    {
        {"nombre_producto", Text.Trim, type text},
        {"categoria", Text.Trim, type text}
    }
),
```

---

### 3. ¿Cuál es la diferencia entre usar `Text.Trim` y `Text.Clean` en M?
Ambas funciones limpian cadenas de texto, pero actúan sobre caracteres distintos:
- **`Text.Trim`:** Elimina exclusivamente los **espacios en blanco** (`" "`) que sobran al principio y al final de un texto (por ejemplo, convierte `" Laptop Pro 15 "` en `"Laptop Pro 15"`), sin tocar los espacios internos entre palabras ni saltos de línea.
- **`Text.Clean`:** Elimina **caracteres de control no imprimibles** (códigos ASCII del 0 al 31, como saltos de línea, retornos de carro o tabulaciones ocultas) que suelen aparecer cuando se exportan datos desde sistemas legacy o archivos planos, pero **no** elimina los espacios en blanco comunes al inicio o al final.

---

### 4. ¿Por qué filtraste los registros `"PRUEBA"` después de estandarizar la categoría y no antes?
Porque en los datos reales provenientes de sistemas legacy, un registro de prueba puede cargarse como `"PRUEBA"`, `"prueba"` o `"Prueba"`. Dado que M es *Case Sensitive*, filtrar antes de estandarizar obligaría a escribir múltiples condiciones (`each [categoria] <> "PRUEBA" and [categoria] <> "prueba" and ...`), corriendo el riesgo de que alguna variante no contemplada se filtre al reporte.

Al ejecutar primero `Text.Proper` en el paso `EstandarizarCategoria`, todas las variantes posibles quedan normalizadas bajo una única forma (`"Prueba"`), permitiendo que un solo filtro simple y seguro elimine todos los casos sin excepciones.
