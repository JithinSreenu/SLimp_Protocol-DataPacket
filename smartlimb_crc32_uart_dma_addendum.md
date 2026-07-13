# SmartLimb CRC32 UART DMA Addendum for STM32U585

This addendum gives a code-oriented implementation for the SmartLimb telemetry packet on the B-U585I-IOT02A using USART1 Virtual COM, CRC32 hardware, UART DMA transmit, and decimal input support through the serial terminal.[cite:1][cite:2][cite:168][cite:270]

The board’s ST-LINK Virtual COM port is connected to USART1, and the default terminal settings are 115200 baud, 8 data bits, no parity, one stop bit, and no flow control.[cite:1][cite:264]

## Packet definition

| Field | Size | Value / Notes |
|---|---:|---|
| Start Byte | 1 | `0xAA` |
| Packet Type | 1 | `0x01` |
| Version | 1 | `0x01` |
| Sequence No. | 2 | Incrementing counter, little-endian |
| Payload Length | 1 | `16` |
| Timestamp | 4 | `HAL_GetTick()` |
| Force | 2 | `int16_t`, recommended x10 scaling for decimals |
| Moment | 2 | `int16_t`, recommended x10 scaling for decimals |
| Knee Angle | 2 | `int16_t`, recommended x100 scaling for decimals |
| Valve Position | 2 | `int16_t`, recommended x100 scaling for decimals |
| Battery mV | 2 | `uint16_t` |
| System State | 1 | enum |
| Fault Code | 1 | enum |
| CRC32 | 4 | Hardware CRC32 over bytes 1..21 |
| Stop Byte | 1 | `0x55` |

Total packet size is 27 bytes.[cite:169]

## Why decimals need scaling

Do not try to enter decimals directly into a packed `int16_t` field. Instead, accept decimal text in the terminal, store it as `float`, and convert it to scaled integers before building the packet.

Recommended scaling:

- Force: x10
- Moment: x10
- Knee angle: x100
- Valve position: x100
- Battery: millivolts

That keeps the packet small and preserves decimal precision.

## Data structures

```c
typedef struct
{
    uint32_t timestamp;
    float force_N;
    float moment_Nm;
    float kneeAngle_deg;
    float valvePosition_percent;
    float batteryVoltage_V;
    uint8_t systemState;
    uint8_t faultCode;
} TelemetryValues_t;
```

```c
typedef struct
{
    uint32_t timestamp;
    int16_t force_N;
    int16_t moment_Nm;
    int16_t kneeAngle_deg;
    int16_t valvePosition_percent;
    uint16_t batteryVoltage_mV;
    uint8_t systemState;
    uint8_t faultCode;
} TelemetryPacket_t;
```

## Conversion helpers

```c
static int16_t pack_x10(float v)
{
    if (v > 3276.7f) v = 3276.7f;
    if (v < -3276.8f) v = -3276.8f;
    return (int16_t)((v >= 0.0f) ? (v * 10.0f + 0.5f) : (v * 10.0f - 0.5f));
}

static int16_t pack_x100(float v)
{
    if (v > 327.67f) v = 327.67f;
    if (v < -327.68f) v = -327.68f;
    return (int16_t)((v >= 0.0f) ? (v * 100.0f + 0.5f) : (v * 100.0f - 0.5f));
}

static uint16_t volts_to_mV(float v)
{
    if (v < 0.0f) v = 0.0f;
    if (v > 65.535f) v = 65.535f;
    return (uint16_t)(v * 1000.0f + 0.5f);
}
```

## ASCII command parser

Use readable terminal commands such as `set force 185.7` instead of manually typing the binary frame.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

TelemetryValues_t g_values = {0};

static void ProcessAsciiCommand(const char *cmd)
{
    float f;
    int i;

    if (sscanf(cmd, "set force %f", &f) == 1)           g_values.force_N = f;
    else if (sscanf(cmd, "set moment %f", &f) == 1)     g_values.moment_Nm = f;
    else if (sscanf(cmd, "set knee %f", &f) == 1)       g_values.kneeAngle_deg = f;
    else if (sscanf(cmd, "set valve %f", &f) == 1)      g_values.valvePosition_percent = f;
    else if (sscanf(cmd, "set batt %f", &f) == 1)       g_values.batteryVoltage_V = f;
    else if (sscanf(cmd, "set state %d", &i) == 1)      g_values.systemState = (uint8_t)i;
    else if (sscanf(cmd, "set fault %d", &i) == 1)      g_values.faultCode = (uint8_t)i;
    else if (strcmp(cmd, "send") == 0)                  (void)Telemetry_SendDMA(&g_values);
}
```

## CRC32 and DMA packet sender

STM32 CRC HAL provides CRC calculation APIs, and UART DMA transmission requires USART interrupt support to complete properly.[cite:168][cite:270][cite:268]

```c
#include "main.h"
#include <string.h>

