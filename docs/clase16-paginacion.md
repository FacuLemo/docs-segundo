
# Clase 16: Paginación en FastAPI

## El por qué:

Hasta ahora tenemos una API bastante completa, con modelos, relaciones y validaciones de datos.
Sin embargo, si deployamos a un cliente real y tanto la api como la DB empieza a crecer, nos vamos a encontrar con un problema de rendimiento; no es lo mismo traer en un GET 10 datos a que traer 50 mil. Peor todavía si los datos vienen anidados.

Para optimizar las consultas es que existe el paginado, donde la api sólo devuelve cierta cantidad de datos (por ejemplo, 10) y para consultar el resto hay que hacer otra petición donde se muestran los próximos 10.

## Paginación Manual
La paginación puede ser implementada en nuestro código de dos maneras distintas: Con Offset o con Páginas.

### Con `limit + offset`
Para implementar el Offset debemos pedir dos argumentos como parámetro query y dejamos que el usuario decida cómo vienen los datos.
Estos parámetros definen cuántos y desde dónde vendrán los datos (No dependen de páginas fijas)

```python
from fastapi import Query

@app.get("/juegos")
def get_juegos(
    #db: Session = Depends(get_db)
    limit: int = Query(10, ge=1, le=50),
    offset: int = Query(0, ge=0)
):
    #Obtención desde SQLModel
    #Offset es a partir de qué registro traerá los datos
    #y el límite es la cantidad que traerá
    query = select(Articulo).offset(offset).limit(limit)
    juegos = db.exec(query).all()
    
    return juegos
```

---

### Con `page + size`
Acá también dejamos libertad al usuario. Para implementar el páginado lo hacemos de la siguiente manera:

```python
@app.get("/juegos")
def get_juegos( 
    #db: Session = Depends(get_db),
    page: int = 1,
    size: int = 10
):
    # Cálculo del offset a partir del nro de página y tamaño
    offset = (page - 1) * size
    # Construcción de la consulta
    query = (
        select(Articulo)
        .order_by(Articulo.id)  
        .offset(offset)
        .limit(size)
    )

    juegos = session.exec(query).all()
    return juegos
```

De esta manera creamos una paginación a mano básica. Esta lógica tendría que volver a ser utilizada en cada uno de nuestros path operations en los que deseemos tener paginación (normalmente todos los get all). Para no aplicar la lógica manualmente podemos decantarnos por usar la librería de `fastapi-pagination`

---

## Usando fastapi-pagination 

Podemos usar la librería de `fastapi-pagination` para tener mejor control de los parámetros y tener un código más limpio.

### 1. Instalación

La instalamos en el entorno virtual

```bash
pip install fastapi-pagination
```


### 2. Implementación 
Usamos las herramientas de la librería para adaptar nuestros Path Operations:

#### 1. Adaptar el Path Operation
```python
from fastapi_pagination import Page, paginate

@app.get("/juegos", response_model=Page[ArticuloPublic]) # Usamos Page como tipo de dato de salida
def get_juegos(
    db: Session = Depends(get_db)
):
    query = select(Juego).order_by(Juego.id) # Armamos la consulta
    return paginate(db, query) # Retornamos un paginate() con la db y la consulta
```

 `paginate` se encarga de añadir el LIMIT, OFFSET a la consulta y devuelve un JSON con 'items', 'total', 'page', etc; donde dentro de items tendremos cada registro obtenido.

#### 2. Agregar paginación al proyecto

```python
# src/main.py
from fastapi_pagination import add_pagination
# ... (luego de instanciar app y routers)
add_pagination(app)

```

Si todo salió bien, deberíamos ver nuestro response body de Juegos paginado, donde los elementos vendrán dentro de un campo llamado `items` con un tamaño por defecto de 50. A su vez, si quisieramos pasar de página y cambiar el tamaño de página tendremos que hacerlo a través de parametros query, de la siguiente manera: `GET /juegos?page=2&size=10`

---

# Cápsula. Filtros y Ordenamientos

Si queremos aplicar filtrado por campos y ordenamiento en nuestros endpoints, necesitamos implementar una **construcción progresiva de queries**: partimos de un `select(Juego)` base y encadenamos condiciones `.where()` únicamente para los parámetros que el cliente envió.

Definiremos los campos que acepten filtrado, y para el ordenamiento dejamos que el cliente elija el campo por el cuál ordenar los datos.

---

### Paso 1: Declarar los parámetros de filtro como opcionales

Los filtros deben tener un valor por defecto de `None` para que el endpoint responda tanto a consultas generales (`/juegos`) como a consultas específicas (`/juegos?genero=rpg`).

