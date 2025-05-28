```python
import re, json

pattern = re.compile(
    r'\{'                 # opening brace
    r'(?:[^{}"]|'         # … anything except braces or quotes
    r'"(?:\\.|[^"\\])*")*'# … or a full JSON string
    r'\}'                 # closing brace
)

def extract_json(text: str):
    m = pattern.search(text)
    return json.loads(m.group(0)) if m else None
```
