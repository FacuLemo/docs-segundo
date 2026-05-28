# Clase 9. Modularización con APIRouter

## 1. Nueva Estructura del Proyecto

Hasta ahora teníamos todo en un solo archivo. Para trabajar profesionalmente, dividimos el código en carpetas y archivos según su responsabilidad. Esto se llama modularizar el proyecto.

**Estructura:**

```text
src/
├── main.py           # Punto de entrada (solo configuración global)
├── schemas/          # Modelos de Pydantic (Antes estaban en main)
    └── juegos.py     # schemas para juegos   
└── routers/          # Carpeta para las rutas (Path operations)
    └── juegos.py     # Lógica de los endpoints de juegos

```

> **Nota:** En esta clase forzamos la nomenclatura de **Schemas** para los modelos de Pydantic, para no confundirlos con los "Modelos" de base de datos (ORM) en el futuro.

> **Nota 2:** En el video yo nombro los .py de cada subcarpeta volviendo a especificar qué es (por ejemplo, juegos_schema.py), pero en realidad esto en una redundancia.

---

## 2. Paso 1: Mover los Schemas

Cortamos las clases `BaseModel` (`Juego`, `JuegoUpdate`) de `main.py` y las pegamos en `schemas.py`.

**Archivo:** `schemas/juegos.py`

```python
from pydantic import BaseModel, Field
from typing import Annotated

class Juego(BaseModel):
    id: Annotated[int, Field(gt=0)]
    titulo: str
    # ... etc

```

---

## 3. Paso 2: Crear el Router

En lugar de usar `app = FastAPI()`, usamos `APIRouter` dentro de nuestros módulos. Esto nos permite definir rutas que luego "conectaremos" a la app principal (en main.py).

**Archivo:** `routers/juegos.py`

```python
from fastapi import APIRouter, HTTPException
from schemas.juegos import Juego, JuegoUpdate # Importamos Juego del módulo schemas

# Creamos la instancia del router
router = APIRouter() # también puede ser "juegos" directamente

# Base de datos simulada (la traemos del main.py)
juegos = [...]

# Cambiamos @app por @router
@router.get("/") #Ponemos "/" en la ruta porque luego encapsulamos todo en un prefijo "juegos"
async def get_juegos() -> list[Juego]:
    return juegos

@router.get("/{id}")
def get_juego(id: int) -> Juego:
    # ... 
    pass

#...

```

---

## 4. Paso 3: Conectar todo en Main

El archivo `main.py` queda mucho más limpio. Su única función ahora es inicializar la app e incluir los routers que vayamos creando.

**Archivo:** `main.py`

```python
from fastapi import FastAPI
from routers.juegos import router as juegos_router # Importamos el router con un alias

app = FastAPI()

app.title = "Nuestra primera app (ahora modularizada)"

# Conectamos el router a la app principal
app.include_router(juegos_router, prefix="/juegos", tags=["juegos"])
# prefix: Agrega "/juegos" antes de todas las rutas de este router
# tags: Agrupa los endpoints en la documentación automática
```

### Ventajas de Modularizar

1. **Orden:** Cada archivo tiene una única responsabilidad.
2. **Rutas Limpias:** En `juegos.py` no hace falta repetir `/juegos/crear`, `/juegos/borrar`. Solo pones `/crear` y el `prefix` del main hace el resto.
3. **Documentación:** El parámetro `tags` agrupa visualmente los endpoints en Swagger UI.
3. **Escalabilidad:** A medida que el proyecto crezca, será más facil mantenerlo y contribuir en él.