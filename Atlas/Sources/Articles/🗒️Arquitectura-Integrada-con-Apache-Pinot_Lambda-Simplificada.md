# 🗺 Arquitectura Integrada con Apache Pinot (Lambda Simplificada)

> **Apache Pinot** es la mejor opción si la prioridad del equipo es el **análisis de baja latencia** y los *dashboards* operativos, que es una evolución directa de la necesidad OLAP tradicional, pero en tiempo real.

Esta es una versión ***probable*** de la **Arquitectura de Referencia Completa** (o Arquitectura Lambda simplificada) que integra todos los componentes de un Lakehouse (S3, Iceberg, Spark, Presto/Trino, etc) con Apache Pinot, soportando tanto la ingesta **Batch (Offline)** desde Iceberg como **Streaming (Realtime)** desde Kafka, todo con un catálogo unificado.

## 🗺️ Arquitectura Integrada con Apache Pinot (Lambda Simplificada)

Esta arquitectura utiliza la fortaleza de cada herramienta: 

- Iceberg para la persistencia transaccional, 
- Kafka para la ingesta en tiempo real 
- y Pinot para la capa de servicio de análisis de baja latencia.

### 1. 💾 Capa de Almacenamiento y Catálogo (La Fuente de la Verdad)

| **Componente**           | **Rol en la Arquitectura**                                   | **Flujos Soportados**                      |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------ |
| **S3**                   | **Store (Almacenamiento):** Almacena todos los datos (archivos Parquet/ORC) y los segmentos de Pinot. | Batch, Streaming (a través de Iceberg)     |
| **Apache Iceberg**       | **Formato de Tabla Transaccional:** Proporciona consistencia (ACID) y gestión de esquemas sobre S3. La capa de datos limpia. | Batch, Streaming (a través de Flink/Spark) |
| **Hive Metastore (HMS)** | **Catálogo Unificado:** Es la fuente de metadatos central. Permite que **Spark**, **Presto/Trino**, y **Pinot** descubran la ubicación y el esquema de las tablas. | Metadata                                   |

### 2. 🏗️ Capa de Ingesta y Procesamiento (Pipeline)

Esta capa define las dos rutas de datos que alimentan a Pinot.

| **Ruta de Ingesta**      | **Herramientas Utilizadas**                                  | **Proceso Clave**                                            |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Streaming (Realtime)** | **Apache Kafka** --> **Pinot Realtime Server**               | Los datos frescos fluyen directamente de **Kafka** a los servidores en tiempo real de Pinot. Pinot consume y sirve las consultas con latencia de milisegundos. *Esta es la ruta más rápida.* |
| **Batch (Offline)**      | **Iceberg (S3)** --> **Apache Spark** --> **Pinot Offline Segment** | **Spark** lee la tabla **Iceberg** (datos históricos), genera segmentos optimizados de Pinot, y notifica al Pinot Controller. Los segmentos se almacenan en S3 y son servidos por los Offline Servers. |

> #### Función de Spark/Flink:
>
> Es fundamental utilizar Spark (o Flink) para un pipeline de streaming robusto:
>
> 1. **Transformación:** Lee de Kafka/fuente y realiza transformaciones.
> 2. **Escritura a Iceberg:** Escribe la salida limpia a una tabla **Iceberg en S3** (persistencia histórica).
> 3. **Carga a Pinot (Batch):** Utiliza un *job* de Spark para leer periódicamente la tabla Iceberg y crear los segmentos **Offline** de Pinot (Ruta Batch).

### 3. ⚡ Capa de Servicio y Consulta (El Motor OLAP)

Pinot actúa como el único punto de entrada para el análisis de baja latencia.

