¡Excelente elección! CORS es uno de esos temas que suele dar muchos dolores de cabeza a los estudiantes la primera vez que intentan conectar su backend con una interfaz de usuario. 

Aquí tienes la propuesta para la **Clase 14**, manteniendo el estilo estructurado y directo.

---

# Clase 14. Configuración de CORS (Conectando el Frontend)

## 1. ¿Qué es CORS y por qué mi navegador me odia?

Hasta ahora, hemos probado nuestra API usando la documentación automática de Swagger (en `http://localhost:8000/docs`). Pero en el mundo real, tu API será consumida por una aplicación frontend separada (por ejemplo, una interfaz web construida con JavaScript, componentes JSX, o un simple HTML).

Si intentas hacer una petición `fetch` desde tu frontend (digamos, corriendo en `http://localhost:3000`) hacia tu backend en FastAPI (`http://localhost:8000`), **el navegador bloqueará la petición y mostrará un error en rojo en la consola.**

Esto no es un error de tu código, es una medida de seguridad de los navegadores llamada **CORS (Cross-Origin Resource Sharing)**. Por defecto, un navegador no permite que un sitio web en el "Origen A" pida datos a un servidor en el "Origen B" a menos que el servidor B dé permiso explícito.



---

## 2. Paso 1: Importar el Middleware de CORS

Para solucionar esto, debemos decirle a FastAPI qué orígenes (qué frontends) tienen permiso para leer nuestra base de datos de juegos. Usaremos un "Middleware", que es un bloque de código que intercepta todas las peticiones antes de que lleguen a nuestros routers.

**Archivo:** `main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware # 1. Importamos el middleware
from sqlmodel import SQLModel
from database import engine
import models
from routers.juegos import router as juegos_router

SQLModel.metadata.create_all(engine)

app = FastAPI()

# ... (El código de CORS va aquí, ver Paso 2) ...

app.include_router(juegos_router, prefix="/juegos", tags=["juegos"])
```

---

## 3. Paso 2: Configurar los Orígenes Permitidos

Ahora, agregamos la configuración a nuestra instancia de `app`.

**Archivo:** `main.py`

```python
# 2. Definimos una lista con las URLs que tienen permiso de acceder a la API
origenes_permitidos = [
    "http://localhost:3000", # Típico puerto para proyectos con React/JSX
    "http://localhost:5500", # Típico puerto para Live Server (HTML/JS básico)
    # "https://midominio.com", # Así se vería en producción
]

# 3. Agregamos el middleware a la aplicación
app.add_middleware(
    CORSMiddleware,
    allow_origins=origenes_permitidos, # Orígenes que pueden hacer peticiones
    allow_credentials=True,            # Permite el envío de cookies/tokens
    allow_methods=["*"],               # Métodos permitidos (GET, POST, PUT, DELETE, etc.)
    allow_headers=["*"],               # Cabeceras permitidas
)
```

> **Nota de Seguridad:** Usar `["*"]` significa "permitir todo". Es muy útil en desarrollo, pero en producción es una buena práctica especificar exactamente qué métodos (ej. `["GET", "POST"]`) y qué orígenes están permitidos para evitar vulnerabilidades.

---

## 4. Paso 3: Probando la Conexión desde el Frontend

Para demostrar que funciona, los alumnos ahora pueden crear un pequeño script en el frontend. Como ya configuramos CORS, este código ejecutado desde el navegador ahora sí logrará obtener los datos de SQLite:

**Ejemplo de consumo (JavaScript/JSX en el frontend):**

```javascript
// Este código iría en el proyecto frontend de tus alumnos
const obtenerJuegos = async () => {
    try {
        const respuesta = await fetch("http://localhost:8000/juegos/");
        const datos = await respuesta.json();
        console.log("Juegos obtenidos de la API:", datos);
        // Aquí usarían los datos para pintar la interfaz
    } catch (error) {
        console.error("Error de CORS o de red:", error);
    }
};

obtenerJuegos();
```

### Resumen de la Clase

1. **El Problema:** Los navegadores bloquean peticiones entre diferentes puertos o dominios por seguridad (CORS).
2. **La Solución:** Usar `CORSMiddleware` en FastAPI para incluir cabeceras HTTP especiales que le dicen al navegador "confía en este origen, es nuestro amigo".
3. **El Resultado:** Tu API de juegos ahora está lista para ser consumida y mostrada en una interfaz visual externa.