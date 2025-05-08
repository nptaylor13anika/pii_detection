```vba
Sub RunPythonShowOutput()
    Dim pythonExe   As String
    Dim scriptPath  As String
    Dim summaryFile As String
    Dim wsh         As Object
    Dim fileNum     As Integer
    Dim outputText  As String
    
    ' 1. Configure paths (wrap in triple quotes for spaces) 
    pythonExe   = """" & "C:\Program Files\Python39\python.exe" & """"    ' :contentReference[oaicite:7]{index=7}
    scriptPath  = """" & "C:\Path With Spaces\script.py" & """"           ' :contentReference[oaicite:8]{index=8}
    summaryFile = "C:\Path With Spaces\output_summary.txt"
    
    ' 2. Delete old summary, if any
    If Dir(summaryFile) <> "" Then Kill summaryFile                        ' :contentReference[oaicite:9]{index=9}
    
    ' 3. Run Python in visible console, wait until it finishes
    Set wsh = CreateObject("WScript.Shell")
    ' The Run method signature: Run(command As String, windowStyle As Integer, waitOnReturn As Boolean)
    ' vbNormalFocus = 1; True waits for completion :contentReference[oaicite:10]{index=10}
    wsh.Run pythonExe & " " & scriptPath, 1, True                        ' :contentReference[oaicite:11]{index=11}
    
    ' 4. Read the entire summary file into a string
    fileNum = FreeFile
    Open summaryFile For Input As #fileNum
    ' Input$(LOF(fileNum), fileNum) reads all characters at once :contentReference[oaicite:12]{index=12}
    outputText = Input$(LOF(fileNum), fileNum)
    Close #fileNum
    
    ' 5. Display the text in cell D4 on Sheet1
    ThisWorkbook.Sheets("Sheet1").Range("D4").Value = outputText
End Sub
```