#define PKT_START_BYTE   0xAA
#define PKT_STOP_BYTE    0x55
#define PKT_TYPE_TELEM   0x01
#define PKT_VERSION      0x01
#define PKT_PAYLOAD_LEN  16
#define PKT_TOTAL_LEN    27

extern CRC_HandleTypeDef hcrc;
extern UART_HandleTypeDef huart1;

static uint16_t g_sequence = 0;
static uint8_t  g_txBusy = 0;
static uint8_t  g_txBuf[PKT_TOTAL_LEN];

static void put_u16_le(uint8_t *b, uint16_t v)
{
    b[0] = (uint8_t)(v & 0xFF);
    b[1] = (uint8_t)((v >> 8) & 0xFF);
}

static void put_i16_le(uint8_t *b, int16_t v)
{
    b[0] = (uint8_t)(v & 0xFF);
    b[1] = (uint8_t)((v >> 8) & 0xFF);
}

static void put_u32_le(uint8_t *b, uint32_t v)
{
    b[0] = (uint8_t)(v & 0xFF);
    b[1] = (uint8_t)((v >> 8) & 0xFF);
    b[2] = (uint8_t)((v >> 16) & 0xFF);
    b[3] = (uint8_t)((v >> 24) & 0xFF);
}

static uint32_t crc32_hw_bytes(const uint8_t *data, uint32_t len)
{
    uint32_t words[6] = {0};

    for (uint32_t i = 0; i < len; i++)
        words[i / 4] |= ((uint32_t)data[i]) << (8U * (i % 4U));

    return HAL_CRC_Calculate(&hcrc, words, (len + 3U) / 4U);
}

static void Telemetry_BuildPacket(const TelemetryValues_t *src, uint8_t *out)
{
    TelemetryPacket_t p;
    uint32_t crc;

    p.timestamp = src->timestamp;
    p.force_N = pack_x10(src->force_N);
    p.moment_Nm = pack_x10(src->moment_Nm);
    p.kneeAngle_deg = pack_x100(src->kneeAngle_deg);
    p.valvePosition_percent = pack_x100(src->valvePosition_percent);
    p.batteryVoltage_mV = volts_to_mV(src->batteryVoltage_V);
    p.systemState = src->systemState;
    p.faultCode = src->faultCode;

    out[0] = PKT_START_BYTE;
    out[1] = PKT_TYPE_TELEM;
    out[2] = PKT_VERSION;
    put_u16_le(&out[3], g_sequence);
    out[5] = PKT_PAYLOAD_LEN;

    put_u32_le(&out[6],  p.timestamp);
    put_i16_le(&out[10], p.force_N);
    put_i16_le(&out[12], p.moment_Nm);
    put_i16_le(&out[14], p.kneeAngle_deg);
    put_i16_le(&out[16], p.valvePosition_percent);
    put_u16_le(&out[18], p.batteryVoltage_mV);
    out[20] = p.systemState;
    out[21] = p.faultCode;

    crc = crc32_hw_bytes(&out[1], 21);
    put_u32_le(&out[22], crc);
    out[26] = PKT_STOP_BYTE;
}

HAL_StatusTypeDef Telemetry_SendDMA(const TelemetryValues_t *src)
{
    if (g_txBusy)
        return HAL_BUSY;

    Telemetry_BuildPacket(src, g_txBuf);
    g_txBusy = 1;

    if (HAL_UART_Transmit_DMA(&huart1, g_txBuf, PKT_TOTAL_LEN) != HAL_OK)
    {
        g_txBusy = 0;
        return HAL_ERROR;
    }

    g_sequence++;
    return HAL_OK;
}

void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
        g_txBusy = 0;
}
```

## CubeMX checklist

For this project, configure:

- CRC peripheral enabled.[cite:168][cite:169]
- USART1 asynchronous mode for ST-LINK VCP.[cite:1][cite:264]
- UART DMA transmit enabled.[cite:270]
- DMA interrupt enabled.[cite:270]
- USART global interrupt enabled.[cite:270][cite:268]
- ADC channels for the load cell, hall sensors, and battery input if they are analog sources.[cite:2]

## Best upgrade path

The cleanest upgrade is to split the firmware into three layers:

1. Acquisition layer: read ADC, sensors, state, and faults.
2. Packet layer: convert decimal engineering values to fixed-point, serialize, and compute CRC32.
3. Transport layer: send the frame using UART DMA and handle completion.

That will make your code easier to debug now and much easier to extend later to BLE, Wi‑Fi, logging, or inter-MCU communication.
