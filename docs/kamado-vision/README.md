# Kamado Vision — lectura del termómetro de domo por visión

Investigación de hardware, arquitectura y costos para leer la aguja del termómetro
analógico de un kamado con una cámara y publicar la temperatura en una app/página.

Contexto: Buenos Aires, agosto 2026. Cotización usada para convertir: **USD 1 ≈ ARS 1.545**
(blue, 15/08/2026). Precios ARS verificados marcados con ✔; el resto son estimaciones de
rango de mercado local.

---

## 1. Lo primero: acotar el problema real

Antes del hardware, tres cosas que cambian el diseño:

**a) El termómetro de domo es la parte imprecisa, no la visión.**
Los diales de domo tienen error típico de ±10–15 °C, y además miden en la cúpula, no en
la parrilla. La diferencia domo↔parrilla que reporta la comunidad va de ~20 °C a ~70 °C
según el cocido, el deflector y si hay comida tapando el flujo. En cambio, un pipeline de
visión sobre un dial rígidamente encuadrado te da **±1–2 °C de error de lectura**.

Conclusión práctica: la visión no es el cuello de botella de precisión. El sistema va a ser
tan bueno como el reloj que está leyendo. Vale la pena asumirlo explícitamente y, si te
importa el número absoluto, validar contra una termocupla (§6, Plan E) para armar la curva
de corrección una sola vez.

**b) La cámara y el dial quedan rígidos entre sí.**
Este es el mayor simplificador del proyecto. Si el soporte es rígido, el centro del dial, el
radio y el rango angular son constantes conocidas. El problema se reduce a *"encontrar el
ángulo de la aguja dentro de un anillo fijo"* — no hace falta detectar el círculo por frame,
ni Hough, ni una CNN.

**c) Es un ambiente hostil para electrónica.**
Tapa a 150–300 °C, sol directo, humo graso, vapor, lluvia, y el asado es de noche. El
diseño tiene que separar *dónde está el sensor* de *dónde está la computadora*.

---

## 2. Restricciones físicas del montaje

| Restricción | Implicancia de diseño |
|---|---|
| El bisel del termómetro quema; la tapa irradia | Cámara a ≥ 25–30 cm, fuera de la columna térmica del damper superior. Electrónica (ESP32/Pi) nunca sobre la tapa. |
| Vidrio/acrílico del dial → reflejo especular | El enemigo #1. Iluminación **off-axis** (LED a 30–45° del eje óptico, no frontal), visera, y opcionalmente film polarizador sobre la lente. |
| Vapor y grasa condensan en la lente | Visera/parasol pequeño, lente apuntando levemente hacia arriba, y rutina de limpieza. |
| Se cocina de noche | Iluminación propia obligatoria (LED blanco continuo o IR — el IR anda bien sobre dial blanco con aguja negra). |
| Exterior: lluvia, sol | IP54 mínimo para lo que quede afuera. |
| WiFi en el patio | Medir RSSI antes de comprar. Si es malo: antena externa, repetidor, o cámara con cable. |
| Alimentación 5 V | Preferir 220 V + fuente. Batería sólo si no hay toma cerca (un asado son 4–12 h). |

**Truco de robustez que recomiendo desde el día 1:** pegar 3 marcadores fiduciales
(círculos negros sobre vinilo blanco, o un ArUco chico) alrededor del bisel. Si alguien mueve
la cámara, o el soporte vibra, el sistema recalcula la homografía por frame y la calibración
no se rompe nunca. Cuesta $0 y elimina la falla más probable del proyecto.

---

## 3. El pipeline de visión (es más simple de lo que parece)

Con montaje rígido + fiduciales, esto alcanza:

1. **Warp**: detectar los 3 fiduciales → homografía → dial siempre en la misma pose canónica.
2. **Unwrap polar** alrededor del centro conocido: para cada ángulo θ ∈ [0, 360), muestrear
   los radios r ∈ [r_min, r_max] y promediar la intensidad. Da un vector de 360 valores.
3. **Argmax de oscuridad** con interpolación sub-grado (centroide del pico) → ángulo de la aguja.
   Resolver la ambigüedad aguja/contrapeso restringiendo al barrido válido del dial.
4. **Mapeo lineal** ángulo → °C, calibrado con dos marcas impresas del dial (los kamado son lineales).
5. **Filtro temporal**: mediana móvil de N lecturas + rechazo de saltos > X °C/min. Mata los
   falsos por humo denso o reflejo transitorio.

