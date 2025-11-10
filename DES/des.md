# Documento de Diseño de Script de Normalización de Archivos

### 1. Introducción y Propósito

Este documento describe el diseño y la funcionalidad de un script de Python cuya función principal es normalizar y subseleccionar columnas de archivos de datos delimitados, utilizando archivos de configuración JSON como plantilla.

El propósito central es garantizar que los archivos de salida (_out.txt) contengan únicamente las columnas requeridas y especificadas en la configuración, generando un log de errores para las columnas faltantes.

### 2. Alcance y Contexto

El script está diseñado para ejecutarse en un entorno donde se tienen definidos directorios de entrada (configuración y archivos base) y un directorio de salida principal. La transformación se basa en un mapeo estricto de nombres de columnas.

#### Dependencias Requeridas

- os: Para la gestión de rutas y la creación de directorios.
- json: Para la lectura de los archivos de configuración.
- pandas (pd): Para la lectura, manipulación y exportación de DataFrames.

#### Variables de Entorno/Ruta (Asumidas)

El script depende de las siguientes variables de ruta predefinidas:

- `content`: Lista de nombres de archivos de configuración JSON a procesar.
- `config_dir`: Directorio donde se encuentran los archivos de configuración JSON.
- `file_dir`: Directorio donde se encuentran los archivos de datos base (.txt).
- `main_dir`: Directorio raíz para almacenar los resultados procesados.
`
### 3. Flujo de Procesamiento Detallado

El script opera a través de un proceso iterativo doble:

#### Paso 1: Lectura de Configuración

1. Iteración de Archivos de Configuración (content): El script recorre la lista content, abriendo y cargando cada archivo JSON desde config_dir.

2. Identificación de Categoría: El nombre del archivo de configuración (file) se usa para determinar la carpeta de destino (cacs o cc).

3. Iteración de Layouts: Por cada archivo de configuración, se itera sobre las claves dentro de config['layouts']. Cada clave representa un layout (y, por extensión, el nombre de un archivo de datos base).

#### Paso 2: Transformación de Datos

1. Lectura de Archivo Base: Se lee el archivo de datos base correspondiente: f"{files_dir}/{layout}.txt".

2. Se asume que este archivo está delimitado por el carácter pipe (|) y utiliza codificación latin 1.

3. Validación de Columnas: Se compara la lista de columnas requeridas (config_columns) con las columnas existentes en el archivo base (data_columns).

#### Construcción del DataFrame de Salida (df_out):

1. Se inicializa un DataFrame vacío.

2. Se itera sobre cada columna requerida en la configuración:

- Columna Existente: Si la columna existe en el archivo base, se copia la columna completa al nuevo df_out.

- Columna Faltante: Si la columna no existe, se registra el error en un archivo de log y se omite esa columna, continuando con la siguiente.

#### Paso 3: Exportación de Archivos y Organización

1. Definición de Ruta de Salida: Se verifica el nombre del archivo de configuración original (file).

- Si contiene "cacs", la salida va a f"{main_dir}/cacs".

- Si contiene "cc", la salida va a f"{main_dir}/cc".

2. Creación de Directorio: El directorio de destino (cacs o cc) se crea si no existe (os.makedirs(..., exist_ok=True)).

3. Exportación: El df_out resultante se guarda en la ruta de destino: f"{main_dir}/[categoria]/{layout}_out.txt".

La exportación mantiene el formato delimitado por pipe (|) y la codificación latin 1.

### 4. Estructuras de Entrada (Inputs)

#### 4.1. Archivos de Configuración (JSON)

- Ubicación: config_dir/

Ejemplo de Estructura:
```json
{
  "layouts": {
    "layout_123": {
      "COLUMNA_REQUERIDA_A": {},
      "COLUMNA_REQUERIDA_B": {}
    },
    "layout_456": {
      "OTRA_COLUMNA": {},
      "MAS_COLUMNAS": {}
    }
  }
}
```

#### 4.2. Archivos de Datos Base (TXT)

- Ubicación: files_dir/
- Nomenclatura: {layout}.txt (Ej: layout_123.txt)
- Formato: Delimitado por |, codificación latin 1.

Ejemplo de Contenido:

COLUMNA_REQUERIDA_A|COLUMNA_EXTRA|COLUMNA_REQUERIDA_B
Valor1|X|Valor3
Valor2|Y|Valor4


### 5. Salidas (Outputs)

#### 5.1. Archivos de Datos Procesados

- Ubicación: main_dir/cacs/ o main_dir/cc/
- Nomenclatura: {layout}_out.txt (Ej: layout_123_out.txt)
- Formato: Delimitado por |, codificación latin 1.
- Contenido: Incluye solo las columnas que fueron especificadas en el JSON y que existían en el archivo base.

#### 5.2. Archivos de Log de Errores

- Ubicación: Directorio de ejecución del script (donde se lanza el proceso).
- Nomenclatura: {layout}_log.txt (Ej: layout_123_log.txt)
- Contenido: Un registro por cada columna solicitada en la configuración pero no encontrada en el archivo de datos base.

Ejemplo de Contenido del Log:

```txt
La columna COLUMNA_FALTANTE no existe o esta mal escrita en el archivo base
```


### 6. Consideraciones de Manejo de Errores
|Tipo de Error|Detección|Acción|
|---|---|---|
Columna Faltante|if column not in data_columns:|Se escribe un registro en el archivo {layout}_log.txt. El script continue al siguiente campo, no detiene la ejecución.
Archivo Base Faltante|pd.read_csv(f"{files_dir}/{layout}.txt", ...)|No cubierto en el script. Se asume que layout.txt existe. Si no, lanzará una excepción FileNotFoundError y detendrá la ejecución.
JSON Inválido|json.load(f)|No cubierto en el script. Lanzará un json.JSONDecodeError y detendrá la ejecución.