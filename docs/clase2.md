
# Clase 2.  Crear un proyecto de FastAPI

## 1. Configuración del Entorno 

Para crear un proyecto de fastAPI, necesitaremos instalarlo dentro de un entorno virtual.

Python nos permite aislar instalaciones (dependencias) dentro de un directorio, para que sean usados únicamente por los proyectos contenidos en la misma.

### Preparación del Proyecto

1. **Crear entorno virtual:**
```bash
python3 -m venv nombre

```
> Nosotros como nombre del entorno virtual estaremos usando venv, por lo que el comando quedaría `python3 -m venv venv`


2. **Activar el entorno:**

Una vez creado el entorno virtual, tendremos que activarlo desde nuestra consola. Lo hacemos con el siguiente comando:

* **Linux / Mac:**
```bash
source nombre/bin/activate

```


* **Windows:**
```powershell
.\nombre\Scripts\activate

```

> Si estuvieran usando una consola de linux en windows (como lo puede ser git bash), tendrían que usar un comando como `source venv/Scripts/activate`



3. **Instalación de dependencias:**
* (Solo si es necesario en Linux) Instalar pip: `sudo apt install python3-pip` 


* Verificar paquetes instalados: `pip list` 


* **Instalar FastAPI (versión estándar):**
Una vez que estamos dentro del entorno virtual, ya podemos instalar los paquetes de FastAPI.

```bash
pip install fastapi[standard]

```


* Al instalarse podemos generar archivo que plasme las versiones del proyecto con el comando: `pip freeze` 


> **Tip Importante:** ¡No olvides seleccionar el intérprete correcto en VSCode! De esta manera podrás detectar errores y programar de manera más fluida.

---

## 2. Primeros Pasos: Hola Mundo


**Archivo: `main.py`**

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def home():
    return "¡Hola Mundo!"

```

> También puedes retornar HTML directamente importando `HTMLResponse` de `fastapi.responses`. 

---

## CÁPSULA. Gestión Moderna de Paquetes con `uv`

Una alternativa mucho más rápida a pip. 🔗 **Repositorio:** [github.com/astral-sh/uv](https://github.com/astral-sh/uv) 

### Instalación de `uv`

* **macOS y Linux:**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh

```


* **Windows:**
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

```



### Flujo de trabajo con `uv`

1. **Inicializar proyecto:**
```bash
uv init nombre

```


2. **Agregar FastAPI:**
```bash
uv add fastapi[standard]

```


3. **Correr el servidor de desarrollo:**
```bash
uv run fastapi dev

```

4. **Gestión de dependencias:**
* Congelar versiones (equivalente a freeze): `uv lock` 


* Sincronizar entorno: `uv sync`