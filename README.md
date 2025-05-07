# pii_detection

```python
# pii_redactor_v2.py  –  stdlib‑only redactor with smarter heuristics
from __future__ import annotations
import re, json
from dataclasses import dataclass, field
from collections import OrderedDict, defaultdict
from typing import Dict, Tuple

# ---------------------------------------------------------------------
# 1)  Field‑label normalisation  --------------------------------------
LABEL_MAP = {
    # personal id
    "ssn": "SSN",  "social security number": "SSN",
    "itin": "ITIN", "tin": "ITIN",
    "ein": "EIN",
    # people
    "taxpayer name": "NAME", "spouse name": "NAME",
    "dependent name": "NAME", "name": "NAME",
    "dob": "DOB", "date of birth": "DOB",
    # contact
    "phone": "PHONE", "phone number": "PHONE",
    "email": "EMAIL", "address": "ADDRESS", "zip": "ZIP",
    # bank / finance
    "routing": "ROUTING", "routing number": "ROUTING",
    "bank acct": "BANK_ACCT", "bank account": "BANK_ACCT",
    "account #": "BANK_ACCT", "caf": "CAF",
    # network
    "ip": "IP", "ip address": "IP",
}

# ---------------------------------------------------------------------
# 2)  Validation / detection regexes  ---------------------------------
REGEX_PATTERNS = OrderedDict([
    # core numeric IDs
    ("SSN",        r"(?<!\d)(\d{3})[-\s]?(\d{2})[-\s]?(\d{4})(?!\d)"),
    ("ITIN",       r"(?<!\d)9\d{2}[-\s]?8\d[-\s]?\d{4}(?!\d)"),
    ("EIN",        r"(?<!\d)\d{2}[-\s]?\d{7}(?!\d)"),
    ("CAF",        r"(?i)\bCAF[\s:#-]*\d{8,9}\b"),
    # contact
    ("PHONE",      r"(?<!\d)(?:\+1[-.\s]?)?(?:\(\d{3}\)|\d{3})[-.\s]?\d{3}[-.\s]?\d{4}(?!\d)"),
    ("EMAIL",      r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"),
    # bank
    ("ROUTING",    r"(?i)\brouting[\s:#-]*\d{9}\b"),
    ("BANK_ACCT",  r"(?i)\b(?:acct(?:\.|ount)?|bank\s*acct\.?|account)[\s:#-]*\d{6,17}\b"),
    # addresses
    ("ZIP",        r"\b[A-Z]{2}\s+\d{5}(?:-\d{4})?\b"),   # e.g. “NC 27603”
    # dates
    ("DOB",        r"(0[1-9]|1[0-2])[\/\-\.](0[1-9]|[12]\d|3[01])[\/\-\.](19|20)\d\d"),
    # network
    ("IP",         r"\b(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}"
                   r"(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\b"),
])

VALIDATION_RX = {tag: re.compile(pat, re.IGNORECASE) for tag, pat in REGEX_PATTERNS.items()}

# ---------------------------------------------------------------------
TABLE_ROW_RX   = re.compile(r"^\|\s*([^|]+?)\s*\|\s*([^|]+?)\s*\|", re.MULTILINE)
DIVIDER_ROW_RX = re.compile(r"^\s*\|\s*:?-+:?\s*\|", re.MULTILINE)   # --- row

@dataclass
class PIIRedactor:
    label_map: Dict[str, str] = field(default_factory=lambda: LABEL_MAP)
    regex_patterns: Dict[str, str] = field(default_factory=lambda: REGEX_PATTERNS)
    mapping: "OrderedDict[str, str]" = field(default_factory=OrderedDict)
    counter: defaultdict = field(default_factory=lambda: defaultdict(int))

    # -----------------------------------------------------------------
    # Helpers
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

    # -----------------------------------------------------------------
    # Pass 1 – labelled table cells
    def _pass_labeled(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        new = {}
        for sheet, md in md_dict.items():
            lines = md.splitlines()
            i = 0
            while i < len(lines):
                m = TABLE_ROW_RX.match(lines[i])
                if m:
                    # detect if *this* is the header row (next line is dividers)
                    is_header = (i + 1 < len(lines)) and DIVIDER_ROW_RX.match(lines[i + 1])
                    if not is_header:
                        field, val = m.group(1).strip(), m.group(2).strip()
                        tag = self.label_map.get(field.lower())
                        if tag and val:
                            # validate if we have a regex for that tag
                            rx = VALIDATION_RX.get(tag)
                            if (rx and rx.fullmatch(val)) or (tag == "NAME"):
                                md, _ = self._redact_value(md, val, tag)
                    i += 1
                else:
                    i += 1
            new[sheet] = md
        return new

    # -----------------------------------------------------------------
    # Pass 2 – mirror any value we already learned (dependants etc.)
    def _pass_mirror(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        if not self.mapping:
            return md_dict
        escaped = sorted(map(re.escape, self.mapping.values()), key=len, reverse=True)
        combined = re.compile("|".join(escaped))
        new = {}
        for sheet, md in md_dict.items():
            new[sheet] = combined.sub(
                lambda m: next(ph for ph, v in self.mapping.items() if v == m.group(0)),
                md
            )
        return new

    # -----------------------------------------------------------------
    # Pass 3 – pick up structured PII via regex
    def _pass_structured(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        new = {}
        for sheet, md in md_dict.items():
            for tag, pat in self.regex_patterns.items():
                rx = re.compile(pat)
                for m in rx.finditer(md):
                    md, _ = self._redact_value(md, m.group(0), tag)
            new[sheet] = md
        return new

    # -----------------------------------------------------------------
    # Pass 4 – extra names that share the TP/Spouse last‑name(s)
    def _pass_dependents(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        last_names = {
            v.split()[-1]
            for ph, v in self.mapping.items()
            if ph.startswith("[NAME_")
        }
        if not last_names:
            return md_dict
        name_rx = re.compile(
            r"\b[A-Z][a-z]+(?:\s+[A-Z]\.)?\s+(?:%s)\b" % "|".join(map(re.escape, last_names))
        )
        new = {}
        for sheet, md in md_dict.items():
            for m in name_rx.finditer(md):
                md, _ = self._redact_value(md, m.group(0), "NAME")
            new[sheet] = md
        return new

    # -----------------------------------------------------------------
    def redact(self, md_dict: Dict[str, str]) -> Tuple[Dict[str, str], Dict[str, str]]:
        step1 = self._pass_labeled(md_dict)
        step2 = self._pass_mirror(step1)
        step3 = self._pass_structured(step2)
        final = self._pass_dependents(step3)
        return final, self.mapping

    # -----------------------------------------------------------------
    @staticmethod
    def unredact(text: str, mapping: Dict[str, str]) -> str:
        for ph, val in mapping.items():
            text = text.replace(ph, val)
        return text
```
