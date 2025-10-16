# TinyEXE
A Win32 GUI app in under 20kb

This is a simple Hello World app showing 2 things:

<img width="377" height="292" alt="image" src="https://github.com/user-attachments/assets/742a94b3-9777-4ec7-8525-88bbee220807" />

- A few people don't like that tB, unlike VB6, doesn't have a runtime DLL dependency; the Forms engine etc is built right into the exe. This makes the exe itself larger, even though a VB6 exe + msvbvm60.dll is roughly the same size as a tB exe. Why this 1.4MB difference matters with modern drive sizes and internet speeds, and most languages/toolchains starting even bigger, I don't know, but it's come up. This app demonstrates that it's not simply a twinBASIC inefficiency, and tB exes can be quite tiny: This app compiles to under 20kb for both 32 and 64bit despite having a GUI, dozens of APIs + constants, embedded version info, and 90 lines of code.
- It accomplishes this using twinBASIC's ability to easily set your own entry point. Previously demonstrated and originally added for drivers, it's also applicable to regular GUI apps, just don't use Subsystem: Native. This forgoes any of the built in tools, making it even smaller than a non-GUI console app. But this means doing absolutely everything yourself. This presents an opportunity to learn how a Windows exe really works, starting with an actual entry point that's hidden and automatic even in a typical C app... the `wWinMain` or similar entry point isn't the *real* entry point, it's called by the real one, in C usually `wWinMainCRTStartup`. This is why if you try using `wWinMain` as the entry point in tB you'll get garbage for the arguments like command line... a real entry point has no arguments and retrieves those values itself, then calls `wWinMain`.\
twinBASIC allows overriding this real entry point... if you wanted all the default background setup, that's what the normal Startup Object setting is for. The app continues from there to register its own custom top level window class, create it and some child controls, then run its own message pump: the core of a Win32 GUI app that listens for input and other messages in a loop that runs until the window is destroyed, triggering the exit process, which must be handled carefully to avoid the process lingering on in the background despite being finished. This is very similar to subclassing.

This is the full app, requiring a Windows Development Library reference, (though you'll want the actual .twinproj as there's a lot of Project Settings changes to make this work):

```vb
Module MainModule
    
Private m_hwnd As LongPtr, hEdit As LongPtr, hBtn As LongPtr

Private Const wndClass = "TBSmallExeMainWndCls"
Private Const wndName = "Tiny twinBASIC EXE"


'This is a very minimal implementation of the real entry point for an exe.
'In C/C++ this would be wWinMainCRTStartup or such; hidden and inserted by 
'the linker, which calls wWinMain etc. Normally this does more, like calls
'global constructors, looks up the 2 arguments defaulted below, etc.
'VB6 and twinBASIC do even more.
Public Function RealMain() As Long
    ExitProcess(wWinMain(GetModuleHandle(0), 0, GetCommandLineW(), SW_SHOW))
End Function


Public Function wWinMain(ByVal hInstance As LongPtr, ByVal hPrevInstance As LongPtr, _
                         ByVal pCmdLine As LongPtr, ByVal nCmdShow As Long) As Long
    
    'CoInitializeEx(0, COINIT_APARTMENTTHREADED) 'Uncomment to use anything with COM
    
    Dim hr As Long = S_OK
    
    Dim wcex As WNDCLASSEX
    wcex.cbSize = LenB(wcex)
    wcex.style = CS_HREDRAW Or CS_VREDRAW Or CS_DBLCLKS
    wcex.lpfnWndProc = AddressOf WindowProc
    wcex.hInstance = hInstance
    wcex.hCursor = LoadCursor(0, IDC_ARROW)
    wcex.hbrBackground = GetStockObject(LTGRAY_BRUSH)
    wcex.lpszMenuName = 1
    wcex.lpszClassName = StrPtr(wndClass)
    
    RegisterClassEx(wcex)
    
     m_hwnd = CreateWindowExW(0, StrPtr(wndClass), StrPtr(wndName), WS_OVERLAPPEDWINDOW, _
                            CW_USEDEFAULT, CW_USEDEFAULT, 400, 300, 0, 0, hInstance, ByVal 0)
        hEdit = CreateWindowExW(0, StrPtr(WC_EDITW), 0, WS_CHILD Or WS_VISIBLE Or WS_BORDER Or ES_MULTILINE, _
                                 50, 10, 285, 200, m_hwnd, 0, hInstance, ByVal 0)
        hBtn = CreateWindowExW(0, StrPtr(WC_BUTTONW), 0, WS_CHILD Or WS_VISIBLE, _
                                125, 215, 125, 40, m_hwnd, 101, hInstance, ByVal 0)
        
        SendMessage hEdit, WM_SETFONT, GetStockObject(DEFAULT_GUI_FONT), ByVal 1
        SendMessage hBtn, WM_SETFONT, GetStockObject(DEFAULT_GUI_FONT), ByVal 1
        
        SetWindowTextW hEdit, StrPtr("Hello World!")
        SetWindowTextW hBtn, StrPtr("Click Me")
        ShowWindow m_hwnd, SW_SHOW
        UpdateWindow m_hwnd
        
        Dim tMSG As MSG
        While GetMessage(tMSG, 0, 0, 0)
            TranslateMessage tMSG
            DispatchMessage tMSG
        Wend
        
        UnregisterClassW StrPtr(wndClass), hInstance
        
        'CoUninitialize()
End Function

Private Function WindowProc(ByVal hWnd As LongPtr, ByVal uMsg As Long, ByVal wParam As LongPtr, ByVal lParam As LongPtr) As LongPtr
    Dim result As LongPtr
    
    Select Case uMsg
        Case WM_COMMAND
            If LOWORD(wParam) = 101 Then
                Dim cch As Long = GetWindowTextLengthW(hEdit)
                If cch > 0 Then
                    Dim buf As String = String$(cch + 1, 0)
                    GetWindowTextW(hEdit, StrPtr(buf), cch + 1)
                    MessageBoxW m_hwnd, StrPtr(buf), StrPtr("You typed..."), MB_OK
                End If
            End If
            
        Case WM_CLOSE
            If m_hwnd Then
                DestroyWindow m_hwnd
                m_hwnd = 0
            End If
            
        Case WM_DESTROY
            PostQuitMessage(0)

        Case Else
            result = DefWindowProc(hWnd, uMsg, wParam, lParam)
    End Select
    
    WindowProc = result
End Function

End Module
```
