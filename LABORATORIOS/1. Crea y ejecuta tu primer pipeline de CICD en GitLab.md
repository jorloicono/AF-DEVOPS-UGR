# Crea y ejecuta tu primer pipeline de CI/CD en GitLab

---

## Requisitos previos

Antes de comenzar, asegúrate de tener:

* Un proyecto en GitLab donde quieras usar CI/CD.
* El rol de **Maintainer** o **Owner** en el proyecto.
* Si no tienes un proyecto, puedes crear uno público y gratuito en [https://gitlab.com](https://gitlab.com).

---

## Pasos para crear y ejecutar tu primer pipeline

### 1. Asegúrate de tener *runners* disponibles

En GitLab, los *runners* son agentes que ejecutan los trabajos (jobs) de CI/CD.

* Si estás usando **GitLab.com**, puedes **omitir este paso**. GitLab.com proporciona *runners* por defecto.

Para verificar los *runners* disponibles:

1. En la barra lateral izquierda, selecciona **Buscar** o navega hasta tu proyecto.
2. Ve a **Settings > CI/CD**.
3. Expande la sección **Runners**.
4. Si tienes al menos un *runner* activo (círculo verde), puedes procesar tus jobs.

> Si no puedes ver esta configuración, contacta al administrador de GitLab.

---

### Si no tienes un *runner*

1. Instala [GitLab Runner](https://docs.gitlab.com/runner/install/) en tu máquina local.
2. Registra el *runner* en tu proyecto. Selecciona el *executor* tipo `shell`.

Cuando ejecutes tus jobs, se ejecutarán en tu máquina local.

---

### 2. Crea el archivo `.gitlab-ci.yml`

Este archivo se coloca en la raíz de tu repositorio y contiene las instrucciones para GitLab CI/CD.

Define:

* La estructura y orden de los *jobs* a ejecutar.
* Las decisiones que debe tomar el *runner* bajo ciertas condiciones.

#### Pasos para crear el archivo:

1. En la barra lateral izquierda, busca tu proyecto.
2. Ve a **Code > Repository**.
3. Arriba de la lista de archivos, selecciona la rama (usa `main` o `master` si no estás seguro).
4. Haz clic en el ícono ➕ y selecciona **New file**.
5. En el campo de nombre, escribe: `.gitlab-ci.yml`
6. Pega el siguiente contenido:

```yaml
build-job:
  stage: build
  script:
    - echo "Hello, $GITLAB_USER_LOGIN!"

test-job1:
  stage: test
  script:
    - echo "This job tests something"

test-job2:
  stage: test
  script:
    - echo "This job tests something, but takes more time than test-job1."
    - echo "After the echo commands complete, it runs the sleep command for 20 seconds"
    - echo "which simulates a test that runs 20 seconds longer than test-job1"
    - sleep 20

deploy-prod:
  stage: deploy
  script:
    - echo "This job deploys something from the $CI_COMMIT_BRANCH branch."
  environment: production
```

> Este ejemplo tiene cuatro *jobs*: `build-job`, `test-job1`, `test-job2`, y `deploy-prod`.

Haz clic en **Commit changes**.

Esto inicia el pipeline y ejecuta los *jobs* definidos.

---

## Verifica el estado de tu pipeline

1. Ve a **Build > Pipelines**.
2. Deberías ver un pipeline con tres etapas.

Haz clic en el **ID del pipeline** para ver una visualización gráfica del mismo.

Haz clic en un nombre de job (por ejemplo, `deploy-prod`) para ver los detalles del job: estado, duración y registros (logs).

¡Has creado tu primer pipeline de CI/CD en GitLab!

---

## Consejos sobre `.gitlab-ci.yml`

* Consulta la [referencia completa de sintaxis YAML para CI/CD](https://docs.gitlab.com/ee/ci/yaml/).
* Usa el **Editor de pipelines** para editar el archivo.
* Cada job tiene una sección `script` y pertenece a una etapa `stage`.
* Los jobs dentro de una misma etapa se ejecutan en paralelo si hay *runners* disponibles.
* Usa la palabra clave `needs` para ejecutar jobs fuera de orden y acelerar tu pipeline.
* Usa `rules` para definir cuándo ejecutar o saltar un job. (`only` y `except` siguen siendo válidos, pero no deben usarse junto con `rules` en un mismo job).
* Usa `cache` y `artifacts` para mantener archivos entre jobs y etapas.
* Usa `default` para definir configuraciones comunes, como `before_script` y `after_script`.

