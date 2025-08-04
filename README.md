#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>
#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Max72xxPanel.h>
#include <EEPROM.h>
#include <DNSServer.h>
#include <FS.h>
#include <math.h>
#include <ArduinoJson.h>

#define MAX_MACROS 15
#define INITIAL_MACRO_COUNT 6
#define CUSTOM_MACRO_COUNT (MAX_MACROS - INITIAL_MACRO_COUNT)
#define MAX_MACRO_LENGTH 64

enum DisplayMode {
    MODE_SCROLLING,
    MODE_STATIC,
    MODE_EFFECTS
};
enum AnimationEffect {
    EFFECT_NONE,
    EFFECT_FLASHLIGHT,
    EFFECT_POLICE_SIREN
};

struct EepromData {
    byte brightness_percent;
    byte speed;
    byte speed_percent;
    byte displayMode;
    byte animationEffect;
    int macroCount;
    char macros[MAX_MACROS][MAX_MACRO_LENGTH + 1];
    int battery_capacity_mah;
};
#define EEPROM_SIZE sizeof(EepromData)

const char* ssid = "SedLight";
const char* pwd = "14041990";
ESP8266WebServer server(80);
DNSServer dnsServer;

Max72xxPanel matrix = Max72xxPanel(15, 1, 2); // 15 модулей MAX7219, 1 строка, 2 столбца (или 15x8 матрица, 1 дисплей, 2 каскада)

EepromData settings;
byte liveBrightness;
int liveSpeed;
bool brightnessChanged = false;

char currentTapeText[MAX_MACRO_LENGTH + 1] = "Не сбивайте меня пожалуйста";
int tickerSpacer = 1;
int tickerCharWidth = 5 + tickerSpacer; // Ширина символа + пробел
int currentPos = 0; // Текущая позиция прокрутки текста
size_t initialTotalHeap = 0; // Общий объем кучи при старте
static String displayText = ""; // Текст для отображения (используется для расчета ширины)
static int textWidth = 0; // Ширина текста в пикселях

// Удалены переменные для измерения температуры

float currentBatteryVoltage = 0.0; // Текущее напряжение батареи
float batteryPercentage = 0.0; // Процент заряда батареи
unsigned long lastBatteryReadTime = 0; // Время последнего измерения батареи
const int SENSOR_POWER_PIN = D4; // Пин для управления питанием датчиков (D4 на ESP8266)
const float R1_BATTERY = 330000.0; // Резистор R1 для делителя напряжения батареи
const float R2_BATTERY = 100000.0; // Резистор R2 для делителя напряжения батареи
const float ADC_VOLTAGE = 1.0; // Опорное напряжение АЦП ESP8266 (1.0V)
const float BATTERY_MIN_V = 3.0; // Минимальное напряжение батареи (разряжено)
const float BATTERY_MAX_V = 4.2; // Максимальное напряжение батареи (полностью заряжено)

const float VOLTAGE_CALIBRATION_FACTOR = 0.9468; // Коэффициент калибровки напряжения батареи

// Удалены константы для термистора

unsigned long lastFlashlightFrameTime = 0; // Время последнего кадра эффекта "Фонарик"
int flashlightFrameIndex = 0; // Индекс кадра для эффекта "Фонарик"
bool flashlightGrowing = true; // Направление анимации "Фонарик" (растет/уменьшается)
const int flashlightDelay = 100; // Задержка между кадрами "Фонарика"

unsigned long lastPoliceFlashTime = 0; // Время последней вспышки эффекта "Полиция"
bool policeFlashState = false; // Текущее состояние вспышки (вкл/выкл)
int policeFlashesInBlock = 0; // Количество вспышек в текущем блоке
int policeCurrentBlock = 0; // Текущий блок вспышек (0 - левая сторона, 1 - правая)
const int policeFlashesPerBlock = 3; // Количество вспышек на один блок
const int policeFlashDelay = 100; // Задержка между вспышками "Полиции"

