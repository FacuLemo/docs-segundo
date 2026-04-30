# Clase 6. Validaciones y Pydantic

## 1. Validaciones Avanzadas (Query & Path)

FastAPI nos permite validar datos numéricos y de texto directamente en los parámetros de la función usando `Query()` y `Path()`, para parámetros query y de ruta respectivamente. Se importan desde fastapi.

### Palabras Clave de Validación

Cada una de las siguientes palabras se usan como parámetro de las funciones `Query()` y `Path()`

* **Longitud:** `min_length`, `max_length` (para strings).
* **Numéricos:**
* `gt`: Greater than (mayor que `>`).
* `ge`: Greater than or equal (mayor o igual que `>=`).
* `lt`: Less than (menor que `<`).
* `le`: Less than or equal (menor o igual que `<=`).


* **Metadatos (de cara al frontend):** `alias`, `deprecated`, `description`.

* **Uso:**
* Parámetro de ruta (Path param)
```python
@app.get("/validar/{id}")  # Path operation que sólo retorna el parametro de ruta
async def validar_id(
    id: int = Path( #<- parametro tipo entero, asignado por defecto a Path()
        gt=0, #<- Mayor que 0
        description="Se lee el id y se devuelve. Debe ser >0", #<- Texto de ayuda
    )
):
    #lógica...
    return {"id validado": id}
```

* Parámetro Query
```python
@app.get("/query")  # Path operation que retorna el parámetro Query. Se accede localhost:8000/query?q=hola
async def validar_id(
    q: str = Query( #<- parametro tipo entero, asignado por defecto a Path()
        max_length=50, #<- NO debe tener más de 50 caracteres
        description="Se lee el query 'q' y se devuelve. No debe superar los 50 caracteres", #<- Texto de ayuda
        deprecated=True, #<- Aviso de que el query está en desuso (obsoleto) (por ejemplo)
    )
):
    #lógica...
    return {"query": q}
```


### Implementación con `Annotated`

Es la forma moderna (recomendada) de declarar validaciones. Ya no se define una igualdad a la función `Path` o `Query` sino a un `Annotated`

```python
from typing import Annotated
from fastapi import FastAPI, Query, Path

# Ejemplo con Query Param "q" (validando un filtro)
# /items?q=hola (q debe tener min 3 chars y max 50)
@app.get("/items")
def items(q: Annotated[str | None, Query(min_length=3, max_length=50)] = None): 
    #^^^Acá el query puede venir vacío, y por defecto es None
    return {"q": q}


# Ejemplo con Path Param (validando un ID)
# /items/5 (id debe ser mayor a 0)
@app.get("/items/{id}")
def get_item(id: Annotated[int, Path(title="ID del item", gt=0)]):
    return {"id": id}

```

> **Nota:** Además del `Query` y `Path` existe el `Field()`, usado para modelos de Pydantic (ver abajo).

---

## 2. Esquemas con Pydantic

En lugar de recibir parámetros sueltos, usamos clases para definir la estructura de nuestros datos. Esto garantiza validación automática de tipos, directamente desde un modelo en lugar de cuando lo recibimos en el parámetro.

Entonces creamos: (en `main.py`)

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