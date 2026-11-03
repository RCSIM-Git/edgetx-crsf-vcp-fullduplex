# 📡 EdgeTX — Dwukierunkowy CRSF USB-VCP oraz Klonowanie Telemetrii w Czasie Rzeczywistym dla GCS

[![License: GPL v2](https://img.shields.io/badge/License-GPL_v2-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)
[![EdgeTX Version](https://img.shields.io/badge/EdgeTX-2.10-orange.svg)](https://github.com/EdgeTX/edgetx)
[![Status](https://img.shields.io/badge/Status-Funkcjonalny_%26_Przetestowany-success.svg)](#-obsługiwane-aparatury-i-pliki-binarne)

> 🌐 **Language / Język:** [English](README.md) | **Polski**

Niniejszy fork systemu **EdgeTX** odblokowuje **prawdziwą komunikację dwukierunkową (Full-Duplex)** za pośrednictwem pojedynczego standardowego kabla USB-C pomiędzy aparaturą RC a komputerem PC / stacją naziemną GCS (Ground Control Station) lub panelem telemetrycznym.

Łączy prace rozwojowe EdgeTX PR **#7630** (*CRSF Trainer over USB-VCP*) z naszym autorskim **silnikiem klonowania telemetrii w trybie pełnego dupleksu (	elemetrySetMirrorCb)**, przekształcając wspierane aparatury w ultrawydajne transceivery CRSF o minimalnym opóźnieniu.

---

## 🎯 Problem i Nasze Rozwiązanie

### Przedtem:
* **Tylko tryb Simplex:** Standardowe tryby joysticka USB lub bazowy PR #7630 umożliwiały wysyłanie kanałów sterujących z PC do radia, ale **żadna telemetria nie wracała do komputera**.
* **Plątanina sprzętu:** Odbiór telemetrii na PC wymagał zewnętrznych odbiorników ExpressLRS/Crossfire, konwerterów USB-UART (FTDI) lub drugiego modułu radiowego podłączonego przez adaptery szeregowe.

### Z tym Forkiem:
* 🏎️ **Sterowanie wejściowe (PC $\rightarrow$ Radio):** Przesyłaj do 16 kanałów CRSF z częstotliwością 100–250 Hz bezpośrednio z kierownicy sim-racingowej, pedałów, gamepada lub planera autonomicznego GCS.
* 🛰️ **Telemetria wyjściowa (Radio $\rightarrow$ PC GCS):** Telemetria na żywo z Twojego modelu (Bateria, Link Statistics/RSSI/LQ/SNR, współrzędne/prędkość GPS, IMU/Attitude, Wariometr) jest klonowana w czasie rzeczywistym przez ten sam wirtualny port COM (USB-VCP) z powrotem do oprogramowania na PC.
* 🔌 **Zero przejściówek i dongli:** Dokładnie jeden standardowy kabel USB-C wpięty do górnego portu aparatury obsługuje cały ruch sterujący i telemetryczny.

---

## 🏗️ Architektura i Przepływ Danych

`	ext
[ Model RC / Dron / Łazik ]
      │  ▲
      │  │  Łącze radiowe Over-The-Air RF (ExpressLRS / Crossfire)
      ▼  │
[ Aparatura RadioMaster (Wewnętrzny / Zewnętrzny moduł RF) ]
      │  ▲
      │  │  Mikser EdgeTX i Rdzeń Klonowania Telemetrii
      ▼  │
[ Sterownik USB-VCP Full-Duplex CDC (115200 - 420000 bps) ]
      │  ▲
      │  │  Standardowy kabel USB-C
      ▼  │
[ Stacja Naziemna PC (RCSIM-GCS, Mission Planner, Rejestrator Telemetrii) ]
  • Kanały 1–16 (100–250 Hz)       ──> Wysyłane do aparatury
  • Telemetria Live (0x14, 0x08, 0x02, 0x86) <── Odbierana przez GCS
`

---

## 📦 Obsługiwane Aparatury i Pliki Binarne

| Model Aparatury | Architektura Docelowa | Optymalizacja Flash | Status | Uwagi |
|:---|:---|:---:|:---:|:---|
| **RadioMaster MT12** | Pistoletowa (Auta/Łodzie) | Tak (Usunięto PXX/Heli) | ✅ Przetestowany i Potwierdzony | Aparatura kołowa/pistoletowa |
| **RadioMaster Pocket** | Kompaktowa ręczna | Tak (Limit 512 KB) | ✅ Przetestowany i Potwierdzony | Zoptymalizowany pod Flash 512KB |
| **RadioMaster Boxer** | Pełnowymiarowa | Standardowa | ⚠️ Eksperymentalny | Pełny rozmiar pamięci Flash |
| **RadioMaster TX16S / MKII** | Kolorowy ekran LCD Touch | Standardowa | ⚠️ Eksperymentalny | Cel kolorowego LCD |
| **RadioMaster TX12 MKII** | Kompaktowa | Tak (Limit 512 KB) | ⚠️ Eksperymentalny | Zoptymalizowany pod Flash 512KB |
| **RadioMaster TX12 (V1)** | Kompaktowa | Tak (Limit 512 KB) | ⚠️ Eksperymentalny | Zoptymalizowany pod Flash 512KB |
| **RadioMaster Zorro** | Typ Gamepad | Tak (Limit 512 KB) | ⚠️ Eksperymentalny | Zoptymalizowany pod Flash 512KB |

---

## ⚙️ Jak to Działa (Implementacja C++)

Modyfikacja wprowadza poprawki w module trasowania portów szeregowych EdgeTX (
adio/src/serial.cpp):
1. **Inicjalizacja portu dwukierunkowego:** Ustawienie params.direction = ETX_Dir_TX_RX na sterowniku portu USB-VCP.
2. **Podpięcie pod strumień telemetrii:** Rejestracja dedykowanego wywołania zwrotnego (callback) do 	elemetrySetMirrorCb(ctx, sendByte) w momencie aktywacji trybu CRSF Trainer przez VCP:
   `cpp
   if (drv && drv->setReceiveCb) {
       crsfTrainerStart(ctx, drv);
       telemetrySetMirrorCb(ctx, sendByte); // Klonuje telemetrię CRSF na żywo przez USB-VCP
   } else {
       crsfTrainerStop();
       telemetrySetMirrorCb(nullptr, nullptr);
   }
   `
3. **Optymalizacja dla układów z pamięcią Flash 512 KB:** Zastosowano agresywne flagi Link-Time Optimization (LTO) oraz wyłączono nieużywane, przestarzałe protokoły (PXX1, PXX2, HELI, GHOST), dzięki czemu firmware mieści się stabilnie w pamięci STM32 o pojemności 512KB bez utraty kluczowych funkcji EdgeTX.

---

## 🛠️ Krok 1: Wgrywanie Oprogramowania do Aparatury

Najbezpieczniejszą metodą aktualizacji firmware jest użycie wbudowanego bootloadera EdgeTX:

1. **Dostęp do karty pamięci SD:**
   - Włącz aparaturę, podłącz ją do komputera kablem USB-C i wybierz opcję **USB Storage (SD)**.
   - Alternatywnie wyjmij kartę MicroSD z aparatury i umieść ją w czytniku kart PC.
2. **Kopiowanie pliku oprogramowania:**
   - Przejdź do folderu FIRMWARE/ w głównym katalogu karty MicroSD.
   - Skopiuj odpowiedni plik .bin dla Twojego modelu aparatury do folderu FIRMWARE/.
3. **Bezpieczne odłączenie i uruchomienie Bootloadera:**
   - Bezpiecznie wysuń nośnik SD / odłącz kabel USB.
   - Wyłącz aparaturę.
   - Przytrzymaj oba poziome trymery do środka (w stronę przycisku zasilania) i naciśnij przycisk **Zasilania**, aby wejść w tryb **EdgeTX Bootloader**.
4. **Wgranie firmware (Flash):**
   - Wybierz opcję **Write Firmware** na ekranie aparatury.
   - Wybierz skopiowany plik .bin i zatwierdź, przytrzymując przycisk Enter / rolkę.
   - Po osiągnięciu 100% paska postępu wybierz **Exit**, aby zrestartować aparaturę w normalnym trybie EdgeTX.

---

## ⚙️ Krok 2: Konfiguracja Menu w EdgeTX

Skonfiguruj model w aparaturze, aby akceptował dane trenera z wirtualnego portu szeregowego USB:

### 1. Ustawienia Systemowe (SYS)
* **Radio Setup (SYS -> Radio Setup):**
  - Ustaw USB Mode na **VCP** (lub **Ask** i wybieraj *VCP / Serial* przy każdym podłączeniu kabla USB).
* **Hardware Configuration (SYS -> Hardware):**
  - Przewiń do ustawienia portu USB-VCP (lub VCP) i zmień jego tryb na **CRSF Trainer**.

### 2. Konfiguracja Modelu (MDL)
* **Tryb Trenera (MDL -> Model Setup):**
  - Przewiń w dół do sekcji **Trainer**:
    - Mode: **CRSF**
    - Channels: **CH1 - CH16**
* **Przypisanie Funkcji Trenera (MDL -> Special Functions):**
  - Dodaj nową funkcję specjalną:
    - Switch: ON (lub przypisz fizyczny przełącznik, np. SA / SB, do aktywacji sterowania z PC)
    - Action: **Trainer**
    - Value: **Sticks** (lub **Axis**)
    - Enable: Zaznaczone (ON)

---

## 🧪 Krok 3: Weryfikacja i Diagnostyka

Aby zweryfikować dwukierunkową komunikację za pomocą skryptu Python:

`ash
python crsf_vcp_duplex_test.py --port COM5 --baud 115200
`

**Oczekiwane komunikaty w konsoli:**
`	ext
======================================================================
  CRSF USB-VCP Full-Duplex Test
======================================================================
Opened COM5 (115200 bps).
[TX] 100 Hz Control loop active (Sending 16 RC channels)...
   <-- [TELEMETRY] LINK STATS: RSSI: -42 dBm | LQ: 100% | SNR: 12 dB
   <-- [TELEMETRY] BATTERY: 8.35 V | 1.2 A | 450 mAh
   <-- [TELEMETRY] GPS: 52.2297°N, 21.0122°E | Speed: 24.1 km/h | Sats: 16
`

---

## 🏎️ Krok 4: Integracja ze Stacją Naziemną RCSIM-GCS

1. Uruchom aplikację **RCSIM-GCS** na komputerze PC.
2. Przejdź do zakładki **Kokpit / Połączenie**:
   - Wybierz wirtualny port COM aparatury (np. COM5 - STMicroelectronics Virtual COM Port).
   - Prędkość transmisji (Baudrate): 115200 bps.
   - Protokół: CRSF Direct (USB-VCP).
3. Przypisz osie kierownicy sim-racingowej, pedałów (gazu i hamulca) oraz hamulca ręcznego w zakładce **Konfiguracja Wejść**.
4. Uzbrój pojazd (ARM) – sterowanie wraz z siłowym sprzężeniem zwrotnym FFB z fizycznej telemetrii modelu jest aktywne!

---

## 🛠️ Budowanie ze Źródeł

Kompletny zestaw flag CMake, wymagania dotyczące kompilatora krzyżowego (rm-none-eabi-gcc) oraz powtarzalne instrukcje budowania dla wszystkich obsługiwanych aparatur znajdują się w pliku [BUILD_INSTRUCTIONS.md](BUILD_INSTRUCTIONS.md).

---

## 📜 Licencja i Informacje o Autorstwie

* **Projekt bazowy:** [EdgeTX](https://github.com/EdgeTX/edgetx)
* **Bazowy Pull Request:** PR [#7630](https://github.com/EdgeTX/edgetx/pull/7630) autorstwa BelixRogner (eat/crsf-trainer-over-usb-vcp, commit e5784ee5)
* **Kompletny kod źródłowy:** Opublikowany w repozytorium [RCSIM-Git/edgetx-crsf-vcp-fullduplex](https://github.com/RCSIM-Git/edgetx-crsf-vcp-fullduplex).
* **Licencja:** Oprogramowanie dystrybuowane na licencji [GNU General Public License v2.0 (GPL-2.0)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html).
* **Zastrzeżenie:** Nieoficjalne kompilacje społecznościowe rozwijane dla projektu RCSIM. Udostępniane bez jakiejkolwiek gwarancji.