// HTML-страница для веб-интерфейса, хранится в PROGMEM для экономии ОЗУ
const char page[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Панель Управления: Бегущая Строка</title>
    <style>
        html, body { margin: 0; padding: 0; }
        body { line-height: 1.0; }
        h1, h2, h3, h4, h5, h6, p, ul, ol, li { margin: 0; padding: 0; }
        input, button, select, textarea { font-family: inherit; font-size: inherit; }
        * { box-sizing: border-box; }

        :root {
            --bg-color: #1a1a2e;
            --card-bg: #2a2a3e;
            --accent-color: #6a5acd;
            --accent-color-hover: #8a7acd;
            --text-color: #e0e0e0;
            --secondary-text-color: #b0b0b0;
            --input-bg: #3c3c5a;
            --delete-color: #e74c3c;
            --delete-color-hover: #c0392b;
            --border-radius: 12px;
            --gap-size: 8px;
            --padding-card: 12px;
        }

        body {
            font-family: 'Segoe UI', Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            padding: var(--gap-size);
            display: flex;
            flex-direction: column;
            align-items: center;
            min-height: 10vh;
            font-size: 0.9rem;
        }

        .main-container {
            width: 100%;
            max-width: 800px;
            display: flex;
            flex-direction: column;
            gap: var(--gap-size);
        }

        .status-bar {
            width: 100%;
            background-color: var(--card-bg);
            border-radius: var(--border-radius);
            padding: 10px 12px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
            font-size: 0.9em;
            font-weight: bold;
        }

        .status-bar-left, .status-bar-right {
            display: flex;
            align-items: center;
            gap: 8px;
        }

        .battery-icon {
            font-size: 1.2em;
            line-height: 1;
            width: 25px;
            height: 14px;
            border: 2px solid var(--text-color);
            border-radius: 3px;
            position: relative;
            box-sizing: content-box;
        }

        .battery-icon::after {
            content: '';
            position: absolute;
            top: 50%;
            right: -5px;
            transform: translateY(-50%);
            width: 3px;
            height: 6px;
            background-color: var(--text-color);
            border-radius: 0 1px 1px 0;
        }

        .battery-level {
            height: 100%;
            background-color: #4CAF50; /* Green */
            border-radius: 1px;
            transition: width 0.5s ease, background-color 0.5s ease;
        }

        .battery-icon.low .battery-level { background-color: #F44336; } /* Red */
        .battery-icon.medium .battery-level { background-color: #FFC107; } /* Orange */
        
        .tab-container {
            width: 100%;
            background-color: var(--card-bg);
            border-radius: var(--border-radius);
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
            padding: 0;
            display: flex;
            flex-direction: column;
            min-height: 500px;
        }
        
        .tab-buttons {
            display: flex;
            width: 100%;
            background-color: var(--card-bg);
            border-radius: var(--border-radius) var(--border-radius) 0 0;
            border-bottom: 2px solid rgba(255, 255, 255, 0.1);
        }
        
        .tab-button {
            flex-grow: 1;
            padding: 15px 0;
            text-align: center;
            background-color: var(--card-bg);
            border: none;
            color: var(--secondary-text-color);
            font-size: 1em;
            cursor: pointer;
            transition: color 0.3s ease, background-color 0.3s ease;
            position: relative;
        }
        
        .tab-button.active {
            color: var(--accent-color);
            background-color: var(--bg-color);
            border-radius: 0;
        }

        .tab-button:first-child.active {
            border-radius: var(--border-radius) 0 0 0;
        }
        
        .tab-button:last-child.active {
            border-radius: 0 var(--border-radius) 0 0;
        }
        
        .tab-button:first-child {
            border-radius: var(--border-radius) 0 0 0;
        }
        
        .tab-button:last-child {
            border-radius: 0 var(--border-radius) 0 0;
        }
        
        .tab-button:not(:last-child) {
             border-right: 1px solid rgba(255, 255, 255, 0.1);
        }
        
        .tab-content {
            padding: var(--gap-size);
            flex-grow: 1;
            display: none;
            animation: fadeIn 0.5s ease-out;
            background-color: var(--bg-color);
            border-radius: 0 0 var(--border-radius) var(--border-radius);
        }
        
        .tab-content.active {
            display: block;
        }

        .dashboard-container {
            width: 100%;
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
            gap: var(--gap-size);
            justify-content: center;
        }

        .card.full-width {
            grid-column: 1 / -1;
        }
        
        .card {
            background: linear-gradient(to bottom right, var(--card-bg), var(--temp-color, #5a60c0));
            border-radius: var(--border-radius);
            padding: var(--padding-card);
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
            display: flex;
            flex-direction: column;
            gap: 7px;
            position: relative;
            overflow: hidden;
            border: none;
            transition: transform 0.2s ease, box-shadow 0.2s ease, background 0.5s ease-in-out;
        }

        .card:hover {
            transform: translateY(-1px);
            box-shadow: 0 6px 12px rgba(0, 0, 0, 0.3);
        }

        .card-header {
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 5px;
            font-size: 1em;
            font-weight: bold;
            color: var(--text-color);
            padding-bottom: 5px;
            border-bottom: 1px solid rgba(255, 255, 255, 0.1);
            margin-bottom: 5px;
        }

        .card-header .icon {
            /* display: none; */ /* Убрано для прямого отображения иконки */
            font-size: 1.2em; /* Добавлено для размера иконки */
            line-height: 1; /* Добавлено для выравнивания */
        }

        .card.wifi-info .card-header::before { content: '📡'; }
        .card.system-info .card-header::before { content: '🧠'; }
        /* .card.temperature-info .card-header::before { content: '🌡️'; } */ /* Удалено */
        .card.text-input .card-header::before { content: '✍️'; }
        .card.macros .card-header::before { content: '⚡'; }
        .card.visual-effects .card-header::before { content: '✨'; }
        .card.management .card-header::before { content: '🔧'; }
        /* .card.battery-info .card-header::before { content: '🔋'; } */ /* Убрано, иконка теперь в HTML */

        .input-group {
            display: flex;
            gap: 4px;
            width: 100%;
        }
        
        .input-group-vertical {
            display: flex;
            flex-direction: column;
            gap: 4px;
            width: 100%;
        }

        .input-group-vertical input[type="number"] {
            width: 100%;
        }

        .input-group-vertical label {
            color: var(--secondary-text-color);
            font-size: 0.9em;
        }

        .select-group {
            display: flex;
            flex-direction: column;
            gap: 4px;
            width: 100%;
        }

        .select-group label {
            color: var(--secondary-text-color);
            font-size: 0.9em;
        }

        .input-group input[type="text"],
        .input-group-vertical input[type="number"],
        .select-group select {
            flex-grow: 1;
            padding: 8px 9px;
            border: 1px solid rgba(255, 255, 0, 0.1);
            border-radius: var(--border-radius);
            background-color: var(--input-bg);
            color: var(--text-color);
            font-size: 0.9em;
            outline: none;
            transition: border-color 0.3s ease;
            box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.2);
        }
        
        .select-group select,
        .select-group label {
            font-size: 0.9em;
        }

        .input-group input[type="text"]:focus,
        .input-group-vertical input[type="number"]:focus,
        .select-group select:focus {
            border-color: var(--accent-color);
            box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.3), 0 0 5px rgba(106, 90, 205, 0.5);
        }
        
        .input-group-vertical input[type="number"] {
             text-align: center;
        }


        .input-group input[type="text"]::placeholder {
            color: var(--secondary-text-color);
        }
        .input-group-vertical input[type="number"]::placeholder {
            color: var(--secondary-text-color);
        }

        .button,
        .input-group input[type="submit"] {
            padding: 8px 14px;
            border: none;
            border-radius: var(--border-radius);
            background-color: var(--accent-color);
            color: white;
            font-size: 0.9em;
            cursor: pointer;
            transition: background-color 0.3s ease, transform 0.1s ease, box-shadow 0.3s ease;
            white-space: nowrap;
            box-shadow: 0 2px 4px rgba(0,0,0,0.3);
        }

        .button[disabled],
        .input-group input[type="submit"][disabled] {
            opacity: 0.5;
            cursor: not-allowed;
            box-shadow: none;
        }

        .button.red {
            background-color: var(--delete-color);
        }

        .button.red:hover {
            background-color: var(--delete-color-hover);
        }

        .button.orange {
            background-color: #ff9800;
        }
        .button.orange:hover {
            background-color: #f57c00;
        }

        .button:hover,
        .input-group input[type="submit"]:hover {
            background-color: var(--accent-color-hover);
            transform: translateY(-2px);
            box-shadow: 0 4px 8px rgba(0,0,0,0.4);
        }
        .input-group-vertical .button {
            width: 100%;
        }

        .macros-container {
             display: flex;
             flex-direction: column;
             gap: 6px;
        }

        .macros-grid {
            display: flex;
            flex-wrap: wrap;
            gap: 6px;
        }

        .macro-item {
            display: flex;
            align-items: center;
            gap: 4px;
            background-color: var(--input-bg);
            border-radius: var(--border-radius);
            border: 1px solid rgba(255, 255, 255, 0.1);
            overflow: hidden;
            box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
            transition: transform 0.1s ease, box-shadow 0.1s ease;
        }

        .macro-item:hover {
            transform: translateY(-1px);
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
        }

        .macro-item .macro-button {
            background: none;
            color: var(--text-color);
            border: none;
            cursor: pointer;
            padding: 6px;
            font-size: 0.9em;
            line-height: 1.2;
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
            flex-grow: 1;
            text-align: left;
            border-radius: 0;
        }

        .macro-item .macro-button:hover {
            background-color: var(--card-bg);
        }

        .macro-item .delete-button {
            background-color: var(--delete-color);
            color: white;
            border: none;
            cursor: pointer;
            padding: 6px 8px;
            font-size: 0.9em;
            line-height: 1;
            transition: background-color 0.3s ease;
        }

        .macro-item .delete-button:hover {
            background-color: var(--delete-color-hover);
        }

        .info-card h3 {
            font-size: 1.2em;
            margin-top: 0;
            margin-bottom: 5px;
            text-align: center;
            color: var(--text-color);
            line-height: 1.1;
        }

        .info-card p {
            font-size: 0.8em;
            color: var(--secondary-text-color);
            margin: 0;
            text-align: center;
            line-height: 1.1;
            margin-bottom: 7px;
        }

        .info-item {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 1px 0;
            border-bottom: 1px dashed rgba(255, 255, 255, 0.05);
            font-size: 0.9em;
            line-height: 1.1;
            margin: 0;
        }

        .info-item:last-child {
            border-bottom: none;
        }

        .info-label {
            color: var(--secondary-text-color);
        }

        .info-value {
            color: var(--text-color);
            font-weight: bold;
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
        }
        .info-value.error {
            color: #ff5252;
        }
        .info-value.no-data {
            color: var(--secondary-text-color);
        }

        .settings .input-group {
            margin-top: 10px;
        }

        .slider-group-horizontal,
        .select-group-horizontal {
            display: flex;
            flex-direction: row;
            gap: 15px;
            align-items: center;
            justify-content: space-around;
            width: 100%;
            padding: 0 5px;
        }

        .select-group-horizontal {
             flex-wrap: wrap;
        }

        .select-group-horizontal .select-group {
            flex: 1 1 45%; /* Allows two items per row on larger screens */
        }

        .slider-wrapper {
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 5px;
            width: 100%;
            max-width: 250px;
        }

        .slider-wrapper label {
            font-size: 0.9em;
            font-weight: bold;
            color: var(--text-color);
            text-align: center;
        }

        .slider-wrapper input[type="range"] {
            width: 100%;
            -webkit-appearance: none; /* Remove default styling for WebKit browsers */
            background: transparent; /* Make track transparent */
            cursor: pointer;
        }

        /* Styles for WebKit browsers (Chrome, Safari) */
        .slider-wrapper input[type="range"]::-webkit-slider-runnable-track {
            width: 100%;
            height: 8px;
            background: linear-gradient(to right, var(--accent-color) 0%, var(--accent-color) 0%, var(--input-bg) 0%, var(--input-bg) 100%);
            border-radius: 5px;
            border: 1px solid rgba(255, 255, 255, 0.1);
            box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.2);
        }

        .slider-wrapper input[type="range"]::-moz-range-track {
            width: 100%;
            height: 8px;
            background: linear-gradient(to right, var(--accent-color) 0%, var(--accent-color) 0%, var(--input-bg) 0%, var(--input-bg) 100%);
            border-radius: 5px;
            border: 1px solid rgba(255, 255, 255, 0.1);
            box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.2);
        }

        .slider-wrapper input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none; /* Remove default thumb styling */
            height: 20px;
            width: 20px;
            background: #ffffff;
            border-radius: 50%;
            cursor: grab;
            margin-top: -6.5px; /* Adjust to center thumb vertically */
            box-shadow: 0 1px 4px rgba(0, 0, 0, 0.4);
            border: none;
            transition: box-shadow 0.3s ease;
        }

        .slider-wrapper input[type="range"]::-webkit-slider-thumb:active {
            cursor: grabbing;
            box-shadow: 0 1px 6px rgba(0, 0, 0, 0.6);
        }

        /* Styles for Firefox */
        .slider-wrapper input[type="range"]::-moz-range-thumb {
            height: 20px;
            width: 20px;
            background: #ffffff;
            border-radius: 50%;
            cursor: grab;
            border: none;
            box-shadow: 1px 4px rgba(0, 0, 0, 0.4);
            transition: box-shadow 0.3s ease;
        }
        .slider-wrapper input[type="range"]::-moz-range-thumb:active {
            cursor: grabbing;
            box-shadow: 0 1px 6px rgba(0, 0, 0, 0.6);
        }

        .slider-wrapper input[type="range"]::-moz-range-progress {
          background: var(--accent-color);
          border-radius: 5px;
          height: 8px;
        }

        /* Disabled slider styles */
        .slider-wrapper input[type="range"]:disabled::-webkit-slider-runnable-track {
            background: linear-gradient(to right, var(--secondary-text-color) 0%, var(--secondary-text-color) 0%, var(--input-bg) 0%, var(--input-bg) 100%) !important;
        }

        .slider-wrapper input[type="range"]:disabled::-moz-range-progress {
          background: var(--secondary-text-color) !important;
        }

        .slider-wrapper input[type="range"]:disabled::-webkit-slider-thumb,
        .slider-wrapper input[type="range"]:disabled::-moz-range-thumb {
            background: var(--secondary-text-color) !important;
            cursor: not-allowed !important;
        }

        .slider-wrapper input[type="range"]:disabled {
             opacity: 0.5;
             cursor: not-allowed;
        }

        .slider-minmax-labels {
            display: flex;
            justify-content: space-between;
            width: 100%;
            font-size: 0.65em;
            color: var(--secondary-text-color);
            margin-top: -3px;
        }

        .slider-value {
            font-size: 0.75em;
            color: var(--secondary-text-color);
            text-align: center;
        }

        .slider-group-horizontal {
            gap: 20px;
        }

        .center-button-group {
            display: flex;
            justify-content: center;
            width: 100%;
        }

        .button-group-vertical {
            display: flex;
            flex-direction: column;
            gap: 6px;
        }

        .button-group-vertical .button {
            width: 100%;
            text-align: center;
        }

        /* Удалены стили для #tempCard */

        #batteryCard {
            background: linear-gradient(to bottom right, var(--card-bg), var(--battery-color, #5a60c0));
            transition: background 0.5s ease-in-out;
        }

        /* Удалены стили для #tempCard h3 */

        .temp-scale-container {
            display: flex;
            flex-direction: column;
            align-items: center;
            width: 100%;
            margin-bottom: 5px;
        }

        .scale-labels-top, .scale-labels-bottom {
            display: flex;
            justify-content: space-between;
            width: 100%;
            padding: 0 5px;
            font-size: 0.75em;
            color: var(--secondary-text-color);
        }

        .scale-labels-top span:first-child { text-align: left; }
        .scale-labels-top span:last-child { text-align: right; }

        .scale-labels-bottom {
            justify-content: space-between;
            margin-top: 5px;
            padding: 0 5px;
        }
        .scale-labels-bottom span:nth-child(2) {
            transform: translateX(-50%);
        }

        .temp-scale {
            position: relative;
            width: 100%;
            height: 10px;
            background: linear-gradient(to right,
                hsl(240, 70%, 50%), /* Blue */
                hsl(180, 70%, 50%), /* Cyan */
                hsl(120, 70%, 50%), /* Green */
                hsl(60, 70%, 50%),  /* Yellow */
                hsl(0, 70%, 50%)    /* Red */
            );
            border-radius: 5px;
            overflow: hidden;
            margin-top: 5px;
            border: 1px solid rgba(255, 255, 255, 0.1);
            box-shadow: inset 0 1px 3px rgba(0,0,0,0.2);
        }

        .temp-scale.battery-scale {
             background: linear-gradient(to right,
                hsl(0, 70%, 50%),   /* Red */
                hsl(60, 70%, 50%),  /* Yellow */
                hsl(120, 70%, 50%)  /* Green */
            );
        }

        .temp-indicator {
            position: absolute;
            top: -2px;
            height: 14px;
            width: 4px;
            background-color: white;
            border-radius: 2px;
            box-shadow: 0 0 5px rgba(0, 0, 0, 0.5);
            transition: left 0.5s ease;
        }

        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(-10px); }
            to { opacity: 1; transform: translateY(0); }
        }

        @media (max-width: 600px) {
            body { padding: 5px; font-size: 0.85rem; }
            .status-bar { font-size: 0.85em; padding: 8px 10px; }
            .battery-icon { width: 22px; height: 12px; }
            .card { padding: 10px; gap: 6px; }
            .card-header { font-size: 0.95em; gap: 4px; padding-bottom: 4px; margin-bottom: 4px; }
            .card-header .icon { font-size: 1.05em; }
            .input-group input[type="text"], .button, .input-group input[type="submit"] { padding: 7px 10px; font-size: 0.8em; }
            .macros-grid { gap: 4px; margin-top: 4px; }
            .macro-item { gap: 3px; }
            .macro-item .macro-button, .macro-item .delete-button { padding: 5px; font-size: 0.8em; }
            .info-card h3 { font-size: 1.05em; margin-bottom: 1px; }
            .info-card p { font-size: 0.7em; margin-bottom: 5px; }
            .info-item { padding: 0; font-size: 0.8em; }
            .slider-group-horizontal, .select-group-horizontal { flex-direction: column; gap: 10px; }
            .slider-wrapper, .select-group { max-width: 100%; }
            .slider-wrapper input[type="range"] { width: 90%; }
            .slider-value { font-size: 0.7em; margin-top: 3px; }
        }

        .simple-footer {
            width: 100%;
            max-width: 800px;
            text-align: center;
            color: var(--secondary-text-color);
            margin-top: 20px;
            padding: 10px;
            font-size: 0.8em;
        }
    </style>
