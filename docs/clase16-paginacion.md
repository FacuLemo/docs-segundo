
# Clase 16: Paginación en FastAPI

## El por qué:

Hasta ahora tenemos una API bastante completa, con modelos, relaciones y validaciones de datos.
Sin embargo, si deployamos a un cliente real y tanto la api como la DB empieza a crecer, nos vamos a encontrar con un problema de rendimiento; no es lo mismo traer en un GET 10 datos a que traer 50 mil. Peor si los datos vienen anidados.

Para eso existe el paginado, donde la api sólo devuelve cierta cantidad de datos (por ejemplo, 10) y para consultar el resto hay que hacer otra petición donde se muestran los próximos 10.

## Paginación Manual
Esta paginacion la podemos implementar nosotros en nuestro código de dos maneras distintas: Con Offset o con paginado propiamente dicho.

### Con `limit + offset`
Para implementar el Offset debemos pedir dos argumentos como parámetro query y dejamos que el usuario decida cómo vienen los datos.
Estos parámetros definen cuántos y desde dónde vendrán los datos (No dependen de páginas fijas)

```python
from fastapi import Query

@app.get("/users")
def list_users(
    limit: int = Query(10, ge=1, le=50),
    offset: int = Query(0, ge=0)
):
    return fake_db[offset: offset + limit]
```

---

### Con `page + size`
Acá también dejamos libertad al usuario. Para implementar el páginado lo hacemos de la siguiente manera:

```python
@app.get("/users")
def list_users(page: int = 1, size: int = 10):
    start = (page - 1) * size
    return fake_db[start:start + size]
```

---

## Usando fastapi-pagination 

Podemos usar una librería para tener mejor control de estos parámetros:

### 1. Instalación

La instalamos en el entorno virtual

```bash
pip install fastapi-pagination

```


### 2. Implementación mínima
Usamos las herramientas de la librería para adaptar nuestros Path Operations:

```python
from fastapi_pagination import Page, paginate, add_pagination

class UserOut(BaseModel):
    id: int
    name: str

@app.get("/users", response_model=Page[UserOut])
def get_users():
    return paginate(db)

add_pagination(app)

```

De esta manera tendríamos la versión más básica de fastapi-pagination.


---

## Código Completo (Todo en uno)

Este es el esqueleto completo de cómo se ve tu archivo principal. Incluye las importaciones correctas, la lógica de la base de datos, el filtrado, el ordenamiento y la inicialización de la paginación.

```python
from typing import Optional
from fastapi import FastAPI, Query, Depends, HTTPException
from sqlmodel import SQLModel, Field, Session, select, col
from fastapi_pagination import Page, add_pagination
from fastapi_pagination.ext.sqlmodel import paginate

app = FastAPI(title="API de Juegos con Paginación, Filtros y Ordenamiento")

# ==========================================
# 1. MODELOS DE SQLMODEL (Ejemplo simplificado)
# ==========================================
class Juego(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    titulo: str
    genero: str
    estudio: str
    score: float

# Usaremos el mismo modelo para la salida pública por simplicidad, 
# pero puedes usar tu propio 'JuegoPublic'
JuegoPublic = Juego 

# ==========================================
# 2. EL ENDPOINT PAGINADO
# ==========================================
@app.get("/juegos", response_model=Page[JuegoPublic], tags=["Juegos"])
def listar_juegos(
    db: Session = Depends(get_db),
    # Parámetros de filtrado
    genero: Optional[str] = Query(None, description="Filtrar por género"),
    estudio: Optional[str] = Query(None, description="Filtrar por estudio"),
    # Parámetros de ordenamiento
    sort_by: Optional[str] = Query(None, description="Columna para ordenar"),
    order: str = Query("asc", description="Dirección: 'asc' o 'desc'")
):
    # Iniciamos la consulta base
    query = select(Juego)

    # --- APLICAR FILTROS ---
    if genero:
        query = query.where(col(Juego.genero).ilike(f"%{genero}%"))
    if estudio:
        query = query.where(col(Juego.estudio).ilike(f"%{estudio}%"))

    # --- APLICAR ORDENAMIENTO ---
    if sort_by:
        campo_orden = getattr(Juego, sort_by, None)
        
        # Validar que la columna exista para evitar errores 500
        if not campo_orden:
            raise HTTPException(
                status_code=400, 
                detail=f"Campo de ordenamiento inválido: '{sort_by}'"
            )
            
        if order.lower() == "desc":
            query = query.order_by(campo_orden.desc())
        else:
            query = query.order_by(campo_orden.asc())

    # --- APLICAR PAGINACIÓN Y EJECUTAR ---
    # Esto añade el LIMIT, OFFSET y devuelve el formato JSON con 'items', 'total', 'page', etc.
    return paginate(db, query)

# ==========================================
# 3. INICIALIZAR LA PAGINACIÓN EN LA APP
# ==========================================
add_pagination(app)

```

---

## Resumen de Uso (Parámetros de URL)
Si implementamos la librería como en el código anterior, nuestra API será capaz de procesar múltiples escenarios dinámicos enviando los parámetros adecuados en la URL (`query parameters`).

* **Paginación básica (por defecto devuelve page=1, size=50):**
`GET /juegos`
* **Cambiar la página y la cantidad de elementos:**
`GET /juegos?page=3&size=20`
* **Aplicar filtros simples o múltiples:**
`GET /juegos?genero=RPG`
`GET /juegos?genero=Aventura&estudio=Nintendo`
* **Aplicar ordenamiento ascendente o descendente:**
`GET /juegos?sort_by=titulo` *(ascendente por defecto)*
`GET /juegos?sort_by=score&order=desc`
* **La consulta completa:**
`GET /juegos?genero=Shooter&sort_by=score&order=desc&page=1&size=10`