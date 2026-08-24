# Clase 13. Migración a SQLModel: Unificando Modelos y Schemas

## 1. El Problema de la Duplicación: ¿Por qué SQLModel?

En las clases anteriores logramos conectar nuestra API a una base de datos. Sin embargo, seguramente notaste algo molesto: **tuvimos que escribir casi el mismo código dos veces**. 

Teníamos un `schemas.Juego` (Pydantic) para validar los datos de la API y un `models.Juego` (SQLAlchemy) para guardar en la base de datos. Si mañana queremos agregar el campo "fecha_lanzamiento", tendríamos que modificar ambos archivos. Esto es propenso a errores y difícil de mantener.

Aquí es donde entra **SQLModel**. Es una librería creada por el mismo autor de FastAPI (Sebastián Ramírez) diseñada para resolver exactamente este problema. 



**¿Qué hace SQLModel?**
Combina el poder de SQLAlchemy y Pydantic en una sola clase. Defines tu modelo una sola vez y sirve tanto para validar los datos de la API como para crear la tabla en la base de datos.

---

## 2. Paso 1: Instalación y Actualización del Motor

Primero, debes instalar la librería en tu entorno virtual: `pip install sqlmodel`.

Luego, vamos a simplificar nuestro archivo de configuración. Notarás que ahora importamos casi todo directamente desde `sqlmodel`.

**Archivo:** `database.py`

```python
from sqlmodel import create_engine, Session, SQLModel

sqlite_url = "sqlite:///./sql_app.db"

# El motor se crea igual, pero lo importamos de sqlmodel
engine = create_engine(sqlite_url, connect_args={"check_same_thread": False})

# Actualizamos nuestra dependencia para usar la Sesión de SQLModel
def get_db():
    with Session(engine) as session:
        yield session
```

---

## 3. Paso 2: La Magia de Unificar (Adiós schemas.py)

Este es el cambio más importante. Ya no necesitamos la separación estricta entre `models.py` y `schemas.py` para nuestras tablas básicas. Vamos a refactorizar nuestro modelo.

**Archivo:** `models.py`

```python
from sqlmodel import SQLModel, Field

# 1. EL MODELO BASE: Contiene los campos comunes a todos; sin crear una tabla
class JuegoBase(SQLModel):
    titulo: str = Field(index=True)
    plataforma: str
    precio: float

# 2. EL MODELO DE TABLA: Hereda de Base, agrega el ID y se guarda en la DB
class Juego(JuegoBase, table=True): #Table True indica que creará una tabla en la db
    id: int | None = Field(default=None, primary_key=True)

# 3. EL MODELO DE CREACIÓN: Lo que el usuario envía (No tiene ID)
class JuegoCreate(JuegoBase):
    pass # Hereda todo de JuegoBase tal cual

# 4. EL MODELO DE LECTURA (Público): Lo que devolvemos al usuario
class JuegoPublic(JuegoBase):
    id: int 
```

> **Nota:** Puedes eliminar tu archivo `schemas.py` original (o dejarlo solo para esquemas muy específicos que no van a la base de datos, como un esquema para cambiar contraseñas).

---

## 4. Paso 3: Inicializar la Base de Datos

En nuestro punto de entrada, el código para crear las tablas es casi idéntico, pero usando el objeto `SQLModel`.

**Archivo:** `main.py`

```python
from fastapi import FastAPI
from sqlmodel import SQLModel
from database import engine
import models # Importamos para que SQLModel detecte las tablas
from routers.juegos import router as juegos_router

# Crea las tablas usando SQLModel en lugar de Base de SQLAlchemy
SQLModel.metadata.create_all(engine)

app = FastAPI()
app.include_router(juegos_router, prefix="/juegos", tags=["juegos"])
```

---

## 5. Paso 4: Refactorizando el Router con SQLModel

Las consultas cambian ligeramente a una sintaxis más moderna e intuitiva usando `select`.

**Archivo:** `routers/juegos.py`

```python
from fastapi import APIRouter, HTTPException, Depends
from sqlmodel import Session, select
from ..database import get_db
from ..models import Juego # Ahora importamos el modelo unificado

router = APIRouter()

# --- GET (Leer todos) ---
@router.get("/", response_model=list[Juego])
def get_juegos(db: Session = Depends(get_db)):
    # Usamos select() de SQLModel
    juegos = db.exec(select(Juego)).all()
    return juegos

# --- POST (Crear) ---
@router.post("/", response_model=Juego)
def create_juego(juego: Juego, db: Session = Depends(get_db)):
    # Convertimos el modelo de creación al modelo de base de datos
    juego_db = Juego.model_validate(juego_in)
    # El modelo entrante validado ES el modelo de DB; no hay que pisar nada
    db.add(juego_db)
    db.commit()
    db.refresh(juego)
    return juego_db

# --- GET por ID ---
@router.get("/{id}", response_model=Juego)
def get_juego(id: int, db: Session = Depends(get_db)):
    # db.get() es una forma súper rápida de buscar por clave primaria
    juego = db.get(Juego, id)
    if not juego:
        raise HTTPException(status_code=404, detail="Juego no encontrado")
    return juego
```

### Ventajas de esta Migración

1. **Código DRY (Don't Repeat Yourself):** Escribimos la estructura del dato una sola vez.
2. **Soporte Nativo:** Al ser del mismo creador que FastAPI, la integración con el editor de código (autocompletado) y la validación es perfecta.
3. **Sintaxis Moderna:** Operaciones como `db.get()` o `select()` son más limpias que la sintaxis clásica de SQLAlchemy.


