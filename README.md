```vba
' Standard Module, e.g., Module1
Option Explicit

Private Const PY_EXE   As String = "C:\Python39\python.exe"
Private Const PY_FILE  As String = "C:\Demos\simplified_script.py"

Sub RunPythonAndDisplay()
    Dim sh As Object, cmd As String, exec As Object, txt As String
    
    ' 1. Build command
    cmd = """" & PY_EXE & """" & " " & """" & PY_FILE & """"
    
    ' 2. Launch & capture StdOut (silent window)
    Set sh = CreateObject("WScript.Shell")                            ' :contentReference[oaicite:3]{index=3}
    Set exec = sh.Exec("cmd /c " & cmd)                               ' /c auto‑closes the hidden cmd

    Do While exec.Status = 0: DoEvents: Loop                          ' wait ‑ status 0 = running
    txt = exec.StdOut.ReadAll                                         ' get all output lines at once
    
    ' 3. Drop result into B2 of this sheet
    With ThisWorkbook.Worksheets("Demo")
        .Range("B2").Value = txt                                      ' :contentReference[oaicite:4]{index=4}
    End With
End Sub
```