Es ~80 líneas de OpenCV, corre en cualquier cosa, y es determinístico y depurable. Es el
enfoque que recomiendo por sobre una CNN.

**Alternativas si no querés escribir el pipeline:**
- **AI-on-the-edge-device** (jomjol): firmware ESP32-CAM/ESP32-S3 que ya trae una red para
  agujas analógicas, web UI, calibración por ROI, MQTT + autodiscovery de Home Assistant,
  REST y OTA. Pensado para medidores de agua/gas, pero el caso "aguja sobre dial" es idéntico.
  Cadencia ~1 lectura/min, que para un asado sobra.
- Referencias clásicas: el sample de Intel `analog-gauge-reader`, `Ghostbird/Analogue-Gauge-Reader`,
  y el paper *Under pressure: learning-based analog gauge reading in the wild* (arXiv 2404.08785)
  si algún día querés generalizar a diales arbitrarios.

**Expectativa de precisión:** con el dial ocupando ~300 px y un barrido típico de ~270° para
40–400 °C, la resolución angular de ~0.5° equivale a ~0.7 °C. El error de lectura queda
dominado por el ancho de la aguja, no por el algoritmo: **±1–2 °C**.

---

## 4. Opciones de cámara + micro

### Cámaras

| Opción | Pros | Contras | ARS aprox. |
|---|---|---|---|
| **ESP32-CAM (OV2640 2 MP)** | Cámara + micro + WiFi en una sola placa, baratísima | Foco fijo, mala en poca luz, ruido/EMI (bandas rosas), WiFi flojo sin antena externa, brownouts | ✔ ~20.000 |
| **Cámara CSI OV5647 5 MP** | Barata, nativa de Pi, foco manual ajustable | Cable CSI corto (15–30 cm) obliga a poner el Pi cerca del kamado | ~20.000–35.000 |
| **Raspberry Pi Camera Module 3 (IMX708)** | Autofoco PDAF + HDR — lo mejor contra el brillo del vidrio | Importada, la más cara, sigue atada al cable CSI | ~60.000–90.000 |
| **Webcam USB (Logitech C270 o similar)** | Cable largo → el Pi vive protegido lejos del calor. Plug & play | Calidad media, poca luz mediocre | ~35.000–60.000 |
| **Cámara IP WiFi (TP-Link Tapo)** | Sensor decente, **IR nocturno integrado**, gabinete listo, RTSP, alimentación 220 V | Necesita toma cerca; C100/C200 son de interior (usar C310/C320WS para exterior) | ✔ C100 36.120–44.602 · ✔ C200 39.927 · C310/C320WS ~60.000–85.000 |

### Micros / cómputo

| Opción | Veredicto |
|---|---|
| **Raspberry Pi 3 (la que ya tenés)** | Sobra. El pipeline corre a 1 fps usando <10% de CPU. Cero costo marginal. **Es la opción por defecto.** |
| **ESP32-CAM / ESP32-S3** | Sólo si querés el sistema completo en una cajita autónoma con firmware ya hecho. |
| **Pi Zero 2 W** | Compacto y lindo, pero pagás ✔ ARS 33.000–61.261 por algo que la Pi 3 ya hace. |
| **Pi 5** | ✔ ~ARS 129.980. Innecesario acá por un orden de magnitud. |
| **Notebook/PC de casa** | Válido para la fase de prototipo: la cámara IP manda RTSP y procesás donde quieras. |

---

## 5. Arquitectura de la app

El backend es trivial en volumen: 1 lectura cada 5–30 s durante 4–12 h ≈ unos pocos miles de
puntos por cocción. Cualquier cosa aguanta.

**Recomendado (costo de operación $0/mes):**

```
[cámara] → [Pi 3: captura + CV] → SQLite ─┬─→ FastAPI + página web (Chart.js)
                                          └─→ bot de Telegram (alarmas)
                                    ↑
                          Cloudflare Tunnel (acceso remoto, gratis)
```

- **FastAPI + SQLite** en la Pi: `/api/current`, `/api/session/{id}`, y un front de una sola
  página con curva de temperatura, setpoint y banda objetivo.
- **Acceso remoto**: Cloudflare Tunnel o Tailscale, ambos gratis, sin abrir puertos.
- **Alertas**: bot de Telegram. Es lo que realmente vas a usar — "se te fue a 300 °C",
  "bajó de 100 °C, revisá el carbón", "estable hace 2 h". Instantáneo y gratis.
- **PWA**: la misma página con manifest + service worker se instala en el celular. No hace
  falta app nativa.

