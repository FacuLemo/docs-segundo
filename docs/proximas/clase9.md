# Clase 9. Modularización con APIRouter

## 1. Nueva Estructura del Proyecto

Hasta ahora teníamos todo en un solo archivo. Para trabajar profesionalmente, dividimos el código en carpetas y archivos según su responsabilidad.

**Estructura recomendada:**

```text
mi_proyecto/
├── main.py           # Punto de entrada (solo configuración global)
├── schemas.py        # Modelos de Pydantic (Antes estaban en main)
└── routers/          # Carpeta para las rutas
    └── juegos.py     # Lógica de los endpoints de juegos

```

> **Nota:** En esta clase renombramos el concepto de "Modelos" (Pydantic) a **Schemas** para no confundirlos con los "Modelos" de base de datos (ORM) en el futuro.

---

## 2. Paso 1: Mover los Schemas

Cortamos las clases `BaseModel` (`Juego`, `JuegoUpdate`) de `main.py` y las pegamos en `schemas.py`.

**Archivo:** `schemas.py`

```python
from pydantic import BaseModel, Field
from typing import Annotated

class Juego(BaseModel):
    id: Annotated[int, Field(gt=0)]
    titulo: str
    # ... resto de campos

```

---

## 3. Paso 2: Crear el Router

En lugar de usar `app = FastAPI()`, usamos `APIRouter` dentro de nuestros módulos. Esto nos permite definir rutas que luego "conectaremos" a la app principal.

**Archivo:** `routers/juegos.py`

```python
from fastapi import APIRouter, HTTPException
from ..schemas import Juego, JuegoUpdate # Importamos desde la carpeta superior

# Creamos la instancia del router
router = APIRouter()

# Base de datos simulada (ahora vive aquí o en un servicio aparte)
juegos = [...]

# Cambiamos @app por @router
# Nota: Si usamos prefix en main, aquí la ruta puede ser solo "/"
@router.get("/") 
async def get_juegos() -> list[Juego]:
    return juegos

@router.get("/{id}")
def get_juego(id: int) -> Juego:
    # ... lógica de búsqueda
    pass

```

---

## 4. Paso 3: Conectar todo en Main

El archivo `main.py` queda mucho más limpio. Su única función ahora es inicializar la app e incluir los routers que vayamos creando.

**Archivo:** `main.py`

```python
from fastapi import FastAPI
from routers.juegos import router as juegos_router # Importamos el router

app = FastAPI()

app.title = "Nuestra primera app"

# Conectamos el router a la app principal
# prefix: Agrega "/juegos" antes de todas las rutas de este router
# tags: Agrupa los endpoints en la documentación automática
app.include_router(juegos_router, prefix="/juegos", tags=["juegos"])

```

### Ventajas de Modularizar

1. **Orden:** Cada archivo tiene una única responsabilidad.
2. **Rutas Limpias:** En `juegos.py` no hace falta repetir `/juegos/crear`, `/juegos/borrar`. Solo pones `/crear` y el `prefix` del main hace el resto.
3. **Documentación:** El parámetro `tags` agrupa visualmente los endpoints en Swagger UI.
3. **Escalabilidad:** A medida que el proyecto crezca, será más facil mantenerlo y contribuir en él.