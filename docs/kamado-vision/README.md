# Kamado Vision — lectura del termómetro de domo por visión

Leer la aguja del termómetro analógico de un kamado con una cámara y mandar la temperatura
al celular.

**Restricciones de diseño:**
1. El nodo va **a batería**. Nada de cable a una computadora.
2. **Todo se procesa en el ESP32.** Cero servidores, cero backend que mantener.

Contexto: Buenos Aires, agosto 2026. Cotización usada: **USD 1 ≈ ARS 1.545** (blue, 15/08/2026).
Precios ARS verificados marcados con ✔; el resto son estimaciones de rango de mercado local.

---

## 1. La buena noticia: el ESP32 procesa esto de sobra

El pipeline correcto para este problema no es una CNN ni OpenCV: es **~5 ms de aritmética
entera**. Corre en el ESP32 con margen de sobra, y sin librerías.

### Por qué es tan barato

Tres cosas se combinan:

- **La cámara entrega gris directo.** El driver `esp32-camera` soporta `PIXFORMAT_GRAYSCALE`.
  No hay JPEG que decodificar: el buffer que llega del sensor ya es el que vas a procesar.
- **La cámara y el dial quedan rígidos entre sí.** El centro, el radio y el barrido angular
  son constantes conocidas. No hace falta detectar círculos, ni Hough, ni homografías.
- **Sólo cambia el ángulo de la aguja.** Todo el problema es encontrar la dirección más
  oscura dentro de un anillo fijo.

### El algoritmo completo

```c
#define N_ANG 360
#define N_RAD  40

static uint32_t lut[N_ANG * N_RAD];   // ~57 KB en PSRAM

// Una sola vez, en setup(): offsets de píxel para cada (ángulo, radio)
void build_lut(int cx, int cy, int r0, int r1, int W) {
  for (int a = 0; a < N_ANG; a++) {
    float th = (a * 2.0f * (float)M_PI) / N_ANG;
    float s = sinf(th), c = cosf(th);
    for (int r = 0; r < N_RAD; r++) {
      int rr = r0 + (r * (r1 - r0)) / N_RAD;
      lut[a * N_RAD + r] = (uint32_t)((cy - (int)(rr * c)) * W + (cx + (int)(rr * s)));
    }
  }
}

// Por frame: 14.400 lecturas + sumas → menos de 1 ms a 240 MHz
float needle_angle(const uint8_t *img) {
  uint32_t prof[N_ANG];
  for (int a = 0; a < N_ANG; a++) {
    uint32_t acc = 0;
    const uint32_t *p = &lut[a * N_RAD];
    for (int r = 0; r < N_RAD; r++) acc += img[p[r]];
    prof[a] = acc;                          // la aguja es el mínimo de brillo
  }
  int best = argmin_in_sweep(prof);         // restringido al barrido válido del dial
  return best + parabolic_subbin(prof, best);   // interpolación sub-grado
}

// Los diales de kamado son lineales: dos puntos de calibración y listo
float temp_c(float ang) { return T0 + (ang - A0) * (T1 - T0) / (A1 - A0); }
```

Son unas 150–200 líneas contando la captura, el filtro y el envío.

### Presupuesto de cómputo

| Etapa | Costo |
|---|---|
| Captura del frame (lectura del sensor) | ~60–100 ms |
| Centrado: umbral + centroide del disco claro, ROI 300×300 | ~2 ms |
| Unwrap polar + argmax (14.400 muestras vía LUT) | **<1 ms** |
| Mapeo a °C + filtro temporal | µs |

**La visión cuesta menos del 5 % del tiempo que tarda la cámara en leer el frame.**
Procesar a bordo no es un compromiso: es más simple que mandarlo a un servidor.

**RAM:** frame de 640×480 en gris son 307 KB, más 57 KB de LUT. Los 8 MB de PSRAM del
XIAO ESP32S3 lo tragan sin despeinarse.

### Robustez sin homografías

En la versión con servidor la idea era pegar marcadores fiduciales y resolver una homografía
por frame. A bordo hay algo más barato y casi tan bueno:

1. **Auto-centrado.** El dial es un disco claro sobre fondo más oscuro. Umbral + centroide te
   da el centro en cada frame, en ~2 ms y 15 líneas. Absorbe golpes y vibración.