**Alternativas:**
- **Home Assistant** (si ya lo usás o pensás usarlo): MQTT + autodiscovery, dashboard,
  histórico y automatizaciones sin escribir backend. Es el camino más corto, y es el que
  AI-on-the-edge-device soporta nativamente.
- **Nube gestionada**: Grafana Cloud / InfluxDB Cloud free tier para el histórico, front en
  Vercel. Más piezas, mismo resultado.

---

## 6. Los cinco planes, con costos

Todos los precios en ARS. ✔ = precio verificado en comercio argentino; el resto, rango estimado.

### Plan A — "Todo en el borde": ESP32-CAM + AI-on-the-edge-device

| Ítem | ARS |
|---|---|
| ESP32-CAM AI-Thinker (OV2640) ✔ | 20.000 |
| Antena WiFi externa + pigtail | 3.000 |
| Fuente 5 V 2 A + capacitores (fix de brownout) | 8.000 |
| microSD 8–16 GB | 6.000 |
| LED blanco + difusor + resistencia | 3.000 |
| Caja IP54 + soporte/brazo | 10.000 |
| **Total** | **~50.000 (≈ USD 32)** |

**Pros:** el costo más bajo con visión, firmware ya escrito y mantenido, MQTT + HA listo, OTA.
**Contras:** la cámara más débil del lote justo en el escenario más difícil (vidrio + poca luz);
el ESP32-CAM tiene fama merecida de problemático (antena, brownout, EMI). Si el reflejo te
gana, tenés poco margen para mejorar la óptica.
**Cuándo elegirlo:** querés algo autónomo, barato, y no te molesta pelear con el hardware.

---

### Plan B — "Con lo que ya tenés": Pi 3 + webcam USB con extensión ⭐ recomendado

| Ítem | ARS |
|---|---|
| Raspberry Pi 3 (ya la tenés) | 0 |
| Webcam USB (C270 o similar) | 45.000 |
| Extensión USB activa 5 m | 15.000 |
| microSD 32 GB | 10.000 |
| Fuente 5 V 2.5 A | 10.000 |
| Caja/visera para la webcam + brazo articulado | 12.000 |
| LED blanco off-axis + fuente | 5.000 |
| **Total** | **~97.000 (≈ USD 63)** |
| *Variante mínima:* cámara CSI OV5647 en vez de webcam, Pi cerca del kamado en caja aislada | **~60.000 (≈ USD 39)** |

**La clave de este plan:** con extensión USB activa, **sólo la webcam se expone al calor y la
intemperie**; la Pi vive adentro o en un gabinete protegido. Es el plan con mejor relación
riesgo/costo, y el que más rápido itera (Python, OpenCV, todo el stack en un solo lugar).
**Contras:** hay que escribir el pipeline (que es corto) y el backend (que también).

---

### Plan C — "Cámara buena, cómputo aparte": Tapo IP + Pi 3 por RTSP

| Ítem | ARS |
|---|---|
| TP-Link Tapo C310 / C320WS (exterior, IP66) | 70.000 |
| *(alternativa interior bajo techo: Tapo C100 ✔ 36.120 / C200 ✔ 39.927)* | — |
| Raspberry Pi 3 (ya la tenés) | 0 |
| microSD 32 GB + fuente | 20.000 |
| Soporte/brazo | 8.000 |
| **Total** | **~98.000 (≈ USD 63)** · versión bajo techo con C100: **~64.000** |

**Pros:** sensor bastante mejor que cualquier módulo de maker, **IR nocturno resuelto de
fábrica** (y el IR anda muy bien sobre aguja negra en dial blanco), gabinete y garantía
incluidos, y de yapa te queda video del asado. La Pi sólo consume RTSP.
**Contras:** necesita 220 V cerca del kamado; hay que habilitar la cuenta RTSP en la app Tapo;
el IR puede reflejar en el vidrio (mitigable con ángulo).
**Cuándo elegirlo:** si el patio ya tiene toma y querés mínima pelea con óptica e iluminación.

---

### Plan D — "Compacto y prolijo": Pi Zero 2 W + Camera Module 3

| Ítem | ARS |
|---|---|
| Raspberry Pi Zero 2 W ✔ | 45.000 |
| Camera Module 3 (IMX708, autofoco + HDR) | 75.000 |
| Cable CSI Zero + microSD + fuente | 25.000 |
| Caja impresa 3D + soporte | 12.000 |
| **Total** | **~157.000 (≈ USD 102)** |

