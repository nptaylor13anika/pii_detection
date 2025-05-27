```python
from pathlib import Path
from typing import Tuple

from openpyxl import load_workbook
from openpyxl.utils import range_boundaries
from openpyxl.worksheet.worksheet import Worksheet

def copy_block(src_ws: Worksheet, src_range: str) -> Tuple[list[list], list]:
    """
    Extract the values **and** cell objects (for styling) from `src_range`
    and return (values, merged_ranges) where

      values = [ [cell1, cell2, …], … ]  (2‑D list, row‑major)
      merged_ranges = [openpyxl.worksheet.cell_range.CellRange, …]
    """
    min_col, min_row, max_col, max_row = range_boundaries(src_range)
    rows = []
    for r in src_ws.iter_rows(
        min_row=min_row, max_row=max_row,
        min_col=min_col, max_col=max_col,
    ):
        rows.append([c for c in r])          # keep full cell objects

    # merged ranges entirely inside the block
    within_block = []
    for cr in src_ws.merged_cells.ranges:
        a, b, c, d = cr.bounds
        if a >= min_col and c <= max_col and b >= min_row and d <= max_row:
            within_block.append(cr)
    return rows, within_block


def paste_block(ws: Worksheet, rows, merges, blank_rows: int, start_row: int = 1, start_col: int = 1, copy_style: bool = True):
    """
    Insert enough rows, paste `rows` starting at (start_row, start_col),
    add `blank_rows` beneath, re‑create merged cells & styles if requested.
    """
    n_rows   = len(rows)
    n_cols   = len(rows[0])
    total_ins = n_rows + blank_rows
    ws.insert_rows(start_row, total_ins)

    # write values & (optional) styles
    for r_off, src_row in enumerate(rows):
        dest_r = start_row + r_off
        for c_off, src_cell in enumerate(src_row):
            dest_c = start_col + c_off
            tgt = ws.cell(dest_r, dest_c, value=src_cell.value)
            if copy_style:
                tgt.font          = src_cell.font
                tgt.fill          = src_cell.fill
                tgt.border        = src_cell.border
                tgt.alignment     = src_cell.alignment
                tgt.number_format = src_cell.number_format

    # re‑create merged ranges (offset for new position)
    for cr in merges:
        a, b, c, d = cr.bounds              # original bounds
        ws.merge_cells(
            start_row=start_row + (b - cr.min_row),
            start_column=start_col + (a - cr.min_col),
            end_row=start_row + (d - cr.min_row),
            end_column=start_col + (c - cr.min_col),
        )


def insert_template_block(
    src_path: str | Path,
    src_sheet: str,
    src_range: str,
    tgt_path: str | Path,
    out_path: str | Path | None = None,
    blank_rows: int = 2,
    copy_style: bool = True,
):
    """High‑level helper tying everything together."""
    src_wb = load_workbook(src_path, data_only=True)
    src_ws = src_wb[src_sheet]

    rows, merges = copy_block(src_ws, src_range)

    tgt_wb = load_workbook(tgt_path)
    for ws in tgt_wb.worksheets:
        paste_block(ws, rows, merges, blank_rows, copy_style=copy_style)

    save_to = out_path or tgt_path
    tgt_wb.save(save_to)
    print(f"Template inserted into every sheet → {save_to}")


# ---------- example usage ----------

insert_template_block(
    src_path  = "template.xlsx",   # workbook 1
    src_sheet = "HeaderSheet",     # sheet holding the block
    src_range = "A1:L4",           # block to copy
    tgt_path  = "report.xlsx",     # workbook 2 (each sheet will get the block)
    out_path  = "report_with_header.xlsx",  # leave None to overwrite
    blank_rows = 3,                # add 3 empty rows after the header
)
```
