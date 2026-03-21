
# Writeup EXTREMO - Máquina Literal

> Este documento está diseñado para que entiendas TODO desde cero.
> No solo qué hacer, sino POR QUÉ funciona cada paso.

---

# ÍNDICE COMPLETO

1. Virtualización y problema inicial
2. Redes (NAT vs Bridge explicado profundamente)
3. Descubrimiento de red
4. Enumeración con Nmap (nivel interno)
5. Análisis del servicio web
6. Fuzzing explicado en profundidad
7. Vite interno (cómo funciona realmente)
8. Vulnerabilidad Arbitrary File Read (nivel interno)
9. Explotación manual paso a paso
10. Enumeración de archivos sensibles
11. JWT explicado a nivel criptográfico
12. Forja de tokens (por qué funciona realmente)
13. Burp Suite (flujo real de ataque)
14. RCE (qué está pasando internamente)
15. Reverse shell (flujo de sockets explicado)
16. Post-explotación profunda
17. Git leaks (por qué Git filtra secretos)
18. SSH y claves privadas (modelo de seguridad)
19. Privilege Escalation (conceptos internos)
20. Hash cracking (cómo funciona realmente)
21. Conclusión técnica completa

---

# 1. VIRTUALIZACIÓN

El problema no es “VMware falla”, el problema es:

→ **Formato OVF no compatible completamente**

VMware y VirtualBox interpretan OVF de forma distinta.

Esto genera:
- errores de importación
- incompatibilidad de hardware virtual

SOLUCIÓN REAL:
→ Separar hipervisores pero unificar red

---

# 2. REDES (ESTO ES CLAVE)

## NAT

- VM detrás del host
- No visible en la red real
- Traducción de IP (NAT)

## BRIDGE

- VM = dispositivo real en la red
- Obtiene IP del router
- Comunicación directa

👉 IMPORTANTE:
Si no entiendes esto, no entiendes hacking en labs.

---

# 3. DESCUBRIMIENTO

Nmap usa:
- ARP
- ICMP
- TCP SYN

La clave aquí:
→ MAC Address revela el hipervisor

VirtualBox:
08:00:27

Esto NO es casualidad:
es el OUI asignado por IEEE.

---

# 4. NMAP (INTERNO)

Cuando haces:

nmap -sCV

internamente ocurre:

1. SYN scan
2. banner grabbing
3. fingerprinting
4. scripts NSE

Esto NO es magia:
son firmas + heurísticas.

---

# 5. SERVICIO WEB

Puerto 3000 → típico Node.js

Pero lo importante fue:
vite.config.js

Eso significa:

→ servidor de desarrollo expuesto

GRAVE error.

---

# 6. FUZZING

Fuzzing ≠ brute force

Es:
→ exploración inteligente basada en wordlists

El filtro --fs elimina ruido:
mismo tamaño = misma respuesta

---

# 7. VITE (INTERNO)

Vite sirve archivos directamente del filesystem.

Problema:
→ validación incorrecta de rutas

---

# 8. VULNERABILIDAD

Arbitrary File Read ocurre porque:

1. validación incompleta
2. normalización incorrecta
3. bypass con query (?raw??)

Esto rompe el control de acceso.

---

# 9. EXPLOTACIÓN

Leer:

/etc/passwd

NO es para contraseñas
es para:

→ enumerar usuarios

---

# 10. .ENV

Esto es CRÍTICO.

.env contiene:
- secretos
- configuración
- lógica interna

Aquí rompimos:

→ autenticación (JWT)

---

# 11. JWT (CRYPTO)

JWT = Header + Payload + Signature

Signature = HMAC(secret, data)

Si tienes el secret:

→ eres el servidor

---

# 12. FORJA

No hackeas nada.

Simplemente:
usas la misma clave.

---

# 13. BURP

Burp no hackea.

Te permite:
→ ver y modificar tráfico real

---

# 14. RCE

Cuando ejecutas:

cmd=id

El backend hace algo como:

exec(cmd)

Eso es:
→ ejecución directa del sistema

---

# 15. REVERSE SHELL

Flujo real:

1. víctima conecta a atacante
2. abre socket TCP
3. redirige stdin/stdout

Esto convierte:

→ comando → shell interactiva

---

# 16. POST-EXPLOTACIÓN

Buscar:
- credenciales
- servicios internos
- repositorios

---

# 17. GIT

Git guarda TODO el historial.

Aunque borres algo:

→ sigue en commits

---

# 18. SSH

Clave privada = acceso total

Pero requiere:
chmod 600

---

# 19. PRIVESC

sudo -l = oro

GTFOBins = manual de abuso

---

# 20. HASH

Hash ≠ contraseña

Hashcat:
prueba millones de combinaciones

---

# 21. CONCLUSIÓN

Cadena completa:

Vite → File Read → JWT → RCE → Shell → Git → SSH → Sudo → Root

Esto es una cadena realista.

---

# FINAL

Si entiendes este documento:

→ ya no estás siguiendo pasos
→ estás entendiendo hacking real