</head>
<body>
    <div class="main-container">
        <div class="status-bar">
            <div class="status-bar-left">
                <span id="displayModeStatus">Бегущая строка</span>
            </div>
            <div class="status-bar-right">
                <span id="batteryPercentStatus">--</span>
                <div id="batteryIcon" class="battery-icon">
                    <div id="batteryLevel" class="battery-level"></div>
                </div>
            </div>
        </div>
        
        <div class="tab-container">
            <div class="tab-buttons">
                <button class="tab-button active" onclick="showTab('main')">Главный экран</button>
                <button class="tab-button" onclick="showTab('service')">Сервис</button>
            </div>
            
            <div id="mainTab" class="tab-content active">
                <div class="dashboard-container">
                    <div class="card visual-effects full-width">
                        <div class="card-header">
                            <span class="icon"></span><h2>Режим отображения</h2>
                        </div>
                        <div class="select-group-horizontal">
                            <div class="select-group">
                                <label for="displayModeSelect">Основной режим:</label>
                                <select id="displayModeSelect" onchange="saveSetting('displayMode', this.value); updateUIForMode(this.value);">
                                    <option value="0">Бегущая строка</option>
                                    <option value="2">Эффекты</option>
                                </select>
                            </div>
                            <div class="select-group" id="animationEffectGroup">
                                <label for="animationEffectSelect">Визуальный эффект:</label>
                                <select id="animationEffectSelect" onchange="saveSetting('animationEffect', this.value); updateUIForMode(document.getElementById('displayModeSelect').value);">
                                    <option value="1">Фонарь</option>
                                    <option value="2">Полиция</option>
                                </select>
                            </div>
                        </div>
                        <div class="slider-group-horizontal" style="margin-top: 15px;">
                            <div class="slider-wrapper">
                                <label for="brightnessSlider">Яркость (%):</label>
                                <input type="range" id="brightnessSlider" name="brightness_percent" min="0" max="100" value="100" oninput="updateLiveSetting('brightness_percent', this.value); updateSliderValue('brightnessSliderValue', this.value); updateSliderFill(this);" onchange="debouncedSaveSetting('brightness_percent', this.value);">
                                <div class="slider-minmax-labels">
                                    <span>min</span>
                                    <span>max</span>
                                </div>
                                <div class="slider-value"><span id="brightnessSliderValue">100</span>%</div>
                            </div>
                            <div class="slider-wrapper" id="speedSliderWrapper">
                                <label for="speedSlider">Скорость (%):</label>
                                <input type="range" id="speedSlider" name="speed_percent" min="0" max="100" value="0" oninput="updateLiveSetting('speed_percent', this.value); updateSliderValue('speedSliderValue', this.value); updateSliderFill(this);" onchange="debouncedSaveSetting('speed_percent', this.value);">
                                <div class="slider-minmax-labels">
                                    <span>min</span>
                                    <span>max</span>
                                </div>
                                <div class="slider-value"><span id="speedSliderValue">0</span>%</div>
                                <div class="center-button-group">
                                    <button type="button" class="button" onclick="setOptimalSpeed()">Оптимальная скорость</button>
                                </div>
                            </div>
                        </div>
                    </div>

                    <div class="card macros full-width" id="macrosCard">
                        <div class="card-header">
                            <span class="icon"></span><h2>Макросы</h2>
                        </div>
                        <div class="macros-container">
                            <div class="macros-grid" id="macroButtons">
                            </div>
                        </div>
                        <form action="/add_macro" method="post" class="input-group" id="addMacroForm" style="margin-top: 7px;">
                            <input type="text" name="new_macro_text" placeholder="Добавить новый макрос..." required maxlength="60">
                            <button type="submit" class="button">Добавить</button>
                        </form>
                    </div>

                    <div class="card text-input full-width" id="textInputCard">
                        <div class="card-header">
                            <span class="icon"></span><h2>Свой текст</h2>
                        </div>
                        <form action="/send_text" method="post" class="input-group">
                            <input type="text" name="text" id="textInput" placeholder="Введите текст здесь..." required>
                            <input type="submit" value="Отправить" class="button">
                        </form>
                    </div>
                </div>
            </div>
            
            <div id="serviceTab" class="tab-content">
                <div class="dashboard-container">
                    <div class="card wifi-info info-card">
                        <div class="card-header">
                            <span class="icon"></span><h2>Wi-Fi</h2>
                        </div>
                        <h3><span id="wifiIpAddress">0.0.0.0</span></h3>
                        <p>IP Адрес Устройства</p>
                        <div class="info-item">
                            <span class="info-label">SSID:</span>
                            <span class="info-value" id="wifiSsid">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Клиенты:</span>
                            <span class="info-value" id="wifiConnectedClients">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">MAC Адрес:</span>
                            <span class="info-value" id="wifiMacAddress">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Канал:</span>
                            <span class="info-value" id="wifiChannel">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Режим Wi-Fi:</span>
                            <span class="info-value" id="wifiMode">--</span>
                        </div>
                    </div>

                    <div class="card system-info info-card">
                        <div class="card-header">
                            <span class="icon"></span><h2>System</h2>
                        </div>
                        <h3><span id="memOverview">--</span></h3>
                        <p>Память ОЗУ</p>
                        <div class="info-item">
                            <span class="info-label">Общая Флеш:</span>
                            <span class="info-value" id="flashTotal">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Размер Скетча:</span>
                            <span class="info-value" id="sketchSize">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">ID Чипа:</span>
                            <span class="info-value" id="chipId">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Версия Ядра SDK:</span>
                            <span class="info-value" id="sdkVersion">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Частота ЦПУ:</span>
                            <span class="info-value" id="cpuFreq">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Время работы:</span>
                            <span class="info-value" id="uptime">--</span>
                        </div>
                    </div>

                    <!-- Удален блок "Температура" -->

                    <div class="card battery-info info-card full-width" id="batteryCard">
                        <div class="card-header">
                            <span class="icon">🔋</span><h2>Батарея</h2>
                        </div>
                        <h3><span id="batteryPercent">--</span></h3>
                        <p>Заряд</p>

                        <div class="temp-scale-container">
                            <div class="scale-labels-top">
                                <span>1%</span>
                                <span>100%</span>
                            </div>
                            <div class="temp-scale battery-scale">
                                <div class="temp-indicator" id="batteryIndicator"></div>
                            </div>
                            <div class="scale-labels-bottom">
                                <span>3.0V</span>
                                <span>3.7V</span>
                                <span>4.2V</span>
                            </div>
                        </div>

                        <div class="info-item">
                            <span class="info-label">Напряжение:</span>
                            <span class="info-value" id="batteryVoltage">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Состояние:</span>
                            <span class="info-value" id="batteryStatus">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Прогноз:</span>
                            <span class="info-value" id="batteryPrediction">--</span>
                        </div>
                        <div class="info-item">
                            <span class="info-label">Последний замер:</span>
                            <span class="info-value" id="lastBatteryRead">--</span>
                        </div>
                    </div>

                    <div class="card management full-width">
                        <div class="card-header">
                            <span class="icon"></span><h2>Коррекция и сброс</h2>
                        </div>
                        <div class="button-group-vertical">
                            <form action="/save_settings" method="post" class="input-group-vertical" id="batteryCapacityForm">
                                <label for="batteryCapacityInput">Ёмкость аккум. (1000-3500 мАч):</label>
                                <div class="input-group">
                                    <input type="number" id="batteryCapacityInput" name="battery_capacity_mah" placeholder="По умолчанию 3000 мАч" min="1000" max="3500" required>
                                    <button type="submit" class="button">Сохранить</button>
                                </div>
                            </form>
                            <!-- Удалена кнопка "Сбросить статистику температуры" -->
                            <button class="button orange" onclick="rebootEsp()">Перезагрузить контроллер</button>
                            <button class="button red" onclick="resetSettings()">Сброс настроек на заводские</button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    <div class="simple-footer">
        Разработано тем самым Вилли
    </div>

    <script>
        const INITIAL_MACRO_COUNT = 6;
        const MAX_MACROS = 15;

        // Обновляет текстовое значение слайдера
        function updateSliderValue(id, value) {
            document.getElementById(id).innerText = value;
        }

        // Обновляет заливку трека слайдера в зависимости от значения
        function updateSliderFill(slider) {
            const value = slider.value;
            const max = slider.max;
            const percent = (value / max) * 100;
            const trackColor = slider.disabled ? 'var(--secondary-text-color)' : 'var(--accent-color)';
            slider.style.background = `linear-gradient(to right, ${trackColor} 0%, ${trackColor} ${percent}%, var(--input-bg) ${percent}%, var(--input-bg) 100%)`;
        }

        // Функция debounce для задержки выполнения функции
        function debounce(func, timeout = 300) {
            let timer;
            return (...args) => {
                clearTimeout(timer);
                timer = setTimeout(() => { func.apply(this, args); }, timeout);
            };
        }

        // Отправляет живое обновление настройки на ESP (без сохранения в EEPROM)
        function updateLiveSetting(settingName, value) {
            const formData = new URLSearchParams();
            formData.append(settingName, value);
            fetch('/live_update', {
                method: 'POST',
                headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                body: formData.toString()
            }).then(response => {
                if (!response.ok) {
                    console.error('Failed to update live setting');
                }
            }).catch(error => {
                console.error('Error updating live setting:', error);
            });
        }

        // Отправляет настройку на ESP для сохранения в EEPROM
        function saveSetting(settingName, value) {
            const formData = new URLSearchParams();
            formData.append(settingName, value);
            fetch('/save_settings', {
                method: 'POST',
                headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                body: formData.toString()
            }).then(response => response.text()).then(data => {
                console.log(`Setting ${settingName} saved: ${data}`);
            }).catch(error => {
                console.error(`Error saving ${settingName}:`, error);
            });
        }

        // Дебаунс для функции saveSetting
        const debouncedSaveSetting = debounce(saveSetting, 300);

        // Устанавливает оптимальную скорость прокрутки
        function setOptimalSpeed() {
            const optimalSpeed = 75; // Оптимальная скорость в процентах
            const speedSlider = document.getElementById('speedSlider');
            if (!speedSlider.disabled) {
                speedSlider.value = optimalSpeed;
                updateSliderValue('speedSliderValue', optimalSpeed);
                updateSliderFill(speedSlider);

                updateLiveSetting('speed_percent', optimalSpeed);
                saveSetting('speed_percent', optimalSpeed);
            }
        }

        // Отправляет пользовательский текст на ESP
        function sendText(text) {
            fetch('/send_text', {
                method: 'POST',
                headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                body: 'text=' + encodeURIComponent(text)
            }).then(response => response.text()).then(data => {
                console.log(data);
                if (data === "OK") {
                    console.log('Текст отправлен!');
                } else {
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert('Ошибка при отправке текста: ' + data); 
                }
            }).catch(error => {
                console.error('Ошибка при отправке текста:', error);
                // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                alert('Не удалось отправить текст.');
            });
        }

        // Удаляет макрос по индексу
        function deleteMacro(index) {
            // Использование confirm() не рекомендуется в iframe, но оставлено по исходному коду
            if (!confirm("Вы уверены, что хотите удалить этот макрос?")) {
                return;
            }
            fetch('/delete_macro?index=' + index, { method: 'POST' })
            .then(response => response.json())
            .then(data => {
                if (data.status === "OK") {
                    console.log('Макрос удален успешно.');
                    loadInfo(); // Перезагружаем информацию для обновления списка макросов
                } else {
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert('Ошибка при удалении макроса: ' + data.message);
                }
            })
            .catch(error => {
                console.error('Ошибка при удалении макроса:', error);
                // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                alert('Не удалось удалить макрос.');
            });
        }

        // Обновляет элементы UI в зависимости от выбранного режима отображения
        function updateUIForMode(mode) {
            const isScrollingMode = (mode === '0'); // 0 - MODE_SCROLLING

            const textInputCard = document.getElementById('textInputCard');
            const macrosCard = document.getElementById('macrosCard');
            const speedSliderWrapper = document.getElementById('speedSliderWrapper');
            const displayModeStatus = document.getElementById('displayModeStatus');
            const animationEffectSelect = document.getElementById('animationEffectSelect');
            const animationEffectGroup = document.getElementById('animationEffectGroup');
            
            if (isScrollingMode) {
                textInputCard.style.display = 'flex';
                macrosCard.style.display = 'flex';
                speedSliderWrapper.style.display = 'flex';
                displayModeStatus.innerText = 'Бегущая строка';
                animationEffectGroup.style.display = 'none'; // Скрываем выбор эффектов
            } else { // MODE_EFFECTS
                textInputCard.style.display = 'none';
                macrosCard.style.display = 'none';
                speedSliderWrapper.style.display = 'none';
                const effectText = animationEffectSelect.options[animationEffectSelect.selectedIndex].text;
                displayModeStatus.innerText = `Эффекты: ${effectText}`;
                animationEffectGroup.style.display = 'flex'; // Показываем выбор эффектов
            }
        }

        // Обработчик отправки формы текста
        document.querySelector('form[action="/send_text"]').addEventListener('submit', function(event) {
            event.preventDefault(); // Предотвращаем стандартную отправку формы
            const textInput = document.getElementById('textInput');
            sendText(textInput.value);
            textInput.value = ''; // Очищаем поле ввода
        });

        // Обработчик отправки формы добавления макроса
        document.getElementById('addMacroForm').addEventListener('submit', function(event) {
            event.preventDefault();
            const form = event.target;
            const newMacroInput = form.querySelector('input[name="new_macro_text"]');
            const addButton = form.querySelector('button');
            const newMacroText = newMacroInput.value;
            addButton.disabled = true; // Отключаем кнопку, чтобы предотвратить повторную отправку

            fetch('/add_macro', {
                method: 'POST',
                headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                body: 'new_macro_text=' + encodeURIComponent(newMacroText)
            }).then(response => response.json()).then(data => {
                if (data.status === "OK") {
                    console.log('Макрос добавлен успешно.');
                    newMacroInput.value = ''; // Очищаем поле ввода
                    loadInfo(); // Перезагружаем информацию для обновления списка макросов
                } else {
                    console.error('Ошибка при добавлении макроса:', data.message);
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert('Ошибка: ' + data.message);
                }
            }).catch(error => {
                console.error('Ошибка при добавлении макроса:', error);
                // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                alert('Не удалось добавить макрос.');
            }).finally(() => {
                addButton.disabled = false; // Включаем кнопку обратно
            });
        });
        
        // Обработчик отправки формы емкости батареи
        document.getElementById('batteryCapacityForm').addEventListener('submit', function(event) {
            event.preventDefault();
            const form = event.target;
            const batteryCapacityInput = form.querySelector('input[name="battery_capacity_mah"]');
            const saveButton = form.querySelector('button[type="submit"]');

            const batteryCapacity = parseInt(batteryCapacityInput.value);
            const min = parseInt(batteryCapacityInput.min);
            const max = parseInt(batteryCapacityInput.max);

            // Валидация введенного значения
            if (isNaN(batteryCapacity) || batteryCapacity < min || batteryCapacity > max) {
                // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                alert(`Пожалуйста, введите значение в диапазоне от ${min} до ${max} мАч.`);
                batteryCapacityInput.focus();
                return;
            }
            
            saveButton.disabled = true; // Отключаем кнопку
            const formData = new URLSearchParams();
            formData.append('battery_capacity_mah', batteryCapacity);
            fetch('/save_settings', {
                method: 'POST',
                headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                body: formData.toString()
            }).then(response => response.text()).then(data => {
                if (data === "OK") {
                    console.log('Емкость аккумулятора сохранена.');
                    loadInfo(); // Перезагружаем информацию для обновления данных батареи
                } else {
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert('Ошибка при сохранении емкости.');
                }
            }).catch(error => {
                console.error('Ошибка при сохранении емкости:', error);
                // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                alert('Не удалось сохранить емкость.');
            }).finally(() => {
                saveButton.disabled = false; // Включаем кнопку обратно
            });
        });

        // Загружает всю информацию с ESP и обновляет UI
        function loadInfo() {
            fetch('/info').then(response => {
                if (!response.ok) throw new Error('Network response was not ok');
                return response.json();
            }).then(data => {
                // Обновление информации о Wi-Fi
                document.getElementById('wifiIpAddress').innerText = data.wifi.ip || '0.0.0.0';
                document.getElementById('wifiSsid').innerText = data.wifi.ssid || 'Недоступно (AP)';
                
                const connectedClients = data.wifi.connectedClients;
                if (connectedClients > 0) {
                     document.getElementById('wifiConnectedClients').innerText = `Подключено: ${connectedClients}`;
                } else {
                     document.getElementById('wifiConnectedClients').innerText = `Нет подключенных клиентов`;
                }

                document.getElementById('wifiMacAddress').innerText = data.wifi.mac || '--';
                document.getElementById('wifiChannel').innerText = data.wifi.channel || '--';
                document.getElementById('wifiMode').innerText = data.wifi.mode || '--';

                // Обновление информации о системе
                const freeHeap = data.system.freeHeap || 0;
                const initialTotalHeap = data.system.totalHeap || 0;
                let overviewText = 'Нет данных';
                if (initialTotalHeap > 0) {
                    const freeHeapKB = (freeHeap / 1024).toFixed(1);
                    const totalHeapKB = (initialTotalHeap / 1024).toFixed(1);
                    overviewText = `${freeHeapKB} KB из ${totalHeapKB} KB`;
                } else {
                    overviewText = `${(freeHeap / 1024).toFixed(1)} KB`;
                }
                document.getElementById('memOverview').innerText = overviewText;
                document.getElementById('flashTotal').innerText = (data.system.flashTotal ? (data.system.flashTotal / (1024 * 1024)).toFixed(2) + ' MB' : 'Нет данных');
                document.getElementById('sketchSize').innerText = (data.system.sketchSize ? (data.system.sketchSize / 1024).toFixed(2) + ' KB' : 'Нет данных');
                document.getElementById('chipId').innerText = data.system.chipId || 'Нет данных';
                document.getElementById('sdkVersion').innerText = data.system.sdkVersion || 'Нет данных';
                document.getElementById('cpuFreq').innerText = (data.system.cpuFreq ? data.system.cpuFreq + ' МГц' : 'Нет данных');
                document.getElementById('uptime').innerText = data.system.uptime || 'Нет данных';

                // Обновление информации о батарее
                const batteryCard = document.getElementById('batteryCard');
                const batteryVoltage = data.battery.voltage;
                const batteryPercent = data.battery.percent;
                const batteryStatus = data.battery.status;
                const lastBatteryRead = data.battery.lastRead;
                const batteryPrediction = data.battery.prediction;

                document.getElementById('batteryVoltage').innerText = formatVoltage(batteryVoltage);
                document.getElementById('batteryPercent').innerText = formatPercentage(batteryPercent);
                document.getElementById('batteryStatus').innerText = batteryStatus;
                document.getElementById('lastBatteryRead').innerText = lastBatteryRead;
                document.getElementById('batteryPrediction').innerText = batteryPrediction;


                if (batteryVoltage !== 0) {
                    const minVolts = 3.0;
                    const maxVolts = 4.2;
                    const clampedVolts = Math.max(minVolts, Math.min(maxVolts, batteryVoltage));
                    const positionPercent = (clampedVolts - minVolts) / (maxVolts - minVolts) * 100;
                    document.getElementById('batteryIndicator').style.left = `${positionPercent}%`;

                    // Изменение цвета карточки в зависимости от заряда батареи
                    const hue = (batteryPercent / 100) * 120; // От красного (0) до зеленого (120)
                    const color = `hsl(${hue}, 70%, 50%)`;
                    batteryCard.style.setProperty('--battery-color', color);
                } else {
                    document.getElementById('batteryIndicator').style.left = '50%'; // Центрируем индикатор, если нет данных
                    batteryCard.style.removeProperty('--battery-color'); // Удаляем пользовательский цвет
                }

                // Обновление статус-бара батареи
                document.getElementById('batteryPercentStatus').innerText = (batteryPercent > 0) ? `${batteryPercent}%` : '--';
                const batteryLevel = document.getElementById('batteryLevel');
                const batteryIcon = document.getElementById('batteryIcon');
                if (batteryPercent > 0) {
                    batteryLevel.style.width = `${batteryPercent}%`;
                    batteryIcon.classList.remove('low', 'medium');
                    if (batteryPercent <= 20) {
                        batteryIcon.classList.add('low'); // Красный для низкого заряда
                    } else if (batteryPercent <= 50) {
                        batteryIcon.classList.add('medium'); // Оранжевый для среднего заряда
                    }
                } else {
                    batteryLevel.style.width = '0%';
                    batteryIcon.classList.remove('low', 'medium');
                }

                // Обновление настроек слайдеров и селектов
                const brightnessSlider = document.getElementById('brightnessSlider');
                if (data.settings.brightness_percent !== undefined) {
                    brightnessSlider.value = data.settings.brightness_percent;
                    updateSliderValue('brightnessSliderValue', data.settings.brightness_percent);
                    updateSliderFill(brightnessSlider);
                }

                const speedSlider = document.getElementById('speedSlider');
                if (data.settings.speed_percent !== undefined) {
                    speedSlider.value = data.settings.speed_percent;
                    updateSliderValue('speedSliderValue', data.settings.speed_percent);
                    updateSliderFill(speedSlider);
                }
                
                const batteryCapacityInput = document.getElementById('batteryCapacityInput');
                if(data.settings.battery_capacity_mah !== undefined && data.settings.battery_capacity_mah > 0) {
                     batteryCapacityInput.value = data.settings.battery_capacity_mah;
                }

                const displayModeSelect = document.getElementById('displayModeSelect');
                if (data.settings.displayMode !== undefined) {
                    displayModeSelect.value = data.settings.displayMode;
                    updateUIForMode(data.settings.displayMode.toString()); // Обновляем UI в зависимости от режима
                }

                const animationEffectSelect = document.getElementById('animationEffectSelect');
                if (data.settings.animationEffect !== undefined) {
                    animationEffectSelect.value = data.settings.animationEffect;
                    // Если текущий режим - эффекты, обновляем UI для отображения правильного текста эффекта
                    if (data.settings.displayMode.toString() === '2') {
                         updateUIForMode('2');
                    }
                }

                // Динамическое создание кнопок макросов
                let macroButtonsDiv = document.getElementById('macroButtons');
                macroButtonsDiv.innerHTML = ''; // Очищаем текущие кнопки
                if (data.macros && data.macros.length > 0) {
                    data.macros.forEach((macro, index) => {
                        const macroItem = document.createElement('div');
                        macroItem.className = 'macro-item';
                        const macroButton = document.createElement('button');
                        macroButton.type = 'button';
                        macroButton.className = 'macro-button';
                        macroButton.onclick = () => sendText(macro); // При клике отправляем текст макроса

                        macroButton.textContent = macro;
                        macroButton.title = macro; // Подсказка при наведении

                        macroItem.appendChild(macroButton);
                        // Добавляем кнопку удаления только для пользовательских макросов
                        if (index >= INITIAL_MACRO_COUNT) {
                            const deleteButton = document.createElement('button');
                            deleteButton.type = 'button';
                            deleteButton.className = 'delete-button';
                            deleteButton.onclick = () => deleteMacro(index);
                            deleteButton.textContent = '✖';
                            macroItem.appendChild(deleteButton);
                        }
                        macroButtonsDiv.appendChild(macroItem);
                    });
                }
            }).catch(error => {
                console.error('Ошибка при загрузке информации:', error);
            });
        }

        // Переключает вкладки
        function showTab(tabName) {
            const tabs = document.querySelectorAll('.tab-content');
            const buttons = document.querySelectorAll('.tab-button');
            
            tabs.forEach(tab => {
                tab.classList.remove('active');
            });
            
            buttons.forEach(button => {
                button.classList.remove('active');
            });
            
            document.getElementById(tabName + 'Tab').classList.add('active');
            document.querySelector(`.tab-button[onclick="showTab('${tabName}')"]`).classList.add('active');
        }

        // Форматирует температуру для отображения (функция оставлена, но не используется)
        function formatTemperature(temp) {
            if (temp === -1000.0) {
                return 'Датчик не подкл.';
            }
            return temp.toFixed(1) + ' °C';
        }

        // Форматирует напряжение для отображения
        function formatVoltage(voltage) {
            if (voltage === 0.0) {
                return 'Недоступно';
            }
            return voltage.toFixed(2) + ' V';
        }

        // Форматирует процент для отображения
        function formatPercentage(percent) {
            if (percent === 0) {
                 return 'Недоступно';
            }
            return percent + '%';
        }

        // Предотвращает фокусировку слайдеров при клике (для мобильных устройств)
        document.addEventListener('DOMContentLoaded', (event) => {
            const sliders = document.querySelectorAll('input[type="range"]');
            sliders.forEach(slider => {
                slider.addEventListener('focus', (e) => {
                    e.preventDefault();
                });
            });
        });

        // Отправляет команду перезагрузки ESP
        function rebootEsp() {
            // Использование confirm() не рекомендуется в iframe, но оставлено по исходному коду
            if (confirm("Вы уверены, что хотите перезагрузить ESP?")) {
                fetch('/reboot', { method: 'POST' }).then(response => {
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert("ESP перезагружается...");
                }).catch(error => {
                    console.error('Error rebooting:', error);
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert("Не удалось отправить команду перезагрузки.");
                });
            }
        }

        // Отправляет команду сброса настроек ESP
        function resetSettings() {
            // Использование confirm() не рекомендуется в iframe, но оставлено по исходному коду
            if (confirm("Вы уверены, что хотите сбросить все настройки к заводским? Это удалит все пользовательские макросы.")) {
                fetch('/reset_settings', { method: 'POST' }).then(response => {
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert("Настройки сброшены, устройство перезагружается.");
                    setTimeout(() => { window.location.reload(); }, 5000); // Перезагружаем страницу после сброса
                }).catch(error => {
                    console.error('Error resetting settings:', error);
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert("Не удалось сбросить настройки.");
                });
            }
        }

        // Отправляет команду сброса статистики температуры (функция удалена из UI)
        function resetTemperatureStats() {
             // Использование confirm() не рекомендуется в iframe, но оставлено по исходному коду
             if (confirm("Вы уверены, что хотите сбросить статистику температуры (минимум/максимум/среднюю)?")) {
                fetch('/reset_temp_stats', { method: 'POST' }).then(response => {
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert("Статистика температуры сброшена.");
                    loadInfo(); // Обновляем информацию на странице
                }).catch(error => {
                    console.error('Error resetting temperature stats:', error);
                    // Использование alert() не рекомендуется в iframe, но оставлено по исходному коду
                    alert("Не удалось сбросить статистику.");
                });
             }
        }

        // Загружаем информацию при загрузке страницы и затем каждые 5 секунд
        window.onload = loadInfo;
        setInterval(loadInfo, 5000);
    </script>
