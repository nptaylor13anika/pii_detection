```vba
Option Explicit
Sub RunSimplifiedPython()

    '── 1.  EDIT just these two lines ──────────────────────────
    Const PYTHON_EXE  As String = "C:\Program Files\Python311\python.exe"
    Const SCRIPT_FILE As String = "C:\My Scripts\demo_script.py"
    '────────────────────────────────────────────────────────────

    Dim tempOut As String, pyCmd As String, fullCmd As String
    Dim sh As Object, fso As Object, ts As Object, txt As String
    
    tempOut = Environ$("TEMP") & "\py_output.txt"

    '► Wrap each real path in its own quotes
    pyCmd = """" & PYTHON_EXE & """" & " " & """" & SCRIPT_FILE & """"

    '► Build one compound command for cmd.exe
    fullCmd = "cmd /k " & _
              pyCmd & " > """ & tempOut & """ 2>&1" & _
              " & type """ & tempOut & """" & _
              " & pause"            ' pause leaves window open for errors

    Debug.Print fullCmd            '← lets you copy‑paste to a manual CMD

    Set sh = CreateObject("WScript.Shell")
    sh.Run fullCmd, 1, True        ' 1 = normal window, wait=True

    '► Pull console text into Excel
    Set fso = CreateObject("Scripting.FileSystemObject")
    If fso.FileExists(tempOut) Then
        Set ts = fso.OpenTextFile(tempOut, 1)
        txt = ts.ReadAll: ts.Close
        With ThisWorkbook.Sheets("Sheet1").Range("D4")
            .Value = txt
            .WrapText = True
        End With
        fso.DeleteFile tempOut, True
    Else
        MsgBox "Temp output not created—check console for Python errors.", vbExclamation
    End If
End Sub
```
