# docdb_ingestion — Testing

pytest runs `unittest`-style test classes.

## Test files

| File | Covers |
|---|---|
| `tests/test_aws_secrets.py` | DSN resolution with the flag on and off; secret key shapes; percent-encoding; errors that never leak values; refetch and retry once after rotation; importing without `boto3` |
| `tests/test_extra_data_capture.py` | `xml_to_dict` (nesting, lists, attributes, tail text, depth limit) and `extra_data` capture through `extract_document_data` |
| `tests/test_frontfile_runner.py` | Delivery classification, run-summary arithmetic, report emails and halt behaviour, with the API, the DB and the pipeline stubbed |
| `tests/test_notifications.py` | The SMTP helper |
| `tests/test_integration.py` | Index parsing and one inner ZIP, against a local sample delivery (set the sample path before running) |

`tests/test_empty_xml.py` and `tests/test_process_folder.py` are helper scripts; `test_empty_xml.py` creates a `tmp_dummy/` folder in the current directory when imported.

## How to run

To keep the repo clean, run from a scratch directory:

```
cd <any scratch dir>
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=~/repos/bdds/docdb_ingestion \
  ~/repos/bdds/docdb_ingestion/.venv/bin/python -m pytest -p no:cacheprovider -q \
  ~/repos/bdds/docdb_ingestion/tests/test_{aws_secrets,extra_data_capture,frontfile_runner,notifications}.py
```

From the repo root, `pytest tests/` also works, and collects `test_empty_xml.py` as well.

## Test data

EPO DOCDB data is public; keep fixtures small (a few `exchange-document` records per file).
