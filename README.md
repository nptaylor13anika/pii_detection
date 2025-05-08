```vba
Sub RunPythonAndPopulateDict()
    Dim pythonExe  As String
    Dim scriptPath As String
    Dim shell      As Object
    Dim proc       As Object
    Dim allText    As String
    Dim dictText   As String
    Dim reBlock    As New RegExp
    Dim reKV       As New RegExp
    Dim blockMatch As MatchCollection
    Dim kvMatches  As MatchCollection
    Dim i          As Long
    
    ' --- 1. Configure your paths (wrap in quotes if they contain spaces) ---
    pythonExe  = """" & "C:\Path With Spaces\python.exe" & """"
    scriptPath = """" & "C:\Path With Spaces\script.py" & """"
    
    ' --- 2. Launch Python and grab everything it prints ---
    Set shell = CreateObject("WScript.Shell")
    Set proc  = shell.Exec(pythonExe & " " & scriptPath)
    Do While Not proc.StdOut.AtEndOfStream
        allText = allText & proc.StdOut.ReadLine & vbCrLf
    Loop
    
    ' --- 3. Extract only the JSON-dict block between your markers ---
    With reBlock
        .Pattern   = "<<RESULT_START>>(.*)<<RESULT_END>>"
        .Global    = False
        .MultiLine = True
    End With
    If reBlock.Test(allText) Then
        Set blockMatch = reBlock.Execute(allText)
        dictText = blockMatch(0).SubMatches(0)
    Else
        MsgBox "Could not find RESULT block in Python output.", vbExclamation
        Exit Sub
    End If
    
    ' --- 4. Find each "key":"value" pair inside that JSON string ---
    With reKV
        ' Captures "key":"value" blocks; handles escaped quotes if needed
        .Pattern   = """([^""]+)"":"?"?([^""]+)""?"
        .Global    = True
        .MultiLine = False
    End With
    Set kvMatches = reKV.Execute(dictText)
    
    ' --- 5. Dump into the "output" sheet, starting at row 1 (A=keys, B=values) ---
    With ThisWorkbook.Sheets("output")
        .Cells.ClearContents
        i = 1
        Dim m As Match
        For Each m In kvMatches
            .Cells(i, 1).Value = m.SubMatches(0)  ' the key
            .Cells(i, 2).Value = m.SubMatches(1)  ' the value
            i = i + 1
        Next m
    End With
End Sub
```
