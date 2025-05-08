```vba
' === Module1 ===
Option Explicit

Private Const PY_EXE  As String = "C:\Path With Spaces\Python\python.exe"
Private Const PY_FILE As String = "C:\Another Path\Data Scripts\simplified_script.py"

Sub RunPythonAndDisplay()
    Dim sh   As Object
    Dim exec As Object
    Dim cmd  As String
    Dim txt  As String
    
    Set sh = CreateObject("WScript.Shell")
    
    '--- 1. Build and launch -----------------------------------------------
    cmd = """" & PY_EXE & """ """ & PY_FILE & """"      'A: direct call
    Set exec = sh.Exec(cmd)                             ':contentReference[oaicite:4]{index=4}
    
    '--- 2. Wait for completion (MSDN loop) --------------------------------
    Do While exec.Status = 0                            '0 = running
        DoEvents                                        'keeps Excel responsive
    Loop                                                ':contentReference[oaicite:5]{index=5}
    
    '--- 3. Read both StdOut and StdErr -------------------------------
    txt = exec.StdOut.ReadAll & exec.StdErr.ReadAll     'catch silent errors
    
    If Len(txt) = 0 Then txt = "(no output ‑ check paths or script errors)"
    
    '--- 4. Show in sheet ---------------------------------------------------
    With ThisWorkbook.Worksheets("Demo")
        .Range("B2").Value = txt                        ':contentReference[oaicite:6]{index=6}
    End With
End Sub

```
