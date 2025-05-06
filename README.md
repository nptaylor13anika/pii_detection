# pii_detection

```python
"""
PII Redaction and Restoration Tool for IRS Consolidated Reports

This script performs schema-aware and regex-based PII detection on Markdown exports
of Excel sheets (e.g., TP Summary, Transactions, Letters). It produces a redacted
Markdown file and a JSON mapping of placeholders to original values, and can restore
original values.

Usage:
    python pii_redactor.py --input report.md --redacted report_redacted.md --map redaction_map.json
    python pii_redactor.py --restore slm_response.md --map redaction_map.json --output report_filled.md --restore
"""
import re
import json
import argparse
from collections import defaultdict, OrderedDict

# 1. Label-to-entity mapping (schema-aware)
LABEL_MAP = {
    'ssn': 'SSN', 'social security': 'SSN',
    'itin': 'ITIN', 'individual tax': 'ITIN',
    'ein': 'EIN', 'fed id': 'EIN', 'business id': 'EIN',
    'dob': 'DOB', 'date of birth': 'DOB',
    'first name': 'NAME', 'taxpayer name': 'NAME', 'spouse name': 'NAME',
    'addr': 'ADDRESS', 'address': 'ADDRESS', 'city': 'ADDRESS', 'state': 'ADDRESS', 'zip': 'ZIP',
    'routing': 'ROUTING', 'aba': 'ROUTING',
    'bank acct': 'BANK_ACCT', 'account #': 'BANK_ACCT',
    'caf': 'CAF'
}

# 2. Core regex patterns (out-of-place scan)
PII_REGEXES = OrderedDict([
    ('SSN',      re.compile(r"(?<!\d)\d{3}[- ]?\d{2}[- ]?\d{4}(?!\d)")),
    ('ITIN',     re.compile(r"(?<!\d)9\d{2}[- ]?8\d[- ]?\d{4}(?!\d)")),
    ('EIN',      re.compile(r"(?<!\d)\d{2}[- ]?\d{7}(?!\d)")),
    ('ROUTING',  re.compile(r"(?<!\d)\d{9}(?!\d)")),
    ('ZIP',      re.compile(r"\b\d{5}(?:-\d{4})?\b")),
    ('DOB_ISO',  re.compile(r"\d{4}-\d{2}-\d{2}")),
    ('DOB',      re.compile(r"(0[1-9]|1[0-2])[\/\-\.](0[1-9]|[12]\d|3[01])[\/\-\.](19|20)\d\d")),
    ('IP',       re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b")),
])

LBL_VALUE_RE = re.compile(r"(?i)(ssn|ein|dob)[: ]+(\*{3}[- ]?\*{2}[- ]?\d{4}|\d{3}[- ]?\d{2}[- ]?\d{4}|\d{2}[- ]?\d{7})")

# Placeholder generation

def _make_placeholder(label, counter):
    idx = counter[label]
    counter[label] += 1
    return f"[{label}_{idx}]"

# Markdown parser: split into table and text blocks

def parse_markdown_blocks(lines):
    blocks = []
    buf, in_table = [], False
    for line in lines:
        if line.strip().startswith('|'):
            if not in_table:
                if buf:
                    blocks.append(('text', ''.join(buf)))
                    buf = []
                in_table = True
            buf.append(line)
        else:
            if in_table:
                blocks.append(('table', ''.join(buf)))
                buf = []
                in_table = False
            buf.append(line)
    if buf:
        blocks.append(('table' if in_table else 'text', ''.join(buf)))
    return blocks

# Parse a Markdown table into headers + rows

def parse_table(md_table):
    lines = [l for l in md_table.splitlines() if l.strip()]
    header, sep, *rows = lines
    headers = [h.strip() for h in header.strip('|').split('|')]
    data_rows = [ [c.strip() for c in r.strip('|').split('|')] for r in rows if '|' in r ]
    return headers, data_rows

# 3. Redaction logic

def redact_markdown(text):
    lines = text.splitlines(keepends=True)
    blocks = parse_markdown_blocks(lines)

    # Determine PII columns by header tokens
    pii_columns = defaultdict(set)
    for typ, blk in blocks:
        if typ != 'table': continue
        headers, _ = parse_table(blk)
        for idx, hdr in enumerate(headers):
            low = hdr.lower()
            for token, ent in LABEL_MAP.items():
                if token in low:
                    pii_columns[ent].add(idx)

    # Prepare state
    counter = defaultdict(int)
    mapping = OrderedDict()
    out = []

    # Process blocks
    for typ, blk in blocks:
        if typ == 'table':
            headers, rows = parse_table(blk)
            # Rebuild table with redaction
            out.append('|' + '|'.join(headers) + '|\n')
            out.append('|' + '|'.join(['---']*len(headers)) + '|\n')
            for row in rows:
                new_cells = []
                for idx, cell in enumerate(row):
                    # Trusted redaction if header says PII
                    ent = next((e for e, cols in pii_columns.items() if idx in cols), None)
                    if ent:
                        ph = _make_placeholder(ent, counter)
                        mapping[ph] = cell
                        new_cells.append(ph)
                    else:
                        # leave for regex scan later
                        new_cells.append(cell)
                out.append('|' + '|'.join(new_cells) + '|\n')
        else:
            # Text: run inline label/value regex
            def rep_lbl(m):
                ent = m.group(1).upper()
                val = m.group(2)
                ph = _make_placeholder(ent, counter)
                mapping[ph] = val
                return f"{m.group(1)}:{ph}"
            blk = LBL_VALUE_RE.sub(rep_lbl, blk)
            # Now run regex-based out-of-place scan
            for ent, rx in PII_REGEXES.items():
                def repl(m):
                    val = m.group(0)
                    # optional sanity checks per entity
                    ph = _make_placeholder(ent, counter)
                    mapping[ph] = val
                    return ph
                blk = rx.sub(repl, blk)
            out.append(blk)

    redacted = ''.join(out)
    return redacted, mapping

# Restoration logic

def restore_text(text, mapping):
    for ph, val in mapping.items():
        text = text.replace(ph, val)
    return text

# CLI
if __name__ == '__main__':
    p = argparse.ArgumentParser(description='PII Redaction and Restoration Tool')
    p.add_argument('--input',  required=True, help='Markdown input file')
    p.add_argument('--redacted', help='Write redacted Markdown')
    p.add_argument('--map',     required=True, help='Read/Write JSON mapping')
    p.add_argument('--restore', action='store_true', help='Restore mode')
    p.add_argument('--output',  help='Output file for restore mode')
    args = p.parse_args()

    with open(args.input, 'r', encoding='utf-8') as f:
        text = f.read()

    if args.restore:
        mapping = json.load(open(args.map))
        filled = restore_text(text, mapping)
        with open(args.output, 'w', encoding='utf-8') as f:
            f.write(filled)
    else:
        redacted, mapping = redact_markdown(text)
        with open(args.redacted, 'w', encoding='utf-8') as f:
            f.write(redacted)
        with open(args.map, 'w', encoding='utf-8') as f:
            json.dump(mapping, f, indent=2)
```
