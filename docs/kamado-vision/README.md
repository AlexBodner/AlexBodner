# Kamado Vision — lectura del termómetro de domo por visión

Investigación de hardware, arquitectura y costos para leer la aguja del termómetro
analógico de un kamado con una cámara y publicar la temperatura en una app/página.

**Restricción de diseño: el nodo va a batería.** Nada de cable USB a una computadora.
Eso reordena todo: descarta la Raspberry Pi como nodo de captura, y convierte al ESP32 en
la plataforma correcta.

Contexto: Buenos Aires, agosto 2026. Cotización usada: **USD 1 ≈ ARS 1.545** (blue, 15/08/2026).
Precios ARS verificados marcados con ✔; el resto son estimaciones de rango de mercado local.

---

## 1. Lo primero: acotar el problema real

**a) El termómetro de domo es la parte imprecisa, no la visión.**
Los diales de domo tienen error típico de ±10–15 °C, y además miden en la cúpula, no en la
parrilla. La diferencia domo↔parrilla que reporta la comunidad va de ~20 °C a ~70 °C según el
cocido, el deflector y si hay comida tapando el flujo. Un pipeline de visión sobre un dial
rígidamente encuadrado te da **±1–2 °C de error de lectura**. La visión no es el cuello de
botella de precisión: el sistema va a ser tan bueno como el reloj que está leyendo.

**b) La cámara y el dial quedan rígidos entre sí.**
El mayor simplificador del proyecto. Con soporte rígido, el centro del dial, el radio y el
rango angular son constantes conocidas. El problema se reduce a *"encontrar el ángulo de la
aguja dentro de un anillo fijo"* — no hace falta detectar el círculo por frame, ni Hough, ni CNN.

**c) No necesitás video: necesitás una lectura por minuto.**
Ésta es la clave de todo el diseño a batería. Un asado es un proceso térmico lento; la
temperatura no salta en 10 segundos. Con una lectura cada 30–60 s el nodo puede **dormir el
98 % del tiempo**, y ahí es donde nace la autonomía de días en vez de horas.

**d) Es un ambiente hostil.**
Tapa a 150–300 °C, sol directo, humo graso, vapor, lluvia, y el asado es de noche.

---

## 2. El presupuesto de energía (la decisión central)

### Por qué el ESP32 y no la Raspberry Pi

Sólo el ESP32 puede **deep sleep de verdad**: 14 µA en un XIAO ESP32S3, ~5 µA en el chip
pelado. Una Raspberry Pi no tiene modo de suspensión útil — su piso es el idle, unos
80–120 mA en una Zero 2 W y ~260 mA en una Pi 3. Esa diferencia es de tres órdenes de
magnitud y define todo lo demás.

### Ciclo de trabajo, 1 lectura por minuto

| Etapa | Duración | Corriente |
|---|---|---|
| Boot + init de cámara | ~1.0 s | ~120 mA |
| Flash LED + captura | ~0.2 s | ~250 mA |
| Asociación WiFi + POST | ~2.5 s | ~180 mA (picos de 350 mA) |
| Deep sleep | 56 s | 14 µA (XIAO) · 3–8 mA (ESP32-CAM) |

→ **~0.2 mAh por lectura** en el XIAO. Corriente media: **~12 mA**.

### Autonomía comparada

| Configuración | Consumo medio | Batería | Autonomía |
|---|---|---|---|
| **XIAO ESP32S3 + deep sleep** | ~12 mA | LiPo 2000 mAh | **~7 días continuos** (≈14 asados) |
| **XIAO ESP32S3 + panel solar 2 W** | ~12 mA | LiPo 2000 + panel | **Indefinida** |
| ESP32-CAM + deep sleep | ~20 mA | 18650 2500 mAh | ~4–5 días (≈10 asados) |
| ESP32-CAM con AI-on-the-edge *(no duerme)* | ~180 mA | powerbank 10.000 | ~35 h (3 asados) |
| Pi Zero 2 W *(no puede dormir)* | ~150–300 mA | powerbank 10.000 | 20–40 h (2–3 asados) |
| Pi 3 *(no puede dormir)* | ~500 mA | powerbank 10.000 | **~13 h — un solo asado** |
| Termocupla + ESP32-C3, 1 lect./10 s | ~1.5 mA | 18650 2500 mAh | **~2 meses** |

