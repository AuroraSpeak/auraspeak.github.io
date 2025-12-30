# Aura Speak Blog

Ein persönlicher Blog, gebaut mit Hugo und dem Coder Theme.

## Setup

1. Hugo installieren:
   ```bash
   # macOS
   brew install hugo
   
   # Linux
   sudo apt-get install hugo
   ```

2. Repository klonen:
   ```bash
   git clone https://github.com/AuroraSpeak/auraspeak.github.io.git
   cd auraspeak.github.io
   ```

3. Theme initialisieren (falls nötig):
   ```bash
   git submodule update --init --recursive
   ```

## Entwicklung

Lokalen Entwicklungsserver starten:
```bash
hugo server -D
```

Die Website ist dann unter `http://localhost:1313` erreichbar.

## Build

Statische Website bauen:
```bash
hugo
```

Die generierten Dateien befinden sich im `public/` Verzeichnis.

## Projektstruktur

- `content/` - Blog-Posts und Seiten
- `layouts/` - Custom Layouts und Partials
- `assets/` - CSS und andere Assets
- `themes/coder/` - Hugo Coder Theme (Submodule)
- `public/` - Generierte statische Dateien (nicht im Git)

## Schriftarten

- **Inter** - Für alle Text-Inhalte
- **JetBrains Mono Nerd Font** - Für Code-Blöcke

Die Schriftarten werden automatisch von Google Fonts bzw. jsDelivr CDN geladen.
