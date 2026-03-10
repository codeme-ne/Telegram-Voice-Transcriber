# Projektdokumentation – Telegram Voice Transcriber

> **Bewerbungsunterlage** | Rolle: Automation & KI-Experte | Zielunternehmen: Veritas

---

## 1. Projektname & Repository

| Feld | Inhalt |
|------|--------|
| **Projektname** | Telegram Voice Transcriber |
| **Repository** | [codeme-ne/Telegram-Voice-Transcriber](https://github.com/codeme-ne/Telegram-Voice-Transcriber) |
| **Typ** | Open-Source-Eigenentwicklung |
| **Lizenz** | MIT |
| **Status** | Aktiv / Produktionsreif |

---

## 2. Kurzbeschreibung

**Telegram Voice Transcriber** ist ein lokal laufendes Python-Tool, das Sprachnachrichten aus Telegram-Chats automatisch exportiert, offline transkribiert und als strukturiertes Markdown-Dokument ausgibt – vollständig datenschutzkonform, ohne Cloud-Dienste und ohne Telegram Premium.

Das Projekt löst ein konkretes Produktivitätsproblem: Gespräche, die als Sprachnachrichten stattfinden, sind schwer durchsuchbar, archivierbar und auswertbar. Dieses Tool schließt diese Lücke durch vollautomatische, lokale KI-Verarbeitung.

---

## 3. Verwendeter Stack

### Kernabhängigkeiten

| Kategorie | Technologie | Zweck |
|-----------|-------------|-------|
| **Sprache** | Python 3.10+ | Gesamte Implementierung |
| **KI / Speech-to-Text** | [faster-whisper](https://github.com/SYSTRAN/faster-whisper) | Lokale Transkription via OpenAI Whisper (quantisiert) |
| **Telegram API** | [Telethon](https://github.com/LonamiWebs/Telethon) | MTProto-Client für Nachrichtenexport |
| **Web UI** | [Streamlit](https://streamlit.io/) | Browserbasiertes Interface ohne JavaScript |
| **CLI** | [Typer](https://typer.tiangolo.com/) + [Rich](https://github.com/Textualize/rich) | Automatisierungsfreundliche Kommandozeile |
| **Async** | asyncio / pytest-asyncio | Nicht-blockierende Telegram-Kommunikation |
| **Containerisierung** | Docker | Reproduzierbares Deployment |

### Entwicklungs- & Testwerkzeuge

| Werkzeug | Zweck |
|----------|-------|
| pytest + pytest-mock | Unit- und Integrationstests (13 Module) |
| pytest-cov | Testabdeckungsmessung |
| freezegun | Zeitabhängige Tests |
| pyproject.toml | Modernes Paketmanagement (PEP 517/518) |

### Infrastruktur

- **Daten lokal**: kein externer Speicher, keine Cloud-API
- **Session-Persistenz**: StringSession (Telethon) in `.data/`
- **State-Dateien**: JSON-basierter Verarbeitungsstand für Resume-Fähigkeit
- **Deployments**: Vercel (Landing Page) + Docker (Applikation)

---

## 4. Ziele

### Primäres Ziel

Automatisierung des manuellen, zeitaufwändigen Prozesses der Aufzeichnung und Archivierung von Telegram-Sprachnachrichten – mit vollständiger Datenkontrolle beim Nutzer.

### Konkrete Anforderungen

| # | Ziel | Ergebnis |
|---|------|----------|
| Z1 | Kein Telegram Premium erforderlich | ✅ Nutzt kostenlose Telegram-API |
| Z2 | Vollständige Offline-Verarbeitung | ✅ Whisper läuft lokal, kein Cloud-Aufruf |
| Z3 | Datenschutzkonformität | ✅ Alle Daten verbleiben auf dem lokalen Rechner |
| Z4 | Einfache Bedienbarkeit | ✅ Web UI für Einsteiger, CLI für Automation |
| Z5 | Flexibles Filtern | ✅ Nach Datum, Absender, Nachrichtentyp |
| Z6 | Unterbrechbarer Betrieb | ✅ Persistenter State erlaubt Resume nach Abbruch |
| Z7 | Strukturierte Ausgabe | ✅ Markdown gruppiert nach Tag und Absender |

---

## 5. Umsetzung

### 5.1 Systemarchitektur

```
┌─────────────────────────────────────────────────────────┐
│                        Interfaces                        │
│          Streamlit Web UI         CLI (Typer)            │
└──────────────────┬────────────────────┬─────────────────┘
                   │                    │
                   ▼                    ▼
┌─────────────────────────────────────────────────────────┐
│                   ProcessingPipeline                     │
│  (Orchestrierung via Protocol-basierter Dependency-DI)  │
└──┬──────────────┬──────────────┬───────────────┬────────┘
   │              │              │               │
   ▼              ▼              ▼               ▼
TelegramCollector  MediaDownloader  WhisperTranscriber  MarkdownExporter
(Telethon)        (Cache + atomic  (faster-whisper,    (gruppiert nach
                   file write)      lazy loading)       Tag & Timezone)
                                        │
                                        ▼
                                  ProcessingState
                                  (JSON, resume-fähig)
```

### 5.2 Datenfluss

```
1. Authentifizierung (Phone → Code → optional 2FA)
          ↓
2. Nachrichtensammlung: TelegramCollector.collect()
   → Filterung nach Datum, Typ (VOICE/AUDIO/VIDEO_NOTE)
          ↓
3. Pro Nachricht (ProcessingPipeline._process_message):
   a. Bereits verarbeitet? → überspringen (ProcessingState)
   b. Download via MediaDownloader → .data/cache/
   c. Transkription via WhisperTranscriber (bei Bedarf)
   d. TranscriptEntry erstellen
          ↓
4. MarkdownExporter.render() → Markdown-Datei
5. ProcessingState.flush() → state.json persistieren
```

### 5.3 Schlüsselmuster & Design-Entscheidungen

| Muster | Begründung |
|--------|------------|
| **Protocol-basiertes DI** | Entkopplung ermöglicht vollständiges Unit-Testing ohne echte Telegram-Verbindung |
| **Lazy Model Loading** | Whisper-Modell (~2 GB) wird nur geladen, wenn tatsächlich Transkription nötig ist |
| **Atomische Dateioperationen** | Temp-Datei → Rename verhindert korrupte Zustände bei Unterbrechung |
| **Dry-Run-Modus** | Vorschau ohne Downloads/Transkription für schnelle Validierung |
| **Frozen Dataclasses** | Immutable Konfigurationsobjekte verhindern unbeabsichtigte Änderungen |
| **Zweistufige Filterung** | Erst Collection-Stage (Datum/Typ), dann Pipeline-Stage (Absender/Jahr) |

### 5.4 Modulübersicht

| Modul | Verantwortlichkeit |
|-------|--------------------|
| `cli.py` | Typer CLI, Async-Einstiegspunkt, Auth-Flow |
| `config.py` | Konfigurationsdataklassen, Pfadberechnung, Datumsparsen |
| `filters.py` | `MessageType`-Enum, `FilterConfig`, `should_include_message()` |
| `models.py` | `MessageEnvelope`, `TranscriptEntry`, `MessageSummary` |
| `pipeline.py` | `ProcessingPipeline` mit Dry-Run und Vollbetrieb |
| `tg_client.py` | `TelegramCollector` für Nachrichtensammlung |
| `state.py` | `ProcessingState` für Resume-Fähigkeit |
| `dry_run.py` | `DryRunReport` für Vorschaustatistiken |
| `transcribe.py` | `WhisperTranscriber` Wrapper |
| `download.py` | `MediaDownloader` mit Cache-Logik |
| `export_md.py` | `MarkdownExporter` mit Tages-Gruppierung |
| `web_auth.py` | Zustandsmaschine für Web-Authentifizierung |
| `async_helpers.py` | Persistenter Event-Loop für Streamlit-Kompatibilität |

### 5.5 Testabdeckung

```
13 Testmodule | Pytest + pytest-asyncio
─────────────────────────────────────────
test_async_helpers.py   Async Event-Loop-Persistenz
test_cli.py             CLI-Argument-Parsing & Konfiguration
test_config.py          Slugify, Pfadberechnung, Typen-Validierung
test_download.py        Cache-Hit / Cache-Miss, atomische Writes
test_dry_run.py         Vorschaustatistiken
test_filters.py         Absender-, Typ-, Jahresfilter (7 Fälle)
test_markdown_export.py Tages-Gruppierung, Timezone-Lokalisierung
test_message_type.py    Typ-Erkennung aus Telethon-Attributen
test_pipeline.py        Orchestrierung (Dry-Run, Full, Resume)
test_state.py           Persistenz, Max-History-Trimming
test_tg_client.py       Nachrichtensammlung, Datumsfilter
test_transcriber.py     Whisper-Modell-Aufruf mit Defaults
test_web_auth.py        Auth-Zustandsmaschine

Ausführung: pytest --cov=telegram_voice_transcriber --cov-report=term-missing
```

---

## 6. Vorher-Nachher-Ergebnisse

### Problem vor dem Projekt

| Situation | Aufwand |
|-----------|---------|
| Sprachnachricht manuell anhören | ~1× Echtzeit (1 Min Nachricht = 1 Min Aufwand) |
| Inhalt notieren / zusammenfassen | Zusätzlich 3–5 Min manuell |
| Chat durchsuchen | Nicht möglich bei Sprachnachrichten |
| Archivierung / Dokumentation | Kein strukturiertes Format |
| 50 Sprachnachrichten auswerten | ~3–4 Stunden manuell |

### Ergebnis nach dem Projekt

| Situation | Aufwand |
|-----------|---------|
| 50 Sprachnachrichten exportieren + transkribieren | ~5–10 Min (CPU) |
| Volltext-Durchsuchbarkeit | Sofort via Markdown |
| Strukturierte Tagesansicht | Automatisch gruppiert |
| Archivierung | Versionierbar als `.md`-Datei |
| Wiederholte Läufe (Resume) | Nur neue Nachrichten werden verarbeitet |

### Messbare Verbesserungen

```
⏱ Zeitersparnis:    ~95 % weniger manuelle Verarbeitungszeit
🔍 Durchsuchbarkeit: 0 % → 100 % (vollständiger Volltext)
🔒 Datenschutz:      100 % lokal (keine Cloud-API)
♻️  Resumbarkeit:    Beliebig unterbrechbar ohne Datenverlust
📁 Ausgabeformat:    Maschinenlesbar & menschenlesbar (Markdown)
```

### Beispielausgabe (Markdown-Export)

```markdown
# Transkript – Projektteam (2025)

## 2025-03-15

**09:14 – Max Mustermann:** Ich habe den Bericht fertiggestellt und warte
auf euer Feedback. Können wir heute Nachmittag kurz telefonieren? 🎙️ [#ID: 12345]

**09:32 – Anna Beispiel:** Ja, 15 Uhr passt mir gut. Ich schaue mir den
Bericht vorher noch mal an. 🎙️ [#ID: 12346]

## 2025-03-16

**10:05 – Max Mustermann:** Das Feedback war sehr hilfreich. Ich passe
die Struktur noch heute an. 🎙️ [#ID: 12389]
```

---

## 7. Herausforderungen & Lösungsansätze

| Herausforderung | Lösung |
|-----------------|--------|
| Streamlit ist synchron, Telethon asynchron | Persistenter Daemon-Thread mit eigenem Event-Loop (`async_helpers.py`) |
| Whisper-Modell zu groß für Sofortladen | Lazy Loading: Modell wird erst geladen wenn `requires_transcription()` true |
| Unterbrechungen bei großen Chats | JSON-State mit atomischen Writes + Processed-ID-Set |
| Telegram-Auth in Web UI (stateful) | Explicit State Machine (`AuthState` Enum) mit `WebAuthManager` |
| Verschiedene Nachrichtentypen | `MessageType`-Enum + zweistufige Filterung |
| Timezone-Inkonsistenzen | UTC-Speicherung, lokale Ausgabe via `tzlocal` |

---

## 8. KI-Einsatz & Automatisierungsaspekte

### Eingesetzte KI-Technologie

| Komponente | Details |
|------------|---------|
| **Modell** | OpenAI Whisper (via faster-whisper, quantisiert) |
| **Modellgrößen** | `tiny`, `base`, `small`, `medium`, `large-v3` (konfigurierbar) |
| **Sprachen** | Mehrsprachig, Standard: Deutsch (`de`) |
| **Laufzeit** | 100 % lokal, kein API-Key, keine Latenz durch Netzwerk |
| **Performance** | faster-whisper: ~4× schneller als Original-Whisper durch CTranslate2-Quantisierung |
| **VAD-Filter** | Voice Activity Detection verhindert Transkription von Stille |

### Automatisierungspotenzial

```bash
# Vollautomatische Ausführung via CLI (z.B. als Cronjob)
tg-transcribe "Projektteam" --year 2025 --model large-v3 --language de

# Mehrere Chats nacheinander
for chat in "Team A" "Team B" "Kunden"; do
  tg-transcribe "$chat" --year 2025
done

# In CI/CD-Pipeline oder Docker
docker run -e TG_API_ID=$ID -e TG_API_HASH=$HASH \
  telegram-transcriber tg-transcribe "Backup Chat" --year 2025
```

---

## 9. Relevanz für Automation & KI-Projekte bei Veritas

### Transferierbare Kompetenzen

| Kompetenz | Nachweis im Projekt |
|-----------|---------------------|
| **KI-Modell-Integration** | faster-whisper eingebunden, konfigurierbar, lazy-loaded |
| **Prozessautomatisierung** | Vollständig automatisierbare CLI-Pipeline |
| **Software-Architektur** | Protocol-basiertes DI, saubere Modul-Trennung |
| **Async-Programmierung** | asyncio, pytest-asyncio, Thread-Pool für Streamlit |
| **Testgetriebene Entwicklung** | 13 Testmodule, TDD-Philosophie |
| **Datenschutz by Design** | Lokale Verarbeitung, keine Cloud-Abhängigkeiten |
| **Benutzerfreundlichkeit** | Zwei Interfaces (Web + CLI) für unterschiedliche Zielgruppen |
| **DevOps** | Docker, Umgebungsvariablen, reproduzierbares Setup |

---

## 10. Quick Start

```bash
# 1. Repository klonen
git clone https://github.com/codeme-ne/Telegram-Voice-Transcriber
cd Telegram-Voice-Transcriber

# 2. Abhängigkeiten installieren
pip install -e '.[dev]'

# 3. API-Credentials konfigurieren
cp .env.example .env
# TG_API_ID und TG_API_HASH eintragen (von my.telegram.org)

# 4a. Web UI starten
streamlit run app.py

# 4b. Oder CLI nutzen
tg-transcribe "Chat Name" --year 2025 --dry-run
```

**Systemvoraussetzungen:** Python 3.10+, ffmpeg, ~2 GB Festplattenspeicher (Whisper-Modell)

---

*Erstellt für die Bewerbung als Automation & KI-Experte | © 2025*