*Un powerbank de "10.000 mAh" son 37 Wh a 3.7 V → ~6.400 mAh reales en el riel de 5 V
después de las pérdidas del booster.*

### Tres resultados contraintuitivos que salen de estos números

**1. La iluminación nocturna es prácticamente gratis.** El LED sólo prende durante los
~200 ms de la captura, sincronizado con el obturador. Aun un LED brillante de 300 mA cuesta
0.017 mAh por lectura — **1 mAh por hora**. El miedo a "la luz me come la batería" no aplica
cuando duty-cicleás. El ESP32-CAM ya trae el flash en GPIO4 justo para esto.

**2. Conviene mandar el JPEG y hacer la visión en el servidor, no en el nodo.** Lo que domina
el consumo es la **asociación WiFi** (~2 s), no el payload: un JPEG de 640×480 son 30–50 KB y
agrega ~0.3 s. La diferencia energética es ruido, y a cambio desarrollás el pipeline en
Python sobre la notebook, guardás las imágenes para depurar y podés reprocesar el histórico
cuando mejores el algoritmo. Ganás muchísimo por casi nada.
> Truco: guardá BSSID, canal e IP estática en memoria RTC. La reconexión baja de ~2 s a
> ~400 ms y la energía activa cae unas 4×.

**3. Nunca uses un powerbank USB con un ESP32 que duerme.** Los powerbanks cortan la salida
cuando el consumo baja de 50–100 mA durante 15–60 s — exactamente lo que hace un nodo en deep
sleep. Se apaga y no vuelve. Va **LiPo o 18650 directo**, sin powerbank en el medio. Para la
Pi Zero, si igual vas por ahí, necesitás un banco con modo *trickle* (los Anker se activan con
doble click) o directamente un UPS HAT.

### Cadencia adaptativa (opcional, pero linda)

No hace falta muestrear cada minuto las 24 h. El nodo despierta cada 15 min; si la lectura
supera los 50 °C entra en "modo cocción" y pasa a 60 s. Cuando baja, vuelve a dormir largo.
El consumo fuera de asado cae a casi cero y la autonomía pasa de días a meses. Con solar es
redundante, pero sin solar vale la pena.

---

## 3. Restricciones físicas del montaje

| Restricción | Implicancia de diseño |
|---|---|
| El bisel quema; la tapa irradia | Cámara a ≥ 25–30 cm, fuera de la columna térmica del damper superior. |
| **La batería odia el calor** | LiPo y Li-ion no deben pasar de ~45–60 °C. Batería **abajo en el carro, a la sombra**, nunca sobre la tapa ni en una caja negra sellada al sol de Buenos Aires. Si te preocupa, LiFePO4 tolera mejor. |
| Vidrio del dial → reflejo especular | El enemigo n.º 1. Iluminación **off-axis** (LED a 30–45° del eje óptico, nunca frontal), visera, y opcionalmente film polarizador. |
| Vapor y grasa condensan en la lente | Parasol chico, lente apuntando levemente hacia arriba, limpieza entre cocciones. |
| Se cocina de noche | LED sincronizado al obturador (ver §2). El IR también anda bien sobre aguja negra en dial blanco. |
| Exterior: lluvia y sol | IP54 mínimo. Prensaestopa para el cable del panel solar. |
| WiFi en el patio | Medir RSSI *antes* de comprar. Antena externa o repetidor si hace falta. |

**Truco de robustez, desde el día 1:** pegá tres marcadores fiduciales alrededor del bisel
—círculos negros sobre vinilo blanco, o un ArUco chico—. Si alguien mueve la cámara o el
soporte vibra, el servidor recalcula la homografía por frame y la calibración no se rompe
nunca. Cuesta $0 y elimina la falla más probable del proyecto.

---

## 4. El pipeline de visión

