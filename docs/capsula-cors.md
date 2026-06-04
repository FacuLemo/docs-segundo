# Cápsula CORS. Conectando a un Frontend

## 1. ¿Qué es CORS ?

Hasta ahora, hemos probado nuestra API usando la documentación automática de Swagger (en `http://localhost:8000/docs`). Pero en aplicaciones reales, tu API será consumida por una aplicación frontend separada (por ejemplo, una interfaz web construida con JavaScript, componentes JSX, o un simple HTML).

Si intentas hacer una petición `fetch` desde tu frontend (digamos, corriendo en `http://localhost:3500`) hacia tu backend en FastAPI (`http://localhost:8000`), **el navegador bloqueará la petición y mostrará un error en rojo en la consola.**

Esto no es un error de tu código, es una medida de seguridad de los navegadores llamada **CORS (Cross-Origin Resource Sharing)**. Por defecto, un navegador no permite que un sitio web en el "Origen A" pida datos a un servidor en el "Origen B" a menos que el servidor B dé permiso explícito.


## 2. Uso de CORS

Para solucionar esto, debemos decirle a FastAPI qué orígenes (qué frontends) tienen permiso para leer y usar nuestros path operations. Usaremos un "Middleware", que es un bloque de código que intercepta todas las peticiones antes de que lleguen a nuestros routers.

**Archivo:** `main.py`

```python
#...
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Agregamos el middleware a la aplicación
app.add_middleware(
    CORSMiddleware,
    allow_origins=[ #Qué origenes permitimos:
        "http://localhost:5500", # Una app de Live Server, sino react en puerto 3000
        # "https://misuperfrontend.com", # Así se vería en producción
        # "*", con la wildcard (asterisco) permitimos absolutamente todas las fuentes
    ], 
    allow_credentials=True,            # Permite el envío de cookies/tokens en los headers
    allow_methods=["*"],               # Métodos permitidos (GET, POST, PUT, DELETE, etc.)
    allow_headers=["*"],               # headers permitidos
)

app.include_router(juegos_router, prefix="/juegos", tags=["juegos"])
#...
```

> El middleware es código que se ejecuta después de una petición y antes de nuestro path operation. Asimismo, se ejecuta después del return de nuestro path operation pero antes de mandar la response. Entonces, el flujo sería: Request -> Middleware -> Path Operation -> Middleware -> Response

## Probando el funcionamiento

Ahora que ya configuramos CORS, el fetch de javascript ejecutado desde un live server ahora sí logrará obtener los datos de SQLite:

**Ejemplo de consumo (JavaScript/JSX en el frontend):**

```javascript
// Este código iría en el proyecto frontend de tus alumnos
const obtenerJuegos = async () => {
    try {
        const respuesta = await fetch("http://localhost:8000/juegos/");
        const datos = await respuesta.json();
        console.log("Juegos obtenidos de la API:", datos);
        // Aquí usarían los datos 
    } catch (error) {
        console.error("Error de CORS o de red:", error);
    }
};

obtenerJuegos();
```

### TL;DR

1. **El Problema:** Los navegadores bloquean peticiones entre diferentes puertos o dominios por seguridad (CORS).
2. **La Solución:** Usar `CORSMiddleware` en FastAPI para incluir cabeceras HTTP especiales que le dicen al navegador "confiá en este origen".
3. **El Resultado:** Tu API ahora está lista para ser consumida y mostrada en una interfaz visual externa.