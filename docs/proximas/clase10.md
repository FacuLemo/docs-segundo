# Clase 10. Persistencia de Datos con SQLite y SQLAlchemy

## 1. Introducción a la Capa de Datos

Hasta ahora, al reiniciar el servidor de FastAPI, todos nuestros juegos desaparecían porque se guardaban en una lista de Python. Para solucionar esto, integraremos **SQLite**, una base de datos ligera que guarda la información en un archivo local `.db`.

Para interactuar con la base de datos de forma profesional, utilizaremos **SQLAlchemy**, que es un **ORM** (Object Relational Mapper). Esto nos permite trabajar con tablas de bases de datos como si fueran clases de Python.

Lo instalaremos con
```bash
pip install sqlalchemy
```

**Nueva Estructura del Proyecto:**

```text
mi_proyecto/
├── main.py
├── schemas.py        # Validaciones de Pydantic (Entrada/Salida API)
├── database.py       # NUEVO: Configuración de la conexión
├── models/           # NUEVO: Modelos de SQLAlchemy (Tablas de DB)
|   └── juegos_models.py
└── routers/
    └── juegos_routers.py

```

---

## 2. Paso 1: Configurar la Conexión

Creamos el archivo `database.py`. Aquí definiremos cómo se conecta FastAPI a SQLite.

**Archivo:** `database.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

# 1. Definimos la URL de la base de datos (se creará un archivo llamado sql_app.db)
sqlite_url = "sqlite:///./sql_app.db"

# 2. Creamos el motor (engine)
# check_same_thread=False es necesario solo para SQLite
engine = create_engine(sqlite_url, connect_args={"check_same_thread": False})

# 3. Creamos una sesión local para interactuar con la DB
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# 4. Clase base para que nuestros modelos hereden de ella
Base = declarative_base()

```

---

## 3. Paso 2: Crear el Modelo de Base de Datos

Es vital no confundir los **Schemas** (Pydantic) con los **Modelos** (SQLAlchemy). Los modelos definen cómo se ve la tabla en el disco duro.

**Archivo:** `models.py`

```python
from database import Base
from sqlalchemy import Column, Integer, String, Float

class Juego(Base):
    __tablename__ = "juegos" # Nombre de la tabla en SQLite

    id = Column(Integer, primary_key=True, index=True)
    titulo = Column(String)
    plataforma = Column(String)
    precio = Column(Float)

```

---

## 4. Paso 3: El Proveedor de Sesiones (get_db)

Para que cada petición a nuestra API tenga su propia conexión a la base de datos y se cierre al terminar, usamos una **dependencia**.

**Añadir a:** `database.py` (o en un archivo de utilidades)

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close() # Nos aseguramos de cerrar la conexión siempre

```

---

## 5. Paso 4: Inicialización en Main

Debemos decirle a FastAPI que cree las tablas en el archivo SQLite al iniciar la aplicación si estas no existen.

**Archivo:** `main.py`

```python
from fastapi import FastAPI
from database import engine, Base
from routers.juegos import router as juegos_router

# Crea todas las tablas definidas en models.py
Base.metadata.create_all(bind=engine)

app = FastAPI()
app.include_router(juegos_router, prefix="/juegos", tags=["juegos"])

```

---

### Conceptos Clave de esta Clase

1. **ORM (SQLAlchemy):** Nos permite evitar escribir código SQL manualmente y usar objetos Python en su lugar.
2. **Modelos vs Schemas:** El **Modelo** es para la base de datos; el **Schema** es para la validación de los datos que envía el usuario (Pydantic).
3. **SessionLocal:** Es la "fábrica" de conexiones que usaremos para realizar operaciones CRUD.

¿Te gustaría que preparemos la **Clase 11** para modificar los endpoints del router y que ya empiecen a guardar y leer datos reales de la base de datos?