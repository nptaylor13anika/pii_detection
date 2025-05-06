# pii_detection

```python
"""
PII Redactor as a Python class: schema-aware for 'TP Summary' and regex-based for others.

Usage:
    redactor = PiiRedactor()
    redacted_dict, mapping = redactor.process(markdown_tables_dict)
"""
import re
from collections import defaultdict, OrderedDict

class PiiRedactor:
    # Mapping tokens in headers to PII entity labels
    LABEL_MAP = {
        'ssn': 'SSN', 'social security': 'SSN',
        'itin': 'ITIN', 'individual tax': 'ITIN',
        'ein': 'EIN', 'fed id': 'EIN', 'business id': 'EIN',
        'dob': 'DOB', 'date of birth': 'DOB',
        'first name': 'NAME', 'taxpayer name': 'NAME', 'spouse name': 'NAME',
        'addr': 'ADDRESS', 'address': 'ADDRESS',
        'city': 'ADDRESS', 'state': 'ADDRESS', 'zip': 'ZIP',
        'routing': 'ROUTING', 'aba': 'ROUTING',
        'bank acct': 'BANK_ACCT', 'account #': 'BANK_ACCT',
        'caf': 'CAF'
    }

    # Regex patterns for out-of-place PII detection
    PII_REGEXES = OrderedDict([
        ('SSN',     re.compile(r"(?<!\d)\d{3}[- ]?\d{2}[- ]?\d{4}(?!\d)")),
        ('ITIN',    re.compile(r"(?<!\d)9\d{2}[- ]?8\d[- ]?\d{4}(?!\d)")),
        ('EIN',     re.compile(r"(?<!\d)\d{2}[- ]?\d{7}(?!\d)")),
        ('ROUTING', re.compile(r"(?<!\d)\d{9}(?!\d)")),
        ('ZIP',     re.compile(r"\b\d{5}(?:-\d{4})?\b")),
        ('DOB_ISO', re.compile(r"\d{4}-\d{2}-\d{2}")),
        ('DOB',     re.compile(r"(0[1-9]|1[0-2])[\/\-\.](0[1-9]|[12]\d|3[01])[\/\-\.](19|20)\d\d")),
        ('IP',      re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b")),
    ])
    # Inline label:value regex (e.g. "SSN: 123-45-6789")
    LBL_VALUE_RE = re.compile(r"(?i)(ssn|ein|dob)[: ]+(\*{3}[- ]?\*{2}[- ]?\d{4}|\d{3}[- ]?\d{2}[- ]?\d{4}|\d{2}[- ]?\d{7})")

    def __init__(self):
        # counters and final mapping will be built per run
        pass

    def _placeholder(self, label, counter):
        idx = counter[label]
        counter[label] += 1
        return f"[{label}_{idx}]"

    def process(self, md_dict):
        """
        md_dict: dict of sheet_name -> markdown string
        Returns: (redacted_dict, mapping_dict)
          redacted_dict: same keys, redacted markdown
          mapping_dict: OrderedDict placeholder -> original value
        """
        counter = defaultdict(int)
        mapping = OrderedDict()
        redacted = {}

        # 1) Handle TP Summary schema-aware extraction
        tp_md = md_dict.get('TP Summary', '')
        redacted['TP Summary'] = self._redact_tp_summary(tp_md, mapping, counter)

        # 2) Handle other sheets: regex and inline labels
        for sheet, text in md_dict.items():
            if sheet == 'TP Summary':
                continue
            redacted[sheet] = self._redact_other(text, mapping, counter)

        return redacted, mapping

    def _redact_tp_summary(self, text, mapping, counter):
        """
        Parses TP Summary markdown table, redacts all PII columns fully.
        """
        lines = text.splitlines(keepends=True)
        # extract header line
        # assume first non-empty |...| line is header, next is separator
        hdr_idx = next(i for i,l in enumerate(lines) if l.strip().startswith('|'))
        headers = [h.strip() for h in lines[hdr_idx].strip('|').split('|')]
        # identify PII columns
        pii_cols = set()
        for idx, hdr in enumerate(headers):
            low = hdr.lower()
            for token, ent in self.LABEL_MAP.items():
                if token in low:
                    pii_cols.add(idx)
        # rebuild table
        out = []
        for i, l in enumerate(lines):
            if not l.strip().startswith('|'):
                out.append(l); continue
            cells = [c.strip() for c in l.strip('|').split('|')]
            if i <= hdr_idx+1:  # header or separator
                out.append(l)
            else:
                new = []
                for idx, cell in enumerate(cells):
                    if idx in pii_cols and cell:
                        ph = self._placeholder(self.LABEL_MAP.get(self._find_label(headers[idx].lower()), 'PII'), counter)
                        mapping[ph] = cell
                        new.append(ph)
                    else:
                        new.append(cell)
                out.append('|' + '|'.join(new) + '|\n')
        return ''.join(out)

    def _find_label(self, hdr_lower):
        for token, ent in self.LABEL_MAP.items():
            if token in hdr_lower:
                return token
        return None

    def _redact_other(self, text, mapping, counter):
        """
        Inline label extraction + regex-based out-of-place redaction.
        """
        # inline label:value
        def inline_repl(m):
            ent = m.group(1).upper()
            val = m.group(2)
            ph = self._placeholder(ent, counter)
            mapping[ph] = val
            return f"{m.group(1)}:{ph}"
        text = self.LBL_VALUE_RE.sub(inline_repl, text)
        # regex scan
        for ent, rx in self.PII_REGEXES.items():
            def repl(m):
                val = m.group(0)
                ph = self._placeholder(ent, counter)
                mapping[ph] = val
                return ph
            text = rx.sub(repl, text)
        return text

    def restore(self, text, mapping):
        """Restore placeholders to original values in any text."""
        for ph, val in mapping.items():
            text = text.replace(ph, val)
        return text
```
