# Clase 4. Documentación automática; Lectura y Creación de datos

En la clase de hoy vamos a ver:

* 
**Documentación Automática:** FastAPI al servirlo a través de uvicorn (en `localhost:8000/`) genera documentación interactiva en `.../docs` y `.../redoc`. El primero es el que vamos a estar utilizando, ya que utiliza un formato Swagger y nos da acceso a varias herramientas para probar nuestra api.


* 
**Simulación de Base de Datos:** Con el objetivo de crear un CRUD, vamos a simular una db usando una lista de diccionarios en memoria. A esta lista de diccionarios la vamos a estar modificando para la consulta, creación, actualización y borrado de los datos en la misma.


* 
**Parámetros:** Al momento de obtener un registro de nuestra db simulada, lo encontraremos por su ID. Para obtener ese id, lo vamos a pasar como parametro por la ruta. Luego, para modificar un dato existente de ese registro, estaremos consultado el parametro query. Aprenderemos la diferencia entre estos dos.



## 2. Configuración Inicial y "Base de Datos"

Veremos un pantallazo de lo que crea FastAPI en `localhost:8000/docs`.
Podremos modificar metadatos de nuestra api si definimos atributos a nuestro objeto app:

```python
from fastapi import FastAPI

app = FastAPI()

#Metadatos de la documentación
app.title = "Mi aplicación de juegos"
app.summary = "Resumen de la app"
app.version = "1.0.0"
```

Luego, podemos empezar a esquemar nuestra db simulada:


```python
#Simulación de Base de Datos
juegos = [
    {"id": 1, "titulo": "Super Mario Bros.", "genero": "Plataformas"},
    {"id": 2, "titulo": "World of Warcraft", "genero": "MMORPG"},
    {"id": 3, "titulo": "Sonic the hedgehog", "genero": "Plataformas"},
]

```

## 3. Métodos GET (Lectura)

### Obtener todos los juegos

Este endpoint simple devuelve la lista completa de juegos.

```python
@app.get("/juegos")
async def get_juegos():
    return juegos  

```

!!! note "¿`async def` o `def` solito?"
    FastAPI implementa la posibilidad de hacer funciones asíncronas, optimizando muchísimo el rendimiento de nuestra api cuando mucha gente utiliza endpoints al mismo tiempo. Por ahora, no lo vamos a estar utilizando, pero no hay problema si ahora las definís como `async def`.



### Obtener un juego por ID

Ahora, al obtener un juego específico, es cuando hacemos uso del **Path Parameter**.
Definimos una variable cuando declaramos la ruta del endpoint, y luego lo definimos como parámetro de la función.

Cuando obtuvimos ese id, simplemente buscamos una coincidencia en nuestra "db" y devolvemos el diccionario obtenido.
```python
@app.get("/juego/{id}")  # Parámetro de Ruta
def juego(id: int):
    for juego in juegos:
        if juego["id"] == id: # Buscamos coincidencia de id en nuestros registros
           return juego   
    return []                  # Si no encontró nada no devuelve nada

```

## 4. Método POST (Creando un registro)

Antes necesitabamos saber un ID para obtener un registro existente. Ahora, que vamos a crear un nuevo, necesitamos enviar los datos que componen ese registro (`id`, `título`, `género`).

Vamos a crear un endpoint de manera que podamos crear un registro desde la URL de la siguiente manera:
```
127.0.0.1:8000/agregar-juego/?id=X&titulo=ejemplo&genero=ejemplo
```

Para definir un Query Param, debemos definir cómo lo vamos a recibir desde los parámetros de función del path operation, pero sin especificarlos en la ruta del endpoint:

```python
@app.post("/agregar-juego")  # Los parámetros son de tipo Query 
def agregar_juego(id: int, titulo: str, genero: str): # Definidos acá
    juego_nuevo = {"id": id, "titulo": titulo, "genero": genero} 
    juegos.append(juego_nuevo) # Creamos el registro y lo guardamos en la db simulada
    return juegos 

```

!!! note "¿Le pasamos el id?"
    Normalmente en una base de datos real, el ID es un campo autoincremental y nunca se crea intencionalmente o se modifica. En nuestro caso, al estar simulando una a través de una lista y sin una estructura definida, se lo tenemos que pasar para que no quede vacío el campo (que en nuestro caso, es una clave-valor del diccionario).

---
