
# Clase 5. Actualización y Borrado de datos

1. Introducción al Body 

Para métodos que modifican datos complejos (PUT, PATCH), dejamos de usar Query Parameters y pasamos a usar el **Body** de la petición.

* **Requisito:** Importar `Body` de FastAPI.
* **Requisito:** Importar `Optional` de `typing` (para PATCH).

## 2. Método PUT (Actualización Completa)

El método PUT se usa para reemplazar un recurso entero. Aquí usamos `Body()` para indicar que los datos viajan en el cuerpo del mensaje y no en la URL.

```python
from fastapi import Body

@app.put("/editar-juego")
def editar_juego(id: int = Body(), titulo: str = Body(), genero: str = Body()):
    for juego in juegos:
        if juego["id"] == id: #Para la coincidencia de id, pisamos los valores del diccionario
            juego["titulo"] = titulo
            juego["genero"] = genero
            return {"msg": "Juego editado correctamente"}
    return []

```

## 3. Método DELETE (Eliminación)

En este caso volvemos a requerir un **Path Parameter** para identificar qué recurso eliminar.

```python
@app.delete("/borrar-juego/{id}") #Parametro de ruta
def borrar_juego(id: int):
    for juego in juegos:
        if juego["id"] == id:
            juegos.remove(juego)  # Borramos la coincidencia del dict
            return juegos # Retorna la lista actualizada
    return {"msg": "Juego no encontrado"}

```

## 4. Método PATCH (Actualización Parcial)

El PATCH es ideal para modificar solo *algunos* campos (ej. cambiar el título pero mantener el género).

* Usamos `Optional` o `| None` para indicar que el dato puede o no venir.
* Usamos condicionales (`if titulo else...`) para actualizar solo lo que el usuario envió.

```python
from typing import Optional

@app.patch("/editar-juego")
def editar_juego_parcialmente(
    id: int, 
    titulo: Optional[str] = Body(default=None), 
    genero: str | None = None
):
    for juego in juegos:
        if juego["id"] == id:
            # Solo actualiza si 'titulo' tiene valor, sino deja el que estaba
            juego["titulo"] = titulo if titulo else juego["titulo"]
            juego["genero"] = genero if genero else juego["genero"]
            return juego
            
    return {"msg": "No se encontró el juego a editar"}

```

> De todas maneras, este método no es muy usado en el mundo de las API REST. Usualmente se suele optar por usar directamente el PUT.