# pii_detection

```python
#!/usr/bin/env python3
"""
pii_redactor.py  ·  Pure‑standard‑library PII redaction helper
--------------------------------------------------------------

•   Accepts a dict {sheet_name: markdown_text}.
•   Redacts PII in four waves (labeled cells, mirror, regex patterns,
    dependent names), producing:
        - redacted_dict   {sheet: redacted_markdown}
        - mapping         {placeholder: original_value}
•   Unredaction helper restores originals after your LLM/SLM replies.

New in this version
-------------------
✓  Labeled **DOB, DOD, ZIP** always redact (even bare 5‑digit ZIPs).  
✓  Added `DOD` tag (`Date of Death`) plus regex.  
✓  Validation relaxed for ZIP/DOB/DOD when value is explicitly labeled.  
✓  Still zero external dependencies – standard library only.
"""

from __future__ import annotations
import re
import json
import argparse
import pathlib
from dataclasses import dataclass, field
from collections import OrderedDict, defaultdict
from typing import Dict, Tuple, List

# ----------------------------------------------------------------------
# 1. Canonical PII labels (lower‑case, punctuation removed)
# ----------------------------------------------------------------------
LABEL_MAP: Dict[str, str] = {
    # identity numbers
    "ssn": "SSN",
    "socialsecuritynumber": "SSN",
    "itin": "ITIN",
    "tin": "TIN",
    "taxidentificationnumber": "TIN",
    "ein": "EIN",
    # names & dates
    "taxpayername": "NAME",
    "spousename": "NAME",
    "dependentname": "NAME",
    "name": "NAME",
    "dob": "DOB",
    "dateofbirth": "DOB",
    "dod": "DOD",
    "dateofdeath": "DOD",
    # contact
    "phone": "PHONE",
    "phonenumber": "PHONE",
    "email": "EMAIL",
    "address": "ADDRESS",
    "city": "ADDRESS",
    "state": "ADDRESS",
    "zip": "ZIP",
    "zipcode": "ZIP",
    # banking
    "routing": "ROUTING",
    "routingnumber": "ROUTING",
    "bankaccount": "BANK_ACCT",
    "bankacct": "BANK_ACCT",
    "account": "BANK_ACCT",
    "account#": "BANK_ACCT",
    "caf": "CAF",
    # network
    "ip": "IP",
    "ipaddress": "IP",
}

# ----------------------------------------------------------------------
# 2. Regex recognisers for unlabeled PII (pattern pass)
# ----------------------------------------------------------------------
_DATE_RX = r"(0[1-9]|1[0-2])[\/\-\.](0[1-9]|[12]\d|3[01])[\/\-\.](19|20)\d\d"

REGEX_PATTERNS: "OrderedDict[str, str]" = OrderedDict([
    # numeric IDs
    ("SSN",        r"(?<!\d)(\d{3})[-\s]?(\d{2})[-\s]?(\d{4})(?!\d)"),
    ("ITIN",       r"(?<!\d)9\d{2}[-\s]?8\d[-\s]?\d{4}(?!\d)"),
    ("TIN",        r"(?<!\d)\d{3}[-\s]?\d{2}[-\s]?\d{4}(?!\d)"),  # generic 9‑digit tax‑ID
    ("EIN",        r"(?<!\d)\d{2}[-\s]?\d{7}(?!\d)"),
    ("CAF",        r"(?i)\bCAF[\s:#-]*\d{8,9}\b"),
    # contact & network
    ("PHONE",      r"(?<!\d)(?:\+1[-.\s]?)?(?:\(\d{3}\)|\d{3})[-.\s]?\d{3}[-.\s]?\d{4}(?!\d)"),
    ("EMAIL",      r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"),
    ("IP",         r"\b(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}"
                   r"(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\b"),
    # banking
    ("ROUTING",    r"(?i)\brouting[\s:#-]*\d{9}\b"),
    ("BANK_ACCT",  r"(?i)\b(?:acct(?:\.|ount)?|bank\s*acct\.?|account)"
                   r"[\s:#-]*\d{6,17}\b"),
    # addresses & dates
    ("ZIP",        r"\b[A-Z]{2}\s+\d{5}(?:-\d{4})?\b"),  # unlabeled requires state + ZIP
    ("DOB",        _DATE_RX),
    ("DOD",        _DATE_RX),
])

VALIDATION_RX: Dict[str, re.Pattern] = {
    tag: re.compile(pat, re.IGNORECASE) for tag, pat in REGEX_PATTERNS.items()
}
# allow bare 9‑digit routing numbers when value labeled "Routing"
VALIDATION_RX["ROUTING"] = re.compile(r"\d{9}$")

# ----------------------------------------------------------------------
# 3. Markdown helpers
# ----------------------------------------------------------------------
TABLE_ROW_RX   = re.compile(r"^\|\s*([^|]+?)\s*\|\s*([^|]+?)\s*\|", re.MULTILINE)
DIVIDER_ROW_RX = re.compile(r"^\s*\|\s*:?-+:?\s*(\|\s*:?-+:?\s*)+\|?\s*$", re.MULTILINE)

# ----------------------------------------------------------------------
# 4. PIIRedactor class
# ----------------------------------------------------------------------
@dataclass
class PIIRedactor:
    label_map: Dict[str, str] = field(default_factory=lambda: LABEL_MAP)
    regex_patterns: Dict[str, str] = field(default_factory=lambda: REGEX_PATTERNS)
    mapping: "OrderedDict[str, str]" = field(default_factory=OrderedDict)
    counter: defaultdict = field(default_factory=lambda: defaultdict(int))

    # ---- internals ---------------------------------------------------
    def _placeholder(self, tag: str) -> str:
        ph = f"[{tag}_{self.counter[tag]}]"
        self.counter[tag] += 1
        return ph

    def _redact_value(self, text: str, value: str, tag: str) -> Tuple[str, str]:
        if value in self.mapping.values():
            ph = next(ph for ph, v in self.mapping.items() if v == value)
        else:
            ph = self._placeholder(tag)
            self.mapping[ph] = value
        return text.replace(value, ph), ph

    # ---- label utilities --------------------------------------------
    def _norm_label(self, label: str) -> str:
        return re.sub(r"[^a-z]", "", label.lower())

    def _detect_tag(self, raw_label: str) -> str | None:
        norm = self._norm_label(raw_label)
        return next((t for lbl, t in self.label_map.items() if lbl in norm), None)

    # ---- pass 1: labeled table cells & PII columns -------------------
    def _redact_table_block(self, lines: List[str], start: int, end: int) -> None:
        header_cells = [c.strip() for c in lines[start].split("|")[1:-1]]
        pii_cols: List[Tuple[int, str]] = []
        for idx, cell in enumerate(header_cells):
            tag = self._detect_tag(cell)
            if tag:
                pii_cols.append((idx, tag))
        if not pii_cols:
            return

        for i in range(start + 2, end):
            cells = lines[i].split("|")
            if len(cells) < 3:
                continue
            for idx, tag in pii_cols:
                if idx + 1 >= len(cells):
                    continue
                val = cells[idx + 1].strip()
                if not val:
                    continue
                rx = VALIDATION_RX.get(tag)
                # relax validation for NAME, ADDRESS, ZIP, DOB, DOD
                relaxed_tags = ("NAME", "ADDRESS", "ZIP", "DOB", "DOD")
                if rx and not rx.fullmatch(val) and tag not in relaxed_tags:
                    continue
                if tag == "ROUTING" and not rx.fullmatch(val):
                    continue
                lines[i], _ = self._redact_value(lines[i], val, tag)

    def _pass_labeled(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        new = {}
        for sheet, md in md_dict.items():
            lines = md.splitlines()
            # single‑row Field | Value tables
            for match in TABLE_ROW_RX.finditer(md):
                field, val = match.group(1).strip(), match.group(2).strip()
                tag = self._detect_tag(field)
                if tag and val:
                    rx = VALIDATION_RX.get(tag)
                    relaxed = tag in ("NAME", "ADDRESS", "ZIP", "DOB", "DOD")
                    if (rx is None) or rx.fullmatch(val) or relaxed:
                        md, _ = self._redact_value(md, val, tag)

            # multi‑row tables – redact columns
            block_idx: List[Tuple[int, int]] = []
            cur: List[int] = []
            for idx, line in enumerate(lines + [""]):  # sentinel
                if line.lstrip().startswith("|"):
                    cur.append(idx)
                elif cur:
                    block_idx.append((cur[0], idx))
                    cur = []
            for b_start, b_end in block_idx:
                if b_end - b_start >= 3 and DIVIDER_ROW_RX.match(lines[b_start + 1]):
                    self._redact_table_block(lines, b_start, b_end)
            new[sheet] = "\n".join(lines)
        return new

    # ---- pass 2: mirror already captured values ---------------------
    def _pass_mirror(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        if not self.mapping:
            return md_dict
        escaped = sorted(map(re.escape, self.mapping.values()), key=len, reverse=True)
        combined_rx = re.compile("|".join(escaped))
        new = {}
        for sheet, md in md_dict.items():
            def _sub(m):
                val = m.group(0)
                ph = next(ph for ph, v in self.mapping.items() if v == val)
                return ph
            new[sheet] = combined_rx.sub(_sub, md)
        return new

    # ---- pass 3: structured regex recognisers -----------------------
    def _pass_structured(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        new = {}
        for sheet, md in md_dict.items():
            for tag, pat in self.regex_patterns.items():
                rx = re.compile(pat)
                for m in rx.finditer(md):
                    md, _ = self._redact_value(md, m.group(0), tag)
            new[sheet] = md
        return new

    # ---- pass 4: dependent names (same last name) -------------------
    def _pass_dependents(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        last_names = {v.split()[-1] for ph, v in self.mapping.items() if ph.startswith("[NAME_")}
        if not last_names:
            return md_dict
        dep_rx = re.compile(
            r"\b[A-Z][a-z]+(?:\s+[A-Z]\.)?\s+(?:%s)\b" % "|".join(map(re.escape, last_names))
        )
        new = {}
        for sheet, md in md_dict.items():
            for m in dep_rx.finditer(md):
                md, _ = self._redact_value(md, m.group(0), "NAME")
            new[sheet] = md
        return new

    # ---- public API --------------------------------------------------
    def redact(self, md_dict: Dict[str, str]) -> Tuple[Dict[str, str], Dict[str, str]]:
        step1 = self._pass_labeled(md_dict)
        step2 = self._pass_mirror(step1)
        step3 = self._pass_structured(step2)
        final = self._pass_dependents(step3)
        return final, self.mapping

    @staticmethod
    def unredact(text: str, mapping: Dict[str, str]) -> str:
        for ph, val in mapping.items():
            text = text.replace(ph, val)
        return text
```
