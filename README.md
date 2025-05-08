```vba
Option Explicit

Sub RunSimplifiedPython()

    Const PYTHON_EXE As String = "C:\Program Files\Python311\python.exe"   ' <‑‑ adjust
    Const SCRIPT_FILE As String = "C:\My Scripts\demo_script.py"          ' <‑‑ adjust
    Dim tempOut As String, cmd As String, shell As Object
    Dim fso As Object, ts As Object, outputText As String

    tempOut = Environ$("TEMP") & "\py_output.txt"

    ' ---- Build a fully‑quoted command string ----
    cmd = "cmd /k " & _
          """" & PYTHON_EXE & """" & " " & _
          """" & SCRIPT_FILE & """" & " " & _
          "> " & """" & tempOut & """" & " 2>&1 " & _
          "& type " & """" & tempOut & """" & _
          " & pause"                   ' <-- remove ‘& pause’ after it works

    Set shell = CreateObject("WScript.Shell")
    shell.Run cmd, 1, True             ' windowstyle 1 = normal, wait = True

    ' ---- Read the output back into Excel ----
    Set fso = CreateObject("Scripting.FileSystemObject")
    If fso.FileExists(tempOut) Then
        Set ts = fso.OpenTextFile(tempOut, 1)
        outputText = ts.ReadAll
        ts.Close
        With ThisWorkbook.Sheets("Sheet1").Range("D4")
            .Value = outputText
            .WrapText = True
        End With
        ' comment out the next line while debugging if you want to inspect the file
        fso.DeleteFile tempOut, True
    Else
        MsgBox "Python did not create the output file." & vbCrLf & _
               "Check the console window for error messages.", vbExclamation
    End If

End Sub
```
