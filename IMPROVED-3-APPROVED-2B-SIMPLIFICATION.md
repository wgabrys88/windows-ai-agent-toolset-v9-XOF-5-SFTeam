```python
import base64
import ctypes
import json
import os
import struct
import sys
import time
import urllib.request
import zlib
from ctypes import wintypes
from dataclasses import dataclass, field
from functools import wraps
from typing import Any, Optional

@dataclass(frozen=True)
class Config:
    lmstudio_endpoint: str = "http://localhost:1234/v1/chat/completions"
    lmstudio_model: str = "qwen3-vl-2b-instruct"
    lmstudio_timeout: int = 240
    lmstudio_temperature: float = 0.5
    
    planner_max_tokens: int = 1800
    executor_max_tokens: int = 1400
    
    screen_capture_w: int = 1536
    screen_capture_h: int = 864
    dump_dir: str = "dumps"
    max_steps: int = 50
    
    ui_settle_delay: float = 0.3
    turn_delay: float = 1.5
    char_input_delay: float = 0.01
    
    enable_loop_recovery: bool = True
    loop_recovery_cooldown: int = 3
    
    review_interval: int = 7
    memory_compression_threshold: int = 12

CFG = Config()

class ExecutionLogger:
    def __init__(self, dump_dir: str):
        self.dump_dir = dump_dir
        self.screenshots_dir = os.path.join(dump_dir, "screenshots")
        self.log_file = os.path.join(dump_dir, "execution_log.txt")
        os.makedirs(self.screenshots_dir, exist_ok=True)
        with open(self.log_file, "w", encoding="utf-8") as f:
            f.write("=" * 80 + "\n")
            f.write("EXECUTION LOG START\n")
            f.write("=" * 80 + "\n\n")
        self.api_call_counter = 0
    
    def log(self, message: str) -> None:
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(f"[{time.strftime('%H:%M:%S')}] {message}\n")
    
    def log_section(self, title: str) -> None:
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(f"\n{'=' * 80}\n{title}\n{'=' * 80}\n\n")
    
    def log_api_request(self, agent: str, payload: dict[str, Any]) -> None:
        self.api_call_counter += 1
        redacted = self._redact_payload(payload)
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(f"\n--- API REQUEST #{self.api_call_counter} ({agent}) ---\n")
            f.write(json.dumps(redacted, indent=2, ensure_ascii=False))
            f.write("\n")
    
    def log_api_response(self, agent: str, response: dict[str, Any]) -> None:
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(f"\n--- API RESPONSE #{self.api_call_counter} ({agent}) ---\n")
            f.write(json.dumps(response, indent=2, ensure_ascii=False))
            f.write("\n")
    
    def log_tool_execution(self, turn: int, tool_name: str, args: Any, result: str) -> None:
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(f"\n[TURN {turn}] TOOL: {tool_name}\n")
            f.write(f"Args: {args}\n")
            f.write(f"Result: {result}\n")
    
    def log_state_update(self, state_info: dict[str, Any]) -> None:
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(f"\nSTATE UPDATE:\n")
            f.write(json.dumps(state_info, indent=2, ensure_ascii=False))
            f.write("\n")
    
    def save_screenshot(self, png: bytes, turn: int) -> str:
        path = os.path.join(self.screenshots_dir, f"turn_{turn:04d}.png")
        with open(path, "wb") as f:
            f.write(png)
        self.log(f"Screenshot: {os.path.basename(path)}")
        return path
    
    def _redact_payload(self, payload: dict[str, Any]) -> dict[str, Any]:
        if isinstance(payload, dict):
            out = {}
            for k, v in payload.items():
                if k == "url" and isinstance(v, str) and v.startswith("data:image/png;base64,"):
                    b64 = v.split(",", 1)[1]
                    out[k] = f"<base64_png len={len(b64)}>"
                else:
                    out[k] = self._redact_payload(v)
            return out
        if isinstance(payload, list):
            return [self._redact_payload(x) for x in payload]
        return payload

LOGGER = ExecutionLogger(CFG.dump_dir)

def log_api_call(agent_name: str):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            payload = kwargs.get('payload') or (args[0] if args else None)
            if payload:
                LOGGER.log_api_request(agent_name, payload)
            result = func(*args, **kwargs)
            if isinstance(result, dict):
                LOGGER.log_api_response(agent_name, result)
            return result
        return wrapper
    return decorator

for attr in ["HCURSOR", "HICON", "HBITMAP", "HGDIOBJ", "HBRUSH", "HDC"]:
    if not hasattr(wintypes, attr):
        setattr(wintypes, attr, wintypes.HANDLE)
if not hasattr(wintypes, "ULONG_PTR"):
    wintypes.ULONG_PTR = ctypes.c_size_t

user32 = ctypes.WinDLL("user32", use_last_error=True)
gdi32 = ctypes.WinDLL("gdi32", use_last_error=True)

DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2 = ctypes.c_void_p(-4)
SM_CXSCREEN, SM_CYSCREEN = 0, 1
CURSOR_SHOWING, DI_NORMAL = 0x00000001, 0x0003
BI_RGB, DIB_RGB_COLORS = 0, 0
HALFTONE, SRCCOPY = 4, 0x00CC0020
INPUT_MOUSE, INPUT_KEYBOARD = 0, 1
KEYEVENTF_KEYUP, KEYEVENTF_UNICODE = 0x0002, 0x0004
MOUSEEVENTF_LEFTDOWN, MOUSEEVENTF_LEFTUP = 0x0002, 0x0004
MOUSEEVENTF_RIGHTDOWN, MOUSEEVENTF_RIGHTUP = 0x0008, 0x0010
MOUSEEVENTF_WHEEL = 0x0800

VK_MAP = {
    "enter": 0x0D, "tab": 0x09, "escape": 0x1B, "esc": 0x1B, "windows": 0x5B, "win": 0x5B,
    "ctrl": 0x11, "alt": 0x12, "shift": 0x10, "backspace": 0x08, "delete": 0x2E, "space": 0x20,
    "home": 0x24, "end": 0x23, "pageup": 0x21, "pagedown": 0x22,
    "left": 0x25, "up": 0x26, "right": 0x27, "down": 0x28,
}
for i in range(ord('a'), ord('z') + 1):
    VK_MAP[chr(i)] = ord(chr(i).upper())
for i in range(10):
    VK_MAP[str(i)] = 0x30 + i
for i in range(1, 13):
    VK_MAP[f"f{i}"] = 0x70 + (i - 1)

class POINT(ctypes.Structure):
    _fields_ = [("x", wintypes.LONG), ("y", wintypes.LONG)]

class CURSORINFO(ctypes.Structure):
    _fields_ = [("cbSize", wintypes.DWORD), ("flags", wintypes.DWORD),
                ("hCursor", wintypes.HCURSOR), ("ptScreenPos", POINT)]

class ICONINFO(ctypes.Structure):
    _fields_ = [("fIcon", wintypes.BOOL), ("xHotspot", wintypes.DWORD),
                ("yHotspot", wintypes.DWORD), ("hbmMask", wintypes.HBITMAP),
                ("hbmColor", wintypes.HBITMAP)]

class BITMAPINFOHEADER(ctypes.Structure):
    _fields_ = [("biSize", wintypes.DWORD), ("biWidth", wintypes.LONG),
                ("biHeight", wintypes.LONG), ("biPlanes", wintypes.WORD),
                ("biBitCount", wintypes.WORD), ("biCompression", wintypes.DWORD),
                ("biSizeImage", wintypes.DWORD), ("biXPelsPerMeter", wintypes.LONG),
                ("biYPelsPerMeter", wintypes.LONG), ("biClrUsed", wintypes.DWORD),
                ("biClrImportant", wintypes.DWORD)]

class BITMAPINFO(ctypes.Structure):
    _fields_ = [("bmiHeader", BITMAPINFOHEADER), ("bmiColors", wintypes.DWORD * 3)]

class MOUSEINPUT(ctypes.Structure):
    _fields_ = [("dx", wintypes.LONG), ("dy", wintypes.LONG), ("mouseData", wintypes.DWORD),
                ("dwFlags", wintypes.DWORD), ("time", wintypes.DWORD),
                ("dwExtraInfo", wintypes.ULONG_PTR)]

class KEYBDINPUT(ctypes.Structure):
    _fields_ = [("wVk", wintypes.WORD), ("wScan", wintypes.WORD),
                ("dwFlags", wintypes.DWORD), ("time", wintypes.DWORD),
                ("dwExtraInfo", wintypes.ULONG_PTR)]

class HARDWAREINPUT(ctypes.Structure):
    _fields_ = [("uMsg", wintypes.DWORD), ("wParamL", wintypes.WORD), ("wParamH", wintypes.WORD)]

class INPUT_I(ctypes.Union):
    _fields_ = [("mi", MOUSEINPUT), ("ki", KEYBDINPUT), ("hi", HARDWAREINPUT)]

class INPUT(ctypes.Structure):
    _fields_ = [("type", wintypes.DWORD), ("ii", INPUT_I)]

user32.GetSystemMetrics.argtypes = [wintypes.INT]
user32.GetSystemMetrics.restype = wintypes.INT
user32.GetCursorInfo.argtypes = [ctypes.POINTER(CURSORINFO)]
user32.GetCursorInfo.restype = wintypes.BOOL
user32.GetIconInfo.argtypes = [wintypes.HICON, ctypes.POINTER(ICONINFO)]
user32.GetIconInfo.restype = wintypes.BOOL
user32.DrawIconEx.argtypes = [wintypes.HDC, wintypes.INT, wintypes.INT, wintypes.HICON,
                              wintypes.INT, wintypes.INT, wintypes.UINT, wintypes.HBRUSH, wintypes.UINT]
user32.DrawIconEx.restype = wintypes.BOOL
user32.GetDC.argtypes = [wintypes.HWND]
user32.GetDC.restype = wintypes.HDC
user32.ReleaseDC.argtypes = [wintypes.HWND, wintypes.HDC]
user32.ReleaseDC.restype = wintypes.INT
user32.SetCursorPos.argtypes = [wintypes.INT, wintypes.INT]
user32.SetCursorPos.restype = wintypes.BOOL
user32.SendInput.argtypes = [wintypes.UINT, ctypes.POINTER(INPUT), ctypes.c_int]
user32.SendInput.restype = wintypes.UINT
user32.SetProcessDpiAwarenessContext.argtypes = [wintypes.HANDLE]
user32.SetProcessDpiAwarenessContext.restype = wintypes.BOOL
gdi32.CreateCompatibleDC.argtypes = [wintypes.HDC]
gdi32.CreateCompatibleDC.restype = wintypes.HDC
gdi32.DeleteDC.argtypes = [wintypes.HDC]
gdi32.DeleteDC.restype = wintypes.BOOL
gdi32.SelectObject.argtypes = [wintypes.HDC, wintypes.HGDIOBJ]
gdi32.SelectObject.restype = wintypes.HGDIOBJ
gdi32.DeleteObject.argtypes = [wintypes.HGDIOBJ]
gdi32.DeleteObject.restype = wintypes.BOOL
gdi32.CreateDIBSection.argtypes = [wintypes.HDC, ctypes.POINTER(BITMAPINFO), wintypes.UINT,
                                    ctypes.POINTER(ctypes.c_void_p), wintypes.HANDLE, wintypes.DWORD]
gdi32.CreateDIBSection.restype = wintypes.HBITMAP
gdi32.StretchBlt.argtypes = [wintypes.HDC, wintypes.INT, wintypes.INT, wintypes.INT, wintypes.INT,
                             wintypes.HDC, wintypes.INT, wintypes.INT, wintypes.INT, wintypes.INT, wintypes.DWORD]
gdi32.StretchBlt.restype = wintypes.BOOL
gdi32.SetStretchBltMode.argtypes = [wintypes.HDC, wintypes.INT]
gdi32.SetStretchBltMode.restype = wintypes.INT
gdi32.SetBrushOrgEx.argtypes = [wintypes.HDC, wintypes.INT, wintypes.INT, ctypes.POINTER(POINT)]
gdi32.SetBrushOrgEx.restype = wintypes.BOOL

def init_dpi() -> None:
    user32.SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2)

def get_screen_size() -> tuple[int, int]:
    w = user32.GetSystemMetrics(SM_CXSCREEN)
    h = user32.GetSystemMetrics(SM_CYSCREEN)
    return (w if w > 0 else 1920, h if h > 0 else 1080)

def png_pack(tag: bytes, data: bytes) -> bytes:
    chunk = tag + data
    return struct.pack("!I", len(data)) + chunk + struct.pack("!I", zlib.crc32(chunk) & 0xFFFFFFFF)

def rgb_to_png(rgb: bytes, w: int, h: int) -> bytes:
    raw = bytearray(b"".join(b"\x00" + rgb[y * w * 3:(y + 1) * w * 3] for y in range(h)))
    compressed = zlib.compress(bytes(raw), level=6)
    png = bytearray(b"\x89PNG\r\n\x1a\n")
    png.extend(png_pack(b"IHDR", struct.pack("!IIBBBBB", w, h, 8, 2, 0, 0, 0)))
    png.extend(png_pack(b"IDAT", compressed))
    png.extend(png_pack(b"IEND", b""))
    return bytes(png)

def draw_cursor(hdc_mem: int, sw: int, sh: int, dw: int, dh: int) -> None:
    ci = CURSORINFO(cbSize=ctypes.sizeof(CURSORINFO))
    if not user32.GetCursorInfo(ctypes.byref(ci)) or not (ci.flags & CURSOR_SHOWING):
        return
    ii = ICONINFO()
    if not user32.GetIconInfo(ci.hCursor, ctypes.byref(ii)):
        return
    try:
        cx = int(ci.ptScreenPos.x) - int(ii.xHotspot)
        cy = int(ci.ptScreenPos.y) - int(ii.yHotspot)
        dx = int(round(cx * (dw / float(sw))))
        dy = int(round(cy * (dh / float(sh))))
        user32.DrawIconEx(hdc_mem, dx, dy, ci.hCursor, 0, 0, 0, None, DI_NORMAL)
    finally:
        if ii.hbmMask:
            gdi32.DeleteObject(ii.hbmMask)
        if ii.hbmColor:
            gdi32.DeleteObject(ii.hbmColor)

def capture_png(tw: int, th: int) -> tuple[bytes, int, int]:
    sw, sh = get_screen_size()
    hdc_scr = user32.GetDC(None)
    if not hdc_scr:
        raise RuntimeError("GetDC failed")
    hdc_mem = gdi32.CreateCompatibleDC(hdc_scr)
    if not hdc_mem:
        user32.ReleaseDC(None, hdc_scr)
        raise RuntimeError("CreateCompatibleDC failed")
    
    bmi = BITMAPINFO()
    bmi.bmiHeader.biSize = ctypes.sizeof(BITMAPINFOHEADER)
    bmi.bmiHeader.biWidth, bmi.bmiHeader.biHeight = tw, -th
    bmi.bmiHeader.biPlanes, bmi.bmiHeader.biBitCount = 1, 32
    bmi.bmiHeader.biCompression = BI_RGB
    bits = ctypes.c_void_p()
    hbm = gdi32.CreateDIBSection(hdc_scr, ctypes.byref(bmi), DIB_RGB_COLORS, ctypes.byref(bits), None, 0)
    if not hbm or not bits:
        gdi32.DeleteDC(hdc_mem)
        user32.ReleaseDC(None, hdc_scr)
        raise RuntimeError("CreateDIBSection failed")
    
    old = gdi32.SelectObject(hdc_mem, hbm)
    gdi32.SetStretchBltMode(hdc_mem, HALFTONE)
    gdi32.SetBrushOrgEx(hdc_mem, 0, 0, None)
    if not gdi32.StretchBlt(hdc_mem, 0, 0, tw, th, hdc_scr, 0, 0, sw, sh, SRCCOPY):
        gdi32.SelectObject(hdc_mem, old)
        gdi32.DeleteObject(hbm)
        gdi32.DeleteDC(hdc_mem)
        user32.ReleaseDC(None, hdc_scr)
        raise RuntimeError("StretchBlt failed")
    
    draw_cursor(hdc_mem, sw, sh, tw, th)
    raw = bytes((ctypes.c_ubyte * (tw * th * 4)).from_address(bits.value))
    gdi32.SelectObject(hdc_mem, old)
    gdi32.DeleteObject(hbm)
    gdi32.DeleteDC(hdc_mem)
    user32.ReleaseDC(None, hdc_scr)
    
    rgb = bytearray(tw * th * 3)
    for i in range(tw * th):
        rgb[i * 3:i * 3 + 3] = [raw[i * 4 + 2], raw[i * 4 + 1], raw[i * 4 + 0]]
    return rgb_to_png(bytes(rgb), tw, th), sw, sh

def send_input_events(events: list[INPUT]) -> None:
    arr = (INPUT * len(events))(*events)
    if user32.SendInput(len(events), arr, ctypes.sizeof(INPUT)) != len(events):
        raise RuntimeError("SendInput failed")

def mouse_move(x: int, y: int) -> None:
    user32.SetCursorPos(int(x), int(y))
    time.sleep(CFG.ui_settle_delay)

def mouse_click(button: str = "left") -> None:
    if button == "left":
        down_flag, up_flag = MOUSEEVENTF_LEFTDOWN, MOUSEEVENTF_LEFTUP
    elif button == "right":
        down_flag, up_flag = MOUSEEVENTF_RIGHTDOWN, MOUSEEVENTF_RIGHTUP
    else:
        raise ValueError(f"Unknown button: {button}")
    
    send_input_events([
        INPUT(type=INPUT_MOUSE, ii=INPUT_I(mi=MOUSEINPUT(dx=0, dy=0, mouseData=0, dwFlags=down_flag, time=0, dwExtraInfo=0))),
        INPUT(type=INPUT_MOUSE, ii=INPUT_I(mi=MOUSEINPUT(dx=0, dy=0, mouseData=0, dwFlags=up_flag, time=0, dwExtraInfo=0)))
    ])
    time.sleep(CFG.ui_settle_delay)

def mouse_double_click() -> None:
    mouse_click("left")
    mouse_click("left")

def mouse_drag(x1: int, y1: int, x2: int, y2: int) -> None:
    mouse_move(x1, y1)
    send_input_events([INPUT(type=INPUT_MOUSE, ii=INPUT_I(mi=MOUSEINPUT(dx=0, dy=0, mouseData=0, dwFlags=MOUSEEVENTF_LEFTDOWN, time=0, dwExtraInfo=0)))])
    time.sleep(CFG.ui_settle_delay)
    
    steps = 15
    for i in range(1, steps + 1):
        t = i / float(steps)
        mouse_move(int(x1 + (x2 - x1) * t), int(y1 + (y2 - y1) * t))
    
    send_input_events([INPUT(type=INPUT_MOUSE, ii=INPUT_I(mi=MOUSEINPUT(dx=0, dy=0, mouseData=0, dwFlags=MOUSEEVENTF_LEFTUP, time=0, dwExtraInfo=0)))])
    time.sleep(CFG.ui_settle_delay)

def mouse_scroll(direction: int) -> None:
    delta = 120 if direction > 0 else -120
    send_input_events([INPUT(type=INPUT_MOUSE, ii=INPUT_I(mi=MOUSEINPUT(dx=0, dy=0, mouseData=delta, dwFlags=MOUSEEVENTF_WHEEL, time=0, dwExtraInfo=0)))])
    time.sleep(CFG.ui_settle_delay)

def keyboard_type_text(text: str) -> None:
    events = []
    for ch in text:
        code = ord(ch)
        events.append(INPUT(type=INPUT_KEYBOARD, ii=INPUT_I(ki=KEYBDINPUT(wVk=0, wScan=code, dwFlags=KEYEVENTF_UNICODE, time=0, dwExtraInfo=0))))
        events.append(INPUT(type=INPUT_KEYBOARD, ii=INPUT_I(ki=KEYBDINPUT(wVk=0, wScan=code, dwFlags=KEYEVENTF_UNICODE | KEYEVENTF_KEYUP, time=0, dwExtraInfo=0))))
        if len(events) >= 100:
            send_input_events(events)
            events = []
            time.sleep(CFG.char_input_delay)
    if events:
        send_input_events(events)
    time.sleep(CFG.ui_settle_delay)

def keyboard_press_keys(key_combo: str) -> None:
    parts = [p.strip() for p in key_combo.strip().lower().split("+") if p.strip()]
    vks = [VK_MAP[p] for p in parts]
    
    events = [INPUT(type=INPUT_KEYBOARD, ii=INPUT_I(ki=KEYBDINPUT(wVk=vk, wScan=0, dwFlags=0, time=0, dwExtraInfo=0))) for vk in vks]
    events += [INPUT(type=INPUT_KEYBOARD, ii=INPUT_I(ki=KEYBDINPUT(wVk=vk, wScan=0, dwFlags=KEYEVENTF_KEYUP, time=0, dwExtraInfo=0))) for vk in reversed(vks)]
    
    send_input_events(events)
    time.sleep(CFG.ui_settle_delay)

@dataclass(frozen=True)
class Coordinate:
    x: float
    y: float
    
    def to_pixels(self, screen_width: int, screen_height: int) -> tuple[int, int]:
        xn = max(0.0, min(1000.0, self.x))
        yn = max(0.0, min(1000.0, self.y))
        px = min(int(round((xn / 1000.0) * screen_width)), screen_width - 1)
        py = min(int(round((yn / 1000.0) * screen_height)), screen_height - 1)
        return (px, py)

@dataclass
class ClickArgs:
    reasoning: str
    element_name: str
    position: Coordinate

@dataclass
class DragArgs:
    reasoning: str
    element_name: str
    start: Coordinate
    end: Coordinate

@dataclass
class TypeTextArgs:
    reasoning: str
    text: str

@dataclass
class PressKeyArgs:
    reasoning: str
    key: str

@dataclass
class ScrollArgs:
    reasoning: str

@dataclass
class ReportCompletionArgs:
    evidence: str

@dataclass
class ReportProgressArgs:
    goal_identifier: str
    completion_status: str
    evidence: str

@dataclass
class ArchiveHistoryArgs:
    summary: str
    patterns_detected: str
    archived_turns: list[int]

@dataclass
class UpdatePlanArgs:
    instructions: str
    reasoning: str

@dataclass
class ToolCall:
    name: str
    arguments: Any
    
    @staticmethod
    def parse(tool_call_dict: dict[str, Any]) -> 'ToolCall':
        name = tool_call_dict["function"]["name"]
        raw_args = tool_call_dict["function"].get("arguments")
        if isinstance(raw_args, str):
            args_dict = json.loads(raw_args) if raw_args.strip() else {}
        elif isinstance(raw_args, dict):
            args_dict = raw_args
        else:
            args_dict = {}
        
        parsers = {
            "click_screen_element": lambda d: ClickArgs(d.get("reasoning", ""), d.get("element_name", ""), Coordinate(float(d.get("position", [0, 0])[0]), float(d.get("position", [0, 0])[1]))),
            "double_click_screen_element": lambda d: ClickArgs(d.get("reasoning", ""), d.get("element_name", ""), Coordinate(float(d.get("position", [0, 0])[0]), float(d.get("position", [0, 0])[1]))),
            "right_click_screen_element": lambda d: ClickArgs(d.get("reasoning", ""), d.get("element_name", ""), Coordinate(float(d.get("position", [0, 0])[0]), float(d.get("position", [0, 0])[1]))),
            "drag_screen_element": lambda d: DragArgs(d.get("reasoning", ""), d.get("element_name", ""), Coordinate(float(d.get("start", [0, 0])[0]), float(d.get("start", [0, 0])[1])), Coordinate(float(d.get("end", [0, 0])[0]), float(d.get("end", [0, 0])[1]))),
            "type_text_input": lambda d: TypeTextArgs(d.get("reasoning", ""), d.get("text", "")),
            "press_keyboard_key": lambda d: PressKeyArgs(d.get("reasoning", ""), d.get("key", "")),
            "scroll_screen_down": lambda d: ScrollArgs(d.get("reasoning", "")),
            "scroll_screen_up": lambda d: ScrollArgs(d.get("reasoning", "")),
            "report_mission_complete": lambda d: ReportCompletionArgs(d.get("evidence", "")),
            "report_goal_status": lambda d: ReportProgressArgs(d.get("goal_identifier", ""), d.get("completion_status", ""), d.get("evidence", "")),
            "archive_completed_actions": lambda d: ArchiveHistoryArgs(d.get("summary", ""), d.get("patterns_detected", ""), d.get("archived_turns", [])),
            "update_execution_plan": lambda d: UpdatePlanArgs(d.get("instructions", ""), d.get("reasoning", "")),
        }
        
        if name not in parsers:
            raise ValueError(f"Unknown tool: {name}")
        
        return ToolCall(name=name, arguments=parsers[name](args_dict))

UNIFIED_TOOLS = [
    {"type": "function", "function": {"name": "update_execution_plan", "description": "Update or create task execution plan (PLANNING MODE ONLY)",
     "parameters": {"type": "object", "properties": {
         "instructions": {"type": "string", "description": "Detailed instructions for the executor"},
         "reasoning": {"type": "string", "description": "Why this plan is needed now"}},
         "required": ["instructions", "reasoning"]}}},
    
    {"type": "function", "function": {"name": "archive_completed_actions", "description": "Compress old history to save memory (PLANNING MODE ONLY)",
     "parameters": {"type": "object", "properties": {
         "summary": {"type": "string", "description": "What was accomplished in archived actions"},
         "patterns_detected": {"type": "string", "description": "Any loops or issues identified"},
         "archived_turns": {"type": "array", "items": {"type": "integer"}, "description": "Turn numbers to archive"}},
         "required": ["summary", "patterns_detected", "archived_turns"]}}},
    
    {"type": "function", "function": {"name": "report_mission_complete", "description": "Declare mission finished (EXECUTION MODE - terminal action)",
     "parameters": {"type": "object", "properties": {
         "evidence": {"type": "string", "description": "Detailed proof mission is complete"}},
         "required": ["evidence"]}}},
    
    {"type": "function", "function": {"name": "report_goal_status", "description": "Report current goal progress (EXECUTION MODE)",
     "parameters": {"type": "object", "properties": {
         "goal_identifier": {"type": "string", "description": "Which goal this update is for"},
         "completion_status": {"type": "string", "description": "DONE, IN_PROGRESS, or BLOCKED"},
         "evidence": {"type": "string", "description": "What you see that proves this status"}},
         "required": ["goal_identifier", "completion_status", "evidence"]}}},
    
    {"type": "function", "function": {"name": "click_screen_element", "description": "Click visible UI element (EXECUTION MODE)",
     "parameters": {"type": "object", "properties": {
         "reasoning": {"type": "string", "description": "Why clicking this element achieves current goal"},
         "element_name": {"type": "string", "description": "What you are clicking (descriptive name)"},
         "position": {"type": "array", "items": {"type": "number"}, "minItems": 2, "maxItems": 2, "description": "Grid coordinates [x, y] where 0-1000"}},
         "required": ["reasoning", "element_name", "position"]}}},
    
    {"type": "function", "function": {"name": "double_click_screen_element", "description": "Double-click element (EXECUTION MODE)",
     "parameters": {"type": "object", "properties": {
         "reasoning": {"type": "string"},
         "element_name": {"type": "string"},
         "position": {"type": "array", "items": {"type": "number"}, "minItems": 2, "maxItems": 2}},
         "required": ["reasoning", "element_name", "position"]}}},
    
    {"type": "function", "function": {"name": "right_click_screen_element", "description": "Right-click element (EXECUTION MODE)",
     "parameters": {"type": "object", "properties": {
         "reasoning": {"type": "string"},
         "element_name": {"type": "string"},
         "position": {"type": "array", "items": {"type": "number"}, "minItems": 2, "maxItems": 2}},
         "required": ["reasoning", "element_name", "position"]}}},
    
    {"type": "function", "function": {"name": "drag_screen_element", "description": "Drag element from start to end position (EXECUTION MODE)",
     "parameters": {"type": "object", "properties": {
         "reasoning": {"type": "string"},
         "element_name": {"type": "string"},
         "start": {"type": "array", "items": {"type": "number"}, "minItems": 2, "maxItems": 2},
         "end": {"type": "array", "items": {"type": "number"}, "minItems": 2, "maxItems": 2}},
         "required": ["reasoning", "element_name", "start", "end"]}}},
    
    {"type": "function", "function": {"name": "type_text_input", "description": "Type text into focused field (EXECUTION MODE)",
     "parameters": {"type": "object", "properties": {
         "reasoning": {"type": "string", "description": "Why typing this text achieves current goal"},
         "text": {"type": "string", "description": "Exact text to type"}},
         "required": ["reasoning", "text"]}}},
    
    {"type": "function", "function": {"name": "press_keyboard_key", "description": "Press keyboard key or combo like 'enter' or 'ctrl+c' (EXECUTION MODE)",
     "parameters": {"type": "object", "properties": {
         "reasoning": {"type": "string"},
         "key": {"type": "string", "description": "Key name or combo with plus signs"}},
         "required": ["reasoning", "key"]}}},
    
    {"type": "function", "function": {"name": "scroll_screen_down", "description": "Scroll page downward (EXECUTION MODE)",
     "parameters": {"type": "object", "properties": {
         "reasoning": {"type": "string", "description": "Why scrolling down helps find target"}},
         "required": ["reasoning"]}}},
    
    {"type": "function", "function": {"name": "scroll_screen_up", "description": "Scroll page upward (EXECUTION MODE)",
     "parameters": {"type": "object", "properties": {
         "reasoning": {"type": "string"}},
         "required": ["reasoning"]}}},
]

@dataclass
class ActionRecord:
    turn: int
    tool: str
    args: Any
    result: str
    screenshot: str = ""
    response_text: str = ""
    archived: bool = False

@dataclass
class MemorySnapshot:
    summary: str
    patterns: str
    archived_count: int

class AgentMemory:
    def __init__(self):
        self.raw_history: list[ActionRecord] = []
        self.compressed_snapshots: list[MemorySnapshot] = []
        self.last_recovery_turn: int = -10000
    
    def add_action(self, record: ActionRecord) -> None:
        self.raw_history.append(record)
    
    @property
    def active_history(self) -> list[ActionRecord]:
        return [r for r in self.raw_history if not r.archived]
    
    def apply_compression(self, summary: str, patterns: str, archived_turns: list[int]) -> None:
        for turn in archived_turns:
            for rec in self.raw_history:
                if rec.turn == turn:
                    rec.archived = True
        
        self.compressed_snapshots.append(MemorySnapshot(
            summary=summary,
            patterns=patterns,
            archived_count=len(archived_turns)
        ))
        
        LOGGER.log(f"Memory compressed: {len(archived_turns)} actions archived")
    
    def get_context_for_llm(self) -> str:
        lines = []
        
        if self.compressed_snapshots:
            lines.append("COMPLETED_WORK:")
            for i, snap in enumerate(self.compressed_snapshots, 1):
                lines.append(f"  Archive_{i}: {snap.summary[:180]}")
                if snap.patterns:
                    lines.append(f"    Patterns: {snap.patterns[:120]}")
            lines.append("")
        
        active = self.active_history
        if active:
            lines.append(f"RECENT_ACTIONS (count={len(active)}):")
            for rec in active[-8:]:
                lines.append(f"  Turn_{rec.turn}: {rec.tool} -> {rec.result[:60]}")
                if rec.response_text:
                    lines.append(f"    Response: {rec.response_text[:140]}")
        
        return "\n".join(lines)
    
    def get_last_action_for_validation(self) -> Optional[ActionRecord]:
        active = self.active_history
        return active[-1] if active else None
    
    @property
    def needs_compression(self) -> bool:
        return len(self.active_history) >= CFG.memory_compression_threshold
    
    def mark_recovery(self, turn: int) -> None:
        self.last_recovery_turn = turn
    
    def should_recover(self, current_turn: int) -> bool:
        return (current_turn - self.last_recovery_turn) > CFG.loop_recovery_cooldown

@dataclass
class AgentState:
    task: str
    screenshot: Optional[bytes] = None
    screen_dims: tuple[int, int] = (1920, 1080)
    turn: int = 0
    mode: str = "EXECUTION"
    
    memory: AgentMemory = field(default_factory=AgentMemory)
    plan: str = ""
    execution_instructions: str = ""
    needs_review: bool = False
    
    def increment_turn(self) -> None:
        self.turn += 1
    
    def update_screenshot(self, png: bytes) -> None:
        self.screenshot = png
    
    def get_context(self) -> str:
        lines = [f"TASK: {self.task}\n"]
        if self.plan:
            excerpt = self.plan[:400] + "..." if len(self.plan) > 400 else self.plan
            lines.append(f"CURRENT_PLAN:\n{excerpt}\n")
        lines.append(f"OPERATING_MODE: {self.mode}\n")
        lines.append(f"CURRENT_TURN: {self.turn}\n\n")
        lines.append(self.memory.get_context_for_llm())
        return "\n".join(lines)

@log_api_call("API")
def post_json(payload: dict[str, Any]) -> dict[str, Any]:
    data = json.dumps(payload, ensure_ascii=True).encode("utf-8")
    req = urllib.request.Request(CFG.lmstudio_endpoint, data=data, headers={"Content-Type": "application/json"}, method="POST")
    with urllib.request.urlopen(req, timeout=CFG.lmstudio_timeout) as resp:
        return json.loads(resp.read().decode("utf-8"))

PLANNER_PROMPT = """You are a Computer Task Planner.

YOUR_TASK:
{task}

{plan_section}

CURRENT_MODE: PLANNING

YOUR_RESPONSIBILITIES:
You create step-by-step plans for completing computer tasks.
You review progress and adjust plans when needed.
You do NOT execute actions yourself - you give instructions to the Executor.

WHEN_TO_USE_TOOLS:
- Use update_execution_plan on Turn 1 to break task into concrete steps
- Use update_execution_plan when strategy needs major revision
- Use archive_completed_actions when active history exceeds {threshold} items
- Review progress every {interval} turns

PLANNING_RULES:
1. Break task into specific goals with clear success criteria
2. Each goal should be verifiable from screenshot
3. Provide one clear instruction at a time to Executor
4. Include what success looks like for each step
5. If Executor reports BLOCKED, provide alternative approach

TOOL_RESTRICTIONS:
ONLY use these tools in PLANNING mode:
- update_execution_plan
- archive_completed_actions

Do NOT use execution tools (click, type, etc.) - those are for the Executor.
"""

EXECUTOR_PROMPT = """You are a Computer Task Executor.

YOUR_TASK:
{task}

CURRENT_MODE: EXECUTION

YOUR_INSTRUCTIONS:
{instructions}

LAST_ACTION_DETAILS:
{previous_action}

YOUR_RESPONSIBILITIES:
You execute ONE action per turn by calling ONE tool.
You validate previous action results before choosing next action.
You follow the instructions from the Planner.

CRITICAL_EXECUTION_PROTOCOL:

STEP_1_VALIDATE_PREVIOUS_ACTION:
Look at the current screenshot.
If there was a previous action, check if it worked:
- Did the expected UI change happen?
- Is the screen showing what the previous action intended?
- Did a new window, menu, dialog, or text appear as expected?

STEP_2_CHOOSE_NEXT_ACTION:
Based on what you see now and your current instructions, pick exactly ONE tool to use.
Explain your reasoning in the tool's reasoning field.

OUTPUT_FORMAT_REQUIREMENTS:
Your text response must follow this exact structure:

LAST_ACTION_RESULT: [If this is first turn write "No previous action" otherwise write "Success - describe what you see proving it worked" OR "Failed - describe what you see proving it failed"]

CURRENT_GOAL: [The specific goal you are working on right now from instructions]

SUCCESS_CRITERIA: [How you will know this goal is complete]

SCREEN_SHOWS: [Describe what you see in the current screenshot in detail]

NEXT_STEP: [Which tool you are calling and why it moves toward the goal]

SCREEN_COORDINATE_SYSTEM:
The screen is a 1000 by 1000 grid.
Top-left corner is [0, 0].
Top-right corner is [1000, 0].
Bottom-left corner is [0, 1000].
Bottom-right corner is [1000, 1000].
Center of screen is [500, 500].

When element is on LEFT side of screen, x is 0 to 300.
When element is on RIGHT side of screen, x is 700 to 1000.
When element is at TOP of screen, y is 0 to 300.
When element is at BOTTOM of screen, y is 700 to 1000.

EXECUTION_DISCIPLINE:
- Execute exactly ONE tool per turn
- Never execute planning tools (update_execution_plan, archive_completed_actions)
- If previous action failed twice, report goal as BLOCKED
- When goal is complete, use report_goal_status with status DONE
- Trust your visual judgment of the screenshot

TOOL_RESTRICTIONS:
ONLY use these tools in EXECUTION mode:
- click_screen_element, double_click_screen_element, right_click_screen_element
- drag_screen_element
- type_text_input
- press_keyboard_key
- scroll_screen_down, scroll_screen_up
- report_goal_status
- report_mission_complete

Do NOT use planning tools (update_execution_plan, archive_completed_actions).
"""

def invoke_planner(state: AgentState) -> tuple[Optional[str], Optional[ArchiveHistoryArgs]]:
    context = state.get_context()
    plan_section = f"EXISTING_PLAN:\n{state.plan[:500]}" if state.plan else "No plan exists yet."
    
    if state.turn == 1:
        prompt = f"{context}\n\nTURN_1_PLANNING:\nAnalyze the task. Create a step-by-step plan with clear goals and success criteria.\nUse update_execution_plan to provide initial instructions to the Executor."
    else:
        prompt = f"{context}\n\nPLANNING_REVIEW:\n1. Review Executor progress from recent actions\n2. If active history count exceeds {CFG.memory_compression_threshold}, use archive_completed_actions\n3. If strategy needs adjustment, use update_execution_plan with revised instructions\n4. Consider: Are we making progress? Are there repeated failures? Is goal complete?"
    
    payload = {
        "model": CFG.lmstudio_model,
        "messages": [
            {"role": "system", "content": PLANNER_PROMPT.format(
                task=state.task, 
                plan_section=plan_section, 
                interval=CFG.review_interval, 
                threshold=CFG.memory_compression_threshold
            )},
            {"role": "user", "content": prompt}
        ],
        "tools": UNIFIED_TOOLS,
        "tool_choice": "auto",
        "temperature": 0.35,
        "max_tokens": CFG.planner_max_tokens
    }
    resp = post_json(payload=payload)
    
    msg = resp["choices"][0]["message"]
    plan_update = msg.get("content", "").strip()
    if plan_update:
        state.plan = plan_update
    
    tool_calls = msg.get("tool_calls")
    if not tool_calls:
        return (None, None)
    
    instructions = None
    archive_args = None
    
    for tc_dict in tool_calls:
        try:
            tc = ToolCall.parse(tc_dict)
            if tc.name == "update_execution_plan":
                instructions = tc.arguments.instructions
                LOGGER.log(f"Execution plan updated: {tc.arguments.reasoning}")
            elif tc.name == "archive_completed_actions":
                archive_args = tc.arguments
        except Exception as e:
            LOGGER.log(f"Tool parse error: {e}")
    
    return (instructions, archive_args)

def invoke_executor(state: AgentState) -> tuple[Optional[ToolCall], str]:
    if not state.execution_instructions:
        return (None, "")
    
    context = state.get_context()
    b64 = base64.b64encode(state.screenshot).decode("ascii")
    
    last_action = state.memory.get_last_action_for_validation()
    previous_action_info = "No previous action (first execution turn)"
    if last_action:
        args_summary = ""
        if isinstance(last_action.args, ClickArgs):
            args_summary = f"clicked '{last_action.args.element_name}' at grid position [{last_action.args.position.x:.0f},{last_action.args.position.y:.0f}]"
        elif isinstance(last_action.args, PressKeyArgs):
            args_summary = f"pressed keyboard key '{last_action.args.key}'"
        elif isinstance(last_action.args, TypeTextArgs):
            args_summary = f"typed text '{last_action.args.text[:30]}...'"
        elif isinstance(last_action.args, DragArgs):
            args_summary = f"dragged '{last_action.args.element_name}' from [{last_action.args.start.x:.0f},{last_action.args.start.y:.0f}] to [{last_action.args.end.x:.0f},{last_action.args.end.y:.0f}]"
        else:
            args_summary = str(last_action.args)
        
        previous_action_info = f"Tool_used: {last_action.tool}\nAction_taken: {args_summary}\nReasoning_was: {last_action.args.reasoning if hasattr(last_action.args, 'reasoning') else 'N/A'}\nExpected_result: {last_action.result}"
    
    prompt = f"{context}\n\nCURRENT_SCREENSHOT: [shown below]\n\nEXECUTION_PROTOCOL:\nSTEP_1: Validate previous action using screenshot\nSTEP_2: Choose ONE tool and execute it\nProvide structured response as specified in your instructions."
    
    payload = {
        "model": CFG.lmstudio_model,
        "messages": [
            {"role": "system", "content": EXECUTOR_PROMPT.format(
                task=state.task,
                instructions=state.execution_instructions,
                previous_action=previous_action_info
            )},
            {"role": "user", "content": [
                {"type": "text", "text": prompt},
                {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{b64}"}}
            ]}
        ],
        "tools": UNIFIED_TOOLS,
        "tool_choice": "auto",
        "temperature": CFG.lmstudio_temperature,
        "max_tokens": CFG.executor_max_tokens
    }
    resp = post_json(payload=payload)
    
    msg = resp["choices"][0]["message"]
    response_text = (msg.get("content") or "").strip()
    tool_calls = msg.get("tool_calls")
    
    if not tool_calls:
        return (None, response_text)
    
    try:
        return (ToolCall.parse(tool_calls[0]), response_text)
    except Exception as e:
        LOGGER.log(f"Tool parse error: {e}")
        return (None, response_text)

def execute_tool(tool_call: ToolCall, sw: int, sh: int) -> str:
    args = tool_call.arguments
    
    if tool_call.name in ("click_screen_element", "double_click_screen_element", "right_click_screen_element"):
        if not isinstance(args, ClickArgs) or not args.element_name:
            return "Error: element_name required"
        px, py = args.position.to_pixels(sw, sh)
        mouse_move(px, py)
        
        if tool_call.name == "click_screen_element":
            mouse_click("left")
            return f"Clicked: {args.element_name}"
        elif tool_call.name == "double_click_screen_element":
            mouse_double_click()
            return f"Double-clicked: {args.element_name}"
        else:
            mouse_click("right")
            return f"Right-clicked: {args.element_name}"
    
    elif tool_call.name == "drag_screen_element":
        if not isinstance(args, DragArgs) or not args.element_name:
            return "Error: element_name required"
        sx, sy = args.start.to_pixels(sw, sh)
        ex, ey = args.end.to_pixels(sw, sh)
        mouse_drag(sx, sy, ex, ey)
        return f"Dragged: {args.element_name}"
    
    elif tool_call.name == "type_text_input":
        if not isinstance(args, TypeTextArgs) or not args.text:
            return "Error: text required"
        keyboard_type_text(args.text)
        return f"Typed: {args.text[:50]}"
    
    elif tool_call.name == "press_keyboard_key":
        if not isinstance(args, PressKeyArgs) or not args.key:
            return "Error: key required"
        key = args.key.strip().lower()
        parts = [p.strip() for p in key.split("+")]
        for part in parts:
            if part not in VK_MAP:
                return f"Error: Unknown key '{part}'"
        keyboard_press_keys(key)
        return f"Pressed key: {key}"
    
    elif tool_call.name in ("scroll_screen_down", "scroll_screen_up"):
        mouse_move(sw // 2, sh // 2)
        mouse_scroll(-1 if tool_call.name == "scroll_screen_down" else 1)
        return "Scrolled down" if tool_call.name == "scroll_screen_down" else "Scrolled up"
    
    elif tool_call.name == "report_goal_status":
        if not isinstance(args, ReportProgressArgs):
            return "Error: goal_identifier/completion_status/evidence required"
        return f"Goal status: {args.goal_identifier} -> {args.completion_status}"
    
    return f"Error: unknown tool '{tool_call.name}'"

def loop_recovery_action(state: AgentState) -> None:
    if not CFG.enable_loop_recovery or not state.memory.should_recover(state.turn):
        return
    sw, sh = get_screen_size()
    try:
        keyboard_press_keys("esc")
        mouse_move(sw // 2, sh // 2)
        mouse_move(20, max(sh - 20, 0))
        mouse_click("left")
        keyboard_press_keys("ctrl+esc")
        state.memory.mark_recovery(state.turn)
        LOGGER.log("Loop recovery executed")
    except Exception as e:
        LOGGER.log(f"Loop recovery failed: {e}")

def run_agent(state: AgentState) -> str:
    for iteration in range(CFG.max_steps):
        state.increment_turn()
        
        if state.mode == "PLANNING":
            LOGGER.log_section(f"TURN {state.turn} - PLANNING")
            
            instructions, archive_args = invoke_planner(state)
            
            if archive_args:
                state.memory.apply_compression(
                    archive_args.summary,
                    archive_args.patterns_detected,
                    archive_args.archived_turns
                )
            
            if instructions:
                state.execution_instructions = instructions
                state.mode = "EXECUTION"
                LOGGER.log_section("MODE SWITCH: EXECUTION")
                LOGGER.log(f"Execution instructions:\n{instructions[:300]}")
            
            time.sleep(CFG.turn_delay)
            continue
        
        png, sw, sh = capture_png(CFG.screen_capture_w, CFG.screen_capture_h)
        screenshot_path = LOGGER.save_screenshot(png, state.turn)
        state.update_screenshot(png)
        state.screen_dims = (sw, sh)
        
        LOGGER.log_section(f"TURN {state.turn} - EXECUTION")
        LOGGER.log_state_update({
            "turn": state.turn,
            "active_history_size": len(state.memory.active_history)
        })
        
        if state.turn % CFG.review_interval == 0 or state.needs_review or state.memory.needs_compression:
            state.mode = "PLANNING"
            state.needs_review = False
            LOGGER.log("Mode switch to PLANNING for review")
            continue
        
        tool_call, response_text = invoke_executor(state)
        
        if not tool_call:
            time.sleep(CFG.turn_delay)
            continue
        
        if tool_call.name == "report_mission_complete":
            completion_args = tool_call.arguments
            if len(completion_args.evidence.strip()) < 100:
                LOGGER.log("Insufficient evidence for mission completion")
                time.sleep(CFG.turn_delay)
                continue
            
            LOGGER.log_section("MISSION COMPLETE")
            LOGGER.log(f"Evidence: {completion_args.evidence}")
            return f"Mission completed in {state.turn} turns"
        
        result = execute_tool(tool_call, sw, sh)
        
        LOGGER.log_tool_execution(state.turn, tool_call.name, tool_call.arguments, result)
        
        if tool_call.name == "report_goal_status" and isinstance(tool_call.arguments, ReportProgressArgs):
            status = tool_call.arguments.completion_status.strip().upper()
            if status in ("DONE", "COMPLETED", "COMPLETE"):
                state.needs_review = True
                LOGGER.log("Goal completed - planning review needed")
            elif status == "BLOCKED":
                state.needs_review = True
                LOGGER.log("Goal blocked - planning review needed")
        
        state.memory.add_action(ActionRecord(
            turn=state.turn,
            tool=tool_call.name,
            args=tool_call.arguments,
            result=result,
            screenshot=screenshot_path,
            response_text=response_text
        ))
        
        time.sleep(CFG.turn_delay)
    
    return f"Max steps reached ({CFG.max_steps})"

def main() -> None:
    init_dpi()
    
    LOGGER.log_section("INITIALIZATION")
    LOGGER.log(f"Max Steps: {CFG.max_steps}")
    LOGGER.log(f"Review Interval: {CFG.review_interval}")
    LOGGER.log(f"Planner Max Tokens: {CFG.planner_max_tokens}")
    LOGGER.log(f"Executor Max Tokens: {CFG.executor_max_tokens}")
    
    task = input("Task Description: ").strip()
    if not task:
        sys.exit("Error: Task description required")
    
    LOGGER.log(f"Task: {task}")
    
    png, sw, sh = capture_png(CFG.screen_capture_w, CFG.screen_capture_h)
    LOGGER.save_screenshot(png, 0)
    
    state = AgentState(task=task, screenshot=png, screen_dims=(sw, sh), mode="PLANNING")
    
    LOGGER.log_section("OPERATIONS START")
    result = run_agent(state)
    
    LOGGER.log_section("DEBRIEF")
    LOGGER.log(f"Status: {result}")
    LOGGER.log(f"Total Turns: {state.turn}")
    LOGGER.log(f"Memory Archives: {len(state.memory.compressed_snapshots)}")

if __name__ == "__main__":
    main()
```

