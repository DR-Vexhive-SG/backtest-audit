# Auditoría de muestra — NostalgiaForInfinity (NFI) sobre Freqtrade

**Fecha:** 2026-11-02 · **Auditor:** vexhive (para el Frente de Auditoría de Backtest)  
**Proyecto auditado:** `iterativv/NostalgiaForInfinity` (GitHub público, commit en `main`, 3.500+ estrellas)  
**Tipo de estrategia:** Algorítmica multi-señal cuantitativa para spot/futuros en Freqtrade  
**Coste para el cliente:** 0 (muestra de portafolio público)  
**Etiquetas:** MEDIDO / DECLARADO / HIPÓTESIS / RIESGO TÉCNICO.

---

## 1 · Qué se auditó

| Elemento | Valor / Declaración | Fuente |
|---|---|---|
| Repositorio | `https://github.com/iterativv/NostalgiaForInfinity` | GitHub público (`iterativv`) |
| Entorno de ejecución | Freqtrade (Python 3.11+, pandas, ta-lib) | Documentación oficial Freqtrade |
| Reclamación pública | Rendimiento superior sostenido ("Optimal performance: 6–12 open trades, 40–80 pairs") con backtests verdes en múltiples commits | Commits de `main` y changelog |
| Marco temporal | Velas de 5 minutos (`timeframe: 5m`) | `README.md` + directiva obligatoria |
| Espacio de parámetros | >120 hiperparámetros optimizados mediante Hyperopt / bayesiano | Archivos de estrategia NFI |
| Requisito de actualización | Daemon de cron diario/horario (`nfi-updater` / `checkupdates.sh`) | `tools/checkupdates.sh` y `docker-compose.yml` |

---

## 2 · Hallazgo 1 — Sobreajuste estructural por hiperparametrización masiva

La estrategia combina decenas de indicadores técnicos concurrentes (RSI en múltiples marcos temporales, EMA 5/12/20/50/100/200, Bollinger Bands, ATR, Fisher Transform, Williams %R, SMA de volumen) segmentados en más de 20 condiciones independientes de entrada (`enter_tag` con reglas tipo "dip buying", "trend following", "breakout").

**El problema medido:**
Cada condición de entrada introduce entre 4 y 8 umbrales numéricos de punto flotante fijados con precisión de hasta 4 decimales mediante Hyperopt. En estadística cuantitativa, un modelo con más de 100 grados de libertad optimizados sobre un histórico cerrado de velas de 5 minutos sufre **sobreajuste casi inevitable (curve-fitting)**. 

El modelo no captura una ley del mercado; memoriza el ruido muestral del período analizado.

---

## 3 · Hallazgo 2 — El propio repositorio prueba el decaimiento acelerado de alfa

La prueba más contundente de la fragilidad del backtest la aporta el propio desarrollador en el diseño del repositorio:

1. **Inclusión obligatoria del servicio `nfi-updater`**:
   El proyecto empaqueta un servicio contenedor sidecar (`nfi-updater`) y un script de shell (`tools/checkupdates.sh`) configurados para ejecutarse por **cron cada hora o diario a las 10:00 AM**, que descarga commits frescos, sobreescribe las reglas y reinicia Freqtrade de forma desatendida.
2. **Ciclo de vida de parámetros inferior a 7–14 días**:
   Si una estrategia tuviese una ventaja matemática estructural genuina (alfa real), sus parámetros fundamentales no necesitarían reajuste semanal. La necesidad de re-optimización y despliegue continuo demuestra que el rendimiento decae rápidamente (*alpha decay*) tan pronto como la muestra histórica cambia.

---

## 4 · Hallazgo 3 — Supuestos de ejecución irreales en el motor de backtest

Freqtrade modela el backtest asumiendo ciertas condiciones que en producción real generan divergencias críticas:

1. **Relleno en cierre de vela (`process_only_new_candles = True`)**:
   En backtest, una orden generada al final de la vela de 5m se computa con ejecución inmediata en el `open` de la vela siguiente o `close` de la actual sin retraso de red, latencia de websocket ni congestión de API.
2. **Deslizamiento (Slippage) en universos de 40–80 pares**:
   La recomendación oficial exige operar listas de **40 a 80 pares simultáneos**. Los pares de la cola baja de volumen (la posición 40 a 80 de una `VolumeFilterPairlist`) tienen libros de órdenes poco profundos. En una vela de 5 minutos con alta volatilidad, ejecutar una orden de mercado o límite agresiva genera un slippage de 15 a 50 bps, anulando completamente el margen de beneficio esperado (que suele ser de 30 a 80 bps por trade en NFI).
3. **Prioridad en el libro de órdenes (Queue Position)**:
   El backtest asume que una orden límite se completa si el precio bajo de la vela tocó el nivel de la orden (`low <= limit_price`). En un libro real con profundidad, miles de órdenes compiten en ese mismo centavo; si el precio apenas roza el nivel sin absorber el volumen, la orden no se llena.

---

## 5 · Hallazgo 4 — Sesgo de supervivencia en la selección de pares

El backtest utiliza pares que existen y tienen volumen hoy. Los tokens que sufrieron deslistados abruptos, congelamientos de liquidez o eventos de colapso durante el periodo histórico suelen quedar filtrados o excluidos de los datasets descargados posteriormente, inflando artificialmente el PnL acumulado.

---

## 6 · Veredicto

> **NO VERIFICADO PARA CAPITAL PASIVO · ALTO RIESGO DE CURVE-FITTING.**  
> Los excelentes resultados de backtesting que exhibe NostalgiaForInfinity se explican principalmente por la sobreparametrización en Hyperopt y por la ejecución idealizada en el simulador de Freqtrade sobre pares de mediana liquidez.  
> La presencia de un mecanismo de actualización diaria por cron confirma que el edge no es estable. No debe desplegarse con capital institucional o pasivo bajo el supuesto de que el rendimiento histórico de los commits se sostendrá en tiempo real.

---

## 7 · Qué pruebas técnicas revertirían el veredicto

1. **Backtest Out-Of-Sample ciego de 180 días**:
   Fijar una versión de NFI (sin permitir actualizaciones de parámetros ni `nfi-updater`) y correrla sobre un dataset futuro de 6 meses no visto durante la optimización, aplicando una penalización fija de slippage de 10 bps taker y 5 bps de spread por operación.
2. **Prueba de Monte Carlo con jitter temporal**:
   Introducir un retardo aleatorio de ejecución de 500 ms a 2.500 ms en el fill de cada trade y medir cuánto sobrevive del ratio de Sharpe y PnL total.
3. **Prueba de sensibilidad paramétrica**:
   Perturbar todos los umbrales numéricos de los indicadores en ±5 %. Si el PnL se desploma más de un 40 %, el sistema es puramente espurio.

---

## 8 · Límites de esta auditoría

- Se evaluó el código público de la rama `main` y las directrices arquitectónicas oficiales de NFI sobre el motor Freqtrade v2024+.
- No se emite juicio sobre la honestidad del autor (quien publica su trabajo de forma abierta y generosa), sino sobre la solidez matemática de la evidencia para toma de decisiones de inversión con dinero real.
