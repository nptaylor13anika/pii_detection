# pii_detection

```python
# ----------------------------------------------
# color_print.py
# ----------------------------------------------
def color_print(text: str, color: str = "reset", *, end: str = "\n", flush: bool = False) -> None:
    """
    Print *text* in a specified *color* on an ANSI‑capable terminal.

    Parameters
    ----------
    text   : str
        The message you want to display.
    color  : str
        Any key from the `colors` dict below (case‑insensitive).
        Unknown keys silently fall back to 'reset' (no color).
    end    : str, optional
        What to print after *text* (just like the built‑in print).
    flush  : bool, optional
        Whether to forcibly flush the output buffer.
    """
    colors = {
        # standard 8
        "black": 30,   "red": 31,      "green": 32,     "yellow": 33,
        "blue": 34,    "magenta": 35,  "cyan": 36,      "white": 37,
        # bright/high‑intensity 8 (add 60)
        "bright_black": 90,  "bright_red": 91,     "bright_green": 92,
        "bright_yellow": 93, "bright_blue": 94,    "bright_magenta": 95,
        "bright_cyan": 96,   "bright_white": 97,
        # reset / default
        "reset": 0
    }

    code = colors.get(color.lower(), 0)          # default to “reset”
    # \033 is ESC, “[<code>m” selects the color, “[0m” resets it again
    print(f"\033[{code}m{text}\033[0m", end=end, flush=flush)

# -----------------------------------------------------------------
# EXAMPLES ---------------------------------------------------------
if __name__ == "__main__":
    color_print("Error!", "red")
    color_print("Success!", "green")
    color_print("Heads‑up:", "bright_yellow", end=" ")
    color_print("something notable")
```