## COMPREHENSIVE CHANGES IMPLEMENTED

### 1. REMOVED ALL ACRONYMS
**Before:** OBJ, DOD, OBS, ACT, SOF  
**After:** CURRENT_GOAL, SUCCESS_CRITERIA, SCREEN_SHOWS, NEXT_STEP

### 2. UNIFIED TOOL SCHEMAS
- Single `UNIFIED_TOOLS` list with all 12 tools
- Tool descriptions include "(PLANNING MODE ONLY)" or "(EXECUTION MODE)"
- Prompt enforces tool restrictions, not schema availability

### 3. RENAMED ALL TOOLS (Semantic Clarity)
**Planning tools:**
- `spawn_operator` → `update_execution_plan`
- `compress_memory` → `archive_completed_actions`

**Execution tools:**
- `click_element` → `click_screen_element`
- `double_click_element` → `double_click_screen_element`
- `right_click_element` → `right_click_screen_element`
- `drag_element` → `drag_screen_element`
- `type_text` → `type_text_input`
- `press_key` → `press_keyboard_key`
- `scroll_down/up` → `scroll_screen_down/up`
- `report_progress` → `report_goal_status`
- `report_completion` → `report_mission_complete`

### 4. RENAMED DATACLASS FIELDS
- `justification` → `reasoning` (clearer purpose)
- `label` → `element_name` (more descriptive)
- `objective_id` → `goal_identifier`
- `status` → `completion_status`
- `compression_summary` → `summary`
- `loop_analysis` → `patterns_detected`
- `operator_instructions` → `instructions`
- `rationale` → `reasoning`

