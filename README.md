# pii_detection

```python
"""
pii_redactor.py  –  Pure‑stdlib PII redaction helper for IRS consolidated‑report Markdown.
"""
from __future__ import annotations
import re, json
from dataclasses import dataclass, field
from collections import OrderedDict, defaultdict
from typing import Dict, Tuple, List

# ----------------------------------------------------------------------
# 1.  Configurable label → canonical‑tag map (edit to taste)
LABEL_MAP = {
    # SSN / ITIN / EIN
    "ssn": "SSN",
    "social security number": "SSN",
    "itin": "ITIN",
    "ein": "EIN",
    # People & dates
    "taxpayer name": "NAME",
    "spouse name": "NAME",
    "name": "NAME",
    "dob": "DOB",
    "date of birth": "DOB",
    # Contact
    "phone": "PHONE",
    "phone number": "PHONE",
    "email": "EMAIL",
    "address": "ADDRESS",
    "zip": "ZIP",
    # Banking
    "routing": "ROUTING",
    "routing number": "ROUTING",
    "bank account": "BANK_ACCT",
    "account #": "BANK_ACCT",
    "caf": "CAF",
}

# ----------------------------------------------------------------------
# 2.  Regex recognizers for structured PII (after the labeled pass)
REGEX_PATTERNS = OrderedDict([
    ("SSN",        r"(?<!\d)(\d{3})[-\s]?(\d{2})[-\s]?(\d{4})(?!\d)"),
    ("ITIN",       r"(?<!\d)9\d{2}[-\s]?8\d[-\s]?\d{4}(?!\d)"),
    ("EIN",        r"(?<!\d)\d{2}[-\s]?\d{7}(?!\d)"),
    ("PHONE",      r"(?<!\d)(?:\+1[-.\s]?)?(?:\(\d{3}\)|\d{3})[-.\s]?\d{3}[-.\s]?\d{4}(?!\d)"),
    ("ROUTING",    r"(?<!\d)\d{9}(?!\d)"),
    ("BANK_ACCT",  r"(?i)\b(?:acct(?:\.|ount)?|bank\s*acct\.?|account)[\s:#-]*\d{6,17}\b"),
    ("DOB",        r"(0[1-9]|1[0-2])[\/\-\.](0[1-9]|[12]\d|3[01])[\/\-\.](19|20)\d\d"),
    ("ZIP",        r"\b\d{5}(?:-\d{4})?\b"),
    # add more if needed …
])

# ----------------------------------------------------------------------
TABLE_ROW_RX = re.compile(
    r"^\|\s*([^|]+?)\s*\|\s*([^|]+?)\s*\|", re.MULTILINE
)

@dataclass
class PIIRedactor:
    label_map: Dict[str, str] = field(default_factory=lambda: LABEL_MAP)
    regex_patterns: Dict[str, str] = field(default_factory=lambda: REGEX_PATTERNS)
    placeholder_ctr: defaultdict = field(default_factory=lambda: defaultdict(int))
    mapping: "OrderedDict[str, str]" = field(default_factory=OrderedDict)

    # ----------------------------
    def _placeholder(self, tag: str) -> str:
        ph = f"[{tag}_{self.placeholder_ctr[tag]}]"
        self.placeholder_ctr[tag] += 1
        return ph

    # ----------------------------
    def _redact_value(
        self, text: str, value: str, tag: str
    ) -> Tuple[str, str]:
        """Replace *all* occurrences of value with a stable placeholder."""
        if value in self.mapping.values():
            # Already seen – reuse existing placeholder
            ph = next(k for k, v in self.mapping.items() if v == value)
        else:
            ph = self._placeholder(tag)
            self.mapping[ph] = value
        # simple global replace; could use re.escape if overlap risk
        return text.replace(value, ph), ph

    # ----------------------------
    def _pass1_labeled_tables(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        out = {}
        for name, md in md_dict.items():
            for field, val in TABLE_ROW_RX.findall(md):
                tag = self.label_map.get(field.strip().lower())
                if tag and val.strip():
                    md, _ = self._redact_value(md, val.strip(), tag)
            out[name] = md
        return out

    # ----------------------------
    def _pass2_mirror_known_values(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        if not self.mapping:
            return md_dict
        # Build one big regex of all captured values (longest first)
        escaped = sorted(map(re.escape, self.mapping.values()), key=len, reverse=True)
        combined = re.compile("|".join(escaped))
        out = {}
        for name, md in md_dict.items():
            def _sub(m):
                val = m.group(0)
                ph = next(k for k, v in self.mapping.items() if v == val)
                return ph
            out[name] = combined.sub(_sub, md)
        return out

    # ----------------------------
    def _pass3_structured_regex(self, md_dict: Dict[str, str]) -> Dict[str, str]:
        out = {}
        for name, md in md_dict.items():
            for tag, pattern in self.regex_patterns.items():
                rx = re.compile(pattern)
                for m in rx.finditer(md):
                    md, _ = self._redact_value(md, m.group(0), tag)
            out[name] = md
        return out

    # ----------------------------
    def redact(self, md_dict: Dict[str, str]) -> Tuple[Dict[str, str], Dict[str, str]]:
        """
        Returns (redacted_md_dict, placeholder→value mapping).
        """
        step1 = self._pass1_labeled_tables(md_dict)
        step2 = self._pass2_mirror_known_values(step1)
        step3 = self._pass3_structured_regex(step2)
        return step3, self.mapping

    # ----------------------------
    @staticmethod
    def unredact(text: str, mapping: Dict[str, str]) -> str:
        for ph, val in mapping.items():
            text = text.replace(ph, val)
        return text


# ----------------------------------------------------------------------
# Example CLI usage ----------------------------------------------------
if __name__ == "__main__":
    import pathlib, argparse

    p = argparse.ArgumentParser(description="Redact consolidated‑report Markdown.")
    p.add_argument("src_dir", help="Directory of *.md files (one per sheet)")
    p.add_argument("--out", default="out_redacted", help="Output dir for redacted files")
    args = p.parse_args()

    src = pathlib.Path(args.src_dir)
    out = pathlib.Path(args.out)
    out.mkdir(exist_ok=True)

    # Load Markdown files into a dict
    md_dict = {f.stem: f.read_text() for f in src.glob("*.md")}

    redactor = PIIRedactor()
    redacted_dict, mapping = redactor.redact(md_dict)

    # Write results
    for name, txt in redacted_dict.items():
        (out / f"{name}.md").write_text(txt, newline="\n")
    (out / "redaction_map.json").write_text(json.dumps(mapping, indent=2))

    print(f"Redacted files in {out}/ – mapping saved to redaction_map.json")
```
