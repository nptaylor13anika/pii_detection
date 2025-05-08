```vba
Option Explicit
Sub RunSimplifiedPython()

    '*** 1.  EDIT ONLY THESE TWO LINES *********************************
    Const PYTHON_EXE  As String = "C:\Program Files\Python311\python.exe"
    Const SCRIPT_FILE As String = "C:\My Scripts\demo_script.py"
    '*******************************************************************

    Dim tempOut As String, pyCmd As String, fullCmd As String
    Dim sh As Object, fso As Object, ts As Object, txt As String

    tempOut = Environ$("TEMP") & "\py_output.txt"

    '--- Build the Python part (every path individually quoted) --------
    pyCmd = """" & PYTHON_EXE & """" & " " & """" & SCRIPT_FILE & """"

    '--- Wrap that for cmd.exe: capture → echo → pause -----------------
    fullCmd = "cmd /k " & pyCmd & _
              " > """ & tempOut & """ 2>&1" & _
              " & type """ & tempOut & """" & _
              " & pause"

    Debug.Print fullCmd      '← **Copy this line into a real CMD window**

    Set sh = CreateObject("WScript.Shell")
    sh.Run fullCmd, 1, True  '1 = normal window, wait=True (macro blocks)

    '--- Pull the console text back into Excel ------------------------
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
        MsgBox "Python did not create the log file—check the console for errors.", vbExclamation
    End If
End Sub
```
