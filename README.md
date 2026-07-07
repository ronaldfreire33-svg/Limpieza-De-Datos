# Limpieza básica de tablas en SQL

*Ejercicio práctico — proceso personal de un analista Jr.*

> **Nota sobre este ejercicio**
> Para fines demostrativos se usó una tabla de solo 20 filas, con fallas típicas (nulos, vacíos, espacios, mayúsculas/minúsculas mezcladas y códigos ruido). El objetivo no es el volumen de datos, sino documentar el razonamiento y los pasos que seguí para diagnosticar y corregir una tabla — el mismo proceso aplica a tablas de miles de filas.
> 
<img width="446" height="450" alt="32Captura de pantalla 2026-07-07 151216" src="https://github.com/user-attachments/assets/5cff5094-2e10-443b-a858-53cb7afaaf4e" />

## Contexto

Este documento resume el proceso que seguí para limpiar una tabla llamada `data`, simulando una tarea real de limpieza de datos como analista Jr. Trabajé sobre cuatro columnas: `precio`, `cliente`, `producto` y `categoria`.

## Paso 1 — Exploración inicial

Antes de tocar cualquier dato, reviso la estructura de la tabla para entender qué columnas tiene.

```sql
SELECT * FROM data LIMIT 5;
```

Esto no corrige nada todavía — solo sirve para ubicarme.

<img width="800" height="319" alt="image" src="https://github.com/user-attachments/assets/bfc3e0d4-9c09-4721-8407-7f75e7058076" />

## Paso 2 — Diagnóstico de nulos y vacíos

El siguiente paso es medir, no adivinar. Antes de decidir cómo limpiar una columna, corro un conteo de nulos (`NULL`) y vacíos (`''`) por columna.

```sql
SELECT
    COUNT(*) AS total_filas,
    SUM(CASE WHEN precio IS NULL THEN 1 ELSE 0 END) AS nulls_precio,
    SUM(CASE WHEN cliente = '' THEN 1 ELSE 0 END) AS vacios_cliente,
    SUM(CASE WHEN cliente IS NULL THEN 1 ELSE 0 END) AS nulls_cliente
FROM "data";
```
<img width="794" height="300" alt="image 2" src="https://github.com/user-attachments/assets/601a977e-9162-4f05-a162-ae51ad7366f1" />



**Resultado:** total_filas = 20, nulls_precio = 5, vacios_cliente = 1, nulls_cliente = 2.

### Aprendizaje clave de este paso

- `NULL` y `''` (vacío) son cosas distintas. Un `= ''` nunca detecta un `NULL`, y un `IS NULL` nunca detecta un `''`. Hay que medir ambos por separado.
- Las columnas numéricas (como `precio`) nunca pueden ser `''` — solo `NULL`. Por eso ahí el conteo de vacíos siempre da 0, y es normal.
- En la grilla del editor, `NULL` y `''` se ven igual (celda en blanco). La única forma confiable de diferenciarlos es con código, no a simple vista.

## Cómo decido qué código usar, según el diagnóstico

| Situación de la columna | Código a usar |
|---|---|
| Numérica, 0 nulos | Se usa directa, sin envolver nada |
| Numérica, con nulos | `COALESCE(columna, 0)` |
| Texto, solo NULL | `COALESCE(columna, 'valor')` |
| Texto, NULL + vacíos/espacios | `COALESCE(NULLIF(TRIM(columna), ''), 'valor')` |

## Cómo funciona la cadena TRIM → NULLIF → COALESCE

```sql
COALESCE(NULLIF(TRIM(columna), ''), 'Texto de reemplazo')
```

- **TRIM(columna):** quita espacios invisibles al inicio/final. Un `' '` se convierte en `''`.
- **NULLIF(..., ''):** si el resultado es `''`, lo convierte en `NULL`. Así, vacíos y nulos quedan unificados.
- **COALESCE(..., 'valor'):** reemplaza ese `NULL` unificado por el texto que yo defina.

El TRIM lo dejo casi como hábito de seguridad, porque los espacios en blanco no se ven en la grilla — no rompe nada aplicarlo aunque la columna no los tenga.

## Paso 2.B — Diagnóstico de anomalías (texto mal escrito o duplicado)

Una vez resuelto NULL/vacío, sigo con datos que sí existen pero están mal escritos, duplicados por mayúsculas, o mezclados con ruido (códigos que no deberían estar).

```sql
SELECT producto, categoria, COUNT(*) AS repeticiones
FROM "data"
GROUP BY producto, categoria
ORDER BY producto, categoria DESC;
```
<img width="547" height="350" alt="image 3" src="https://github.com/user-attachments/assets/f1f0cb86-38ae-4152-bea9-bf14d3707604" />

> **Nota:** en un `ORDER BY` con varias columnas, el ASC/DESC no se hereda entre columnas — cada una necesita su propia dirección si quiero controlarla.

Con este resultado identifiqué que **producto** y **categoria** tenían el mismo tipo de falla: mayúsculas y minúsculas mezcladas ("Teclado" / "teclado" / "TECLADO"), sin necesitar duplicados reales fuera de eso.

## Paso 3 — Solución de las anomalías

Para estandarizar texto mal escrito por mayúsculas/espacios, uso TRIM junto con UPPER (o LOWER, según el criterio del reporte).

```sql
SELECT id,
    COALESCE(precio, 0) AS precio_limpio,
    COALESCE(NULLIF(TRIM(UPPER(cliente)), ''), 'Cliente Desconocido') AS cliente_limpio,
    COALESCE(NULLIF(TRIM(UPPER(producto)), ''), 'producto_desconocido') AS producto_limpio,
    TRIM(UPPER(categoria)) AS categoria_limpia
FROM "data";
```
<img width="721" height="500" alt="image 4" src="https://github.com/user-attachments/assets/9e15352c-caf1-4b90-9e8b-bfa818138dab" />


## Guardar la tabla limpia para reutilizarla

Para no repetir la consulta cada vez que la necesito, la guardo como una vista. Si los datos originales de `data` cambian, la vista se actualiza sola porque solo guarda la consulta, no una copia fija:

```sql
CREATE VIEW data_limpia AS
SELECT id,
    COALESCE(precio, 0) AS precio_limpio,
    COALESCE(NULLIF(TRIM(UPPER(cliente)), ''), 'Cliente Desconocido') AS cliente_limpio
FROM "data";
```

Después, la uso como cualquier otra tabla, con los nombres de columna nuevos:

```sql
SELECT * FROM data_limpia WHERE cliente_limpio = 'MARIA LOPEZ';
```

## Reflexión personal

- Antes de limpiar cualquier columna, primero diagnostico — no asumo qué está sucio.
- NULL y vacío (`''`) son errores distintos y requieren manejo distinto; hay que medir ambos por separado.
- No limpio de más: si me piden limpiar dos columnas, no toco las demás sin que me lo pidan.
- Armo la cadena de código mínima necesaria según el diagnóstico, no la máxima por costumbre.
- Documentar qué reemplacé y por qué es parte del trabajo, no un extra — ayuda a quien revise el resultado después.