Corre en el servidor (la Pi 3 enchufada adentro), sobre los JPEG que manda el nodo.
Con montaje rígido más fiduciales, esto alcanza — unas 80 líneas de OpenCV:

1. **Warp**: detectar los 3 fiduciales → homografía → dial siempre en la misma pose canónica.
2. **Unwrap polar**: para cada ángulo θ ∈ [0, 360), muestrear los radios r ∈ [r_min, r_max] y
   promediar intensidad. Da un vector de 360 valores.
3. **Argmax de oscuridad** con interpolación sub-grado (centroide del pico) → ángulo de la
   aguja. La ambigüedad aguja/contrapeso se resuelve restringiendo al barrido válido del dial.
4. **Mapeo lineal** ángulo → °C, calibrado con dos marcas impresas. Los diales de kamado son lineales.
5. **Filtro temporal**: mediana móvil de N lecturas + rechazo de saltos > X °C/min. Mata los
   falsos por humo denso o reflejo transitorio.

**Precisión esperable:** con el dial ocupando ~300 px y un barrido de ~270° para 40–400 °C, la
resolución angular de ~0.5° equivale a ~0.7 °C. El error queda dominado por el ancho de la
aguja, no por el algoritmo: **±1–2 °C**.

**Si no querés escribir el pipeline:** [AI-on-the-edge-device](https://github.com/jomjol/AI-on-the-edge-device)
(jomjol) es firmware para ESP32-CAM que ya trae red para agujas analógicas, web UI, MQTT con
autodiscovery de Home Assistant, REST y OTA. **Pero no hace deep sleep** — corre un servidor
web permanente a ~180 mA. Es la opción correcta si el nodo va enchufado, y la equivocada si va
a batería.

Referencias si lo armás vos: el sample `analog-gauge-reader` de Intel,
`Ghostbird/Analogue-Gauge-Reader`, y el paper *Under pressure: learning-based analog gauge
reading in the wild* (arXiv 2404.08785).

---

## 5. Arquitectura

La batería obliga a partir el sistema en dos, y eso resulta ser lo correcto igual:

```
  ── AFUERA, a batería ──────────┐        ── ADENTRO, enchufado ──────────────────
                                 │
  [XIAO ESP32S3 + LiPo + solar]  │        [Raspberry Pi 3]
   deep sleep 56 s               │──WiFi──▶ pipeline CV (OpenCV)
   despierta → flash → foto      │         SQLite  ─┬─▶ FastAPI + página (PWA)
   POST del JPEG → duerme        │                  └─▶ bot de Telegram (alarmas)
                                 │                        ▲
                                 │              Cloudflare Tunnel (acceso remoto)
```

**La Pi 3 que ya tenés no desaparece — cambia de rol.** Deja de ser el nodo de captura (donde
la batería la mata) y pasa a ser la estación base enchufada adentro: corre el pipeline, la
base de datos, la web y las alertas. Es exactamente el trabajo para el que sirve, y encima
resuelve el problema de la señal WiFi: el nodo sólo tiene que llegar al router, no a internet.

- **FastAPI + SQLite** en la Pi: `/api/current`, `/api/session/{id}`, y un front de una página
  con curva de temperatura, setpoint y banda objetivo.
- **Acceso remoto**: Cloudflare Tunnel o Tailscale, gratis, sin abrir puertos.
- **Alertas**: bot de Telegram. Es lo que realmente vas a usar — "se te fue a 300 °C",
  "bajó de 100, revisá el carbón", "estable hace dos horas".
- **PWA**: la misma página con manifest + service worker se instala en el celular. No hace
  falta app nativa.
- **Alternativa sin Pi**: el nodo postea a Supabase / InfluxDB Cloud free tier y el front va en
  Vercel. Menos piezas en casa, más dependencias externas.
- **Alternativa Home Assistant**: si ya lo usás, MQTT + autodiscovery y no escribís backend.

---

## 6. Los planes, con costos

Todos los precios en ARS. ✔ = verificado en comercio argentino; el resto, rango estimado.

### Estación base — común a todos los planes

| Ítem | ARS |
|---|---|
| Raspberry Pi 3 (ya la tenés) | 0 |
| microSD 32 GB + fuente | 20.000 |
| **Subtotal** | **20.000** |

---

### Plan A — XIAO ESP32S3 Sense + LiPo + solar ⭐ recomendado

| Ítem | ARS |
|---|---|
| Seeed XIAO ESP32S3 Sense (OV3660, 8 MB PSRAM) | 55.000 |
| LiPo 1S 2000 mAh + conector JST | 12.000 |
| Panel solar 5–6 V 2 W | 15.000 |
| Módulo de carga solar (CN3065) o TP4056 ✔2.648 + diodo | 5.000 |
| LED blanco + resistencia (sincronizado al obturador) | 3.000 |
| Caja IP54 + brazo/soporte | 15.000 |
| **Total** | **~105.000 (≈ USD 68)** |

**Por qué gana con la restricción de batería:** es literalmente la placa diseñada para esto.
**14 µA en deep sleep**, cargador LiPo integrado en la placa (no necesitás TP4056 aparte),
USB-C, 8 MB de PSRAM, y la cámara OV3660 es mejor que la OV2640 del ESP32-CAM. Con el panel
solar es un nodo de *poner y olvidarse*: a 12 mA de media consume ~1.1 Wh/día y un panel de
2 W en Buenos Aires cosecha unos 4 Wh/día útiles. Cuatro veces de margen.

**En contra:** hay que escribir el firmware (captura + deep sleep + POST — es corto, pero es
tuyo). Placa importada, precio local a confirmar en [Starware](https://tienda.starware.com.ar/producto/modulo-camara-mic-programable-seeed-xiao-esp32-s3-sense-cam-mic-wifi-ble-sku-3139/).

---

### Plan B — ESP32-CAM + 18650 (económico)

| Ítem | ARS |
|---|---|
| ESP32-CAM AI-Thinker (OV2640) ✔ | 20.000 |
| Antena WiFi externa + pigtail | 3.000 |
| Celda 18650 2500 mAh + portapilas | 12.000 |
| Módulo TP4056 USB-C ✔2.648 | 3.000 |
| Regulador + capacitores (fix de brownout) | 5.000 |
| microSD 8 GB | 6.000 |
| LED blanco + difusor | 3.000 |
| Caja IP54 + brazo | 15.000 |
| **Total** | **~67.000 (≈ USD 43)** |

**A favor:** el más barato con visión, y con deep sleep propio llega a ~4–5 días por carga.
**En contra:** el regulador AMS1117 de la placa consume 3–8 mA aun dormido — no es catástrofe,
pero es 200–500× peor que el XIAO. Sumale antena floja, brownouts y EMI. Y la OV2640 con foco
fijo es la peor cámara del lote justo en el escenario óptico más difícil.
**Ojo:** si le ponés el firmware AI-on-the-edge, se acabó la batería — ese firmware no duerme.

---

### Plan C — Pi Zero 2 W + Camera Module 3 + powerbank

| Ítem | ARS |
|---|---|
| Raspberry Pi Zero 2 W ✔ | 45.000 |
| Camera Module 3 (IMX708, autofoco + HDR) | 75.000 |
| Powerbank 10.000 mAh con modo *trickle* ✔20.495–61.600 | 40.000 |
| microSD + cable CSI Zero | 20.000 |
| Caja + soporte | 15.000 |
| **Total** | **~195.000 (≈ USD 126)** |

**A favor:** la mejor óptica del lote por lejos. Autofoco PDAF y HDR son exactamente lo que
querés contra el brillo del bisel.
**En contra:** el doble de caro, **y no puede dormir**. 20–40 h por carga, o sea recargar cada
dos o tres asados. Además el corte abrupto de energía corrompe la SD: hay que montar el rootfs
read-only con overlayfs o poner un UPS HAT con apagado seguro. Y el powerbank puede cortar por
bajo consumo.
**Cuándo:** sólo si los planes A/B fallan por brillo y necesitás atacar el problema con óptica.

---

### Plan D — Cámara IP a batería (Tapo C400 / C420 / C425) ❌ descartar

Parece la respuesta obvia —batería, panel solar, IP66, visión nocturna, 180 días— y **no
sirve para esto**. Dos razones que la matan:

1. **No exponen RTSP.** TP-Link confirma que los modelos a batería no soportan RTSP/ONVIF
   precisamente por el diseño de bajo consumo. No hay forma de sacarles el frame.
2. **Sólo graban por detección de movimiento.** Una aguja que se mueve un grado por minuto no
   dispara el PIR. La cámara se queda dormida justo cuando la necesitás.

Lo anoto para que no pierdas un fin de semana y ARS 100.000 descubriéndolo. Las Tapo **con
cable** (C100 ✔36.120, C200 ✔39.927, C310/C320WS ~70.000) sí dan RTSP continuo — pero eso es
volver al cable de alimentación.

---

### Plan E — Sin visión: termocupla tipo K + ESP32-C3 a batería

| Ítem | ARS |
|---|---|
| ESP32-C3 SuperMini (más eficiente que el DevKit clásico) | 12.000 |
| Módulo MAX31855 + termocupla tipo K | 18.000 |
| 18650 + TP4056 + portapilas | 15.000 |
| Caja | 8.000 |
| **Total** | **~53.000 (≈ USD 34)** |

**A batería es todavía mejor que con cable.** Sin cámara, sin flash, sin JPEG: el ciclo activo
es de ~300 ms y el consumo medio queda en ~1.5 mA. **Dos meses con una sola 18650**, incluso
midiendo cada 10 segundos.

Y hay que decirlo con todas las letras: **si el objetivo es saber la temperatura, esto le gana
a la visión en todo.** ±1.5 °C reales, mide a nivel de parrilla —que es lo que importa para
cocinar—, no le afecta la noche ni el humo ni el reflejo, y es más barato. La sonda entra por
el damper superior o por la junta, sin perforar nada.

La visión gana si el objetivo es otro: no intervenir el kamado, leer el instrumento que ya
está, o el desafío técnico en sí.

**Recomendación concreta:** hacelo igual, aunque hagas visión. Por ARS 53.000 te da el
*ground truth* para validar el pipeline y armar la curva domo→parrilla de una vez. Es la mejor
plata del proyecto.

---

## 7. Comparación

| | A: XIAO S3 | B: ESP32-CAM | C: Zero 2 W | D: Tapo batería | E: Termocupla |
|---|---|---|---|---|---|
| Costo ARS (+ base 20.000) | 105.000 | 67.000 | 195.000 | — | 53.000 |
| Costo USD | 68 | 43 | 126 | — | 34 |
| **Autonomía** | **7 días / ∞ con solar** | 4–5 días | 20–40 h | — | **2 meses** |
| Deep sleep | 14 µA | 3–8 mA | ✗ imposible | — | 14 µA |
| Calidad de imagen | Media-alta | Baja | Muy alta | — | — |
| Nocturno | LED sincronizado | LED sincronizado | LED sincronizado | — | N/A |
| Software a escribir | Firmware + pipeline | Firmware + pipeline | Todo | — | Poco |
| Precisión final | ±10–15 °C* | ±10–15 °C* | ±10–15 °C* | — | **±1.5 °C** |
| Riesgo | Bajo | Medio | Medio-alto | **No funciona** | Muy bajo |

\* Dominado por el error del propio termómetro de domo, no por la visión.

---

## 8. Plan de trabajo por fases

**Fase 0 — Validar sin comprar nada (un fin de semana, ARS 0).**
Prendé el kamado, poné el celular en un trípode improvisado y sacá 40–60 fotos del dial
durante un cocido completo, incluyendo de noche y con humo. Escribí el pipeline polar en
Python sobre ese dataset en la notebook y medí el error.
**Elimina el ~80 % del riesgo antes del primer peso gastado.** Si el reflejo del vidrio
arruina las fotos, lo descubrís acá, gratis.

**Fase 1 — Estación base (2 días, ARS 20.000).**
Pi 3 adentro con un endpoint `POST /frame` que guarda el JPEG a disco y corre el pipeline.
Probalo mandándole imágenes desde la notebook. Todavía no hay nodo.

**Fase 2 — Nodo a batería (1 semana).**
Firmware del XIAO: deep sleep con timer, wake → flash → captura → POST → sleep. Medir el
consumo real con un USB tester o un shunt y comparar con la tabla de §2. Ajustar la cadencia.

**Fase 3 — Montaje definitivo (1 fin de semana).**
Brazo rígido, fiduciales pegados al bisel, LED off-axis, visera, batería a la sombra abajo,
panel solar orientado al norte. Calibración de dos puntos. **Acá se gana o se pierde el proyecto.**

**Fase 4 — App (1 semana).**
Página con la curva en vivo, Cloudflare Tunnel y bot de Telegram con alarmas de banda.

**Fase 5 — Validación (2–3 cocciones).**
Termocupla del Plan E en paralelo. Curva domo→parrilla, tabla de corrección y error real
del sistema.

---

## 9. Riesgos y mitigaciones

| Riesgo | Prob. | Mitigación |
|---|---|---|
| **Reflejo especular en el vidrio del dial** | Alta | LED off-axis 30–45°, film polarizador, visera, cámara ~20° fuera del eje. Se detecta en Fase 0. |
| **Powerbank corta por bajo consumo** | Alta *(si usás powerbank)* | No usar powerbank con un nodo que duerme. LiPo/18650 directo. |
| **Batería degradada por calor** | Media | Batería abajo en el carro, a la sombra, lejos de la tapa. Caja ventilada, no negra sellada. |
| La cámara se mueve y rompe la calibración | Media | Fiduciales + homografía por frame. Resuelto de raíz. |
| Condensación o grasa en la lente | Media | Visera, lente levemente hacia arriba, limpieza entre cocciones. |
| WiFi insuficiente en el patio | Media | Medir RSSI antes de comprar. Antena externa o repetidor. El nodo sólo debe llegar al router. |
| Corrupción de SD por corte abrupto *(planes con Pi)* | Media | rootfs read-only con overlayfs, o UPS HAT con apagado seguro. |
| Humo denso tapa el dial | Baja | El filtro temporal (mediana + rechazo de saltos) ya lo cubre. |
| Precisión final decepcionante | Alta | Es limitación del termómetro, no del sistema. Se gestiona con expectativas + Plan E. |

---

## 10. Recomendación

1. **Hacé la Fase 0 este fin de semana.** Costo cero, decide todo lo demás.
2. **Plan A** — XIAO ESP32S3 Sense + LiPo + panel solar, ≈ ARS 105.000. Es la única
   configuración que da autonomía real e indefinida, y la placa está hecha para exactamente
   este caso de uso.
3. **Estación base con la Pi 3 que ya tenés**, ≈ ARS 20.000. Cambia de rol: deja de ser el
   nodo y pasa a ser el servidor enchufado adentro.
4. **Sumá el Plan E en paralelo**, ≈ ARS 53.000, como ground truth. No es opcional si querés
   saber qué tan bien anda el sistema.
5. **Si el precio local del XIAO no cierra**, el Plan B (ESP32-CAM + 18650, ARS 67.000) hace
   lo mismo con más pelea y menos autonomía.

**Presupuesto recomendado: ~ARS 178.000 (≈ USD 115)**, con ARS 0 de costo mensual.
Versión económica con Plan B: **~ARS 140.000 (≈ USD 91)**.

---

## Fuentes

- ESP32-CAM: [Starware](https://tienda.starware.com.ar/producto/modulo-wifi-bt-camara-arduino-esp32-cam-ov2640-2mp/) · [EB Electrónica](https://www.ebelectronica.com.ar/arduino/esp32-cam) · [MercadoLibre AR](https://listado.mercadolibre.com.ar/esp32-cam)
- XIAO ESP32S3 Sense: [Starware (AR)](https://tienda.starware.com.ar/producto/modulo-camara-mic-programable-seeed-xiao-esp32-s3-sense-cam-mic-wifi-ble-sku-3139/) · [Seeed Studio](https://www.seeedstudio.com/XIAO-ESP32S3-Sense-p-5639.html) · [wiki: 14 µA deep sleep](https://wiki.seeedstudio.com/es/xiao_esp32s3_getting_started/) · [atomic14: specs y gotchas](https://www.atomic14.com/esp32/boards/xiao-esp32-s3/)
- Consumo y deep sleep ESP32: [DeepBlue Embedded](https://deepbluembedded.com/esp32-sleep-modes-power-consumption/) · [DroneBot Workshop](https://dronebotworkshop.com/esp32-low-power/) · [SolderHub](https://solderhub.com/articles/esp32-deep-sleep-battery-life-guide)
- Powerbanks que cortan por bajo consumo: [Hackaday](https://hackaday.com/2024/03/05/a-simple-hack-for-running-low-power-gear-from-a-usb-battery-pack/) · [Raspberry Pi Forums](https://forums.raspberrypi.com/viewtopic.php?t=345977)
- Cámaras a batería sin RTSP: [Tapo FAQ — RTSP/ONVIF](https://www.tapo.com/us/faq/724/) · [TP-Link — ver cámaras por RTSP](https://www.tp-link.com/us/support/faq/2680/) · [Tapo FAQ — cámaras a batería](https://www.tapo.com/us/faq/370/)
- Raspberry Pi: [Siranet](https://www.siranet.com.ar/productos/raspberry-pi-zero-2w/) · [Elemon](https://www.elemon.com.ar/products/detail/code/AR681RA300) · [Arsumo](https://arsumo.com.ar/productos/raspberry-pi-5-8gb-ram-ultima-generacion-de-alto-rendimiento/)
- Cámaras IP con cable: [Rosario Seguridad](https://www.rosarioseguridad.com.ar/camara-tapo-ip-wifi-interior-2mpx-c100-4829) · [Comeros](https://www.comeros.com.ar/tienda/camara-ip-tp-link-tapo-c100-wifi-1080p/) · [Aware Solutions](https://tienda.awaresolutions.com.ar/familias/tapo/)
- Carga y baterías: [Nakama TP4056 USB-C ✔](https://www.nakamaelectronica.com.ar/productos/modulo-cargador-18650-tp4056-con-usb-c/) · [Starware TP4056](https://tienda.starware.com.ar/producto/modulo-carga-generico-tp4056-micro-usb-37v-1a-bateria-18650-3/) · [paneles solares mini AR](https://listado.mercadolibre.com.ar/panel-solar-6v-5w)
- Termocupla/MAX6675: [Cyberofice (CABA)](https://www.cyberofice.com.ar/producto/modulo-max6675-max-6675-max-6675-termocupla-k-arduino/) · [TodoMicro](https://www.todomicro.com.ar/sensores/615-modulo-termometro-max6675-termocupla-tipo-k.html)
- Cotización: [Página 12, 12/08/2026](https://www.pagina12.com.ar/2026/08/12/dolar-blue-dolar-hoy-a-cuanto-cotizan-el-miercoles-12-de-agosto-de-2026/) · [La Nación](https://www.lanacion.com.ar/dolar-hoy/)
- Firmware: [jomjol/AI-on-the-edge-device](https://github.com/jomjol/AI-on-the-edge-device) · [integración Home Assistant](https://jomjol.github.io/AI-on-the-edge-device-docs/Integration-Home-Assistant/)
- Lectura de diales: [Intel analog-gauge-reader](https://www.intel.com/content/www/us/en/developer/articles/technical/analog-gauge-reader-using-opencv.html) · [Ghostbird](https://github.com/Ghostbird/Analogue-Gauge-Reader) · [Under pressure (arXiv 2404.08785)](https://arxiv.org/html/2404.08785v1)
- Domo vs parrilla: [Big Green Egg Forum](https://eggheadforum.com/discussion/1201373/dome-temp-vs-grate-temp) · [Pitmaster Club](https://pitmaster.amazingribs.com/forum/charcoal-kamados/big-green-egg/941963-dome-temp-vs-grid-temp) · [BBQ Brethren](https://www.bbq-brethren.com/threads/temp-difference-between-grate-and-dome-for-kamado-bbq.308997/)
