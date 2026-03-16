
# Kerberos explicado con el símil de un parque de atracciones

## Introducción

Kerberos es un protocolo de autenticación usado principalmente en **Active Directory** cuyo objetivo es **autenticar usuarios sin enviar contraseñas por la red**.

Para entenderlo mejor lo compararemos con **un parque de atracciones**.

---

# Arquitectura de Kerberos

![Flujo Kerberos](kerberos_flow.jpg)

## Componentes principales

### Usuario (Client)
Es quien quiere acceder a un servicio.

Ejemplo:

usuario@dominio.local

En el símil sería **el visitante del parque**.

---

### KDC (Key Distribution Center)

El **KDC** es el sistema central de autenticación.

Normalmente se ejecuta en el **Domain Controller**.

El KDC contiene:

- **Authentication Service (AS)**
- **Ticket Granting Service (TGS)**
- **Base de datos de identidades (Active Directory)**

Dentro del KDC se almacenan:

- hashes de contraseñas de usuarios
- cuentas de servicio
- SPN (Service Principal Names)
- grupos y privilegios
- políticas de seguridad

Ejemplos de políticas:

- política de contraseñas
- expiración de tickets
- control de privilegios
- reglas de autenticación

Por eso el KDC funciona como:

> taquilla central + base de datos + sistema de reglas del parque

---

### Base de datos de identidades

El KDC consulta **Active Directory** para verificar:

- identidad del usuario
- hash de contraseña
- pertenencia a grupos
- privilegios
- servicios disponibles (SPN)

---

# Equivalencia con el parque

| Kerberos | Parque |
|---|---|
Usuario | Visitante |
KDC | Taquilla central |
TGT | Pulsera del parque |
TGS | Ticket de atracción |
AP | Atracción |
SPN | Operador de la atracción |

---

# Flujo normal de Kerberos

### 1️⃣ AS_REQ
Usuario → KDC

El usuario solicita autenticación.

En el parque:

"Hola, quiero entrar".

---

### 2️⃣ AS_REP
El KDC entrega un **TGT (Ticket Granting Ticket)**.

En el parque equivale a:

**la pulsera del parque**.

---

### 3️⃣ TGS_REQ
Usuario → KDC

El usuario pide un ticket para un servicio específico.

Ejemplo:

"Quiero subir a la montaña rusa".

---

### 4️⃣ TGS_REP
El KDC entrega el **ticket de servicio**.

---

### 5️⃣ AP_REQ
Usuario → Servicio (AP)

El usuario presenta el ticket.

---

### 6️⃣ AP_REP
El servicio valida el ticket.

---

# Preauthentication

La **preauthentication** obliga al usuario a demostrar que conoce su contraseña antes de recibir el TGT.

El cliente envía:

timestamp cifrado con su clave.

El KDC lo descifra usando el hash almacenado en Active Directory.

---

# Qué es el Timestamp en Kerberos

Un timestamp es una **marca de tiempo**.

Ejemplo:

2026-03-16 18:42:15

Se usa para evitar **replay attacks**.

---

## Cómo funciona

El cliente envía:

EncryptedTimestamp

Conceptualmente:

Encrypt(timestamp, clave_usuario)

El KDC intenta descifrarlo.

Si el timestamp es válido → autenticación correcta.

---

## Qué ve un atacante

Un atacante solo verá datos cifrados:

AS_REQ  
EncryptedTimestamp = 7A1B23F8...

La contraseña **no viaja por la red**.

---

# Ataques contra Kerberos

![Comparación ataques](kerberos_attacks.png)

---

## AS-REP Roasting

Ocurre cuando la preauthentication está desactivada.

Configuración vulnerable:

Do not require Kerberos preauthentication

Entonces el KDC devuelve directamente:

AS_REP

El atacante puede capturar ese ticket y crackearlo offline.

---

# Kerberoasting explicado en profundidad

Aquí está la parte clave que conecta todo.

### Problema que Kerberos debe resolver

Cuando un usuario accede a un servicio:

SMB  
HTTP  
MSSQL  

El servicio necesita saber:

> ¿este usuario está autenticado?

Pero el servicio **no conoce la contraseña del usuario**.

Solo el **KDC** conoce las contraseñas.

---

## Qué hace el KDC

Cuando el usuario solicita un ticket de servicio:

TGS_REQ

El KDC crea un **Service Ticket** que contiene:

- usuario
- grupos
- privilegios
- timestamp
- session key

Pero ese ticket debe ir **cifrado**.

---

## ¿Con qué clave se cifra?

Si el ticket se cifrara con la contraseña del usuario:

el servicio no podría descifrarlo.

Porque el servicio **no conoce la contraseña del usuario**.

---

## Solución del diseño de Kerberos

El ticket se cifra con la **clave del servicio**.

Ejemplo:

HTTP/webserver

Ese servicio tiene una cuenta en Active Directory.

El ticket se cifra con:

clave_servicio

Entonces el flujo es:

Usuario → Servicio  
ticket cifrado con clave_servicio

El servicio puede descifrarlo usando su propia clave.

---

## Símil del parque

Cada atracción tiene **su propia llave**.

La taquilla crea el ticket y lo cierra con:

la llave de esa atracción.

Cuando llegas:

el operador usa su llave y abre el ticket.

---

## Aquí aparece Kerberoasting

Si un atacante obtiene el ticket de servicio:

TGS_REP

puede intentar descifrarlo offline.

Proceso:

1. adivinar contraseña
2. generar clave
3. intentar descifrar ticket
4. si funciona → contraseña del servicio

---

## Por qué el ataque funciona

Porque cualquier usuario autenticado puede pedir tickets de servicio.

Entonces el atacante puede:

- pedir muchos tickets
- guardarlos
- crackearlos offline

---

## Por qué es peligroso

Las cuentas de servicio suelen tener:

- contraseñas antiguas
- contraseñas débiles
- privilegios altos

Si se obtiene la contraseña del servicio:

se puede acceder al servicio o escalar privilegios.

---

# Resumen final

Kerberos funciona como un parque:

1. te autenticas en la taquilla
2. recibes pulsera (TGT)
3. pides ticket de atracción (TGS)
4. presentas ticket en la atracción (AP)

Ataques principales:

| Ataque | Objetivo |
|---|---|
AS-REP Roasting | contraseña de usuario |
Kerberoasting | contraseña de servicio |
Golden Ticket | clave KRBTGT |
Silver Ticket | clave de servicio |
