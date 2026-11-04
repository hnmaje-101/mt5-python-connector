# mt5-connectors

Conectores async para **MetaTrader 5**: ejecución de órdenes + datos de mercado con **fallback automático** a fuentes externas cuando el broker bloquea la lectura de datos en MT5.

*EN: async MT5 connectors — order execution plus market data with automatic external-data fallback and price-offset calibration.*

## Qué incluye

| Clase | Para qué sirve |
|---|---|
| `MT5Connector` | Datos (OHLCV, ticks, precio) y órdenes (market, cerrar, modificar SL/TP, posiciones) |
| `ExternalDataConnector` | Datos desde Twelve Data / yfinance, con caché, límite de llamadas y fallback |
| `HybridConnector` | MT5 para órdenes y datos; si los datos de MT5 fallan, usa externos y reintenta MT5 solo |

Sin dependencias propias ocultas: logging estándar, sin `tenacity`, sin módulos `core.*`.

## Instalación

```bash
pip install -r requirements.txt     # Python 3.9+
```
`MetaTrader5` solo existe para **Windows** (o Wine). Los conectores externos funcionan en cualquier sistema.

## Uso rápido

```python
from mt5_connectors import HybridConnector

hybrid = HybridConnector(login=123, password="...", server="Broker-Live",
                         twelvedata_key="...", symbol="XAUUSD")
async with hybrid:
    df = await hybrid.get_ohlcv("XAUUSD", "H1", bars=500)   # velas cerradas
    px = await hybrid.get_current_price("XAUUSD")
    print(hybrid.status())                                   # fuente activa, offset, último error
    await hybrid.market_order("XAUUSD", "buy", 0.01, sl=2300.0, tp=2330.0, dry_run=True)
```
Ver `examples/quickstart.py` y `.env.example`.

## Seguridad en órdenes

- `allow_trading=False` por defecto: las órdenes reales fallan hasta que lo actives. `dry_run=True` valida con `order_check` sin enviar nada.
- El volumen se valida contra mínimo/máximo del símbolo y se redondea **hacia abajo** al paso.
- Si MT5 está caído, las órdenes lanzan `MT5ConnectionError` (nunca se pierden en silencio).

## Recuperación ante respuestas ambiguas (timeouts)

Una orden puede ejecutarse aunque la respuesta se pierda. Por eso el conector **nunca reenvía órdenes solo**:

- Cada orden lleva un `client_id` (se guarda en el comentario de MT5 como `#id`). Si no lo das, se genera uno.
- Ante `timeout`, error de conexión o respuesta vacía, consulta MT5 (posiciones abiertas e historial de deals) **antes de decidir**. Si encuentra la ejecución, la devuelve con `reconciled=True`.
- Si no la encuentra, lanza `MT5OrderAmbiguousError` (con `client_id` y `not_found`). Para reintentar, **reutiliza el mismo `client_id`**: si la orden ya existe, se devuelve `already_executed=True` y no se envía nada.
- `reconcile(client_id)` permite consultar el estado manualmente. `modify_position` es idempotente por naturaleza y se reconcilia comparando SL/TP.

```python
try:
    res = await hybrid.market_order("XAUUSD", "buy", 0.01, client_id="ord20261103a")
except MT5OrderAmbiguousError as e:
    # nunca reintentes sin client_id; con el mismo id es seguro
    res = await hybrid.market_order("XAUUSD", "buy", 0.01, client_id=e.client_id)
```
Límite: se detecta por el comentario de la orden; si un broker lo borra o lo modifica, la reconciliación falla y recibirás `MT5OrderAmbiguousError` (nunca una duplicación silenciosa). Pruébalo en tu demo.

## Importante: precios externos ≠ precios de tu broker

- `GC=F` (yfinance) son **futuros**, no XAU/USD spot. Twelve Data tampoco coincide exacto con tu broker.
- `HybridConnector` calibra un **offset** (precio MT5 − precio externo) y lo aplica a los datos externos. Requiere que MT5 entregue al menos el precio actual. Si no puede calibrar, `status()["offset_calibrated"]` es `False` y se avisa en el log: no calcules SL/TP con esos datos.
- El offset es constante; la diferencia real entre futuro y spot varía. Es una aproximación.
- En datos externos, `bid`/`ask` son iguales a `last` (`estimated=True`) y `spread_points` es `None`. No sirven para decidir ejecución.
- `real_volume` es 0 en datos externos.

## Otros detalles

- **Hora del servidor:** MT5 etiqueta las velas con hora del broker, no UTC. Usa `MT5Connector(server_utc_offset_hours=2)` (tu GMT del broker) para obtener índices en UTC real.
- **Velas cerradas:** `closed_only=True` (por defecto) excluye la vela en formación en todos los conectores.
- **`bars` se respeta en todos los timeframes** (antes H1 devolvía siempre 60).
- **Límites:** Twelve Data gratis = 8 llamadas/min, 800/día (el conector respeta el límite y pasa al siguiente proveedor). yfinance limita el historial intradía (1m: 7 días, 5m–30m: 60 días).
- **Timeframe H4 en yfinance** se construye reagrupando velas de 1h (puede no alinear con las H4 de tu broker).
- Alpha Vantage no está incluido (no ofrece XAU intradía fiable en plan gratis). Puedes añadir proveedores propios: cualquier objeto con `name`, `fetch_ohlcv` y `fetch_price`.
- Cada usuario debe usar **su propia API key** y respetar los términos de Yahoo Finance y Twelve Data.

## Prueba en cuenta demo

`python examples/demo_check.py` prueba conexión, datos, ticks y un ciclo completo de orden (abrir 0.01, modificar SL/TP, cerrar). **Aborta si la cuenta no es demo.** Guarda `demo_report.txt` (sin contraseña).

## Tests

```bash
pip install pytest && pytest
```
18 tests offline (sin red ni MetaTrader). Antes de usar dinero real, prueba en **cuenta demo**.

## Aviso legal

Software de herramientas técnicas. No es asesoría financiera ni promete resultados. Operar CFD/futuros conlleva riesgo de pérdida total. Licencia MIT: ver `LICENSE`.

## Autor y soporte

© 2026 Roger David Fernandez Pineda. Todos los derechos reservados.

Soporte: hn.maje.504.hn@gmail.com
