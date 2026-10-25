# TryHackMe RootMe — Informe técnico

**Actividad:** Reconocimiento, explotación y remediación de vulnerabilidades  
**Plataforma:** TryHackMe  
**Laboratorio:** RootMe  
**Tipo de práctica:** CTF / prueba de penetración en entorno controlado

## 1. Objetivo

Evaluar la seguridad de la máquina **RootMe** de TryHackMe mediante tareas de reconocimiento, enumeración de servicios, identificación de vulnerabilidades, explotación controlada y escalada de privilegios.

Asimismo, analizar las posibles consecuencias de las debilidades identificadas y proponer medidas correctivas que permitan reducir el riesgo de explotación en entornos reales.

## 2. Descripción y alcance del laboratorio

El laboratorio RootMe permite practicar técnicas de reconocimiento y explotación de vulnerabilidades en un sistema Linux dentro de un entorno controlado.

- **Plataforma:** TryHackMe.
- **Máquina objetivo:** RootMe.
- **Tipo de actividad:** CTF y prueba de penetración.
- **Objetivo técnico:** Obtener acceso inicial al sistema y comprobar la posibilidad de alcanzar privilegios de administrador.
- **Alcance:** Exclusivamente la máquina asignada dentro del entorno autorizado de TryHackMe.

Todas las pruebas descritas en este informe se enmarcan en el contexto del laboratorio.

## 3. Herramientas utilizadas

- **Nmap:** identificación de puertos abiertos y enumeración de servicios.
- **Gobuster:** descubrimiento de directorios y rutas web.
- **Navegador web:** inspección de la aplicación y del formulario de carga de archivos.
- **PHP reverse shell:** prueba de ejecución de código y obtención de acceso inicial en el entorno controlado.
- **Comandos de Linux:** enumeración del sistema, inspección de permisos y verificación de privilegios.
- **GTFOBins:** consulta de técnicas de abuso de binarios con permisos especiales.

## 4. Metodología y resultados

### 4.1. Reconocimiento inicial

Se utilizó Nmap para identificar los puertos abiertos y los servicios expuestos por la máquina objetivo.

**Comando utilizado como referencia:**

```bash
nmap -sC -sV -oN nmap-inicial.txt IP_DEL_OBJETIVO
```

Las opciones utilizadas permiten ejecutar scripts predeterminados, detectar versiones de servicios e identificar los resultados en un archivo de texto.

Los resultados del reconocimiento sirven para determinar qué servicios requieren una enumeración más detallada.

**Evidencia:** [01-nmap.png](evidencias/01-nmap.png)

**Nota:** los puertos, servicios y versiones deben corresponder a los resultados reales del escaneo realizado durante la práctica.

### 4.2. Enumeración de directorios web

Se utilizó Gobuster para identificar rutas y recursos accesibles en el servidor web.

**Comando utilizado como referencia:**

```bash
gobuster dir -u http://IP_DEL_OBJETIVO \
-w /usr/share/wordlists/dirb/common.txt
```

Durante la enumeración se identificaron las siguientes rutas:

| Ruta | Resultado observado | Relevancia |
|---|---|---|
| `/index.php` | HTTP 200 | Página web accesible |
| `/panel` | Redirección HTTP 301 | Ruta asociada al formulario de carga |
| `/uploads` | Redirección HTTP 301 | Ruta asociada a los archivos cargados |
| `/css` | Redirección HTTP 301 | Recursos de estilos |
| `/js` | Redirección HTTP 301 | Recursos JavaScript |
| `/.htaccess` | HTTP 403 | Acceso denegado |
| `/.htpasswd` | HTTP 403 | Acceso denegado |
| `/server-status` | HTTP 403 | Acceso denegado |

Los códigos HTTP deben interpretarse según el comportamiento de cada recurso. Un código 301 indica una redirección y un 403 indica que el servidor denegó el acceso a la solicitud. Por sí solos, estos códigos no demuestran la existencia de una vulnerabilidad.

**Evidencia:** [02-gobuster.png](evidencias/02-gobuster.png)

### 4.3. Identificación y explotación de la carga de archivos

Durante la inspección de la aplicación se identificó un formulario de carga de archivos en la ruta `/panel`. Se observó que los archivos enviados podían quedar disponibles en la ruta asociada a `/uploads`.

En el laboratorio se utilizó un archivo PHP de prueba para comprobar si el servidor permitía ejecutar código del lado del servidor desde esa ubicación. Según los resultados documentados de la práctica, esta prueba permitió obtener una shell inversa y continuar con la enumeración local.

**Debilidad identificada:** controles insuficientes en la carga de archivos que permiten almacenar y ejecutar scripts en una ubicación accesible desde la web.

**Impacto potencial:**

- Ejecución remota de comandos con los permisos de la cuenta del servidor web.
- Acceso no autorizado a archivos y recursos del sistema.
- Compromiso de la cuenta utilizada por el servicio web.
- Posibilidad de avanzar hacia otras cuentas o privilegios si existen configuraciones adicionales inseguras.

**Evidencias:**

- [03-formulario-carga.png](evidencias/03-formulario-carga.png)
- [04-carga-archivo.png](evidencias/04-carga-archivo.png)

### 4.4. Enumeración local y escalada de privilegios

