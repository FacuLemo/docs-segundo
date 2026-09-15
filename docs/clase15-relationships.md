
# Clase 15: Relaciones en DB con Alembic y SQLModel

### **Qué vemos:**

1. El concepto de llaves foráneas (*Foreign Keys*).
2. Aprender a definir relaciones bidireccionales usando `Relationship` de SQLModel.
3. Actualizar modelos de datos SQLModel (`Base`, `Table`, `Public`) para manejar datos anidados de forma segura.
4. Consumir estas relaciones a través de *Path Operations* (Endpoints) en FastAPI.

---

### **Fase 1: Introducción**

Supongamos que tenemos que crear un Estudio para nuestros Juegos. Por ejemplo, de Mario Bros el estudio de desarrollo sería Nintendo. Pero no tenemos que escribir en str "Nintendo" para cada registro, porque puede generar inconsistencias (por ejemplo, que algunos registros estén en minúscula y otros en mayúscula).

En estos casos lo que hay que hacer es crear una tabla nueva que almacene los estudios, y adaptar juego para que guarde una **relación** con dichos registros. De esta forma, tenemos consistencia y acceso a más datos que sólo el nombre (todos los campos que agreguemos a la tabla Estudio).

### **Fase 2: Refactorizando los Modelos**

Usando lo que teníamos anteriormente, así es como debería evolucionar el archivo de modelos:

**1. Crear la nueva entidad (`Estudio`):**

```python
from typing import List, Optional
from sqlmodel import SQLModel, Field, Relationship

class EstudioBase(SQLModel):
    nombre: str = Field(index=True)
    pais: str

class Estudio(EstudioBase, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    # Relación hacia Juego: Un estudio tiene una lista de juegos
    juegos: List["Juego"] = Relationship(back_populates="estudio")

class EstudioCreate(EstudioBase):
    pass

class EstudioPublic(EstudioBase):
    id: int

```

**2. Actualizar la entidad existente (`Juego`):**
Ahora modificamos Juego para que también guarde relación con Estudio.

```python
class JuegoBase(SQLModel):
    titulo: str = Field(index=True)
    genero: str
    score: int
    # 1. Agregamos la Foreign Key apuntando a la tabla 'estudio'
    estudio_id: Optional[int] = Field(default=None, foreign_key="estudio.id")

class Juego(JuegoBase, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    # 2. Agregamos la Relación inversa
    estudio: Optional[Estudio] = Relationship(back_populates="juegos")

class JuegoCreate(JuegoBase):
    pass

class JuegoPublic(JuegoBase):
    id: int

```

### **Fase 3: El truco de SQLModel - Modelos con Relaciones**

Ahora que está la relación hay que tener cuidado con lo siguiente: Si devolvemos el modelo `Juego` crudo, podríamos generar un bucle infinito (El juego llama al estudio, el estudio llama al juego...).

Entonces hay que crear schemas "Anidados", y usarlos en los path operations

```python
# Para cuando pides un Juego y quieres ver los datos de su Estudio
class JuegoPublicNested(JuegoPublic):
    estudio: Optional[EstudioPublic] = None

# Para cuando pides un Estudio y quieres ver todos sus Juegos
class EstudioPublicNested(EstudioPublic):
    juegos: List[JuegoPublic] = []

```
> No se olviden que en SQLModel un schema y un modelo están juntos en la misma `class`, y un modelo sólo es tabla si tiene `table=True`.

#### **Fase 4: Migraciones de DB**
Terminamos con los cambios en los modelos, es hora de correr migraciones.

Usamos alembic para correr los siguientes comandos:
Creamos la migración:
```bash
alembic revision --autogenerate -m "Nueva tabla Estudio y relación con Juego"
```
Ejecutamos la migración:
```bash
alembic upgrade head
```

> **Nota:** Si falla al ejecutar la migración, busquen manualmente el archivo de migración y busquen el método de `create_foreign_key`. Si el primer argumento está en None, cámbienlo por `'fk_juego_estudio_id`.


### **Fase 5:  Path Operations en FastAPI**
Ahora toca llevar esto a los endpoints.

1. **Crear un Estudio:** `POST /estudios/` (Usando `EstudioCreate`).
2. **Crear un Juego con Estudio:** `POST /juegos/` (Ahora en el JSON se debe enviar el `estudio_id`).
3. **Leer un Estudio con sus Juegos:**
```python
@app.get("/estudios/{estudio_id}", response_model=EstudioPublicNested)
def read_estudio(estudio_id: int, db: Session = Depends(get_db)):
    estudio = db.get(Estudio, estudio_id)
    if not estudio:
        raise HTTPException(status_code=404, detail="Estudio no encontrado")
    return estudio
```
> Presten atencion al response_model, que ahora se usa la clase Nested.


### Repaso
* Foreign Key es el dato duro (el ID en la tabla). Relationship es la herramienta de Python para navegar entre los objetos. Se deben usar Schemas Nested para no hacer bucles infinitos.

