import tkinter as tk
from tkinter import ttk, messagebox, simpledialog
import json
import hashlib
import os
import collections
import time
import threading

# ══════════════════════════════════════════════
#  CẤU HÌNH MÀU SẮC & HẰNG SỐ
# ══════════════════════════════════════════════
BG         = "#1e1e2e"
BG2        = "#2a2a3e"
ACCENT     = "#7c3aed"
ACCENT2    = "#a78bfa"
BTN_BG     = "#3730a3"
BTN_HOV    = "#4f46e5"
TEXT       = "#f1f5f9"
TEXT2      = "#94a3b8"
ERR        = "#ef4444"
OK         = "#22c55e"
WARN       = "#f59e0b"

C_FREE     = "#334155"   # ô trống
C_WALL     = "#0f172a"   # tường
C_VISITED  = "#1d4ed8"   # BFS đã duyệt
C_PATH     = "#fbbf24"   # đường ngắn nhất
C_ALT      = "#7c3aed"   # đường DFS khác
C_AGENT    = "#f97316"   # agent
C_START    = "#16a34a"   # điểm bắt đầu
C_END      = "#dc2626"   # điểm kết thúc
C_BORDER   = "#475569"

DIRS       = [(-1,0),(1,0),(0,-1),(0,1)]
ACCOUNTS_FILE = "accounts.json"

SAMPLE_GRID = [
    [0,1,0,0,0,0],
    [0,1,0,1,1,0],
    [0,0,0,1,0,0],
    [1,1,0,0,0,1],
    [0,0,1,1,0,0],
    [0,0,0,0,0,0],
]

# ══════════════════════════════════════════════
#  QUẢN LÝ TÀI KHOẢN
# ══════════════════════════════════════════════

