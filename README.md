# HLD – Arquitectura de Datos para Yachay-Shop

**Universidad Yachay Tech · Maestría en Ciencia de Datos**
Taller 04 – Diseño de Arquitectura de Datos para un E-Commerce

Documento de diseño de alto nivel (*High Level Design*, HLD) que propone una arquitectura de datos para **Yachay-Shop**, un e-commerce de tamaño medio que vende por sitio web, aplicación móvil y tiendas físicas.

---

## Objetivos del taller

1. Entender y aplicar teoría de arquitectura de datos.
2. Identificar los elementos importantes de la arquitectura del sistema.
3. Visualizar la estructura en un documento HLD.

## Problema

Yachay-Shop tiene sus datos repartidos en cinco sistemas aislados:

| Fuente | Datos |
| --- | --- |
| Sitio web y app móvil | Clickstream, pedidos online, perfiles de usuario |
| Tiendas físicas (POS) | Transacciones, inventario, tarjetas de fidelización |
| CRM | Perfiles de cliente, historial de pedidos, campañas |
| Gestión de inventarios | Existencias por almacén y tienda |
| Marketing externo | Google Analytics, Facebook Ads |

**Desafíos:** silos de datos, gran volumen y variedad de formatos, y necesidad de procesamiento en tiempo real.

**Necesidades del negocio:** Vista 360 del cliente, inventario en tiempo real en todos los canales, medición de la eficacia del marketing, y reportes y pronósticos de ventas.

## Solución propuesta

Una arquitectura **Lakehouse con modelo medallion (Bronze → Silver → Gold)**, con dos caminos de ingestión.

```
 FUENTES            INGESTIÓN              LAKEHOUSE (Delta Lake)          CONSUMO
┌──────────┐      ┌─────────────┐  ─── camino en tiempo real (< 5 s) ───►  API de inventario
│ Web/App  │─────►│             │                                         Dashboard operativo
│ POS      │─────►│  Streaming  │──►  ┌────────┐   ┌────────┐   ┌──────┐
│ Inventar.│─────►│ Kafka + CDC │     │ Bronze │──►│ Silver │──►│ Gold │──► Power BI
└──────────┘      └─────────────┘     │ crudo  │   │ limpio │   │negocio│──► Ciencia de datos
┌──────────┐      ┌─────────────┐  ┌─►│        │   │  MDM   │   │      │──► Reverse ETL (CRM)
│ CRM      │─────►│   Batch     │──┘  └────────┘   └────────┘   └──────┘
│ Marketing│─────►│ Airbyte +   │
└──────────┘      │ Airflow     │
                  └─────────────┘
 Transversal: catálogo y linaje · RBAC y PII · calidad de datos · orquestación · monitoreo
```

### Decisiones clave

| Decisión | Elección | Motivo |
| --- | --- | --- |
| Almacenamiento | **Lakehouse** (combinación de Data Lake + Data Warehouse) | Acepta datos estructurados, semiestructurados y no estructurados a bajo costo, con transacciones ACID y rendimiento para BI |
| Ingestión | **Streaming** (Kafka, CDC con Debezium) + **Batch** (Airbyte, Airflow) | Cada fuente entra según la latencia que necesita |
| Organización | **Medallion**: Bronze (crudo), Silver (limpio y unificado), Gold (modelos de negocio) | Refinar por etapas sin perder nunca el dato original |
| Procesamiento | **Híbrido tipo Kappa** | Streaming y batch escriben en las mismas tablas; no se duplica la lógica |
| Vista 360 | **MDM** con `golden_customer_id` | Une al mismo cliente de web, app, POS y CRM |
| Modelo analítico | **Esquema estrella** (Kimball) en Gold | Modelo natural para Power BI |
| Escalabilidad | Separación almacenamiento/cómputo, particiones en Kafka, autoescalado | Soporta picos de ventas flash sin rediseño |

### Stack de referencia

El diseño es agnóstico de proveedor. Como implementación de referencia usa:

- **Ingestión:** Apache Kafka, Schema Registry, Debezium, Airbyte
- **Procesamiento:** Apache Spark, Apache Flink, dbt
- **Almacenamiento:** Object storage + Delta Lake
- **Orquestación:** Apache Airflow
- **Gobierno y calidad:** Unity Catalog, Great Expectations
- **Consumo:** Power BI, notebooks, MLflow
- **Caché de inventario:** Redis / DynamoDB / Cosmos DB

Cada componente tiene un equivalente gestionado en Azure, AWS o GCP.

## Contenido del documento

1. Resumen ejecutivo
2. Contexto y requerimientos
3. Decisión: Data Warehouse, Data Lake o ambos
4. Arquitectura de alto nivel (diagrama)
5. Métodos de ingestión por fuente
6. Transformación y unificación de datos
7. Almacenamiento y modelo de datos
8. Procesamiento batch y en tiempo real
9. Consumo: BI, reportes y analítica predictiva
10. Gobierno, seguridad, calidad y observabilidad
11. Escalabilidad
12. Trazabilidad, implementación y conclusiones

## Estructura del repositorio

```
.
├── main.tex      # Documento HLD completo en LaTeX (incluye el diagrama en TikZ)
└── README.md     # Este archivo
```

## Cómo compilar

### Opción 1: Overleaf (recomendado)

1. Ir a [Overleaf](https://www.overleaf.com) → **New Project** → **Upload Project**.
2. Subir `main.tex` (o un `.zip` con el repositorio).
3. Compilar con **pdfLaTeX** (compilador por defecto).

### Opción 2: Local en Windows (PowerShell)

Requiere [MiKTeX](https://miktex.org/download) o TeX Live instalado.

```powershell
pdflatex main.tex
pdflatex main.tex   # segunda pasada para el índice y las referencias
```

> Se compila dos veces para que se generen correctamente el índice y las referencias cruzadas.

### Paquetes LaTeX utilizados

`babel` (español), `tikz` (diagrama), `tabularx`, `booktabs`, `longtable`, `float`, `fancyhdr`, `titlesec`, `hyperref`, entre otros. Overleaf los incluye todos.

## Glosario rápido

| Sigla | Significado |
| --- | --- |
| HLD | High Level Design – diseño de alto nivel |
| CDC | Change Data Capture – captura de cambios en una base de datos |
| MDM | Master Data Management – gestión de datos maestros (cliente único) |
| ACID | Atomicidad, Consistencia, Aislamiento, Durabilidad |
| ETL | Extract, Transform, Load |
| SCD | Slowly Changing Dimension – historial de cambios en dimensiones |
| POS | Point of Sale – punto de venta |
| CRM | Customer Relationship Management |
| KPI | Key Performance Indicator |
| ROAS | Return on Ad Spend – retorno por dólar invertido en publicidad |
| RFM | Recency, Frequency, Monetary – segmentación de clientes |
| PII | Personally Identifiable Information – datos personales |
| RBAC | Role-Based Access Control – control de acceso por roles |
| LOPDP | Ley Orgánica de Protección de Datos Personales (Ecuador) |
| SLA | Service Level Agreement – acuerdo de nivel de servicio |

## Autor

**Joel** – Maestría en Ciencia de Datos, Universidad Yachay Tech
Octubre 2026
