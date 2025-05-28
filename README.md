```python
import regex, json

json_pattern = regex.compile(r'''
    \{                     # opening brace
        (?:                # non‑capturing:
            [^{}]          #   – any char except braces
          | "(?:\\.|[^"\\])*"   #   – or a quoted string (handles \" escapes)
          | (?R)           #   – OR a recursive‑call: another {...}
        )*                 #   … repeated any number of times
    \}                     # closing brace
''', regex.VERBOSE | regex.DOTALL)

text = '''
    some noise
    {"user": {"name": "Ada", "roles": ["admin", "editor"]}, "active": true}
    trailing noise
'''

match = json_pattern.search(text)
if match:
    data = json.loads(match.group(0))
    print(data['user']['roles'])   # ['admin', 'editor']
```
