# Práctica: Artefactos, Cache y Needs en GitLab CI/CD

> Objetivo: Comprender cómo funcionan **artefactos**, **cache** y **needs** dentro de los pipelines de GitLab, y cómo se usan para compartir y optimizar resultados entre jobs.

---

## 1️⃣ Crear proyecto en GitLab

1. Entra a [https://gitlab.com](https://gitlab.com)
2. Crea un nuevo proyecto llamado **demo-artifacts**
3. No inicialices con README ni .gitignore.
4. Clona el repositorio localmente:

```bash
git clone git@gitlab.com:usuario/demo-artifacts.git
cd demo-artifacts
````

---

## 2️⃣ Crear estructura básica del proyecto

Creamos algunos archivos:

```bash
mkdir src build
echo "console.log('Ejecutando aplicación de demo');" > src/app.js
echo "# Demo de artefactos en GitLab CI/CD" > README.md
```

Agrega y haz commit:

```bash
git add .
git commit -m "Proyecto base para práctica de artefactos"
git push origin master
```

---

## 3️⃣ Crear el pipeline `.gitlab-ci.yml`

Crea un archivo `.gitlab-ci.yml` en la raíz del proyecto con el siguiente contenido:

```yaml
stages:
  - build
  - test
  - deploy

# =====================
# 1. JOB DE BUILD
# =====================
build-job:
  stage: build
  script:
    - echo "Compilando el proyecto..."
    - mkdir build
    - echo "Resultado de compilación $(date)" > build/output.txt
    - cat build/output.txt
  artifacts:
    paths:
      - build/
    expire_in: 1 hour
    when: always

# =====================
# 2. JOB DE TEST
# =====================
test-job:
  stage: test
  needs: ["build-job"]     # Espera los artefactos del job anterior
  script:
    - echo "Ejecutando pruebas..."
    - cat build/output.txt
    - echo "✅ Pruebas completadas exitosamente"
  artifacts:
    reports:
      junit: report.xml
    paths:
      - report.xml
    expire_in: 1 hour

# =====================
# 3. JOB DE DEPLOY
# =====================
deploy-job:
  stage: deploy
  needs: ["test-job"]
  script:
    - echo "Desplegando aplicación..."
    - cat build/output.txt
    - echo "Despliegue completado"
  when: manual
```

---

## 4️⃣ Explicación práctica de cada sección

| Concepto        | Qué hace                                                                          | Ejemplo en la práctica                               |
| --------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------- |
| **`artifacts`** | Archivos generados por un job que se pueden descargar o pasar a otros jobs.       | `build-job` genera `build/output.txt` y lo comparte. |
| **`paths`**     | Define qué rutas se guardarán como artefactos.                                    | `build/`                                             |
| **`expire_in`** | Tiempo de vida de los artefactos.                                                 | `1 hour`, `1 week`, etc.                             |
| **`when`**      | Cuándo guardar artefactos (`always`, `on_success`, `on_failure`).                 | En `build-job` usamos `always`.                      |
| **`needs`**     | Permite que un job reciba artefactos directamente sin esperar todo el stage.      | `test-job` obtiene `build/output.txt` del build.     |
| **`reports`**   | Archivos especiales que GitLab interpreta (como reportes de pruebas o cobertura). | `test-job` genera un `report.xml` tipo `junit`.      |

---

## 5️⃣ Ejecutar el pipeline

1. Haz commit y push del archivo `.gitlab-ci.yml`:

```bash
git add .gitlab-ci.yml
git commit -m "Agrega pipeline con artefactos y needs"
git push origin master
```

2. Abre tu proyecto en GitLab → **CI/CD → Pipelines**
3. Verás las tres etapas:

   * **build**
   * **test**
   * **deploy**

---

## 6️⃣ Verificar artefactos

1. En la interfaz de GitLab → **Jobs → build-job**
2. Al finalizar, verás un botón **"Browse"** o **"Download artifacts"**.
3. Puedes abrir o descargar el archivo generado:
   `build/output.txt`

También puedes usar artefactos en otros jobs (como el test o deploy), sin necesidad de volver a generar los archivos.

---

## 7️⃣ Probar con Cache (para acelerar builds)

Agrega cache al job de build:

```yaml
build-job:
  stage: build
  script:
    - echo "Compilando el proyecto con cache..."
    - mkdir -p build
    - echo "Resultado cacheado $(date)" > build/output.txt
  cache:
    key: build-cache
    paths:
      - build/
  artifacts:
    paths:
      - build/
```

### 🔍 Diferencia clave:

| Elemento      | Persiste entre pipelines               | Se comparte entre jobs  | Objetivo                                             |
| ------------- | -------------------------------------- | ----------------------- | ---------------------------------------------------- |
| **Artifacts** | ❌ No (solo dentro del pipeline actual) | ✅ Sí                    | Compartir resultados entre stages del mismo pipeline |
| **Cache**     | ✅ Sí                                   | ✅ (si se usa misma key) | Acelerar pipelines futuros reutilizando archivos     |

---

## 8️⃣ Ejemplo avanzado con artefactos de prueba y despliegue

Agrega al final del `.gitlab-ci.yml`:

```yaml
generate-report:
  stage: test
  script:
    - echo "<testsuite><testcase name='demo-test'/></testsuite>" > test-report.xml
  artifacts:
    reports:
      junit: test-report.xml
    expire_in: 30 min

deploy-summary:
  stage: deploy
  needs: ["generate-report"]
  script:
    - echo "Usando reporte de pruebas:"
    - cat test-report.xml
    - echo "Despliegue exitoso"
```

GitLab mostrará los resultados de `generate-report` en la sección **"Tests"** del pipeline.
`deploy-summary` podrá leer ese reporte gracias a `needs`.

---

## 9️⃣ Limpieza

Si quieres borrar artefactos manualmente:

* Desde la interfaz: *Job → Artifacts → Delete*
* O con la API:
  [GitLab API Artifacts Docs](https://docs.gitlab.com/ee/api/job_artifacts.html#delete-artifacts)

---