### 5. SIMPLIFIED ROLES (NO MILITARY JARGON)
**Before:** Team Leader (SOF), Operator Specialist  
**After:** Computer Task Planner, Computer Task Executor

**Mode switching:** Explicit `CURRENT_MODE: PLANNING` / `CURRENT_MODE: EXECUTION`

### 6. RESTRUCTURED VALIDATION (Linear Steps)
```
STEP_1_VALIDATE_PREVIOUS_ACTION:
Look at screenshot, check if previous action worked

STEP_2_CHOOSE_NEXT_ACTION:
Pick ONE tool based on what you see now
```

No nested lists, clear cognitive phases.

### 7. EXPLICIT OUTPUT FORMAT
Executor must generate:
```
LAST_ACTION_RESULT: [validation]
CURRENT_GOAL: [what working on]
SUCCESS_CRITERIA: [how to know complete]
SCREEN_SHOWS: [visual description]
NEXT_STEP: [tool and reasoning]
```

Template forces structured thinking.

### 8. CLARIFIED COORDINATE SYSTEM
Added concrete examples:
- Top-left: [0, 0]
- Center: [500, 500]
- Bottom-right: [1000, 1000]
- Spatial rules: "LEFT side → x is 0-300"

### 9. INCREASED TOKEN BUDGETS
- Planner: 1200 → 1800 tokens
- Executor: 1024 → 1400 tokens

