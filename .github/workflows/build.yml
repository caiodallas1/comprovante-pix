import tkinter as tk
from tkinter import ttk, filedialog, messagebox
import os
import shutil
import json
from datetime import datetime
import threading
import sys

# --- Tentar importar tkinterdnd2 ---
try:
    from tkinterdnd2 import TkinterDnD, DND_FILES
    DND_AVAILABLE = True
except ImportError:
    DND_AVAILABLE = False

# ─────────────────────────────────────────────
CONFIG_FILE = os.path.join(os.path.expanduser("~"), ".comprovantes_pix_config.json")

MESES = {
    1: "Janeiro", 2: "Fevereiro", 3: "Março", 4: "Abril",
    5: "Maio", 6: "Junho", 7: "Julho", 8: "Agosto",
    9: "Setembro", 10: "Outubro", 11: "Novembro", 12: "Dezembro"
}

def load_config():
    if os.path.exists(CONFIG_FILE):
        with open(CONFIG_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {"pasta_base": "", "historico": []}

def save_config(cfg):
    with open(CONFIG_FILE, "w", encoding="utf-8") as f:
        json.dump(cfg, f, ensure_ascii=False, indent=2)

def get_pasta_destino(pasta_base, data=None):
    if data is None:
        data = datetime.now()
    ano  = str(data.year)
    mes  = MESES[data.month]
    dia  = str(data.day).zfill(2)
    return os.path.join(pasta_base, ano, mes, dia)

def salvar_arquivo(src_path, pasta_base, nome_custom=None):
    """Copia o arquivo para a pasta correta, retorna (destino, erro)."""
    try:
        destino_dir = get_pasta_destino(pasta_base)
        os.makedirs(destino_dir, exist_ok=True)

        nome_orig = os.path.basename(src_path)
        ext = os.path.splitext(nome_orig)[1]

        if nome_custom and nome_custom.strip():
            nome_final = nome_custom.strip()
            if not nome_final.lower().endswith(ext.lower()):
                nome_final += ext
        else:
            nome_final = nome_orig

        # Evitar sobrescrita
        destino = os.path.join(destino_dir, nome_final)
        if os.path.exists(destino):
            base, ex = os.path.splitext(nome_final)
            ts = datetime.now().strftime("%H%M%S")
            nome_final = f"{base}_{ts}{ex}"
            destino = os.path.join(destino_dir, nome_final)

        shutil.copy2(src_path, destino)
        return destino, None
    except Exception as e:
        return None, str(e)

# ─────────────────────────────────────────────
class App:
    def __init__(self):
        self.cfg = load_config()
        self.arquivos_pendentes = []  # lista de paths arrastados

        # Janela principal
        if DND_AVAILABLE:
            self.root = TkinterDnD.Tk()
        else:
            self.root = tk.Tk()

        self.root.title("Comprovantes PIX")
        self.root.geometry("560x640")
        self.root.resizable(False, False)
        self.root.configure(bg="#0f1117")

        self._build_ui()

        if DND_AVAILABLE:
            self._setup_dnd()

        self.root.mainloop()

    # ── UI ─────────────────────────────────────
    def _build_ui(self):
        root = self.root
        BG   = "#0f1117"
        CARD = "#1a1d27"
        ACC  = "#00d4aa"
        ACC2 = "#0099ff"
        TXT  = "#e8eaf0"
        MUTED= "#6b7280"

        # Título
        hdr = tk.Frame(root, bg=BG, pady=14)
        hdr.pack(fill="x")
        tk.Label(hdr, text="💸", font=("Segoe UI Emoji", 28), bg=BG).pack()
        tk.Label(hdr, text="Comprovantes PIX", font=("Segoe UI", 18, "bold"),
                 bg=BG, fg=TXT).pack()
        tk.Label(hdr, text="Arraste comprovantes e salve automaticamente por data",
                 font=("Segoe UI", 9), bg=BG, fg=MUTED).pack()

        # ── Pasta base ──
        pf = tk.Frame(root, bg=CARD, padx=16, pady=12)
        pf.pack(fill="x", padx=20, pady=(0, 10))

        tk.Label(pf, text="📁  Pasta de destino", font=("Segoe UI", 10, "bold"),
                 bg=CARD, fg=ACC).pack(anchor="w")

        row = tk.Frame(pf, bg=CARD)
        row.pack(fill="x", pady=(6, 0))

        self.pasta_var = tk.StringVar(value=self.cfg.get("pasta_base", ""))
        entry_pasta = tk.Entry(row, textvariable=self.pasta_var,
                               font=("Segoe UI", 9), bg="#252836",
                               fg=TXT, insertbackground=TXT,
                               relief="flat", bd=0)
        entry_pasta.pack(side="left", fill="x", expand=True,
                         ipady=6, ipadx=8)

        tk.Button(row, text="Escolher", font=("Segoe UI", 9, "bold"),
                  bg=ACC2, fg="white", relief="flat", bd=0,
                  activebackground="#007acc", cursor="hand2",
                  command=self._escolher_pasta,
                  padx=12, pady=4).pack(side="left", padx=(8, 0))

        # Preview do caminho
        self.preview_var = tk.StringVar()
        self._atualizar_preview()
        tk.Label(pf, textvariable=self.preview_var,
                 font=("Segoe UI", 8), bg=CARD, fg=MUTED,
                 anchor="w").pack(anchor="w", pady=(4, 0))

        self.pasta_var.trace_add("write", lambda *_: self._atualizar_preview())

        # ── Nome customizado ──
        nf = tk.Frame(root, bg=CARD, padx=16, pady=12)
        nf.pack(fill="x", padx=20, pady=(0, 10))

        tk.Label(nf, text="✏️  Nome do arquivo (opcional)",
                 font=("Segoe UI", 10, "bold"), bg=CARD, fg=ACC).pack(anchor="w")
        tk.Label(nf, text="Deixe em branco para manter o nome original",
                 font=("Segoe UI", 8), bg=CARD, fg=MUTED).pack(anchor="w", pady=(2, 6))

        self.nome_var = tk.StringVar()
        tk.Entry(nf, textvariable=self.nome_var,
                 font=("Segoe UI", 10), bg="#252836",
                 fg=TXT, insertbackground=TXT,
                 relief="flat", bd=0).pack(fill="x", ipady=7, ipadx=8)

        # ── Zona de drop ──
        self.drop_frame = tk.Frame(root, bg=CARD, bd=2, relief="flat",
                                   padx=10, pady=10)
        self.drop_frame.pack(fill="x", padx=20, pady=(0, 10))

        self.drop_inner = tk.Frame(self.drop_frame, bg="#252836",
                                   pady=30)
        self.drop_inner.pack(fill="both", expand=True)

        self.drop_icon  = tk.Label(self.drop_inner, text="⬇️",
                                   font=("Segoe UI Emoji", 32), bg="#252836")
        self.drop_icon.pack()

        self.drop_label = tk.Label(self.drop_inner,
                                   text="Arraste os comprovantes aqui" if DND_AVAILABLE
                                   else "Drag & Drop não disponível\nUse o botão abaixo",
                                   font=("Segoe UI", 11), bg="#252836", fg=TXT,
                                   justify="center")
        self.drop_label.pack(pady=(6, 0))

        self.drop_sub = tk.Label(self.drop_inner,
                                 text="PDF, PNG, JPG, JPEG aceitos",
                                 font=("Segoe UI", 8), bg="#252836", fg=MUTED)
        self.drop_sub.pack()

        # Botão alternativo
        btn_row = tk.Frame(root, bg=BG)
        btn_row.pack(fill="x", padx=20, pady=(0, 8))

        tk.Button(btn_row, text="📂  Selecionar arquivo(s)",
                  font=("Segoe UI", 10), bg="#252836", fg=TXT,
                  relief="flat", bd=0, cursor="hand2",
                  activebackground="#3a3d4f",
                  command=self._selecionar_arquivos,
                  pady=8).pack(fill="x")

        # ── Log ──
        lf = tk.Frame(root, bg=BG)
        lf.pack(fill="both", expand=True, padx=20, pady=(0, 14))

        tk.Label(lf, text="📋  Histórico",
                 font=("Segoe UI", 9, "bold"), bg=BG, fg=MUTED).pack(anchor="w")

        log_bg = tk.Frame(lf, bg=CARD)
        log_bg.pack(fill="both", expand=True, pady=(4, 0))

        self.log_text = tk.Text(log_bg, height=8, bg=CARD, fg=TXT,
                                font=("Consolas", 8), relief="flat",
                                bd=0, state="disabled",
                                wrap="word", padx=8, pady=6,
                                insertbackground=TXT)
        self.log_text.pack(fill="both", expand=True)

        sb = tk.Scrollbar(log_bg, command=self.log_text.yview, bg=CARD)
        self.log_text.configure(yscrollcommand=sb.set)

        self.log_text.tag_config("ok",    foreground="#00d4aa")
        self.log_text.tag_config("erro",  foreground="#ff5555")
        self.log_text.tag_config("info",  foreground="#6b7280")

        self._log("App iniciado. Pronto para salvar comprovantes.", "info")
        for h in self.cfg.get("historico", [])[-5:]:
            self._log(h, "ok")

    def _setup_dnd(self):
        for w in (self.drop_inner, self.drop_label, self.drop_icon, self.drop_sub):
            w.drop_target_register(DND_FILES)
            w.dnd_bind("<<Drop>>", self._on_drop)
            w.dnd_bind("<<DragEnter>>", self._on_drag_enter)
            w.dnd_bind("<<DragLeave>>", self._on_drag_leave)

    # ── Eventos ────────────────────────────────
    def _on_drag_enter(self, event):
        self.drop_inner.configure(bg="#1e3a3a")
        for w in (self.drop_label, self.drop_icon, self.drop_sub):
            w.configure(bg="#1e3a3a")

    def _on_drag_leave(self, event):
        self.drop_inner.configure(bg="#252836")
        for w in (self.drop_label, self.drop_icon, self.drop_sub):
            w.configure(bg="#252836")

    def _on_drop(self, event):
        self._on_drag_leave(None)
        raw = event.data
        # tkinterdnd2 encapsula paths com {} se tiver espaço
        paths = self.root.tk.splitlist(raw)
        self._processar_arquivos(list(paths))

    def _escolher_pasta(self):
        p = filedialog.askdirectory(title="Escolha a pasta base de comprovantes")
        if p:
            self.pasta_var.set(p)
            self.cfg["pasta_base"] = p
            save_config(self.cfg)

    def _selecionar_arquivos(self):
        files = filedialog.askopenfilenames(
            title="Selecionar comprovantes",
            filetypes=[("Imagens e PDF", "*.pdf *.png *.jpg *.jpeg"),
                       ("Todos", "*.*")]
        )
        if files:
            self._processar_arquivos(list(files))

    def _atualizar_preview(self):
        base = self.pasta_var.get()
        if base:
            p = get_pasta_destino(base)
            self.preview_var.set(f"→ {p}")
        else:
            self.preview_var.set("→ Nenhuma pasta selecionada")

    # ── Salvar ─────────────────────────────────
    def _processar_arquivos(self, paths):
        pasta_base = self.pasta_var.get().strip()
        if not pasta_base:
            messagebox.showwarning("Pasta não definida",
                                   "Selecione a pasta de destino primeiro!")
            return

        nome = self.nome_var.get().strip()

        def _run():
            for path in paths:
                path = path.strip().strip("{}")
                if not os.path.isfile(path):
                    self._log(f"❌  Não encontrado: {path}", "erro")
                    continue

                destino, erro = salvar_arquivo(path, pasta_base, nome if len(paths) == 1 else None)
                if erro:
                    self._log(f"❌  Erro ao salvar {os.path.basename(path)}: {erro}", "erro")
                else:
                    msg = f"✅  {os.path.basename(destino)}"
                    self._log(msg, "ok")
                    hist = self.cfg.setdefault("historico", [])
                    hist.append(msg)
                    if len(hist) > 50:
                        self.cfg["historico"] = hist[-50:]
                    save_config(self.cfg)

            # Limpar nome após salvar
            self.root.after(0, lambda: self.nome_var.set(""))

        threading.Thread(target=_run, daemon=True).start()

    # ── Log ────────────────────────────────────
    def _log(self, msg, tag="info"):
        def _do():
            self.log_text.configure(state="normal")
            ts = datetime.now().strftime("%H:%M:%S")
            self.log_text.insert("end", f"[{ts}] {msg}\n", tag)
            self.log_text.see("end")
            self.log_text.configure(state="disabled")
        self.root.after(0, _do)


if __name__ == "__main__":
    App()