</body>
</html>
)rawliteral";

// Функция для преобразования UTF-8 в кодировку, подходящую для матрицы (вероятно, CP1251)
// Эта функция необходима, так как Max72xxPanel не поддерживает UTF-8 напрямую.
void utf8rus(const char* source, char* target, size_t target_size) {
    size_t i, target_idx = 0;
    memset(target, 0, target_size); // Очищаем целевой буфер

    for (i = 0; source[i] != '\0' && target_idx < target_size - 1; ++i) {
        unsigned char n = source[i];
        if (n >= 0xC0) { // Если это начало многобайтового символа UTF-8
            switch (n) {
                case 0xD0: { // Символы из диапазона U+0400 - U+047F (часть кириллицы)
                    if (source[i+1] == '\0') break; // Проверка на конец строки
                    n = source[++i]; // Берем следующий байт
                    if (n == 0x81) { n = 0xA8; } // 'Ё'
                    else if (n >= 0x90 && n <= 0xBF) { n = n + 0x2F; } // А-П
                    break;
                }
                case 0xD1: { // Символы из диапазона U+0480 - U+04FF (остальная часть кириллицы)
                    if (source[i+1] == '\0') break; // Проверка на конец строки
                    n = source[++i]; // Берем следующий байт
                    if (n == 0x91) { n = 0xB7; } // 'ё'
                    else if (n >= 0x80 && n <= 0x8F) { n = n + 0x6F; } // Р-Я
                    break;
                }
            }
        }
        target[target_idx++] = n; // Записываем преобразованный байт
    }
    target[target_idx] = '\0'; // Завершаем строку нулевым символом
}

