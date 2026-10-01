# Log Praktikum Protokol Komunikasi IoT - SPI & nRF24L01

Dokumentasi eksperimen dan implementasi komunikasi serial menggunakan protokol **SPI (Serial Peripheral Interface)** antar board ESP32 serta transmisi nirkabel menggunakan modul **nRF24L01+**.

---

## Daftar Isi
- [1. Komunikasi SPI Antar Board (Board-to-Board)](#1-komunikasi-spi-antar-board-board-to-board)
  - [Konfigurasi Hardware](#1-komunikasi-spi-antar-board-board-to-board)
  - [Tabel Wiring / Pinout](#tabel-wiring-spi-antar-board)
  - [Source Code: ESP32-S3 (Master)](#source-code-esp32-s3-master)
  - [Source Code: ESP32-WROOM (Slave)](#source-code-esp32-wroom-slave)
  - [Catatan & Troubleshooting](#catatan--troubleshooting-spi)
- [2. Komunikasi Nirkabel SPI dengan nRF24L01](#2-komunikasi-nirkabel-spi-dengan-nrf24l01)
  - [Konfigurasi Hardware](#2-komunikasi-nirkabel-spi-dengan-nrf24l01)
  - [Tabel Wiring / Pinout](#tabel-wiring-nrf24l01)
  - [Source Code: ESP32-S3 (Sender / Transmitter)](#source-code-esp32-s3-sender--transmitter)
  - [Source Code: ESP32 DevKit (Receiver)](#source-code-esp32-devkit-receiver)

---

## 1. Komunikasi SPI Antar Board (Board-to-Board)

Pengujian komunikasi bus SPI *full-duplex* antar mikrokontroler secara langsung dengan topologi **Master - Slave**.
- **Master:** ESP32-S3
- **Slave:** ESP32-WROOM *(digunakan karena ESP32 DevKit bermasalah)*

### Tabel Wiring (SPI Antar Board)

| Sinyal SPI | ESP32-S3 (Master) | ESP32-WROOM (Slave) | Keterangan |
| :--- | :--- | :--- | :--- |
| **MOSI** | GPIO 11 | GPIO 22 | Master Out Slave In |
| **MISO** | GPIO 13 | GPIO 19 | Master In Slave Out |
| **SCLK** | GPIO 12 | GPIO 18 | Serial Clock |
| **CS / SS** | GPIO 10 | GPIO 5 | Chip Select / Slave Select |
| **GND** | GND | GND | Common Ground (Shared GND) |

### Source Code: ESP32-S3 (Master)

```cpp
// SPI Master
#include <SPI.h>

// Konfigurasi pin HSPI
#define HSPI_MOSI 11
#define HSPI_MISO 13
#define HSPI_SCLK 12
#define HSPI_CS   10

SPIClass hspi(HSPI);

void setup() {
  Serial.begin(115200);

  // Inisialisasi HSPI
  hspi.begin(HSPI_SCLK, HSPI_MISO, HSPI_MOSI, HSPI_CS);

  pinMode(HSPI_CS, OUTPUT);
  digitalWrite(HSPI_CS, HIGH);

  Serial.println("SPI Master Siap!");
}

void loop() {
  const char *pesan = "Halo dari Master!";
  uint8_t txData[32] = {0};
  uint8_t rxData[32] = {0};

  // Salin pesan ke buffer
  strncpy((char *)txData, pesan, sizeof(txData) - 1);

  // Mulai transaksi SPI
  hspi.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));

  // Pilih Slave
  digitalWrite(HSPI_CS, LOW);

  // Kirim dan terima data sekaligus
  hspi.transferBytes(txData, rxData, sizeof(txData));

  // Lepaskan Slave
  digitalWrite(HSPI_CS, HIGH);

  hspi.endTransaction();

  Serial.print("Master mengirim: ");
  Serial.println(pesan);

  Serial.print("Master menerima: ");
  Serial.println((char *)rxData);

  Serial.println("------------------------");

  delay(2000);
}
```

### Source Code: ESP32-WROOM (Slave)

```cpp
// SPI Slave ESP32
#include <Arduino.h>
#include "driver/spi_slave.h"
#include "driver/gpio.h"

// Konfigurasi pin HSPI
#define HSPI_MOSI 22
#define HSPI_MISO 19
#define HSPI_SCLK 18
#define HSPI_CS   5

#define SPI_HOST_USED SPI2_HOST

// Buffer data
uint8_t rxData[32];
uint8_t txData[32];

void setup() {
  Serial.begin(115200);

  delay(1000);

  Serial.println("SPI Slave memulai...");

  // Konfigurasi pin SPI
  spi_bus_config_t buscfg = {};

  buscfg.mosi_io_num = HSPI_MOSI;
  buscfg.miso_io_num = HSPI_MISO;
  buscfg.sclk_io_num = HSPI_SCLK;
  buscfg.quadwp_io_num = -1;
  buscfg.quadhd_io_num = -1;

  // Konfigurasi Slave
  spi_slave_interface_config_t slvcfg = {};

  slvcfg.spics_io_num = HSPI_CS;
  slvcfg.flags = 0;
  slvcfg.queue_size = 1;
  slvcfg.mode = 0;

  // Inisialisasi SPI Slave
  esp_err_t result = spi_slave_initialize(
    SPI_HOST_USED,
    &buscfg,
    &slvcfg,
    SPI_DMA_DISABLED
  );

  if (result != ESP_OK) {
    Serial.printf("Gagal inisialisasi SPI: %d\n", result);
    while (1) {
      delay(1000);
    }
  }

  // Siapkan balasan awal
  memset(txData, 0, sizeof(txData));
  strcpy((char *)txData, "Halo juga dari Slave!");

  Serial.println("SPI Slave siap menerima data!");
}

void loop() {
  spi_slave_transaction_t trans = {};

  // Siapkan buffer transaksi
  trans.length = sizeof(rxData) * 8;
  trans.tx_buffer = txData;
  trans.rx_buffer = rxData;

  // Menunggu Master mengirimkan data
  esp_err_t result = spi_slave_transmit(
    SPI_HOST_USED,
    &trans,
    portMAX_DELAY
  );

  if (result == ESP_OK) {
    Serial.print("Slave menerima: ");
    Serial.println((char *)rxData);

    Serial.println("Transaksi selesai!");

    // Siapkan balasan untuk transaksi selanjutnya
    memset(txData, 0, sizeof(txData));
    strcpy((char *)txData, "Halo juga dari Slave!");

    // Bersihkan buffer penerimaan
    memset(rxData, 0, sizeof(rxData));
  }
}
```

### Catatan & Troubleshooting (SPI)

> [!NOTE]
> Ketika konfigurasi board ditukar menjadi **ESP32-WROOM sebagai Slave** dan **ESP32 DevKit sebagai Master**, pengiriman data dari slave bermasalah (pengiriman dari master ke slave berjalan normal, tetapi balasan dari slave tidak terbaca).

---

## 2. Komunikasi Nirkabel SPI dengan nRF24L01

Implementasi pengiriman data nirkabel menggunakan modul transceiver **nRF24L01+** yang berkomunikasi via SPI dengan mikrokontroler.
- **Transmitter (Sender):** ESP32-S3
- **Receiver:** ESP32 DevKit *(ESP32-WROOM bermasalah pada pengujian ini)*

### Tabel Wiring (nRF24L01)

| Pin nRF24L01 | ESP32-S3 (Sender) | ESP32 DevKit (Receiver) | Fungsi |
| :--- | :--- | :--- | :--- |
| **CE** | GPIO 4 | GPIO 16 | Chip Enable (RX/TX mode control) |
| **CSN** | GPIO 5 | GPIO 5 | SPI Chip Select Not |
| **SCK** | GPIO 12 | GPIO 18 | SPI Clock |
| **MOSI** | GPIO 11 | GPIO 23 | SPI MOSI |
| **MISO** | GPIO 13 | GPIO 19 | SPI MISO |
| **VCC** | 3.3V | 3.3V | Catu daya 3.3V |
| **GND** | GND | GND | Ground |

### Source Code: ESP32-S3 (Sender / Transmitter)

```cpp
#include <SPI.h>
#include <nRF24L01.h>
#include <RF24.h>

// Konfigurasi pin SPI & CE/CSN ESP32-S3
constexpr int PIN_CE   = 4;
constexpr int PIN_CSN  = 5;
constexpr int PIN_SCK  = 12;
constexpr int PIN_MISO = 13;
constexpr int PIN_MOSI = 11;

SPIClass spiBus(FSPI);
RF24 radio(PIN_CE, PIN_CSN);

const byte alamat[6] = "ESP01";

struct DataKirim {
  uint32_t nomor;
  float nilai;
};

uint32_t nomorPaket = 0;

void setup() {
  Serial.begin(115200);
  delay(500);

  // Inisialisasi SPI: SCK, MISO, MOSI, CS
  spiBus.begin(PIN_SCK, PIN_MISO, PIN_MOSI, PIN_CSN);

  if (!radio.begin(&spiBus)) {
    Serial.println("nRF24 tidak terdeteksi. Periksa kabel dan catu daya.");
    while (true) delay(1000);
  }

  radio.setPALevel(RF24_PA_LOW);
  radio.setDataRate(RF24_1MBPS);
  radio.setChannel(100);
  radio.openWritingPipe(alamat);
  radio.stopListening();

  Serial.println("Sender siap");
}

void loop() {
  DataKirim data = { nomorPaket++, 23.5f };

  bool terkirim = radio.write(&data, sizeof(data));

  Serial.print("Paket ");
  Serial.print(data.nomor);
  Serial.println(terkirim ? " terkirim" : " gagal");

  delay(1000);
}
```

### Source Code: ESP32 DevKit (Receiver)

```cpp
#include <SPI.h>
#include <nRF24L01.h>
#include <RF24.h>

// Konfigurasi pin SPI & CE/CSN ESP32 DevKit
constexpr int PIN_CE   = 16;
constexpr int PIN_CSN  = 5;
constexpr int PIN_SCK  = 18;
constexpr int PIN_MISO = 19;
constexpr int PIN_MOSI = 23;

SPIClass spiBus(VSPI);
RF24 radio(PIN_CE, PIN_CSN);

const byte alamat[6] = "ESP01";

struct DataKirim {
  uint32_t nomor;
  float nilai;
};

void setup() {
  Serial.begin(115200);
  delay(500);

  spiBus.begin(PIN_SCK, PIN_MISO, PIN_MOSI, PIN_CSN);

  if (!radio.begin(&spiBus)) {
    Serial.println("nRF24 tidak terdeteksi. Periksa kabel dan catu daya.");
    while (true) delay(1000);
  }

  radio.setPALevel(RF24_PA_LOW);
  radio.setDataRate(RF24_1MBPS);
  radio.setChannel(100);
  radio.openReadingPipe(1, alamat);
  radio.startListening();

  Serial.println("Receiver siap");
}

void loop() {
  if (radio.available()) {
    DataKirim data;
    radio.read(&data, sizeof(data));

    Serial.print("Paket diterima: ");
    Serial.print(data.nomor);
    Serial.print(" | nilai: ");
    Serial.println(data.nilai);
  }
}
```