2. **Chequeo de cordura.** Verificá que las marcas oscuras del dial caigan en los ángulos
   esperados dentro del perfil polar. Si el patrón no coincide, mandá `descalibrado` en vez
   de un número inventado. Diez líneas, y es la diferencia entre un sistema que falla ruidoso
   y uno que miente en silencio.

### Precisión

Con el dial ocupando ~300 px y un barrido de ~270° para 40–400 °C, la resolución angular de
~0.5° equivale a ~0.7 °C. El error queda dominado por el ancho de la aguja: **±1–2 °C**.

Pero ojo con la expectativa: **el termómetro de domo es la parte imprecisa, no la visión.**
Los diales de domo tienen error típico de ±10–15 °C y miden en la cúpula, no en la parrilla —
la diferencia domo↔parrilla que reporta la comunidad va de 20 a 70 °C. El sistema va a ser
tan bueno como el reloj que está leyendo.

---

## 2. A dónde van los datos, sin montar nada

El nodo postea a un servicio IoT gratuito y vos mirás desde el celular. Ninguna de estas
opciones requiere que mantengas infraestructura.

| Servicio | Free tier | Qué te da | Para este caso |
|---|---|---|---|
| **Blynk** | 20 datastreams por template, **30.000 mensajes cada 30 días** | **App nativa** iOS/Android con widgets de gauge y gráfico, notificaciones push | ⭐ Lo más parecido a "una app" sin programarla |
| **Adafruit IO** | 10 feeds, 5 dashboards, 30 datos/min, **30 días de historial** | Dashboard web, MQTT y REST, triggers a webhooks | Buena alternativa web. Alertas por mail/SMS son de pago |
| **ThingSpeak** | 4 canales, 3M mensajes/año, mínimo 15 s entre updates | Gráficos, análisis con MATLAB | El más generoso en cuota, el más feo |
| **Bot de Telegram** | Sin límites prácticos | Alertas push y **foto del dial a pedido** | ⭐ Sumalo a cualquiera de los anteriores |

### La recomendación

