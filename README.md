```vba
' === Module1 ===
Option Explicit

Private Const PY_FILE  As String = "C:\Path With Spaces\simplified_script.py"
Private Const LOG_FILE As String = "C:\Temp\py_run_log.txt"

Sub RunPythonVisibleAndCapture()
    Dim wsh As Object, cmd As String, fso As Object, txt As String
    
    ' 1) Build PowerShell command:
    '    -NoExit keeps window open, -NoProfile starts fast,
    '    python -u streams unbuffered, Tee-Object mirrors to file.
    cmd = "powershell.exe -NoProfile -NoExit -Command " & _
          """python -u """"& '" & PY_FILE & "' & """" 2>&1 " & _
          "| Tee-Object -FilePath '" & LOG_FILE & "'"""
    
    ' 2) Launch visible & wait
    Set wsh = CreateObject("WScript.Shell")
    wsh.Run cmd, 1, True          '1 = show window, True = wait :contentReference[oaicite:4]{index=4}
    
    ' 3) Pull log into D4
    Set fso = CreateObject("Scripting.FileSystemObject")
    txt = fso.OpenTextFile(LOG_FILE, 1).ReadAll         :contentReference[oaicite:5]{index=5}
    ThisWorkbook.Worksheets("Demo").Range("D4").Value = txt
End Sub
```
