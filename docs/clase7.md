# Clase 7. Pydantic Avanzado & REST Clients

## 1. Validaciones con `Field` en Pydantic

En lugar de validar solo en la ruta, validamos directamente en el modelo de datos. Esto permite definir restricciones como rangos numéricos o longitudes mínimas directamente en la clase.

**Importación necesaria:**

```python
from pydantic import BaseModel, Field

```

**Ejemplo Básico:**

```python
class Juego(BaseModel):
    # Validamos que el ID sea mayor a 0
    id: int = Field(gt=0, examples=[2])
    
    # Validamos longitud mínima para textos
    titulo: str = Field(min_length=4, examples=["Mario", "sonic"])
    genero: str = Field(min_length=4, examples=["Aventura"])
    
    # Validamos que el score esté entre 0 y 100
    score: int = Field(gt=0, le=100, examples=[86])

```

---

## 2. Uso Moderno: `Annotated` y Valores por Defecto

La forma recomendada (y más compatible con otras herramientas de Python) es usar `Annotated`. Además, aquí vemos cómo asignar un **valor por defecto** (`default=50`) para que el campo sea opcional al crear el objeto.

```python
from typing import Annotated
from pydantic import BaseModel, Field

class Juego(BaseModel):
    id: Annotated[int, Field(gt=0, examples=[2])]
    
    titulo: Annotated[str, Field(min_length=4, examples=["Mario"])]
    
    genero: Annotated[str, Field(min_length=4, examples=["Aventura"])]
    
    # Al tener un default, si el usuario no envía score, será 50
    score: Annotated[int, Field(gt=0, le=100, examples=[86], default=50)]

```

---

## 3. Refactorización de Endpoints (HTTPException)

Limpiamos el código eliminando conversiones manuales (`model_dump`) y usando excepciones estándar de HTTP para errores (como el 404).

**Cambios Clave:**

1. **Tipado de retorno:** `-> Juego` (FastAPI se encarga de convertir el objeto a JSON).
2. **Manejo de Errores:** Usamos `raise HTTPException` en lugar de devolver listas vacías o diccionarios de error manuales.

```python
from fastapi import FastAPI, HTTPException

@app.get("/juego/{id}")
def juego(id: int) -> Juego:
    for juego in juegos:
        if juego["id"] == id:
            return juego  # Retornamos el objeto directo, sin .model_dump() ni dict()
            
    # Si termina el bucle y no lo encuentra, lanzamos el error 404
    raise HTTPException(status_code=404, detail="Juego no encontrado")

```

> **Nota:** Al usar Pydantic en el retorno, ya no hace falta usar `.model_dump()` en el POST o PUT si defines que tu función retorna el modelo `Juego`. FastAPI lo serializa automáticamente.

---

## 4. Cápsula: Clientes REST

Es fundamental probar nuestra API con herramientas profesionales en lugar de solo el navegador.

* **Thunderclient:** Extensión ligera para VS Code.
* **Postman:** El estándar de la industria (muy completo pero pesado).
* **Bruno (Recomendado):**
    * Open Source.
    * Guarda las colecciones directamente en tu carpeta del proyecto (git-friendly).
    * Tiene una interfaz limpia y rápida.

> Consulten el video de la cápsula para aprender a usar estos clientes.