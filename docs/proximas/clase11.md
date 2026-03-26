Siguiendo el orden lógico del curso, una vez que tenemos configurada la base de datos y los modelos, la **Clase 11** debe enfocarse en cómo conectar esos modelos con los endpoints que ya teníamos.

Aquí aprenderemos a usar la **Inyección de Dependencias** para realizar operaciones CRUD reales.

---

# Clase 11. Operaciones CRUD con SQLAlchemy

## 1. El concepto de Session y Dependencia

Para que nuestros endpoints puedan escribir o leer de SQLite, necesitan una sesión de base de datos activa. En FastAPI, esto se logra pasando la sesión como un parámetro en la función del endpoint.

**Importante:** Usaremos la función `get_db` que creamos en la clase anterior para asegurar que cada petición abra y cierre su propia conexión.

---

## 2. Paso 1: Actualizar los Imports del Router

Ahora nuestro router necesita interactuar con la base de datos, los modelos de SQLAlchemy.

**Archivo:** `routers/juegos.py`

```python
from fastapi import APIRouter, HTTPException, Depends
from sqlalchemy.orm import Session
from database import get_db
from import models, schemas

router = APIRouter()

```

---

## 3. Paso 2: Endpoint para Leer Datos (GET)

En lugar de devolver una lista estática, consultaremos la tabla `juegos` a través del ORM.

**Archivo:** `routers/juegos.py`

```python
@router.get("/", response_model=list[schemas.Juego])
def get_juegos(db: Session = Depends(get_db)):
    # .query(models.Juego) accede a la tabla
    # .all() trae todos los registros
    return db.query(models.Juego).all()

```

> **Nota:** `Depends(get_db)` se encarga de inyectar la sesión de la base de datos automáticamente en la variable `db`.

---

## 4. Paso 3: Endpoint para Crear Datos (POST)

Aquí es donde vemos la diferencia real entre el **Schema** (lo que recibimos) y el **Modelo** (lo que guardamos).

**Archivo:** `routers/juegos.py`

```python
@router.post("/", response_model=schemas.Juego)
def create_juego(juego: schemas.Juego, db: Session = Depends(get_db)):
    # 1. Creamos la instancia del Modelo con los datos del Schema
    db_juego = models.Juego(
        titulo=juego.titulo,
        plataforma=juego.plataforma,
        precio=juego.precio
    )
    
    # 2. Agregamos y confirmamos en la DB
    db.add(db_juego)
    db.commit()
    
    # 3. Refrescamos para obtener el ID generado automáticamente
    db.refresh(db_juego)
    
    return db_juego

```

---

## 5. Paso 4: Buscar por ID y Manejo de Errores

Para buscar un elemento específico, usamos el método `.filter()`.

```python
@router.get("/{id}", response_model=schemas.Juego)
def get_juego(id: int, db: Session = Depends(get_db)):
    db_juego = db.query(models.Juego).filter(models.Juego.id == id).first()
    
    if not db_juego:
        raise HTTPException(status_code=404, detail="Juego no encontrado")
        
    return db_juego

```

---

### Resumen de la Clase

1. **Depends(get_db):** Es la herramienta que nos da acceso a la base de datos dentro de cada función.
2. **db.commit():** Es obligatorio para guardar cualquier cambio (Crear, Actualizar, Borrar).
3. **Manejo de flujo:** Recibimos un `Schema`, lo convertimos a `Modelo`, lo guardamos y FastAPI lo vuelve a convertir a `Schema` automáticamente al enviarlo como respuesta gracias a `response_model`.