**Pros:** la mejor óptica del lote — autofoco y HDR son exactamente lo que querés contra el
brillo del bisel. Todo en una cajita montada al carro del kamado.
**Contras:** el doble de caro que el Plan B por una mejora que probablemente no necesites, y
el cómputo queda expuesto al calor.
**Cuándo elegirlo:** si los Planes B/C fallan por brillo y querés atacarlo con óptica.

---

### Plan E — Sin visión: termocupla tipo K + ESP32 *(referencia de comparación)*

| Ítem | ARS |
|---|---|
| ESP32 DevKit | 12.000 |
| Módulo MAX6675 (o MAX31855) + termocupla tipo K | 18.000 |
| Fuente + caja | 16.000 |
| **Total** | **~46.000 (≈ USD 30)** |

Hay que decirlo con todas las letras: **si el objetivo es *saber la temperatura*, esto le gana
a la visión en todo** — ±1.5 °C reales, 1 Hz, mide a nivel de parrilla (que es lo que importa
para cocinar), no le afecta la noche ni el humo, y es más barato. La sonda entra por el damper
superior o por la junta, sin perforar nada.

La visión gana si el objetivo es otro: no intervenir el kamado, leer el instrumento que ya está,
o el desafío técnico en sí.

**Recomendación concreta:** hacé el Plan E igual, aunque hagas visión. Por ARS ~30.000 te da el
*ground truth* para validar el pipeline y armar la curva domo→parrilla de una vez. Es la mejor
plata del proyecto.

---

## 7. Comparación

| | A: ESP32-CAM | B: Pi 3 + USB ⭐ | C: Tapo + Pi 3 | D: Zero 2W + Cam3 | E: Termocupla |
|---|---|---|---|---|---|
| Costo ARS | ~50.000 | ~97.000 | ~98.000 | ~157.000 | ~46.000 |
| Costo USD | ~32 | ~63 | ~63 | ~102 | ~30 |
| Calidad de imagen | Baja | Media | Alta | Muy alta | — |
| Nocturno | LED propio | LED propio | **IR de fábrica** | LED propio | N/A |
| Software a escribir | Casi nada | Todo | Casi todo | Todo | Poco |
| Resistencia al calor | Regular | **Buena** | Buena | Regular | Excelente |
| Precisión final | ±10–15 °C* | ±10–15 °C* | ±10–15 °C* | ±10–15 °C* | **±1.5 °C** |
| Riesgo del proyecto | Medio-alto | **Bajo** | Bajo | Bajo | Muy bajo |

\* Dominado por el error del propio termómetro de domo, no por la visión.

---

## 8. Plan de trabajo por fases

**Fase 0 — Validar sin comprar nada (un fin de semana, ARS 0).**
Prendé el kamado, poné el celular en un trípode improvisado y sacá 40–60 fotos del dial
durante un cocido completo, incluyendo de noche y con humo. Escribí el pipeline polar en
Python sobre ese dataset en la notebook y medí el error.
**Esta fase elimina el ~80 % del riesgo antes del primer peso gastado.** Si el reflejo del
vidrio arruina las fotos, lo vas a descubrir acá, gratis, y ahí se decide si hace falta ir a
Plan C/D o directamente a Plan E.

**Fase 1 — Hardware mínimo (1 semana).** Plan B con la Pi 3. Captura periódica a disco,
pipeline corriendo como servicio systemd, log a SQLite. Sin web todavía.

**Fase 2 — Montaje definitivo (1 fin de semana).** Brazo rígido, fiduciales pegados al bisel,
LED off-axis, visera. Calibración de dos puntos. Acá se gana o se pierde el proyecto.

**Fase 3 — App (1 semana).** FastAPI + SQLite + página con Chart.js + Cloudflare Tunnel +
bot de Telegram con alarmas de banda.

**Fase 4 — Validación (2–3 cocciones).** Termocupla del Plan E en paralelo. Curva domo→parrilla,
tabla de corrección, y reporte de error real del sistema.

---

## 9. Riesgos y mitigaciones

| Riesgo | Prob. | Mitigación |
|---|---|---|
| **Reflejo especular en el vidrio del dial** | Alta | LED off-axis 30–45°, film polarizador, visera, ángulo de cámara ~20° fuera del eje. Detectarlo en Fase 0. |
| Condensación/grasa en la lente | Media | Visera, lente apuntando levemente hacia arriba, limpieza entre cocciones. |
| La cámara se mueve → calibración rota | Media | **Fiduciales + homografía por frame.** Resuelto de raíz. |
| Electrónica cocinada por el calor | Media | Distancia ≥ 30 cm, fuera del damper, cómputo lejos (Plan B/C). |
| WiFi insuficiente en el patio | Media | Medir RSSI antes de comprar. Antena externa o repetidor. |
| Humo denso tapa el dial | Baja | Filtro temporal (mediana + rechazo de saltos) ya lo cubre. |
| Precisión final decepcionante | Alta | Es limitación del termómetro, no del sistema. Gestionarlo con expectativas + Plan E de validación. |

