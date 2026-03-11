# Flujo CI/CD completo con GitLab y Docker

## 1. Objetivo

Vamos a crear un **flujo de CI/CD** que haga lo siguiente:

1. Ejecutar pruebas automáticas con **pytest**.
2. Construir una **imagen Docker** de nuestra aplicación.
3. Subir la imagen al **GitLab Container Registry** automáticamente cuando se haga push a la rama `main`.

Esto asegura que:

* Tu código siempre pasa las pruebas antes de construir la imagen.
* Todas las imágenes están versionadas y disponibles en tu registro privado.
* La CI/CD se ejecuta de forma automatizada en cada commit o merge request.

---

## 2. Requisitos previos

Antes de empezar, necesitas:

1. **Cuenta GitLab** y un **repositorio** activo.
2. **Activar el Container Registry** en tu proyecto:

   * Ve a **Settings → Packages & Registries → Container Registry**.
   * Debe estar activado.
3. Tener **Docker instalado localmente** si quieres probar builds antes de push.
4. **Token de acceso personal** de GitLab con permisos `read_registry` y `write_registry` (para login automático en CI/CD).

---

## 3. Estructura del proyecto

Organizamos el proyecto así:

```
mi-proyecto/
├─ app/
│  └─ main.py        # Código de la app
├─ tests/
│  └─ test_main.py   # Pruebas unitarias
├─ requirements.txt  # Dependencias de Python
├─ Dockerfile        # Construcción de la imagen Docker
└─ .gitlab-ci.yml    # Pipeline CI/CD
```

Explicación:

* **`app/`**: contiene tu aplicación principal.
* **`tests/`**: contiene pruebas unitarias o de integración.
* **`requirements.txt`**: lista de paquetes Python necesarios.
* **`Dockerfile`**: instrucción de construcción de la imagen.
* **`.gitlab-ci.yml`**: pipeline de CI/CD para GitLab.

---

## 4. Código de la aplicación

Creamos una app **muy simple** para poder probar todo:

**`app/main.py`**:

```python
# app/main.py

def suma(a, b):
    """Función simple para sumar dos números"""
    return a + b

def main():
    """Función principal que imprime un ejemplo de suma"""
    print("Ejemplo de suma: 2 + 3 =", suma(2, 3))

if __name__ == "__main__":
    main()
```

**Explicación:**

* `suma(a, b)`: función que devuelve la suma de dos números.
* `main()`: imprime un ejemplo en consola.
* `if __name__ == "__main__"`: asegura que la función `main()` solo se ejecute si ejecutamos `python app/main.py`.

---

## 5. Pruebas con pytest

Creamos un archivo de prueba para nuestra función `suma`:

**`tests/test_main.py`**:

```python
# tests/test_main.py
from app.main import suma

def test_suma():
    # Prueba suma de enteros positivos
    assert suma(2, 3) == 5
    # Prueba suma con cero
    assert suma(0, 0) == 0
    # Prueba suma con números negativos
    assert suma(-1, 1) == 0
```

**Explicación:**

* `pytest` busca funciones que empiecen con `test_`.
* `assert` verifica que el resultado sea el esperado.
* Esto asegura que tu función `suma` se comporta correctamente antes de construir la imagen Docker.

---

## 6. Dependencias de Python

**`requirements.txt`**:

```
pytest
```

* Solo necesitamos `pytest` para las pruebas.
* Si tu app necesitara más librerías, las agregas aquí.
* GitLab CI/CD instalará estas dependencias antes de ejecutar tests.

---

## 7. Dockerfile

Creamos el Dockerfile para construir la imagen:

**`Dockerfile`**:

```dockerfile
# Dockerfile
FROM python:3.12-slim

# Crear directorio de trabajo
WORKDIR /app

# Copiar y instalar dependencias
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar todo el código
COPY . .

# Comando por defecto al iniciar el contenedor
CMD ["python", "app/main.py"]
```

**Explicación paso a paso:**

1. `FROM python:3.12-slim`: base ligera con Python 3.12.
2. `WORKDIR /app`: define el directorio de trabajo dentro del contenedor.
3. `COPY requirements.txt .`: copia el archivo de dependencias.
4. `RUN pip install --no-cache-dir -r requirements.txt`: instala dependencias.
5. `COPY . .`: copia todo el código al contenedor.
6. `CMD ["python", "app/main.py"]`: comando que se ejecuta al iniciar el contenedor.

---

## 8. Pipeline GitLab CI/CD

**`.gitlab-ci.yml`**:

```yaml
stages:
  - test
  - build
  - deploy

variables:
  # Nombre de la imagen en el GitLab Container Registry
  IMAGE_NAME: "$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG"

# Job de pruebas
test:
  stage: test
  image: python:3.12
  before_script:
    - pip install -r requirements.txt
  script:
    - pytest tests/
  only:
    - branches
    - merge_requests

# Job de build Docker
build:
  stage: build
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t $IMAGE_NAME .
  only:
    - branches
    - merge_requests

# Job de deploy Docker
deploy:
  stage: deploy
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker push $IMAGE_NAME
  only:
    - main
```

**Explicación paso a paso:**

1. **Variables:**

   * `IMAGE_NAME` genera el nombre de la imagen usando la variable de GitLab `CI_REGISTRY_IMAGE` y la rama actual (`CI_COMMIT_REF_SLUG`).

2. **Job `test`:**

   * Imagen: `python:3.12`.
   * Instala dependencias y ejecuta `pytest`.
   * Solo se ejecuta en ramas o merge requests.

3. **Job `build`:**

   * Imagen: `docker:24.0.5`.
   * Usa `docker:dind` (Docker-in-Docker) para construir la imagen dentro del runner.
   * Hace login en el registro de GitLab y construye la imagen.
   * Se ejecuta en todas las ramas y merge requests.

4. **Job `deploy`:**

   * Solo se ejecuta en la rama `main`.
   * Hace push de la imagen al GitLab Container Registry.
   * Permite versionar automáticamente la imagen con la rama o hash del commit.

---

## 9. Configuración de variables en GitLab

Ve a **Settings → CI/CD → Variables** y agrega:

| Variable               | Valor                                                |
| ---------------------- | ---------------------------------------------------- |
| `CI_REGISTRY_USER`     | tu usuario de GitLab                                 |
| `CI_REGISTRY_PASSWORD` | tu token de acceso personal con permisos de registro |

GitLab ya provee automáticamente:

* `CI_REGISTRY`
* `CI_PROJECT_PATH`
* `CI_COMMIT_REF_SLUG`

---

## 10. Probar todo localmente

Antes de hacer push, puedes probar localmente:

```bash
# Ejecutar app
python app/main.py
# Salida esperada: Ejemplo de suma: 2 + 3 = 5

# Ejecutar pruebas
pytest tests/

# Construir Docker
docker build -t mi-app .

# Ejecutar Docker
docker run --rm mi-app
```

Si todo funciona localmente, también funcionará en GitLab CI/CD.

---

## 11. Flujo completo en GitLab

1. Haces push a cualquier rama → **Job de test** se ejecuta.
2. Si pasa → **Job de build** construye la imagen Docker.
3. Si es la rama `main` → **Job de deploy** sube la imagen al **Container Registry**.

Esto asegura **automatización completa** y **control de calidad** en cada commit.


---


