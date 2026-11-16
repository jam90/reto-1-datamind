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

## Qué construís vosotros

La planta ya funciona, pero **no recuerda nada**. Lo que se ve en su web es el instante presente; en
cuanto pasa, se pierde. Hoy nadie puede contestar a preguntas como estas:

- ¿Cuántas piezas KO hubo ayer, y de qué impresora salieron?
- ¿Cuánto tardó de media un pedido esta semana?
- ¿Qué piezas se imprimieron con el lote de bobina `PLA-AZU-001-01`?
- ¿Cuántas veces se ha atascado `bambu-03` este mes, y qué hizo el operario cada vez?

Los datos para responderlas pasan por MQTT, pero nadie los guarda: MQTT entrega cada mensaje a quien
esté escuchando en ese momento y lo olvida. Vuestro asistente tiene que responder justo a preguntas
así, de modo que antes necesita esa memoria. **No tocáis la planta: construís al lado lo que escucha,
guarda y responde.**

```
Planta ──MQTT──▶ Productor ──▶ Kafka ──▶ Consumidor ──▶ PostgreSQL
(ya existe)      (vosotros)    (vosotros) (vosotros)     (vosotros)
```

- **Productor:** un programa vuestro que se suscribe al MQTT de la planta y publica cada mensaje en
  Kafka. Es el puente entre la planta y la empresa.
- **Kafka:** guarda los mensajes en orden y durante días. Si vuestro consumidor se cae una hora, al
  volver sigue por donde iba y no pierde nada; con MQTT a secas, esa hora se habría perdido. Además,
  varios consumidores pueden leer lo mismo sin estorbarse: uno hacia PostgreSQL, otro hacia Elasticsearch.
- **Consumidor:** lee de Kafka, limpia y da forma a los datos y los escribe en PostgreSQL.
- **PostgreSQL:** lo que tiene estructura y relaciones, para preguntar con SQL. Una pieza pertenece a
  un pedido, se imprimió en una impresora con un lote, pasó o no la cámara y tuvo o no una alarma:
  eso son tablas con claves.
- **Elasticsearch:** el texto para buscar: el manual, los partes diarios, los partes del operario y
  los textos de las alarmas.

Una decisión que os toca: la telemetría de cada segundo son más de dos millones de mensajes al día.
Guardarla entera, resumida, solo una parte o no guardarla es diseño vuestro, y hay que justificarlo.

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

## Pistas: de dónde puede salir cada parte del reto

Son ideas, no obligaciones. El diseño es vuestro y hay que justificarlo.

| El reto pide | En la planta tenéis |
|---|---|
| Scraping de fuentes sin API, que se actualizan | Los partes diarios (cambian durante el día) y el manual de la planta. Y documentación pública de los fabricantes de las máquinas, por ejemplo los códigos de alarma de las impresoras: <https://wiki.bambulab.com/en/hms/home> |
| Datos en tiempo real (Kafka → PostgreSQL) | Los mensajes MQTT: pedidos, piezas, avisos, alarmas, órdenes de mantenimiento |
| Texto para indexar y buscar (Elasticsearch) | Los capítulos del manual, los textos de las alarmas y los **partes que escribe el operario** al acabar cada tarea (llegan por MQTT y salen en el parte diario) |
| Preguntas que comparan o analizan datos en vivo | «¿Qué impresoras están paradas y por qué?», «¿cuántas piezas KO ha habido hoy y de qué tipo?» |
| Visualizaciones en Kibana | Incidencias por código y por máquina, piezas KO, tiempos del operario, consumo eléctrico |
| Evaluar el asistente | Preguntas cuya respuesta se puede comprobar en el manual o en la web de la planta |

Antes de hacer scraping de una web que no sea la de la planta, mirad sus condiciones de uso y su
`robots.txt`: forma parte del apartado legal y ético del reto.