| **Componente**            | **Rol en la Arquitectura**                                   | **Experiencia de Usuario**                                   |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Pinot Realtime Server** | Sirve los datos ingeridos directamente desde Kafka (minutos/segundos recientes). | Latencia de **milisegundos**.                                |
| **Pinot Offline Server**  | Sirve los segmentos históricos generados a partir de la ingesta Batch de Iceberg. | Latencia baja (decenas de ms).                               |
| **Pinot Broker**          | **Unificación de Consultas:** Recibe la consulta SQL y la divide entre los servidores **Realtime** y **Offline**, consolidando los resultados. | **Vista Lógica Única** de datos históricos y frescos. (El reemplazo funcional de MS OLAP). |
| **Presto/Trino**          | **Análisis Ad-Hoc/Federación:** Se conecta directamente al **HMS/Iceberg** en S3. | Usado para análisis exploratorio profundo o *joins* federados que no requieren la latencia de Pinot. |

> ### Flujo de Consulta Unificado
>
> El usuario final o la aplicación de BI (por ejemplo, Apache Superset) solo se conecta al **Pinot Broker**.
>
> | **Tipo de Consulta**         | **Punto de Conexión** | **Rutas Internas de Datos**             |
> | ---------------------------- | --------------------- | --------------------------------------- |
> | **Dashboards Operacionales** | **Pinot Broker**      | Realtime Server $\oplus$ Offline Server |
> | **Exploración Histórica**    | **Presto/Trino**      | Directamente Iceberg/S3 (vía HMS)       |
> | **Segmentación de Usuarios** | **Pinot Broker**      | Realtime Server (principalmente)        |

Esta arquitectura te permite tener lo mejor de ambos mundos: la **fiabilidad histórica** del Data Lakehouse (Iceberg/S3) y la **velocidad operativa** del Real-Time OLAP (Pinot/Kafka).

---

## Notas:

> **Apache Pinot Realtime Server**  es una base de datos OLAP (procesamiento analítico en línea) en tiempo  real y de código abierto que permite la ingesta y el análisis de datos  con latencia ultra baja.   Está diseñado para manejar flujos de  datos en tiempo real, como de fuentes como Apache Kafka, y proporciona  resultados instantáneos para análisis y paneles interactivos, haciéndolo ideal para casos de uso que requieren una rápida toma de decisiones,  como análisis de clics, monitoreo de rendimiento de aplicaciones y  personalización.   Tal vez puedas leer referencias en https://www-uber-com.translate.goog/en-GB/blog/pinot-for-low-latency/?_x_tr_sl=en&_x_tr_tl=es&_x_tr_hl=es&_x_tr_pto=sge y en https://www.youtube.com/watch?v=HycNRCzkrjg&t=38s

> **Apache Pinot offline** es un componente de la arquitectura de [Apache Pinot](https://docs.pinot.apache.org/basics/concepts/components/cluster/server) que se encarga de almacenar y servir consultas de análisis sobre datos  que se han cargado previamente en "segmentos" offline. Su función  principal es manejar la porción de los datos que no son ingeridos en  tiempo real, recibiendo estos datos desde un repositorio de segmentos y  respondiendo a las consultas de análisis que le llegan a través de los  brokers.     Tal vez puedas leer referencias en https://www-uber-com.translate.goog/en-GB/blog/pinot-for-low-latency/?_x_tr_sl=en&_x_tr_tl=es&_x_tr_hl=es&_x_tr_pto=sge y en https://www.youtube.com/watch?v=HycNRCzkrjg&t=38s

>  **Apache Pinot Broker** es un componente de la arquitectura de [Apache Pinot](https://www.uber.com/blog/serving-millions-of-apache-pinot-queries-with-neutrino/), la base de datos OLAP de código abierto. Su función principal es **recibir consultas de los clientes a través de un puerto HTTP**, **analizarlas** y luego **enviar esas consultas a los servidores de datos** que contienen los segmentos de información necesarios para responderla. Finalmente, **recopila los resultados** de los servidores y los **consolida en una única respuesta** para el cliente que hizo la solicitud.  Tal ves puedas leer referencias eb https://www-uber-com.translate.goog/blog/serving-millions-of-apache-pinot-queries-with-neutrino/?_x_tr_sl=en&_x_tr_tl=es&_x_tr_hl=es&_x_tr_pto=sge y en https://www-uber-com.translate.goog/en-GB/blog/pinot-for-low-latency/?_x_tr_sl=en&_x_tr_tl=es&_x_tr_hl=es&_x_tr_pto=sge

