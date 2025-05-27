```python
"""
Copy a rectangular block from one workbook into the top of **every** sheet in
another workbook, while keeping

• cell values
• basic styles (font, fill, border, alignment, number‑format)
• merged cells that fall inside the block
• the row heights of the copied block
• a thick black border around the pasted rectangle

(openpyxl ≥ 3.1 tested)
"""

from pathlib import Path
from copy import copy
from typing import List, Tuple

from openpyxl import load_workbook
from openpyxl.utils import range_boundaries
from openpyxl.worksheet.worksheet import Worksheet
from openpyxl.worksheet.cell_range import CellRange
from openpyxl.styles import Border, Side


# ───────────────────────── helpers ───────────────────────── #

def _copy_block(src_ws: Worksheet, src_range: str) -> Tuple[
    List[List],                # rows of Cell objects
    List[CellRange],           # merged ranges inside the block
    List[float | None],        # row heights
    int, int                   # (min_row, min_col) of the block
]:
    """Slice `src_range` out of `src_ws` once; return everything needed."""
    min_col, min_row, max_col, max_row = range_boundaries(src_range)

    rows: List[List] = [
        [cell for cell in row]
        for row in src_ws.iter_rows(
            min_row=min_row, max_row=max_row,
            min_col=min_col, max_col=max_col,
        )
    ]

    merges = [
        cr for cr in src_ws.merged_cells.ranges
        if cr.min_col >= min_col and cr.max_col <= max_col
        and cr.min_row >= min_row and cr.max_row <= max_row
    ]

    row_heights = [
        (src_ws.row_dimensions[r].height
         if r in src_ws.row_dimensions and
            src_ws.row_dimensions[r].height is not None else None)
        for r in range(min_row, max_row + 1)
    ]

    return rows, merges, row_heights, min_row, min_col


def _apply_thick_border(
    ws: Worksheet,
    start_row: int,
    start_col: int,
    n_rows: int,
    n_cols: int,
    colour: str = "000000",
) -> None:
    """Draw a thick border around an `n_rows × n_cols` rectangle."""
    thick = Side(style="thick", color=colour)

    for r_off in range(n_rows):
        for c_off in range(n_cols):
            r = start_row + r_off
            c = start_col + c_off
            cell = ws.cell(r, c)

            top    = thick if r_off == 0 else cell.border.top
            bottom = thick if r_off == n_rows - 1 else cell.border.bottom
            left   = thick if c_off == 0 else cell.border.left
            right  = thick if c_off == n_cols - 1 else cell.border.right

            cell.border = Border(top=top, bottom=bottom, left=left, right=right)


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
    """Insert the template block at (start_row,start_col) with spacing below."""
    n_rows = len(rows)
    n_cols = len(rows[0]) if rows else 0

    # 1️⃣  Make space
    ws.insert_rows(start_row, n_rows + blank_rows)

    # 2️⃣  Copy cells and row heights
    for r_off, (src_row, height) in enumerate(zip(rows, row_heights)):
        dest_r = start_row + r_off

        if height is not None:
            ws.row_dimensions[dest_r].height = height  # customHeight is inferred

        for c_off, src_cell in enumerate(src_row):
            dest_c = start_col + c_off
            tgt = ws.cell(dest_r, dest_c, value=src_cell.value)

            if copy_style:
                tgt.font          = copy(src_cell.font)
                tgt.fill          = copy(src_cell.fill)
                tgt.border        = copy(src_cell.border)
                tgt.alignment     = copy(src_cell.alignment)
                tgt.number_format = src_cell.number_format

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

    # 4️⃣  Draw thick border around the pasted rectangle
    _apply_thick_border(ws, start_row, start_col, n_rows, n_cols)


# ───────────────────── public wrapper ───────────────────── #

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
    Copy `src_range` (src_path/src_sheet) into the top of every sheet
    in `tgt_path`, add `blank_rows` empty rows below, and save.
    """
    src_wb = load_workbook(src_path, data_only=True)
    rows, merges, row_heights, src_min_row, src_min_col = _copy_block(
        src_wb[src_sheet], src_range
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

    tgt_wb.save(out_path or tgt_path)
    print(f"✅  Header inserted into every sheet → {out_path or tgt_path}")


# ───────────────────────── example run ───────────────────────── #
if __name__ == "__main__":
    insert_template_block(
        src_path="template.xlsx",      # workbook that holds the header
        src_sheet="Header",            # sheet with the header block
        src_range="A1:L4",             # range to copy (adjust!)
        tgt_path="report.xlsx",        # workbook to receive the header
        out_path="report_with_header.xlsx",  # overwrite if None
        blank_rows=3,                  # spacer rows after header
        copy_style=True,
    )
```
