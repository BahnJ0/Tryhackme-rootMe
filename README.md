# Tryhackme-rootMe
Informe técnico de reconocimiento, explotación y remediación del laboratorio RootMe


# Informe de Pentesting — RootMe (TryHackMe)

## 1. Objetivo

El objetivo de esta práctica fue evaluar la seguridad de la máquina **RootMe** de TryHackMe dentro de un entorno controlado y autorizado. Se realizaron tareas de reconocimiento, enumeración de servicios, identificación de vulnerabilidades, explotación controlada y escalada de privilegios.

Además, se analizaron las posibles consecuencias de las debilidades encontradas y se propusieron medidas para prevenirlas.

## 2. Descripción del laboratorio

- **Plataforma:** TryHackMe
- **Máquina:** RootMe
- **Tipo de actividad:** CTF / prueba de penetración en laboratorio
- **Objetivo:** Obtener acceso inicial al sistema y alcanzar privilegios de administrador.
- **Alcance:** Exclusivamente la máquina asignada por TryHackMe.

## 3. Herramientas utilizadas

- **Nmap:** reconocimiento de puertos y servicios.
- **Gobuster:** enumeración de directorios y rutas web.
- **Navegador web:** inspección de la aplicación y del formulario de carga.
- **PHP reverse shell:** comprobación controlada de la posibilidad de ejecutar código PHP desde archivos cargados.
- **Comandos de Linux:** enumeración de archivos y permisos, verificación de identidad y comprobación de privilegios.
- **GTFOBins:** consulta de técnicas de escalada de privilegios asociadas a binarios con permisos especiales.

## 4. Metodología y resultados

### 4.1. Reconocimiento inicial

Se realizó un escaneo con Nmap para identificar los puertos abiertos y los servicios expuestos por la máquina objetivo.

Comando utilizado como referencia:

```bash
nmap -sC -sV -oN nmap-inicial.txt IP_DEL_OBJETIVO
```

Los resultados permitieron identificar los servicios disponibles para orientar la enumeración posterior.

**Evidencia:** `evidencias/01-nmap.png`

> Añadir los puertos, servicios y versiones exactas obtenidas en el escaneo original.

### 4.2. Enumeración de directorios web

Se utilizó Gobuster para descubrir rutas y recursos accesibles en el servidor web.

```bash
gobuster dir -u http://IP_DEL_OBJETIVO \
-w /usr/share/wordlists/dirb/common.txt
```

Entre los resultados observados se encontraron las siguientes rutas:

| Ruta | Resultado observado | Relevancia |
|---|---|---|
| `/index.php` | HTTP 200 | Página accesible |
| `/panel` | Redirección HTTP 301 | Formulario de carga de archivos |
| `/uploads` | Redirección HTTP 301 | Ruta asociada a los archivos cargados |
| `/css` | Redirección HTTP 301 | Recursos de estilos |
| `/js` | Redirección HTTP 301 | Recursos JavaScript |
| `/.htaccess` | HTTP 403 | Acceso denegado |
| `/.htpasswd` | HTTP 403 | Acceso denegado |
| `/server-status` | HTTP 403 | Acceso denegado |

Los códigos HTTP corresponden a los resultados observados durante la enumeración. Un código 301 indica una redirección, mientras que un 403 indica que el acceso fue denegado; ninguno de ellos demuestra por sí solo que exista una vulnerabilidad.

**Evidencia:** `evidencias/02-gobuster.png`

### 4.3. Identificación y explotación de la carga insegura de archivos

Al acceder a `/panel`, se identificó un formulario que permitía subir archivos al servidor. Se comprobó que la aplicación aceptaba una carga y que el archivo quedaba disponible en la ruta asociada a `/uploads`.

Durante el laboratorio se utilizó un archivo PHP de prueba para verificar si el servidor permitía ejecutar código del lado del servidor desde esa ubicación. La prueba permitió obtener una shell inversa y continuar con la enumeración local.

**Debilidad identificada:** controles insuficientes sobre la carga de archivos y posibilidad de ejecutar scripts en un directorio accesible desde la web.

**Impacto potencial:** un atacante podría ejecutar comandos con los permisos del proceso web, acceder a información del sistema y utilizar ese acceso como punto de partida para otros ataques.

**Evidencias:**
- `evidencias/03-formulario-carga.png`
- `evidencias/04-carga-archivo.png`
- `evidencias/05-acceso-inicial.png`

### 4.4. Enumeración local y escalada de privilegios

Una vez obtenido el acceso inicial, se enumeraron los archivos del sistema que contaban con el bit SUID habilitado.

```bash
find / -type f -user root -perm -u=s 2>/dev/null
```

