```vba
Option Explicit

Sub RunPythonDemo()

    ' === 1. EDIT THESE THREE LINES ======================================
    Const PY_EXE   As String = "C:\Program Files\Python\Python312\python.exe"
    Const PY_FILE  As String = "C:\Users\Noah Taylor\Demo Scripts\simplified_demo.py"
    Const LOG_FILE As String = Environ$("TEMP") & "\demo_log.txt"
    ' ====================================================================

    Dim cmd As String, sh As Object

    ' Build a single command that    (a) keeps the console visible,
    '                                (b) runs your script,
    '                                (c) writes output to log,
    '                                (d) closes the window when done.
    cmd = "cmd.exe /k " & _
          """" & PY_EXE & """ " & _
          """" & PY_FILE & """ " & _
          "> """ & LOG_FILE & """ 2>&1 & type """ & LOG_FILE & """ & exit"

    ' 1️⃣  RUN visibly so the analyst sees real‑time prints.
    Shell cmd, vbNormalFocus

    ' 2️⃣  AFTER the window closes, read the captured text and drop it in D4.
    With ThisWorkbook.Sheets(1)        ' first sheet = the one with the button
        .Range("D4").Value = ReadAllText(LOG_FILE)
        .Range("D4").WrapText = False   ' optional – prevents tall cells
    End With

End Sub

' Helper to read entire text file into a string
Private Function ReadAllText(f As String) As String
    Dim ff As Integer: ff = FreeFile
    Dim txt As String
    Open f For Input As #ff
        txt = Input$(LOF(ff), ff)
    Close #ff
    ReadAllText = txt
End Function
```
