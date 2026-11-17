# Builds — four office computers

## 1. Accountant
- **Load:** Excel, 1C, browser with 5–10 tabs, occasional video call.
- **Bottleneck:** [что тормозит первым, например, SSD при большом объёме файлов]
- **Trade-off:** [что сэкономил и почему, например, 6 ядер вместо 8]
- **Total:** 506061 KZT (budget.csv, rows 2–7)

## 2. Sales manager
- **Load:** CRM in browser, 30–40 tabs, video calls several hours a day.
- **Bottleneck:** memory, not CPU. Chrome with many tabs plus a call uses 11–12 GB.
- **Trade-off:** [например, Ryzen 5 5600 вместо Ryzen 7, а сэкономленное пошло на 32 ГБ]
- **Total:** 688298 KZT (budget.csv, rows 8–13)

## 3. Web developer
- **Load:** IDE, Docker, local servers, builds and tests.
- **Bottleneck:** CPU during builds, memory with Docker.
- **Trade-off:** [например, монитор не самый дорогой, потому что для кода цветопередача не важна]
- **Total:** 826538 KZT (budget.csv, rows 14–19)

## 4. Designer
- **Load:** Photoshop, Figma, large files, colour-critical work.
- **Bottleneck:** monitor colour accuracy and memory for large files.
- **Trade-off:** [например, процессор Ryzen 7 вместо более мощного, потому что графика важнее CPU]
- **Total:** 544817 KZT (budget.csv, rows 20–25)