```python
#en los parámetros de la funcion del path operation:
genero: str | None = Query(None, description="Filtrar por género"),
estudio: str | None = Query(None, description="Filtrar por estudio"),

```

* **`str | None`**: Marca el query param como no obligatorio en OpenAPI/Swagger.
* **`description`**: Documenta el uso del parámetro en la UI interactiva `/docs`.

---

### Paso 2: Crear la consulta base antes de evaluar condiciones

Inicializa la sentencia SQL sin ejecutarla (`session.exec` o `paginate` la ejecutarán al final):

```python
query = select(Juego)

```

En este punto, `query` representa simplemente `SELECT * FROM juego`.

---

### Paso 3: Evaluar y encadenar `.where()` condicionalmente

Verifica si cada parámetro contiene un valor (distinto de `None` o cadena vacía) y actualiza la variable `query`:

```python
if genero:
    query = query.where(col(Juego.genero).ilike(f"%{genero}%"))

if estudio:
    query = query.where(col(Juego.estudio).ilike(f"%{estudio}%"))

```

* **`col(Juego.genero)`**: Helper de SQLModel que asegura el tipado estático y el acceso a los métodos de columna de SQLAlchemy.
* **`.ilike("%...%")`**: Búsqueda insensible a mayúsculas y minúsculas (*case-insensitive*) con comodines `SQL LIKE`.
* **Comportamiento acumulativo (AND implícito)**: Si el cliente envía ambos parámetros (`?genero=rpg&estudio=fromsoftware`), encadenar `.where()` equivale a un `AND` lógico en SQL:
```sql
WHERE juego.genero ILIKE '%rpg%' AND juego.estudio ILIKE '%fromsoftware%'

```


---

### Paso 4: Validar y aplicar ordenamiento seguro

Antes de aplicar el ordenamiento, valida que el campo enviado por el usuario realmente exista en el modelo para evitar fallos de servidor o inyecciones de atributos internos. Recordemos que el cliente escribirá el campo por el cuál ordernar:

```python
if sort_by:
    campo_orden = getattr(Juego, sort_by, None) # Obtiene el campo si existe, sino guarda None
    
    # 1. Validación de existencia
    if not campo_orden:
        raise HTTPException(
            status_code=400, 
            detail=f"Campo de ordenamiento inválido: '{sort_by}'"
        )
    
    # 2. Dirección de orden
    if order.lower() == "desc":
        query = query.order_by(campo_orden.desc()) # descendiente
    else: 
        query = query.order_by(campo_orden.asc()) # ascendiente por defecto

```

---

### Paso 5: Delegar la paginación a `paginate()`

Una vez que `query` tiene todos los filtros y el orden aplicados, pásala a `paginate(db, query)`.

```python
return paginate(db, query)

```

`fastapi-pagination` realiza dos acciones en la base de datos automáticamente:

1. Genera un `SELECT COUNT(*) FROM (...)` con los mismos filtros para calcular el total de registros coincidentes.
2. Inyecta el `.limit()` y `.offset()` correspondientes a la página solicitada (`page` y `size`) y retorna el esquema `Page[JuegoPublic]`.

---

## Path Operation Final


```python
from typing import Optional
from fastapi import FastAPI, Query, Depends, HTTPException
from sqlmodel import SQLModel, Field, Session, select, col
from fastapi_pagination import Page, add_pagination, paginate

#...

@app.get("/juegos", response_model=Page[JuegoPublic], tags=["Juegos"])
def listar_juegos(
    db: Session = Depends(get_db),
    # Parámetros de filtrado
    genero: str | None = Query(None, description="Filtrar por género"),
    estudio: str | None = Query(None, description="Filtrar por estudio"),
    # Parámetros de ordenamiento
    sort_by: str | None = Query(None, description="Columna para ordenar"),
    order: str = Query("asc", description="Dirección: 'asc' o 'desc'")
):
    # Iniciamos la consulta base
    query = select(Juego)

    # --- APLICAMOS FILTROS ---
    if genero:
        query = query.where(col(Juego.genero).ilike(f"%{genero}%"))
    if estudio:
        query = query.where(col(Juego.estudio).ilike(f"%{estudio}%"))

    # --- APLICAMOS ORDENAMIENTO ---
    if sort_by:
        campo_orden = getattr(Juego, sort_by, None)
        
        if not campo_orden:
            raise HTTPException(
                status_code=400, 
                detail=f"Campo de ordenamiento inválido: '{sort_by}'"
            )
            
        if order.lower() == "desc":
            query = query.order_by(campo_orden.desc())
        else:
            query = query.order_by(campo_orden.asc())

    # --- APLICAMOS PAGINACIÓN Y EJECUTAMOS ---
    return paginate(db, query)
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