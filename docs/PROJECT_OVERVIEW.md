# 00 — Project Overview

## Что это

**Logic Linux** — порт на Linux/Mono программного обеспечения киоска самообслуживания (платёжный терминал) на базе Windows-приложения. Система принимает наличные (CashCode/CCNet), печатает чеки (Citizen PPU 700), взаимодействует с сервером мониторинга и платежным сервером, ведёт учёт платежей в MySQL.

## История

| Этап | Статус |
|------|--------|
| Оригинальная Windows-сборка (Terminal.zip) | исходная база |
| Портирование на Linux/Mono | выполнено |
| MySQL миграция (SQL Server → MySQL) | 10/10 тестов PASS |
| Device layer fix (GetDeviceProperties) | 13/13 тестов PASS |
| RSA keys fix (валидные 2048-бит тест-ключи) | PASS |
| Mock monitoring/payment + E2E | 13/13 PASS |
| Production readiness audit | выполнен (deploy/audit документы) |

## Текущее состояние

- Рабочая директория: `/root/Logic/MonoDebugTemp/`
- Baseline проверен: DB 10/10, DEVICE 13/13, CASH-IN 16/16, PRINTER 13/13, E2E 13/13, STARTUP PASS, NETWORK PASS, PAYMENT PASS, MONITOR PASS, RSA PASS (test keys)
- Production deployment подготовлен: `install_logic_linux.sh`, systemd unit, deploy README
- Производственные ключи/адреса — внешний blocker

## Ключевые термины

| Термин | Значение |
|--------|----------|
| PointId | идентификатор терминала в системе |
| CashCodeName | логическое имя купюроприёмника (`Cup`) |
| PrinterName | логическое имя принтера (`printer`) |
| CCNet | протокол CashCode bill acceptor |
| Citizen PPU 700 | модель принтера |
| Emonitor | канал связи с мониторинговым сервером |
| SinkLib/Formatter | HTTP-клиент платежного протокола |
| MonitorURL/MonitorPort | адрес мониторинга (:333) |
| PaymentUri | адрес платежного сервера (:14111) |