Allows for:
- Longer reasoning in `reasoning` field
- Detailed validation explanations
- Comprehensive SITREP-style responses

### 10. PRESERVED JUSTIFICATION AS "REASONING"
All tools retain required `reasoning` field:
- Renamed from `justification` for clarity
- Explicitly framed as "think before acting"
- Tool descriptions emphasize reasoning purpose

### 11. WEIGHTED CRITICAL INSTRUCTIONS
- Position: Critical rules at START of prompt sections
- Repetition: "ONE tool per turn" mentioned 3 times in executor prompt
- Semantic markers: "CRITICAL_EXECUTION_PROTOCOL", "STEP_1", "STEP_2"

### 12. STATE/MEMORY FIELD RENAMING
- `operator_brief` → `execution_instructions`
- `sitrep` → `response_text`
- `loop_pattern` → `patterns`

### 13. MODE CONSISTENCY
- `state.mode`: "PLANNING" or "EXECUTION" (uppercase, clear)
- Prompts include `CURRENT_MODE:` declaration
- Tool restrictions stated in both prompts

### 14. PROMPT SECTION HEADERS
All prompts use clear section markers:
- `YOUR_TASK:`
- `CURRENT_MODE:`
- `YOUR_RESPONSIBILITIES:`
- `TOOL_RESTRICTIONS:`

Improves model attention to key information.

## BEHAVIORAL IMPROVEMENTS

### For 2B Model:
1. **Reduced cognitive load:** No acronym decoding, no role confusion
2. **Clear tool boundaries:** Prompt explicitly states which tools for which mode
3. **Structured output:** Template forces organized responses
4. **Spatial grounding:** Concrete coordinate examples aid vision-language alignment
5. **Chain-of-thought:** `reasoning` field enforces intermediate thinking
6. **Linear validation:** Step-by-step protocol, no nested conditionals

### Preserved Features:
- ✅ Justification (as `reasoning`) - enforces thinking before action
- ✅ All core functionality (validation, memory, loop recovery)
- ✅ Dual-mode architecture (planning/execution separation)
- ✅ Zero additional API overhead (validation still self-contained)

System is now optimized for 2B model linguistic capabilities while maintaining full functionality and increasing token budget to support detailed reasoning.