// Форматирует время работы устройства в читаемый вид
String formatUptime(unsigned long milliseconds) {
    unsigned long seconds = milliseconds / 1000;
    unsigned long minutes = seconds / 60;
    unsigned long hours = minutes / 60;
    unsigned long days = hours / 24;
    String uptime = "";
    if (days > 0) { uptime += String(days) + "д "; }
    if (hours % 24 > 0 || days > 0) { uptime += String(hours % 24) + "ч "; }
    if (minutes % 60 > 0 || hours > 0) { uptime += String(minutes % 60) + "м "; }
    uptime += String(seconds % 60) + "с";
    return uptime;
}

// Сохраняет текущие настройки в EEPROM
void saveSettingsToEEPROM() {
    EEPROM.begin(EEPROM_SIZE);
    EEPROM.put(0, settings); // Записываем структуру настроек
    EEPROM.commit(); // Подтверждаем запись
    EEPROM.end(); // Закрываем EEPROM
    Serial.println("Settings and macros saved to EEPROM.");
}

// Загружает настройки из EEPROM
void loadSettingsFromEEPROM() {
    EEPROM.begin(EEPROM_SIZE);
    EEPROM.get(0, settings); // Считываем структуру настроек
    EEPROM.end();

    // Проверка и установка значений по умолчанию, если они некорректны
    if (settings.brightness_percent < 0 || settings.brightness_percent > 100) settings.brightness_percent = 100;
    if (settings.speed_percent < 0 || settings.speed_percent > 100) settings.speed_percent = 70;
    if (settings.displayMode < 0 || settings.displayMode > 2) settings.displayMode = MODE_SCROLLING;
    if (settings.animationEffect < 0 || settings.animationEffect > 2) settings.animationEffect = EFFECT_FLASHLIGHT;
    settings.speed = 120 - (settings.speed_percent * 0.9); // Расчет скорости из процента
    if (settings.macroCount < 0 || settings.macroCount > MAX_MACROS) settings.macroCount = 0;
    
    if (settings.battery_capacity_mah < 1000 || settings.battery_capacity_mah > 3500) {
        settings.battery_capacity_mah = 3000; // Емкость по умолчанию
    }

    // Восстановление начальных макросов, если их меньше чем INITIAL_MACRO_COUNT
    if (settings.macroCount < INITIAL_MACRO_COUNT) {
        const char* initialMacros[] = {
            "Держите дистанцию)",
            "Не сбивайте меня пожалуйста)",
            "Качусь колбаской)",
            "Меня тоже дома ждут)",
            "Не бибикай ну",
            "SEND NUDES"
        };
        for (int i = 0; i < INITIAL_MACRO_COUNT; ++i) {
            strncpy(settings.macros[i], initialMacros[i], MAX_MACRO_LENGTH);
        }
        settings.macroCount = INITIAL_MACRO_COUNT;
        saveSettingsToEEPROM(); // Сохраняем восстановленные макросы
        Serial.println("Initial macros restored.");
    }

    // Установка текущего текста для прокрутки из первого макроса
    if (settings.macroCount > 0) { strncpy(currentTapeText, settings.macros[0], MAX_MACRO_LENGTH); }
    liveBrightness = (byte)((settings.brightness_percent * 15) / 100.0); // Расчет яркости для матрицы (0-15)
    liveSpeed = settings.speed; // Установка живой скорости
}