def load_accounts():
    if os.path.exists(ACCOUNTS_FILE):
        with open(ACCOUNTS_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {}

def save_accounts(accounts):
    with open(ACCOUNTS_FILE, "w", encoding="utf-8") as f:
        json.dump(accounts, f, indent=2, ensure_ascii=False)

def hash_pw(pw):
    return hashlib.sha256(pw.encode()).hexdigest()

# ══════════════════════════════════════════════
#  THUẬT TOÁN BFS / DFS
# ══════════════════════════════════════════════

def bfs(grid, start, end):
    rows, cols = len(grid), len(grid[0])
    queue   = collections.deque([start])
    visited = {start}
    parent  = {start: None}
    log     = [start]
    while queue:
        curr = queue.popleft()
        if curr == end:
            break
        for dr, dc in DIRS:
            nr, nc = curr[0]+dr, curr[1]+dc
            nxt = (nr, nc)
            if 0<=nr<rows and 0<=nc<cols and grid[nr][nc]==0 and nxt not in visited:
                visited.add(nxt)
                parent[nxt] = curr
                queue.append(nxt)
                log.append(nxt)
    if end not in parent:
        return None, log
    path = []
    node = end
    while node is not None:
        path.append(node)
        node = parent[node]
    path.reverse()
    return path, log

def dfs_all_paths(grid, start, end):
    rows, cols = len(grid), len(grid[0])
    all_paths = []
    visited   = set()
    def dfs(node, path):
        if node == end:
            all_paths.append(path[:])
            return
        visited.add(node)
        for dr, dc in DIRS:
            nr, nc = node[0]+dr, node[1]+dc
            nxt = (nr, nc)
            if 0<=nr<rows and 0<=nc<cols and grid[nr][nc]==0 and nxt not in visited:
                path.append(nxt)
                dfs(nxt, path)
                path.pop()
        visited.remove(node)
    dfs(start, [start])
    return all_paths

# ══════════════════════════════════════════════
#  TIỆN ÍCH WIDGET
# ══════════════════════════════════════════════

def style_btn(btn, bg=BTN_BG, fg=TEXT, font=("Segoe UI", 10, "bold")):
    btn.configure(bg=bg, fg=fg, font=font, relief="flat",
                  cursor="hand2", padx=12, pady=6, bd=0,
                  activebackground=BTN_HOV, activeforeground=TEXT)

def label(parent, text, size=10, bold=False, color=TEXT, **kw):
    w = tk.Label(parent, text=text,
                 font=("Segoe UI", size, "bold" if bold else "normal"),
                 bg=kw.pop("bg", BG), fg=color, **kw)
    return w

def entry(parent, show=None, width=24):
    e = tk.Entry(parent, font=("Segoe UI", 11), bg=BG2, fg=TEXT,
                 insertbackground=TEXT, relief="flat",
                 highlightthickness=1, highlightbackground=ACCENT,
                 highlightcolor=ACCENT2, bd=0, width=width, show=show or "")
    return e

# ══════════════════════════════════════════════
#  MÀN HÌNH ĐĂNG NHẬP / ĐĂNG KÝ
# ══════════════════════════════════════════════

class AuthWindow:
    def __init__(self, root):
        self.root  = root
        self.accounts = load_accounts()
        self.logged_in_user = None

        root.title("Pathfinder – Đăng nhập")
        root.configure(bg=BG)
        root.resizable(False, False)
        self._center(root, 440, 540)
        self._build()

    def _center(self, win, w, h):
        win.update_idletasks()
        x = (win.winfo_screenwidth()  - w) // 2
        y = (win.winfo_screenheight() - h) // 2
        win.geometry(f"{w}x{h}+{x}+{y}")

    def _build(self):
        # Header
        hdr = tk.Frame(self.root, bg=ACCENT, height=6)
        hdr.pack(fill="x")

        tk.Frame(self.root, bg=BG, height=20).pack()
        label(self.root, "🔷 BFS / DFS PATHFINDER", size=16, bold=True,
              color=ACCENT2).pack()
        label(self.root, "Đăng nhập để bắt đầu", size=10, color=TEXT2).pack(pady=(2,20))

        # Tabs
        self.tab_frame = tk.Frame(self.root, bg=BG2, bd=0)
        self.tab_frame.pack(fill="x", padx=40)
        self.tab_login  = tk.Button(self.tab_frame, text="  Đăng nhập  ",
                                     command=self._show_login)
        self.tab_signup = tk.Button(self.tab_frame, text="  Đăng ký  ",
                                     command=self._show_signup)
        for b in (self.tab_login, self.tab_signup):
            b.configure(bg=BG2, fg=TEXT2, font=("Segoe UI",10), relief="flat",
                        cursor="hand2", bd=0, pady=8,
                        activebackground=ACCENT, activeforeground=TEXT)
            b.pack(side="left", padx=2)

        # Content area
        self.content = tk.Frame(self.root, bg=BG)
        self.content.pack(fill="both", expand=True, padx=40, pady=10)

        self.msg_var = tk.StringVar()
        self.msg_lbl = tk.Label(self.root, textvariable=self.msg_var,
                                 font=("Segoe UI", 9), bg=BG, fg=ERR)
        self.msg_lbl.pack(pady=4)

        self._show_login()

    def _clear_content(self):
        for w in self.content.winfo_children():
            w.destroy()

    def _show_login(self):
        self._clear_content()
        self.msg_var.set("")
        self.tab_login.configure(bg=ACCENT, fg=TEXT)
        self.tab_signup.configure(bg=BG2, fg=TEXT2)

        label(self.content, "Tên đăng nhập", size=9, color=TEXT2, bg=BG).pack(anchor="w", pady=(14,2))
        self.login_user = entry(self.content)
        self.login_user.pack(fill="x", ipady=6)

        label(self.content, "Mật khẩu", size=9, color=TEXT2, bg=BG).pack(anchor="w", pady=(10,2))
        self.login_pw = entry(self.content, show="●")
        self.login_pw.pack(fill="x", ipady=6)

        tk.Frame(self.content, bg=BG, height=16).pack()
        btn = tk.Button(self.content, text="Đăng nhập →",
                        command=self._do_login)
        style_btn(btn, bg=ACCENT, font=("Segoe UI",11,"bold"))
        btn.pack(fill="x", ipady=6)

        self.login_pw.bind("<Return>", lambda e: self._do_login())

    def _show_signup(self):
        self._clear_content()
        self.msg_var.set("")
        self.tab_signup.configure(bg=ACCENT, fg=TEXT)
        self.tab_login.configure(bg=BG2, fg=TEXT2)

        label(self.content, "Họ tên hiển thị", size=9, color=TEXT2, bg=BG).pack(anchor="w", pady=(8,2))
        self.su_name = entry(self.content)
        self.su_name.pack(fill="x", ipady=5)

        label(self.content, "Tên đăng nhập", size=9, color=TEXT2, bg=BG).pack(anchor="w", pady=(8,2))
        self.su_user = entry(self.content)
        self.su_user.pack(fill="x", ipady=5)

        label(self.content, "Mật khẩu (≥ 6 ký tự)", size=9, color=TEXT2, bg=BG).pack(anchor="w", pady=(8,2))
        self.su_pw = entry(self.content, show="●")
        self.su_pw.pack(fill="x", ipady=5)

        label(self.content, "Nhập lại mật khẩu", size=9, color=TEXT2, bg=BG).pack(anchor="w", pady=(8,2))
        self.su_pw2 = entry(self.content, show="●")
        self.su_pw2.pack(fill="x", ipady=5)

        tk.Frame(self.content, bg=BG, height=10).pack()
        btn = tk.Button(self.content, text="Tạo tài khoản ✓",
                        command=self._do_signup)
        style_btn(btn, bg=OK, font=("Segoe UI",11,"bold"))
        btn.pack(fill="x", ipady=6)

    def _msg(self, txt, color=ERR):
        self.msg_var.set(txt)
        self.msg_lbl.configure(fg=color)

    def _do_login(self):
        u = self.login_user.get().strip()
        p = self.login_pw.get()
        if not u or not p:
            self._msg("⚠ Vui lòng nhập đủ thông tin!")
            return
        if u not in self.accounts:
            self._msg("❌ Tên đăng nhập không tồn tại!")
            return
        if self.accounts[u]["password"] != hash_pw(p):
            self._msg("❌ Sai mật khẩu!")
            return
        self.logged_in_user = u
        self._msg(f"✅ Chào {self.accounts[u]['name']}!", color=OK)
        self.root.after(700, self._open_main)

    def _do_signup(self):
        name = self.su_name.get().strip()
        u    = self.su_user.get().strip()
        p    = self.su_pw.get()
        p2   = self.su_pw2.get()
        if not name or not u or not p:
            self._msg("⚠ Vui lòng nhập đủ thông tin!")
            return
        if len(u) < 3:
            self._msg("⚠ Tên đăng nhập ít nhất 3 ký tự!")
            return
        if len(p) < 6:
            self._msg("⚠ Mật khẩu ít nhất 6 ký tự!")
            return
        if p != p2:
            self._msg("⚠ Mật khẩu không khớp!")
            return
        if u in self.accounts:
            self._msg("⚠ Tên đăng nhập đã tồn tại!")
            return
        self.accounts[u] = {
            "name": name,
            "password": hash_pw(p),
            "created": time.strftime("%Y-%m-%d %H:%M"),
            "history": []
        }
        save_accounts(self.accounts)
        self._msg(f"✅ Tạo tài khoản thành công! Hãy đăng nhập.", color=OK)
        self.root.after(1000, self._show_login)

    def _open_main(self):
        self.root.withdraw()
        main_win = tk.Toplevel(self.root)
        app = MainApp(main_win, self.logged_in_user, self.accounts, self.root)


# ══════════════════════════════════════════════
#  ỨNG DỤNG CHÍNH
# ══════════════════════════════════════════════

class MainApp:
    def __init__(self, root, username, accounts, auth_root):
        self.root       = root
        self.username   = username
        self.accounts   = accounts
        self.auth_root  = auth_root
        self.grid_data  = []
        self.start_pt   = None
        self.end_pt     = None
        self.cell_size  = 60
        self.anim_id    = None
        self.bfs_path   = None
        self.all_paths  = []
        self.visited_log= []

        root.title(f"Pathfinder – {accounts[username]['name']}")
        root.configure(bg=BG)
        root.resizable(True, True)
        self._center(root, 1100, 750)
        root.protocol("WM_DELETE_WINDOW", self._on_close)
        self._build_ui()

    def _center(self, win, w, h):
        win.update_idletasks()
        x = (win.winfo_screenwidth()  - w) // 2
        y = (win.winfo_screenheight() - h) // 2
        win.geometry(f"{w}x{h}+{x}+{y}")

    def _on_close(self):
        self.root.destroy()
        self.auth_root.destroy()

    # ── BUILD UI ──────────────────────────────
    def _build_ui(self):
        # Top bar
        topbar = tk.Frame(self.root, bg=ACCENT, height=44)
        topbar.pack(fill="x")
        topbar.pack_propagate(False)
        tk.Label(topbar, text="  🔷 BFS/DFS PATHFINDER",
                 font=("Segoe UI",13,"bold"), bg=ACCENT, fg=TEXT).pack(side="left", padx=10)
        user_name = self.accounts[self.username]["name"]
        tk.Label(topbar, text=f"👤 {user_name}",
                 font=("Segoe UI",10), bg=ACCENT, fg=TEXT).pack(side="right", padx=16)

        # Main layout
        main = tk.Frame(self.root, bg=BG)
        main.pack(fill="both", expand=True)

        # Left panel – controls
        left = tk.Frame(main, bg=BG2, width=280)
        left.pack(side="left", fill="y", padx=(8,4), pady=8)
        left.pack_propagate(False)
        self._build_left(left)

        # Right panel – canvas + log
        right = tk.Frame(main, bg=BG)
        right.pack(side="left", fill="both", expand=True, padx=(4,8), pady=8)
        self._build_right(right)

    # ── LEFT PANEL ────────────────────────────
    def _build_left(self, parent):
        def sec(txt):
            f = tk.Frame(parent, bg=ACCENT, height=2)
            f.pack(fill="x", padx=8, pady=(14,0))
            tk.Label(parent, text=txt, font=("Segoe UI",9,"bold"),
                     bg=BG2, fg=ACCENT2).pack(anchor="w", padx=10, pady=(2,6))

        # ─ Lưới
        sec("⬛  CÀI ĐẶT LƯỚI")
        row_f = tk.Frame(parent, bg=BG2); row_f.pack(fill="x", padx=8)
        tk.Label(row_f, text="Hàng:", font=("Segoe UI",9), bg=BG2, fg=TEXT2).pack(side="left")
        self.rows_var = tk.StringVar(value="6")
        tk.Spinbox(row_f, from_=2, to=20, textvariable=self.rows_var, width=4,
                   font=("Segoe UI",10), bg=BG, fg=TEXT, buttonbackground=ACCENT,
                   relief="flat").pack(side="left", padx=4)
        tk.Label(row_f, text="Cột:", font=("Segoe UI",9), bg=BG2, fg=TEXT2).pack(side="left", padx=(8,0))
        self.cols_var = tk.StringVar(value="6")
        tk.Spinbox(row_f, from_=2, to=20, textvariable=self.cols_var, width=4,
                   font=("Segoe UI",10), bg=BG, fg=TEXT, buttonbackground=ACCENT,
                   relief="flat").pack(side="left", padx=4)

        b1 = tk.Button(parent, text="📋  Tạo lưới trống", command=self._new_grid)
        style_btn(b1, bg=BTN_BG); b1.pack(fill="x", padx=8, pady=(6,2))
        b2 = tk.Button(parent, text="✨  Dùng lưới mẫu", command=self._load_sample)
        style_btn(b2, bg="#0f766e"); b2.pack(fill="x", padx=8, pady=2)

        # ─ Điểm A/B
        sec("📍  ĐIỂM ĐI / ĐẾN")
        grid_f = tk.Frame(parent, bg=BG2); grid_f.pack(padx=8, pady=2)
        for col, txt in enumerate(["","Hàng","Cột"]):
            tk.Label(grid_f, text=txt, font=("Segoe UI",9,"bold"),
                     bg=BG2, fg=TEXT2, width=6).grid(row=0, column=col, pady=2)
        for row_i, (lbl, col) in enumerate([("🟢 A (Bắt đầu)", C_START),
                                              ("🔴 B (Kết thúc)", C_END)], 1):
            tk.Label(grid_f, text=lbl, font=("Segoe UI",9), bg=BG2,
                     fg=col, width=14, anchor="w").grid(row=row_i, column=0, pady=3, sticky="w")
        self.ar_var = tk.StringVar(value="0")
        self.ac_var = tk.StringVar(value="0")
        self.br_var = tk.StringVar(value="5")
        self.bc_var = tk.StringVar(value="5")
        for i, (rv, cv) in enumerate([(self.ar_var, self.ac_var),
                                       (self.br_var, self.bc_var)], 1):
            tk.Spinbox(grid_f, from_=0, to=19, textvariable=rv, width=4,
                       font=("Segoe UI",10), bg=BG, fg=TEXT, relief="flat",
                       buttonbackground=ACCENT).grid(row=i, column=1, padx=4)
            tk.Spinbox(grid_f, from_=0, to=19, textvariable=cv, width=4,
                       font=("Segoe UI",10), bg=BG, fg=TEXT, relief="flat",
                       buttonbackground=ACCENT).grid(row=i, column=2, padx=4)

        # ─ Thuật toán
        sec("⚙️  THUẬT TOÁN")
        self.algo_var = tk.StringVar(value="BFS")
        f_algo = tk.Frame(parent, bg=BG2); f_algo.pack(padx=8, pady=4)
        for txt in ("BFS", "DFS", "Cả hai"):
            tk.Radiobutton(f_algo, text=txt, variable=self.algo_var, value=txt,
                           font=("Segoe UI",10), bg=BG2, fg=TEXT,
                           selectcolor=ACCENT, activebackground=BG2,
                           activeforeground=ACCENT2).pack(side="left", padx=6)

        speed_f = tk.Frame(parent, bg=BG2); speed_f.pack(fill="x", padx=8, pady=2)
        tk.Label(speed_f, text="Tốc độ:", font=("Segoe UI",9), bg=BG2, fg=TEXT2).pack(side="left")
        self.speed_var = tk.IntVar(value=80)
        tk.Scale(speed_f, from_=10, to=300, orient="horizontal",
                 variable=self.speed_var, bg=BG2, fg=TEXT, troughcolor=BG,
                 highlightthickness=0, length=120, showvalue=False).pack(side="left", padx=4)
        tk.Label(speed_f, textvariable=tk.StringVar(), font=("Segoe UI",8),
                 bg=BG2, fg=TEXT2).pack(side="left")

        b_run = tk.Button(parent, text="▶  CHẠY TÌM ĐƯỜNG", command=self._run)
        style_btn(b_run, bg=ACCENT, font=("Segoe UI",11,"bold"))
        b_run.pack(fill="x", padx=8, pady=(10,2))

        b_stop = tk.Button(parent, text="⏹  Dừng animation", command=self._stop_anim)
        style_btn(b_stop, bg="#7f1d1d")
        b_stop.pack(fill="x", padx=8, pady=2)

        b_reset = tk.Button(parent, text="🔄  Reset lưới", command=self._reset_view)
        style_btn(b_reset, bg="#374151")
        b_reset.pack(fill="x", padx=8, pady=2)

        # ─ Lịch sử
        sec("📜  LỊCH SỬ TÌM KIẾM")
        self.hist_box = tk.Text(parent, height=6, font=("Consolas",8),
                                bg=BG, fg=TEXT2, relief="flat",
                                state="disabled", wrap="word")
        self.hist_box.pack(fill="x", padx=8, pady=4)
        self._refresh_history()

    # ── RIGHT PANEL ───────────────────────────
    def _build_right(self, parent):
        # Canvas area
        self.canvas_frame = tk.Frame(parent, bg=BG)
        self.canvas_frame.pack(fill="both", expand=True)

        self.canvas = tk.Canvas(self.canvas_frame, bg=BG, highlightthickness=0)
        self.canvas.pack(fill="both", expand=True)
        self.canvas.bind("<Button-1>", self._canvas_click)

        # Bottom status bar
        bot = tk.Frame(parent, bg=BG2, height=32)
        bot.pack(fill="x", pady=(4,0))
        bot.pack_propagate(False)
        self.status_var = tk.StringVar(value="Hãy tạo hoặc chọn lưới mẫu để bắt đầu.")
        tk.Label(bot, textvariable=self.status_var,
                 font=("Segoe UI",9), bg=BG2, fg=TEXT2, anchor="w").pack(side="left", padx=10)

        # Stats bar
        stats = tk.Frame(parent, bg=BG)
        stats.pack(fill="x")
        self.stat_bfs  = self._stat_card(stats, "📏 Bước ngắn nhất (BFS)", "—")
        self.stat_dfs  = self._stat_card(stats, "🔢 Tổng đường đến B", "—")
        self.stat_visit= self._stat_card(stats, "🔍 Ô đã duyệt", "—")
        self.stat_time = self._stat_card(stats, "⏱ Thời gian (ms)", "—")

        # Log paths
        log_lbl = tk.Label(parent, text="  📋 Tất cả đường đến B (DFS) – luôn hiển thị dù chọn BFS:",
                           font=("Segoe UI",9,"bold"), bg=BG, fg=ACCENT2, anchor="w")
        log_lbl.pack(fill="x")
        log_frame = tk.Frame(parent, bg=BG); log_frame.pack(fill="x", pady=(0,4))
        self.log_box = tk.Text(log_frame, height=5, font=("Consolas",8),
                               bg=BG2, fg=TEXT, relief="flat", state="disabled")
        sb = tk.Scrollbar(log_frame, command=self.log_box.yview, bg=BG2)
        self.log_box.configure(yscrollcommand=sb.set)
        sb.pack(side="right", fill="y")
        self.log_box.pack(fill="both", expand=True)

    def _stat_card(self, parent, title, value):
        f = tk.Frame(parent, bg=BG2, relief="flat")
        f.pack(side="left", fill="x", expand=True, padx=3, pady=3)
        tk.Label(f, text=title, font=("Segoe UI",8), bg=BG2, fg=TEXT2).pack(pady=(4,0))
        var = tk.StringVar(value=value)
        tk.Label(f, textvariable=var, font=("Segoe UI",14,"bold"),
                 bg=BG2, fg=ACCENT2).pack(pady=(0,4))
        return var

    # ── GRID MANAGEMENT ───────────────────────
    def _new_grid(self):
        try:
            r = int(self.rows_var.get())
            c = int(self.cols_var.get())
        except:
            messagebox.showerror("Lỗi", "Số hàng/cột không hợp lệ!"); return
        self.grid_data = [[0]*c for _ in range(r)]
        self._draw_grid()
        self._set_status("Lưới mới tạo. Click ô để đổi 0↔1.")

    def _load_sample(self):
        self.grid_data = [row[:] for row in SAMPLE_GRID]
        self.rows_var.set(str(len(SAMPLE_GRID)))
        self.cols_var.set(str(len(SAMPLE_GRID[0])))
        self._draw_grid()
        self._set_status("Đã tải lưới mẫu 6×6.")

    def _canvas_click(self, event):
        if not self.grid_data:
            return
        cs = self.cell_size
        c = (event.x - 4) // cs
        r = (event.y - 4) // cs
        rows = len(self.grid_data)
        cols = len(self.grid_data[0])
        if 0 <= r < rows and 0 <= c < cols:
            self.grid_data[r][c] ^= 1   # toggle 0/1
            self._draw_grid()

    def _draw_grid(self, visited=None, path=None, agent=None, alt_paths=None):
        if not self.grid_data:
            return
        self.canvas.delete("all")
        rows = len(self.grid_data)
        cols = len(self.grid_data[0])

        # Tính cell size tự động
        cw = (self.canvas.winfo_width()  - 8) // cols
        ch = (self.canvas.winfo_height() - 8) // rows
        cs = max(20, min(cw, ch, 80))
        self.cell_size = cs

        visited  = set(visited or [])
        path_set = set(path or [])

        alt_set = set()
        if alt_paths:
            for ap in alt_paths:
                for cell in ap:
                    alt_set.add(cell)

        start = self._get_start()
        end   = self._get_end()

        for r in range(rows):
            for c in range(cols):
                x0 = 4 + c * cs
                y0 = 4 + r * cs
                x1 = x0 + cs - 1
                y1 = y0 + cs - 1
                pos = (r, c)

                if self.grid_data[r][c] == 1:
                    fill = C_WALL
                elif pos == start:
                    fill = C_START
                elif pos == end:
                    fill = C_END
                elif agent and pos == agent:
                    fill = C_AGENT
                elif pos in path_set:
                    fill = C_PATH
                elif pos in visited:
                    fill = C_VISITED
                elif pos in alt_set:
                    fill = C_ALT
                else:
                    fill = C_FREE

                self.canvas.create_rectangle(x0, y0, x1, y1,
                                              fill=fill, outline=C_BORDER, width=1)
                # số trong ô
                txt_col = "#94a3b8" if self.grid_data[r][c]==0 else "#334155"
                if pos == agent:
                    self.canvas.create_text(x0+cs//2, y0+cs//2, text="●",
                                             font=("Segoe UI",cs//3,"bold"), fill="white")
                elif pos == start:
                    self.canvas.create_text(x0+cs//2, y0+cs//2, text="A",
                                             font=("Segoe UI",cs//3,"bold"), fill="white")
                elif pos == end:
                    self.canvas.create_text(x0+cs//2, y0+cs//2, text="B",
                                             font=("Segoe UI",cs//3,"bold"), fill="white")
                else:
                    self.canvas.create_text(x0+cs//2, y0+cs//2,
                                             text=str(self.grid_data[r][c]),
                                             font=("Segoe UI", max(8, cs//4)), fill=txt_col)

        # Vẽ đường path
        if path and len(path) > 1:
            for i in range(len(path)-1):
                r1,c1 = path[i];   r2,c2 = path[i+1]
                px1 = 4 + c1*cs + cs//2; py1 = 4 + r1*cs + cs//2
                px2 = 4 + c2*cs + cs//2; py2 = 4 + r2*cs + cs//2
                self.canvas.create_line(px1,py1,px2,py2, fill="#fef08a",
                                         width=3, arrow=tk.LAST, arrowshape=(8,10,4))

    # ── HELPERS ───────────────────────────────
    def _get_start(self):
        try:
            r = int(self.ar_var.get()); c = int(self.ac_var.get())
            rows = len(self.grid_data); cols = len(self.grid_data[0]) if self.grid_data else 0
            if 0<=r<rows and 0<=c<cols and self.grid_data[r][c]==0:
                return (r, c)
        except: pass
        return None

    def _get_end(self):
        try:
            r = int(self.br_var.get()); c = int(self.bc_var.get())
            rows = len(self.grid_data); cols = len(self.grid_data[0]) if self.grid_data else 0
            if 0<=r<rows and 0<=c<cols and self.grid_data[r][c]==0:
                return (r, c)
        except: pass
        return None

    def _set_status(self, txt):
        self.status_var.set(txt)

    def _stop_anim(self):
        if self.anim_id:
            self.root.after_cancel(self.anim_id)
            self.anim_id = None
            self._set_status("⏹ Đã dừng animation.")

    def _reset_view(self):
        self._stop_anim()
        self._draw_grid()
        for v in (self.stat_bfs, self.stat_dfs, self.stat_visit, self.stat_time):
            v.set("—")
        self._set_status("Đã reset. Nhấn ▶ để chạy lại.")

    def _refresh_history(self):
        hist = self.accounts[self.username].get("history", [])
        self.hist_box.configure(state="normal")
        self.hist_box.delete("1.0", "end")
        if not hist:
            self.hist_box.insert("end", "(Chưa có lịch sử)")
        else:
            for h in reversed(hist[-10:]):
                self.hist_box.insert("end", h + "\n")
        self.hist_box.configure(state="disabled")

    def _add_history(self, entry_txt):
        h = self.accounts[self.username].setdefault("history", [])
        h.append(entry_txt)
        save_accounts(self.accounts)
        self._refresh_history()

    def _write_log(self, txt):
        self.log_box.configure(state="normal")
        self.log_box.delete("1.0", "end")
        self.log_box.insert("end", txt)
        self.log_box.configure(state="disabled")

    # ── RUN ───────────────────────────────────
    def _run(self):
        if not self.grid_data:
            messagebox.showwarning("Chưa có lưới", "Hãy tạo lưới trước!"); return
        start = self._get_start()
        end   = self._get_end()
        if not start:
            messagebox.showwarning("Lỗi", "Điểm A không hợp lệ hoặc nằm trên tường!"); return
        if not end:
            messagebox.showwarning("Lỗi", "Điểm B không hợp lệ hoặc nằm trên tường!"); return
        if start == end:
            messagebox.showwarning("Lỗi", "Điểm A và B phải khác nhau!"); return

        self._stop_anim()
        algo = self.algo_var.get()

        # ── BFS: luôn chạy để tìm đường ngắn nhất ──────────────
        t0 = time.perf_counter()
        bfs_path, visited_log = bfs(self.grid_data, start, end)
        t_bfs = time.perf_counter() - t0

        # ── DFS: LUÔN chạy để đếm tất cả đường (bất kể chọn BFS/DFS/Cả hai) ──
        self._set_status("🌲 Đang đếm tất cả đường có thể (DFS)...")
        self.root.update_idletasks()
        t0 = time.perf_counter()
        all_paths = dfs_all_paths(self.grid_data, start, end)
        t_dfs = time.perf_counter() - t0

        self.bfs_path    = bfs_path or []
        self.all_paths   = all_paths
        self.visited_log = visited_log

        # ── Stats ────────────────────────────────────────────────
        bfs_steps = len(bfs_path) - 1 if bfs_path else "∞"
        self.stat_bfs.set(str(bfs_steps))
        self.stat_dfs.set(str(len(all_paths)))
        self.stat_visit.set(str(len(visited_log)))
        self.stat_time.set(f"{(t_bfs + t_dfs)*1000:.1f}")

        # ── Log tất cả đường DFS ────────────────────────────────
        if all_paths:
            shortest_len = min(len(p)-1 for p in all_paths)
            lines = [
                f"  Tổng số đường: {len(all_paths)}",
                f"  Đường ngắn nhất: {bfs_steps} bước",
                f"  Đường dài nhất : {max(len(p)-1 for p in all_paths)} bước",
                f"  Đường có ≤ {shortest_len+2} bước: "
                f"{sum(1 for p in all_paths if len(p)-1 <= shortest_len+2)}",
                "─" * 48,
            ]
            for i, p in enumerate(all_paths, 1):
                tag = " ← NGẮN NHẤT" if len(p)-1 == bfs_steps else ""
                lines.append(f"[{i:3}] ({len(p)-1} bước){tag}: {' → '.join(map(str, p))}")
            self._write_log("\n".join(lines))
        else:
            self._write_log("❌ Không có đường nào từ A đến B.")

        # ── Lưu lịch sử tài khoản ──────────────────────────────
        ts = time.strftime("%m/%d %H:%M")
        self._add_history(
            f"{ts} | A={start}→B={end} | BFS:{bfs_steps}b | Tổng đường:{len(all_paths)}"
        )

        if not bfs_path:
            messagebox.showinfo("Kết quả", f"❌ Không tìm thấy đường từ {start} đến {end}!")
            self._draw_grid()
            return

        # Animation
        self._animate(start, end, visited_log, bfs_path, all_paths, algo)

    # ── ANIMATION ─────────────────────────────
    def _animate(self, start, end, visited_log, bfs_path, all_paths, algo):
        speed = max(10, self.speed_var.get())

        # Pha 1 – BFS duyệt
        phase1_len  = len(visited_log)
        # Pha 2 – Hiện đường BFS (pause)
        phase2_end  = phase1_len + 15
        # Pha 3 – DFS paths
        phase3_end  = phase2_end + (15 if all_paths else 0)
        # Pha 4 – Agent di chuyển
        phase4_len  = len(bfs_path)
        phase4_end  = phase3_end + phase4_len + 15
        total = phase4_end

        frame = [0]

        def step():
            f = frame[0]
            if f >= total:
                self._draw_grid(path=bfs_path,
                                alt_paths=all_paths if algo in ("DFS","Cả hai") else None)
                n = len(all_paths)
                self._set_status(
                    f"✅ Hoàn tất!  BFS ngắn nhất: {len(bfs_path)-1} bước  |  "
                    f"Tổng đường đến B: {n} đường"
                )
                return

            if f < phase1_len:
                self._draw_grid(visited=visited_log[:f+1])
                self._set_status(f"🔍 BFS đang duyệt... ô [{f+1}/{phase1_len}]")

            elif f < phase2_end:
                self._draw_grid(visited=visited_log, path=bfs_path)
                self._set_status(f"✅ BFS hoàn tất! Đường ngắn nhất: {len(bfs_path)-1} bước")

            elif f < phase3_end:
                self._draw_grid(path=bfs_path,
                                alt_paths=all_paths if all_paths else None)
                self._set_status(f"🌲 DFS tìm thấy {len(all_paths)} đường đi (màu tím)")

            else:
                move_i = f - phase3_end
                if move_i < phase4_len:
                    agent = bfs_path[move_i]
                    self._draw_grid(path=bfs_path, agent=agent,
                                    alt_paths=all_paths if algo in ("DFS","Cả hai") else None)
                    self._set_status(f"🚀 Agent di chuyển... bước {move_i}/{len(bfs_path)-1}  {agent}")
                else:
                    self._draw_grid(path=bfs_path)
                    self._set_status(f"🏁 Đã đến đích! Tổng: {len(bfs_path)-1} bước")

            frame[0] += 1
            self.anim_id = self.root.after(speed, step)

        step()

# ══════════════════════════════════════════════
#  ENTRY POINT
# ══════════════════════════════════════════════

if __name__ == "__main__":
    root = tk.Tk()
    app  = AuthWindow(root)
    root.mainloop()