**Blynk para la app + un bot de Telegram para las alertas y las fotos.** Blynk te da la curva
y el gauge en el celular sin escribir una línea de front. El bot de Telegram lo agregás en 30
líneas ([UniversalTelegramBot](https://github.com/witnessmenow/Universal-Arduino-Telegram-Bot)
está probadísimo en ESP32-CAM) y te sirve para dos cosas que Blynk no hace tan bien: avisarte
*"se te fue a 300 °C"* con un mensaje que leés sin abrir nada, y responder a `/foto` con la
imagen real del dial — que es tu única ventana de depuración cuando algo no cierra.

### Ojo con la cuota de Blynk

30.000 mensajes cada 30 días son **1.000 por día**, o sea uno cada 86 segundos. Si mandás
1 lectura por minuto las 24 h te pasás. La cadencia adaptativa (§3) lo resuelve sola: fuera
de asado, una lectura cada 15 min son 96 mensajes/día; ocho asados de 12 h al mes suman
~5.760. Total ~8.600 de 30.000. Cómodo.

Es un caso lindo donde la optimización de batería y la de cuota son la misma.

---

## 3. El presupuesto de energía

### Por qué ESP32 y no Raspberry Pi

Sólo el ESP32 puede hacer **deep sleep de verdad**: 14 µA en un XIAO ESP32S3. Una Raspberry Pi
no tiene modo de suspensión útil — su piso es el idle, ~80–120 mA en una Zero 2 W y ~260 mA en
una Pi 3. Tres órdenes de magnitud. **Con esta arquitectura la Pi 3 no participa en absoluto:
queda libre para otra cosa.**

### Ciclo de trabajo

| Etapa | Duración | Corriente |
|---|---|---|
| Boot + init de cámara | ~1.0 s | ~120 mA |
| Flash LED + captura | ~0.2 s | ~250 mA |
| **Procesamiento (todo el CV)** | **~5 ms** | ~50 mA |
| Guardar JPEG a microSD | ~20 ms | ~80 mA |
| Conexión + envío de ~20 bytes | ~1.5 s | ~180 mA (picos de 350 mA) |
| Deep sleep | resto | 14 µA (XIAO) · 3–8 mA (ESP32-CAM) |

→ **~0.13 mAh por lectura.** Corriente media a 1 lectura/min: **~8 mA**.

### Autonomía

| Configuración | Consumo medio | Batería | Autonomía |
|---|---|---|---|
| **XIAO ESP32S3, 1 lect./min** | ~8 mA | LiPo 2000 mAh | **~10 días** (≈20 asados) |
| **XIAO ESP32S3 + cadencia adaptativa** | ~1 mA fuera de asado | LiPo 2000 mAh | **~2 meses** |
| **XIAO ESP32S3 + panel solar 2 W** | ~8 mA | LiPo + panel | **Indefinida** |
| ESP32-CAM + deep sleep | ~15 mA | 18650 2500 mAh | ~6 días |
| ESP32-CAM con AI-on-the-edge *(no duerme)* | ~180 mA | powerbank 10.000 | ~35 h |
| Termocupla + ESP32-C3, 1 lect./10 s | ~1.5 mA | 18650 2500 mAh | ~2 meses |

### Sobre el ahorro de mandar sólo el número

Sí ahorra, pero **menos de lo que parece**: mandar 20 bytes en vez de un JPEG de 40 KB baja el
tiempo de radio de ~2.5 s a ~1.5 s, y esos 1.5 s son casi todos **asociación WiFi**, no
transferencia. El consumo medio pasa de ~12 mA a ~8 mA.

Dicho derecho: **la razón para procesar a bordo es la simplicidad, no la batería.** Te ahorra
un servidor entero, que es mucho más valioso que 4 mA.

> Truco: guardá BSSID, canal e IP estática en memoria RTC. La asociación baja de ~1.5 s a
> ~400 ms y el consumo activo cae otro tanto.

### Tres cosas más que salen de estos números

**1. La iluminación nocturna es prácticamente gratis.** El LED sólo prende durante los ~200 ms
de la captura, sincronizado con el obturador. Aun un LED de 300 mA cuesta 0.017 mAh por
lectura — **1 mAh por hora**. El ESP32-CAM ya trae el flash en GPIO4 justo para esto.

**2. Nunca uses un powerbank USB con un nodo que duerme.** Los powerbanks cortan la salida
cuando el consumo baja de 50–100 mA durante 15–60 s — exactamente lo que hace un ESP32 en deep
sleep. Se apaga y no vuelve. Va **LiPo o 18650 directo**.

**3. Cadencia adaptativa.** El nodo despierta cada 15 min; si la lectura supera los 50 °C entra
en "modo cocción" y pasa a 60 s. Fuera de asado el consumo cae a casi cero. Resuelve la
autonomía y la cuota de Blynk de un saque.

---

## 4. Qué perdés al no tener servidor (y cómo taparlo)

Vale la pena ser honesto: procesar a bordo tiene costos reales. Casi todos los tapa **la
microSD**, que ambas placas ya traen.

| Lo que perdés | Cómo lo tapás |
|---|---|
| **No hay imágenes para depurar.** Si el número sale mal, no sabés por qué. | Guardá el JPEG a la microSD en cada lectura: ~20 ms y energía despreciable. Cuando algo no cierra, sacás la tarjeta. Y `/foto` en Telegram te da la vista en vivo. |
| **No podés reprocesar el histórico** cuando mejorás el algoritmo. | El archivo de JPEGs en la SD *es* tu dataset. Lo bajás a la notebook y reprocesás offline. |
| **La calibración vive en el firmware**: cambiarla implica reflashear. | Hacé la Fase 0 en la notebook y hardcodeá las constantes. Si te encontrás reflasheando seguido, agregá un modo AP con una página de config que guarde en NVS (~100 líneas). |
| **El historial dura lo que dure el free tier** (30 días en Adafruit IO). | La SD guarda todo desde siempre. El servicio en la nube es la vista, no el archivo. |

**La microSD es la contraparte del servidor.** Es lo que hace que "sin servidor" no signifique
"a ciegas".

---

## 5. Arquitectura

```
   ── AFUERA, a batería ────────────────────┐
                                            │
   [XIAO ESP32S3 Sense]                     │
    despierta cada 60 s                     │        ┌──────────────┐      ┌──────────┐
    flash → foto en gris                    │──WiFi──▶  Blynk       │─────▶│ tu       │
    unwrap polar → ángulo → °C  (~5 ms)     │  20 B  │  (free tier) │      │ celular  │
    manda el número → duerme                │        └──────────────┘      └──────────┘
         │                                  │        ┌──────────────┐           ▲
         └──▶ [microSD] archivo de JPEGs    │──HTTPS─▶  Telegram    │───────────┘
              (depuración y dataset)        │        │  bot         │  alertas + /foto
   ─────────────────────────────────────────┘        └──────────────┘
```

Piezas que tenés que mantener: **ninguna.** Ni base de datos, ni backend, ni túnel, ni
certificados, ni una Raspberry prendida las 24 h.

---

## 6. Restricciones físicas del montaje

| Restricción | Implicancia de diseño |
|---|---|
| El bisel quema; la tapa irradia | Cámara a ≥ 25–30 cm, fuera de la columna térmica del damper superior. |
| **La batería odia el calor** | LiPo y Li-ion no deben pasar de ~45–60 °C. Batería **abajo en el carro, a la sombra** — nunca sobre la tapa ni en una caja negra sellada al sol. LiFePO4 tolera mejor si te preocupa. |
| **Vidrio del dial → reflejo especular** | El enemigo n.º 1. Iluminación **off-axis** (LED a 30–45° del eje óptico, nunca frontal), visera, y opcionalmente film polarizador. |
| Vapor y grasa condensan en la lente | Parasol chico, lente apuntando levemente hacia arriba, limpieza entre cocciones. |
| Se cocina de noche | LED sincronizado al obturador. |
| Exterior: lluvia y sol | IP54 mínimo. Prensaestopa para el cable del panel solar. |
| WiFi en el patio | Medir RSSI *antes* de comprar. El nodo tiene que llegar al router. |

---

## 7. Los planes, con costos

Todos los precios en ARS. ✔ = verificado en comercio argentino; el resto, rango estimado.
**Ya no hay estación base: no hay nada que sumar.**

### Plan A — XIAO ESP32S3 Sense + LiPo + solar ⭐ recomendado

| Ítem | ARS |
|---|---|
| Seeed XIAO ESP32S3 Sense (OV3660, 8 MB PSRAM, cargador LiPo a bordo) | 55.000 |
| LiPo 1S 2000 mAh + conector JST | 12.000 |
| Panel solar 5–6 V 2 W | 15.000 |
| Módulo de carga solar (CN3065) o TP4056 ✔2.648 + diodo | 5.000 |
| microSD 32 GB (archivo de JPEGs) | 10.000 |
| LED blanco + resistencia (sincronizado al obturador) | 3.000 |
| Caja IP54 + brazo/soporte | 15.000 |
| **Total** | **~115.000 (≈ USD 74)** |

**Por qué gana:** es literalmente la placa para esto. **14 µA en deep sleep**, cargador LiPo
integrado (no necesitás TP4056 aparte), USB-C, 8 MB de PSRAM para el frame y la LUT, slot de
microSD, y la OV3660 es mejor que la OV2640 del ESP32-CAM. Con el panel es un nodo de *poner y
olvidarse*: consume ~0.7 Wh/día y un panel de 2 W en Buenos Aires cosecha unos 4 Wh/día útiles.

**En contra:** hay que escribir el firmware. Es corto —captura, CV, envío, sleep— pero es tuyo.
Placa importada; el precio local conviene confirmarlo en
[Starware](https://tienda.starware.com.ar/producto/modulo-camara-mic-programable-seeed-xiao-esp32-s3-sense-cam-mic-wifi-ble-sku-3139/).

---

### Plan B — ESP32-CAM + 18650 (económico)

| Ítem | ARS |
|---|---|
| ESP32-CAM AI-Thinker (OV2640) ✔ | 20.000 |
| Antena WiFi externa + pigtail | 3.000 |
| Celda 18650 2500 mAh + portapilas | 12.000 |
| Módulo TP4056 USB-C ✔ | 3.000 |
| Regulador + capacitores (fix de brownout) | 5.000 |
| microSD 16 GB | 8.000 |
| LED blanco + difusor | 3.000 |
| Caja IP54 + brazo | 15.000 |
| **Total** | **~69.000 (≈ USD 45)** |

**A favor:** el más barato, y el mismo firmware corre igual. Tiene 4 MB de PSRAM, alcanza.
**En contra:** el regulador AMS1117 consume 3–8 mA aun dormido — 200–500× peor que el XIAO.
Sumale antena floja, brownouts y EMI. Y la OV2640 con foco fijo es la peor cámara del lote
justo en el escenario óptico más difícil.
**Ojo:** el firmware AI-on-the-edge-device procesa a bordo, sí, pero **no duerme** (levanta un
servidor web permanente a ~180 mA) y necesita un broker MQTT o Home Assistant del otro lado.
No sirve ni para batería ni para "sin servidor".

---

### Plan C — Sin visión: termocupla tipo K + ESP32-C3

| Ítem | ARS |
|---|---|
| ESP32-C3 SuperMini | 12.000 |
| Módulo MAX31855 + termocupla tipo K | 18.000 |
| 18650 + TP4056 + portapilas | 15.000 |
| Caja | 8.000 |
| **Total** | **~53.000 (≈ USD 34)** |

Con el requisito de simplicidad, este plan se vuelve **imbatible**: sin cámara, sin CV, sin
calibración, sin reflejos. Son ~50 líneas de firmware. Consumo medio ~1.5 mA: **dos meses con
una 18650** midiendo cada diez segundos.

Y hay que decirlo con todas las letras: **si el objetivo es saber la temperatura, esto le gana
a la visión en todo.** ±1.5 °C reales, mide a nivel de parrilla —que es lo que importa para
cocinar—, no le afecta la noche ni el humo ni el reflejo, y es más barato. La sonda entra por
el damper superior o por la junta, sin perforar nada.

La visión gana si el objetivo es otro: no intervenir el kamado, leer el instrumento que ya
está, o el desafío técnico en sí.

**Recomendación concreta:** hacelo igual, aunque hagas visión. Por ARS 53.000 te da el
*ground truth* para validar el pipeline y armar la curva domo→parrilla de una vez.

---

### Descartados por la restricción de simplicidad

| Opción | Por qué queda afuera |
|---|---|
| **Pi Zero 2 W + Camera Module 3** | La mejor óptica del lote, pero no puede dormir (20–40 h por carga) y necesita backend propio. Sólo si A y B fallan por brillo. |
| **Cámara IP a batería (Tapo C400/C420/C425)** | **No funciona.** TP-Link confirma que los modelos a batería no exponen RTSP ni ONVIF, y sólo graban por detección de movimiento: una aguja que se mueve un grado por minuto no dispara el PIR nunca. |
| **Home Assistant / MQTT propio** | Es exactamente el servidor que no querés mantener. |

---

## 8. Comparación

| | A: XIAO S3 | B: ESP32-CAM | C: Termocupla |
|---|---|---|---|
| Costo ARS | 115.000 | 69.000 | 53.000 |
| Costo USD | 74 | 45 | 34 |
| **Autonomía** | **10 días · ∞ con solar** | ~6 días | **2 meses** |
| Servidores a mantener | **0** | **0** | **0** |
| Deep sleep | 14 µA | 3–8 mA | 14 µA |
| Firmware a escribir | ~200 líneas | ~200 líneas | **~50 líneas** |
| Calidad de imagen | Media-alta | Baja | — |
| Precisión final | ±10–15 °C* | ±10–15 °C* | **±1.5 °C** |
| Riesgo | Bajo | Medio | Muy bajo |

\* Dominado por el error del propio termómetro de domo, no por la visión.

---

## 9. Plan de trabajo por fases

**Fase 0 — Validar sin comprar nada (un fin de semana, ARS 0).**
Prendé el kamado, poné el celular en un trípode improvisado y sacá 40–60 fotos del dial
durante un cocido completo, incluyendo de noche y con humo. Escribí el unwrap polar en Python
sobre ese dataset en la notebook, medí el error, y **anotá las constantes**: centro, radios,
ángulos y temperaturas de dos marcas. Esas constantes van hardcodeadas al firmware.
**Elimina el ~80 % del riesgo antes del primer peso gastado.**

**Fase 1 — Portar el algoritmo (2 días).**
Traducir el pipeline de Python a C. Es mecánico: el unwrap con LUT son 20 líneas. Probalo
alimentándolo con las mismas imágenes de la Fase 0 cargadas desde la microSD, y verificá que
da los mismos números que Python.

**Fase 2 — Nodo completo (1 semana).**
Deep sleep con timer, wake → flash → captura → CV → envío a Blynk → JPEG a la SD → sleep.
Medí el consumo real con un USB tester o un shunt y compará con la tabla de §3. Sumá el bot
de Telegram.

**Fase 3 — Montaje definitivo (1 fin de semana).**
Brazo rígido, LED off-axis, visera, batería a la sombra abajo, panel solar orientado al norte.
Recalibrar con el montaje final. **Acá se gana o se pierde el proyecto.**

**Fase 4 — Validación (2–3 cocciones).**
Termocupla del Plan C en paralelo. Curva domo→parrilla, tabla de corrección y error real.

---

## 10. Riesgos y mitigaciones

| Riesgo | Prob. | Mitigación |
|---|---|---|
| **Reflejo especular en el vidrio del dial** | Alta | LED off-axis a 30–45°, film polarizador, visera, cámara ~20° fuera del eje. Se detecta en Fase 0. |
| **Depurar a ciegas** | Alta | JPEG a la microSD en cada lectura + `/foto` por Telegram. Sin eso, un número raro no tiene explicación. |
| **El powerbank corta por bajo consumo** | Alta *(si usás uno)* | No usar powerbank con un nodo que duerme. LiPo o 18650 directo. |
| **Batería degradada por calor** | Media | Batería abajo en el carro, a la sombra, lejos de la tapa. Caja ventilada, no negra sellada. |
| La cámara se mueve y rompe la calibración | Media | Auto-centrado por centroide + chequeo de cordura que reporta `descalibrado` en vez de mentir. |
| Condensación o grasa en la lente | Media | Visera, lente levemente hacia arriba, limpieza entre cocciones. |
| Reflashear para ajustar la calibración | Media | Hardcodear tras la Fase 0. Si molesta, modo AP con página de config que guarde en NVS. |
| Pasarse de la cuota de Blynk | Baja | Cadencia adaptativa: 15 min fuera de asado, 60 s cocinando. ~8.600 de 30.000 mensajes. |
| Humo denso tapa el dial | Baja | Filtro temporal (mediana + rechazo de saltos > X °C/min). |
| Precisión final decepcionante | Alta | Es limitación del termómetro, no del sistema. Se gestiona con expectativas + Plan C. |

---

## 11. Recomendación

1. **Hacé la Fase 0 este fin de semana.** Costo cero, y decide todo lo demás — incluyendo las
   constantes de calibración que van al firmware.
2. **Plan A** — XIAO ESP32S3 Sense + LiPo + panel solar, ≈ ARS 115.000. Todo a bordo, autonomía
   indefinida, cero infraestructura.
3. **Blynk para la app + bot de Telegram para alertas y fotos.** Ambos gratis, ambos sin
   servidor propio.
4. **microSD desde el día 1.** Es lo que hace que "sin servidor" no signifique "a ciegas".
5. **Sumá el Plan C en paralelo**, ≈ ARS 53.000, como ground truth.
6. **Si el precio local del XIAO no cierra**, el Plan B (ESP32-CAM, ARS 69.000) corre el mismo
   firmware con menos autonomía y peor cámara.

**Presupuesto recomendado: ~ARS 168.000 (≈ USD 109)**, con ARS 0 de costo mensual y cero
servidores. Versión económica con Plan B: **~ARS 122.000 (≈ USD 79)**.

Comparado con la versión con servidor: **sale más barato, hay menos hardware, y no queda nada
prendido en casa.**

---

## Fuentes

- XIAO ESP32S3 Sense: [Starware (AR)](https://tienda.starware.com.ar/producto/modulo-camara-mic-programable-seeed-xiao-esp32-s3-sense-cam-mic-wifi-ble-sku-3139/) · [Seeed Studio](https://www.seeedstudio.com/XIAO-ESP32S3-Sense-p-5639.html) · [wiki — 14 µA deep sleep](https://wiki.seeedstudio.com/es/xiao_esp32s3_getting_started/) · [atomic14: specs y gotchas](https://www.atomic14.com/esp32/boards/xiao-esp32-s3/)
- Free tiers: [Blynk — límites](https://docs.blynk.io/en/blynk.console/limits) · [Blynk — pricing](https://www.blynk.io/pricing) · [Adafruit IO — API y límites](https://io.adafruit.com/api/docs/) · [Adafruit IO — overview](https://learn.adafruit.com/welcome-to-adafruit-io?view=all) · [ThingSpeak — licencia gratuita](https://thingspeak.mathworks.com/pages/license_faq)
- Telegram desde ESP32: [UniversalTelegramBot](https://github.com/witnessmenow/Universal-Arduino-Telegram-Bot) · [Random Nerd Tutorials — ESP32-CAM manda fotos](https://randomnerdtutorials.com/telegram-esp32-cam-photo-arduino/) · [jameszah/ESP32-CAM-Telegram](https://github.com/jameszah/ESP32-Cam-Telegram)
- Consumo y deep sleep ESP32: [DeepBlue Embedded](https://deepbluembedded.com/esp32-sleep-modes-power-consumption/) · [DroneBot Workshop](https://dronebotworkshop.com/esp32-low-power/) · [SolderHub](https://solderhub.com/articles/esp32-deep-sleep-battery-life-guide)
- Powerbanks que cortan por bajo consumo: [Hackaday](https://hackaday.com/2024/03/05/a-simple-hack-for-running-low-power-gear-from-a-usb-battery-pack/) · [Raspberry Pi Forums](https://forums.raspberrypi.com/viewtopic.php?t=345977)
- Cámaras a batería sin RTSP: [Tapo FAQ — RTSP/ONVIF](https://www.tapo.com/us/faq/724/) · [Tapo FAQ — cámaras a batería](https://www.tapo.com/us/faq/370/)
- ESP32-CAM: [Starware](https://tienda.starware.com.ar/producto/modulo-wifi-bt-camara-arduino-esp32-cam-ov2640-2mp/) · [EB Electrónica](https://www.ebelectronica.com.ar/arduino/esp32-cam) · [fallas conocidas](https://randomnerdtutorials.com/esp32-cam-troubleshooting-guide/)
- Carga y baterías: [Nakama TP4056 USB-C ✔](https://www.nakamaelectronica.com.ar/productos/modulo-cargador-18650-tp4056-con-usb-c/) · [Starware TP4056](https://tienda.starware.com.ar/producto/modulo-carga-generico-tp4056-micro-usb-37v-1a-bateria-18650-3/) · [paneles solares mini](https://listado.mercadolibre.com.ar/panel-solar-6v-5w)
- Termocupla K: [Cyberofice (CABA)](https://www.cyberofice.com.ar/producto/modulo-max6675-max-6675-max-6675-termocupla-k-arduino/) · [TodoMicro](https://www.todomicro.com.ar/sensores/615-modulo-termometro-max6675-termocupla-tipo-k.html)
- Cotización: [Página 12, 12/08/2026](https://www.pagina12.com.ar/2026/08/12/dolar-blue-dolar-hoy-a-cuanto-cotizan-el-miercoles-12-de-agosto-de-2026/) · [La Nación](https://www.lanacion.com.ar/dolar-hoy/)
- Lectura de diales: [Intel analog-gauge-reader](https://www.intel.com/content/www/us/en/developer/articles/technical/analog-gauge-reader-using-opencv.html) · [Ghostbird](https://github.com/Ghostbird/Analogue-Gauge-Reader) · [Under pressure (arXiv 2404.08785)](https://arxiv.org/html/2404.08785v1) · [jomjol/AI-on-the-edge-device](https://github.com/jomjol/AI-on-the-edge-device)
- Domo vs parrilla: [Big Green Egg Forum](https://eggheadforum.com/discussion/1201373/dome-temp-vs-grate-temp) · [Pitmaster Club](https://pitmaster.amazingribs.com/forum/charcoal-kamados/big-green-egg/941963-dome-temp-vs-grid-temp) · [BBQ Brethren](https://www.bbq-brethren.com/threads/temp-difference-between-grate-and-dome-for-kamado-bbq.308997/)
