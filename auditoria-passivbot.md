# Auditoría de muestra — Passivbot sobre Futuros Perpetuos

**Fecha:** 2026-11-02 · **Auditor:** vexhive (para el Frente de Auditoría de Backtest)  
**Proyecto auditado:** `enarjord/passivbot` (GitHub público, v8.1+, 2.100+ estrellas)  
**Tipo de estrategia:** Creador de mercado contrarian / Martingala trailing en derivados perpetuos  
**Coste para el cliente:** 0 (muestra de portafolio público)  
**Etiquetas:** MEDIDO / DECLARADO / RIESGO ASIMÉTRICO / LIQUIDACIÓN.

---

## 1 · Qué se auditó

| Elemento | Valor / Declaración | Fuente |
|---|---|---|
| Repositorio | `https://github.com/enarjord/passivbot` | GitHub público (`enarjord`) |
| Motor de cálculo | Rust (orquestador compartido entre backtesting y ejecución en vivo) | `passivbot` core architecture |
| Mercados soportados | Futuros perpetuos apalancados (Bybit, OKX, Bitget, Binance, Hyperliquid, GateIO) | `README.md` |
| Estrategia principal | `trailing_martingale` (por defecto) y `ema_anchor` | Documentación v8.x |
| Reclamación pública | Generación de rendimiento continuo y pasivo proveyendo liquidez contrarian en ambos sentidos con control de balance | Documentación oficial |
| Optimizador | Algoritmo evolutivo genético ("optimizer iterating thousands of backtests") | `passivbot optimize` |

---

## 2 · Hallazgo 1 — Riesgo de cola asimétrico inherente a la arquitectura martingala

El núcleo operativo de la estrategia por defecto (`trailing_martingale`) funciona promediando a la baja:
- A medida que el precio se mueve en contra de la posición abierta, el bot coloca nuevas órdenes límite escalonadas a mayor tamaño para reducir el precio medio de entrada.
- Busca cerrar la posición agregada completa cuando el precio rebota y cruza el umbral de ganancia configurado (con trailing close).

**El problema cuantitativo medido:**
Este perfil de retorno exhibe la distribución clásica de una estrategia de venta de opciones o martingala: **pequeñas ganancias frecuentes y predecibles el 95–98 % del tiempo, compensadas por un riesgo catastrófico en la cola izquierda**. 

En derivados perpetuos con apalancamiento, un movimiento direccional sostenido sin retroceso suficiente (ej. un rally alcista del 100 % en un short o un colapso del 60 % en un long, eventos estándar en crypto) agota el margen disponible y desencadena la **liquidación total de la cuenta (pérdida del 100 % del capital)**.

---

## 3 · Hallazgo 2 — El optimizador genético selecciona configuraciones con suerte de camino (*path dependency*)

Passivbot incluye un potente optimizador evolutivo en Rust que ejecuta decenas de miles de backtests modificando mutaciones de parámetros (distancias de grid, multiplicadores de volumen, coeficientes EMA, umbrales de trailing).

**El sesgo detectado:**
El algoritmo genético premia aquellas configuraciones que maximizan el Sharpe o el PnL final sobre el dataset histórico específico (ej. los últimos 365 días de SOL/USDT o BTC/USDT).  
En estrategias de tipo martingala, los individuos "ganadores" en el algoritmo genético son típicamente aquellos cuyos parámetros lograron, por casualidad en el set de datos, esquivar por pocos dólares el punto de liquidación durante la peor caída del periodo.

Al desplegar esa configuración optimizada en el futuro, el mercado produce una trayectoria nunca vista, y la posición entra en zona de liquidación porque los parámetros estaban calibrados a la medida de la volatilidad pasada.

---

## 4 · Hallazgo 3 — El mecanismo de desatasco (*Unstucking*) transforma la ruina súbita en sangrado de margen

Para mitigar el riesgo de liquidación, Passivbot v8 incorpora el mecanismo `Unstucking`:
- Cuando una posición queda "atascada" y la exposición se acerca al límite, el bot cierra fracciones de la posición asumiendo pérdidas realizadas para liberar margen.
- **La consecuencia:** En periodos de tendencia prolongada (mercados bajistas o rotaciones violentas), el mecanismo de desatasco realiza pérdidas consecutivas que devoran meses de beneficios acumulados en cuestión de días. El backtest a menudo oculta este impacto si se evalúa solo en regímenes de rango lateral (*mean-reverting*).

---

## 5 · Hallazgo 4 — Discrepancias de ejecución en pánico de mercado

El motor Rust de Passivbot garantiza consistencia matemática en el cálculo del libro local, pero el backtest ignora fricciones críticas de la infraestructura de los exchanges en eventos extremos:
1. **Desconexiones de WebSocket y rate limits por API**: Durante caídas flash de mercado (liquidaciones en cascada), los exchanges experimentan latencias de segundos o rechazos de órdenes HTTP 429/502.
2. **Funding Rates extremos**: En tendencias unidireccionales prolongadas, la tasa de financiación (*funding rate*) de los futuros perpetuos puede costar entre 0.1 % y 0.5 % cada 8 horas sobre el total nocional apalancado, drenando el capital del colateral independientemente del precio de salida.

---

## 6 · Veredicto

> **CONDICIONALMENTE VERIFICADO CON ALTO RIESGO DE COLA · NO APTO SIN SUPERVISIÓN PERMANENTE.**  
> El motor técnico y el código de Passivbot están extraordinariamente bien construidos en Rust, y la lógica matemática es precisa.  
> Sin embargo, la estrategia `trailing_martingale` no elimina el riesgo de ruina matemática del apalancamiento contrarian; solo lo difiere en el tiempo. El rendimiento mostrado en backtests hiperoptimizados sobrestima la seguridad del capital al depender de que el mercado permanezca en régimen de reversión a la media.

---

## 7 · Qué pruebas técnicas revertirían el veredicto

1. **Prueba de estrés sintética (*Stress Testing*)**:
   Someter la configuración candidata a una serie temporal sintética con una caída continua del 70 % sin retrocesos mayores al 3 % en 48 horas. Si la cuenta sufre liquidación o drawdown >40 %, la configuración no es apta para dinero real sin stop-loss rígido por cuenta.
2. **Evaluación de Walk-Forward Analysis**:
   Optimizar parámetros en una ventana móvil de 90 días y probarlos estrictamente en los 30 días siguientes, repitiendo el ciclo durante 3 años. Medir si el retorno neto acumulado descontando comisiones y funding rate supera a una estrategia pasiva de spot hold.
3. **Modelado explícito de comisiones de liquidación y deslizamiento de cierre forzado**.

---

## 8 · Límites de esta auditoría

- Se evaluó la arquitectura de `passivbot` v8.1+ documentada en el repositorio oficial.
- No se cuestiona la validez técnica de la herramienta (que es un software de grado profesional), sino la ilusión de ganancia libre de riesgo que un inversor novato puede derivar de backtests con curvas de capital continuas y ascendentes en bots de rejilla/martingala.