// Удалена функция readThermistor()

// Чтение напряжения батареи
void readBatteryVoltage() {
    digitalWrite(SENSOR_POWER_PIN, HIGH); // Включаем питание датчика
    delay(10); // Небольшая задержка для стабилизации

    float analogValue = analogRead(A0); // Читаем аналоговое значение

    digitalWrite(SENSOR_POWER_PIN, LOW); // Выключаем питание датчика

    // Расчет напряжения батареи с учетом делителя и калибровки
    currentBatteryVoltage = analogValue * (ADC_VOLTAGE / 1024.0) * (R1_BATTERY + R2_BATTERY) / R2_BATTERY * VOLTAGE_CALIBRATION_FACTOR;

    // Расчет процента заряда
    if (currentBatteryVoltage > BATTERY_MAX_V) {
        batteryPercentage = 100;
    } else if (currentBatteryVoltage < BATTERY_MIN_V) {
        batteryPercentage = 1; // Минимальный процент, чтобы показать, что батарея разряжена
    } else {
        batteryPercentage = ((currentBatteryVoltage - BATTERY_MIN_V) / (BATTERY_MAX_V - BATTERY_MIN_V)) * 100;
    }
}

// Прогнозирует оставшееся время работы батареи
String predictBatteryLife() {
    float averageConsumption_mA = 0;
    
    // Оценка среднего потребления в зависимости от режима и яркости
    if (settings.displayMode == MODE_SCROLLING) {
        averageConsumption_mA = 80 + (settings.brightness_percent / 100.0) * 40; // 80-120mA
    } else if (settings.displayMode == MODE_EFFECTS) {
        if (settings.animationEffect == EFFECT_FLASHLIGHT) {
            averageConsumption_mA = 60 + (settings.brightness_percent / 100.0) * 30; // 60-90mA
        } else if (settings.animationEffect == EFFECT_POLICE_SIREN) {
            averageConsumption_mA = 90 + (settings.brightness_percent / 100.0) * 50; // 90-140mA
        }
    } else { // MODE_STATIC или неизвестный режим
        averageConsumption_mA = 25; // Базовое потребление
    }

    // Проверка на достаточность данных для прогноза
    if (settings.battery_capacity_mah <= 0 || currentBatteryVoltage <= 0 || averageConsumption_mA <= 0) {
        return "Недостаточно данных";
    }

    float remainingCapacity_mah = (settings.battery_capacity_mah * batteryPercentage) / 100.0;
    
    // Прогноз времени работы. Коэффициент 0.8 может быть для учета нелинейности разряда или запаса.
    float batteryLife_hours = (remainingCapacity_mah / averageConsumption_mA) * 0.8;

    if (batteryLife_hours < 1.0) {
        float batteryLife_minutes = batteryLife_hours * 60;
        if (batteryLife_minutes < 1) {
             return "< 1 мин (примерно)";
        }
        return String((int)batteryLife_minutes) + " мин (примерно)";
    } else {
        return String((int)batteryLife_hours) + " ч (примерно)";
    }
}

