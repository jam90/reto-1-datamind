# Reto 1 · DataMind sobre la planta

Material de partida del equipo para el reto **«DataMind: Scraping, Indexación y Respuesta»**.
El enunciado completo está en [`Reto1_MIA_PIA_BDA_SBD - Alumno.docx`](Reto1_MIA_PIA_BDA_SBD%20-%20Alumno.docx);
esto solo explica **quién es vuestro cliente y qué datos tenéis**.

## El cliente: una planta de impresión 3D

El cliente de DataBot Solutions es una fábrica de piezas impresas en 3D que trabaja sola, las 24 horas:
seis impresoras, cuatro robots móviles, brazos, cubas, una cinta con cámara de calidad, un almacén
automático y un operario que atiende las averías. Es una **planta simulada** (un gemelo digital), pero
sus datos salen como saldrían de una planta de verdad, y no para.

Su problema es el del enunciado: tiene muchos datos, dispersos y en constante actualización, y nadie
que no sea especialista puede preguntarles nada. Quiere un **asistente que responda preguntas sobre el
funcionamiento de la planta**: qué es cada máquina, qué significa una alarma, qué está pasando ahora,
qué se hizo la última vez que falló algo.

Ya conocéis las máquinas: son las del reto anterior (impresoras Bambu Lab, robots MiR, Dobot).
**InfluxDB y Grafana no entran en este reto.**

## Qué tenéis

Acceso a los datos de la planta, no a su código. Como en un cliente real.

| Qué | Dónde | Para qué |
|---|---|---|
| **Manual de la planta** | `http://SERVIDOR:18080/manual` | Explica las máquinas, el proceso, el catálogo, los códigos de alarma y todos los mensajes. Son páginas web, **sin API** |
| **Datos en vivo (MQTT)** | `SERVIDOR`, puerto `11898`, con usuario y contraseña | Telemetría de cada máquina cada segundo, avisos, pedidos, alarmas, partes de mantenimiento, energía. **Solo lectura** |
| **Web de la planta** | `http://SERVIDOR:18080` | Ver la planta en marcha, **mandar pedidos y provocar averías** para vuestras pruebas |
| **Partes diarios** | `http://SERVIDOR:18080/partes` | Una página por día con lo que ha pasado: pedidos, producción, piezas KO, alarmas, partes de mantenimiento y energía. La de hoy **cambia durante el día**. Tampoco tiene API |

La dirección del servidor, el usuario y la contraseña os los da el profesor. **No los subáis a ningún repositorio.**

Para empezar a mirar:

```bash
# todos los avisos de la planta, según ocurren
mosquitto_sub -h SERVIDOR -p 11898 -u USUARIO -P CONTRASEÑA -t 'factory/events/#' -v

# la portada del manual y el parte de hoy
curl http://SERVIDOR:18080/manual
curl http://SERVIDOR:18080/partes
```

## Qué le falta a la planta

La planta ya funciona, y funciona sola. Lo que no tiene es **memoria**, ni nadie a quien preguntar.

Lo que se ve en su web es el instante presente; en cuanto pasa, se pierde. Hoy nadie puede contestar a
preguntas como estas:

- ¿Cuántas piezas KO hubo ayer, y de qué impresora salieron?
- ¿Cuánto tardó de media un pedido esta semana?
- ¿Qué piezas se imprimieron con el lote de bobina `PLA-AZU-001-01`?
- ¿Cuántas veces se ha atascado `bambu-03` este mes, y qué hizo el operario cada vez?
- ¿Qué significa el código `0300-0900-0002-0002` y qué hay que hacer cuando salta?

Los datos para responderlas existen: pasan por MQTT o están escritos en el manual y en los partes. Pero
MQTT entrega cada mensaje a quien esté escuchando en ese momento y lo olvida, y el manual no contesta
preguntas. Eso es lo que el cliente os encarga: **darle a la planta memoria y voz.**

## Dónde entráis vosotros

**Al lado de la planta, no en medio.** La planta tiene su propio sistema nervioso (MQTT): por ahí entran
los pedidos, se mandan las órdenes a las máquinas y vuelven los avisos. Eso no lo tocáis, y tiene que
seguir funcionando igual aunque vuestro sistema esté apagado.

```
   LA PLANTA (ya existe, funciona sola)
   pedidos ─▶ órdenes ─▶ máquinas y robots ─▶ avisos
                      │
                      │  MQTT, solo lectura            manual y partes (páginas web)
                      ▼                                            │
              algo vuestro que escucha                 algo vuestro que va a buscarlos
                      │                                            │
                      ▼                                            ▼
                 ·········  lo que diseñéis: guardar, ordenar, relacionar, indexar  ·········
                                               │
                                               ▼
                              alguien a quien se le puede preguntar
```

