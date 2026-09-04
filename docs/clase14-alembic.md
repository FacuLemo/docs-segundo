
### Relaciones, Migraciones con Alembic y SQLModel

#### 1. ¿Qué es Alembic?

Supongamos que queremos que nuestro juego tenga una relación de clave foránea a `Estudio`. No sólo hay que crear el modelo sino también el campo `estudio_id` a `Juego`. Cuando quiera hacer lo segundo, SQLAlchemy intentará crear la tabla primero, verá que ya existe y no hará nada, no agregando finalmente el nuevo campo. **Alembic** existe para tener un "control de versiones" (como Git) pero para la estructura de la base de datos. Cada cambio debe quedar registrado como un "commit", acá llamado "migración".

#### 2. Instalación y Configuración

instalamos alembic:

```bash
pip install alembic

```

Luego, inicializarlo en la carpeta del proyecto:

```bash
alembic init alembic

```



Esto creará una carpeta `/alembic` y un archivo `alembic.ini`.




#### 3. Conectar Alembic con SQLModel 

1. **En app.py o main.py**: Comentar el SQLModel.metadata.create_all()

No crearemos más la base de datos desde el ORM sino que lo haremos a través de Alembic.
Para esto comentamos la línea que creaba la base de datos. 
También borraremos la base de datos que ya tenemos para evitar conflictos posteriores (db_juegos.sql).


Hay que editar el archivo `alembic/env.py`:

2. **Importar tus modelos:** Alembic necesita conocer tus clases.
```python
from sqlmodel import SQLModel
from app.models import Juego, Estudio # Asegúrate de importar todos

```


3. **Configurar el target_metadata:**
Busca la línea `target_metadata = None` y cámbiala por:
```python
target_metadata = SQLModel.metadata

#Luego agregar a los metadata: render_as_batch=True
```


4. **Más configuraciones**
Dentro de `script.py.mako` importar sqlmodel:
```python
import sqlmodel
```

En `alembic.ini` poner la dirección de la base de datos:
```python
#Al rededor de la lína 89
sqlalchemy.url = sqlite:///./db_juegos_sqlmodel.db
```

<!-- 
En el mismo `env.py`, asegúrate de que use tu URL de base de datos  -->

#### 4. Generar la Migración Automática

Una vez que el código de los modelos tiene las relaciones (como las que planeamos antes), ejecuta:

```bash
alembic revision --autogenerate -m "crear tabla estudio y relacion con juego"

```

* **Qué hace esto:** Alembic compara tus clases de Python con la base de datos real y genera un script en `alembic/versions/` con los cambios necesarios para que en la db real haya lo mismo que en tus models.

#### 5. Aplicar los cambios

Para que los cambios se reflejen en el archivo `.db` o en el servidor:

```bash
alembic upgrade head

```

---
Recomiendo ver la clase con atención para que puedan ir siguiendo esto de las migraciones.