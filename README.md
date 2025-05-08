```vba
'=== Module1 ================================================================
Option Explicit

Private Const PY_EXE   As String = "C:\Path With Spaces\python.exe"
Private Const PY_FILE  As String = "C:\Path With Spaces\simplified_script.py"
Private Const LOG_FILE As String = "C:\Temp\py_demo_log.txt"

'––– WinAPI declarations –––––––––––––––––––––––––––––––––––––––––––––––––––
Private Declare PtrSafe Function OpenProcess Lib "kernel32" ( _
        ByVal dwDesiredAccess As Long, ByVal bInheritHandle As Long, _
        ByVal dwProcessId As Long) As LongPtr
Private Declare PtrSafe Function WaitForSingleObject Lib "kernel32" ( _
        ByVal hHandle As LongPtr, ByVal dwMilliseconds As Long) As Long
Private Declare PtrSafe Function CloseHandle Lib "kernel32" ( _
        ByVal hObject As LongPtr) As Long

Sub RunPythonVisibleAndCapture()
    Dim cmd As String, procID As Long, hProc As LongPtr
    Dim FSO As Object, txt As String
    Const INFINITE As Long = &HFFFFFFFF
    
    '1) Build PowerShell + Tee command (quotes survive spaces)
    cmd = "powershell.exe -NoExit -Command ""python -u " & _
          "'" & PY_FILE & "' 2>&1 | Tee-Object -FilePath '" & LOG_FILE & "'"""
    
    '2) Launch visible console and get process ID
    procID = Shell(cmd, vbNormalFocus)                    'visible window :contentReference[oaicite:5]{index=5}
    
    '3) Wait for window to close (user can read/scroll)   :contentReference[oaicite:6]{index=6}
    hProc = OpenProcess(&H100000, 0, procID)               'SYNCHRONIZE access
    Call WaitForSingleObject(hProc, INFINITE)
    Call CloseHandle(hProc)
    
    '4) Read the log file and paste into D4               :contentReference[oaicite:7]{index=7}
    Set FSO = CreateObject("Scripting.FileSystemObject")
    txt = FSO.OpenTextFile(LOG_FILE).ReadAll
    ThisWorkbook.Worksheets("Demo").Range("D4").Value = txt
End Sub
```
