# Clase 12. Completando el CRUD y Resumen de Base de Datos

## 1. Actualizar un Registro (PUT)

Para actualizar un registro existente, primero debemos buscarlo en la base de datos. Si existe, reemplazamos sus valores con los datos nuevos proporcionados por el usuario y guardamos los cambios.

**Archivo:** `routers/juegos.py`

```python
@router.put("/{id}", response_model=schemas.Juego)
def update_juego(id: int, juego_actualizado: schemas.JuegoUpdate, db: Session = Depends(get_db)):
    # 1. Buscamos el juego por su ID
    db_juego = db.query(models.Juego).filter(models.Juego.id == id).first()
    
    # 2. Si no existe, devolvemos un error 404
    if not db_juego:
        raise HTTPException(status_code=404, detail="Juego no encontrado")
    
    # 3. Actualizamos los atributos del modelo con los datos del schema
    db_juego.titulo = juego_actualizado.titulo
    db_juego.plataforma = juego_actualizado.plataforma
    db_juego.precio = juego_actualizado.precio
    
    # 4. Confirmamos los cambios en la base de datos y refrescamos
    db.commit()
    db.refresh(db_juego)
    
    return db_juego
```

> **Nota:** Aquí usamos un schema hipotético llamado `JuegoUpdate` (que los alumnos deberían tener en `schemas.py` desde clases anteriores) para validar los datos de entrada específicos de una actualización.

---

## 2. Eliminar un Registro (DELETE)

La operación de borrado es la más sencilla, pero requiere la misma precaución de verificar si el elemento existe antes de intentar eliminarlo.

**Archivo:** `routers/juegos.py`

```python
@router.delete("/{id}")
def delete_juego(id: int, db: Session = Depends(get_db)):
    # 1. Buscamos el juego
    db_juego = db.query(models.Juego).filter(models.Juego.id == id).first()
    
    if not db_juego:
        raise HTTPException(status_code=404, detail="Juego no encontrado")
    
    # 2. Marcamos el objeto para ser eliminado
    db.delete(db_juego)
    
    # 3. Confirmamos la transacción
    db.commit()
    
    # En un DELETE es común devolver un simple mensaje o un status 204 No Content
    return {"mensaje": f"El juego con id {id} fue eliminado exitosamente"}
```

---

## 3. Gran Resumen: Integración de Bases de Datos en FastAPI



Para que los alumnos no pierdan de vista el "bosque" por mirar los "árboles", este es el flujo completo de lo que hemos construido en estas últimas clases para conectar FastAPI con SQLite:

1. **La Configuración (`database.py`):** * Creamos el `engine` (el motor que se comunica con SQLite).
   * Creamos `SessionLocal` (la fábrica de sesiones para interactuar con el motor).
   * Definimos `Base` (la clase maestra de la que heredarán nuestras tablas).

2. **Las Tablas de la DB (`models.py`):**
   * Creamos clases que heredan de `Base`.
   * Usamos `Column`, `Integer`, `String` de SQLAlchemy para definir la estructura real de los datos en el disco duro.

3. **La Inicialización (`main.py`):**
   * Usamos `Base.metadata.create_all(bind=engine)` para que, al arrancar la app, SQLAlchemy revise si las tablas existen en el archivo `.db` y las cree si hacen falta.

4. **El Gestor de Conexiones (`get_db`):**
   * Creamos una función generadora con `yield` que crea una nueva sesión de base de datos y se asegura de cerrarla (`db.close()`) cuando la petición termina.

5. **La Inyección en los Routers (`routers/`):**
   * En cada endpoint que necesita leer o escribir, agregamos el parámetro `db: Session = Depends(get_db)`.
   * FastAPI se encarga mágicamente de ejecutar `get_db()`, darnos la conexión lista para usar y cerrarla al final.

6. **El Flujo de Datos Diario (El CRUD):**
   * **Entrada:** El usuario envía JSON $\rightarrow$ FastAPI lo valida usando **Schemas** (Pydantic).
   * **Procesamiento:** Convertimos los datos del Schema a **Modelos** (SQLAlchemy) para usar `db.add()`, `.query()`, etc. Guardamos con `db.commit()`.
   * **Salida:** FastAPI toma el Modelo (ORM), lo convierte de nuevo a Schema (gracias a `response_model`) y lo devuelve como JSON al usuario.