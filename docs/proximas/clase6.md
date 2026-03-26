# Clase 6. Validaciones y Pydantic

## 1. Validaciones Avanzadas (Query & Path)

FastAPI nos permite validar datos numéricos y de texto directamente en los parámetros de la función usando `Query` y `Path`.

### Palabras Clave de Validación

* **Longitud:** `min_length`, `max_length` (para strings).
* **Numéricos:**
* `gt`: Greater than (mayor que `>`).
* `ge`: Greater than or equal (mayor o igual que `>=`).
* `lt`: Less than (menor que `<`).
* `le`: Less than or equal (menor o igual que `<=`).


* **Metadatos:** `alias`, `deprecated`, `description`.

### Implementación con `Annotated`

Es la forma moderna (recomendada) de declarar validaciones.

```python
from typing import Annotated
from fastapi import FastAPI, Query, Path

# Ejemplo con Query Param "q" (validando un filtro)
# /items?q=hola (q debe tener min 3 chars y max 50)
def items(q: Annotated[str | None, Query(min_length=3, max_length=50)] = None):
    return {"q": q}

# Ejemplo con Path Param (validando un ID)
# /items/5 (id debe ser mayor a 0)
@app.get("/items/{id}")
def get_item(id: Annotated[int, Path(title="ID del item", gt=0)]):
    return {"id": id}

```

> **Nota:** `Field()` funciona igual que `Query` y `Path` pero se usa dentro de los modelos Pydantic (ver abajo).

---

## 2. Esquemas con Pydantic

En lugar de recibir parámetros sueltos, usamos clases para definir la estructura de nuestros datos. Esto garantiza validación automática de tipos.

Entonces creamos: (en `main.py` o `schemas.py`)

```python
from pydantic import BaseModel
from typing import Optional

# Esquema principal para el Juego
class Juego(BaseModel):
    id: int  
    titulo: str
    genero: str
    score: int

# Esquema específico para actualización (PUT/PATCH)
# A veces queremos que NO se pueda editar el ID, por eso creamos otro modelo
class JuegoUpdate(BaseModel):
    titulo: str
    genero: str
    score: int

```

---

## 3. Tipado de Respuestas y Errores (GET)

Podemos definir qué estructura de datos va a devolver nuestra API usando `->` en la función. Esto ayuda a FastAPI a filtrar datos y generar documentación correcta.

```python
from fastapi import HTTPException

# 1. Va a retornar una lista de objetos Juego
@app.get("/juegos")
async def get_juegos() -> list[Juego]: 
    return juegos

# 2. Retornar un solo objeto Juego + Manejo de Error 404 (se debe importar)
@app.get("/juego/{id}")
def juego(id: int) -> Juego:
    for juego in juegos:
        if juego["id"] == id:
            return juego
    
    # Si no lo encuentra, lanzamos una excepción HTTP
    raise HTTPException(status_code=404, detail="Juego no encontrado")

```

---

## 4. Recibir Esquemas en POST y PUT

Ahora nuestros endpoints son mucho más limpios. En lugar de pedir `titulo`, `genero`, `score` por separado, pedimos un objeto `juego` completo.

### Método POST (Crear)

```python
@app.post("/agregar-juego")
def agregar_juego(juego: Juego) -> list[Juego]:
    # .model_dump() convierte el objeto Pydantic a un diccionario de Python
    # Nota: en versiones viejas de Pydantic v1 se usaba .dict()
    juegos.append(juego.model_dump()) 
    return juegos

```

### Método PUT (Editar)

Aquí combinamos un **Path Parameter** (`id`) para saber cuál editar, y un **Body** (`juego`) con los datos nuevos.

```python
@app.put("/editar-juego/{id}")
def editar_juego(id: int, juego: JuegoUpdate) -> Juego:
    for item in juegos:
        if item["id"] == id:
            item["titulo"] = juego.titulo
            item["genero"] = juego.genero
            item["score"] = juego.score
            return item
            
    raise HTTPException(status_code=404, detail="Juego no encontrado")

```