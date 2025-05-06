# pii_detection

```python
# redact.py  –  no imports beyond the stdlib
import re, json, uuid
from collections import defaultdict, OrderedDict

PII_REGEXES = OrderedDict([
    # label, compiled_pattern
    ("SSN_MASKED",  re.compile(r"\*{3}[-\s]?\*{2}[-\s]?\d{4}", re.VERBOSE)),
    ("SSN",         re.compile(r"(?<!\d)(\d{3})[-\s]?(\d{2})[-\s]?(\d{4})(?!\d)", re.VERBOSE)),
    ("ITIN",        re.compile(r"(?<!\d)9\d{2}[-\s]?8\d[-\s]?\d{4}(?!\d)", re.VERBOSE)),
    ("EIN",         re.compile(r"(?<!\d)\d{2}[-\s]?\d{7}(?!\d)", re.VERBOSE)),
    ("ROUTING",     re.compile(r"(?<!\d)\d{9}(?!\d)", re.VERBOSE)),
    ("BANK_ACCT",   re.compile(r"(?i)\b(?:acct(?:\.|ount)?|bank\s*acct\.?|account)\s*[:#-]?\s*(\d{6,17})\b", re.VERBOSE)),
    ("CAF",         re.compile(r"(?i)\bCAF\s*[:#-]?\s*(\d{8,9})\b", re.VERBOSE)),
    ("IP",          re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b")),
    ("DOB",         re.compile(r"(0[1-9]|1[0-2])[\/\-\.](0[1-9]|[12]\d|3[01])[\/\-\.](19|20)\d\d", re.VERBOSE)),
    ("ISO_DATE",    re.compile(r"\d{4}-\d{2}-\d{2}")),
    ("CITY_STATE_ZIP", re.compile(r"\b[A-Z][a-zA-Z]+,\s+[A-Z]{2}\s+\d{5}(?:-\d{4})?\b")),
    ("ADDRESS_1",   re.compile(r"\b\d{1,5}\s+(?:[A-Z][a-z]+\s+)+(?:Ave|St|Rd|Blvd|Ln|Dr|Ct|Pl|Way)\b", re.VERBOSE)),
    ("ZIP",         re.compile(r"\b\d{5}(?:-\d{4})?\b")),
    ("NAME",        re.compile(r"\b[A-Z][a-z]+(?:\s+[A-Z]\.)?(?:\s+[A-Z][a-z]+)+\b")),
])

def _placeholder(label, counter):
    return f"[{label}_{counter[label]}]"

def redact(text: str):
    """Redact PII and return (redacted_text, mapping_dict)."""
    counter   = defaultdict(int)
    mapping   = OrderedDict()      # preserves encounter order
    redacted  = text

    # Pass 1 – scan & build a global match list (start, end, label, value)
    hits = []
    for label, regex in PII_REGEXES.items():
        hits.extend((m.start(), m.end(), label, m.group(0)) for m in regex.finditer(text))
    hits.sort()                    # left‑to‑right prevents offset drift

    # Pass 2 – build redacted string incrementally
    parts = []
    last  = 0
    for start, end, label, value in hits:
        if start < last:           # overlap with an earlier replacement
            continue
        parts.append(text[last:start])
        ph = _placeholder(label, counter)
        parts.append(ph)
        counter[label] += 1
        mapping[ph] = value
        last = end
    parts.append(text[last:])
    redacted = "".join(parts)
    return redacted, mapping

def unredact(text: str, mapping: dict):
    """Replace placeholders with original values."""
    for ph, val in mapping.items():
        text = text.replace(ph, val)
    return text

# --------------------------------------------------
if __name__ == "__main__":
    md_source = Path("report.md").read_text()
    redacted, mapping = redact(md_source)

    Path("report_redacted.md").write_text(redacted)
    Path("redaction_map.json").write_text(json.dumps(mapping, indent=2))
    # Later:  filled = unredact(slms_response, json.load(open("redaction_map.json")))
```