La búsqueda permitió identificar un intérprete de Python con permisos SUID. En determinadas condiciones, un intérprete con este permiso puede ejecutar instrucciones conservando privilegios elevados, lo que constituye una vía de escalada.

Se contrastó el comportamiento con la documentación de GTFOBins y se verificó el nivel de privilegios alcanzado durante el laboratorio.

**Debilidad identificada:** asignación insegura del bit SUID a un intérprete capaz de ejecutar comandos del sistema.

**Impacto potencial:** un usuario local con acceso al sistema podría aprovechar esa configuración para ejecutar comandos con privilegios de otro usuario, incluido `root`, y comprometer la máquina.

**Evidencias:**
- `evidencias/06-enumeracion-suid.png`
- `evidencias/07-verificacion-privilegios.png`

Las flags se obtuvieron durante la práctica. Por seguridad, no se incluyen sus valores en este informe público.

## 5. Vulnerabilidades y remediaciones

### 5.1. Carga insegura de archivos

**Causa:** la aplicación permitía cargar archivos y almacenarlos en una ubicación desde la que era posible ejecutar código PHP.

**Riesgos:**
- Ejecución remota de comandos.
- Acceso no autorizado a archivos y recursos.
- Compromiso de la cuenta del servicio web.
- Posible escalada de privilegios si existen otras configuraciones débiles.

**Remediaciones recomendadas:**
1. Permitir únicamente extensiones necesarias mediante una lista de autorización.
2. Validar el contenido real del archivo en el servidor; no confiar solo en el nombre o el `Content-Type` enviado por el cliente.
3. Generar nombres aleatorios para los archivos y evitar nombres proporcionados directamente por el usuario.
4. Almacenar las cargas fuera del directorio público del servidor web siempre que sea posible.
5. Deshabilitar la ejecución de scripts en el directorio de cargas.
6. Aplicar permisos mínimos a la cuenta del servidor web y limitar su acceso al sistema de archivos.
7. Registrar y supervisar las cargas para detectar comportamientos anómalos.

### 5.2. Configuración insegura de permisos SUID

**Causa:** un intérprete de Python disponía del bit SUID habilitado, permitiendo abusar de su capacidad para ejecutar instrucciones con privilegios elevados.

**Riesgos:**
- Escalada de privilegios local.
- Acceso a información protegida.
- Modificación de archivos críticos.
- Control administrativo del sistema.

**Remediaciones recomendadas:**
1. Retirar el bit SUID de Python si no existe una necesidad operativa legítima.
2. Auditar periódicamente los binarios SUID y SGID del sistema.
3. Aplicar el principio de mínimo privilegio.
4. Evitar asignar permisos SUID a intérpretes, shells y herramientas que puedan ejecutar comandos arbitrarios.
5. Investigar los permisos especiales después de instalar software o modificar la configuración del sistema.

> Las medidas deben probarse previamente en un entorno de administración controlado para evitar afectar servicios legítimos.

## 6. Evidencias

Las capturas deben almacenarse en la carpeta `evidencias/` del repositorio.

| Archivo sugerido | Contenido |
|---|---|
| `01-nmap.png` | Puertos y servicios detectados |
| `02-gobuster.png` | Directorios encontrados |
| `03-formulario-carga.png` | Formulario de `/panel` |
| `04-carga-archivo.png` | Resultado de la prueba de carga |
| `05-acceso-inicial.png` | Evidencia de acceso inicial, sin datos sensibles |
| `06-enumeracion-suid.png` | Binario con permisos SUID identificado |
| `07-verificacion-privilegios.png` | Verificación del nivel de privilegios |

Las capturas deben mostrar únicamente la información necesaria para demostrar cada etapa. No deben contener flags completas, contraseñas, claves privadas, tokens ni otros secretos.

## 7. Conclusiones y aprendizajes

La práctica permitió comprobar cómo una debilidad en la carga de archivos puede convertirse en un punto de entrada al sistema cuando se permite ejecutar contenido cargado por un usuario.

También se observó que una configuración incorrecta de permisos SUID puede transformar un acceso local limitado en un compromiso completo del sistema.

El ejercicio reforzó la importancia de combinar el reconocimiento de servicios, la enumeración web, el análisis de permisos de Linux y la consulta de referencias técnicas para comprender no solo cómo explotar una debilidad en un laboratorio, sino también cómo prevenirla.

## 8. Referencias

- TryHackMe — RootMe: https://tryhackme.com/room/rrootme
- GTFOBins: https://gtfobins.github.io/
- OWASP — File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html

---

**Autor:** [Tu nombre o alias]  
**Fecha:** [Fecha de realización]  
**Repositorio:** [Enlace al repositorio de GitHub]
