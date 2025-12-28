# SerialWDM [![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/KunYi/SerialWDM)

🔧 **Overview**

SerialWDM is a sample Windows serial port driver (WDM) that demonstrates core components of a classic serial driver: ISR (Interrupt Service Routine), power and PnP support, synchronous/asynchronous I/O, and multiport handling. The repository also includes analysis documents useful for modernizing to KMDF.

---

## Key Files 🔍

- `isr.c`, `immediat.c`: interrupt and immediate processing
- `read.c`, `write.c`, `pktxfer.c`: data transfer and packet handling
- `power.c`, `pnp.c`: power management and PnP
- `openclos.c`, `purge.c`: device open/close and buffer purge logic
- `log.c`, `serlog.mc`: logging and event definitions
- `serial.h`, `serialp.h`, `precomp.h`: public/internal headers

---

## Documentation & Migration Notes 📚

- `docs/KMDF_Migration_Analysis.md`: analysis and guidance for migrating to KMDF
- `docs/ISR_Analysis.md`: ISR design analysis
- `docs/Multiport_Analysis.md`: multiport considerations
- `docs/WinXP_Arch_Recommendation.md`: recommendations for legacy platforms

---

## Provenance & License ⚖️

- Source: Portions of this code are derived from the Microsoft Windows 2003 Server SP1 DDK (build 3790.1830), specifically `src/kernel/serial`.
- License: This repository states that licensing follows the Microsoft DDK internal specification. If you intend to reuse or redistribute the code, consult Microsoft licensing terms and get confirmation from the repository owner before doing so.

---

## Testing & Debugging 🔧

- Use WinDbg (kernel debugger) to inspect interrupts, IRP flows, and power-state transitions.
- Event logging (`serlog.mc` / `log.c`) is available for runtime diagnostics.
