# Microsoft Teams Forensic Suite

A standalone digital forensics tool for parsing, analyzing and viewing **Microsoft Teams** artifacts. Each investigation is organized into its own self-contained **case**, with a Teams-style desktop viewer, searchable data and CSV reports.

> Runs fully offline. The built-in web server binds to `127.0.0.1` only, and no data leaves the machine.

---

## Features

- **Case-based workflow:** every run creates its own portable folder under `cases/`.
- **Multi-source ingestion** of Teams data (see [Data Sources](#data-sources)).
- **Message normalization:** raw Teams markup is turned into readable events:
  - Adaptive Cards (Base64 JSON)
  - Call started/ended events, with participants and durations
  - Members added/removed, topic changes, meeting policy updates
  - File shares and call recordings
- **Contact resolution:** maps IDs such as `8:orgid:<guid>` to real names, emails, roles and departments.
- **Media and document extraction:** recovers shared images and documents from the cache. It detects file types by magic bytes and filters out avatars, emojis, icons and tiny telemetry images (< 3.5 KB).
- **Calls and calendar:** reconstructs call history and scheduled meetings.
- **Desktop-style viewer:** tabs for Chats, Search, Calls, Meetings, Files, Media, Contacts and Overview.
- **Full-text search** across a case's messages.
- **Exports:** per-chat HTML export, CSV reports, a media ZIP and a text audit report.
- **Dual timestamps:** shown in UTC and IST (UTC+5:30). See [Customization](#customization).

---

## Requirements

- Windows (paths and launcher are Windows-oriented)
- Python 3.9+, with `tkinter` (tick "tcl/tk and IDLE" in the Python installer)
- Microsoft Edge or Google Chrome, for the app-style window (falls back to your default browser)

## Installation

```bash
pip install -r requirements.txt
pip install --no-deps dfindexeddb
```

`dfindexeddb` is installed with `--no-deps` because it pins `python-snappy` and `zstd`, which need a C compiler on Windows. The tool replaces both with `cramjam` at runtime.

## Usage

```bash
python teams_forensic_tool.py
```

Or use the `run_tool.bat` launcher if you have one.

1. The tool starts a local server (default port **5005**, or the next free port) and opens the viewer in a standalone Edge/Chrome window.
2. Enter a **Case Reference Name**.
3. Click to select the Teams data folder. The picker starts at the first Teams location it finds on the machine.
4. Watch the progress bar and live log while the 5-step pipeline runs.
5. Browse the results in the viewer, or open the case folder to get the CSVs and media.

Stop the server with `Ctrl+C` in the terminal.

---

## Data Sources

**New Teams:**
```
%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Roaming\Microsoft\MSTeams\IndexedDB
```

**Classic Teams:**
```
%APPDATA%\Microsoft\Teams\IndexedDB
```

You can also point the tool at a copy of these folders taken from another machine or forensic image.

The tool tries these ingestion sources in order and stops once it has enough message data:

| Priority | Source | Notes |
|---|---|---|
| 1 | `Teams_candidate_records.csv` | Searched in the target folder, the tool folder, the parent folder and up to 8 ancestors. |
| 2 | `Teams_Parsed_Success/*.json` | Pre-parsed IndexedDB dumps (UTF-16). |
| 3 | Raw LevelDB (`*.indexeddb.leveldb`) | Parsed directly with `dfindexeddb`. |
| + | `082469.json` | Optional write-ahead-log records, merged in if present. |

### IndexedDB database IDs used

| Database IDs | Content |
|---|---|
| 21, 110 | User profiles / contacts |
| 8, 104 | Conversation metadata |
| 9, 105 | Reply chains and messages |
| 93, 191 | Calls |
| 37, 128 | Calendar |

---

## Processing Pipeline

1. **Ingest** LevelDB / IndexedDB records and candidate logs.
2. **Correlate** contacts and the colleague directory.
3. **Process messages**, normalizing XML and JSON events.
4. **Extract media**: shared images and documents.
5. **Process calls and meetings.**
6. **Generate outputs**: CSVs, viewer and audit report.

---

## Output

Everything for a case is written to `cases/<CaseName>/`:

```
cases/
└── Case_<name>_<timestamp>/
    ├── Teams_Forensic_Viewer.html          # Standalone viewer
    ├── teams_forensic_data.js              # Dataset used by the viewer
    ├── Teams_Conversations_Summary.csv
    ├── Teams_Chat_Messages_Complete.csv
    ├── Teams_Calls_History.csv
    ├── Teams_Calendar_Meetings.csv
    ├── Teams_Shared_Files.csv
    ├── Teams_Contacts_Directory.csv
    ├── <audit report>.txt                  # Case audit report
    └── teams_media/
        └── documents/                      # Extracted images and files
```

CSV files are UTF-8 with BOM, so they open correctly in Excel. A case folder is portable and can be copied to another machine.

---

## Architecture

| Component | Technology |
|---|---|
| Backend / API | Flask (threaded, localhost only) |
| Raw DB parsing | `dfindexeddb`, with `cramjam` standing in for Snappy/Zstd |
| Image handling | Pillow |
| Folder picker | tkinter |
| Desktop window | Edge/Chrome `--app=` mode |
| Frontend | HTML, CSS and vanilla JavaScript (generated by the script) |

### HTTP endpoints

| Endpoint | Purpose |
|---|---|
| `GET /` | Case manager / home |
| `GET /api/cases` | List all cases and their stats |
| `POST /api/select_folder` | Pick a folder and start parsing |
| `POST /api/parse_path` | Start parsing a given path |
| `GET /api/progress` | Parsing progress and logs |
| `POST /api/search` | Full-text search within a case |
| `POST /api/export_chat_html` | Export a single chat as HTML |
| `POST /api/open_case_folder` | Open the case folder in Explorer |
| `GET /cases/<id>/viewer` | Open a case's viewer |
| `GET /cases/<id>/report` | Case audit report |
| `GET /download/media_zip` | Download extracted media as ZIP |

---

## Customization

- **Timezone:** the IST offset is hardcoded in `format_time()` (`timedelta(hours=5, minutes=30)`). Change it to your local timezone.
- **Default folder:** `native_folder_picker()` contains a hardcoded `E:\Sandeep_Analysis` start path. Remove or change it.
- **Port:** the default starting port is set in `find_available_port(start_port=5005)`.
- **Upload limit:** `MAX_CONTENT_LENGTH` is set to 2 GB.

---

## Forensic Best Practices

- Work on a **forensic copy** of the Teams data, never the live evidence. The tool reads the files but does not hash them or enforce write protection.
- Record **hashes** (e.g. SHA-256) of the source folder before and after processing.
- Close Teams before copying the live folders so LevelDB files are not locked or mid-write.
- Treat the output as a **reconstruction**. Deleted or overwritten LevelDB records may be partial, and message edit/delete flags depend on what remains on disk.
- Document the tool version, the date and the operator in your case notes.

## Limitations

- Windows-focused (paths, launcher, browser detection).
- The timezone display is fixed to UTC and IST unless you edit the code.
- Teams storage formats change between client versions. Newer builds may need parser updates.
- Parsing very large datasets can take a while and uses significant memory.

## Disclaimer

Use this tool only on data you are legally authorized to examine. The authors are not responsible for misuse or for conclusions drawn from its output.
