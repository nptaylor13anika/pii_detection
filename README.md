```vba
Option Explicit

Sub RunSimplifiedPython()

    Dim shell As Object
    Dim pythonExe As String, scriptPath As String, tempOut As String
    Dim cmd As String, fso As Object, ts As Object, outputText As String

    ' ===== EDIT THESE TWO LINES =====
    pythonExe = "C:\Python311\python.exe"          ' 1) full path to python.exe
    scriptPath = ThisWorkbook.Path & "\demo_script.py" ' 2) your script name
    ' =================================

    tempOut = Environ$("TEMP") & "\py_output.txt"

    ' Build a command that
    '   • opens a visible cmd window (style 1)
    '   • runs the script, redirecting all output to tempOut
    '   • echoes the file so you see the lines scroll in that same window
    cmd = "cmd /c """ & pythonExe & """ """ & scriptPath & _
          """ > """ & tempOut & """ 2>&1 & type """ & tempOut & """"

    Set shell = CreateObject("WScript.Shell")

    ' Run:  windowstyle = 1 (normal window), wait = True (macro pauses)
    shell.Run cmd, 1, True

    ' ===== Pull the output back into Excel =====
    Set fso = CreateObject("Scripting.FileSystemObject")
    If fso.FileExists(tempOut) Then
        Set ts = fso.OpenTextFile(tempOut, 1)
        outputText = ts.ReadAll
        ts.Close
        ' Dump into D4 of the sheet that owns the button
        With ThisWorkbook.Sheets("Sheet1").Range("D4")
            .Value = outputText
            .WrapText = True
        End With
        ' Optional clean‑up
        fso.DeleteFile tempOut, True
    End If

End Sub
```
