# fabricator

Transform a Google Sheet URL into a fully executed [Islandora Workbench](https://mjordan.github.io/islandora_workbench_docs/) task.

## Overview

- Content creators can work in Google Sheets to prepare a spreadsheet
- [A Google Appscript](./google/appsscript) is embeded in the sheet to allow easily checking their work
- When the spreadsheet is ready, it can be ingested into Islandora/Drupal via Islandora Workbench by [supplying the sheet URL in the GitHub Action](../.././actions/workflows/run.yml)
  - requires write access on this repo to execute the workflow

```mermaid
sequenceDiagram
    actor Alice
    Alice->>Google Sheets: Edit 1
    Alice->>Google Sheets: Edit 2
    Alice->>Google Sheets: Edit ...
    Alice->>Google Sheets: Edit N
    Alice->>Google Sheets: Click check my work
    Google Sheets->>Fabricator: Check this CSV
    Fabricator->>Alice: Looks good 🚀
    Alice->>GitHub: Run workbench workflow
    GitHub->>Self-hosted Runner: Run workbench workflow
    Self-hosted Runner->>Islandora Workbench: python3 workbench
    Islandora Workbench->>Drupal: entity CUD
    Islandora Workbench->>GitHub: logs
    Alice->>GitHub: View action logs
```

## Development

This is an http service with two routes:

- `/workbench/check`
  - check if a google sheet content is well formed
- `/workbench/transform`
  - transform a google sheet CSV export into a workbench CSV

### Start the server

```
export SHARED_SECRET=changeme
go build
nohup ./fabricator &
echo $! > pid
```

### Get a workbench CSV from a google sheet CSV

```
curl -s \
  -H "X-Secret: $SHARED_SECRET" \
  -XPOST \
  -o target.zip \
  --upload-file source.csv \
  http://localhost:8080/workbench/transform
```

### Stop the service

```
kill $(cat pid)
```

## Adding new columns to the ingest template

If the ingest template needs a new column added, these are the code changes that are needed

- Add the column to [the ingest template](https://docs.google.com/spreadsheets/d/1iB7GsnfvhQO_c6TzJb7qwCnItqju0PMC8mNWepYqsnU/edit#gid=0), making row one the human-friendly label
- Make the necessary changes to [go-islandora](https://github.com/lehigh-university-libraries/go-islandora)
  - Add the column label and machine name to [the sheets struct in go-islandora](https://github.com/lehigh-university-libraries/go-islandora/blob/965bd728379bf2a9aa0ddb1fb46ec05fda636d87/cmd/sheetsStructs.go#L61)
  - generate the structs `go build && ./go-islandora generate sheets-structs --output=workbench.yaml`
- Make the necessary changes in this repo
  - Update go.mod in this repo `go get -u github.com/lehigh-university-libraries/go-islandora@main`
  - Add any necessary [checks](./internal/handlers/check.go) and [tests](./internal/handlers/check_test.go)
- Deploy the new image to the staging server
```
isle-stage
cd /opt/islandora/d10_lehigh_agile
sudo docker compose --profile prod pull
sudo systemctl restart islandora
```

TODO: This should eventually be able to be automatted, and the ingest template is simply generated by this repo.
