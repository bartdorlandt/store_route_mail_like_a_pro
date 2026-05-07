# How to store and route your (physical) mail like a pro

**Personal edition** — a practical system for capturing, routing, and finding documents years later.

## The problem

Physical mail, contracts, invoices, and notifications pile up fast. Retrieving a specific document from two years ago is nearly impossible without a system. Add a company, bookkeeping, and a tax authority that asks you to prove things from five years back — and the problem compounds.

This project documents the system built to solve that: automated document capture, routing, and long-term storage using open-source tooling and a small Python glue script.

## Solution overview

1. Scan a physical document with the Dropbox camera app — it enhances contrast, detects corners, and produces a PDF.
2. Dropbox syncs the PDF to a Synology NAS via CloudSync.
3. Drop the file into the appropriate routing folder on the NAS (or directly from laptop/phone).
4. Hourly, a Python script processes the folders and routes each file to its destination(s).
5. Successfully processed files are moved to `done/<folder>/` to avoid reprocessing.

## Stack

| Component                                                       | Purpose                                    |
| --------------------------------------------------------------- | ------------------------------------------ |
| Dropbox camera app                                              | Scan physical mail → PDF → cloud sync      |
| Synology NAS + CloudSync                                        | Local hosting and folder sync              |
| [Paperless-ngx](https://github.com/paperless-ngx/paperless-ngx) | Document management: store, tag, search    |
| Python + `uv`                                                   | Glue script to route files to destinations |
| Docker + docker-compose                                         | Run Paperless-ngx and its dependencies     |
| Scheduler (cron/Synology Task Scheduler)                        | Run the script hourly                      |

## Paperless-ngx

Paperless-ngx is an open-source document management system with a web interface, REST API, email ingestion, and folder watching. It supports correspondents, document types, storage paths, tags, and custom workflows for self-organizing your archive.

The Docker Compose stack runs: `paperless-ngx`, `postgres`, `redis`, `gotenberg` (PDF conversion), and `tika` (content extraction). See `docker-compose.yml` for the full configuration.

## Routing destinations

Documents are routed to one or more of:

- **Paperless** — long-term searchable archive
- **Bookkeeping system** — via email attachment
- **Bookkeeper** — via email attachment

## Folder-based routing

Place a file in the folder matching where it should go. The script handles the rest.

```
PROCESS_FOLDER/
  to_paperless/
  to_bookkeeping/
  to_bookkeeping_paperless/
  to_paperless_bookkeeper/
  to_bookkeeper/
  done/
    to_paperless/
    to_bookkeeping/
    ...
```

```python
process_folder(MAIN_PATH / "to_paperless",              processors=[paperless_processor])
process_folder(MAIN_PATH / "to_bookkeeping",            processors=[bookkeeping_processor])
process_folder(MAIN_PATH / "to_bookkeeping_paperless",  processors=[paperless_processor, bookkeeping_processor])
process_folder(MAIN_PATH / "to_paperless_bookkeeper",   processors=[paperless_processor, to_person_processor])
process_folder(MAIN_PATH / "to_bookkeeper",             processors=[to_person_processor])
```

A file is only moved to `done/` when all processors for that folder succeed. On failure, an error notification email is sent.

## Code design

The script uses a `FileProcessor` Protocol so processors are interchangeable:

```python
class FileProcessor(Protocol):
    def process(self, filepath: Path) -> bool: ...
```

Two implementations ship: `PaperlessAPIProcessor` (uploads via REST API) and `EmailProcessor` (sends as email attachment). Adding a new destination means implementing the protocol — no changes to the routing logic.

Configuration is fully environment-variable driven (API tokens, SMTP credentials, folder paths).

## Requirements

- Synology NAS with CloudSync and Docker support
- Docker + docker-compose
- Python 3.9+ via `uv` (recommended — easy version and dependency management on NAS)
- SMTP credentials for outbound email
- A scheduler (Synology Task Scheduler or cron)

## Takeaways

- `uv` makes managing a modern Python version and dependencies straightforward, even on a NAS
- The `Protocol`-based processor pattern keeps routing logic clean and extensible
- The time saved finding documents — and the peace of mind — far outweighs the setup effort

## Full code

The full source code is available at: https://github.com/bartdorlandt/paperless_email_processor

## About

**Bart Dorlandt** is the owner of Dream Networking and Automation. Twenty years in network engineering and automation, the last ten focused on: *there must be a better way*.

- LinkedIn: https://linkedin.com/in/bartdorlandt/
- Website: https://dreamnetworking.nl/
