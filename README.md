```python
"""
Copy a template block from one workbook into the top of **every** sheet in
another workbook, preserving:

• Cell values
• Basic cell styles (font, fill, border, alignment, number‑format)
• Merged cells that live inside the copied block
• *Row heights* of the template rows
• Row heights of the original target sheet (openpyxl shifts them for us)

Usage (adjust the paths/range to suit):

    insert_template_block(
        src_path="template.xlsx",
        src_sheet="Header",
        src_range="A1:L4",
        tgt_path="report.xlsx",
        out_path="report_with_header.xlsx",
        blank_rows=3,          # empty rows after the header
        copy_style=True,       # include formatting
    )
"""

from pathlib import Path
from copy import copy
from typing import List, Tuple

from openpyxl import load_workbook
from openpyxl.utils import range_boundaries
from openpyxl.worksheet.worksheet import Worksheet
from openpyxl.worksheet.cell_range import CellRange


# --------------------------------------------------------------------------- #
#  Helpers                                                                    #
# --------------------------------------------------------------------------- #

def _copy_block(src_ws: Worksheet, src_range: str) -> Tuple[
    List[List], List[CellRange], List[float | None], int, int
]:
    """
    Slice `src_range` (e.g. "A1:L4") out of `src_ws` **once** and return:

        rows         → 2‑D list of *Cell* objects (for style/value access)
        merges       → list of CellRange objects fully inside the range
        row_heights  → list of floats / None for each template row
        min_row, min_col → top‑left coordinate of the block (for merge offset)

    Everything downstream can run on these Python objects without re‑reading
    the workbook.
    """
    min_col, min_row, max_col, max_row = range_boundaries(src_range)

    # --- capture the cells ---
    rows: List[List] = []
    for row in src_ws.iter_rows(
        min_row=min_row, max_row=max_row,
        min_col=min_col, max_col=max_col,
    ):
        rows.append([cell for cell in row])          # keep *cell* objects

    # --- capture merged ranges that live wholly inside the block ---
    merges = [
        cr for cr in src_ws.merged_cells.ranges
        if cr.min_col >= min_col and cr.max_col <= max_col
        and cr.min_row >= min_row and cr.max_row <= max_row
    ]

    # --- capture row heights ---
    row_heights: List[float | None] = []
    for r in range(min_row, max_row + 1):
        dim = src_ws.row_dimensions.get(r)
        row_heights.append(dim.height if (dim and dim.height is not None) else None)

    return rows, merges, row_heights, min_row, min_col


def _paste_block(
    ws: Worksheet,
    *,
    rows: List[List],
    merges: List[CellRange],
    row_heights: List[float | None],
    src_origin: Tuple[int, int],
    blank_rows: int,
    start_row: int,
    start_col: int,
    copy_style: bool = True,
) -> None:
    """
    Insert the template block into `ws` beginning at (start_row, start_col).

    • Inserts `len(rows) + blank_rows` rows so existing data shifts downward.
    • Writes cell values (and styles if `copy_style=True`).
    • Copies row heights for the template rows.
    • Re‑creates merged cells inside the new block.
    """
    n_rows = len(rows)
    total_insert = n_rows + blank_rows

    # 1️⃣  Shift everything down to make room
    ws.insert_rows(start_row, total_insert)

    # 2️⃣  Write the template rows and copy row heights / styles
    for r_off, src_row in enumerate(rows):
        dest_r = start_row + r_off

        # 2a. Row height
        if row_heights[r_off] is not None:
            rd = ws.row_dimensions[dest_r]
            rd.height = row_heights[r_off]
            rd.customHeight = True

        # 2b. Cell values (+ styles)
        for c_off, src_cell in enumerate(src_row):
            dest_c = start_col + c_off
            tgt = ws.cell(dest_r, dest_c, value=src_cell.value)

            if copy_style:
                tgt.font          = copy(src_cell.font)
                tgt.fill          = copy(src_cell.fill)
                tgt.border        = copy(src_cell.border)
                tgt.alignment     = copy(src_cell.alignment)
                tgt.number_format = src_cell.number_format  # already str

    # 3️⃣  Re‑create merged cells
    src_min_row, src_min_col = src_origin
    for cr in merges:
        row_shift = cr.min_row - src_min_row
        col_shift = cr.min_col - src_min_col

        ws.merge_cells(
            start_row=start_row + row_shift,
            start_column=start_col + col_shift,
            end_row=start_row + row_shift + (cr.max_row - cr.min_row),
            end_column=start_col + col_shift + (cr.max_col - cr.min_col),
        )


# --------------------------------------------------------------------------- #
#  Public wrapper                                                             #
# --------------------------------------------------------------------------- #

def insert_template_block(
    *,
    src_path: str | Path,
    src_sheet: str,
    src_range: str,
    tgt_path: str | Path,
    out_path: str | Path | None = None,
    blank_rows: int = 2,
    copy_style: bool = True,
    start_row: int = 1,
    start_col: int = 1,
) -> None:
    """
    High‑level convenience wrapper.

    • `src_path`, `src_sheet`, `src_range` define the template block.
    • The block is inserted at `start_row`, `start_col` (default A1)
      on *every* sheet of `tgt_path`.
    • `blank_rows` empty rows are left beneath the block.
    • The result is saved to `out_path` (or overwrites `tgt_path` if None).
    """
    src_wb = load_workbook(src_path, data_only=True)
    src_ws = src_wb[src_sheet]

    rows, merges, row_heights, src_min_row, src_min_col = _copy_block(
        src_ws, src_range
    )

    tgt_wb = load_workbook(tgt_path)

    for ws in tgt_wb.worksheets:
        _paste_block(
            ws,
            rows=rows,
            merges=merges,
            row_heights=row_heights,
            src_origin=(src_min_row, src_min_col),
            blank_rows=blank_rows,
            start_row=start_row,
            start_col=start_col,
            copy_style=copy_style,
        )

    save_to = out_path or tgt_path
    tgt_wb.save(save_to)
    print(f"✅  Header inserted into every sheet → {save_to}")


# --------------------------------------------------------------------------- #
#  Example run (remove or adapt in your own script)                           #
# --------------------------------------------------------------------------- #
if __name__ == "__main__":
    insert_template_block(
        src_path="template.xlsx",
        src_sheet="Header",
        src_range="A1:L4",
        tgt_path="report.xlsx",
        out_path="report_with_header.xlsx",
        blank_rows=3,
        copy_style=True,
    )
```
