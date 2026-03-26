# Clase 8. Documentación y Response Model


## 1. Documentar Errores (Responses)

Por defecto, FastAPI solo documenta la respuesta exitosa (Código 200). Para que en la documentación aparezca que nuestra API puede devolver un error **404 Not Found**, debemos indicarlo explícitamente en el decorador.

Esto hace que Swagger muestre un ejemplo del JSON de error.

```python
from fastapi import FastAPI, HTTPException

# Definimos el diccionario de respuestas adicionales
@app.get("/juego/{id}", responses={
    404: {
        "description": "Juego no encontrado",
        "content": {
            "application/json": {
                "example": {"detail": "Juego no encontrado"}
            }
        },
    }
})
def obtener_juego(id: int):
    # nuestra lógica...
    pass

```

**¿Qué hace este código?** 

* **responeses={...}** En el decorador, documenta las posibles respuestas: (además del 200 OK).
* **404:** Indica y documenta el código de estado HTTP. Dentro se abre otro dict:
* **description:** Texto explicativo que aparecerá en Swagger.
* **content -> -> example:** Muestra un JSON de ejemplo en la documentación para que el frontend sepa qué esperar si falla.

Con esto estamos documentando cómo llegan los datos en caso que no coincida ningún id.

---

## 2. El `response_model`

En las clases anteriores usamos el tipado de Python con la flechita (`-> list[Juego]`) para validar la respuesta. Existe una forma alternativa (y muy común en versiones anteriores de FastAPI) que es usar el parámetro `response_model` dentro del decorador.

Ambas formas funcionan, y pueden coexistir.

```python
# Forma A: Tipado moderno (Python 3.10+)
@app.get("/juegos")
def get_juegos() -> list[Juego]:
    return juegos

# Forma B: Usando el parámetro de FastAPI (Alternativa)
@app.get("/juegos", response_model=list[Juego])
def get_juegos():
    return juegos

```

> **Nota:** `response_model` tiene prioridad sobre el tipo de retorno de la función. Se usa mucho cuando quieres filtrar datos (por ejemplo, devolver un objeto que tiene contraseña en la base de datos, pero usar un `response_model` que NO incluya el campo contraseña para enviarlo al usuario).