// Настройка светодиодной матрицы
void setupDisplay() {
    matrix.setIntensity(liveBrightness); // Установка яркости
    matrix.setFont(NULL); // Используем стандартный шрифт
    matrix.setTextSize(1); // Размер текста
    matrix.setRotation(0); // Без поворота
    matrix.fillScreen(LOW); // Очистка экрана
}

// Удалена функция resetTemperatureStats()

// Обработчик корневого запроса (отправка HTML-страницы)
void handleRoot() {
    server.send_P(200, "text/html", page);
}

// Обработчик запроса информации (отправка JSON-данных)
void handleGetInfo() {
    DynamicJsonDocument doc(1024); // Создаем JSON-документ

    // Информация о Wi-Fi
    JsonObject wifi = doc.createNestedObject("wifi");
    wifi["ip"] = WiFi.softAPIP().toString();
    wifi["ssid"] = WiFi.softAPSSID();
    wifi["connectedClients"] = WiFi.softAPgetStationNum();
    wifi["mac"] = WiFi.softAPmacAddress();
    wifi["channel"] = WiFi.channel();
    
    String wifiClientsText = "Нет подключенных клиентов";
    int numClients = WiFi.softAPgetStationNum();
    if (numClients > 0) {
        if (numClients == 1) {
            wifiClientsText = "Подключен 1";
        } else {
            wifiClientsText = "Подключено " + String(numClients);
        }
    }
    wifi["connectedClientsText"] = wifiClientsText;
    
    wifi["mode"] = "AP";

    // Информация о системе
    JsonObject system = doc.createNestedObject("system");
    system["freeHeap"] = ESP.getFreeHeap();
    system["totalHeap"] = initialTotalHeap;
    system["flashTotal"] = ESP.getFlashChipSize();
    system["sketchSize"] = ESP.getSketchSize();
    system["chipId"] = ESP.getChipId();
    system["sdkVersion"] = ESP.getSdkVersion();
    system["cpuFreq"] = ESP.getCpuFreqMHz();
    system["uptime"] = formatUptime(millis());

    // Удалена информация о температуре из JSON

    // Информация о батарее
    JsonObject battery = doc.createNestedObject("battery");
    battery["voltage"] = currentBatteryVoltage;
    battery["percent"] = (int)batteryPercentage;
    battery["status"] = (currentBatteryVoltage > BATTERY_MAX_V) ? "Полный заряд" : (currentBatteryVoltage < BATTERY_MIN_V) ? "Критический" : "Норма";
    battery["lastRead"] = String((millis() - lastBatteryReadTime) / 1000) + "с назад";
    battery["prediction"] = predictBatteryLife();

    // Текущие настройки
    JsonObject settingsObj = doc.createNestedObject("settings");
    settingsObj["brightness_percent"] = settings.brightness_percent;
    settingsObj["speed_percent"] = settings.speed_percent;
    settingsObj["displayMode"] = settings.displayMode;
    settingsObj["animationEffect"] = settings.animationEffect;
    settingsObj["battery_capacity_mah"] = settings.battery_capacity_mah;

    // Макросы
    JsonArray macros = doc.createNestedArray("macros");
    for (int i = 0; i < settings.macroCount; ++i) {
        macros.add(settings.macros[i]);
    }
    
    String jsonString;
    serializeJson(doc, jsonString); // Сериализуем JSON-документ в строку
    server.send(200, "application/json", jsonString); // Отправляем JSON-ответ
}

// Обработчик отправки текста
void handleSendText() {
    if (server.hasArg("text")) {
        String newText = server.arg("text");
        newText.trim(); // Удаляем пробелы по краям
        if (newText.length() > 0) {
            char temp_buffer[MAX_MACRO_LENGTH + 1];
            utf8rus(newText.c_str(), temp_buffer, sizeof(temp_buffer)); // Преобразуем UTF-8
            strncpy(currentTapeText, temp_buffer, sizeof(currentTapeText)); // Копируем текст
            currentPos = matrix.width(); // Сбрасываем позицию прокрутки
            displayText = ""; // Очищаем для перерасчета ширины
            matrix.fillScreen(LOW); // Очищаем экран матрицы
            matrix.setCursor(0, 0);
            matrix.print(currentTapeText);
            textWidth = (matrix.width() * strlen(currentTapeText)) + tickerSpacer; // Пересчитываем ширину текста
            server.send(200, "text/plain", "OK");
        } else {
            server.send(400, "text/plain", "Text is empty");
        }
    } else {
        server.send(400, "text/plain", "Missing text argument");
    }
}

// Обработчик добавления макроса
void handleAddMacro() {
    if (server.hasArg("new_macro_text")) {
        String newMacro = server.arg("new_macro_text");
        newMacro.trim();
        if (newMacro.length() > MAX_MACRO_LENGTH) {
            server.send(400, "application/json", "{\"status\": \"error\", \"message\": \"Macro is too long\"}");
            return;
        }
        if (settings.macroCount >= MAX_MACROS) {
            server.send(400, "application/json", "{\"status\": \"error\", \"message\": \"Max macros reached\"}");
            return;
        }

        char temp_buffer[MAX_MACRO_LENGTH + 1];
        strncpy(temp_buffer, newMacro.c_str(), sizeof(temp_buffer));
        temp_buffer[sizeof(temp_buffer) - 1] = '\0'; // Убедимся, что строка завершается нулем
        strncpy(settings.macros[settings.macroCount], temp_buffer, MAX_MACRO_LENGTH);
        settings.macroCount++;
        saveSettingsToEEPROM(); // Сохраняем изменения
        server.send(200, "application/json", "{\"status\": \"OK\"}");
    } else {
        server.send(400, "application/json", "{\"status\": \"error\", \"message\": \"Missing macro text\"}");
    }
}

// Обработчик удаления макроса
void handleDeleteMacro() {
    if (server.hasArg("index")) {
        int index = server.arg("index").toInt();
        // Проверяем, что индекс корректен и это пользовательский макрос (не из INITIAL_MACRO_COUNT)
        if (index >= 0 && index < settings.macroCount && index >= INITIAL_MACRO_COUNT) {
            // Сдвигаем все последующие макросы влево
            for (int i = index; i < settings.macroCount - 1; ++i) {
                strncpy(settings.macros[i], settings.macros[i+1], MAX_MACRO_LENGTH);
            }
            settings.macroCount--; // Уменьшаем счетчик макросов
            strncpy(settings.macros[settings.macroCount], "", MAX_MACRO_LENGTH); // Очищаем последнюю позицию
            saveSettingsToEEPROM(); // Сохраняем изменения
            server.send(200, "application/json", "{\"status\": \"OK\"}");
        } else {
            server.send(400, "application/json", "{\"status\": \"error\", \"message\": \"Invalid index\"}");
        }
    } else {
        server.send(400, "application/json", "{\"status\": \"error\", \"message\": \"Missing index\"}");
    }
}

