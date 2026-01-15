# Architecture Overview

## CESMII Work Order Publisher Demo

This document describes the architecture of the CESMII Work Order Publisher demo application, which demonstrates the use of CESMII Smart Manufacturing (SM) Profiles for publishing structured work order data to an MQTT broker (Unified Namespace - UNS).

## System Components

```
+-------------------+       +------------------+       +------------------+
|                   |       |                  |       |                  |
|  Work Order       | ----> |  MQTT Broker     | ----> |  Subscribers     |
|  Publisher        |       |  (UNS)           |       |  (Consumers)     |
|  (Python App)     |       |                  |       |                  |
+-------------------+       +------------------+       +------------------+
        |
        v
+-------------------+
|                   |
|  SM Profiles      |
|  (JSON-LD)        |
|                   |
+-------------------+
```

## Component Descriptions

### 1. Work Order Publisher (`workorder_publisher.py`)

The main Python application responsible for:
- Loading configuration from `config.json`
- Managing demo product definitions with feed ingredients
- Generating work orders every hour
- Publishing work orders to MQTT as retained messages with JSON-LD context

**Key Classes:**
- `DemoProduct`: Represents a product with its ingredients
- `FeedIngredient`: Represents an ingredient with mix proportion
- `WorkOrderGenerator`: Generates work orders conforming to WorkOrderV1 profile
- `MQTTWorkOrderPublisher`: Handles MQTT connection and publishing

### 2. SM Profiles (`smprofiles/`)

JSON-LD profile definitions that serve as "information contracts" between publishers and consumers:

#### FeedIngredientV1.jsonld
Defines the structure for feed ingredient data:
- ProductID (opc:String)
- ProductNumber (opc:Int64)
- ProductName (opc:String)
- LotNumber (opc:String)
- UnitOfMeasure (opc:String)
- Quantity (opc:Double)
- WeightUnitOfMeasure (opc:String)
- Weight (opc:Double)

#### WorkOrderV1.jsonld
Defines the structure for work order data:
- WorkOrderID (opc:String)
- WorkOrderNumber (opc:Int32)
- TimeZone (opc:TimeZoneDataType)
- StartTimeLocal/EndTimeLocal (opc:DateTime)
- StartTimeUTC/EndTimeUTC (opc:UtcTime)
- Product information fields
- FeedIngredients (array of FeedIngredientV1)

### 3. Configuration (`config.json`)

Externalized configuration for:
- MQTT broker connection (host, port, credentials)
- Publish topic for work orders

## Data Flow

1. **Initialization**
   - Application loads configuration from `config.json`
   - Demo products are created with predefined ingredients and mix proportions
   - MQTT connection is established

2. **Work Order Generation** (every hour)
   - Random product selected from demo products
   - Timestamps calculated in UTC and Central Time
   - Quantity generated (random, divisible by 6)
   - Weight calculated (Quantity x 2)
   - Ingredients created with proportional quantities/weights
   - Lot numbers generated (random alphanumeric)

3. **Publishing**
   - Work order serialized as JSON with JSON-LD @context
   - Published to MQTT topic as retained message (QoS 1)
   - Consumers receive data with profile reference for validation

## OPC UA Type System

The SM Profiles use OPC UA data types from the `http://opcfoundation.org/UA/` namespace:

| Type | NodeId | Description |
|------|--------|-------------|
| Boolean | i=1 | True/False |
| Int16 | i=4 | 16-bit signed integer |
| Int32 | i=6 | 32-bit signed integer |
| Int64 | i=8 | 64-bit signed integer |
| Double | i=11 | 64-bit floating point |
| String | i=12 | UTF-8 string |
| DateTime | i=13 | Date and time |
| UtcTime | i=294 | UTC timestamp |
| TimeZoneDataType | i=8912 | Time zone structure |

## JSON-LD Profile Reference

Each published work order includes a JSON-LD `@context` that:
- Maps field names to profile attributes
- References OPC UA and CESMII namespaces
- Provides `profileDefinition` link to the full profile schema

This allows consumers to:
- Validate incoming data against the profile
- Understand data semantics without prior knowledge
- Maintain interoperability across systems

## Directory Structure

```
cesmii/
├── config.json                 # MQTT and topic configuration
├── requirements.txt            # Python dependencies
├── workorder_publisher.py      # Main application
├── smprofiles/
│   ├── FeedIngredientV1.jsonld # Feed ingredient profile
│   └── WorkOrderV1.jsonld      # Work order profile
├── claude.md                   # This architecture document
└── README.md                   # Project documentation
```

## Dependencies

- **paho-mqtt**: MQTT client library for Python
- **pytz**: Timezone handling for Central Time calculations

## Security Considerations

- MQTT credentials stored in config.json (should be secured in production)
- No TLS/SSL configured by default (should be enabled in production)
- Retained messages persist on broker (consider data retention policies)