---

## 10. Recomendación

1. **Hacé la Fase 0 este fin de semana.** Costo cero, decide todo lo demás.
2. **Arrancá con el Plan B** (Pi 3 + webcam USB con extensión activa, ~ARS 97.000). Mejor
   relación riesgo/costo, reusa hardware que ya tenés, y el stack en Python te deja iterar rápido.
3. **Sumá el Plan E en paralelo** (~ARS 46.000) como ground truth. No es opcional si querés
   saber qué tan bien anda el sistema.
4. **Guardá el Plan C** como upgrade si la iluminación nocturna resulta ser el problema —
   el IR de fábrica de una Tapo lo resuelve sin ingeniería.

Presupuesto total recomendado: **~ARS 143.000 (≈ USD 93)**, con ARS 0 de costo mensual de operación.

---

## Fuentes

- Precios ESP32-CAM: [Starware](https://tienda.starware.com.ar/producto/modulo-wifi-bt-camara-arduino-esp32-cam-ov2640-2mp/) · [EB Electrónica](https://www.ebelectronica.com.ar/arduino/esp32-cam) · [MercadoLibre AR](https://listado.mercadolibre.com.ar/esp32-cam)
- Precios Raspberry Pi: [Siranet](https://www.siranet.com.ar/productos/raspberry-pi-zero-2w/) · [Elemon](https://www.elemon.com.ar/products/detail/code/AR681RA300) · [Arsumo](https://arsumo.com.ar/productos/raspberry-pi-5-8gb-ram-ultima-generacion-de-alto-rendimiento/) · [TodoMicro](https://www.todomicro.com.ar/RASPBERRY-PI/2780-placa-raspberry-pi-5-8gb.html)
- Precios cámaras IP: [Rosario Seguridad](https://www.rosarioseguridad.com.ar/camara-tapo-ip-wifi-interior-2mpx-c100-4829) · [Comeros](https://www.comeros.com.ar/tienda/camara-ip-tp-link-tapo-c100-wifi-1080p/) · [Aware Solutions](https://tienda.awaresolutions.com.ar/familias/tapo/)
- Termocupla/MAX6675: [Cyberofice (CABA)](https://www.cyberofice.com.ar/producto/modulo-max6675-max-6675-max-6675-termocupla-k-arduino/) · [TodoMicro](https://www.todomicro.com.ar/sensores/615-modulo-termometro-max6675-termocupla-tipo-k.html)
- Cotización: [Página 12, 12/08/2026](https://www.pagina12.com.ar/2026/08/12/dolar-blue-dolar-hoy-a-cuanto-cotizan-el-miercoles-12-de-agosto-de-2026/) · [La Nación](https://www.lanacion.com.ar/dolar-hoy/)
- Firmware: [jomjol/AI-on-the-edge-device](https://github.com/jomjol/AI-on-the-edge-device) · [docs — integración Home Assistant](https://jomjol.github.io/AI-on-the-edge-device-docs/Integration-Home-Assistant/)
- Lectura de diales: [Intel analog-gauge-reader](https://www.intel.com/content/www/us/en/developer/articles/technical/analog-gauge-reader-using-opencv.html) · [Ghostbird/Analogue-Gauge-Reader](https://github.com/Ghostbird/Analogue-Gauge-Reader) · [Under pressure (arXiv 2404.08785)](https://arxiv.org/html/2404.08785v1)
- Problemas conocidos ESP32-CAM: [Random Nerd Tutorials](https://randomnerdtutorials.com/esp32-cam-troubleshooting-guide/) · [DroneBot Workshop](https://dronebotworkshop.com/esp32-cam-intro/)
- Precisión domo vs parrilla: [Big Green Egg Forum](https://eggheadforum.com/discussion/1201373/dome-temp-vs-grate-temp) · [Pitmaster Club](https://pitmaster.amazingribs.com/forum/charcoal-kamados/big-green-egg/941963-dome-temp-vs-grid-temp) · [BBQ Brethren](https://www.bbq-brethren.com/threads/temp-difference-between-grate-and-dome-for-kamado-bbq.308997/)