Después de obtener el acceso inicial, se revisaron los permisos especiales de los archivos del sistema, prestando atención a los binarios con el bit SUID habilitado.

**Comando utilizado:**

```bash
find / -type f -user root -perm -u=s 2>/dev/null
```

Este comando busca archivos propiedad de `root` que tengan habilitado el bit SUID. Dicho permiso puede provocar que un programa se ejecute con el identificador de usuario efectivo de su propietario, en lugar de hacerlo únicamente con el del usuario que lo invoca.

Durante la práctica se identificó un intérprete de Python con permisos SUID. La presencia de este permiso en un intérprete capaz de ejecutar instrucciones arbitrarias puede permitir el abuso de privilegios, dependiendo de la configuración y del comportamiento del sistema.

Se consultó GTFOBins para contrastar las posibilidades de abuso del binario y se verificó el nivel de privilegios alcanzado en el laboratorio.

**Debilidad identificada:** asignación insegura del permiso SUID a un intérprete capaz de ejecutar código o comandos del sistema.

**Impacto potencial:**

- Escalada de privilegios desde una cuenta de usuario limitada.
- Acceso a archivos protegidos.
- Modificación de archivos críticos.
- Compromiso de la integridad y confidencialidad del sistema.
- Obtención de privilegios administrativos si la configuración vulnerable lo permite.

Las flags se obtuvieron durante la práctica. Sus valores se omiten de este informe público para evitar exponerlas directamente.

## 5. Vulnerabilidades identificadas y remediaciones

### 5.1. Carga insegura de archivos

**Causa:** la aplicación permitía cargar archivos en una ubicación desde la que era posible ejecutar código PHP.

**Riesgos:**

- Ejecución remota de comandos.
- Acceso no autorizado a archivos y recursos.
- Compromiso de la cuenta del servidor web.
- Uso del acceso inicial para intentar una escalada de privilegios.

**Remediaciones recomendadas:**

1. Aplicar una lista de extensiones permitidas según las necesidades de la aplicación.
2. Validar el contenido real de cada archivo en el servidor, sin confiar exclusivamente en su extensión o en el encabezado `Content-Type` enviado por el cliente.
3. Generar nombres aleatorios para los archivos y evitar utilizar directamente los nombres proporcionados por los usuarios.
4. Almacenar los archivos fuera del directorio público del servidor web cuando sea posible.
5. Deshabilitar la ejecución de scripts en el directorio destinado a las cargas.
6. Aplicar permisos mínimos a la cuenta del servidor web y restringir su acceso al sistema de archivos.
7. Registrar y supervisar las cargas para detectar actividad anómala.

### 5.2. Configuración insegura de permisos SUID

**Causa:** un intérprete de Python disponía del bit SUID habilitado, lo que podía permitir la ejecución de instrucciones con privilegios elevados.

**Riesgos:**

- Escalada de privilegios local.
- Acceso a información protegida.
- Modificación de archivos críticos.
- Compromiso de la máquina y de los recursos accesibles desde ella.

**Remediaciones recomendadas:**

1. Retirar el bit SUID de Python si no existe una necesidad operativa legítima.
2. Auditar periódicamente los binarios SUID y SGID del sistema.
3. Aplicar el principio de mínimo privilegio a usuarios, servicios y aplicaciones.
4. Evitar asignar permisos SUID a intérpretes, shells y herramientas capaces de ejecutar instrucciones arbitrarias.
5. Revisar los permisos especiales después de instalar software o modificar la configuración del sistema.
6. Investigar el origen de los permisos inseguros y comprobar que no existan otras vías de escalada.

Las modificaciones de permisos deben probarse en un entorno controlado antes de aplicarse a sistemas de producción, para evitar afectar servicios legítimos.

## 6. Evidencias de la práctica

Las evidencias se organizan en la carpeta `evidencias/` del repositorio. Los archivos previstos documentan las etapas principales del procedimiento:

## 7. Conclusiones y aprendizajes

La práctica permitió analizar cómo una debilidad en la carga de archivos puede convertirse en un punto de entrada al sistema cuando no se aplican controles adecuados y se permite ejecutar contenido cargado por un usuario.

También permitió estudiar los riesgos asociados a una configuración incorrecta de permisos SUID, que puede convertir un acceso local limitado en un compromiso más amplio del sistema.

El ejercicio reforzó la importancia de combinar el reconocimiento de servicios, la enumeración web, el análisis de permisos de Linux y la consulta de documentación técnica para comprender tanto las técnicas de explotación como las medidas necesarias para prevenirlas.

Finalmente, el laboratorio destacó la importancia de documentar los resultados de manera clara, respaldar las conclusiones con evidencias verificables y aplicar el principio de mínimo privilegio para reducir la superficie de ataque.

## 8. Referencias

- [TryHackMe — RootMe](https://tryhackme.com/room/rrootme)
- [GTFOBins](https://gtfobins.github.io/)
- [Documentación oficial de Nmap](https://nmap.org/book/man.html)
- [OWASP — File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [OWASP — Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

---

**Nota de reproducibilidad:** los comandos incluidos son referencias del procedimiento. Para reproducir la práctica, debe utilizarse la dirección IP asignada por TryHackMe y contrastarse cada resultado con las evidencias obtenidas durante la ejecución real del laboratorio.
