```vba
Option Explicit
Sub RunSimplifiedPython()

    Const PYTHON_EXE  As String = "C:\Program Files\Python311\python.exe"
    Const SCRIPT_FILE As String = "C:\My Scripts\demo_script.py"

    Dim tempOut As String, pyCmd As String, fullCmd As String
    Dim shell As Object, fso As Object, ts As Object, txt As String

    tempOut = Environ$("TEMP") & "\py_output.txt"

    ' ──► 1. Build only the Python command first, fully quoted
    pyCmd = """" & PYTHON_EXE & """" & " " & """" & SCRIPT_FILE & """"

    ' ──► 2. Wrap that in a CMD line that captures output,
    '        shows it, then pauses so you can read errors
    fullCmd = "cmd /k " & pyCmd & _
              " > """ & tempOut & """ 2>&1" & _
              " & type """ & tempOut & """" & _
              " & pause"

    ' ──► 3. (Optional) Print the exact command to the Immediate pane
    Debug.Print fullCmd

    Set shell = CreateObject("WScript.Shell")
    shell.Run fullCmd, 1, True          ' 1 = normal window; wait = True

    ' ──► 4. Pull the text back into Excel
    Set fso = CreateObject("Scripting.FileSystemObject")
    If fso.FileExists(tempOut) Then
        Set ts = fso.OpenTextFile(tempOut, 1)
        txt = ts.ReadAll
        ts.Close
        With ThisWorkbook.Sheets("Sheet1").Range("D4")
            .Value = txt
            .WrapText = True
        End With
        fso.DeleteFile tempOut, True     ' comment while debugging if you like
    End If
End Sub
```