La planta habla MQTT y no sabe nada de vuestras herramientas. **La primera pieza es, por tanto, un
puente:** un programa vuestro que escucha lo que dice la planta y lo pasa a vuestro lado. A partir de
ahí, el diseño es vuestro. Estas preguntas os pueden guiar:

**Sobre lo que llega mensaje a mensaje**

- Si vuestro programa se cae una hora, ¿qué pasa con lo que la planta dijo durante esa hora? ¿Qué
  herramienta de las que conocéis guarda los mensajes en orden hasta que alguien los recoge?
- Si dos programas distintos necesitan los mismos mensajes (uno para guardarlos en tablas, otro para
  indexar su texto), ¿se los tienen que pedir los dos a la planta?
- Llegan más de dos millones de mensajes de telemetría al día. ¿Hace falta guardarlos todos, tal cual?

**Sobre lo que tiene relaciones**

- Una pieza pertenece a un pedido, se imprimió en una impresora con un lote de bobina, pasó o no la
  cámara y pudo tener una alarma. Todo eso llega en mensajes sueltos, a destiempo. ¿Dónde se juntan?
  ¿Con qué claves?
- ¿Qué consulta contesta a «cuántas piezas KO hubo ayer por impresora»? ¿Tarda lo mismo con mil filas
  que con un millón?

**Sobre lo que es texto**

- Hay preguntas que no se responden con una tabla: qué significa un código, qué escribió el operario
  en su parte, cómo es el recorrido de una pieza. ¿Dónde está ese texto y cómo se busca en él?
- Alguien escribirá «inpresora atascada». ¿Lo encuentra vuestro buscador?

**Sobre lo que no llega solo**

- El manual y los partes diarios no avisan: hay que ir a buscarlos. ¿Cada cuánto? ¿Cómo sabéis si han
  cambiado desde la última vez?
- Lo que se extrae de una página viene sucio (comas decimales, horas locales, textos). ¿Dónde y con
  qué se limpia y se le da forma? ¿Quién se entera si un día ese proceso falla?

**Sobre el asistente**

- Para contestar «¿por qué está parada la impresora 3?» hacen falta dos cosas distintas: qué le está
  pasando ahora y qué significa. ¿De dónde saca cada una?
- ¿Cómo sabe quien pregunta de dónde ha salido la respuesta? ¿Y cómo sabéis vosotros si responde bien?

## Reglas del juego

- **Por MQTT solo se lee.** Lo que publiquéis por esa entrada no llega a nadie.
- **Desde la web podéis mandar pedidos y provocar averías.** Sois los únicos usando la planta, pero es
  una sola: lo que provoquéis lo veis todos.
- **La planta no guarda histórico de sus datos.** Lo que no escuchéis por MQTT, se pierde: guardarlo es parte de vuestro trabajo. Los partes diarios sí se conservan, pero son un resumen.
- Los robots móviles se ven en la web de la planta y por MQTT. La web del gestor de flota no está abierta.
- El manual puede cambiar cuando la planta cambie, y el parte del día cambia cada pocos minutos. Cada página lleva una **revisión** al pie (y en
  `<meta name="revision">`) que cambia con el contenido.
- Si la web avisa de **«modo rápido»**, la planta va acelerada para pruebas y esos datos no valen para análisis.
- El manual cuenta qué alarma salta y qué se hace, **no la causa de fondo** de cada fallo. Eso está en los datos.

## De dónde puede salir cada parte del reto

Lo que el enunciado pide y lo que la planta ofrece para ello. El diseño es vuestro y hay que justificarlo.

| El reto pide | En la planta tenéis |
|---|---|
| Scraping de fuentes sin API, que se actualizan | Los partes diarios (cambian durante el día) y el manual de la planta. Y documentación pública de los fabricantes de las máquinas, por ejemplo los códigos de alarma de las impresoras: <https://wiki.bambulab.com/en/hms/home> |
| Datos en tiempo real | Los mensajes MQTT: pedidos, piezas, avisos, alarmas, órdenes de mantenimiento, telemetría |
| Texto para indexar y buscar | Los capítulos del manual, los textos de las alarmas y los **partes que escribe el operario** al acabar cada tarea (llegan por MQTT y salen en el parte diario) |
| Preguntas que comparan o analizan datos en vivo | «¿Qué impresoras están paradas y por qué?», «¿cuántas piezas KO ha habido hoy y de qué tipo?» |
| Visualizaciones | Incidencias por código y por máquina, piezas KO, tiempos del operario, consumo eléctrico |
| Evaluar el asistente | Preguntas cuya respuesta se puede comprobar en el manual, en los partes o en la web de la planta |

Antes de hacer scraping de una web que no sea la de la planta, mirad sus condiciones de uso y su
`robots.txt`: forma parte del apartado legal y ético del reto.
