```python
from pathlib import Path
from copy import copy
from typing import List, Tuple

from openpyxl import load_workbook
from openpyxl.utils import range_boundaries
from openpyxl.worksheet.worksheet import Worksheet
from openpyxl.worksheet.cell_range import CellRange


# ───────────────────────── helpers ───────────────────────── #

def _copy_block(src_ws: Worksheet, src_range: str) -> Tuple[
    List[List], List[CellRange], int, int
]:
    """Return the block’s cells, its internal merged ranges, and its origin."""
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

    return rows, merges, min_row, min_col


def _paste_block(
    ws: Worksheet,
    *,
    rows: List[List],
    merges: List[CellRange],
    src_origin: Tuple[int, int],
    blank_rows: int,
    start_row: int,
    start_col: int,
    copy_style: bool = True,
) -> None:
    """Insert `rows` at (start_row,start_col) and leave `blank_rows` beneath."""
    ws.insert_rows(start_row, len(rows) + blank_rows)

    # copy the cells
    for r_off, src_row in enumerate(rows):
        dest_r = start_row + r_off
        for c_off, src_cell in enumerate(src_row):
            dest_c = start_col + c_off
            tgt = ws.cell(dest_r, dest_c, value=src_cell.value)

            if copy_style:
                tgt.font          = copy(src_cell.font)
                tgt.fill          = copy(src_cell.fill)
                tgt.border        = copy(src_cell.border)
                tgt.alignment     = copy(src_cell.alignment)
                tgt.number_format = src_cell.number_format

    # recreate merged cells
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
    """Copy `src_range` (src_path/src_sheet) into the top of every sheet."""
    src_wb = load_workbook(src_path, data_only=True)
    rows, merges, src_min_row, src_min_col = _copy_block(
        src_wb[src_sheet], src_range
    )

    tgt_wb = load_workbook(tgt_path)
    for ws in tgt_wb.worksheets:
        _paste_block(
            ws,
            rows=rows,
            merges=merges,
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
        src_path="template.xlsx",      # workbook holding header
        src_sheet="Header",            # sheet with header block
        src_range="A1:L4",             # block to copy
        tgt_path="report.xlsx",        # workbook to receive header
        out_path="report_with_header.xlsx",
        blank_rows=3,                  # spacer rows after header
        copy_style=True,
    )
```
