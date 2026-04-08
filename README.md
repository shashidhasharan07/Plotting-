import socket
import struct
import threading
import time
from collections import deque, defaultdict
import csv
import os
import atexit
from PyQt5 import QtWidgets, QtCore
import pyqtgraph as pg
# =====================================================
# CONFIG
# =====================================================
GROUP = "239.1.1.1"
PORT = 5005
HEADER_FMT = "!BBBBIQB3x"
HEADER_SIZE = struct.calcsize(HEADER_FMT)
TIME_WINDOW = 10
RIGHT_MARGIN = 2
HISTORY_SECONDS = 120
MAX_BUFFER_POINTS = 200000
MIN_VALUE_CHANGE = 0.5
ALL_WINDOWS = []
GLOBAL_PAUSED = False
TARGET_COLORS = [
"y", # Target 1
"b", # Target 2
"g", # Target 3
"r", # Target 4
"c", # Target 5
"m", # Target 6
"w", # Target 7
(255,165,0), # Target 8 orange
(128,0,128), # Target 9 purple
(0,255,127) # Target 10 spring green
]
# =====================================================
# ICD
# =====================================================
ICD = {
104: {
"type": "dynamic_array",
"count_fmt": "B",
"name" : "Target",
"element": {
"type": "mixed",
"blocks": [
{
"type": "struct",
"fmt": "!BH",
"fields": [
("id", None),
("range", 0.5),
]
},
{
"type": "dynamic_array",
"count_fmt": "B",
"element": {
"type": "struct",
"fmt": "!fB",
"fields": [
("velocity", 0.1),
("status", {
"enum": {
0: "FREE",
1: "LOCKED"
}
})
]
}
}
]
}
},
105: {
"type": "mixed",
"blocks": [
{
"type": "struct",
"fmt": "!ﬀf",
"fields": [
("position.lat", None),
("position.lon", None),
("position.alt", None),
]
},
{
"type": "struct",
"fmt": "!BH",
"fields": [
("health.mode", {
"enum": {
0: "OFF",
1: "STANDBY",
2: "TRACK",
3: "ENGAGE"
}
}),
("health.temperature", 0.1),
]
}
]
}
}
# =====================================================
# PLAYBACK CLOCK
# =====================================================
class PlaybackClock:
def __init__(self):
self.time = None
self.speed = 0.1
self.last = time.time()
self.lock = threading.Lock()
def update(self):
now = time.time()
dt = now - self.last
self.last = now
with self.lock:
if self.time is not None:
self.time += dt * self.speed
def initialize(self, ts):
with self.lock:
if self.time is None:
self.time = ts
def get(self):
with self.lock:
return self.time
def set_speed(self, s):
with self.lock:
self.speed = s
def reset_reference(self):
with self.lock:
self.last = time.time()
playback = PlaybackClock()
# =====================================================
# STORE
# =====================================================
class Store:
def __init__(self):
self.lock = threading.Lock()
self.data = defaultdict(
lambda: defaultdict(lambda: deque(maxlen=MAX_BUFFER_POINTS))
)
self.packet_history = deque(maxlen=MAX_BUFFER_POINTS)
def add(self, opcode, decoded, ts, packet_no):
playback.initialize(ts)
with self.lock:
self.packet_history.append((ts, packet_no))
for k, v in decoded.items():
self.data[opcode][k].append((ts, v))
def prune(self):
play_t = playback.get()
if play_t is None:
return
cutoﬀ = play_t - HISTORY_SECONDS
with self.lock:
for op in self.data:
for p in self.data[op]:
dq = self.data[op][p]
while dq and dq[0][0] < cutoﬀ:
dq.popleft()
def params(self, opcode):
with self.lock:
return sorted(self.data[opcode].keys())
def series(self, opcode, param):
with self.lock:
arr = list(self.data[opcode][param])
if not arr:
return [], []
x, y = zip(*arr)
return list(x), list(y)
def packet_at(self, play_t):
with self.lock:
last = None
for ts, p in self.packet_history:
if ts <= play_t:
last = p
else:
break
return last
store = Store()
# =====================================================
# LOGGER
# =====================================================
class TelemetryLogger:
def __init__(self):
os.makedirs("logs", exist_ok=True)
self.filename = time.strftime(
"logs/telemetry_%Y%m%d_%H%M%S.csv"
)
self.rows = []
self.columns = ["timestamp", "packet_no", "opcode"]
self.lock = threading.Lock()
print("Logging to", self.filename)
def log(self, packet_no, opcode, decoded, labels, ts):
with self.lock:
row = {
"timestamp": ts,
"packet_no": packet_no,
"opcode": opcode
}
# store numeric values
for k, v in decoded.items():
if k not in self.columns:
self.columns.append(k)
row[k] = v
# store enum labels separately
for k, v in labels.items():
label_key = f"{k}_label"
if label_key not in self.columns:
self.columns.append(label_key)
row[label_key] = v
self.rows.append(row)
def close(self):
print("Saving telemetry log...")
with self.lock:
with open(self.filename, "w", newline="") as f:
writer = csv.DictWriter(
f,
fieldnames=self.columns
)
writer.writeheader()
writer.writerows(self.rows)
print("Log saved successfully.")
logger = TelemetryLogger()
atexit.register(logger.close)
# =====================================================
# SAFE ICD DECODER
# =====================================================
# =====================================================
# SAFE ICD DECODER (supports count_fmt and count_from)
# =====================================================
def apply_field_rule(rule, value):
if isinstance(rule, dict) and "enum" in rule:
return value, rule["enum"].get(value, f"UNKNOWN({value})")
if isinstance(rule, (int, float)):
return value * rule, None
return value, None
def decode_struct(icd, payload, oﬀset, prefix, context):
decoded, labels = {}, {}
size = struct.calcsize(icd["fmt"])
if oﬀset + size > len(payload):
return decoded, labels, len(payload)
values = struct.unpack_from(icd["fmt"], payload, oﬀset)
oﬀset += size
for (name, rule), val in zip(icd["fields"], values):
numeric, label = apply_field_rule(rule, val)
key = prefix + name
decoded[key] = numeric
# store in context so later arrays can use it
context[name] = numeric
if label:
labels[key] = label
return decoded, labels, oﬀset
def decode_dynamic(icd, payload, oﬀset, prefix, context):
decoded, labels = {}, {}
# determine array count
if "count_fmt" in icd:
csize = struct.calcsize(icd["count_fmt"])
if oﬀset + csize > len(payload):
return decoded, labels, len(payload)
n = struct.unpack_from(icd["count_fmt"], payload, oﬀset)[0]
oﬀset += csize
elif "count_from" in icd:
n = int(context.get(icd["count_from"], 0))
else:
n = 0
array_name = icd.get("name", "")
for i in range(n):
if oﬀset >= len(payload):
break
if array_name:
sub_prefix = f"{prefix}{array_name}[{i+1}]."
else:
sub_prefix = f"{prefix}{i+1}."
d, l, oﬀset = decode_block(
icd["element"], payload, oﬀset, sub_prefix, context
)
decoded.update(d)
labels.update(l)
return decoded, labels, oﬀset
def decode_mixed(icd, payload, oﬀset, prefix, context):
decoded, labels = {}, {}
for block in icd["blocks"]:
if oﬀset >= len(payload):
break
d, l, oﬀset = decode_block(
block, payload, oﬀset, prefix, context
)
decoded.update(d)
labels.update(l)
return decoded, labels, oﬀset
def decode_block(icd, payload, oﬀset, prefix, context):
t = icd["type"]
if t == "struct":
return decode_struct(icd, payload, oﬀset, prefix, context)
if t == "dynamic_array":
return decode_dynamic(icd, payload, oﬀset, prefix, context)
if t == "mixed":
return decode_mixed(icd, payload, oﬀset, prefix, context)
return {}, {}, oﬀset
def decode_from_icd(opcode, payload):
icd = ICD.get(opcode)
if not icd:
return {}, {}
context = {}
decoded, labels, _ = decode_block(icd, payload, 0, "", context)
return decoded, labels
# =====================================================
# RECEIVER
# =====================================================
def receiver():
sock = socket.socket(
socket.AF_INET,
socket.SOCK_DGRAM,
socket.IPPROTO_UDP
)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(("", PORT))
mreq = struct.pack(
"4sl",
socket.inet_aton(GROUP),
socket.INADDR_ANY
)
sock.setsockopt(
socket.IPPROTO_IP,
socket.IP_ADD_MEMBERSHIP,
mreq
)
print("Listening multicast...")
while True:
pkt, _ = sock.recvfrom(65535)
if len(pkt) < HEADER_SIZE:
continue
header = struct.unpack(
HEADER_FMT,
pkt[:HEADER_SIZE]
)
packet_no = header[0]
opcode = header[1]
payload_len = header[4]
timestamp_us = header[5]
ts = timestamp_us / 1e6
payload = pkt[HEADER_SIZE:HEADER_SIZE + payload_len]
try:
decoded, labels = decode_from_icd(opcode, payload)
except Exception as e:
print(f"Decode error opcode {opcode}: {e}")
continue
store.add(opcode, decoded, ts, packet_no)
logger.log(packet_no, opcode, decoded, labels, ts)
# =====================================================
# UI
# =====================================================
class PlotWindow(QtWidgets.QWidget):
def __init__(self):
super().__init__()
ALL_WINDOWS.append(self)
self.window_paused = False
self.setWindowTitle("Telemetry Viewer")
layout = QtWidgets.QVBoxLayout(self)
row2 = QtWidgets.QHBoxLayout()
# ----------------------------
# Left side controls
# ----------------------------
self.add_plot_btn = QtWidgets.QPushButton("Add New Plot")
self.speed = QtWidgets.QComboBox()
self.speed.addItems(["0.05x","0.1x","0.25x","0.5x","1x"])
current_speed = playback.speed
for i in range(self.speed.count()):
text_value = float(self.speed.itemText(i).replace("x", ""))
if text_value == current_speed:
self.speed.setCurrentIndex(i)
break
self.new_window_btn = QtWidgets.QPushButton("Open New Window")
row2.addWidget(self.add_plot_btn)
row2.addWidget(self.speed)
row2.addWidget(self.new_window_btn)
# push pause buttons to right
row2.addStretch()
# ----------------------------
# Right side controls
# ----------------------------
self.pause_window_btn = QtWidgets.QPushButton("Pause Window")
self.pause_btn = QtWidgets.QPushButton("Pause All")
if GLOBAL_PAUSED:
self.pause_btn.setText("Resume All")
self.close_btn = QtWidgets.QPushButton("Close All")
row2.addWidget(self.pause_window_btn)
row2.addWidget(self.pause_btn)
row2.addWidget(self.close_btn)
layout.addLayout(row2)
# ---------------------------------------------
# Dynamic Plot Container (for + / - plots)
# ---------------------------------------------
self.plot_container = QtWidgets.QVBoxLayout()
layout.addLayout(self.plot_container)
self.plots = []
self.curves = []
self.param_boxes = []
self.value_labels = []
self.packet_labels = []
self.remove_buttons = []
self.opcode_boxes = []
self.last_plotted_values = []
self.add_plot()
self.timer = QtCore.QTimer()
self.timer.timeout.connect(self.update_plot)
self.timer.start(40)
self.pause_btn.clicked.connect(self.toggle_pause)
self.close_btn.clicked.connect(self.close_all)
self.pause_window_btn.clicked.connect(self.toggle_window_pause)
self.new_window_btn.clicked.connect(self.open_new_window)
self.speed.currentTextChanged.connect(self.change_speed)
self.add_plot_btn.clicked.connect(self.add_plot)
def add_plot(self):
axis = pg.DateAxisItem(orientation='bottom')
plot = pg.PlotWidget(
axisItems={'bottom': axis}
)
plot.setClipToView(True)
plot.setDownsampling(mode='peak')
plot.enableAutoRange(axis='y')
# Initialize X axis once
plot.setAutoVisible(x=True)
plot.setAutoVisible(y=True)
plot.enableAutoRange(axis='x')
plot.enableAutoRange(axis='x', enable=False)
curve = plot.plot()
# --------------------------------
# Controls
# --------------------------------
opcode_box = QtWidgets.QComboBox()
param_box = QtWidgets.QComboBox()
opcode_box.setMinimumWidth(120)
param_box.setSizePolicy(
QtWidgets.QSizePolicy.Expanding,
QtWidgets.QSizePolicy.Fixed
)
param_box.setMinimumWidth(250)
# When opcode changes → refresh params
opcode_box.currentIndexChanged.connect(
lambda _, ob=opcode_box, pb=param_box:
self.update_param_box(ob, pb)
)
# --------------------------------
# Status labels
# --------------------------------
value_label = QtWidgets.QLabel("Current Value: ---
")
packet_label = QtWidgets.QLabel("Packet No: ---
")
# --------------------------------
# Layout row
# --------------------------------
top_row = QtWidgets.QHBoxLayout()
remove_btn = QtWidgets.QPushButton("
-
")
remove_btn.setFixedWidth(35)
top_row.addWidget(opcode_box, 2)
top_row.addWidget(param_box, 4)
top_row.addWidget(value_label, 3)
top_row.addWidget(packet_label, 3)
top_row.addWidget(remove_btn)
# --------------------------------
# Add to layout
# --------------------------------
self.plot_container.addLayout(top_row)
self.plot_container.addWidget(plot)
remove_btn.clicked.connect(
lambda _, btn=remove_btn:
self.remove_specific_plot(btn)
)
# --------------------------------
# Store references
# --------------------------------
self.plots.append(plot)
self.curves.append(curve)
self.param_boxes.append(param_box)
self.value_labels.append(value_label)
self.packet_labels.append(packet_label)
self.remove_buttons.append(remove_btn)
self.opcode_boxes.append(opcode_box)
self.last_plotted_values.append(None)
def remove_specific_plot(self, button):
if button not in self.remove_buttons:
return
idx = self.remove_buttons.index(button)
opcode_box = self.opcode_boxes.pop(idx)
self.last_plotted_values.pop(idx)
opcode_box.setParent(None)
opcode_box.deleteLater()
plot = self.plots.pop(idx)
curve = self.curves.pop(idx)
param_box = self.param_boxes.pop(idx)
value_label = self.value_labels.pop(idx)
packet_label = self.packet_labels.pop(idx)
remove_btn = self.remove_buttons.pop(idx)
plot.setParent(None)
param_box.setParent(None)
value_label.setParent(None)
packet_label.setParent(None)
remove_btn.setParent(None)
plot.deleteLater()
param_box.deleteLater()
value_label.deleteLater()
packet_label.deleteLater()
remove_btn.deleteLater()
def toggle_window_pause(self):
self.window_paused = not self.window_paused
# prevent playback jump
playback.reset_reference()
if self.window_paused:
self.pause_window_btn.setText("Resume Window")
else:
self.pause_window_btn.setText("Pause Window")
def toggle_pause(self):
global GLOBAL_PAUSED
GLOBAL_PAUSED = not GLOBAL_PAUSED
# ✅ Reset clock reference to avoid jump
playback.reset_reference()
text = "Resume All" if GLOBAL_PAUSED else "Pause All"
for w in ALL_WINDOWS:
w.pause_btn.setText(text)
def close_all(self):
#ALL_WINDOWS.clear()
QtWidgets.QApplication.quit()
def change_speed(self):
# Get selected speed
speed_value = float(self.speed.currentText().replace("x", ""))
# Update global playback speed
playback.set_speed(speed_value)
# Update all other windows dropdowns
for w in ALL_WINDOWS:
if w is not self:
w.speed.blockSignals(True)
w.speed.setCurrentText(self.speed.currentText())
w.speed.blockSignals(False)
def update_param_box(self, opcode_box, param_box):
if not opcode_box.currentText():
return
op = int(opcode_box.currentText())
params = store.params(op)
current = param_box.currentText()
param_box.blockSignals(True)
param_box.clear()
param_box.addItem("")
param_box.addItems(params)
if current in params:
param_box.setCurrentText(current)
param_box.blockSignals(False)
def open_new_window(self):
win = PlotWindow()
win.resize(900, 600)
win.show()
def closeEvent(self, event):
if self in ALL_WINDOWS:
ALL_WINDOWS.remove(self)
super().closeEvent(event)
def update_plot(self):
# -----------------------------------------
# Playback update
# -----------------------------------------
if not GLOBAL_PAUSED :
playback.update()
store.prune()
# -----------------------------------------
# Refresh opcode list
# -----------------------------------------
for box in self.opcode_boxes:
existing = [
box.itemText(i)
for i in range(box.count())
]
for op in store.data.keys():
if str(op) not in existing:
box.addItem(str(op))
# -----------------------------------------
# Playback time
# -----------------------------------------
play_t = playback.get()
if play_t is None:
return
if self.window_paused:
return
latest_time = play_t
last_value = None
last_packet = None
# -----------------------------------------
# Update EACH plot dynamically
# -----------------------------------------
for i, (opcode_box, combo, plot, curve, value_label, packet_label) in
enumerate(zip(
self.opcode_boxes,
self.param_boxes,
self.plots,
self.curves,
self.value_labels,
self.packet_labels
)):
param = combo.currentText()
opcode_text = opcode_box.currentText()
if not opcode_text:
continue
op = int(opcode_text)
if not param:
curve.setData([], [])
value_label.setText("Current Value: ---
")
packet_label.setText("Packet No: ---
")
continue
x, y = store.series(op, param)
if not y:
curve.setData([], [])
continue
# -----------------------------------------
# Apply playback tolerance
# -----------------------------------------
fx = []
fy = []
last_value = self.last_plotted_values[i]
for t, v in zip(x, y):
if t > play_t + 0.05:
continue
if last_value is None:
fx.append(t)
fy.append(v)
last_value = v
continue
if abs(v - last_value) >= MIN_VALUE_CHANGE:
fx.append(t)
fy.append(v)
last_value = v
if not fy:
curve.setData([], [])
value_label.setText("Current Value: ---
")
packet_label.setText("Packet No: ---
")
continue
# -----------------------------------------
# Detect target color
# -----------------------------------------
color = "c"
import re
match = re.search(r'\[(\d+)\]', param)
if match:
idx = int(match.group(1))
if 1 <= idx <= len(TARGET_COLORS):
color = TARGET_COLORS[idx - 1]
# -----------------------------------------
# Plot data
# -----------------------------------------
curve.setData(
fx,
fy,
pen=pg.mkPen(color, width=2)
)
# -----------------------------------------
# Sliding time window
# -----------------------------------------
if fy:
right = latest_time + RIGHT_MARGIN
left = right - TIME_WINDOW
plot.setXRange(left, right, padding=0)
# Track last value for display
current_value = fy[-1]
self.last_plotted_values[i] = current_value
current_packet = store.packet_at(play_t)
value_label.setText(
f"Current Value: {current_value:.3f}"
)
packet_label.setText(
f"Packet No: {current_packet}"
)
# =====================================================
# MAIN
# =====================================================
def main():
threading.Thread(target=receiver, daemon=True).start()
app = QtWidgets.QApplication([])
win = PlotWindow()
win.resize(900,600)
win.show()
app.exec_()
if __name__ == "__main__":
main()