// Обработчик сохранения настроек (постоянное сохранение в EEPROM)
void handleSaveSettings() {
    if (server.hasArg("brightness_percent")) {
        settings.brightness_percent = server.arg("brightness_percent").toInt();
        liveBrightness = (byte)((settings.brightness_percent * 15) / 100.0);
        matrix.setIntensity(liveBrightness);
        brightnessChanged = true; // Флаг для обновления яркости в loop
    }
    if (server.hasArg("speed_percent")) {
        settings.speed_percent = server.arg("speed_percent").toInt();
        settings.speed = 120 - (settings.speed_percent * 0.9); // Расчет скорости
        liveSpeed = settings.speed;
    }
    if (server.hasArg("displayMode")) {
        settings.displayMode = server.arg("displayMode").toInt();
        currentPos = matrix.width(); // Сбрасываем позицию прокрутки при смене режима
    }
    if (server.hasArg("animationEffect")) {
        settings.animationEffect = server.arg("animationEffect").toInt();
    }
    if (server.hasArg("battery_capacity_mah")) {
        int newCapacity = server.arg("battery_capacity_mah").toInt();
        if (newCapacity >= 1000 && newCapacity <= 3500) {
            settings.battery_capacity_mah = newCapacity;
        } else {
            server.send(400, "text/plain", "Invalid battery capacity. Must be between 1000 and 3500 mAh.");
            return;
        }
    }

    saveSettingsToEEPROM(); // Сохраняем все настройки
    server.send(200, "text/plain", "OK");
}

// Обработчик живого обновления настроек (без сохранения в EEPROM)
void handleLiveUpdate() {
    if (server.hasArg("brightness_percent")) {
        int brightness = server.arg("brightness_percent").toInt();
        liveBrightness = (byte)((brightness * 15) / 100.0);
        matrix.setIntensity(liveBrightness);
    }
    if (server.hasArg("speed_percent")) {
        int speed_percent = server.arg("speed_percent").toInt();
        liveSpeed = 120 - (speed_percent * 0.9);
    }
    server.send(200, "text/plain", "OK");
}

// Обработчик перезагрузки ESP
void handleReboot() {
    server.send(200, "text/plain", "Rebooting...");
    delay(200); // Небольшая задержка перед перезагрузкой
    ESP.restart();
}

// Удален обработчик сброса статистики температуры

// Обработчик сброса всех настроек к заводским (стирание EEPROM)
void handleResetSettings() {
    EEPROM.begin(EEPROM_SIZE);
    for(int i = 0; i < EEPROM_SIZE; ++i) {
        EEPROM.write(i, 0xFF); // Заполняем EEPROM FF (пустые байты)
    }
    EEPROM.commit();
    EEPROM.end();
    server.send(200, "text/plain", "Settings reset. Rebooting...");
    delay(200);
    ESP.restart();
}

// Обработчик для Captive Portal (перенаправление на IP-адрес ESP)
void handleCaptivePortal() {
    server.sendHeader("Location", "http://192.168.4.1"); // IP-адрес ESP8266 в режиме AP
    server.send(302); // Код 302 Found (временное перенаправление)
}

// Функция setup - выполняется один раз при старте
void setup() {
    Serial.begin(115200); // Инициализация последовательного порта
    pinMode(SENSOR_POWER_PIN, OUTPUT); // Настройка пина питания датчиков как выход
    digitalWrite(SENSOR_POWER_PIN, LOW); // Выключаем питание датчиков по умолчанию
    
    loadSettingsFromEEPROM(); // Загружаем настройки из EEPROM
    initialTotalHeap = ESP.getFreeHeap(); // Запоминаем начальный объем свободной памяти

    setupDisplay(); // Настраиваем светодиодную матрицу
    matrix.setIntensity(liveBrightness); // Устанавливаем яркость матрицы

    WiFi.mode(WIFI_AP); // Устанавливаем режим работы Wi-Fi как точка доступа
    WiFi.softAP(ssid, pwd); // Запускаем точку доступа
    dnsServer.start(53, "*", WiFi.softAPIP()); // Запускаем DNS-сервер для Captive Portal

    // Регистрация обработчиков HTTP-запросов
    server.on("/", handleRoot);
    server.on("/info", handleGetInfo);
    server.on("/send_text", HTTP_POST, handleSendText);
    server.on("/add_macro", HTTP_POST, handleAddMacro);
    server.on("/delete_macro", HTTP_POST, handleDeleteMacro);
    server.on("/save_settings", HTTP_POST, handleSaveSettings);
    server.on("/live_update", HTTP_POST, handleLiveUpdate);
    server.on("/reboot", HTTP_POST, handleReboot);
    // server.on("/reset_temp_stats", HTTP_POST, handleResetTempStats); // Удален обработчик
    server.on("/reset_settings", HTTP_POST, handleResetSettings);
    
    // Обработчик для /generate_204 (используется некоторыми ОС для проверки подключения)
    server.on("/generate_204", HTTP_GET, []() {
      server.send(204);
    });

    server.onNotFound(handleCaptivePortal); // Обработчик для всех неопознанных запросов
    
    server.begin(); // Запускаем веб-сервер
    Serial.println("HTTP server started");
    Serial.println("IP address: " + WiFi.softAPIP().toString());
    
    // Инициализация отображения на матрице
    matrix.fillScreen(LOW);
    matrix.setCursor(0, 0);
    matrix.print(currentTapeText);
    textWidth = matrix.width() + matrix.width(); // Начальная ширина текста для прокрутки
    currentPos = matrix.width(); // Начальная позиция прокрутки
}

// Функция loop - выполняется постоянно
void loop() {
    dnsServer.processNextRequest(); // Обработка DNS-запросов
    server.handleClient(); // Обработка клиентских HTTP-запросов

    // Периодическое чтение данных с датчиков
    if (millis() - lastBatteryReadTime >= 5000) { // Каждые 5 секунд
        readBatteryVoltage();
        lastBatteryReadTime = millis();
    }
    
    // Логика отображения в зависимости от выбранного режима
    if (settings.displayMode == MODE_SCROLLING) {
        matrix.fillScreen(LOW); // Очищаем экран
        if (brightnessChanged) { // Если яркость была изменена через веб-интерфейс
            matrix.setIntensity(liveBrightness);
            brightnessChanged = false;
        }
        
        matrix.setCursor(currentPos, 0); // Устанавливаем курсор для прокрутки
        matrix.print(currentTapeText); // Выводим текст
        
        currentPos = currentPos - 1; // Сдвигаем текст влево
        if (currentPos < -textWidth) { // Если текст ушел за пределы экрана
            currentPos = matrix.width(); // Возвращаем текст в начало
        }
        matrix.write(); // Обновляем матрицу
    } else if (settings.displayMode == MODE_EFFECTS) {
        matrix.fillScreen(LOW); // Очищаем экран
        matrix.setIntensity(liveBrightness); // Устанавливаем яркость

        if (settings.animationEffect == EFFECT_FLASHLIGHT) {
            if (millis() - lastFlashlightFrameTime >= flashlightDelay) {
                lastFlashlightFrameTime = millis();
                matrix.fillScreen(LOW); // Очищаем экран для нового кадра
                int yOffset = 0; // Не используется, можно удалить
                int radius = flashlightFrameIndex;
                
                // Логика увеличения/уменьшения радиуса круга
                if (radius >= matrix.height() / 2) {
                    flashlightGrowing = false;
                }
                if (radius <= 0) {
                    flashlightGrowing = true;
                }
                if (flashlightGrowing) {
                    flashlightFrameIndex++;
                } else {
                    flashlightFrameIndex--;
                }
                
                int centerX = matrix.width() / 2;
                int centerY = matrix.height() / 2;
                // Рисуем круги для эффекта фонарика
                matrix.drawCircle(centerX, centerY, radius, HIGH);
                matrix.drawCircle(centerX, centerY, radius - 1, HIGH);
                matrix.drawCircle(centerX, centerY, radius - 2, HIGH);

                if (radius == matrix.height() / 2) { // Сброс индекса, когда круг достигает максимального размера
                     flashlightFrameIndex = 0;
                }
                matrix.write(); // Обновляем матрицу
            }
        } else if (settings.animationEffect == EFFECT_POLICE_SIREN) {
            if (millis() - lastPoliceFlashTime >= policeFlashDelay) {
                lastPoliceFlashTime = millis();

                matrix.fillScreen(LOW); // Очищаем экран для нового кадра
                if (policeCurrentBlock == 0) { // Левая сторона
                    if (policeFlashState) {
                        matrix.drawRect(0, 0, matrix.width()/2, matrix.height(), HIGH); // Рисуем левый прямоугольник
                    }
                } else { // Правая сторона
                    if (policeFlashState) {
                        matrix.drawRect(matrix.width()/2, 0, matrix.width()/2, matrix.height(), HIGH); // Рисуем правый прямоугольник
                    }
                }

                policeFlashState = !policeFlashState; // Переключаем состояние вспышки
                if (!policeFlashState) { // Если вспышка только что выключилась
                    policeFlashesInBlock++;
                    if (policeFlashesInBlock >= policeFlashesPerBlock) { // Если достигнуто количество вспышек в блоке
                        policeFlashesInBlock = 0;
                        policeCurrentBlock = 1 - policeCurrentBlock; // Переключаем блок (левый/правый)
                    }
                }
                matrix.write(); // Обновляем матрицу
            }
        }
    }

    // Задержка для управления скоростью прокрутки (только для режима бегущей строки)
    if (settings.displayMode == MODE_SCROLLING) {
        delay(liveSpeed); // Блокирующая задержка
    }    
}
