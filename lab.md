# Laboratorio Práctico: Pipeline de SAST + SCA, Generación de SBOM y Quality Gate por Severidad

> **Proyecto:** [secure-sdlc-pipeline](https://github.com/kbcasurf/secure-sdlc-pipeline)  
> **Tema:** DevSecOps / Seguridad en el Ciclo de Vida del Desarrollo de Software (Secure SDLC)  
> **Herramientas:** Semgrep (SAST), OWASP Dependency-Check (SCA), Syft (SBOM / CycloneDX), GitHub Actions / Docker  

---

## 1. Objetivos del Laboratorio

Al finalizar este laboratorio, serás capaz de:
1. **Implementar SAST (Static Application Security Testing)** con **Semgrep** para detectar fallos de código fuente (CWE-89 SQLi, CWE-78 Command Injection, CWE-327 Criptografía Débil, etc.).
2. **Implementar SCA (Software Composition Analysis)** con **OWASP Dependency-Check** para identificar dependencias vulnerables con CVEs conocidos en `package.json`.
3. **Generar e inspeccionar un SBOM (Software Bill of Materials)** en formato estándar **CycloneDX** utilizando **Syft**.
4. **Definir y aplicar una política de fallo del build (Quality Gate)** por severidad (CVSS >= 7.0 / Severidad High-Critical) para bloquear despliegues inseguros en el pipeline de CI/CD.
5. **Remediar las vulnerabilidades identificadas** y validar que el pipeline pase de estado fallido (*FAILED*) a exitoso (*PASSED*).

---

## 2. Requisitos Previos

- **Git** instalado en el sistema local.
- **Docker** y **Docker Compose** en ejecución (permite ejecutar los escáneres sin instalar dependencias adicionales).
- **Node.js** (v18 o v20 LTS) y `npm` (opcional, para remediación manual de dependencias).
- Acceso a **GitHub** (cuenta propia para realizar un *fork* o clon del repositorio y ejecutar GitHub Actions).

---

## 3. Arquitectura del Laboratorio

```
  Código Fuente (app/) ──► package.json (Dependencias)
         │                         │
         ▼                         ▼
   [ SAST: Semgrep ]       [ SCA: Dependency-Check ]
         │                         │
         │                         ▼
         │                 [ SBOM: Syft (CycloneDX) ]
         │                         │
         └─────────────┬───────────┘
                       ▼
          [ Quality Gate / Build Break ]
              ¿CVSS >= 7.0 o High/Critical?
                 ├── SÍ  ──► ❌ Build Fallido (Bloqueo)
                 └── NO  ──► ✅ Build Exitoso (Aprobado)
```

---

## 4. Paso a Paso del Laboratorio

### Paso 1: Clonar y Explorar el Entorno

1. Clona el repositorio y sitúate en el directorio de trabajo:
   ```bash
   git clone https://github.com/kbcasurf/secure-sdlc-pipeline.git
   cd secure-sdlc-pipeline
   ```

2. Inspecciona los archivos clave del proyecto:
   - `app/src/server.js`: API Node.js/Express con vulnerabilidades intencionales en el código fuente.
   - `app/src/db.js`: Capa de datos simulada vulnerable a inyección SQL.
   - `app/package.json`: Manifiesto con dependencias intencionalmente obsoletas (`lodash@4.17.11`, `express@4.17.1`).
   - `docs/findings-walkthrough.md`: Catálogo de vulnerabilidades plantadas y su mapeo CWE/OWASP.

---

### Paso 2: Ejecución de SAST con Semgrep

Semgrep analiza el código fuente en busca de patrones de código inseguro sin necesidad de compilar ni ejecutar la aplicación.

#### Opción A: Ejecución local con Docker
Ejecuta Semgrep sobre el directorio `app/`:
```bash
docker run --rm -v "${PWD}:/src" returntocorp/semgrep semgrep scan --config auto app/
```
*(En Windows PowerShell usa `${PWD}`, en Linux/macOS usa `$(pwd)`)*

#### Opción B: Ejecución con Semgrep CLI (si tienes Python/pip)
```bash
pip install semgrep
semgrep scan --config auto --sarif --output semgrep.sarif app/
```

#### Hallazgos esperados en SAST:
- **`app/src/server.js` (Línea ~31):** SQL Injection (CWE-89) al concatenar variables en la query `SELECT * FROM users WHERE name = '...'`.
- **`app/src/server.js` (Línea ~40):** Command Injection (CWE-78) al concatenar entrada en `exec('ping -c 1 ' + host)`.
- **`app/src/server.js` (Línea ~48):** Weak Hashing (CWE-327) por uso de algoritmo MD5 no salteado para contraseñas.
- **`app/src/server.js` (Línea ~55):** Prototype Pollution (CWE-1321) por merge inseguro con Lodash.

---

### Paso 3: Ejecución de SCA con OWASP Dependency-Check

OWASP Dependency-Check examina las dependencias de terceros declaradas en `package.json` y `package-lock.json` contra la base de datos de vulnerabilidades del National Vulnerability Database (NVD).

> [!IMPORTANT]
> **API Key de NVD (Requerida para evitar errores de actualización):**
> La NVD (NIST) impone límites estrictos de tasa (*rate limit*). Sin una API Key gratuita, las descargas suelen fallar con el error:  
> `[ERROR] Error updating the NVD Data - org.owasp.dependencycheck.data.update.exception.UpdateException`.  
> Puedes solicitar tu API key gratuita en: [https://nvd.nist.gov/developers/request-an-api-key](https://nvd.nist.gov/developers/request-an-api-key).  
> Se pasa mediante el argumento `--nvdApiKey "<TU_API_KEY>"` o la variable de entorno `NVD_API_KEY`.

1. Prepara las dependencias y carpetas de trabajo:
   ```bash
   # Generar el package-lock.json para que Dependency-Check analice el árbol completo de dependencias
   docker run --rm -v "${PWD}/app:/app" -w /app node:18-alpine npm install --package-lock-only

   # Crear carpetas para reportes y caché de la base de datos
   mkdir odc-reports
   mkdir odc-data
   ```

2. Ejecuta el escaneo de Dependency-Check mediante su imagen oficial de Docker:
   - **En Linux / macOS:**
     ```bash
     docker run --rm \
       -v "$(pwd)/app:/src" \
       -v "$(pwd)/odc-data:/usr/share/dependency-check/data" \
       -v "$(pwd)/odc-reports:/report" \
       owasp/dependency-check:latest \
       --scan /src \
       --format "ALL" \
       --project "secure-sdlc-demo" \
       --nvdApiKey "TU_API_KEY_AQUI" \
       --nvdApiDelay 6000 \
       --out /report
     ```
   - **En Windows PowerShell:**
     ```powershell
     docker run --rm `
       -v "${PWD}/app:/src" `
       -v "${PWD}/odc-data:/usr/share/dependency-check/data" `
       -v "${PWD}/odc-reports:/report" `
       owasp/dependency-check:latest `
       --scan /src `
       --format "ALL" `
       --project "secure-sdlc-demo" `
       --nvdApiKey "TU_API_KEY_AQUI" `
       --nvdApiDelay 6000 `
       --out /report
     ```
   > [!TIP]
   > El parámetro `--nvdApiDelay 6000` evita el error `HTTP 503 (Service Unavailable)` al espaciar las solicitudes a la API de NIST NVD durante la descarga inicial de la base de datos de CVEs.

3. Abre el archivo de reporte generado en tu navegador:
   - Abre `odc-reports/dependency-check-report.html`.

4. **Análisis de resultados SCA:**
   - Observa las vulnerabilidades asociadas a **`lodash:4.17.11`**:
     - `CVE-2019-10744` (Prototype Pollution, severidad **CRITICAL**, CVSS 9.8).
     - `CVE-2020-8203` (Prototype Pollution, severidad **HIGH**, CVSS 7.4).
   - Observa las dependencias transitivas vulnerables arrastradas por **`express:4.17.1`**.

---

### Paso 4: Generación e Inspección del SBOM con Syft

El SBOM (Software Bill of Materials) genera el inventario de componentes, versiones, hashes de integridad y licencias que conforman el paquete de software.

1. Genera el SBOM en formato **CycloneDX JSON** usando Syft de forma directa:
   - **En Linux / macOS:**
     ```bash
     docker run --rm \
       -v "$(pwd):/workspace" \
       -w /workspace \
       anchore/syft:latest \
       dir:app \
       --source-name "vulnerable-demo-app" \
       --source-version "0.1.0" \
       -o cyclonedx-json=sbom.cyclonedx.json \
       -q
     ```
   - **En Windows PowerShell:**
     ```powershell
     docker run --rm `
       -v "${PWD}:/workspace" `
       -w /workspace `
       anchore/syft:latest `
       dir:app `
       --source-name "vulnerable-demo-app" `
       --source-version "0.1.0" `
       -o cyclonedx-json=sbom.cyclonedx.json `
       -q
     ```

   > [!NOTE]
   > - Usar `-o cyclonedx-json=sbom.cyclonedx.json` permite que Syft escriba el archivo internamente sin depender de redirecciones de PowerShell (`| Out-File`).
   > - `--source-name` y `--source-version` definen formalmente el artefacto en los metadatos del SBOM, eliminando advertencias (`WARN`).
   > - El flag `-q` (*quiet*) suprime mensajes informativos secundarios.

2. Inspecciona el SBOM generado:
   ```bash
   # Ver metadatos y componentes principales
   cat sbom.cyclonedx.json
   ```
3. Verifica que en el nodo `components` se encuentren registrados formalmente los paquetes `express`, `lodash` y sus respectivas versiones.

---

### Paso 5: Implementación de la Política de Fallo del Build (Quality Gate)

En un pipeline de seguridad maduro, el análisis no debe ser meramente informativo; debe **detener el pipeline (Build Break)** si se sobrepasan los umbrales de severidad acordados por la organización.

#### 1. Criterio de la Política de Fallo:
- **SAST (Semgrep):** Detener el pipeline ante cualquier vulnerabilidad de severidad `ERROR` / `HIGH`.
- **SCA (OWASP Dependency-Check):** Detener el pipeline si se detecta cualquier vulnerabilidad con puntaje **CVSS >= 7.0** (`--failOnCVSS 7`).

#### 2. Configurar el Workflow de GitHub Actions:
Edita o actualiza el archivo `.github/workflows/security.yml` para incorporar OWASP Dependency-Check con la política bloqueante activa:

```yaml
name: Secure SDLC Pipeline

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read
  security-events: write

jobs:
  # ---------------------------------------------------------------- SAST
  sast:
    name: SAST · Semgrep (Quality Gate)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep with Quality Gate
        # NOTA: Sin continue-on-error para aplicar fallo estricto
        run: |
          pip install semgrep
          semgrep scan --config auto --error --sarif --output semgrep.sarif app/

      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: semgrep.sarif
          category: semgrep

  # ---------------------------------------------------------------- SCA
  sca-dependency-check:
    name: SCA · OWASP Dependency-Check (Quality Gate CVSS >= 7)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Prepare reports directory
        run: mkdir -p reports

      - name: Run OWASP Dependency-Check
        # Falla el build si encuentra vulnerabilidades con CVSS >= 7.0 (High o Critical)
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
        run: |
          docker run --rm \
            -e NVD_API_KEY="${{ secrets.NVD_API_KEY }}" \
            -v ${{ github.workspace }}/app:/src \
            -v ${{ github.workspace }}/reports:/report \
            owasp/dependency-check:latest \
            --scan /src \
            --format "ALL" \
            --project "Secure SDLC Demo" \
            --out /report \
            --failOnCVSS 7

      - name: Upload Dependency-Check Reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dependency-check-report
          path: reports/

  # ---------------------------------------------------------------- SBOM
  sbom:
    name: SBOM · Syft (CycloneDX)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          path: ./app
          format: cyclonedx-json
          output-file: sbom.cyclonedx.json

      - uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.cyclonedx.json
```

3. **Prueba de fallo (Build Fails):**
   - Realiza un `commit` y `push` de esta configuración.
   - En la pestaña **Actions** de GitHub verás que los jobs de **SAST** y **SCA** terminan en estado **FAILED (rojo)** debido a que detectan vulnerabilidades críticas que superan el umbral configurado.

---

### Paso 6: Remediación de Vulnerabilidades (Shift-Left)

Para superar el Quality Gate y lograr que el build vuelva a verde, se deben corregir los hallazgos:

#### 1. Remediación de Código (SAST)
En `app/src/server.js`:
- **SQL Injection:** Evitar concatenación. En `app/src/db.js`, utilizar consultas parametrizadas.
- **Command Injection:** Reemplazar `exec` por validación estricta de parámetros o llamadas no interpretadas por shell.
- **Weak Crypto:** Reemplazar MD5 por una función criptográficamente segura (ej. `crypto.scryptSync` o biblioteca de derivación de claves).
- **Prototype Pollution:** Evitar `_.merge` con objetos arbitrarios del cuerpo de la petición.

#### 2. Remediación de Dependencias (SCA)
Actualiza `app/package.json` a versiones seguras y libres de los CVEs críticos:

```json
{
  "name": "vulnerable-demo-app",
  "version": "0.1.0",
  "private": true,
  "description": "Secure demo service",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js"
  },
  "dependencies": {
    "express": "^4.21.0",
    "lodash": "^4.17.21"
  }
}
```

Genera el lockfile actualizado:
```bash
cd app
npm install --package-lock-only
cd ..
```

---

### Paso 7: Validación del Pipeline en Verde

1. Haz commit de los cambios corregidos:
   ```bash
   git add app/
   git commit -m "fix(security): resolve SAST findings and upgrade dependencies to patch CVEs"
   git push origin main
   ```

2. Verifica la ejecución del pipeline en GitHub Actions:
   - **Semgrep:** Finaliza sin errores (`0 findings matching threshold`).
   - **OWASP Dependency-Check:** No encuentra vulnerabilidades con CVSS >= 7.0.
   - **SBOM:** Refleja las nuevas versiones seguras en el inventario.
   - **Resultado general:** **Build PASSED (verde) ✅**.

---

## 5. Preguntas de Reflexión y Evaluación

1. **¿Cuál es la diferencia fundamental entre SAST y SCA?**
   - *SAST analiza el código propietario escrito por el equipo de desarrollo; SCA analiza las librerías de terceros y dependencias externas que el proyecto incorpora.*
2. **¿Por qué la generación de SBOM es crucial en el ciclo de vida de desarrollo?**
   - *Permite conocer con exactitud matemática el inventario de software y actuar de forma inmediata cuando se divulga un nuevo Zero-Day o CVE en librerías de terceros (ej. Log4j).*
3. **¿Qué consecuencias tiene en un entorno empresarial un umbral de fallo (Quality Gate) demasiado restrictivo vs. uno demasiado permisivo?**
   - *Demasiado restrictivo (bloquear por vulnerabilidades Low/Info) causa fatiga de alertas y fricción con los desarrolladores; demasiado permisivo deja pasar riesgos críticos a producción.*

---

## 6. Resumen de Comandos Rápidos

| Actividad | Herramienta | Comando Local / Parámetro |
|---|---|---|
| **SAST** | Semgrep | `docker run --rm -v "${PWD}:/src" returntocorp/semgrep semgrep scan --config auto app/` |
| **SCA** | Dependency-Check | `docker run --rm -v "${PWD}/app:/src" -v "${PWD}/odc-reports:/report" owasp/dependency-check --scan /src --failOnCVSS 7` |
| **SBOM** | Syft | `docker run --rm -v "${PWD}/app:/app" anchore/syft dir:/app -o cyclonedx-json > sbom.json` |
| **Gate SAST** | Semgrep | Flag `--error` en la línea de escaneo |
| **Gate SCA** | Dependency-Check | Flag `--failOnCVSS <valor>` (ej. `7` para High/Critical) |
