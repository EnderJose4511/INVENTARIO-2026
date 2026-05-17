import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import sqlite3
from datetime import datetime
from fpdf import FPDF
from tkinter import filedialog
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.lib import colors
from reportlab.lib.styles import getSampleStyleSheet
import random
import os
from PIL import Image, ImageTk
# =================================================
# PARTE 1: BASE DE DATOS Y LÓGICA
# =================================================

DB_NAME = "inventario_motos.db"
DB_VENTAS = "ventas_motos.db"
DB_CREDITOS = "creditos_motos.db"
DB_COMBOS = "combos_motos.db"
DB_CLIENTES = "clientes_motos.db"


def inicializar_db():
    conn = sqlite3.connect(DB_NAME)
    cur = conn.cursor()
    cur.execute(
        """
        CREATE TABLE IF NOT EXISTS repuestos (
            codigo TEXT PRIMARY KEY,
            nombre TEXT,
            modelo TEXT,
            marca TEXT,
            precio_costo REAL,
            precio_venta REAL,
            cantidad INTEGER,
            fecha TEXT
        )
        """
    )

    cur.execute("CREATE TABLE IF NOT EXISTS configuracion (clave TEXT PRIMARY KEY, valor REAL)")
    cur.execute("INSERT OR IGNORE INTO configuracion VALUES ('tasa_dolar', 1.0)")
    conn.commit()
    conn.close()

def inicializar_db_combos():
    conn = sqlite3.connect(DB_COMBOS)
    cur = conn.cursor()
    # Cabecera del combo
    cur.execute("""
        CREATE TABLE IF NOT EXISTS combos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            nombre TEXT UNIQUE,
            precio_venta REAL,
            costo_total REAL
        )
    """)
    # Detalle de componentes
    cur.execute("""
        CREATE TABLE IF NOT EXISTS combo_detalle (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            combo_id INTEGER,
            producto_codigo TEXT,
            cantidad INTEGER,
            FOREIGN KEY(combo_id) REFERENCES combos(id) ON DELETE CASCADE
        )
    """)
    conn.commit()
    conn.close()

def inicializar_db_ventas():
    conn = sqlite3.connect("ventas_motos.db")
    cur = conn.cursor()
    cur.execute(
        """
        CREATE TABLE IF NOT EXISTS ventas_log (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            fecha_hora TEXT,
            productos TEXT,
            total REAL,
            costo_total REAL
        )
        """
    )
    cur.execute(
        """
        CREATE TABLE IF NOT EXISTS notas_credito (
            codigo_nota TEXT PRIMARY KEY,
            factura_origen_id INTEGER,
            cliente TEXT,
            monto_a_favor REAL,
            costo_devuelto REAL,
            estado TEXT -- 'VIGENTE' o 'APLICADA'
        )
        """
    )
    conn.commit()
    conn.close()

def inicializar_db_creditos():
    conn = sqlite3.connect(DB_CREDITOS)
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS cuentas_por_cobrar (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        cliente TEXT,
        fecha_deuda TEXT,
        detalle_productos TEXT,
        monto_total REAL,      -- Lo que falta por pagar (Saldo)
        costo_total REAL,      -- Costo TOTAL original (Para referencia)
        costo_pendiente REAL   -- NUEVA: Lo que falta por asignar a finanzas
        )
    """)
    conn.commit()
    conn.close()

def inicializar_db_clientes():
    """Crea la base de datos centralizada de perfiles de clientes de forma minimalista."""
    conn = sqlite3.connect(DB_CLIENTES)
    cur = conn.cursor()
    cur.execute(
        """
        CREATE TABLE IF NOT EXISTS clientes (
            cedula TEXT PRIMARY KEY,
            nombre TEXT NOT NULL,
            fecha_registro TEXT
        )
        """
    )
    conn.commit()
    conn.close()


def consulta(query, params=(), fetch=True, db_path=DB_NAME):
    conn = sqlite3.connect(db_path, timeout=10)
    try:
        with conn: # Maneja automáticamente COMMIT y ROLLBACK
            cur = conn.cursor()
            cur.execute(query, params)
            if fetch:
                return cur.fetchall()
            return None
    finally:
        conn.close()


# =================================================
# PARTE 2: APLICACIÓN (UNA SOLA VENTANA, PANTALLAS FULL)
# =================================================

class ERPRepuestosApp:
        # -------------------------------------------------
    # PANTALLA DE VENTAS (POS)
    # -------------------------------------------------
    def crear_pantalla_pos(self):
        frame = tk.Frame(self.contenedor, bg="#f4f6f7")
        self.frames["pos"] = frame
        self.carrito = [] # Lista temporal de productos en la venta actual

        # --- BARRA SUPERIOR (Búsqueda y Volver) ---
        top_bar = tk.Frame(frame, bg="#2c3e50", height=60)
        top_bar.pack(fill="x")
        cobro_frame = tk.Frame(top_bar, bg="#2c3e50")
        cobro_frame.pack(side="left", padx=20)

                # Contenedor de Identificación de Clientes
        client_pos_frame = tk.Frame(top_bar, bg="#34495e", padx=10, pady=5, highlightthickness=1, highlightbackground="#455a64")
        client_pos_frame.pack(side="left", padx=15, pady=5)

        tk.Label(client_pos_frame, text="CEDULA:", font=("Segoe UI", 9, "bold"), fg="#f1c40f", bg="#34495e").pack(side="left")
        self.ent_cedula_pos = tk.Entry(client_pos_frame, font=("Segoe UI", 10, "bold"), width=12, justify="center")
        self.ent_cedula_pos.pack(side="left", padx=5)
        
        self.lbl_info_cliente_pos = tk.Label(client_pos_frame, text="Cliente", font=("Segoe UI", 9, "italic"), fg="#bdc3c7", bg="#34495e")
        self.lbl_info_cliente_pos.pack(side="left", padx=5)

        self.ent_cedula_pos.bind("<Return>", lambda e: self.verificar_o_registrar_cliente_pos())


        tk.Label(cobro_frame, text="Tasa Cobro:", font=("Segoe UI", 9, "bold"), fg="#f39c12", bg="#2c3e50").pack(side="left")
        self.ent_tasa_cobro = tk.Entry(cobro_frame, width=8, font=("Segoe UI", 10, "bold"), justify="center")
        self.ent_tasa_cobro.pack(side="left", padx=5)
        
        self.ent_tasa_cobro.insert(0, str(self.tasa_actual)) 

        self.ent_tasa_cobro.bind("<KeyRelease>", lambda e: self.actualizar_tabla_pos())


        tk.Label(top_bar, text="🛒 PUNTO DE VENTA", font=("Segoe UI", 14, "bold"), fg="white", bg="#2c3e50").pack(side="left", padx=20)
        
        self.ent_buscar_pos = tk.Entry(top_bar, font=("Segoe UI", 12), width=30)
        self.ent_buscar_pos.pack(side="left", padx=10, pady=15)
        self.ent_buscar_pos.bind("<KeyRelease>", lambda e: self.filtrar_productos_pos())
        tk.Label(top_bar, text="🔍 Buscar pieza...", fg="white", bg="#2c3e50").pack(side="left")

        tk.Button(
            top_bar, 
            text=" 🚪  CERRAR CAJA Y SALIR ", 
            font=("Segoe UI", 10, "bold"), 
            bg="#c0392b",           # Rojo granate más elegante
            fg="white", 
            activebackground="#e74c3c", 
            activeforeground="white",
            relief="flat",          # Estilo moderno sin bordes 3D
            padx=15, 
            pady=7,                 # Más espaciado interno
            cursor="hand2",         # Cambia el cursor al pasar
            command=lambda: self.mostrar_pantalla("menu", direccion="left")
        ).pack(side="right", padx=20, pady=10)

        # --- CUERPO PRINCIPAL ---
        style = ttk.Style()
        style.configure("Treeview", rowheight=25) # Hace las filas más espaciosas y legibles
        cuerpo = tk.Frame(frame, bg="#f4f6f7")
        cuerpo.pack(fill="both", expand=True, padx=10, pady=10)

        # Panel Izquierdo: Categorías y Productos
        panel_izq = tk.Frame(cuerpo, bg="white", relief="flat")
        panel_izq.pack(side="left", fill="both", expand=True)
        

        canvas_pos = tk.Canvas(panel_izq, bg="white", highlightthickness=0)
        scroll_pos = tk.Scrollbar(panel_izq, orient="vertical", command=canvas_pos.yview)

        self.grid_productos = tk.Frame(canvas_pos, bg="#f5f7f8")

        self.canvas_window_id = canvas_pos.create_window((0, 0), window=self.grid_productos, anchor="nw")
        
        # Configuración de scroll
        canvas_pos.configure(yscrollcommand=scroll_pos.set)
        self._redimensionando = False

        def actualizar_configuracion_canvas(event):
            if self._redimensionando:
                return
            self._redimensionando = True

            def diferir_scrollregion():
                canvas_pos.configure(scrollregion=canvas_pos.bbox("all"))
                self._redimensionando = False
                
            canvas_pos.after(5, diferir_scrollregion)

        # Vinculamos el configure al canvas de forma optimizada
        canvas_pos.bind("<Configure>", actualizar_configuracion_canvas)
        
        # Actualización de región de scroll cuando el contenido interno mute de tamaño
        self.grid_productos.bind(
            "<Configure>", 
            lambda e: canvas_pos.after(5, lambda: canvas_pos.configure(scrollregion=canvas_pos.bbox("all")))
        )
           
        # Empaquetado del sistema de scroll izquierdo
        canvas_pos.pack(side="left", fill="both", expand=True)
        scroll_pos.pack(side="right", fill="y")

        self.canvas_window = canvas_pos.create_window((0, 0), window=self.grid_productos, anchor="nw")
        
        # Panel Derecho: Carrito y Total
        panel_der = tk.Frame(cuerpo, bg="#2c3e50", width=350)
        panel_der.pack(side="right", fill="both")

        tk.Label(panel_der, text="DETALLE DE VENTA", font=("Segoe UI", 12, "bold"), bg="#ecf0f1").pack(pady=10)


        tabla_pos_frame = tk.Frame(panel_der, bg="#2c3e50")
        tabla_pos_frame.pack(fill="both", expand=True, padx=10, pady=5)


    

        # Definición de columnas (Asegúrate de que coincidan con tu imagen)
        cols_pos = ("cant", "desc", "mar", "mod", "subt")
        self.tree_pos = ttk.Treeview(tabla_pos_frame, columns=cols_pos, show="headings", height=12)
        self.tree_pos.bind("<Double-1>", self.modificar_cantidad_carrito)
        # Scrollbar Quirúrgico
        scroll_y_pos = ttk.Scrollbar(tabla_pos_frame, orient="vertical", command=self.tree_pos.yview)
        self.tree_pos.configure(yscrollcommand=scroll_y_pos.set)

        # Encabezados
        headers_pos = {"cant": "Cant", "desc": "Descripción", "mar": "Marca", "mod": "Modelo", "subt": "Subtotal"}
        for c, h in headers_pos.items():
            self.tree_pos.heading(c, text=h)
            # Ajuste de anchos proporcional a tu imagen
            ancho = 50 if c == "desc" else 60
            if c == "cant": ancho = 40
            self.tree_pos.column(c, width=10, anchor="center")

        # Empaquetado con scroll a la derecha
        scroll_y_pos.pack(side="right", fill="y")
        self.tree_pos.pack(side="left", fill="both", expand=True)


        self.tree_pos.heading("cant", text="Cant")
        self.tree_pos.heading("desc", text="Descripción")
        self.tree_pos.heading("mar", text="Marca")
        self.tree_pos.heading("mod", text="Modelo")
        self.tree_pos.heading("subt", text="Subtotal")
        self.tree_pos.column("cant", width=40, anchor="center")
        self.tree_pos.column("desc", width=120, anchor="center")
        self.tree_pos.column("mar", width=80, anchor="center")
        self.tree_pos.column("mod", width=80, anchor="center")
        self.tree_pos.column("subt", width=70, anchor="center")
        
        self.tree_pos.pack(fill="x", padx=10)
        
        def ajustar_ancho_frame(event):
            # Forzamos a que el frame interno tenga el mismo ancho que el canvas
            canvas_pos.itemconfig(self.canvas_window, width=event.width)

        canvas_pos.bind("<Configure>", ajustar_ancho_frame)
        self.grid_productos.bind("<Configure>", lambda e: canvas_pos.configure(scrollregion=canvas_pos.bbox("all")))
        
        canvas_pos.configure(yscrollcommand=scroll_pos.set)
        canvas_pos.pack(side="left", fill="both", expand=True)
        scroll_pos.pack(side="right", fill="y")
        
        def actualizar_scroll(event):
            canvas_pos.configure(scrollregion=canvas_pos.bbox("all"))
        
        self.grid_productos.bind("<Configure>", actualizar_scroll)
        canvas_pos.configure(yscrollcommand=scroll_pos.set)

        # Empaquetado final del sistema de scroll
        canvas_pos.pack(side="left", fill="both", expand=True)
        scroll_pos.pack(side="right", fill="y")
        
        # SOPORTE PARA RUEDA DEL MOUSE (Toque de experto)
        def _on_mousewheel(event):

            win_low, win_high = canvas_pos.yview()
            if win_low == 0 and win_high == 1.0:
                return 
            canvas_pos.yview_scroll(int(-1*(event.delta/120)), "units")


        canvas_pos.bind_all("<MouseWheel>", _on_mousewheel)
        
    
        self.tree_pos.pack(fill="x", padx=10)

        self.frame_totales = tk.Frame(panel_der, bg="white", bd=0, highlightthickness=1, highlightbackground="#dcdde1")
        self.frame_totales.pack(pady=20, padx=20, fill="x")

        self.lbl_subtotal_pos = tk.Label(self.frame_totales, text="Subtotal: $0.00", font=("Segoe UI", 10), fg="#7f8c8d", bg="white")
        self.lbl_subtotal_pos.pack(pady=(10, 0))

        self.lbl_desc_pos = tk.Label(self.frame_totales, text="Descuento: $0.00 (0%)", font=("Segoe UI", 10), fg="#e67e22", bg="white")
        self.lbl_desc_pos.pack()

        self.lbl_total_pos = tk.Label(self.frame_totales, text="TOTAL: $0.00", font=("Segoe UI", 22, "bold"), fg="#27ae60", bg="white")
        self.lbl_total_pos.pack(pady=5)

        self.lbl_bs_pos = tk.Label(self.frame_totales, text="(Bs. 0.00)", font=("Segoe UI", 11), fg="#95a5a6", bg="white")
        self.lbl_bs_pos.pack(pady=(0, 10))


        acciones_frame = tk.Frame(panel_der, bg="#2c3e50")
        acciones_frame.pack(fill="x", padx=20, pady=10)

        acciones_frame.columnconfigure(0, weight=1)
        acciones_frame.columnconfigure(1, weight=1)
        

        tk.Button(
            acciones_frame, 
        text="VACIAR", 
        font=("Segoe UI", 9, "bold"),
        bg="#f39c12", 
        fg="white", 
        relief="flat", 
        cursor="hand2",
        height=3, 
        command=self.vaciar_carrito_completo
        ).grid(row=0, column=0, padx=5, pady=5, sticky="nsew")

        tk.Button(
            acciones_frame, 
            text="BORRAR", 
            font=("Segoe UI", 9, "bold"),
            bg="#e74c3c", 
            fg="white",
            relief="flat",
            cursor="hand2",
            height=3, 
            command=self.eliminar_del_carrito
        ).grid(row=0, column=1, padx=5, pady=5, sticky="nsew")
        
        tk.Button(
            acciones_frame, 
            text="PRECIO", 
            font=("Segoe UI", 9, "bold"),
            bg="#3498db", 
            fg="white", 
            relief="flat", 
            cursor="hand2",
            height=3, 
            command=self.modificar_precio_manual
        ).grid(row=1, column=0, padx=5, pady=5, sticky="nsew")

        
        tk.Button(
            acciones_frame, 
            text="DESCUENTO", 
            font=("Segoe UI", 9, "bold"),
            bg="#d86518", 
            fg="white", 
            relief="flat", 
            cursor="hand2",
            height=3, 
            command=self.abrir_selector_tipo_descuento # Nueva pasarela
        ).grid(row=1, column=1, padx=5, pady=5, sticky="nsew")


        tk.Button(acciones_frame, 
                  text="FINALIZAR Y COBRAR", 
                  font=("Segoe UI", 14, "bold"), 
                  bg="#27ae60",
                  fg="white", 
                  relief="flat",
                  cursor="hand2",
                  command=self.finalizar_venta, 
                  pady=10).grid(row=2, 
                                column=0, 
                                columnspan=2, 
                                padx=5, 
                                pady=5, 
                                sticky="nsew")

        self.filtrar_productos_pos() # Carga inicial

    def abrir_selector_tipo_descuento(self):
        """Despliega una pasarela simétrica para elegir el alcance del descuento."""
        if not self.carrito:
            messagebox.showwarning("Atención", "El carrito está vacío.")
            return

        win = tk.Toplevel(self.root)
        win.title("Alcance del Descuento")
        win.geometry("380x220")
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()

        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (380 // 2)
        y = (win.winfo_screenheight() // 2) - (220 // 2)
        win.geometry(f"+{x}+{y}")

        tk.Label(win, text="🏷️ SELECCIONE TIPO DE DESCUENTO", font=("Segoe UI", 11, "bold"), 
                 fg="#f39c12", bg="#2c3e50", pady=15).pack()

        btns_frame = tk.Frame(win, bg="#2c3e50")
        btns_frame.pack(fill="x", padx=30, pady=10)
        btns_frame.columnconfigure(0, weight=1)
        btns_frame.columnconfigure(1, weight=1)

        estilo_btn = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}

        # Opción A: Descuento por Ítem único (Tu función original)
        tk.Button(btns_frame, text="📦 POR PRODUCTO", bg="#34495e", fg="white",
                  activebackground="#455a64", activeforeground="white",
                  command=lambda: [win.destroy(), self.aplicar_descuento_item()], **estilo_btn).grid(row=0, column=0, padx=4, sticky="nsew")

        # Opción B: Descuento General de toda la factura (Nueva función)
        tk.Button(btns_frame, text="🛒 TOTAL FACTURA", bg="#27ae60", fg="white",
                  activebackground="#2ecc71", activeforeground="white",
                  command=lambda: [win.destroy(), self.aplicar_descuento_general_factura()], **estilo_btn).grid(row=0, column=1, padx=4, sticky="nsew")

        tk.Button(win, text="CANCELAR", bg="#c0392b", fg="white", font=("Segoe UI", 9), relief="flat",
                  command=win.destroy, cursor="hand2").pack(pady=15)

    def abrir_selector_tipo_descuento(self):
        """Despliega una pasarela simétrica para elegir el alcance del descuento."""
        if not self.carrito:
            messagebox.showwarning("Atención", "El carrito está vacío.")
            return

        win = tk.Toplevel(self.root)
        win.title("Alcance del Descuento")
        win.geometry("380x220")
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()

        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (380 // 2)
        y = (win.winfo_screenheight() // 2) - (220 // 2)
        win.geometry(f"+{x}+{y}")

        tk.Label(win, text="🏷️ SELECCIONE TIPO DE DESCUENTO", font=("Segoe UI", 11, "bold"), 
                 fg="#f39c12", bg="#2c3e50", pady=15).pack()

        btns_frame = tk.Frame(win, bg="#2c3e50")
        btns_frame.pack(fill="x", padx=30, pady=10)
        btns_frame.columnconfigure(0, weight=1)
        btns_frame.columnconfigure(1, weight=1)

        estilo_btn = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}

        # Opción A: Descuento por Ítem único (Tu función original)
        tk.Button(btns_frame, text="📦 POR PRODUCTO", bg="#34495e", fg="white",
                  activebackground="#455a64", activeforeground="white",
                  command=lambda: [win.destroy(), self.aplicar_descuento_item()], **estilo_btn).grid(row=0, column=0, padx=4, sticky="nsew")

        # Opción B: Descuento General de toda la factura (Nueva función)
        tk.Button(btns_frame, text="🛒 TOTAL FACTURA", bg="#27ae60", fg="white",
                  activebackground="#2ecc71", activeforeground="white",
                  command=lambda: [win.destroy(), self.aplicar_descuento_general_factura()], **estilo_btn).grid(row=0, column=1, padx=4, sticky="nsew")

        tk.Button(win, text="CANCELAR", bg="#c0392b", fg="white", font=("Segoe UI", 9), relief="flat",
                  command=win.destroy, cursor="hand2").pack(pady=15)

    def aplicar_descuento_general_factura(self):
        """Módulo analítico para prorratear un descuento en todo el carrito."""
        # 1. Auditoría contable del carrito actual en dólares base (Tasa 1.0)
        total_sin_desc = 0.0
        costo_total_carrito = 0.0
        
        for p in self.carrito:
            total_sin_desc += p['precio'] * p['cantidad']
            costo_total_carrito += p['costo'] * p['cantidad']

        # 2. Arquitectura de Ventana Modal Premium Bícroma
        win = tk.Toplevel(self.root)
        win.title("Descuento General de Factura")
        win.geometry("480x520")
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()
        win.resizable(False, False)

        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (480 // 2)
        y = (win.winfo_screenheight() // 2) - (520 // 2)
        win.geometry(f"+{x}+{y}")

        tk.Label(win, text="📊 DESCUENTO TOTAL GLOBAL", font=("Segoe UI", 13, "bold"), fg="#f39c12", bg="#2c3e50", pady=15).pack()

        # Ficha técnica de costos consolidados
        ficha = tk.Frame(win, bg="#34495e", padx=20, pady=15, highlightthickness=1, highlightbackground="#455a64")
        ficha.pack(fill="x", padx=30, pady=5)
        
        tk.Label(ficha, text=f"Subtotal de la Venta: ${total_sin_desc:.2f}", font=("Segoe UI", 10, "bold"), fg="white", bg="#34495e").pack(anchor="w")
        tk.Label(ficha, text=f"Costo Base de Inventario: ${costo_total_carrito:.2f}", font=("Segoe UI", 9), fg="#e74c3c", bg="#34495e").pack(anchor="w")
        
        # Selector de método de cálculo
        radio_frame = tk.Frame(win, bg="#2c3e50", pady=10)
        radio_frame.pack(fill="x", padx=40)
        var_tipo = tk.StringVar(value="porcentaje")
        
        tk.Radiobutton(radio_frame, text="Porcentaje (%)", variable=var_tipo, value="porcentaje", bg="#2c3e50", fg="white", font=("Segoe UI", 10, "bold"), selectcolor="#34495e").pack(side="left", padx=20)
        tk.Radiobutton(radio_frame, text="Monto Fijo ($)", variable=var_tipo, value="monto", bg="#2c3e50", fg="white", font=("Segoe UI", 10, "bold"), selectcolor="#34495e").pack(side="left", padx=20)

        ent_valor = tk.Entry(win, font=("Segoe UI", 18, "bold"), justify="center")
        ent_valor.pack(pady=15, padx=60, fill="x")
        ent_valor.focus()

        lbl_preview = tk.Label(win, text="Ingrese un valor para auditar margen...", font=("Segoe UI", 10, "bold"), fg="#3498db", bg="#2c3e50", wraplength=400)
        lbl_preview.pack(pady=10)

        def previsualizar_margen_global(*args):
            try:
                val_str = ent_valor.get().strip()
                if not val_str:
                    lbl_preview.config(text="Esperando entrada...", fg="#3498db")
                    return
                
                val = float(val_str)
                desc_usd = (val / 100) * total_sin_desc if var_tipo.get() == "porcentaje" else val
                
                if desc_usd < 0 or desc_usd > total_sin_desc:
                    lbl_preview.config(text="⚠️ Monto de descuento fuera de límites", fg="#e74c3c")
                    return

                precio_final_neto = total_sin_desc - desc_usd
                margen_seguridad = costo_total_carrito * 1.15
                
                if precio_final_neto < margen_seguridad:
                    lbl_preview.config(text=f"DENEGADO: Quiebra el 15% de utilidad.\nMínimo a cobrar: ${margen_seguridad:.2f}", fg="#e74c3c")
                else:
                    ganancia = precio_final_neto - costo_total_carrito
                    porc_utilidad = (ganancia / precio_final_neto * 100) if precio_final_neto > 0 else 0
                    lbl_preview.config(text=f"PERMITIDO\nPrecio Final Factura: ${precio_final_neto:.2f}\nGanancia Real: ${ganancia:.2f} ({porc_utilidad:.1f}%)", fg="#2ecc71")
            except ValueError: pass

        ent_valor.bind("<KeyRelease>", previsualizar_margen_global)
        var_tipo.trace_add("write", lambda *a: previsualizar_margen_global())

        def ejecutar_descuento_global():
            try:
                val_str = ent_valor.get().strip()
                if not val_str: return
                
                val = float(val_str)
                desc_total_usd = (val / 100) * total_sin_desc if var_tipo.get() == "porcentaje" else val

                # Validaciones Contables definitivas
                if desc_total_usd < 0 or desc_total_usd > total_sin_desc:
                    messagebox.showerror("Error", "Monto inválido.", parent=win)
                    return

                if (total_sin_desc - desc_total_usd) < (costo_total_carrito * 1.15):
                    messagebox.showerror("Protección de Utilidad", "¡OPERACIÓN ANULADA!\nEl descuento excede el límite del 15% de ganancia corporativa exigida.", parent=win)
                    return

                # --- ALGORITMO QUIRÚRGICO DE DISTRIBUCIÓN PROPORCIONAL ---
                # Repartimos el descuento de forma matemática exacta en cada ítem para que las 
                # estadísticas e impuestos individuales se mantengan consistentes por pieza
                for p in self.carrito:
                    proporcion_del_total = (p['precio'] * p['cantidad']) / total_sin_desc
                    descuento_atribuible_a_este_grupo = desc_total_usd * proporcion_del_total
                    # Guardamos el descuento unitario ajustado
                    p['descuento'] = round(descuento_atribuible_a_este_grupo / p['cantidad'], 4)

                self.actualizar_tabla_pos()
                win.destroy()
                messagebox.showinfo("Éxito", f"Se aplicó un descuento global de ${desc_total_usd:.2f} distribuido en el carrito.")
            except ValueError:
                messagebox.showerror("Error", "Ingrese solo números.", parent=win)

        # Botonera en rejilla simétrica unificada
        btns_frame = tk.Frame(win, bg="#2c3e50")
        btns_frame.pack(side="bottom", fill="x", padx=40, pady=25)
        btns_frame.columnconfigure(0, weight=1)
        btns_frame.columnconfigure(1, weight=1)

        btn_go = tk.Button(btns_frame, relief="flat", text="APLICAR GENERAL", bg="#27ae60", fg="white", 
                           activebackground="#2ecc71", activeforeground="white", command=ejecutar_descuento_global)
        btn_go.grid(row=0, column=0, padx=(0, 5), sticky="nsew")

        btn_cl = tk.Button(btns_frame, relief="flat", text="CANCELAR", bg="#c0392b", fg="white", 
                           activebackground="#e74c3c", activeforeground="white", command=win.destroy)
        btn_cl.grid(row=0, column=1, padx=(5, 0), sticky="nsew")



    def filtrar_productos_pos(self, categoria=None):
        # 1. Limpiar grid visual y configurar columnas
        for widget in self.grid_productos.winfo_children():
            widget.destroy()

        for i in range(5):
            self.grid_productos.grid_columnconfigure(i, weight=1)
        
        self.categoria_actual = categoria
        busqueda = self.ent_buscar_pos.get().strip()

        # 2. MODO CATEGORÍAS (Si no hay búsqueda ni categoría seleccionada)
                # 2. MODO CATEGORÍAS (Si no hay búsqueda ni categoría seleccionada)
                # 2. MODO CATEGORÍAS (Rompiendo el exceso de blanco con contraste estructural)
        if not busqueda and not categoria:
            # CLAVE: Cambiamos el fondo del grid principal a un gris sutil para generar contraste
            self.grid_productos.config(bg="#f5f7f8")
            
            # --- TARJETA DE COMBOS Y PROMOCIONES (EDICIÓN PREMIUM) ---
            card_combo = tk.Frame(
                self.grid_productos, bg="white", highlightthickness=1, highlightbackground="#dcdde1",
                width=175, height=185
            )
            card_combo.grid(row=0, column=0, padx=8, pady=8, sticky="nsew")
            card_combo.pack_propagate(False)

            # Encabezado oscuro texturizado para el Icono (Rompe el blanco)
            head_combo = tk.Frame(card_combo, bg="#ffeaa7", height=85) # Tono dorado pastel de base
            head_combo.pack(fill="x", side="top")
            head_combo.pack_propagate(False)

            lbl_ico_combo = tk.Label(head_combo, text="🎁", font=("Segoe UI", 28), bg="#ffeaa7", fg="#d4ac0d")
            lbl_ico_combo.pack(expand=True)

            # Cuerpo inferior blanco para el texto limpio
            body_combo = tk.Frame(card_combo, bg="white")
            body_combo.pack(fill="both", expand=True)

            lbl_txt_combo = tk.Label(
                body_combo, text="PROMOCIONES\nY COMBOS", font=("Segoe UI", 9, "bold"), 
                fg="#2c3e50", bg="white", justify="center"
            )
            lbl_txt_combo.pack(expand=True)

            # Eventos Hover y Clic vinculados en cascada para Combos
            widgets_combo = [card_combo, head_combo, lbl_ico_combo, body_combo, lbl_txt_combo]
            for w in widgets_combo:
                w.config(cursor="hand2")
                w.bind("<Button-1>", lambda e: self.mostrar_combos_en_pos())
                w.bind("<Enter>", lambda e, cp=card_combo, hc=head_combo, li=lbl_ico_combo: [
                    cp.config(highlightbackground="#f1c40f", bg="#fdfaf2"),
                    hc.config(bg="#f1c40f"), li.config(bg="#f1c40f", fg="white")
                ])
                w.bind("<Leave>", lambda e, cp=card_combo, hc=head_combo, li=lbl_ico_combo: [
                    cp.config(highlightbackground="#dcdde1", bg="white"),
                    hc.config(bg="#ffeaa7"), li.config(bg="#ffeaa7", fg="#d4ac0d")
                ])


            # --- CARGA Y RENDERIZADO DE MARCAS (CATEGORÍAS DE ALTO CONTRASTE) ---
            query = "SELECT DISTINCT marca FROM repuestos ORDER BY marca"
            categorias = consulta(query)
            
            for i, (cat,) in enumerate(categorias, start=1):
                card_cat = tk.Frame(
                    self.grid_productos, bg="white", highlightthickness=1, highlightbackground="#dcdde1",
                    width=175, height=185
                )
                card_cat.grid(row=i//5, column=i%5, padx=8, pady=8, sticky="nsew")
                card_cat.pack_propagate(False)

                # Encabezado Azul Corporativo Oscuro (Corta la monotonía visual)
                head_cat = tk.Frame(card_cat, bg="#2c3e50", height=85)
                head_cat.pack(fill="x", side="top")
                head_cat.pack_propagate(False)

                lbl_ico_cat = tk.Label(head_cat, text="📂", font=("Segoe UI", 26), bg="#2c3e50", fg="#bdc3c7")
                lbl_ico_cat.pack(expand=True)

                # Cuerpo inferior blanco para legibilidad tipográfica pura
                body_cat = tk.Frame(card_cat, bg="white")
                body_cat.pack(fill="both", expand=True)

                lbl_txt_cat = tk.Label(
                    body_cat, text=cat.upper(), font=("Segoe UI", 9, "bold"), 
                    fg="#2c3e50", bg="white", justify="center"
                )
                lbl_txt_cat.pack(expand=True)

                # --- CONGELAMIENTO SEGURO DE VARIABLES Y EFECTOS EN BLOQUE ---
                widgets_cat = [card_cat, head_cat, lbl_ico_cat, body_cat, lbl_txt_cat]
                cmd_cat = lambda e, c_actual=cat: self.filtrar_productos_pos(categoria=c_actual)

                for w in widgets_cat:
                    w.config(cursor="hand2")
                    w.bind("<Button-1>", cmd_cat)
                    
                    # Al pasar el ratón, la cabecera se ilumina con el azul brillante moderno
                    w.bind("<Enter>", lambda e, cp=card_cat, hc=head_cat, li=lbl_ico_cat: [
                        cp.config(highlightbackground="#3498db", bg="#f4f9fc"),
                        hc.config(bg="#3498db"), li.config(bg="#3498db", fg="white")
                    ])
                    w.bind("<Leave>", lambda e, cp=card_cat, hc=head_cat, li=lbl_ico_cat: [
                        cp.config(highlightbackground="#dcdde1", bg="white"),
                        hc.config(bg="#2c3e50"), li.config(bg="#2c3e50", fg="#bdc3c7")
                    ])
            
            # Sincronización del área deslizable
            self.grid_productos.update_idletasks()
            try:
                canvas_p = self.grid_productos.master
                canvas_p.configure(scrollregion=canvas_p.bbox("all"), bg="#f5f7f8")
            except: pass
            return

  

        # 3. MODO PRODUCTOS (Búsqueda Multicriterio)
        if busqueda:
            query = """SELECT codigo, nombre, modelo, precio_venta, cantidad, marca 
                       FROM repuestos 
                       WHERE nombre LIKE ? OR codigo LIKE ? OR modelo LIKE ? 
                       ORDER BY nombre LIMIT 50"""
            params = (f'%{busqueda}%', f'%{busqueda}%', f'%{busqueda}%')
        else:
            query = """SELECT codigo, nombre, modelo, precio_venta, cantidad, marca 
                       FROM repuestos 
                       WHERE marca = ? 
                       ORDER BY nombre"""
            params = (categoria,)

        productos = consulta(query, params)

        # 4. BOTÓN VOLVER
        if not busqueda:
            # Contenedor Base con las dimensiones simétricas del mosaico corporativo
            btn_volver_frame = tk.Frame(
                self.grid_productos, bg="white", highlightthickness=1, highlightbackground="#dcdde1",
                width=175, height=185
            )
            btn_volver_frame.grid(row=0, column=0, padx=8, pady=8, sticky="nsew")
            btn_volver_frame.pack_propagate(False)
            btn_volver_frame.grid_propagate(False)

            # Encabezado Gris Sólido para encapsular la flecha (Corta el exceso de blanco)
            head_volver = tk.Frame(btn_volver_frame, bg="#8a99a6", height=85) # Color gris balanceado de tu imagen
            head_volver.pack(fill="x", side="top")
            head_volver.pack_propagate(False)

            # Icono encapsulado con recuadro sutil integrado
            lbl_ico_back = tk.Label(
                head_volver, text=" 💡 ", font=("Segoe UI", 12), bg="#8a99a6", fg="white",
                highlightthickness=1, highlightbackground="#abb7b7", padx=8, pady=4
            )
            # Nota: Si prefieres usar caracteres de flecha pura por compatibilidad de fuentes:
            lbl_ico_back.config(text=" ← ", font=("Segoe UI", 14, "bold"))
            lbl_ico_back.pack(expand=True)

            # Cuerpo inferior blanco para el texto jerárquico
            body_volver = tk.Frame(btn_volver_frame, bg="white")
            body_volver.pack(fill="both", expand=True)

            lbl_txt_back = tk.Label(
                body_volver, text="VOLVER A\nREPUESTOS", font=("Segoe UI", 9, "bold"), 
                fg="#2c3e50", bg="white", justify="center"
            )
            lbl_txt_back.pack(expand=True)

            # --- VINCULACIÓN DE EVENTOS EN CASCADA CON EFECTO HOVER ---
            widgets_volver = [btn_volver_frame, head_volver, lbl_ico_back, body_volver, lbl_txt_back]
            for w in widgets_volver:
                w.config(cursor="hand2")
                w.bind("<Button-1>", lambda e: self.filtrar_productos_pos()) # Clic regresa al menú de marcas
                
                # Efecto interactivo: se ilumina con el azul del sistema al pasar el mouse
                w.bind("<Enter>", lambda e, cp=btn_volver_frame, hv=head_volver: [
                    cp.config(highlightbackground="#3498db", bg="#f4f9fc"),
                    hv.config(bg="#3498db")
                ])
                w.bind("<Leave>", lambda e, cp=btn_volver_frame, hv=head_volver: [
                    cp.config(highlightbackground="#dcdde1", bg="white"),
                    hv.config(bg="#8a99a6")
                ])
                
            inicio_index = 1
        else:
            inicio_index = 0

        # 5. RENDERIZADO DE PRODUCTOS (Respetando el inicio_index)
                # 5. RENDERIZADO DE PRODUCTOS PROFESIONAL (Tarjetas Estilizadas)
        for i, (cod, nom, mod, pre, cant, mar) in enumerate(productos, start=inicio_index):
            # --- PALETA DE COLORES CORPORATIVA PARA BADGES DE STOCK ---
            if int(cant) <= 0: 
                bg_badge, fg_badge = "#f8d7da", "#721c24"  # Rojo sutil (Agotado)
                estado_txt = "AGOTADO"
            elif int(cant) <= 5: 
                bg_badge, fg_badge = "#fff3cd", "#856404"  # Naranja sutil (Crítico)
                estado_txt = f"STOCK CRÍTICO: {cant}"
            else: 
                bg_badge, fg_badge = "#d4edda", "#155724"  # Verde sutil (Disponible)
                estado_txt = f"DISPONIBLE: {cant}"

            # Contenedor Base de la Tarjeta con Dimensiones Corporativas Fijas
            card_prod = tk.Frame(
                self.grid_productos, 
                bg="white", 
                highlightthickness=1, 
                highlightbackground="#dcdde1",
                padx=12, 
                pady=12,
                width=175,   # Fija el ancho exacto para simetría de 5 columnas
                height=185   # Altura perfecta para evitar saltos y superposiciones
            )
            card_prod.grid(row=i//5, column=i%5, padx=8, pady=8, sticky="nsew")
            
            # Forzado estricto de geometría (Evita que el texto decolore o estire el mosaico)
            card_prod.pack_propagate(False)
            card_prod.grid_propagate(False)

            # Capa 1: Nombre de la Pieza (Negrita corporativa)
            lbl_nom = tk.Label(
                card_prod, text=nom.upper(), font=("Segoe UI", 9, "bold"), 
                fg="#2c3e50", bg="white", wraplength=145, justify="center"
            )
            lbl_nom.pack(fill="x", pady=(2, 2))

            # Capa 2: Compatibilidad (Texto secundario minimalista)
            lbl_comp = tk.Label(
                card_prod, text=f"{mar}  •  {mod}", font=("Segoe UI", 8), 
                fg="#7f8c8d", bg="white", wraplength=145, justify="center"
            )
            lbl_comp.pack(fill="x")

            # Capa 3: Indicador Cambiario / Precio
            lbl_pre = tk.Label(
                card_prod, text=f"${pre:.2f}", font=("Segoe UI", 15, "bold"), 
                fg="#27ae60", bg="white"
            )
            lbl_pre.pack(expand=True, pady=5)

            # Capa 4: Píldora de Inventario inferior
            lbl_stock = tk.Label(
                card_prod, text=estado_txt, font=("Segoe UI", 8, "bold"), 
                bg=bg_badge, fg=fg_badge, padx=8, pady=4
            )
            lbl_stock.pack(fill="x", side="bottom")

            # --- ARQUITECTURA DE EVENTOS INTEGRAL (SOLUCIÓN AL CLOSURE BUG) ---
            widgets_tarjeta = [card_prod, lbl_nom, lbl_comp, lbl_pre, lbl_stock]
            
            # Congelamos el contexto de variables pasando copias locales por defecto en los lambdas
            cmd_venta_directa = lambda e, c=cod, n=nom, m=mod, ma=mar, p=pre, s=cant: \
                                self.agregar_al_carrito(c, n, m, ma, p, s)

            # DETONADOR 2: Clic Derecho (Auditoría e Inspección de Costos)
            cmd_inspeccion = lambda e, c=cod, n=nom, m=mod, ma=mar, p=pre, s=cant: \
                             self.mostrar_inspeccion_producto(c, n, m, ma, p, s)

            for w in widgets_tarjeta:
                w.config(cursor="hand2")
                
                # Asignación quirúrgica de los dos canales de ratón
                w.bind("<Button-1>", cmd_venta_directa) # Clic Izquierdo = Carrito automático
                w.bind("<Button-3>", cmd_inspeccion)    # Clic Derecho = Ficha de costo oculta
                
                # Efecto Hover por Tarjeta Independiente
                w.bind("<Enter>", lambda e, cp=card_prod, ln=lbl_nom, lc=lbl_comp, lp=lbl_pre: [
                    cp.config(bg="#f1f2f6", highlightbackground="#3498db"),
                    ln.config(bg="#f1f2f6"), lc.config(bg="#f1f2f6"), lp.config(bg="#f1f2f6")
                ])
                w.bind("<Leave>", lambda e, cp=card_prod, ln=lbl_nom, lc=lbl_comp, lp=lbl_pre: [
                    cp.config(bg="white", highlightbackground="#dcdde1"),
                    ln.config(bg="white"), lc.config(bg="white"), lp.config(bg="white")
                ])




        
       

            # --- NUEVA FUNCIÓN PARA ELIMINAR DEL CARRITO --- 
    def eliminar_del_carrito(self):
        sel = self.tree_pos.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione un producto para eliminar.")
            return
        

        valores = self.tree_pos.item(sel[0], "values")
        nombre_prod = valores[1]

    
        confirmar = messagebox.askyesno("Confirmar eliminación", 
                                    f"¿Estás seguro de quitar '{nombre_prod}' del carrito?")
        if not confirmar:
                return
        if confirmar:
            indice = self.tree_pos.index(sel)
            del self.carrito[indice]
            self.actualizar_tabla_pos()


        
        
        indice = self.tree_pos.index(sel[0])
        del self.carrito[indice]
        self.actualizar_tabla_pos()




      #ACTUALIZACIÓN DE AGREGAR (CON POPUP DE CANTIDAD)
    def agregar_al_carrito(self, codigo, nombre, modelo, marca, precio, stock_actual):

        

        for item in self.carrito:
            if item['codigo'] == codigo:
                messagebox.showinfo("Aviso", "El producto ya está en el carrito. Haz doble clic en la lista para cambiar la cantidad.")
                return


        if stock_actual < 1:
                    messagebox.showerror("Stock Insuficiente", "No puedes vender más de lo que hay en inventario.")
                    return
        

        res = consulta("SELECT precio_costo FROM repuestos WHERE codigo = ?", (codigo,))
        
        if res and len(res) > 0 and len(res[0]) > 0:
            p_costo = float(res[0][0])
        else:
            p_costo = float(precio) * 0.70 # Fallback de seguridad
        
        


        costo = consulta("SELECT precio_costo FROM repuestos WHERE codigo = ?", (codigo,))
        # Si por alguna razón no hay costo, usamos un estimado (70%) para que no de error
        p_costo = costo[0][0] if costo else (float(precio) * 0.70)

        self.carrito.append({
            'codigo': codigo,
            'nombre': nombre,
            'marca': marca,   
            'modelo': modelo, 
            'precio': float(precio),
            'costo': float(p_costo),
            'cantidad': 1,
            'stock_max': stock_actual
        })
        self.actualizar_tabla_pos()

    

        self.tree_pos.bind("<Double-1>", self.modificar_cantidad_carrito)

    def modificar_cantidad_carrito(self, event=None):
        # 1. Identificar selección y obtener índice real
        sel = self.tree_pos.selection()
        if not sel: return
        
        # Obtenemos el índice numérico de la fila en el Treeview
        idx = self.tree_pos.index(sel[0])
        
        # Seguridad: Verificamos que el índice exista en el carrito
        if idx >= len(self.carrito): return
        item = self.carrito[idx]

        # 2. Determinar límite de stock (Si es combo usa stock_max, si es pieza usa stock_max)
        limite_stock = item.get('stock_max', 1)

        # 3. Ventana Emergente Profesional
        win = tk.Toplevel(self.root)
        win.title("Editar Cantidad")
        win.geometry("400x380") # Un poco más de aire vertical
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()

        # Centrado dinámico
        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (400 // 2)
        y = (win.winfo_screenheight() // 2) - (380 // 2)
        win.geometry(f"400x380+{x}+{y}")

        tk.Label(win, text="📦 AJUSTAR CANTIDAD", font=("Segoe UI", 14, "bold"), 
                 fg="#f39c12", bg="#2c3e50", pady=20).pack()

        # Ficha del producto
        card = tk.Frame(win, bg="#34495e", padx=15, pady=15, highlightthickness=1, highlightbackground="#455a64")
        card.pack(fill="x", padx=30)
        
        tipo_item = "COMBO" if item.get('es_combo') else "PRODUCTO"
        tk.Label(card, text=f"TIPO: {tipo_item}", font=("Segoe UI", 8, "bold"), fg="#bdc3c7", bg="#34495e").pack(anchor="w")
        tk.Label(card, text=item['nombre'], font=("Segoe UI", 11, "bold"), fg="white", bg="#34495e", wraplength=300, justify="left").pack(anchor="w", pady=2)
        tk.Label(card, text=f"Máximo facturable: {limite_stock} unidades", font=("Segoe UI", 9), fg="#2ecc71", bg="#34495e").pack(anchor="w")

        # Entrada de datos
        ent_cant = tk.Entry(win, font=("Segoe UI", 22, "bold"), justify="center", 
                            bg="white", fg="#2c3e50", relief="flat")
        ent_cant.pack(pady=25, padx=50, fill="x")
        ent_cant.insert(0, str(item['cantidad']))
        ent_cant.focus_set()
        ent_cant.selection_range(0, tk.END) # Selecciona el texto para sobreescribir rápido

        def validar_y_guardar(event=None):
            try:
                entrada = ent_cant.get().strip()
                if not entrada: return
                
                nueva_cant = int(entrada)
                
                if 1 <= nueva_cant <= limite_stock:
                    # ACTUALIZACIÓN QUIRÚRGICA: Modificamos el objeto en RAM
                    self.carrito[idx]['cantidad'] = nueva_cant
                    
                    # Refrescamos la tabla visual y recalculamos totales
                    self.actualizar_tabla_pos()
                    win.destroy()
                else:
                    messagebox.showerror("Stock Insuficiente", 
                                       f"La cantidad debe estar entre 1 y {limite_stock}.\n\n"
                                       f"Verifique las existencias de los componentes.", parent=win)
            except ValueError:
                messagebox.showerror("Error de Formato", "Por favor, ingrese un número entero válido.", parent=win)

        # Botón de acción
        tk.Button(win, text="GUARDAR", font=("Segoe UI", 11, "bold"), 
                  bg="#27ae60", fg="white", activebackground="#2ecc71", activeforeground="white",
                  relief="flat", command=validar_y_guardar, cursor="hand2", pady=12).pack(fill="x", padx=50)

        # Bindeo de tecla Enter
        ent_cant.bind("<Return>", validar_y_guardar)




    def modificar_precio_manual(self):
        sel = self.tree_pos.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione un producto.")
            return
        idx = self.tree_pos.index(sel[0])
        item = self.carrito[idx]

        win = tk.Toplevel(self.root)
        win.title("Modificar Precio")
        win.geometry("400x400")
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()

        #centrado
        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (400 // 2)
        y = (win.winfo_screenheight() // 2) - (400 // 2)
        win.geometry(f"400x400+{x}+{y}")
        

        tk.Label(win, text="💲 AJUSTAR PRECIO", font=("Segoe UI", 14, "bold"), fg="#3498db", bg="#2c3e50", pady=20).pack()

        card = tk.Frame(win, bg="#34495e", padx=15, pady=10)
        card.pack(fill="x", padx=30)
        tk.Label(card, text=item['nombre'], font=("Segoe UI", 11, "bold"), fg="white", bg="#34495e").pack(anchor="w")
        tk.Label(card, text=f"Precio Costo: ${item['costo']:.2f}", font=("Segoe UI", 9), fg="#e74c3c", bg="#34495e").pack(anchor="w")

        ent_precio = tk.Entry(win, font=("Segoe UI", 18, "bold"), justify="center", bg="white")
        ent_precio.pack(pady=30, padx=50, fill="x")
        ent_precio.insert(0, f"{item['precio']:.2f}")

        def guardar_precio():
            try:
                nuevo_p = float(ent_precio.get())
                
                # --- BLOQUEO QUIRÚRGICO: MARGEN DE PROTECCIÓN 15% ---
                margen_minimo = item['costo'] * 1.15
                
                if nuevo_p < margen_minimo:
                    messagebox.showerror(
                        "Protección de Margen", 
                        f"¡OPERACIÓN DENEGADA!\n\n"
                        f"El precio mínimo permitido es: ${margen_minimo:.2f}\n"
                        f"(Costo base ${item['costo']:.2f} + 15% de utilidad).\n"
                        f"El monto ingresado (${nuevo_p:.2f}) es insuficiente.",
                        parent=win
                    )
                    return
                # ----------------------------------------------------
                
                self.carrito[idx]['precio'] = nuevo_p
                self.carrito[idx]['descuento'] = 0.0 # Reset descuento por seguridad
                self.actualizar_tabla_pos()
                win.destroy()
            except ValueError:
                messagebox.showerror("Error", "Monto inválido. Ingrese solo números.", parent=win)

        tk.Button(win, text="✅ CAMBIAR PRECIO", font=("Segoe UI", 11, "bold"), bg="#3498db", fg="white", 
                  command=guardar_precio, cursor="hand2", pady=10).pack(fill="x", padx=50)





        
    def actualizar_tabla_pos(self):
        # 1. Limpieza de tabla
        for item in self.tree_pos.get_children(): 
            self.tree_pos.delete(item)

        # 2. Obtención de Tasa de Cobro
        try:
            tasa_cobro = float(self.ent_tasa_cobro.get())
        except:
            tasa_cobro = self.tasa_actual

        # 3. Inicialización de acumuladores
        total_usd_final = 0.0
        total_sin_descuento = 0.0
        
        for p in self.carrito:
            
            # PRECIO AJUSTADO: El corazón de la sincronía
            precio_ajustado = (p['precio'] * tasa_cobro) / self.tasa_actual
            desc_u = p.get('descuento', 0.0)
            
            subt_sin_desc = precio_ajustado * p['cantidad']
            subt_con_desc = (precio_ajustado - desc_u) * p['cantidad']

            # CÁLCULO DE % DE DESCUENTO: Sincronizado con el precio ajustado
            if desc_u > 0:
                # Usamos precio_ajustado para que el % sea real sobre lo que se cobra
                porc_desc = (desc_u / precio_ajustado) * 100
                desc_texto = f"{p['nombre']} (-{porc_desc:.0f}%)"
            else:
                desc_texto = p['nombre']
            
            # Inserción en el Treeview
            self.tree_pos.insert("", "end", values=(
                p['cantidad'], 
                desc_texto,
                p['marca'], 
                p['modelo'], 
                f"${subt_con_desc:.2f}"
            ))
            
            total_usd_final += subt_con_desc
            total_sin_descuento += subt_sin_desc
        
        # 4. Cálculos finales para las etiquetas
        descuento_total = total_sin_descuento - total_usd_final
        total_bs = total_usd_final * self.tasa_actual
        porc_total = (descuento_total / total_sin_descuento * 100) if total_sin_descuento > 0 else 0

        # 5. ACTUALIZACIÓN QUIRÚRGICA DE LA TARJETA DE TOTALES
        self.lbl_subtotal_pos.config(text=f"Subtotal: ${total_sin_descuento:,.2f}")
        self.lbl_desc_pos.config(text=f"Descuento: ${descuento_total:,.2f} ({porc_total:.1f}%)")
        self.lbl_total_pos.config(text=f"TOTAL: ${total_usd_final:,.2f}")
        self.lbl_bs_pos.config(text=f"(Bs. {total_bs:,.2f})")


    def finalizar_venta(self):
        """Despliega la pasarela de confirmación final con corrección estricta de desempaquetado de base de datos."""
        if not self.carrito: 
            messagebox.showwarning("Carrito vacío", "No hay productos para procesar.")
            return
        
        # 1. Obtener la tasa de cobro actual de forma segura
        try:
            tasa_c = float(self.ent_tasa_cobro.get().strip() if hasattr(self, 'ent_tasa_cobro') else self.tasa_actual)
        except:
            tasa_c = self.tasa_actual

        # 2. Calcular el TOTAL REAL original de la orden en caliente
        total_usd = 0.0
        for p in self.carrito:
            precio_ajustado = (p['precio'] * tasa_c) / self.tasa_actual
            desc_u = p.get('descuento', 0.0)
            total_usd += (precio_ajustado - desc_u) * p['cantidad']

        total_bs = total_usd * tasa_c

        # 3. Crear Ventana Modal Transitoria con Geometría Ampliada
        win_modal = tk.Toplevel(self.root)
        win_modal.title("Confirmar Transacción")
        win_modal.geometry("450x640")  
        win_modal.configure(bg="#2c3e50")
        win_modal.transient(self.root)
        win_modal.grab_set()
        win_modal.resizable(False, False)

        win_modal.update_idletasks()
        x = (win_modal.winfo_screenwidth() // 2) - (450 // 2)
        y = (win_modal.winfo_screenheight() // 2) - (640 // 2)
        win_modal.geometry(f"+{x}+{y}")
        
        tk.Label(win_modal, text="🏁 FINALIZAR VENTA", font=("Segoe UI", 16, "bold"), 
                 fg="#27ae60", bg="#2c3e50", pady=15).pack()

        # --- CAPA A: PANEL DE RESUMEN FINANCIERO ---
        resumen = tk.Frame(win_modal, bg="#34495e", padx=20, pady=15)
        resumen.pack(fill="x", padx=40, pady=(5, 10))

        tk.Label(resumen, text="TOTAL A COBRAR:", font=("Segoe UI", 9, "bold"), fg="#bdc3c7", bg="#34495e").pack()
        
        self.lbl_final_usd = tk.Label(resumen, text=f"${total_usd:,.2f}", font=("Segoe UI", 28, "bold"), fg="#27ae60", bg="#34495e")
        self.lbl_final_usd.pack()
        
        self.lbl_final_bs = tk.Label(resumen, text=f"Bs. {total_bs:,.2f}", font=("Segoe UI", 13, "bold"), fg="white", bg="#34495e")
        self.lbl_final_bs.pack(pady=(2, 5))

        self.lbl_vale_descuento_txt = tk.Label(resumen, text="", font=("Segoe UI", 9, "bold"), fg="#f1c40f", bg="#34495e")
        self.lbl_vale_descuento_txt.pack()

        tk.Label(resumen, text="MÉTODO DE PAGO:", font=("Segoe UI", 8, "bold"), fg="#bdc3c7", bg="#34495e").pack(pady=(8, 2))
        self.var_metodo_pago = tk.StringVar(value="DÓLARES ($)")
        combo_metodo = ttk.Combobox(
            resumen, textvariable=self.var_metodo_pago, values=["DÓLARES ($)", "BOLÍVARES (Bs.)"], 
            state="readonly", justify="center", font=("Segoe UI", 9, "bold")
        )
        combo_metodo.pack(pady=2, fill="x", padx=20)

        # --- CAPA B: FORMULARIO DE VALES DE CANJE Y NOTAS DE CRÉDITO ---
        nc_frame = tk.Frame(win_modal, bg="#2c3e50", pady=5)
        nc_frame.pack(fill="x", padx=40)
        
        tk.Label(nc_frame, text="¿Aplica un Vale de Canje / Nota de Crédito?", font=("Segoe UI", 9, "bold"), fg="#bdc3c7", bg="#2c3e50").pack(anchor="w")
        
        ent_codigo_nc = tk.Entry(nc_frame, font=("Segoe UI", 12, "bold"), justify="center", fg="#d35400", width=15)
        nc_frame.pack_propagate(True) # Permitir empaquetado fluido
        ent_codigo_nc.pack(anchor="w", pady=5)
        ent_codigo_nc.insert(0, "NC-")

        lbl_info_nc = tk.Label(win_modal, text="Si aplica un cambio, ingrese el código arriba.", font=("Segoe UI", 8, "italic"), fg="#3498db", bg="#2c3e50")
        lbl_info_nc.pack(pady=2)

        def verificar_vale_canje(*args):
            codigo = ent_codigo_nc.get().strip().upper()
            if len(codigo) < 7: 
                self.lbl_final_usd.config(text=f"${total_usd:,.2f}", fg="#27ae60")
                self.lbl_final_bs.config(text=f"Bs. {total_bs:,.2f}")
                self.lbl_vale_descuento_txt.config(text="")
                lbl_info_nc.config(text="Si aplica un cambio, ingrese el código arriba.", fg="#3498db")
                return
            
            res = consulta("SELECT monto_a_favor, cliente FROM notas_credito WHERE codigo_nota=? AND estado='VIGENTE'", (codigo,), db_path=DB_VENTAS)
            if res and len(res) > 0:
                monto_nc, deudor = res[0] # Acceso seguro al primer registro
                diferencia_usd = max(0.0, total_usd - monto_nc)
                diferencia_bs = diferencia_usd * tasa_c
                
                self.lbl_vale_descuento_txt.config(text=f"🏷️ VALE DE DEVOLUCIÓN ({deudor}): -${monto_nc:.2f}")
                self.lbl_final_usd.config(text=f"${diferencia_usd:,.2f}", fg="#f1c40f") 
                self.lbl_final_bs.config(text=f"Bs. {diferencia_bs:,.2f}")
                lbl_info_nc.config(text=f"✅ Nota de Crédito vinculada con éxito.", fg="#2ecc71")
            else:
                self.lbl_final_usd.config(text=f"${total_usd:,.2f}", fg="#27ae60")
                self.lbl_final_bs.config(text=f"Bs. {total_bs:,.2f}")
                self.lbl_vale_descuento_txt.config(text="")
                lbl_info_nc.config(text="❌ Código inválido o ya quemado en sistema.", fg="#e74c3c")

        ent_codigo_nc.bind("<KeyRelease>", verificar_vale_canje)

        tk.Label(win_modal, text="¿Desea procesar el pago y descontar stock?", 
                 font=("Segoe UI", 9, "bold"), fg="#95a5a6", bg="#2c3e50", pady=10).pack()

        # --- CAPA C: MOTOR INTERNO DE PROCESAMIENTO CONTABLE NETO ---
                # --- CAPA C: MOTOR INTERNO DE PROCESAMIENTO CONTABLE NETO ---
        def ejecutar_procesamiento(es_credito=False):
            """Valida, quema la nota de crédito y despacha la orden regulando canjes menores y créditos automáticos."""
            codigo_nc_aplicar = ent_codigo_nc.get().strip().upper()
            monto_vale_descuento = 0.0
            costo_vale_reingreso = 0.0

            if len(codigo_nc_aplicar) >= 7 and codigo_nc_aplicar.startswith("NC-"):
                res_nc = consulta(
                    "SELECT monto_a_favor, costo_devuelto FROM notas_credito WHERE codigo_nota=? AND estado='VIGENTE'",
                    (codigo_nc_aplicar,), db_path=DB_VENTAS
                )
                if res_nc and len(res_nc) > 0:
                    monto_vale_descuento, costo_vale_reingreso = res_nc[0]
                    consulta("UPDATE notas_credito SET estado='APLICADA' WHERE codigo_nota=?", (codigo_nc_aplicar,), fetch=False, db_path=DB_VENTAS)

            # Logística de protección de caja por canje menor con devolución en efectivo
            if monto_vale_descuento > 0 and total_usd < monto_vale_descuento:
                vuelto_efectivo_entregar = round(monto_vale_descuento - total_usd, 2)
                descuento_aplicable_hoy = total_usd
                porcentaje_usado = total_usd / monto_vale_descuento
                costo_aplicable_hoy = round(costo_vale_reingreso * porcentaje_usado, 2)

                try:
                    cedula_actual = self.ent_cedula_pos.get().strip().replace(".", "").replace("-", "").upper() if hasattr(self, 'ent_cedula_pos') else "GENERICO"
                    if not cedula_actual: cedula_actual = "GENERICO"
                    
                    fecha_egreso = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                    glosa_egreso = f"[EGRESO DEVOLUCIÓN EFECTIVO] - Vuelto al Cliente ID: {cedula_actual} (Origen Vale: {codigo_nc_aplicar})"
                    
                    consulta("INSERT INTO ventas_log (fecha_hora, productos, total, costo_total) VALUES (?, ?, ?, 0.0)",
                             (fecha_egreso, glosa_egreso, -vuelto_efectivo_entregar), fetch=False, db_path=DB_VENTAS)
                    
                    messagebox.showinfo("Entregar Vuelto", 
                                        f"🔄 CAMBIO MENOR - DEVOLUCIÓN EN EFECTIVO\n\n💵 ENTREGUE AL CLIENTE: ${vuelto_efectivo_entregar:.2f} EN EFECTIVO", 
                                        parent=win_modal)
                except Exception as err_egreso:
                    print(f"DEBUG EGRESO: Error al registrar salida física de caja: {err_egreso}")
            else:
                descuento_aplicable_hoy = monto_vale_descuento
                costo_aplicable_hoy = costo_vale_reingreso

            if descuento_aplicable_hoy > 0:
                self.carrito.append({
                    'codigo': "INFO-NC", 'nombre': f"[CANJE APLICADO CON VALE {codigo_nc_aplicar}]",
                    'marca': "CANJE", 'modelo': "BALANCE", 'precio': -descuento_aplicable_hoy,
                    'costo': -costo_aplicable_hoy, 'cantidad': 1
                })

            # --- EXTRAER CÉDULA PARA ASIGNACIÓN DE CONTADO O CRÉDITO ---
            cedula_actual = self.ent_cedula_pos.get().strip().replace(".", "").replace("-", "").upper() if hasattr(self, 'ent_cedula_pos') else ""

            # 5. BIFURCACIÓN DE FLUJO: CRÉDITO AUTOMATIZADO POR CÉDULA
            if es_credito:
                # BLOQUEO DE SEGURIDAD: Impedir créditos a clientes anónimos
                if not cedula_actual or cedula_actual == "GENERICO":
                    messagebox.showerror(
                        "Crédito Denegado", 
                        "❌ ERROR OPERACIONAL\n\nNo se puede registrar una venta a crédito sin un deudor.\n"
                        "Por favor, introduzca primero la cedula del cliente en la barra superior del POS y presione Enter antes de cobrar.", 
                        parent=win_modal
                    )
                    return

                # Consultamos el nombre del cliente asociado a esa cédula en la DB de perfiles
                res_cliente = consulta("SELECT nombre FROM clientes WHERE cedula = ?", (cedula_actual,), db_path=DB_CLIENTES)
                
                if res_cliente and len(res_cliente) > 0:
                    nombre_deudor = res_cliente[0][0]
                    # Logística Profesional: Formateamos el registro combinando Nombre y Cédula
                    cliente_registro_credito = f"{nombre_deudor} (C.I: {cedula_actual})"
                    
                    # Ejecutamos el guardado de forma directa sin abrir subventanas manuales feas
                    self.guardar_credito_final(cliente_registro_credito, win_modal, modo="credito")
                else:
                    messagebox.showerror(
                        "Cliente No Registrado", 
                        f"❌ ERROR DE VALIDACIÓN\n\nLa cedula [{cedula_actual}] no está registrada en el sistema.\n"
                        "Por favor, presione Enter sobre el campo de cedula en el POS para darlo de alta antes de procesar el crédito.", 
                        parent=win_modal
                    )
                    return
            else:
                # Si es de contado, se liquida de forma directa y limpia asociando la cédula al log de ventas
                self.guardar_credito_final("CONTADO", win_modal, modo="contado")



        # --- CAPA D: CONTENEDOR DE ACCIONES EN MATRIZ CORPORATIVA (GRID) ---
        btns_final_container = tk.Frame(win_modal, bg="#2c3e50")
        btns_final_container.pack(side="bottom", fill="x", padx=40, pady=(5, 20))
        btns_final_container.columnconfigure(0, weight=1)

        estilo_final_btn = {"relief": "flat", "height": 2, "font": ("Segoe UI", 10, "bold"), "cursor": "hand2"}

        btn_contado = tk.Button(
            btns_final_container, text="FINALIZAR Y COBRAR (CONTADO)", bg="#27ae60", fg="white", 
            activebackground="#2ecc71", activeforeground="white", command=lambda: ejecutar_procesamiento(es_credito=False), **estilo_final_btn
        )
        btn_contado.grid(row=0, column=0, pady=4, sticky="nsew")

        btn_credito = tk.Button(
            btns_final_container, text="REGISTRAR VENTA A CRÉDITO", bg="#34495e", fg="white", 
            activebackground="#455a64", activeforeground="white", command=lambda: ejecutar_procesamiento(es_credito=True), **estilo_final_btn
        )
        btn_credito.grid(row=1, column=0, pady=4, sticky="nsew")

        btn_cancelar = tk.Button(
            btns_final_container, text="CANCELAR ACCIÓN", bg="#c0392b", fg="white", 
            activebackground="#e74c3c", activeforeground="white", command=win_modal.destroy, **estilo_final_btn
        )
        btn_cancelar.grid(row=2, column=0, pady=(10, 0), sticky="nsew")

        
        
        
    def guardar_credito_final(self, cliente, ventana_principal, modo="contado"):
        """Motor de guardado unificado con asignación jerárquica de tasa y trazabilidad por cedula."""
        if ventana_principal:
            ventana_principal.destroy() 
        
        try:
            # --- EVALUACIÓN QUIRÚRGICA DE PRIORIDAD DE TASA ---
            try:
                tasa_entrada = self.ent_tasa_cobro.get().strip() if hasattr(self, 'ent_tasa_cobro') else ""
                if tasa_entrada and float(tasa_entrada) > 0:
                    tasa_c = float(tasa_entrada) # Prioridad 1: Tasa de cobro específica de la venta
                else:
                    tasa_c = self.tasa_actual   # Prioridad 2: Tasa principal del inicio
            except (ValueError, TypeError):
                tasa_c = self.tasa_actual       # Fallback de seguridad si digitan letras

            total_venta = 0.0
            costo_total_venta = 0.0
            resumen_productos = []

            # Processing del carrito y descuento de inventario
            for p in self.carrito:
                # Sincronizamos los precios del carrito usando la tasa congelada para esta transacción
                p_ajustado = (p['precio'] * tasa_c) / self.tasa_actual
                desc_u = p.get('descuento', 0.0)
                subtotal_item = (p_ajustado - desc_u) * p['cantidad']
                
                total_venta += subtotal_item
                costo_total_venta += p['costo'] * p['cantidad']
                
                detalle = f"[{p['codigo']}] {p['nombre']} x{p['cantidad']} (Ref: ${p_ajustado:.2f})"
                resumen_productos.append(detalle)

                # Descuento físico de existencias (Combos vs Repuestos Sueltos)
                if p.get('es_combo'):
                    for cod_pieza, cant_en_combo in p['receta']:
                        total_unidades_descontar = p['cantidad'] * cant_en_combo
                        consulta("UPDATE repuestos SET cantidad = cantidad - ? WHERE codigo = ?", 
                                (total_unidades_descontar, cod_pieza), fetch=False)
                else:
                    consulta("UPDATE repuestos SET cantidad = cantidad - ? WHERE codigo = ?", 
                            (p['cantidad'], p['codigo']), fetch=False)

            # Cálculo exacto en Bolívares usando la tasa consolidada arriba
            total_bs = total_venta * tasa_c
            fecha_v = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            
            # --- NUEVA TRAZABILIDAD INTEGRAL POR CÉDULA DEL CLIENTE ---
            # Extraemos lo ingresado en la barra superior del POS limpiando caracteres especiales
            cedula_factura = self.ent_cedula_pos.get().strip().replace(".", "").replace("-", "").upper() if hasattr(self, 'ent_cedula_pos') else "GENERICO"
            if not cedula_factura: 
                cedula_factura = "GENERICO"

            # 2. INYECCIÓN DEL SELLO CON LA TASA Y LA CÉDULA DE IDENTIDAD
            if modo == "credito":
                metodo_auditoria = f"[MÉTODO: CRÉDITO PENDIENTE | CEDULA: {cedula_factura}]"
            else:
                metodo_seleccionado = self.var_metodo_pago.get() if hasattr(self, 'var_metodo_pago') else "DÓLARES ($)"
                metodo_auditoria = f"[MÉTODO: {metodo_seleccionado} | TASA REF: {tasa_c:.2f} | CEDULA: {cedula_factura}]"

            # Concatenación en la columna flexible TEXT para auditoría retrocompatible
            lista_productos_con_pago = resumen_productos + [metodo_auditoria]
            productos_str_final = ", ".join(lista_productos_con_pago)
            
            # 3. PERSISTENCIA EN DISCO DENTRO DE LAS DBs
            if modo == "credito":
                consulta("""INSERT INTO cuentas_por_cobrar 
                            (cliente, fecha_deuda, detalle_productos, monto_total, costo_total, costo_pendiente) 
                            VALUES (?, ?, ?, ?, ?, ?)""", 
                         (cliente.upper(), fecha_v, productos_str_final, total_venta, costo_total_venta, costo_total_venta), 
                         fetch=False, db_path=DB_CREDITOS)
                messagebox.showinfo("Éxito", f"Crédito registrado para: {cliente.upper()}")
            else:
                consulta("INSERT INTO ventas_log (fecha_hora, productos, total, costo_total) VALUES (?, ?, ?, ?)", 
                        (fecha_v, productos_str_final, total_venta, costo_total_venta), 
                        fetch=False, db_path=DB_VENTAS)
                
                if "BOLÍVARES" in metodo_auditoria:
                    messagebox.showinfo("Venta Procesada", f"Venta exitosa por Bs. {total_bs:,.2f}\n(A tasa de cobro: {tasa_c:.2f} Bs/USD | ID: {cedula_factura})")
                else:
                    messagebox.showinfo("Venta Procesada", f"Venta exitosa por ${total_venta:,.2f}\n(Asociada al ID: {cedula_factura})")

            # 4. REINICIO AUTOMÁTICO DEL MÓDULO DE CLIENTES DEL POS
            if hasattr(self, 'ent_cedula_pos'):
                self.ent_cedula_pos.delete(0, "end")
                self.lbl_info_cliente_pos.config(text="[Cliente Generico]", fg="#bdc3c7")

            

            # 5. LIMPIEZA Y ACTUALIZACIÓN EN CASCADA DE PANTALLAS
            self.carrito = []
            self.actualizar_tabla_pos()
            self.filtrar_productos_pos()
            self.actualizar_estadisticas() 
            self.cargar_tabla_gestion()
            if "finanzas" in self.frames: self.actualizar_tabla_finanzas()
            if "creditos" in self.frames: self.cargar_tabla_creditos()

        except Exception as e:
            messagebox.showerror("Error Crítico", f"No se pudo completar la operación: {e}")






    def __init__(self, root):
        self.root = root
        self.root.title("MOTO REPUESTOS KRAKEN")
        self.root.geometry("1280x800")
        self.root.minsize(1200, 900)
        #centrado
        self.root.update_idletasks()
        x = (self.root.winfo_screenwidth() // 2) - (1280 // 2)
        y = (self.root.winfo_screenheight() // 2) - (800 // 2)
        self.root.geometry(f"1280x800+{x}+{y}")
        self.categoria_actual = None

        # Dentro de __init__
        self.pagina_actual = 0
        self.items_por_pagina = 50


        inicializar_db()
        inicializar_db_ventas()
        inicializar_db_creditos()
        inicializar_db_combos()
        inicializar_db_clientes()

        try:
            self.root.iconphoto(False, tk.PhotoImage(file='logo.png'))
        except:
            pass

        
        try:
            tasa_db = consulta("SELECT valor FROM configuracion WHERE clave='tasa_dolar'")
            self.tasa_actual = float(tasa_db[0][0]) if tasa_db else 1.0
        except:
            self.tasa_actual = 1.0
        # Diccionario de pantallas
        self.frames = {}

        # Contenedor principal (para manejar pantallas FULL)
        self.contenedor = tk.Frame(self.root, bg="#f4f6f7")
        self.contenedor.pack(fill="both", expand=True)

        # Crear pantallas
        self.crear_pantalla_menu()
        self.crear_pantalla_gestion()
        self.crear_pantalla_registro()
        self.crear_pantalla_reporte()
        self.crear_pantalla_pos()
        self.crear_pantalla_finanzas()
        self.crear_pantalla_creditos()
        self.crear_pantalla_clientes()

        # Mostrar menú principal
        self.mostrar_pantalla("menu", anim=False)

        # Actualizar estadísticas iniciales
        self.actualizar_estadisticas()

    def exportar_a_pdf(self, titulo_reporte, datos, encabezados):

        titulo_up = titulo_reporte.upper()
        
        if "COBRANZA" in titulo_up or "CUENTAS" in titulo_up or "DEUDOR" in titulo_up:
            sub_carpeta = "COBRANZA"
            nombre_base = "REPORTE_COBRANZA"
        elif "FINANZAS" in titulo_up or "VENTAS" in titulo_up or "INGRESOS" in titulo_up:
            sub_carpeta = "FINANZAS"
            nombre_base = "REPORTE_FINANCIERO"
        elif "CRÍTICO" in titulo_up or "BAJO" in titulo_up:
            sub_carpeta = "STOCK_BAJO"
            nombre_base = "ALERTA_STOCK_BAJO"
        else:
            sub_carpeta = "INVENTARIO"
            nombre_base = "INVENTARIO_GENERAL"

        # 2. CREACIÓN DE DIRECTORIOS
        ruta_directorio = os.path.join(os.getcwd(), "REPORTES_KRAKEN", sub_carpeta)
        if not os.path.exists(ruta_directorio):
            os.makedirs(ruta_directorio)

        # 3. NOMBRE AUTOMÁTICO (Nombre + Fecha + Hora)
        fecha_hora = datetime.now().strftime("%Y-%m-%d_%H-%M")
        nombre_archivo = f"{nombre_base}_{fecha_hora}.pdf"

        # 4. LANZAR EL DIÁLOGO DE GUARDADO CON LA RUTA CORRECTA
        archivo = filedialog.asksaveasfilename(
            initialdir=ruta_directorio,
            initialfile=nombre_archivo,
            defaultextension=".pdf",
            filetypes=[("PDF", "*.pdf")],
            title=f"Guardar en: {sub_carpeta}"
        )
        if not archivo: return

        try:
           

            doc = SimpleDocTemplate(archivo, pagesize=letter)
            estilos = getSampleStyleSheet()
            elementos = []

            # 1. Definimos un estilo para las celdas con salto de línea
            estilo_celda = estilos["Normal"]
            estilo_celda.fontSize = 8
            estilo_celda.leading = 10  # Espaciado entre líneas
            estilo_celda.alignment = 1 # Centrado (0=Izq, 1=Cen, 2=Der)

            elementos.append(Paragraph(titulo_reporte, estilos['Title']))
            elementos.append(Paragraph(f"Generado el: {datetime.now().strftime('%Y-%m-%d %H:%M')}", estilos['Normal']))
            elementos.append(Spacer(1, 20))

            # Preparar datos de la tabla
            data_pdf = [encabezados]
            for fila in datos:
                fila_procesada = []
                for i, celda in enumerate(fila):
                    # Si es la columna de Nombre/Descripción (índice 1)
                    if i in [1, 2, 3]:  # Puedes ajustar esto según tus columnas
                        fila_procesada.append(Paragraph(str(celda), estilo_celda))
                    else:
                        fila_procesada.append(str(celda))
                data_pdf.append(fila_procesada)

            # 3. DEFINIR ANCHOS DE COLUMNA (Para que el texto sepa dónde bajar)
            # Ajusta estos valores según tu necesidad (Ancho total carta aprox 540)
            anchos = [70, 160, 80, 80, 40, 40, 50]  # Ejemplo: Código, Nombre, Marca, Precio, Stock, etc.

            # Crear Tabla
            tabla = Table(data_pdf, colWidths=anchos)
            estilo_tabla = TableStyle([
                ('BACKGROUND', (0, 0), (-1, 0), colors.HexColor("#2c3e50")),
                ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
                ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
                ('VALIGN', (0, 0), (-1, -1), 'MIDDLE'), # Centrado vertical importante
                ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
                ('FONTSIZE', (0, 0), (-1, -1), 8),
                ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
                ('BACKGROUND', (0, 1), (-1, -1), colors.whitesmoke),
                ('GRID', (0, 0), (-1, -1), 1, colors.grey)
            ])
            tabla.setStyle(estilo_tabla)
            elementos.append(tabla)

            doc.build(elementos)
            messagebox.showinfo("Éxito", f"Reporte PDF creado en:\n{archivo}")
        except Exception as e:
            messagebox.showerror("Error", f"No se pudo generar el PDF: {e}")


    # -------------------------------------------------
    # UTILIDAD: CAMBIO DE PANTALLAS CON ANIMACIÓN SLIDE
    # -------------------------------------------------
    def mostrar_pantalla(self, nombre, anim=True, direccion="right"):
        frame_destino = self.frames[nombre]
        
        # 1. Identificar pantalla actual
        frame_actual = None
        for f in self.frames.values():
            if f.winfo_ismapped():
                frame_actual = f
                break

        # 2. REFRESCO INTELIGENTE (Punto 2 de tu imagen)
        # Cargamos los datos JUSTO ANTES de mostrar, para que la transición sea fluida
        if nombre == "gestion":
            self.pagina_actual = 0 
            self.cargar_tabla_gestion() 
        elif nombre == "pos":
            self.filtrar_productos_pos() 
        elif nombre == "finanzas":
            self.actualizar_tabla_finanzas()
        elif nombre == "creditos":
            self.cargar_tabla_creditos()
        elif nombre == "reporte":
            self.cargar_reporte_stock_bajo()
        elif nombre == "menu":
            self.actualizar_estadisticas()
        elif nombre == "clientes":
            self.cargar_tabla_clientes()

        # 3. Transición instantánea (Si no hay animación o es la misma pantalla)
        if not anim or frame_actual is None or frame_actual == frame_destino:
            for f in self.frames.values():
                f.place_forget()
            frame_destino.place(relx=0, rely=0, relwidth=1, relheight=1)
            return

        # 4. LÓGICA DE ANIMACIÓN SLIDE PROFESIONAL
        ancho = self.contenedor.winfo_width()
        if ancho <= 1: ancho = self.root.winfo_width()

        # Configuración de coordenadas según dirección
        if direccion == "right":
            inicio_dest, fin_dest = 1.0, 0.0
            inicio_act, fin_act = 0.0, -1.0
        else:
            inicio_dest, fin_dest = -1.0, 0.0
            inicio_act, fin_act = 0.0, 1.0

        pasos = 15 # Menos pasos = más velocidad
        delta_dest = (fin_dest - inicio_dest) / pasos
        delta_act = (fin_act - inicio_act) / pasos

        # Posicionar destino antes de empezar
        frame_destino.place(relx=inicio_dest, rely=0, relwidth=1, relheight=1)

        def animar(i=0, pos_dest=inicio_dest, pos_act=inicio_act):
            if i >= pasos:
                # Finalización limpia: asegurar posición 0 y ocultar anterior
                frame_destino.place(relx=0, rely=0, relwidth=1, relheight=1)
                frame_actual.place_forget()
                return
            
            # Movimiento de coordenadas (Cálculo puro, sin consultas a DB aquí)
            pos_dest += delta_dest
            pos_act += delta_act
            frame_destino.place(relx=pos_dest, rely=0, relwidth=1, relheight=1)
            frame_actual.place(relx=pos_act, rely=0, relwidth=1, relheight=1)
            
            # 10ms es el estándar para animaciones suaves en Tkinter
            self.root.after(10, animar, i + 1, pos_dest, pos_act)

        animar()


    # -------------------------------------------------
    # PANTALLA MENÚ PRINCIPAL (CON SCROLL)
    # -------------------------------------------------
    def crear_pantalla_menu(self):
        frame = tk.Frame(self.contenedor, bg="#f8f9fa") # Gris muy claro de fondo
        self.frames["menu"] = frame

        # --- SISTEMA DE SCROLL (Mantenemos tu lógica pero optimizada) ---
        canvas = tk.Canvas(frame, bg="#f8f9fa", highlightthickness=0)
        scrollbar = tk.Scrollbar(frame, orient="vertical", command=canvas.yview)
        scroll_frame = tk.Frame(canvas, bg="#f8f9fa")
        
        
        self.canvas_menu_id = canvas.create_window((0, 0), window=scroll_frame, anchor="nw")


        def ajustar_ancho_frame(event):
            # Forzamos que el frame interno tenga siempre el mismo ancho que el canvas visible
            canvas_width = event.width
            canvas.itemconfig(self.canvas_menu_id, width=canvas_width)

        # 3. VINCULACIÓN DE EVENTOS
        canvas.bind('<Configure>', ajustar_ancho_frame)
        scroll_frame.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        
        canvas.configure(yscrollcommand=scrollbar.set)

        # Empaquetado
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")


        # --- CABECERA ---
        header_frame = tk.Frame(scroll_frame, bg="#2c3e50", pady=20)
        header_frame.pack(fill="x")

        try:
            # Carga la imagen (ajusta 'logo.png' al nombre de tu archivo)
            img_open = Image.open("logo.png") 
            # Redimensionado quirúrgico (ej. 100x100 píxeles)
            img_res = img_open.resize((270, 180), Image.Resampling.LANCZOS)
            self.logo_img = ImageTk.PhotoImage(img_res)
            
            lbl_logo = tk.Label(header_frame, image=self.logo_img, bg="#2c3e50")
            lbl_logo.pack(pady=(0, 10))
        except Exception as e:
            # Si no encuentra la imagen, mostramos un icono de texto para que no crashee
            tk.Label(header_frame, text="🏍️", font=("Segoe UI", 40), bg="#2c3e50", fg="white").pack()
            print(f"Aviso: No se pudo cargar el logo: {e}")

        tk.Label(
            header_frame,
            text="MOTO REPUESTOS KRAKEN",
            font=("Segoe UI", 26, "bold"),
            bg="#2c3e50",
            fg="white"
        ).pack()
        
        tk.Label(
            header_frame,
            text="Sistema Integral de Gestión de Inventario y Ventas",
            font=("Segoe UI", 10),
            bg="#2c3e50",
            fg="#bdc3c7"
        ).pack()

        # --- PANEL DE ESTADÍSTICAS (KPIs) ---
        stats_container = tk.Frame(scroll_frame, bg="#f8f9fa", pady=20)
        stats_container.pack(fill="x", padx=50)

        self.stats_labels = {}
        # Definimos colores para cada tarjeta para que se vea más profesional
        kpis = [
            ("Productos", "📦", "#3498db"),
            ("Stock Total", "📊", "#27ae60"),
            ("Inversión", "💰", "#e67e22")
        ]

        for i, (tit, icono, color) in enumerate(kpis):
            card = tk.Frame(stats_container, bg="white", highlightthickness=1, highlightbackground="#dcdde1", padx=20, pady=15)
            card.grid(row=0, column=i, padx=10, sticky="nsew")
            stats_container.grid_columnconfigure(i, weight=1)

            tk.Label(card, text=f"{icono} {tit.upper()}", font=("Segoe UI", 9, "bold"), bg="white", fg="#7f8c8d").pack(anchor="w")
            lbl_val = tk.Label(card, text="--", font=("Segoe UI", 18, "bold"), bg="white", fg=color)
            lbl_val.pack(anchor="w", pady=(5, 0))
            self.stats_labels[tit] = lbl_val

        # --- TASA DE CAMBIO (Compacta y centrada) ---
        tasa_frame = tk.Frame(scroll_frame, bg="#f8f9fa", pady=10)
        tasa_frame.pack()
        
        inner_tasa = tk.Frame(tasa_frame, bg="white", padx=15, pady=8, highlightthickness=1, highlightbackground="#dcdde1")
        inner_tasa.pack()

        tk.Label(inner_tasa, text="💵 TASA USD:", font=("Segoe UI", 9, "bold"), bg="white", fg="#2c3e50").pack(side="left")
        self.ent_tasa = tk.Entry(inner_tasa, width=8, font=("Segoe UI", 11, "bold"), relief="flat", justify="center", bg="#f1f2f6")
        self.ent_tasa.pack(side="left", padx=10)
        
        tk.Button(inner_tasa, text="ACTUALIZAR", command=self.guardar_tasa, bg="#2c3e50", fg="white", 
                  font=("Segoe UI", 8, "bold"), relief="flat", padx=10, cursor="hand2").pack(side="left")

        # --- GRILLA DE MÓDULOS (LA MEJORA PRINCIPAL) ---
        grid_botones = tk.Frame(scroll_frame, bg="#f8f9fa", pady=30)
        grid_botones.pack(expand=True)
        for i in range(3):
            grid_botones.columnconfigure(i, weight=1)

        # Estilo para los botones tipo "Tile" (Mosaico)
        def crear_tile(row, col, texto, subtexto, icono, color, comando):
            btn_frame = tk.Frame(grid_botones, bg="white", highlightthickness=1, 
                                highlightbackground="#dcdde1", width=220, height=160)
            btn_frame.grid(row=row, column=col, padx=10, pady=10, sticky="nsew")
            
            # CLAVE: Evita que el frame se encoja o estire según el texto
            btn_frame.pack_propagate(False) 
            
            # 2. Creamos los elementos internos
            lbl_ico = tk.Label(btn_frame, text=icono, font=("Segoe UI", 24), bg="white", fg=color, cursor="hand2")
            lbl_ico.pack(pady=(15, 5))
            
            lbl_tit = tk.Label(btn_frame, text=texto, font=("Segoe UI", 11, "bold"), bg="white", fg="#2c3e50", cursor="hand2")
            lbl_tit.pack()
            
            lbl_sub = tk.Label(btn_frame, text=subtexto, font=("Segoe UI", 8), bg="white", fg="#95a5a6", cursor="hand2")
            lbl_sub.pack(pady=(0, 15))

            widgets_del_boton = [btn_frame, lbl_ico, lbl_tit, lbl_sub]


            for w in widgets_del_boton: # Puedes añadir más para bindear
                w.bind("<Button-1>", lambda e: comando())

                w.bind("<Enter>", lambda e: [x.config(bg="#f1f2f6") for x in widgets_del_boton])
                w.bind("<Leave>", lambda e: [x.config(bg="white") for x in widgets_del_boton])
                
                # Aseguramos que todos tengan el cursor de mano
                w.config(cursor="hand2")

            for i in range(3):
            # Forzamos a que las 3 columnas midan lo mismo
                grid_botones.grid_columnconfigure(i, weight=1, minsize=230)
            

        # Fila 1
        crear_tile(0, 0, "PUNTO DE VENTA", "Facturación rápida (POS)", "🛒", "#2ecc71", 
                   lambda: self.mostrar_pantalla("pos", direccion="right"))
        
        crear_tile(0, 1, "CUENTAS POR COBRAR", "Gestión de créditos y abonos", "💳", "#3498db", 
                   lambda: self.mostrar_pantalla("creditos", direccion="right"))
        
        crear_tile(0, 2, "ADMINISTRACIÓN", "Control de ingresos y ventas", "💰", "#9b59b6", 
                   lambda: self.mostrar_pantalla("finanzas", direccion="right"))

        # Fila 2
        crear_tile(1, 0, "GESTIONAR STOCK", "Editar y eliminar repuestos", "📦", "#e67e22", 
                   lambda: self.mostrar_pantalla("gestion", direccion="right"))
        
        crear_tile(1, 1, "REGISTRAR PRODUCTO", "Añadir nuevas piezas", "📝", "#1abc9c", 
                   lambda: self.mostrar_pantalla("registro", direccion="right"))
        
        crear_tile(1, 2, "REPORTES CRÍTICOS", "Análisis de stock bajo", "📊", "#e74c3c", 
                   lambda: self.mostrar_pantalla("reporte", direccion="right"))
        crear_tile(2, 1, "VALES DE CANJE", "Ver notas de crédito vigentes", "🎟️", "#9b59b6", 
                   lambda: self.abrir_visor_vales_canje())
        crear_tile(2, 2, "GESTIONAR CLIENTES", "Auditar, editar y borrar deudores", "👥", "#2980b9", 
                   lambda: self.mostrar_pantalla("clientes", direccion="right"))
        
        # Sincronizamos los anclajes de las columnas del grid en el bucle del menú principal
        for i in range(3):
            grid_botones.grid_columnconfigure(i, weight=1, minsize=230)
        
        
        exit_container = tk.Frame(scroll_frame, bg="#f8f9fa", pady=40)
        exit_container.pack(fill="x") # El contenedor se estira, pero el botón NO

        tk.Button(
            exit_container, 
            text="❌ CERRAR SISTEMA", 
            font=("Segoe UI", 10, "bold"), 
            bg="#c0392b", 
            fg="white", 
            relief="flat", 
            width=25,          # <--- ANCHO FIJO: No se estira ni se encoge
            pady=12, 
            command=self.root.destroy, 
            cursor="hand2",
            activebackground="#e74c3c"
        ).pack(anchor="center") 
        
        


       
    def guardar_tasa(self):
        try:
            nueva_tasa = float(self.ent_tasa.get())
            if nueva_tasa <= 0: raise ValueError
            
            # Guardar en la base de datos
            consulta("UPDATE configuracion SET valor=? WHERE clave='tasa_dolar'", (nueva_tasa,), fetch=False)
            self.tasa_actual = nueva_tasa
            
            messagebox.showinfo("Éxito", f"Tasa actualizada a: ${nueva_tasa:.2f}")
        except ValueError:
            messagebox.showerror("Error", "Ingrese un número válido mayor a 0")
        except Exception as e:
            messagebox.showerror("Error", f"No se pudo guardar: {e}")


    def actualizar_estadisticas(self):
        res = consulta(
            "SELECT COUNT(*), IFNULL(SUM(cantidad),0), IFNULL(SUM(precio_costo * cantidad),0) FROM repuestos"
        )[0]
        self.stats_labels["Productos"].config(text=res[0])
        self.stats_labels["Stock Total"].config(text=res[1])
        self.stats_labels["Inversión"].config(text=f"${res[2]:.2f}")

    # -------------------------------------------------
    # PANTALLA REGISTRO
    # -------------------------------------------------
    def crear_pantalla_registro(self):
        frame = tk.Frame(self.contenedor, bg="#2c3e50")
        self.frames["registro"] = frame

        cont = tk.Frame(frame, bg="#2c3e50")
        cont.place(relx=0.5, rely=0.5, anchor="center", relwidth=0.5, relheight=0.9)

        tk.Label(
            cont,
            text="📝 Registrar nuevo producto",
            font=("Segoe UI", 20, "bold"),
            bg="#2c3e50",
            fg="#ffffff"
        ).pack(pady=(0, 20))

        form = tk.Frame(cont, bg="#2c3e50")
        form.pack(pady=10, fill="x")

        self.entry_reg = {}

        campos = [
            ("Código", "codigo"),
            ("Nombre", "nombre"),
            ("Modelo", "modelo"),
            ("Marca", "marca"),
            ("Precio costo", "precio_costo"),
            ("Precio venta", "precio_venta"),
            ("Cantidad", "cantidad"),
        ]

        for i, (label, key) in enumerate(campos):
            fila = tk.Frame(form, bg="#2c3e50")
            fila.pack(fill="x", pady=5)
            tk.Label(
                fila,
                text=label,
                font=("Segoe UI", 10),
                bg="#2c3e50",
                fg="#ffffff",
                width=15,
                anchor="w"
            ).pack(side="left")
            ent = tk.Entry(
                fila,
                font=("Segoe UI", 10),
                bg="white",
                fg="#000000",
                relief="flat",
                highlightthickness=1,
                highlightbackground="#2c3e50"
            )
            ent.pack(side="left", fill="x", expand=True, padx=(0, 10))
            self.entry_reg[key] = ent

        # Fecha automática (no editable)
        fila_fecha = tk.Frame(form, bg="#2c3e50")
        fila_fecha.pack(fill="x", pady=5)
        tk.Label(
            fila_fecha,
            text="Fecha",
            font=("Segoe UI", 10),
            bg="#2c3e50",
            fg="#ffffff",
            width=15,
            anchor="w"
        ).pack(side="left")
        self.lbl_fecha_reg = tk.Label(
            fila_fecha,
            text=datetime.now().strftime("%Y-%m-%d"),
            font=("Segoe UI", 10, "italic"),
            bg="#2c3e50",
            fg="#ffffff"
        )
        self.lbl_fecha_reg.pack(side="left")

        # Botones
        btns = tk.Frame(cont, bg="#2c3e50")
        btns.pack(pady=20)

        tk.Button(
            btns,
            text="Guardar",
            font=("Segoe UI", 11, "bold"),
            bg="#2ecc71",
            fg="white",
            bd=0,
            relief="flat",
            padx=20,
            pady=8,
            cursor="hand2",
            command=self.guardar_producto
        ).pack(side="left", padx=10)

        tk.Button(
            btns,
            text="Volver",
            font=("Segoe UI", 11, "bold"),
            bg="#7f8c8d",
            fg="white",
            bd=0,
            relief="flat",
            padx=20,
            pady=8,
            cursor="hand2",
            command=lambda: self.mostrar_pantalla("menu", direccion="left")
        ).pack(side="left", padx=10)

    def guardar_producto(self):
        datos = {k: e.get().strip() for k, e in self.entry_reg.items()}
        if not all(datos.values()):
            messagebox.showwarning("Campos incompletos", "Completa todos los campos.")
            return

        try:
            precio_costo = float(datos["precio_costo"])
            precio_venta = float(datos["precio_venta"])
            cantidad = int(datos["cantidad"])
        except ValueError:
            messagebox.showerror("Error", "Precio y cantidad deben ser numéricos.")
            return

        margen_minimo = precio_costo * 1.15
        if precio_venta < margen_minimo:
            messagebox.showwarning(
                "Margen Insuficiente", 
                f"¡ALERTA DE NEGOCIO!\n\n"
                f"El precio de venta debe tener al menos un 15% de utilidad.\n"
                f"Mínimo sugerido: ${margen_minimo:.2f}\n"
                f"Monto ingresado: ${precio_venta:.2f}"
            )
            self.entry_reg["precio_venta"].focus_set()
            return

        fecha = datetime.now().strftime("%Y-%m-%d")

        try:
            consulta(
                """
                INSERT INTO repuestos (codigo, nombre, modelo, marca,
                                       precio_costo, precio_venta,
                                       cantidad, fecha)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                """,
                (
                    datos["codigo"],
                    datos["nombre"],
                    datos["modelo"],
                    datos["marca"],
                    precio_costo,
                    precio_venta,
                    cantidad,
                    fecha,
                ),
                fetch=False,
            )
            self.actualizar_estadisticas() # Refresca los KPIs del Menú Principal
            self.cargar_tabla_gestion()    # Refresca la tabla en la pantalla de Gestión
            
            # Limpieza del formulario de registro
            for e in self.entry_reg.values():
                e.delete(0, "end")
            self.lbl_fecha_reg.config(text=datetime.now().strftime("%Y-%m-%d"))

            messagebox.showinfo("Éxito", "Producto registrado correctamente.")
            for e in self.entry_reg.values():
                e.delete(0, "end")
            self.lbl_fecha_reg.config(text=datetime.now().strftime("%Y-%m-%d"))
            self.actualizar_estadisticas()
            self.cargar_tabla_gestion()
        except sqlite3.IntegrityError:
            messagebox.showerror("Error", "Ya existe un producto con ese código.")
            self.entry_reg["codigo"].delete(0, "end")
            self.entry_reg["codigo"].focus_set() 

    # -------------------------------------------------
    # PANTALLA GESTIÓN (TREEVIEW + BÚSQUEDA + EDITAR/BORRAR)
    # -------------------------------------------------
    def crear_pantalla_gestion(self):
        frame = tk.Frame(self.contenedor, bg="#2c3e50")
        self.frames["gestion"] = frame

        top = tk.Frame(frame, bg="#2c3e50")
        top.pack(fill="x", pady=10, padx=10)

        tk.Label(
            top,
            text="📦 Gestión de inventario",
            font=("Segoe UI", 20, "bold"),
            bg="#2c3e50",
            fg="#ffffff"
        ).pack(side="left", padx=(0, 20))

        tk.Button(
            top,
            text="⬅ Volver",
            font=("Segoe UI", 10, "bold"),
            bg="#7f8c8d",
            fg="white",
            bd=0,
            relief="flat",
            padx=15,
            pady=5,
            cursor="hand2",
            command=lambda: self.mostrar_pantalla("menu", direccion="left")
        ).pack(side="right")

        # Búsqueda
        search_frame = tk.Frame(frame, bg="#2c3e50")
        search_frame.pack(fill="x", padx=10, pady=(0, 10))

        tk.Label(
            search_frame,
            text="Buscar:",
            font=("Segoe UI", 10),
            bg="#f4f6f7",
            fg="#2c3e50"
        ).pack(side="left")

        self.entry_buscar = tk.Entry(
            search_frame,
            font=("Segoe UI", 10),
            bg="white",
            fg="#2c3e50",
            relief="flat",
            highlightthickness=2,
            highlightbackground="#2c3e50"
        )
        self.entry_buscar.pack(side="left", padx=8, fill="both", expand=True)
        self.entry_buscar.bind("<KeyRelease>", lambda e: self.cargar_tabla_gestion())

        # Tabla
        tabla_frame = tk.Frame(frame, bg="#f4f6f7")
        tabla_frame.pack(fill="both", expand=True, padx=10, pady=10)

        cols = (
            "codigo",
            "nombre",
            "modelo",
            "marca",
            "precio_costo",
            "precio_venta",
            "cantidad",
            "fecha",
        )
        self.tree = ttk.Treeview(
            tabla_frame,
            columns=cols,
            show="headings",
            selectmode="browse"
        )
        self.tree.bind("<Double-1>", lambda event: self.editar_producto())

        panel_inferior_gestion = tk.Frame(frame, bg="#2c3e50", pady=15)
        panel_inferior_gestion.pack(fill="x", side="bottom")

        nav_frame = tk.Frame(panel_inferior_gestion, bg="#2c3e50")
        nav_frame.pack(fill="x", side="top", padx=20, pady=(0, 10))

        estilo_flechas = {
            "bg": "#34495e",
            "fg": "white",
            "activebackground": "#455a64",
            "activeforeground": "white",
            "font": ("Segoe UI", 9, "bold"),
            "relief": "flat",
            "padx": 15,
            "pady": 6,
            "cursor": "hand2"
        }

        tk.Button(nav_frame, text="◀  ANTERIOR", command=self.pagina_anterior, **estilo_flechas).pack(side="left")
        
        self.lbl_paginacion = tk.Label(nav_frame, text="PÁGINA 1", bg="#2c3e50", fg="#bdc3c7", font=("Segoe UI", 10, "bold"))
        self.lbl_paginacion.pack(side="left", expand=True)

        tk.Button(nav_frame, text="SIGUIENTE  ▶", command=self.pagina_siguiente, **estilo_flechas).pack(side="right")

        btns_container_gest = tk.Frame(panel_inferior_gestion, bg="#2c3e50")
        btns_container_gest.pack(side="bottom", pady=5)

        for i in range(4):
            btns_container_gest.columnconfigure(i, weight=1)

        # Estilo de bloques corporativos unificados (Idéntico a Cuentas por Cobrar/Finanzas)
        estilo_bloque_gest = {
            "relief": "flat", 
            "width": 19,      # Ancho ideal para distribuir 4 elementos en pantalla
            "height": 2,      
            "font": ("Segoe UI", 9, "bold"), 
            "cursor": "hand2"
        }


        self.tree.tag_configure("alerta", background="#ffcccc", foreground="#c0392b")
          # El tag "alerta" pondrá la fila en un tono rojo suave con texto rojo oscuro

        encabezados = [
            "Código",
            "Nombre",
            "Modelo",
            "Marca",
            "P. Costo",
            "P. Venta",
            "Cant.",
            "Fecha",
        ]
        for c, t in zip(cols, encabezados):
            self.tree.heading(c, text=t)
            self.tree.column(c, anchor="center", width=100)

        vsb = ttk.Scrollbar(tabla_frame, orient="vertical", command=self.tree.yview)
        hsb = ttk.Scrollbar(tabla_frame, orient="horizontal", command=self.tree.xview)
        self.tree.configure(yscrollcommand=vsb.set, xscrollcommand=hsb.set)

        self.tree.grid(row=0, column=0, sticky="nsew")
        vsb.grid(row=0, column=1, sticky="ns")
        hsb.grid(row=1, column=0, sticky="ew")

        tabla_frame.grid_rowconfigure(0, weight=1)
        tabla_frame.grid_columnconfigure(0, weight=1)

        # Botones de acción
        btns = tk.Frame(frame, bg="#2c3e50")
        btns.pack(pady=10)

        tk.Button(btns_container_gest, text="GESTIONAR COMBOS", bg="#f1c40f", fg="#2c3e50",
                  activebackground="#f39c12", activeforeground="#2c3e50",
                  command=self.abrir_gestion_combos, **estilo_bloque_gest).grid(row=0, column=0, padx=4, pady=3)

        tk.Button(btns_container_gest, text="EDITAR PRODUCTO", bg="#3498db", fg="white",
                  activebackground="#2980b9", activeforeground="white",
                  command=self.editar_producto, **estilo_bloque_gest).grid(row=0, column=1, padx=4, pady=3)

        tk.Button(btns_container_gest, text="ELIMINAR", bg="#e74c3c", fg="white",
                  activebackground="#c0392b", activeforeground="white",
                  command=self.eliminar_producto, **estilo_bloque_gest).grid(row=0, column=2, padx=4, pady=3)

        tk.Button(btns_container_gest, text="EXPORTAR PDF", bg="#27ae60", fg="white",
                  activebackground="#2ecc71", activeforeground="white",
                  command=self.exportar_inventario_pdf, **estilo_bloque_gest).grid(row=0, column=3, padx=4, pady=3)

        self.cargar_tabla_gestion()

    def cargar_tabla_gestion(self):
        # 1. Bloqueamos el refresco visual para ganar velocidad
        self.tree.configure(displaycolumns=()) 
        
        # 2. Limpieza rápida y preparación de variables
        self.tree.delete(*self.tree.get_children())
        filtro = self.entry_buscar.get().strip() if hasattr(self, "entry_buscar") else ""
        
        # Parámetros de paginación
        offset = self.pagina_actual * self.items_por_pagina

        # 3. Consulta optimizada con paso de parámetros corregido
        if filtro:
            like = f"%{filtro}%"
            query = """SELECT codigo, nombre, modelo, marca, precio_costo, precio_venta, cantidad, fecha 
                       FROM repuestos 
                       WHERE codigo LIKE ? OR nombre LIKE ? OR modelo LIKE ? OR marca LIKE ? 
                       ORDER BY nombre LIMIT ? OFFSET ?"""
            # Pasamos los 4 de búsqueda + los 2 de paginación (Total 6)
            datos = consulta(query, (like, like, like, like, self.items_por_pagina, offset))
        else:
            query = """SELECT codigo, nombre, modelo, marca, precio_costo, precio_venta, cantidad, fecha 
                       FROM repuestos 
                       ORDER BY nombre LIMIT ? OFFSET ?"""
            # Pasamos los 2 de paginación
            datos = consulta(query, (self.items_por_pagina, offset))

        # 4. Inserción masiva
        for d in datos:
            # d[6] es la cantidad/stock
            tag = "alerta" if int(d[6]) <= 5 else ""
            self.tree.insert("", "end", values=d, tags=(tag,))

        # 5. Restauramos las columnas y actualizamos indicador visual
        self.tree.configure(displaycolumns="#all")
        
        if hasattr(self, 'lbl_paginacion'):
            self.lbl_paginacion.config(text=f"Página {self.pagina_actual + 1}")


        

    def editar_producto(self):
        sel = self.tree.selection()
        if not sel:
            messagebox.showwarning("Selecciona un registro", "Selecciona un producto para editar.")
            return
        valores = self.tree.item(sel[0], "values")
        codigo = valores[0]

        win = tk.Toplevel(self.root)
        win.title(f"Editar: {codigo}")
        win.geometry("450x650")
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()

        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (450 // 2)
        y = (win.winfo_screenheight() // 2) - (600 // 2)
        win.geometry("450x600+{}+{}".format(x, y))

        # Ventana de edición simple (puede seguir siendo interna, pero aquí uso Toplevel por simplicidad)
        tk.Label(win, text="✏️ EDITAR PRODUCTO", font=("Segoe UI", 14, "bold"), 
                 fg="#3498db", bg="#2c3e50", pady=20).pack()
    
        form_frame = tk.Frame(win, bg="#2c3e50")
        form_frame.pack(fill="both", expand=True, padx=30)

        campos = [
            ("Codigo", "codigo", valores[0]),
            ("Nombre", "nombre", valores[1]),
            ("Modelo", "modelo", valores[2]),
            ("Marca", "marca", valores[3]),
            ("Precio costo", "precio_costo", valores[4]),
            ("Precio venta", "precio_venta", valores[5]),
            ("Cantidad", "cantidad", valores[6]),
        ]

        entries = {}

        for label, key, val in campos:
            # Quitamos el símbolo '$' si viene en el valor para evitar errores en float()
            val_limpio = str(val).replace('$', '').strip()
            
            fila = tk.Frame(form_frame, bg="#2c3e50")
            fila.pack(fill="x", pady=8)
            
            tk.Label(fila, text=label, font=("Segoe UI", 10, "bold"), 
                     fg="#bdc3c7", bg="#2c3e50", width=12, anchor="w").pack(side="left")
            
            ent = tk.Entry(fila, font=("Segoe UI", 11), bg="white", fg="#2c3e50", 
                           relief="flat", highlightthickness=1, highlightbackground="#34495e")
            ent.pack(side="left", fill="x", expand=True)
            ent.insert(0, val_limpio)
            entries[key] = ent

        def guardar_cambios():
            try:
                nuevo_codigo = entries["codigo"].get().strip()
                nombre = entries["nombre"].get().strip()
                modelo = entries["modelo"].get().strip()
                marca = entries["marca"].get().strip()
                precio_costo = float(entries["precio_costo"].get())
                precio_venta = float(entries["precio_venta"].get())
                cantidad = int(entries["cantidad"].get())
            except ValueError:
                messagebox.showerror("Error", "Precio y cantidad deben ser numéricos.")
                return
            
            margen_minimo = precio_costo * 1.15
            if precio_venta < margen_minimo:
                messagebox.showwarning(
                    "Margen insuficiente", 
                    f"No se pueden guardar los cambios.\n\n"
                    f"Para mantener la rentabilidad, el precio debe ser al menos: ${margen_minimo:.2f}",
                    parent=win
                )
                entries["precio_venta"].focus_set()
                return

            consulta(
                """
                UPDATE repuestos
                SET codigo=?, nombre=?, modelo=?, marca=?,
                    precio_costo=?, precio_venta=?, cantidad=?
                WHERE codigo=?
                """,
                (
                    nuevo_codigo,   # 1
                    nombre,         # 2
                    modelo,         # 3
                    marca,          # 4
                    precio_costo,   # 5
                    precio_venta,   # 6
                    cantidad,       # 7
                    codigo 
                ),
                fetch=False,
            )
            messagebox.showinfo("Éxito", "Producto actualizado.")
            win.destroy()
            self.cargar_tabla_gestion()
            self.actualizar_estadisticas()

        btns_modal_frame = tk.Frame(win, bg="#2c3e50")
        btns_modal_frame.pack(side="bottom", fill="x", padx=40, pady=(10, 30))

        # Configuramos 2 columnas con el mismo peso para simetría exacta
        btns_modal_frame.columnconfigure(0, weight=1)
        btns_modal_frame.columnconfigure(1, weight=1)

        # Diccionario de estilos compartidos tipo bloque ejecutivo
        estilo_modal_btn = {
            "relief": "flat",
            "height": 2,
            "font": ("Segoe UI", 9, "bold"),
            "cursor": "hand2"
        }

        # Botón 1: Guardar Cambios (Lado izquierdo, verde esmeralda)
        btn_save = tk.Button(
            btns_modal_frame, 
            text="GUARDAR CAMBIOS", 
            bg="#27ae60", 
            fg="white", 
            activebackground="#2ecc71", 
            activeforeground="white",
            command=guardar_cambios, 
            **estilo_modal_btn
        )
        btn_save.grid(row=0, column=0, padx=(0, 6), sticky="nsew")

        # Botón 2: Cancelar (Lado derecho, rojo granate)
        btn_cancel = tk.Button(
            btns_modal_frame, 
            text="CANCELAR", 
            bg="#c0392b", 
            fg="white", 
            activebackground="#e74c3c", 
            activeforeground="white",
            command=win.destroy, 
            **estilo_modal_btn
        )
        btn_cancel.grid(row=0, column=1, padx=(6, 0), sticky="nsew")

        # Bindeo de la tecla Enter al campo de precio venta para guardar rápido
        entries["precio_venta"].bind("<Return>", lambda e: guardar_cambios())

    def eliminar_producto(self):
        sel = self.tree.selection()
        if not sel:
            messagebox.showwarning("Selecciona un registro", "Selecciona un producto para eliminar.")
            return
        valores = self.tree.item(sel[0], "values")
        codigo = valores[0]

        if messagebox.askyesno("Confirmar", f"¿Eliminar el producto {codigo}?"):
            consulta("DELETE FROM repuestos WHERE codigo=?", (codigo,), fetch=False)
            self.cargar_tabla_gestion()
            self.actualizar_estadisticas()
            messagebox.showinfo("Eliminado", "Producto eliminado correctamente.")

    def pagina_siguiente(self):
        # 1. Contar cuántos productos hay en total (según el filtro actual)
        filtro = self.entry_buscar.get().strip() if hasattr(self, "entry_buscar") else ""
        
        if filtro:
            like = f"%{filtro}%"
            res = consulta("SELECT COUNT(*) FROM repuestos WHERE codigo LIKE ? OR nombre LIKE ? OR modelo LIKE ? OR marca LIKE ?", 
                           (like, like, like, like))
        else:
            res = consulta("SELECT COUNT(*) FROM repuestos")
        
        total_productos = res[0][0] if res else 0
        
        # 2. Calcular el límite máximo de páginas
        # Ejemplo: 105 productos / 50 por página = 2.1 -> Necesitamos 3 páginas (0, 1, 2)
        import math
        total_paginas = math.ceil(total_productos / self.items_por_pagina)

        # 3. Solo avanzar si no hemos llegado a la última página
        if self.pagina_actual + 1 < total_paginas:
            self.pagina_actual += 1
            self.cargar_tabla_gestion()
        else:
            messagebox.showinfo("Fin de lista", "Has llegado al final de los registros disponibles.")

    def pagina_anterior(self):
        # Esta lógica es más simple: no podemos bajar de la página 0
        if self.pagina_actual > 0:
            self.pagina_actual -= 1
            self.cargar_tabla_gestion()

    def exportar_inventario_pdf(self):
        datos = consulta("SELECT codigo, nombre, modelo, marca, precio_costo, precio_venta, cantidad, fecha FROM repuestos ORDER BY nombre")
        encabezados = ["CÓDIGO", "NOMBRE", "MODELO", "MARCA", "COSTO", "VENTA", "CANT.", "FECHA"]
        if not datos:
          messagebox.showinfo("Reporte vacío", "No hay productos en el inventario para exportar.")
          return
        self.exportar_a_pdf("INVENTARIO COMPLETO DE REPUESTOS", datos, encabezados)
        

    # -------------------------------------------------
    # PANTALLA REPORTE STOCK BAJO
    # -------------------------------------------------
    def crear_pantalla_reporte(self):
        frame = tk.Frame(self.contenedor, bg="#2c3e50")
        self.frames["reporte"] = frame

        top = tk.Frame(frame, bg="#2c3e50")
        top.pack(fill="x", pady=10, padx=10)

        tk.Label(
            top,
            text="📊 Reporte de stock bajo",
            font=("Segoe UI", 20, "bold"),
            bg="#2c3e50",
            fg="#ffffff"
        ).pack(side="left", padx=(0, 20))

        tk.Button(
            top,
            text="⬅ Volver",
            font=("Segoe UI", 10, "bold"),
            bg="#7f8c8d",
            fg="white",
            bd=0,
            relief="flat",
            padx=15,
            pady=5,
            cursor="hand2",
            command=lambda: self.mostrar_pantalla("menu", direccion="left")
        ).pack(side="right")

        filtro_frame = tk.Frame(frame, bg="#2c3e50")
        filtro_frame.pack(fill="x", padx=10, pady=(0, 10))

        tk.Label(
            filtro_frame,
            text="Stock mínimo:",
            font=("Segoe UI", 10),
            bg="#f4f6f7",
            fg="#2c3e50"
        ).pack(side="left")

        self.entry_min_stock = tk.Entry(
            filtro_frame,
            font=("Segoe UI", 10),
            width=10,
            bg="white",
            fg="#2c3e50",
            relief="flat",
            highlightthickness=1,
            highlightbackground="#dcdde1"
        )
        self.entry_min_stock.pack(side="left", padx=5)
        self.entry_min_stock.insert(0, "5")

        self.entry_min_stock.bind("<KeyRelease>", lambda e: self.cargar_reporte_stock_bajo())

        tk.Button(
            filtro_frame,
            text="📥 Exportar PDF",
            relief="flat",
            font=("Segoe UI", 9, "bold"),
            bg="#2ecc71", # Color verde para diferenciarlo
            fg="white",
            bd=0,
            padx=10,
            pady=4,
            cursor="hand2",
            command=self.exportar_stock_bajo_pdf # Llamada a la función
        ).pack(side="left", padx=5)

        # Tabla
        tabla_frame = tk.Frame(frame, bg="#f4f6f7")
        tabla_frame.pack(fill="both", expand=True, padx=10, pady=10)

        cols = ("codigo", "nombre", "marca", "modelo", "cantidad")
        self.tree_reporte = ttk.Treeview(
            tabla_frame,
            columns=cols,
            show="headings",
            selectmode="none"
        )

        encabezados = ["Código", "Nombre", "Marca", "Modelo", "Cantidad"]
        for c, t in zip(cols, encabezados):
            self.tree_reporte.heading(c, text=t)
            self.tree_reporte.column(c, anchor="center", width=120)

        vsb = ttk.Scrollbar(tabla_frame, orient="vertical", command=self.tree_reporte.yview)
        self.tree_reporte.configure(yscrollcommand=vsb.set)

        self.tree_reporte.grid(row=0, column=0, sticky="nsew")
        vsb.grid(row=0, column=1, sticky="ns")

        tabla_frame.grid_rowconfigure(0, weight=1)
        tabla_frame.grid_columnconfigure(0, weight=1)

        self.cargar_reporte_stock_bajo()

    def cargar_reporte_stock_bajo(self):
        for row in self.tree_reporte.get_children():
            self.tree_reporte.delete(row)


        valor = self.entry_min_stock.get().strip()

        if not valor:
               return

        try:
            min_stock = int(valor)
        except ValueError:
            # Si escriben letras, detenemos la ejecución sin mostrar mensaje de error
            return

        datos = consulta(
            """
            SELECT codigo, nombre, marca, modelo, cantidad
            FROM repuestos
            WHERE cantidad <= ?
            ORDER BY cantidad ASC
            """,
            (min_stock,),
        )

        for d in datos:
            self.tree_reporte.insert("", "end", values=d)

    def exportar_stock_bajo_pdf(self):
        try:
            min_stock = int(self.entry_min_stock.get())
        except ValueError:
          messagebox.showerror("Error", "El stock mínimo debe ser un número entero para exportar.")
          return
    
        datos = consulta(
        "SELECT codigo, nombre, marca, modelo, cantidad FROM repuestos WHERE cantidad <= ? ORDER BY cantidad ASC", 
        (min_stock,)
        )

        if not datos:
          messagebox.showinfo("Reporte vacío", "No hay productos con stock bajo para exportar.")
          return
    
        encabezados = ["CÓDIGO", "NOMBRE DEL PRODUCTO", "MARCA", "MODELO", "CANT"]
     
        self.exportar_a_pdf(
        f"REPORTE DE STOCK CRÍTICO (Menor o igual a {min_stock} unidades)", 
        datos, 
        encabezados
    )

          # --- AÑADIR ESTA IMPORTACIÓN AL INICIO DEL ARCHIVO ---


        # =================================================
        # MÓDULO DE ADMINISTRACIÓN FINANCIERA (AÑADIR A LA CLASE)
        # =================================================

    def crear_pantalla_finanzas(self):
        """Crea la interfaz de administración financiera y reportes de ventas."""
        #frame = tk.Toplevel(self.root)
        frame = tk.Frame(self.contenedor, bg="#f4f6f7")
        self.frames["finanzas"] = frame
        #frame.title("Administración Financiera y Control de Ventas")
        #frame.geometry("1000x700")
        #frame.configure(bg="#f4f6f7")
        #frame.grab_set()

        # --- TÍTULO Y FILTROS ---
        top = tk.Frame(frame, bg="#2c3e50", pady=15)
        top.pack(fill="x")

        tk.Button(
            top, # O el nombre que tenga tu frame superior (ej. 'top')
            text="⬅ Volver",
            font=("Segoe UI", 10, "bold"),
            bg="#7f8c8d",
            fg="white",
            bd=0,
            relief="flat",
            padx=15,
            pady=5,
            cursor="hand2",
            command=lambda: self.mostrar_pantalla("menu", direccion="left")
        ).pack(side="right", padx=10)
        tk.Label(top, text="📊 CONTROL DE INGRESOS Y VENTAS", font=("Segoe UI", 16, "bold"), fg="white", bg="#2c3e50").pack()

        filter_frame = tk.Frame(frame, bg="#f4f6f7", pady=10)
        filter_frame.pack(fill="x", padx=20)

        tk.Label(filter_frame, text="🔍 Buscar Factura:", font=("Segoe UI", 9, "bold"), bg="#f4f6f7").pack(side="left", padx=(20, 0))
        self.ent_buscar_factura = tk.Entry(filter_frame, font=("Segoe UI", 10), width=15)
        self.ent_buscar_factura.pack(side="left", padx=5)
        
        # Evento: filtra mientras escribes
        self.ent_buscar_factura.bind("<KeyRelease>", lambda e: self.actualizar_tabla_finanzas())

        tk.Label(filter_frame, text="Escala:", font=("Segoe UI", 9, "bold"), bg="#f4f6f7").pack(side="left")
        self.var_escala = tk.StringVar(value="Día")
        self.combo_escala = ttk.Combobox(filter_frame, textvariable=self.var_escala, values=["Día", "Mes", "Año", "Todo"], state="readonly", width=10)
        self.combo_escala.pack(side="left", padx=5)
        
        tk.Label(filter_frame, text="Seleccionar:", font=("Segoe UI", 9, "bold"), bg="#f4f6f7").pack(side="left", padx=(10, 0))
        self.var_valor = tk.StringVar()
        self.combo_valor = ttk.Combobox(filter_frame, textvariable=self.var_valor, state="readonly", width=20)
        self.combo_valor.pack(side="left", padx=5)

        self.combo_escala.bind("<<ComboboxSelected>>", lambda e: self.cargar_valores_disponibles())
        self.combo_valor.bind("<<ComboboxSelected>>", lambda e: self.actualizar_tabla_finanzas())

        

        # --- TABLA DE VENTAS ---
        tabla_frame = tk.Frame(frame, bg="white")
        tabla_frame.pack(fill="both", expand=True, padx=20, pady=10)

        cols = ("id", "fecha", "productos", "total", "ganancia")
        self.tree_finanzas = ttk.Treeview(tabla_frame, columns=cols, show="headings")
        
        headers = {"id": "ID", "fecha": "Fecha", "productos": "Nro. Factura", "total": "Total ($)", "ganancia": "Ganancia Est. ($)"}
        for c, h in headers.items():
            self.tree_finanzas.heading(c, text=h)
            self.tree_finanzas.column(c, width=150 if c in ['id', 'total'] else 200, anchor="center")
            if c == "id": ancho = 50
            self.tree_finanzas.column(c, width=ancho, anchor="center")

            self.tree_finanzas.bind("<Double-1>", self.mostrar_detalle_emergente)
        
        self.tree_finanzas.pack(side="left", fill="both", expand=True)
        
        scroll = ttk.Scrollbar(tabla_frame, orient="vertical", command=self.tree_finanzas.yview)
        scroll.pack(side="right", fill="y")
        self.tree_finanzas.configure(yscrollcommand=scroll.set)

        # --- RESUMEN FINANCIERO ---
        resumen_frame = tk.Frame(frame, bg="#ecf0f1", pady=15)
        resumen_frame.pack(fill="x", side="bottom")

        kpi_finanzas_frame = tk.Frame(resumen_frame, bg="#ecf0f1", pady=10)
        kpi_finanzas_frame.pack(fill="x", side="top", padx=20, pady=(0, 15))

        self.lbl_ingreso_bruto = tk.Label(resumen_frame, text="Ingreso Bruto: $0.00", font=("Segoe UI", 12, "bold"), fg="#2c3e50", bg="#ecf0f1")
        self.lbl_ingreso_bruto.pack(side="left", padx=40)

        self.lbl_caja_usd = tk.Label(kpi_finanzas_frame, text="Caja USD ($): $0.00", font=("Segoe UI", 11, "bold"), fg="#27ae60", bg="#ecf0f1")
        self.lbl_caja_usd.pack(side="left", padx=25)

        # Contador 3: Caja Exclusiva de Bolívares (Punto/Pago Móvil)
        self.lbl_caja_bs = tk.Label(kpi_finanzas_frame, text="Caja Bs. (Bs.): Bs. 0.00", font=("Segoe UI", 11, "bold"), fg="#d35400", bg="#ecf0f1")
        self.lbl_caja_bs.pack(side="left", padx=25)

        self.lbl_ingreso_neto = tk.Label(resumen_frame, text="Ganancia Neta: $0.00", font=("Segoe UI", 12, "bold"), fg="#27ae60", bg="#ecf0f1")
        self.lbl_ingreso_neto.pack(side="left", padx=40)

        btns_container_fin = tk.Frame(resumen_frame, bg="#f4f6f7")
        btns_container_fin.pack(side="bottom", pady=5)

        for i in range(3):
            btns_container_fin.columnconfigure(i, weight=1)

        # ESTILO EN BLOQUE IDÉNTICO A TU IMAGEN
        estilo_sq_fin = {
            "relief": "flat", 
            "width": 18,      
            "height": 2,      
            "font": ("Segoe UI", 9, "bold"), 
            "cursor": "hand2"
        }


        tk.Button(btns_container_fin, text="DEVOLUCIÓN", bg="#c0392b", fg="white",
                  command=self.procesar_devolucion, **estilo_sq_fin).grid(row=0, column=0, padx=3, pady=3)

        tk.Button(btns_container_fin, text="EXPORTAR PDF", bg="#d35400", fg="white",
                  command=self.exportar_finanzas_pdf, **estilo_sq_fin).grid(row=0, column=1, padx=3, pady=3)
        
                # Agrega este botón en la fila 1 de la botonera btns_container_fin de tu pantalla de finanzas:
        tk.Button(btns_container_fin, text="PROCESAR CAMBIO", bg="#2980b9", fg="white",
                  command=self.procesar_nota_credito_cambio, **estilo_sq_fin).grid(row=1, column=0, padx=3, pady=3)


        
        
        self.cargar_valores_disponibles() 
        self.actualizar_tabla_finanzas()

    def procesar_devolucion(self):
        """Procesa la anulación de facturas de forma compatible con sellos multimoneda."""
        sel = self.tree_finanzas.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione la factura para devolver.")
            return

        valores = self.tree_finanzas.item(sel, "values")
        id_venta = valores[0]
        tags = self.tree_finanzas.item(sel, "tags")
        detalle_productos = tags[0] if tags else ""

        if not messagebox.askyesno("Anulación de Factura", 
                                   f"¿Desea anular la factura {valores[2]}?\n\n"
                                   "Los productos se sumarán de nuevo al inventario."):
            return

        try:
            # Separamos cada elemento registrado en la columna 'productos'
            items = detalle_productos.split(", ")
            
            for item_str in items:
                # --- FILTRADO QUIRÚRGICO: IGNORAR EL SELLO MULTIMONEDA ---
                # Si el fragmento es el método de pago, saltamos a la siguiente iteración
                if "MÉTODO" in item_str:
                    continue
                
                # Extraemos el código que está entre [ ]
                if "[" in item_str and "]" in item_str:
                    codigo = item_str[item_str.find("[")+1 : item_str.find("]")]
                    
                    # Extraemos la cantidad de forma segura mitigando fallos de formato
                    try:
                        if " x" in item_str:
                            parte_cant = item_str.split(" x")[1].split(" ")[0]
                            cantidad_a_sumar = int(parte_cant)
                            
                            # ACTUALIZACIÓN DIRECTA POR CÓDIGO EN EL ALMACÉN
                            consulta("UPDATE repuestos SET cantidad = cantidad + ? WHERE codigo = ?", 
                                     (cantidad_a_sumar, codigo), fetch=False)
                    except (IndexError, ValueError) as err:
                        print(f"DEBUG: Registro omitido o ilegible ({item_str}): {err}")
                        continue

            # Eliminar de forma permanente el registro de la venta en la base contable
            consulta("DELETE FROM ventas_log WHERE id = ?", (id_venta,), fetch=False, db_path=DB_VENTAS)

            messagebox.showinfo("Éxito", "Devolución procesada. El inventario ha sido restaurado con éxito.")
            
            # Refrescar en cascada todas las interfaces de forma unificada
            self.actualizar_tabla_finanzas()
            self.actualizar_estadisticas()
            self.cargar_tabla_gestion()
            
        except Exception as e:
            messagebox.showerror("Error Crítico", f"No se pudo procesar la devolución: {e}")



        

    def actualizar_tabla_finanzas(self):
        """Lógica avanzada que preserva los tags de productos, aplica colores por canje y busca por CEDULA."""
        for row in self.tree_finanzas.get_children():
            self.tree_finanzas.delete(row)
            
        escala = self.var_escala.get()
        valor = self.var_valor.get()
        busqueda = self.ent_buscar_factura.get().strip()
        
        # 1. Traemos el costo_total guardado en la transacción original
        query = "SELECT id, date(fecha_hora), productos, total, costo_total FROM ventas_log WHERE 1=1"
        params = []

        if escala != "Todo" and valor:
            formatos = {"Día": "%Y-%m-%d", "Mes": "%Y-%m", "Año": "%Y"}
            query += f" AND strftime('{formatos[escala]}', fecha_hora) = ?"
            params.append(valor)

        # --- MOTOR DE BÚSQUEDA MULTICRITERIO SEGURO (ID, PRODUCTO O CÉDULA) ---
        if busqueda:
            # Saneamos rigurosamente la entrada para auditorías de cédulas directas
            busqueda_limpia = busqueda.replace(".", "").replace("-", "").upper()
            # Filtramos solo caracteres numéricos por si el cajero digita "V-20123456" o "v20123456"
            solo_numeros_cedula = "".join([char for char in busqueda_limpia if char.isdigit()])
            
            id_buscado = busqueda.replace("FACT-", "").replace("fact-", "").lstrip('0')
            
            # Criterio A: Si es un número corto (4 o menos dígitos), se procesa como ID de factura
            if id_buscado.isdigit() and len(id_buscado) <= 4:
                query += " AND id = ?"
                params.append(int(id_buscado))
                
            # Criterio B: Si posee 5 o más dígitos puros, el motor asume de forma estricta que es la CÉDULA
            elif len(solo_numeros_cedula) >= 5:
                query += " AND productos LIKE ?"
                # Escanea el sello inyectado dinámicamente al final de la columna flexible 'productos'
                params.append(f"%CEDULA: {solo_numeros_cedula}%")
                
            # Criterio C: Si es texto alfabético plano, ejecuta la coincidencia regular de repuestos
            else:
                query += " AND productos LIKE ?"
                params.append(f"%{busqueda}%")

        ventas = consulta(query + " ORDER BY id DESC", params, db_path=DB_VENTAS)

        # Configurar colores de realce corporativo en el Treeview
        self.tree_finanzas.tag_configure("factura_canje", background="#e8f4fd", foreground="#1e3d59")

        total_general_usd = 0.0
        caja_efectivo_usd = 0.0
        caja_digital_bs = 0.0
        ganancia_neta_global = 0.0

        for v in ventas:
            id_venta = v[0]
            fecha_v = v[1]
            detalle_v = v[2] if v[2] is not None else ""
            monto_v = v[3] if v[3] is not None else 0.0
            costo_registrado = v[4] if v[4] is not None else 0.0
            
            num_factura = f"FACT-{str(id_venta).zfill(5)}"
            ganancia_v = round(monto_v - costo_registrado, 2)

            # Escaneo analítico de moneda de recepción
            if "BOLÍVARES" in detalle_v:
                try:
                    tasa_str = detalle_v.split("TASA REF: ")[1].split("]")[0]
                    tasa_transaccion = float(tasa_str)
                except:
                    tasa_transaccion = self.tasa_actual
                caja_digital_bs += (monto_v * tasa_transaccion)
                tipo_pago_tag = "Bs."
            else:
                caja_efectivo_usd += monto_v
                tipo_pago_tag = "$"

            # --- CONSERVACIÓN CRÍTICA DE METADATOS Y TAGS DE COLOR ---
            if "CANJE" in detalle_v or "NC-" in detalle_v:
                num_factura_con_moneda = f"{num_factura} ({tipo_pago_tag}) [CANJE]"
                tags_estructurados = (detalle_v, "factura_canje")
            else:
                num_factura_con_moneda = f"{num_factura} ({tipo_pago_tag})"
                tags_estructurados = (detalle_v,)

            total_general_usd += monto_v
            ganancia_neta_global += ganancia_v

            self.tree_finanzas.insert("", "end", values=(
                id_venta, 
                fecha_v, 
                num_factura_con_moneda, 
                f"${monto_v:.2f}", 
                f"${ganancia_v:.2f}"
            ), tags=tags_estructurados)

        # Actualización de la barra de KPIs e información de caja inferior
        if hasattr(self, 'lbl_ingreso_bruto'): self.lbl_ingreso_bruto.config(text=f"Total General: ${total_general_usd:,.2f}")
        if hasattr(self, 'lbl_caja_usd'): self.lbl_caja_usd.config(text=f"Caja USD: ${caja_efectivo_usd:,.2f}")
        if hasattr(self, 'lbl_caja_bs'): self.lbl_caja_bs.config(text=f"Caja Bs.: Bs. {caja_digital_bs:,.2f}")
        if hasattr(self, 'lbl_ingreso_neto'): self.lbl_ingreso_neto.config(text=f"Ganancia Neta: ${ganancia_neta_global:,.2f}")





    def exportar_finanzas_pdf(self):
        """Generación de reporte financiero con desglose explícito de Notas de Crédito y Canjes."""
        try:
            ruta_finanzas = os.path.join(os.getcwd(), "REPORTES_KRAKEN", "FINANZAS")
            if not os.path.exists(ruta_finanzas):
                os.makedirs(ruta_finanzas)

            fecha_id = datetime.now().strftime("%Y-%m-%d_%H-%M")
            nombre_sugerido = f"REPORTE_VENTAS_{fecha_id}.pdf"

            archivo = filedialog.asksaveasfilename(
                initialdir=ruta_finanzas, initialfile=nombre_sugerido,
                defaultextension=".pdf", filetypes=[("PDF", "*.pdf")], title="Guardar Reporte Financiero"
            )
            if not archivo: return

            pdf = FPDF()
            pdf.add_page()
            pdf.set_font("helvetica", "B", 16)

            escala = self.var_escala.get()
            valor = self.var_valor.get()
            periodo_texto = f"Filtro: {escala} ({valor})" if escala != "Todo" else "Historial Completo"
            
            pdf.cell(0, 10, "MOTO REPUESTOS KRAKEN", ln=True, align="C")
            pdf.set_font("helvetica", "", 10)
            pdf.cell(0, 10, f"Periodo: {periodo_texto} | Generado: {datetime.now().strftime('%d/%m/%Y %H:%M')}", align="C")
            pdf.ln(10)

            # Estructuración de columnas (ID, Fecha, Factura, Total, Ganancia, Observaciones)
            pdf.set_fill_color(44, 62, 80)
            pdf.set_text_color(255, 255, 255)
            pdf.set_font("helvetica", "B", 9)
            
            # Reorganizamos anchos: ID(12), Fecha(28), Factura(35), Total(25), Ganancia(25), Observación(65)
            cols = [("ID", 12), ("Fecha", 28), ("Nro. Factura", 35), ("Total $", 25), ("Ganancia $", 25), ("Observación / Canjes", 65)]
            for col, width in cols:
                pdf.cell(width, 10, col, border=1, align="C", fill=True)
            pdf.ln()

            pdf.set_text_color(0, 0, 0)
            pdf.set_font("helvetica", "", 8)
            
            items = self.tree_finanzas.get_children()
            sumatoria_tasas = 0.0
            facturas_con_tasa = 0
            
            for item in items:
                v = self.tree_finanzas.item(item, 'values')
                tags_raw = self.tree_finanzas.item(item, 'tags')
                detalle_v = str(tags_raw) if tags_raw else ""

                total_v = float(str(v[3]).replace('$', '').replace(',', '').strip())
                ganancia_v = float(str(v[4]).replace('$', '').replace(',', '').strip())
                
                # --- EXTRACCIÓN EXTRA-PRECISA DE NOTAS DE CRÉDITO PARA EL PDF ---
                observacion_pdf = "Regular"
                if "NC-" in detalle_v:
                    try:
                        # Buscamos y aislamos el código de la nota de crédito
                        nc_str = detalle_v.split("VALE ")[1].split("]")[0]
                        observacion_pdf = f"Aplica Canje: {nc_str}"
                    except:
                        observacion_pdf = "Aplica Canje / Devolución"

                if "TASA REF: " in detalle_v:
                    try:
                        tasa_str = detalle_v.split("TASA REF: ")[1].split("]")[0].strip()
                        sumatoria_tasas += float(tasa_str)
                        facturas_con_tasa += 1
                    except: pass
                
                # Renderizado de celdas
                pdf.cell(12, 9, str(v[0]), border=1, align="C")
                pdf.cell(28, 9, str(v[1]), border=1, align="C")
                pdf.cell(35, 9, str(v[2]), border=1, align="L")
                pdf.cell(25, 9, f"${total_v:.2f}", border=1, align="C")
                pdf.cell(25, 9, f"${ganancia_v:.2f}", border=1, align="C")
                
                # Imprimir la glosa del canje destacada
                if "Canje" in observacion_pdf:
                    pdf.set_text_color(211, 84, 0) # Naranja para canjes en la tabla
                    pdf.cell(65, 9, observacion_pdf, border=1, align="L")
                    pdf.set_text_color(0, 0, 0)
                else:
                    pdf.cell(65, 9, observacion_pdf, border=1, align="L")
                pdf.ln()

            tasa_promedio_dia = (sumatoria_tasas / facturas_con_tasa) if facturas_con_tasa > 0 else self.tasa_actual

            # Balance macro inferior
            pdf.ln(10)
            pdf.set_font("helvetica", "B", 11)
            pdf.cell(0, 10, "ARQUEO DE CAJA Y BALANCE DEL PERIODO:", ln=True)
            pdf.set_font("helvetica", "", 10)

            bruto_str = self.lbl_ingreso_bruto.cget("text").replace("Total General: $", "").replace(",", "")
            caja_usd_str = self.lbl_caja_usd.cget("text").replace("Caja USD: $", "").replace(",", "")
            caja_bs_str = self.lbl_caja_bs.cget("text").replace("Caja Bs.: Bs. ", "").replace(",", "")
            neto_str = self.lbl_ingreso_neto.cget("text").replace("Ganancia Neta: $", "").replace(",", "")

            bruto_f = float(bruto_str) if bruto_str.strip() else 0.0
            neto_f = float(neto_str) if neto_str.strip() else 0.0
            margen_total = (neto_f / bruto_f * 100) if bruto_f > 0 else 0

            pdf.cell(0, 7, f"- Ingreso Bruto Total Consolidado: ${bruto_f:,.2f}", ln=True)
            pdf.set_text_color(39, 174, 96)
            pdf.cell(0, 7, f"- Caja USD Efectivo / Custodia Digital: ${float(caja_usd_str):,.2f}", ln=True)
            pdf.set_text_color(211, 84, 0)
            pdf.cell(0, 7, f"- Caja Bs. Punto / Pago Móvil Liquidado: Bs. {float(caja_bs_str):,.2f}", ln=True)
            pdf.set_text_color(142, 68, 173)
            pdf.cell(0, 7, f"- TASA DE COBRO PROMEDIO REGISTRADA: {tasa_promedio_dia:.2f} Bs/USD", ln=True)
            pdf.set_text_color(44, 62, 80)
            pdf.cell(0, 7, f"- Utilidad Neta Estimada del Periodo: ${neto_f:,.2f}", ln=True)

            pdf.ln(2)
            pdf.set_text_color(41, 128, 185) 
            pdf.set_font("helvetica", "B", 11)
            pdf.cell(0, 8, f"- MARGEN DE RENDIMIENTO COMERCIAL PROMEDIO: {margen_total:.2f}%", ln=True)

            pdf.set_y(-15)
            pdf.set_text_color(149, 165, 166)
            pdf.set_font("helvetica", "I", 8)
            pdf.cell(0, 10, f"Página {pdf.page_no()}", align="C")

            pdf.output(archivo)
            messagebox.showinfo("Éxito", "Reporte financiero exportado con segregación estricta de canjes.")
        except Exception as e:
            messagebox.showerror("Error PDF", f"Error crítico al compilar el reporte: {e}")





    def vaciar_carrito_completo(self):
        """Limpia quirúrgicamente toda la orden actual."""
        if not self.carrito:
            messagebox.showwarning("Carrito Vacío", "No hay elementos en el carrito para eliminar.")
            return

        # Confirmación de seguridad con estilo profesional
        if messagebox.askyesno("Confirmar Acción", "¿Desea eliminar TODOS los productos del carrito?\nEsta acción no se puede deshacer."):
            self.carrito = [] # Reset de datos
            self.actualizar_tabla_pos() # Reset visual y de totales
            # Notificación sutil en consola o status si existiera

    def cargar_valores_disponibles(self):
        """Extrae de forma quirúrgica los periodos existentes en los registros."""
        escala = self.var_escala.get()
        
        if escala == "Todo":
            self.combo_valor.config(values=["Todos los registros"], state="disabled")
            self.var_valor.set("Todos los registros")
            self.actualizar_tabla_finanzas()
            return
        
        self.combo_valor.config(state="readonly")
        
        # Formatos SQL según la escala
        formatos = {"Día": "%Y-%m-%d", "Mes": "%Y-%m", "Año": "%Y"}
        sql_format = formatos[escala]
        
        # Obtenemos valores únicos que existen en la tabla de ventas
        query = f"SELECT DISTINCT strftime('{sql_format}', fecha_hora) as periodo FROM ventas_log ORDER BY periodo DESC"
        resultados = consulta(query, db_path=DB_VENTAS)
        opciones = [r[0] for r in resultados]
        
        self.combo_valor.config(values=opciones)
        if opciones:
            self.var_valor.set(opciones[0]) # Selecciona el más reciente por defecto
        
        self.actualizar_tabla_finanzas()

    def aplicar_descuento_item(self):
        sel = self.tree_pos.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione un producto del detalle de venta para aplicar descuento.")
            return

        idx = self.tree_pos.index(sel[0])
        item = self.carrito[idx]

        try:
            tasa_c = float(self.ent_tasa_cobro.get())
        except:
            tasa_c = self.tasa_actual
        
            

        precio_ajustado = (item['precio'] * tasa_c) / self.tasa_actual   
        precio_costo_base = item['costo']
        
        # Crear ventana de diálogo personalizada
        win = tk.Toplevel(self.root)
        win.title(f"Descuento - {item['nombre']}")
        win.geometry("500x600")  # ← TAMAÑO CORREGIDO
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()
        win.resizable(False, False)
        
        # Centrar ventana
        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (win.winfo_width() // 2)
        y = (win.winfo_screenheight() // 2) - (win.winfo_height() // 2)
        win.geometry(f"+{x}+{y}")
        
        # Contenedor principal con scroll por si acaso
        main_frame = tk.Frame(win, bg="#2c3e50")
        main_frame.pack(fill="both", expand=True, padx=10, pady=10)
        
        # Título principal
        tk.Label(main_frame, relief="flat", text="🏷️ APLICAR DESCUENTO", font=("Segoe UI", 14, "bold"), 
                 fg="#f39c12", bg="#2c3e50").pack(pady=10)
        
        # Info del producto
        info_frame = tk.Frame(main_frame, bg="#34495e", padx=15, pady=10)
        info_frame.pack(fill="x", pady=5)
        
        tk.Label(info_frame, text=f"📦 {item['nombre']}", 
                 font=("Segoe UI", 12, "bold"), fg="white", bg="#34495e").pack(anchor="w")
        tk.Label(info_frame, text=f"Precio unitario: ${precio_ajustado:.2f}",
                 font=("Segoe UI", 10), fg="#bdc3c7", bg="#34495e").pack(anchor="w")
        tk.Label(info_frame, text=f"Precio Costo Base: ${precio_costo_base:.2f}", 
                 font=("Segoe UI", 10, "bold"), fg="#e74c3c", bg="#34495e").pack(anchor="w") 
        tk.Label(info_frame, text=f"Cantidad: {item['cantidad']} unidad(es)", 
                 font=("Segoe UI", 10), fg="#bdc3c7", bg="#34495e").pack(anchor="w")
        
        if item.get('descuento', 0.0) > 0:
            tk.Label(info_frame, 
                     text=f"⚠️ Descuento actual: ${item['descuento']:.2f} por unidad", 
                     fg="#e74c3c", bg="#34495e", font=("Segoe UI", 10, "bold")).pack(anchor="w")
        
        # Separador
        ttk.Separator(main_frame, orient='horizontal').pack(fill="x", pady=10)
        
        # Selección de tipo
        tk.Label(main_frame, text="Seleccione tipo de descuento:", 
                 font=("Segoe UI", 10, "bold"), fg="white", bg="#2c3e50").pack(anchor="w")
        
        var_tipo = tk.StringVar(value="porcentaje")
        
        radio_frame = tk.Frame(main_frame, bg="#34495e", padx=10, pady=8)
        radio_frame.pack(pady=5, fill="x")
        
        tk.Radiobutton(radio_frame, text="📊 PORCENTAJE (%)", variable=var_tipo, 
                       value="porcentaje", bg="#34495e", fg="white", 
                       selectcolor="#2c3e50", activebackground="#34495e",
                       activeforeground="#f39c12", font=("Segoe UI", 11, "bold"),
                       command=lambda: limpiar_campos()).pack(anchor="w", pady=2)
        
        tk.Radiobutton(radio_frame, text="💰 MONTO ($)", variable=var_tipo, 
                       value="monto", bg="#34495e", fg="white",
                       selectcolor="#2c3e50", activebackground="#34495e",
                       activeforeground="#f39c12", font=("Segoe UI", 11, "bold"),
                       command=lambda: limpiar_campos()).pack(anchor="w", pady=2)
        
        # Entrada de valor
        entrada_frame = tk.Frame(main_frame, bg="#2c3e50")
        entrada_frame.pack(pady=10, fill="x")
        
        tk.Label(entrada_frame, text="Ingrese el valor:", 
                 font=("Segoe UI", 10, "bold"), fg="white", bg="#2c3e50").pack(anchor="w")
        
        ent_valor = tk.Entry(entrada_frame, font=("Segoe UI", 14, "bold"), 
                             justify="center", bg="white", fg="#2c3e50")
        ent_valor.pack(pady=5, fill="x")
        ent_valor.focus()
        
        # Labels informativos
        lbl_equivalencia = tk.Label(main_frame, text="", font=("Segoe UI", 10, "bold"), 
                                    fg="#2ecc71", bg="#2c3e50", wraplength=450)
        lbl_equivalencia.pack(pady=5)
        
        lbl_preview = tk.Label(main_frame, text="", font=("Segoe UI", 10), 
                               fg="#3498db", bg="#2c3e50", wraplength=450)
        lbl_preview.pack(pady=5)
        
        def limpiar_campos():
            ent_valor.delete(0, "end")
            lbl_equivalencia.config(text="")
            lbl_preview.config(text="")
        
        def actualizar_vista_previa(*args):
            try:
                valor_str = ent_valor.get().strip()
                if not valor_str:
                    lbl_equivalencia.config(text="")
                    lbl_preview.config(text="")
                    return
                
                valor = float(valor_str)
                tipo = var_tipo.get()

                # --- PASO CLAVE: USAR EL PRECIO AJUSTADO QUE YA CALCULAMOS ---
                # Usamos la variable 'precio_ajustado_pos' que definimos al abrir la ventana
                precio_referencia = (item['precio'] * float(self.ent_tasa_cobro.get())) / self.tasa_actual
                # -------------------------------------------------------------

                if tipo == "porcentaje":
                    if 0 <= valor <= 100:
                        monto_desc = (valor / 100) * precio_referencia
                        lbl_equivalencia.config(
                            text=f"💲 ${monto_desc:.2f} de descuento por unidad",
                            fg="#2ecc71"
                        )
                    else:
                        lbl_equivalencia.config(text="⚠️ Porcentaje inválido (0-100)", fg="#e74c3c")
                        lbl_preview.config(text="")
                        return
                else:
                    if 0 <= valor <= precio_referencia:
                        monto_desc = valor
                        porc_equiv = (valor / precio_referencia) * 100
                        lbl_equivalencia.config(
                            text=f"📊 Equivale al {porc_equiv:.1f}% de descuento",
                            fg="#2ecc71"
                        )
                    else:
                        lbl_equivalencia.config(text=f"⚠️ Máximo: ${precio_referencia:.2f}", fg="#e74c3c")
                        lbl_preview.config(text="")
                        return
                
                # CÁLCULO DE RENTABILIDAD (MARGEN 30%)
                precio_final = precio_referencia - monto_desc
                margen_minimo = item['costo'] * 1.30
                
                # Semáforo visual de protección
                color_info = "#3498db" if precio_final >= margen_minimo else "#e74c3c"
                alerta = "" if precio_final >= margen_minimo else "\n⚠️ ¡ALERTA: Margen por debajo del 30%!"

                lbl_preview.config(
                    text=f"✅ Precio final: ${precio_final:.2f} c/u | Ahorras: ${monto_desc * item['cantidad']:.2f}{alerta}",
                    fg=color_info
                )
                
            except ValueError:
                lbl_equivalencia.config(text="")
                lbl_preview.config(text="")

        
        ent_valor.bind("<KeyRelease>", actualizar_vista_previa)
        
        def aplicar():
            try:
                valor_str = ent_valor.get().strip()
                if not valor_str: return
                
                valor = float(valor_str)
                tipo = var_tipo.get()
                
                # 1. Obtener el precio ajustado actual por tasa de cobro
                try:
                    tasa_c = float(self.ent_tasa_cobro.get())
                except:
                    tasa_c = self.tasa_actual
                
                precio_ajustado = (item['precio'] * tasa_c) / self.tasa_actual
                
                # 2. Calcular cuánto sería el descuento en dólares
                if tipo == "porcentaje":
                    if not (0 <= valor <= 100):
                        messagebox.showwarning("Error", "El porcentaje debe estar entre 0 y 100", parent=win)
                        return
                    descuento_final = (valor / 100) * precio_ajustado
                else:
                    if not (0 <= valor <= precio_ajustado):
                        messagebox.showwarning("Error", f"El monto debe ser entre $0 y ${precio_ajustado:.2f}", parent=win)
                        return
                    descuento_final = valor

                # --- BLOQUEO QUIRÚRGICO: MARGEN DE PROTECCIÓN 15% ---
                precio_final_con_desc = precio_ajustado - descuento_final
                margen_minimo = item['costo'] * 1.15 # Costo + 15% de margen mínimo (puedes ajustar este porcentaje)
                
            
                
                if precio_final_con_desc < margen_minimo:
                    messagebox.showerror(
                        "Protección de Utilidad", 
                        f"¡DESCUENTO DENEGADO!\n\n"
                        f"El precio final (${precio_final_con_desc:.2f}) es menor al "
                        f"mínimo de protección (${margen_minimo:.2f}).\n"
                        f"(Costo base ${item['costo']:.2f} + 15% de margen).",
                        parent=win
                    )
                    return
                # ----------------------------------------------------
                
                # 3. Guardar y actualizar
                self.carrito[idx]['descuento'] = descuento_final
                self.actualizar_tabla_pos()
                win.destroy()
                
                messagebox.showinfo("Descuento Aplicado", 
                                  f"✅ Descuento de ${descuento_final:.2f} aplicado correctamente.")
                
            except ValueError:
                messagebox.showwarning("Error", "Ingrese un valor numérico válido", parent=win)


        
        def eliminar_descuento():
            if 'descuento' in self.carrito[idx]:
                del self.carrito[idx]['descuento']
            self.actualizar_tabla_pos()
            win.destroy()
            messagebox.showinfo("Descuento Eliminado", "✅ Descuento eliminado correctamente")
        
        # Separador antes de botones
        ttk.Separator(main_frame, orient='horizontal').pack(fill="x", pady=10)
        
        # *** BOTONES - SECCIÓN CRÍTICA ***
        btn_frame = tk.Frame(main_frame, bg="#2c3e50")
        btn_frame.pack(pady=10, fill="x")
        
        # Botón APLICAR - GRANDE Y VISIBLE
        btn_aplicar = tk.Button(btn_frame, text="APLICAR DESCUENTO", 
                                bg="#27ae60", fg="white",
                                font=("Segoe UI", 12, "bold"), 
                                padx=30, pady=12,
                                command=aplicar, cursor="hand2",
                                activebackground="#2ecc71",
                                activeforeground="white")
        btn_aplicar.pack(pady=5, fill="x")
        
        # Frame para botones secundarios
        btn_secundario_frame = tk.Frame(main_frame, bg="#2c3e50")
        btn_secundario_frame.pack(pady=5, fill="x")
        
        if item.get('descuento', 0.0) > 0:
            tk.Button(btn_secundario_frame, text="ELIMINAR DESCUENTO", 
                      bg="#e74c3c", fg="white",
                      font=("Segoe UI", 10, "bold"), 
                      padx=20, pady=8,
                      command=eliminar_descuento, cursor="hand2").pack(side="left", padx=5)
        
        tk.Button(btn_secundario_frame, text="CANCELAR", 
                  bg="#95a5a6", fg="white",
                  font=("Segoe UI", 10, "bold"), 
                  padx=20, pady=8,
                  command=win.destroy, cursor="hand2").pack(side="right", padx=5)
        
        # Botón eliminar descuento (solo si ya tiene descuento)
        if item.get('descuento', 0.0) > 0:
            tk.Button(btn_secundario_frame, text="🗑 ELIMINAR DESCUENTO", 
                      bg="#e74c3c", fg="white",
                      font=("Segoe UI", 10, "bold"), 
                      padx=20, pady=5,
                      command=eliminar_descuento, cursor="hand2").pack(side="left", padx=5)

    
    def mostrar_detalle_emergente(self, event):
        """Despliega la ventana modal extrayendo de forma segura el texto de productos del lote de tags."""
        item_id = self.tree_finanzas.identify_row(event.y)
        if not item_id: 
            return
        
        # --- PASO CLAVE: EXTRAER EXCLUSIVAMENTE LA GLOSA DE COMPRA ---
        tags_raw = self.tree_finanzas.item(item_id, "tags")
        
        if isinstance(tags_raw, (list, tuple)) and len(tags_raw) > 0:
            # Al tomar el índice 0, aislamos la lista original de piezas, ignorando el color "factura_canje"
            detalle_full = tags_raw[0]
        else:
            detalle_full = str(tags_raw) if tags_raw else "Sin detalle"
        
        # El resto de tu función de auditoría (el Canvas, las capas A, B y C del bucle)
        # se mantiene EXACTAMENTE IGUAL a la versión profesional que consolidamos.

        
        # 2. Configuración de Ventana Modal Transitoria con Centrado Forzado Real
        pop = tk.Toplevel(self.root)
        pop.title("AUDITORÍA DE VENTA")
        pop.geometry("500x550") 
        pop.configure(bg="#2c3e50")
        pop.transient(self.root)
        pop.grab_set()
        pop.resizable(False, False)

        # Ocultamos temporalmente en memoria para evitar el "salto visual" a la esquina
        pop.withdraw()

        def _forzar_centrado_emergente_seguro():
            """Mapea la resolución exacta de la pantalla y ancla el popup al centro físico."""
            pop.update_idletasks()
            ancho_pantalla = pop.winfo_screenwidth()
            alto_pantalla = pop.winfo_screenheight()
            pos_x = (ancho_pantalla // 2) - (500 // 2)
            pos_y = (alto_pantalla // 2) - (550 // 2)
            pop.geometry(f"500x550+{pos_x}+{pos_y}")
            pop.deiconify() # Revela la ventana de forma instantánea ya posicionada

        # Otorgamos 10ms asíncronos para que el sistema operativo procese las dimensiones base
        pop.after(10, _forzar_centrado_emergente_seguro)

        tk.Label(pop, text="📋 DETALLES DE TRANSACCIÓN", font=("Segoe UI", 14, "bold"), 
                 fg="#3498db", bg="#2c3e50", pady=20).pack()

        # 3. SISTEMA DE SCROLL QUIRÚRGICO (Canvas)
        scroll_container = tk.Frame(pop, bg="#2c3e50")
        scroll_container.pack(fill="both", expand=True, padx=10)

        canvas = tk.Canvas(scroll_container, bg="#2c3e50", highlightthickness=0)
        scroll = ttk.Scrollbar(scroll_container, orient="vertical", command=canvas.yview)
        
        frame_items = tk.Frame(canvas, bg="#2c3e50")
        canvas_window = canvas.create_window((0, 0), window=frame_items, anchor="nw")

        def configurar_scroll(event):
            canvas.configure(scrollregion=canvas.bbox("all"))
            canvas.itemconfig(canvas_window, width=event.width)

        canvas.bind("<Configure>", configurar_scroll)
        canvas.configure(yscrollcommand=scroll.set)

        canvas.pack(side="left", fill="both", expand=True)
        scroll.pack(side="right", fill="y")

        def _on_mousewheel(event):
            canvas.yview_scroll(int(-1*(event.delta/120)), "units")
        
        pop.bind("<MouseWheel>", _on_mousewheel)

        # 4. TRIPLE CLASIFICACIÓN ANALÍTICA DE PRIORIDADES
        lista_cruda = detalle_full.split(", ")
        
        productos_reales = []
        canjes_aplicados = []
        item_pago = None

        # Separamos el string masivo en 3 matrices contables independientes
        for item_str in lista_cruda:
            if "MÉTODO" in item_str:
                item_pago = item_str
            elif "CANJE" in item_str or "INFO-NC" in item_str:
                canjes_aplicados.append(item_str)
            else:
                productos_reales.append(item_str)

        # --- CAPA A: RENDERIZAR EL MÉTODO DE PAGO ARRIBA (Prioridad 1) ---
        if item_pago:
            texto_pago_limpio = item_pago.replace("|", "  ▸  ").replace("[", "").replace("]", "")
            
            card_pago = tk.Frame(
                frame_items, bg="#1e272e", pady=15, padx=20, 
                highlightthickness=1, highlightbackground="#2f3640"
            )
            card_pago.pack(fill="x", pady=(5, 10), padx=15)
            
            lbl_audit_title = tk.Label(
                card_pago, text="💵 AUDITORÍA DE LIQUIDACIÓN Y TRANSACCIÓN", 
                font=("Segoe UI", 9, "bold"), fg="#2ecc71", bg="#1e272e"
            )
            lbl_audit_title.pack(anchor="w")
            
            lbl_audit_val = tk.Label(
                card_pago, text=texto_pago_limpio, font=("Segoe UI", 11, "bold"), 
                fg="#bdc3c7", bg="#1e272e", wraplength=380, justify="left"
            )
            lbl_audit_val.pack(anchor="w", pady=(8, 0))

        # --- CAPA B: RENDERIZAR LAS NOTAS DE CRÉDITO Y VALES AL MEDIO (Prioridad 2) ---
        for item_canje in canjes_aplicados:
            texto_canje_limpio = item_canje.replace("|", "  ▸  ")
            # Corrección quirúrgica del bug visual de moneda invertida "$-"
            texto_canje_limpio = texto_canje_limpio.replace("Ref: $-", "Deducción Factura: -$")
            texto_canje_limpio = texto_canje_limpio.replace("[INFO-NC]", "").strip()

            card_canje = tk.Frame(
                frame_items, bg="#2f3542", pady=15, padx=20, 
                highlightthickness=1, highlightbackground="#f1c40f"
            )
            card_canje.pack(fill="x", pady=6, padx=15)
            
            lbl_canje_title = tk.Label(
                card_canje, text="🏷️ AJUSTE DE BALANCES POR CANJE / DEVOLUCIÓN", 
                font=("Segoe UI", 9, "bold"), fg="#f1c40f", bg="#2f3542"
            )
            lbl_canje_title.pack(anchor="w")
            
            lbl_canje_val = tk.Label(
                card_canje, text=texto_canje_limpio, font=("Segoe UI", 11, "bold"), 
                fg="white", bg="#2f3542", wraplength=380, justify="left"
            )
            lbl_canje_val.pack(anchor="w", pady=(6, 0))

        # --- CAPA C: RENDERIZAR LOS PRODUCTOS FÍSICOS AL FINAL (Prioridad 3) ---
        for i, item_prod in enumerate(productos_reales):
            texto_prod_limpio = item_prod.replace("|", "  ▸  ")

            card_prod = tk.Frame(
                frame_items, bg="#34495e", pady=15, padx=20, 
                highlightthickness=1, highlightbackground="#3d566e"
            )
            card_prod.pack(fill="x", pady=6, padx=15)
            
            tk.Label(
                card_prod, text=f"📦 ÍTEM #{i+1}", font=("Segoe UI", 9, "bold"), 
                fg="#f39c12", bg="#34495e"
            ).pack(anchor="w")
            
            tk.Label(
                card_prod, text=texto_prod_limpio, font=("Segoe UI", 11), 
                fg="white", bg="#34495e", wraplength=380, justify="left"
            ).pack(anchor="w", pady=(5, 0))

        # 5. Botón de cierre tipo bloque unificado al fondo
        tk.Button(
            pop, text="❌ CERRAR AUDITORÍA", command=pop.destroy, bg="#c0392b", fg="white", 
            activebackground="#e74c3c", activeforeground="white", relief="flat", 
            font=("Segoe UI", 10, "bold"), pady=12, cursor="hand2"
        ).pack(fill="x", padx=40, pady=20)





    def cobrar_deuda(self, id_credito):
        # 1. Traer datos del crédito
        credito = consulta("SELECT cliente, detalle_productos, monto_total, costo_total FROM cuentas_por_cobrar WHERE id=?", 
                        (id_credito,), db_path=DB_CREDITOS)[0]
        
        # 2. Registrar en VENTAS_MOTOS.DB (La oficial de ingresos)
        fecha_pago = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        detalle_venta = f"PAGO CRÉDITO: {credito[0]} - {credito[1]}"
        
        consulta("INSERT INTO ventas_log (fecha_hora, productos, total, costo_total) VALUES (?, ?, ?, ?)", 
                (fecha_pago, detalle_venta, credito[2], credito[3]), 
                fetch=False, db_path=DB_VENTAS)

        # 3. Eliminar de CREDITOS_MOTOS.DB
        consulta("DELETE FROM cuentas_por_cobrar WHERE id=?", (id_credito,), 
                fetch=False, db_path=DB_CREDITOS)
        
        messagebox.showinfo("Pago Exitoso", "El dinero ha sido ingresado a la contabilidad de ventas.")

    
            # -------------------------------------------------
        # PANTALLA CUENTAS POR COBRAR (SISTEMA DE CRÉDITOS)
        # -------------------------------------------------
    def crear_pantalla_creditos(self):
        frame = tk.Frame(self.contenedor, bg="#f4f6f7")
        self.frames["creditos"] = frame

        # --- BARRA SUPERIOR ---
        top = tk.Frame(frame, bg="#2c3e50", pady=15)
        top.pack(fill="x")
        
        tk.Label(top, text="💳 GESTIÓN DE CUENTAS POR COBRAR", font=("Segoe UI", 16, "bold"), fg="white", bg="#2c3e50").pack(side="left", padx=20)

        tk.Button(top, text="⬅ Volver al Menú", font=("Segoe UI", 10, "bold"), bg="#5d6d7e", fg="white",
                  relief="flat", padx=15, pady=5, cursor="hand2",
                  command=lambda: self.mostrar_pantalla("menu", direccion="left")).pack(side="right", padx=20)

        # --- BUSCADOR ---
        search_bar = tk.Frame(frame, bg="white", pady=10, highlightthickness=1, highlightbackground="#dcdde1")
        search_bar.pack(fill="x", padx=20, pady=10)

        tk.Label(search_bar, text="🔍 Buscar Cliente o Deudor:", font=("Segoe UI", 10, "bold"), bg="white", fg="#7f8c8d").pack(side="left", padx=(20, 10))
        self.ent_buscar_deudor = tk.Entry(search_bar, font=("Segoe UI", 11), width=40, relief="flat", highlightthickness=1, highlightbackground="#bdc3c7")
        self.ent_buscar_deudor.pack(side="left", padx=10)
        self.ent_buscar_deudor.bind("<KeyRelease>", lambda e: self.cargar_tabla_creditos())

        # --- TABLA DE CRÉDITOS ---
        tabla_frame = tk.Frame(frame, bg="white")
        tabla_frame.pack(fill="both", expand=True, padx=20, pady=(0, 10))

        cols = ("id", "cliente", "fecha", "monto")
        self.tree_creditos = ttk.Treeview(tabla_frame, columns=cols, show="headings", height=15)
        
        self.tree_creditos.heading("id", text="ID")
        self.tree_creditos.heading("cliente", text="Cliente / Deudor")
        self.tree_creditos.heading("fecha", text="Fecha de Deuda")
        self.tree_creditos.heading("monto", text="Monto Pendiente ($)")
        
        self.tree_creditos.column("id", width=50, anchor="center")
        self.tree_creditos.column("cliente", width=300, anchor="w")
        self.tree_creditos.column("fecha", width=150, anchor="center")
        self.tree_creditos.column("monto", width=150, anchor="center")

        self.tree_creditos.pack(side="left", fill="both", expand=True)
        
        scroll = ttk.Scrollbar(tabla_frame, orient="vertical", command=self.tree_creditos.yview)
        scroll.pack(side="right", fill="y")
        self.tree_creditos.configure(yscrollcommand=scroll.set)

        # --- PANEL DE ACCIONES OPTIMIZADO (GRID) ---
        # Usamos un frame con padding y fondo sutil
                # --- PANEL DE ACCIONES EN CUADRÍCULA (GRID) ---
        btns_container = tk.Frame(frame, bg="#f4f6f7")
        btns_container.pack(side="bottom", pady=15)


        # Configuramos 3 columnas con el mismo peso para que sean simétricas
        for i in range(3):
            btns_container.columnconfigure(i, weight=1)

        # ESTILO COMÚN PARA BOTONES TIPO "CUADRADO"
        # Ajustamos el padding (pady) para que no sean tan altos y parezcan más bloques
        estilo_sq = {
            "relief": "flat", 
            "width": 18,      
            "height": 2,      
            "font": ("Segoe UI", 9, "bold"), 
            "cursor": "hand2"
        }

        tk.Button(btns_container, text="VER DETALLE", bg="#34495e", fg="white",
                  command=self.ver_detalle_credito, **estilo_sq).grid(row=0, column=0, padx=3, pady=3)

        tk.Button(btns_container, text="ABONAR", bg="#2980b9", fg="white",
                  command=self.registrar_abono_credito, **estilo_sq).grid(row=0, column=1, padx=3, pady=3)

        tk.Button(btns_container, text="PAGO TOTAL", bg="#27ae60", fg="white",
                  command=self.procesar_pago_deuda, **estilo_sq).grid(row=0, column=2, padx=3, pady=3)

        # FILA 2: Gestión y Reporte
        tk.Button(btns_container, text="EDITAR", bg="#f39c12", fg="white",
                  command=self.modificar_monto_credito, **estilo_sq).grid(row=1, column=0, padx=3, pady=3)

        tk.Button(btns_container, text="ANULAR", bg="#c0392b", fg="white",
                  command=self.eliminar_deuda_credito, **estilo_sq).grid(row=1, column=1, padx=3, pady=3)

        # NUEVO BOTÓN: EXPORTAR PDF
        tk.Button(btns_container, text="EXPORTAR PDF", bg="#d35400", fg="white",
                  command=self.exportar_creditos_pdf, **estilo_sq).grid(row=1, column=2, padx=3, pady=3)


    def exportar_creditos_pdf(self):
        """Genera un reporte de cobranza profesional con antigüedad de deuda."""
        datos_raw = consulta("SELECT id, cliente, fecha_deuda, monto_total FROM cuentas_por_cobrar ORDER BY fecha_deuda ASC", db_path=DB_CREDITOS)
        
        if not datos_raw:
            messagebox.showinfo("Reporte vacío", "No hay cuentas por cobrar registradas.")
            return

        datos_procesados = []
        fecha_actual = datetime.now()
        suma_total_deuda = 0.0

        for d in datos_raw:
            try:
                # 1. Cálculo de antigüedad
                f_deuda_dt = datetime.strptime(d[2], "%Y-%m-%d %H:%M:%S")
                dias_mora = (fecha_actual - f_deuda_dt).days
                antiguedad_txt = f"{dias_mora} días" if dias_mora > 0 else "Hoy"
                
                # 2. Sumatoria del total
                suma_total_deuda += float(d[3])
                
                # 3. Formateo de fila
                fila = [
                    str(d[0]),
                    str(d[1]).upper(),
                    f_deuda_dt.strftime("%d/%m/%Y"),
                    f"${d[3]:.2f}",
                    antiguedad_txt
                ]
                datos_procesados.append(fila)
            except Exception as e:
                print(f"Error en fila {d}: {e}")

        datos_procesados.append(["", "", "", "", ""]) 
        datos_procesados.append(["", "", "TOTAL DEUDA:", f"${suma_total_deuda:.2f}", ""])

        encabezados = ["ID", "CLIENTE / DEUDOR", "FECHA INICIO", "SALDO PEND.", "TIEMPO"]
        
        self.exportar_a_pdf(
            titulo_reporte="REPORTE CRÍTICO DE CUENTAS POR COBRAR (COBRANZA)",
            datos=datos_procesados,
            encabezados=encabezados
        )

    def registrar_abono_credito(self):
        sel = self.tree_creditos.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione una cuenta para abonar.")
            return

        # 1. Obtener datos actuales
        item_data = self.tree_creditos.item(sel)
        id_credito = item_data["values"][0]
        cliente = item_data["values"][1]
        monto_actual = float(item_data["values"][3].replace("$", "").replace(",", ""))

        # 2. Ventana de Abono Estilizada Premium
        win = tk.Toplevel(self.root)
        win.title("Registrar Abono")
        win.geometry("400x420") # Más altura para el selector
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()

        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (400 // 2)
        y = (win.winfo_screenheight() // 2) - (420 // 2)
        win.geometry(f"+{x}+{y}")

        tk.Label(win, text="💰 REGISTRAR ABONO", font=("Segoe UI", 14, "bold"), fg="#2ecc71", bg="#2c3e50", pady=15).pack()
        tk.Label(win, text=f"Deuda de: {cliente}", font=("Segoe UI", 10), fg="white", bg="#2c3e50").pack()
        tk.Label(win, text=f"Saldo Pendiente: ${monto_actual:.2f}", font=("Segoe UI", 10, "bold"), fg="#bdc3c7", bg="#2c3e50").pack(pady=5)

        # --- NUEVO: SELECTOR DE DIVISA DE COBRANZA ---
        tk.Label(win, text="Moneda de Recepción:", font=("Segoe UI", 9, "bold"), fg="#f39c12", bg="#2c3e50").pack(pady=(5, 0))
        var_moneda_abono = tk.StringVar(value="DÓLARES ($)")
        combo_moneda_abono = ttk.Combobox(
            win, textvariable=var_moneda_abono, values=["DÓLARES ($)", "BOLÍVARES (Bs.)"],
            state="readonly", justify="center", font=("Segoe UI", 10, "bold"), width=15
        )
        combo_moneda_abono.pack(pady=5)

        ent_abono = tk.Entry(win, font=("Segoe UI", 18, "bold"), justify="center")
        ent_abono.pack(pady=15, padx=50, fill="x")
        ent_abono.focus()

        def _ejecutar_abono_interno(event=None):
            try:
                valor_entrada = ent_abono.get().strip()
                if not valor_entrada: return
                
                monto_abono = float(valor_entrada)
                
                res = consulta("SELECT monto_total, costo_pendiente FROM cuentas_por_cobrar WHERE id=?", 
                               (id_credito,), db_path=DB_CREDITOS)
                if not res: return
                saldo_actual_db, costo_pend_db = res[0]

                if monto_abono <= 0 or monto_abono > (saldo_actual_db + 0.01):
                    messagebox.showerror("Error", f"Monto inválido. Máximo permitido: ${saldo_actual_db:.2f}", parent=win)
                    return

                # Algoritmo de prorrateo contable para la ganancia neta
                if monto_abono >= (saldo_actual_db - 0.01):
                    costo_para_esta_venta = costo_pend_db
                    nuevo_saldo = 0
                    nuevo_costo_pend = 0
                else:
                    costo_para_esta_venta = round((monto_abono * costo_pend_db) / saldo_actual_db, 2)
                    nuevo_saldo = round(saldo_actual_db - monto_abono, 2)
                    nuevo_costo_pend = round(costo_pend_db - costo_para_esta_venta, 2)

                # Actualizar la DB de créditos
                consulta("UPDATE cuentas_por_cobrar SET monto_total = ?, costo_pendiente = ? WHERE id = ?", 
                         (nuevo_saldo, nuevo_costo_pend, id_credito), fetch=False, db_path=DB_CREDITOS)

                # --- SELLO MULTIMONEDA PARA EL ABONO ---
                moneda_sel = var_moneda_abono.get()
                sello_pago = f"[MÉTODO: COBRO ABONO {moneda_sel} | TASA REF: {self.tasa_actual:.2f}]"
                detalle_finanzas = f"[ABONO] {cliente} (Saldo restante: ${nuevo_saldo:.2f}), {sello_pago}"
                
                # Inyectamos en la contabilidad oficial de finanzas
                fecha_h = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                consulta("INSERT INTO ventas_log (fecha_hora, productos, total, costo_total) VALUES (?, ?, ?, ?)", 
                         (fecha_h, detalle_finanzas, monto_abono, costo_para_esta_venta), fetch=False, db_path=DB_VENTAS)

                if nuevo_saldo <= 0:
                    consulta("DELETE FROM cuentas_por_cobrar WHERE id = ?", (id_credito,), fetch=False, db_path=DB_CREDITOS)
                    messagebox.showinfo("Liquidado", f"¡Cuenta saldada! El cliente {cliente} pagó la totalidad.")
                else:
                    messagebox.showinfo("Éxito", f"Abono registrado: ${monto_abono:.2f}\nSaldo restante: ${nuevo_saldo:.2f}")

                win.destroy()
                self.cargar_tabla_creditos()
                if "finanzas" in self.frames: self.actualizar_tabla_finanzas()
                self.actualizar_estadisticas()
                
            except ValueError:
                messagebox.showerror("Error", "Ingrese un número válido", parent=win)

        tk.Button(
            win, text="✅ CONFIRMAR ABONO", font=("Segoe UI", 11, "bold"), 
            bg="#27ae60", fg="white", pady=12, cursor="hand2", relief="flat", command=_ejecutar_abono_interno
        ).pack(fill="x", padx=50, pady=10)

        ent_abono.bind("<Return>", _ejecutar_abono_interno)


        
        self.actualizar_tabla_finanzas() # Refresca ingresos
        self.cargar_tabla_creditos()     # Refresca saldo deudor
        self.actualizar_estadisticas()   # Refresca inversión en el menú






    def cargar_tabla_creditos(self):
        """Carga y filtra las deudas pendientes por nombre de cliente."""
        # Limpieza visual
        for row in self.tree_creditos.get_children():
            self.tree_creditos.delete(row)

        hijos = self.tree_creditos.get_children()
        if hijos:
            self.tree_creditos.delete(*hijos)
        
        # Obtener el texto del buscador
        filtro = self.ent_buscar_deudor.get().strip()
        
        if filtro:
            # Búsqueda por nombre (LIKE)
            query = "SELECT id, cliente, fecha_deuda, monto_total, detalle_productos FROM cuentas_por_cobrar WHERE cliente LIKE ? ORDER BY fecha_deuda DESC"
            deudas = consulta(query, (f"%{filtro}%",), db_path=DB_CREDITOS)
        else:
            # Carga completa
            query = "SELECT id, cliente, fecha_deuda, monto_total, detalle_productos FROM cuentas_por_cobrar ORDER BY fecha_deuda DESC"
            deudas = consulta(query, db_path=DB_CREDITOS)
        
        for d in deudas:
            # Insertamos en la tabla: d[0]=ID, d[1]=Cliente, d[2]=Fecha, d[3]=Monto
            # Guardamos el detalle de productos (d[4]) en los tags para la ventana de detalle
        
            self.tree_creditos.insert("", "end", values=(d[0], d[1], d[2], f"{d[3]:.2f}"), tags=(d[4],))
        
    def eliminar_deuda_credito(self):
        """Elimina la deuda y REESTABLECE automáticamente el stock de los productos."""
        sel = self.tree_creditos.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione la cuenta que desea eliminar.")
            return

        # 1. Recuperar datos básicos
        item_data = self.tree_creditos.item(sel)
        id_credito = item_data["values"][0]
        cliente = item_data["values"][1]
        
        # El detalle de productos lo guardamos en los 'tags' al cargar la tabla
        detalle_productos = item_data["tags"][0] if item_data["tags"] else ""

        # 2. Confirmación de seguridad
        if not messagebox.askyesno("Confirmar Anulación", 
                                f"¿Anular deuda de '{cliente}'?\n\n"
                                "✅ Los productos serán devueltos al inventario automáticamente."):
            return

        try:
            # 3. LOGÍSTICA DE REPOSICIÓN DE STOCK
            # El detalle tiene el formato: "[COD] Nombre xCant, [COD2] Nombre xCant"
            items = detalle_productos.split(", ")
            
            for item_str in items:
                if "[" in item_str and "x" in item_str:
                    # Extraemos el código entre corchetes
                    codigo = item_str[item_str.find("[")+1 : item_str.find("]")]
                    
                    # Extraemos la cantidad (lo que está entre 'x' y el siguiente espacio o paréntesis)
                    # Ejemplo: "... x2 (Ref..." -> '2'
                    parte_cant = item_str.split(" x")[1].split(" ")[0]
                    cantidad_a_devolver = int(parte_cant)

                    # Sumamos de nuevo al inventario
                    consulta("UPDATE repuestos SET cantidad = cantidad + ? WHERE codigo = ?", 
                             (cantidad_a_devolver, codigo), fetch=False)

            # 4. BORRADO DEL REGISTRO
            consulta("DELETE FROM cuentas_por_cobrar WHERE id = ?", (id_credito,), 
                     fetch=False, db_path=DB_CREDITOS)
            
            messagebox.showinfo("Éxito", f"Deuda anulada. El inventario de '{cliente}' ha sido restaurado.")
            
            # 5. REFRESCAR PANTALLAS
            self.cargar_tabla_creditos()
            self.actualizar_estadisticas()
            self.cargar_tabla_gestion() # Por si tienes abierta la gestión de stock
            
        except Exception as e:
            messagebox.showerror("Error", f"No se pudo procesar la anulación: {e}")


    def ver_detalle_credito(self):
        sel = self.tree_creditos.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione una cuenta para ver el detalle.")
            return

        # 1. Recuperamos los datos
        item = self.tree_creditos.item(sel)
        cliente = item["values"][1]
        monto = item["values"][3]
        # El detalle de productos está en los tags (índice 0 del tag)
        detalle_full = self.tree_creditos.item(sel, "tags")[0]

        # 2. Configuración de la Ventana Profesional
        pop = tk.Toplevel(self.root)
        pop.title(f"Detalle de Deuda - {cliente}")
        pop.geometry("500x500")
        pop.configure(bg="#2c3e50")
        pop.transient(self.root)
        pop.grab_set()

        # Centrado
        pop.update_idletasks()
        x = (pop.winfo_screenwidth() // 2) - (500 // 2)
        y = (pop.winfo_screenheight() // 2) - (500 // 2)
        pop.geometry(f"+{x}+{y}")

        # Cabecera
        header = tk.Frame(pop, bg="#34495e", pady=15)
        header.pack(fill="x")
        tk.Label(header, text="📋 AUDITORÍA DE CRÉDITO", font=("Segoe UI", 12, "bold"), fg="#3498db", bg="#34495e").pack()
        tk.Label(header, text=f"Cliente: {cliente.upper()}", font=("Segoe UI", 10), fg="white", bg="#34495e").pack()

        # Cuerpo con Scroll (por si son muchos productos)
        container = tk.Frame(pop, bg="#2c3e50", padx=20, pady=20)
        container.pack(fill="both", expand=True)

        canvas = tk.Canvas(container, bg="#2c3e50", highlightthickness=0)
        scroll = ttk.Scrollbar(container, orient="vertical", command=canvas.yview)
        frame_items = tk.Frame(canvas, bg="#2c3e50")

        canvas.create_window((0, 0), window=frame_items, anchor="nw", width=440)
        canvas.configure(yscrollcommand=scroll.set)

        # Renderizado de "Tarjetas" de productos
        productos = detalle_full.split(", ")
        for i, p_str in enumerate(productos):
            card = tk.Frame(frame_items, bg="#34495e", pady=10, padx=15)
            card.pack(fill="x", pady=5)
            
            # Limpiamos el texto para que se vea elegante
            texto_limpio = p_str.replace("[", "").replace("]", " |")
            
            tk.Label(card, text=f"Producto #{i+1}", font=("Segoe UI", 8, "bold"), fg="#f39c12", bg="#34495e").pack(anchor="w")
            tk.Label(card, text=texto_limpio, font=("Segoe UI", 10), fg="white", bg="#34495e", wraplength=400, justify="left").pack(anchor="w")

        canvas.pack(side="left", fill="both", expand=True)
        scroll.pack(side="right", fill="y")
        
        # Ajuste de scrollregion
        frame_items.update_idletasks()
        canvas.config(scrollregion=canvas.bbox("all"))

        # Pie de ventana con el Total
        footer = tk.Frame(pop, bg="#2c3e50", pady=15)
        footer.pack(fill="x")
        tk.Label(footer, text=f"MONTO TOTAL PENDIENTE: {monto}", font=("Segoe UI", 12, "bold"), fg="#27ae60", bg="#2c3e50").pack()
        
        tk.Button(footer, text="ENTENDIDO", command=pop.destroy, bg="#3498db", fg="white", 
                  relief="flat", font=("Segoe UI", 9, "bold"), padx=30, pady=8, cursor="hand2").pack(pady=10)

    def procesar_pago_deuda(self):
        """Liquida la cuenta por cobrar total permitiendo elegir la moneda de pago."""
        sel = self.tree_creditos.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione una cuenta para cobrar.")
            return

        item = self.tree_creditos.item(sel[0])
        id_credito = item["values"][0]
        cliente = item["values"][1]
        
        res = consulta("SELECT detalle_productos, monto_total, costo_pendiente FROM cuentas_por_cobrar WHERE id=?", 
                       (id_credito,), db_path=DB_CREDITOS)
        if not res: return
        detalle_prod, saldo_a_pagar, costo_restante = res[0]

        # 1. Ventana Modal de Liquidación Completa
        win = tk.Toplevel(self.root)
        win.title("Liquidación Total de Deuda")
        win.geometry("420x360")
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()
        win.resizable(False, False)

        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (420 // 2)
        y = (win.winfo_screenheight() // 2) - (360 // 2)
        win.geometry(f"+{x}+{y}")

        tk.Label(win, text="🏁 PAGO TOTAL DE CUENTA", font=("Segoe UI", 13, "bold"), fg="#27ae60", bg="#2c3e50", pady=15).pack()

        # Resumen del cobro final
        resumen = tk.Frame(win, bg="#34495e", padx=15, pady=10, highlightthickness=1, highlightbackground="#455a64")
        resumen.pack(fill="x", padx=40, pady=5)
        tk.Label(resumen, text=f"Cliente: {cliente.upper()}", font=("Segoe UI", 10, "bold"), fg="white", bg="#34495e").pack(anchor="w")
        tk.Label(resumen, text=f"Total a Liquidar: ${saldo_a_pagar:.2f}", font=("Segoe UI", 12, "bold"), fg="#2ecc71", bg="#34495e").pack(anchor="w", pady=(5,0))

        # Selector de moneda de cierre
        tk.Label(win, text="Moneda de Pago Final:", font=("Segoe UI", 9, "bold"), fg="#bdc3c7", bg="#2c3e50").pack(pady=(10, 0))
        var_moneda_cierre = tk.StringVar(value="DÓLARES ($)")
        combo_cierre = ttk.Combobox(
            win, textvariable=var_moneda_cierre, values=["DÓLARES ($)", "BOLÍVARES (Bs.)"],
            state="readonly", justify="center", font=("Segoe UI", 10, "bold"), width=15
        )
        combo_cierre.pack(pady=5)

        def _confirmar_liquidacion_fisica():
            try:
                moneda_sel = var_moneda_cierre.get()
                fecha_pago = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                
                # Sello contable para inyectar en las finanzas
                sello_pago = f"[MÉTODO: COBRO LIQUIDACIÓN {moneda_sel} | TASA REF: {self.tasa_actual:.2f}]"
                detalle_venta = f"[PAGO FINAL CRÉDITO - {cliente}] {detalle_prod}, {sello_pago}"
                
                # Inyección limpia en finanzas (ventas_log)
                consulta("INSERT INTO ventas_log (fecha_hora, productos, total, costo_total) VALUES (?, ?, ?, ?)", 
                         (fecha_pago, detalle_venta, saldo_a_pagar, costo_restante), fetch=False, db_path=DB_VENTAS)

                # Eliminar la deuda saldada del panel de cobranzas
                consulta("DELETE FROM cuentas_por_cobrar WHERE id=?", (id_credito,), fetch=False, db_path=DB_CREDITOS)

                # Mensaje personalizado de éxito
                if "BOLÍVARES" in moneda_sel:
                    total_bs = saldo_a_pagar * self.tasa_actual
                    messagebox.showinfo("Éxito", f"Cuenta liquidada en bolívares.\nIngreso: Bs. {total_bs:,.2f}\nCosto aplicado: ${costo_restante:.2f}")
                else:
                    messagebox.showinfo("Éxito", f"Cuenta liquidada en dólares.\nIngreso: ${saldo_a_pagar:.2f}\nCosto aplicado: ${costo_restante:.2f}")
                
                win.destroy()
                self.cargar_tabla_creditos()
                self.actualizar_estadisticas()
                if "finanzas" in self.frames: self.actualizar_tabla_finanzas()
                
            except Exception as e:
                messagebox.showerror("Error", f"No se pudo procesar el cobro: {e}", parent=win)

        # Botonera unificada en grilla simétrica
        btns_frame = tk.Frame(win, bg="#2c3e50")
        btns_frame.pack(side="bottom", fill="x", padx=40, pady=20)
        btns_frame.columnconfigure(0, weight=1)
        btns_frame.columnconfigure(1, weight=1)

        estilo_btn = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}

        tk.Button(btns_frame, text="✅ PROCESAR", bg="#27ae60", fg="white", activebackground="#2ecc71", command=_confirmar_liquidacion_fisica, **estilo_btn).grid(row=0, column=0, padx=(0, 5), sticky="nsew")
        tk.Button(btns_frame, text="❌ CANCELAR", bg="#c0392b", fg="white", activebackground="#e74c3c", command=win.destroy, **estilo_btn).grid(row=0, column=1, padx=(5, 0), sticky="nsew")



    def modificar_monto_credito(self):
        sel = self.tree_creditos.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione una cuenta para modificar.")
            return

        # 1. Obtener datos actuales del Treeview
        valores = self.tree_creditos.item(sel, "values")
        id_credito = valores[0]
        nombre_actual = valores[1]
        monto_actual = valores[3].replace("$", "").replace(",", "")

        # 2. Ventana emergente de edición profesional
        win = tk.Toplevel(self.root)
        win.title("Editar Registro de Deuda")
        win.geometry("450x400")
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()

        # Centrar ventana
        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (450 // 2)
        y = (win.winfo_screenheight() // 2) - (400 // 2)
        win.geometry(f"+{x}+{y}")

        tk.Label(win, text="✏️ EDITAR DEUDA", font=("Segoe UI", 14, "bold"), 
                 fg="#3498db", bg="#2c3e50", pady=20).pack()
        
        # --- CAMPO: NOMBRE DEL CLIENTE ---
        tk.Label(win, text="Nombre del Cliente / Deudor:", font=("Segoe UI", 10), 
                 fg="#bdc3c7", bg="#2c3e50").pack(anchor="w", padx=50)
        ent_nuevo_nombre = tk.Entry(win, font=("Segoe UI", 12), justify="center")
        ent_nuevo_nombre.pack(pady=(5, 15), padx=50, fill="x")
        ent_nuevo_nombre.insert(0, nombre_actual)

        # --- CAMPO: MONTO ---
        tk.Label(win, text="Monto de la Deuda ($):", font=("Segoe UI", 10), 
                 fg="#bdc3c7", bg="#2c3e50").pack(anchor="w", padx=50)
        ent_nuevo_monto = tk.Entry(win, font=("Segoe UI", 16, "bold"), justify="center", fg="#e67e22")
        ent_nuevo_monto.pack(pady=5, padx=50, fill="x")
        ent_nuevo_monto.insert(0, monto_actual)

        def guardar_ajuste():
            nuevo_n = ent_nuevo_nombre.get().strip()
            try:
                nuevo_m = float(ent_nuevo_monto.get())
                
                if not nuevo_n:
                    messagebox.showwarning("Atención", "El nombre no puede estar vacío.", parent=win)
                    return
                
                if nuevo_m < 0:
                    messagebox.showerror("Error", "El monto no puede ser negativo.", parent=win)
                    return

                # ACTUALIZACIÓN QUIRÚRGICA: Actualizamos Cliente y Monto
                consulta("""UPDATE cuentas_por_cobrar 
                           SET cliente = ?, monto_total = ? 
                           WHERE id = ?""", 
                         (nuevo_n.upper(), nuevo_m, id_credito), fetch=False, db_path=DB_CREDITOS)
                
                messagebox.showinfo("Éxito", f"Registro actualizado:\nCliente: {nuevo_n.upper()}\nMonto: ${nuevo_m:.2f}")
                win.destroy()
                self.cargar_tabla_creditos() # Refrescar la tabla principal
                
            except ValueError:
                messagebox.showerror("Error", "Ingrese un monto válido (números).", parent=win)

        # Botón Guardar
        tk.Button(win, text="💾 GUARDAR CAMBIOS", font=("Segoe UI", 11, "bold"), 
                  bg="#27ae60", fg="white", pady=12, cursor="hand2", relief="flat",
                  command=guardar_ajuste).pack(fill="x", padx=50, pady=25)
        
        ent_nuevo_monto.bind("<Return>", lambda e: guardar_ajuste())




    def abrir_gestion_combos(self):
        win = tk.Toplevel(self.root)
        win.title("Panel de Control de Combos y Promociones")
        win.geometry("1150x720")
        win.configure(bg="#2c3e50")
        win.grab_set()
        
        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (1150 // 2)
        y = (win.winfo_screenheight() // 2) - (720 // 2)
        win.geometry(f"+{x}+{y}")

        self.items_combo_temp = [] 
        self.combo_editando_id = None 

        # --- PANEL IZQUIERDO: LISTA DE COMBOS (Ancho fijo 300) ---
        izq = tk.Frame(win, bg="#34495e", padx=15, pady=15, width=300)
        izq.pack(side="left", fill="y")
        izq.pack_propagate(False) # Obliga a respetar el ancho de 300

        tk.Label(izq, text="📋 COMBOS ACTIVOS", font=("Segoe UI", 11, "bold"), fg="#f1c40f", bg="#34495e").pack(pady=10)
        
        self.tree_combos_existentes = ttk.Treeview(izq, columns=("id", "nom", "precio"), show="headings")
        self.tree_combos_existentes.heading("id", text="ID")
        self.tree_combos_existentes.heading("nom", text="Combo")
        self.tree_combos_existentes.heading("precio", text="Precio")
        self.tree_combos_existentes.column("id", width=30, anchor="center")
        self.tree_combos_existentes.column("nom", width=150)
        self.tree_combos_existentes.column("precio", width=60, anchor="center")
        self.tree_combos_existentes.pack(fill="both", expand=True)

        # --- PANEL CENTRAL: BUSCADOR DE REPUESTOS (Dinámico) ---
        centro = tk.Frame(win, bg="#2c3e50", padx=15, pady=15)
        centro.pack(side="left", fill="both", expand=True)

        tk.Label(centro, text="🔍 1. BUSCAR PIEZAS", font=("Segoe UI", 10, "bold"), fg="white", bg="#2c3e50").pack()
        ent_bus = tk.Entry(centro, font=("Segoe UI", 11))
        ent_bus.pack(fill="x", pady=10)

        # TABLA CON COLUMNA STOCK (4 columnas exactas)
        self.tree_inv_combos = ttk.Treeview(centro, columns=("cod", "nom", "costo", "stock"), show="headings", height=10)
        self.tree_inv_combos.heading("cod", text="Cod")
        self.tree_inv_combos.heading("nom", text="Producto")
        self.tree_inv_combos.heading("costo", text="Costo")
        self.tree_inv_combos.heading("stock", text="Stock")
        
        self.tree_inv_combos.column("cod", width=60, anchor="center")
        self.tree_inv_combos.column("nom", width=150, anchor="w")
        self.tree_inv_combos.column("costo", width=60, anchor="center")
        self.tree_inv_combos.column("stock", width=60, anchor="center")
        self.tree_inv_combos.pack(fill="both", expand=True)

        # --- PANEL DERECHO: CONSTRUCTOR (Ancho fijo 380) ---
        der = tk.Frame(win, bg="#ecf0f1", padx=20, pady=15, width=380)
        der.pack(side="right", fill="y")
        der.pack_propagate(False)

        self.lbl_modo = tk.Label(der, text="🆕 NUEVO COMBO", font=("Segoe UI", 12, "bold"), bg="#ecf0f1", fg="#27ae60")
        self.lbl_modo.pack(pady=10)
        
        tk.Label(der, text="Nombre del Combo:", bg="#ecf0f1", font=("Segoe UI", 9, "bold")).pack(anchor="w")
        ent_nom = tk.Entry(der, font=("Segoe UI", 11))
        ent_nom.pack(fill="x", pady=5)

        self.tree_sel = ttk.Treeview(der, columns=("nom", "cant", "subt"), show="headings", height=8)
        self.tree_sel.heading("nom", text="Pieza"); self.tree_sel.heading("cant", text="Cant"); self.tree_sel.heading("subt", text="Subt")
        self.tree_sel.column("nom", width=140); self.tree_sel.column("cant", width=40, anchor="center"); self.tree_sel.column("subt", width=60, anchor="center")
        self.tree_sel.pack(fill="x", pady=10)

        # Tarjeta de Totales Profesional
        resumen_card = tk.Frame(der, bg="white", highlightthickness=1, highlightbackground="#dcdde1", padx=15, pady=10)
        resumen_card.pack(fill="x", pady=5)
        self.lbl_costo_total = tk.Label(resumen_card, text="COSTO TOTAL BASE: $0.00", font=("Segoe UI", 9, "bold"), bg="white", fg="#c0392b")
        self.lbl_costo_total.pack(anchor="w")
        self.lbl_ganancia = tk.Label(resumen_card, text="Esperando precio...", font=("Segoe UI", 9, "bold"), bg="white", fg="#7f8c8d")
        self.lbl_ganancia.pack(anchor="w")

        tk.Label(der, text="PRECIO VENTA FINAL ($):", bg="#ecf0f1", font=("Segoe UI", 10, "bold")).pack(pady=(10,0))
        ent_pv = tk.Entry(der, font=("Segoe UI", 14, "bold"), justify="center", fg="#27ae60")
        ent_pv.pack(fill="x", pady=5)

        # --- LÓGICA DE FUNCIONES INTERNAS ---
        def refresh_view():
            self.tree_sel.delete(*self.tree_sel.get_children())
            total = 0
            for i in self.items_combo_temp:
                sub = i['cant'] * i['costo']
                total += sub
                self.tree_sel.insert("", "end", values=(i['nom'], i['cant'], f"${sub:.2f}"))
            self.lbl_costo_total.config(text=f"COSTO TOTAL BASE: ${total:.2f}")
            actualizar_rentabilidad()

        def actualizar_rentabilidad(e=None):
            try:
                precio_v = float(ent_pv.get())
                costo_t = float(self.lbl_costo_total.cget("text").split("$")[-1])
                if precio_v < costo_t:
                    self.lbl_ganancia.config(text=f"⚠️ PÉRDIDA: -${(costo_t-precio_v):.2f}", fg="#e74c3c")
                else:
                    ganancia = precio_v - costo_t
                    porc = (ganancia / precio_v * 100) if precio_v > 0 else 0
                    self.lbl_ganancia.config(text=f"✅ GANANCIA: ${ganancia:.2f} ({porc:.1f}%)", fg="#27ae60")
            except: self.lbl_ganancia.config(text="Esperando precio...", fg="#7f8c8d")

        def buscar_rep(e=None):
            self.tree_inv_combos.delete(*self.tree_inv_combos.get_children())
            # SQL pide 4 valores para coincidir con la tabla
            query = "SELECT codigo, nombre, precio_costo, cantidad FROM repuestos WHERE nombre LIKE ? OR codigo LIKE ? LIMIT 15"
            res = consulta(query, (f"%{ent_bus.get()}%", f"%{ent_bus.get()}%"))
            for r in res: self.tree_inv_combos.insert("", "end", values=r)

        def agregar_al_constructor(e=None):
            sel = self.tree_inv_combos.selection()
            if not sel: return
            cod, nom, costo, stock = self.tree_inv_combos.item(sel)["values"]
            if int(stock) <= 0:
                messagebox.showwarning("Sin Stock", "No puedes añadir piezas con stock 0.", parent=win)
                return
            for i in self.items_combo_temp:
                if i['cod'] == cod: 
                    i['cant'] += 1
                    refresh_view(); return
            self.items_combo_temp.append({'cod': cod, 'nom': nom, 'cant': 1, 'costo': float(costo)})
            refresh_view()

        def quitar_pieza(e=None):
            sel = self.tree_sel.selection()
            if not sel: return
            idx = self.tree_sel.index(sel)
            del self.items_combo_temp[idx]
            refresh_view()

        def cargar_combos_db():
            self.tree_combos_existentes.delete(*self.tree_combos_existentes.get_children())
            try:
                conn = sqlite3.connect(DB_COMBOS)
                res = conn.execute("SELECT id, nombre, precio_venta FROM combos").fetchall()
                for r in res: self.tree_combos_existentes.insert("", "end", values=r)
                conn.close()
            except: pass

        # --- BOTONES ---
        tk.Button(centro, relief="flat", text="➕ AÑADIR PIEZA", bg="#2980b9", fg="white", font=("Segoe UI", 9, "bold"), pady=8, command=agregar_al_constructor).pack(fill="x", pady=5)
        tk.Button(der, relief="flat", text="💾 GUARDAR / ACTUALIZAR", bg="#27ae60", fg="white", font=("Segoe UI", 10, "bold"), pady=12, command=lambda: self.ejecutar_guardado_combo(ent_nom, ent_pv, win)).pack(fill="x", pady=10)
        tk.Button(izq, relief="flat", text="🗑️ ELIMINAR", bg="#c0392b", fg="white", font=("Segoe UI", 9), pady=5, command=lambda: self.ejecutar_eliminar_combo(win)).pack(fill="x", pady=10)
        tk.Button(der, relief="flat", text="✨ LIMPIAR / NUEVO", bg="#7f8c8d", fg="white", command=lambda: [win.destroy(), self.abrir_gestion_combos()]).pack(fill="x")

        # BINDEOS QUIRÚRGICOS
        ent_bus.bind("<Return>", buscar_rep)
        self.tree_inv_combos.bind("<Double-1>", agregar_al_constructor)
        self.tree_sel.bind("<Double-1>", quitar_pieza)
        ent_pv.bind("<KeyRelease>", actualizar_rentabilidad)
        
        cargar_combos_db()
        buscar_rep()

                # --- BINDEOS QUIRÚRGICOS ---
        # 1. Buscar al presionar ENTER
        ent_bus.bind("<Return>", lambda e: buscar_rep())
        
        # 2. Añadir al combo con DOBLE CLIC en la tabla del centro
        self.tree_inv_combos.bind("<Double-1>", lambda e: agregar_al_constructor())
        
        # 3. Quitar del combo con DOBLE CLIC en la tabla de la derecha
        self.tree_sel.bind("<Double-1>", quitar_pieza)
        
        # 4. Calcular ganancia mientras escribes el precio
        ent_pv.bind("<KeyRelease>", actualizar_rentabilidad)

    def ejecutar_guardado_combo(self, ent_nom, ent_pv, ventana_win):
        """Procesa el guardado físico en la base de datos de combos."""
        nombre = ent_nom.get().upper().strip()
        precio_v_str = ent_pv.get().strip()
        
        # 1. Validaciones de Seguridad
        if not nombre or not precio_v_str or not self.items_combo_temp:
            messagebox.showwarning("Datos Incompletos", "Por favor, asigne un nombre, precio y al menos un producto al combo.", parent=ventana_win)
            return

        try:
            precio_v = float(precio_v_str)
            # Extraer el costo total del label (quitando el texto y el $)
            costo_t = float(self.lbl_costo_total.cget("text").split("$")[-1])
        except ValueError:
            messagebox.showerror("Error de Formato", "El precio de venta debe ser un número.", parent=ventana_win)
            return

        # 2. Operación de Base de Datos
        try:
            conn = sqlite3.connect(DB_COMBOS)
            cur = conn.cursor()
            
            if self.combo_editando_id:
                # MODO EDICIÓN: Actualizar cabecera y limpiar detalles viejos
                cur.execute("UPDATE combos SET nombre=?, precio_venta=?, costo_total=? WHERE id=?", 
                           (nombre, precio_v, costo_t, self.combo_editando_id))
                cur.execute("DELETE FROM combo_detalle WHERE combo_id=?", (self.combo_editando_id,))
                combo_id = self.combo_editando_id
            else:
                # MODO NUEVO: Insertar cabecera
                cur.execute("INSERT INTO combos (nombre, precio_venta, costo_total) VALUES (?, ?, ?)", 
                           (nombre, precio_v, costo_t))
                combo_id = cur.lastrowid

            # 3. Insertar los componentes actuales de la receta
            for item in self.items_combo_temp:
                cur.execute("INSERT INTO combo_detalle (combo_id, producto_codigo, cantidad) VALUES (?, ?, ?)", 
                           (combo_id, item['cod'], item['cant']))

            conn.commit()
            conn.close()
            
            messagebox.showinfo("Éxito", f"Combo '{nombre}' guardado correctamente.", parent=ventana_win)
            
            # 4. Refresco Total y Cierre
            ventana_win.destroy()
            self.abrir_gestion_combos() # Reabre para ver los cambios
            
        except sqlite3.IntegrityError:
            messagebox.showerror("Error", "Ya existe un combo con ese nombre.", parent=ventana_win)
        except Exception as e:
            messagebox.showerror("Error Crítico", f"No se pudo guardar: {e}", parent=ventana_win)

    def ejecutar_eliminar_combo(self, ventana_win):
        """Elimina el combo seleccionado de la base de datos."""
        sel = self.tree_combos_existentes.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione un combo de la lista izquierda para eliminar.", parent=ventana_win)
            return

        id_c = self.tree_combos_existentes.item(sel)["values"][0]
        nombre_c = self.tree_combos_existentes.item(sel)["values"][1]

        if messagebox.askyesno("Confirmar Eliminación", f"¿Está seguro de eliminar permanentemente el combo: {nombre_c}?", parent=ventana_win):
            try:
                conn = sqlite3.connect(DB_COMBOS)
                cur = conn.cursor()
                # El ON DELETE CASCADE en la DB debería borrar los detalles, 
                # pero lo hacemos manual por seguridad:
                cur.execute("DELETE FROM combo_detalle WHERE combo_id=?", (id_c,))
                cur.execute("DELETE FROM combos WHERE id=?", (id_c,))
                conn.commit()
                conn.close()
                
                messagebox.showinfo("Eliminado", "Combo eliminado correctamente.", parent=ventana_win)
                ventana_win.destroy()
                self.abrir_gestion_combos() # Refrescar
            except Exception as e:
                messagebox.showerror("Error", f"No se pudo eliminar: {e}", parent=ventana_win)



    def mostrar_combos_en_pos(self):
        """Muestra los combos disponibles con el formato premium bicolor y alto contraste."""
        # 1. Limpieza total y seteo del fondo de lienzo de contraste
        for widget in self.grid_productos.winfo_children():
            widget.destroy()
        
        self.grid_productos.config(bg="#f5f7f8")

        # 2. Botón de retorno (Diseño tipo Tarjeta Corporativa de Control)
        btn_volver_frame = tk.Frame(
            self.grid_productos, bg="white", highlightthickness=1, highlightbackground="#dcdde1",
            width=175, height=185
        )
        btn_volver_frame.grid(row=0, column=0, padx=8, pady=8, sticky="nsew")
        btn_volver_frame.pack_propagate(False)

        head_volver = tk.Frame(btn_volver_frame, bg="#7f8c8d", height=85)
        head_volver.pack(fill="x", side="top")
        head_volver.pack_propagate(False)

        lbl_ico_back = tk.Label(head_volver, text="⬅️", font=("Segoe UI", 24), bg="#7f8c8d", fg="white")
        lbl_ico_back.pack(expand=True)

        body_volver = tk.Frame(btn_volver_frame, bg="white")
        body_volver.pack(fill="both", expand=True)

        lbl_txt_back = tk.Label(
            body_volver, text="VOLVER A\nREPUESTOS", font=("Segoe UI", 9, "bold"), 
            fg="#2c3e50", bg="white", justify="center"
        )
        lbl_txt_back.pack(expand=True)

        # Eventos para el botón volver
        widgets_volver = [btn_volver_frame, head_volver, lbl_ico_back, body_volver, lbl_txt_back]
        for w in widgets_volver:
            w.config(cursor="hand2")
            w.bind("<Button-1>", lambda e: self.filtrar_productos_pos())
            w.bind("<Enter>", lambda e, cp=btn_volver_frame, hv=head_volver: [
                cp.config(highlightbackground="#95a5a6", bg="#f9f9f9"),
                hv.config(bg="#95a5a6")
            ])
            w.bind("<Leave>", lambda e, cp=btn_volver_frame, hv=head_volver: [
                cp.config(highlightbackground="#dcdde1", bg="white"),
                hv.config(bg="#7f8c8d")
            ])

        # 3. Consulta de base de datos eficiente
        try:
            conn = sqlite3.connect(DB_COMBOS)
            cur = conn.cursor()
            cur.execute("SELECT id, nombre, precio_venta, costo_total FROM combos")
            combos = cur.fetchall()
            conn.close()
        except Exception as e:
            print(f"Error al consultar combos: {e}")
            return

        if not combos:
            tk.Label(self.grid_productos, text="No hay combos registrados", font=("Segoe UI", 12), fg="#7f8c8d", bg="#f5f7f8").grid(row=0, column=1, padx=20)
            return

        # 4. Renderizado de Mosaicos de Combos (start=1 para respetar el botón Volver)
        for i, (id_c, nom, precio, costo) in enumerate(combos, start=1):
            conn = sqlite3.connect(DB_COMBOS)
            receta = conn.execute("SELECT producto_codigo, cantidad FROM combo_detalle WHERE combo_id=?", (id_c,)).fetchall()
            conn.close()

            stock_combo = self.calcular_stock_disponible_combo(receta)
            
            # Semáforo de criticidad adaptado a paleta pastel premium
            if stock_combo <= 0:
                bg_badge, fg_badge = "#f8d7da", "#721c24"  # Rojo (Agotado)
                estado_txt = "AGOTADO"
            elif stock_combo <= 3:
                bg_badge, fg_badge = "#fff3cd", "#856404"  # Naranja (Crítico)
                estado_txt = f"STOCK CRÍTICO: {stock_combo}"
            else:
                bg_badge, fg_badge = "#d4edda", "#155724"  # Verde (Disponible)
                estado_txt = f"DISPONIBLE: {stock_combo}"

            # Contenedor Base del Mosaico Combo
            card_combo = tk.Frame(
                self.grid_productos, bg="white", highlightthickness=1, highlightbackground="#dcdde1",
                width=175, height=185
            )
            card_combo.grid(row=i//5, column=i%5, padx=8, pady=8, sticky="nsew")
            card_combo.pack_propagate(False)
            card_combo.grid_propagate(False)

            # Encabezado Dorado/Amarillo Premium (Mantiene la identidad de Promoción)
            head_combo = tk.Frame(card_combo, bg="#ffeaa7", height=75)
            head_combo.pack(fill="x", side="top")
            head_combo.pack_propagate(False)

            lbl_nom_combo = tk.Label(
                head_combo, text=nom.upper(), font=("Segoe UI", 9, "bold"), 
                fg="#b38f00", bg="#ffeaa7", wraplength=145, justify="center"
            )
            lbl_nom_combo.pack(expand=True, fill="x", padx=5)

            # Cuerpo inferior blanco para visualización limpia de Costos y Badges
            body_combo = tk.Frame(card_combo, bg="white", padx=10, pady=5)
            body_combo.pack(fill="both", expand=True)

            lbl_pre_combo = tk.Label(
                body_combo, text=f"${precio:.2f}", font=("Segoe UI", 14, "bold"), 
                fg="#27ae60", bg="white"
            )
            lbl_pre_combo.pack(expand=True, pady=(5, 2))

            lbl_stock_combo = tk.Label(
                body_combo, text=estado_txt, font=("Segoe UI", 8, "bold"), 
                bg=bg_badge, fg=fg_badge, padx=5, pady=3
            )
            lbl_stock_combo.pack(fill="x", side="bottom", pady=(0, 5))

            # --- ARQUITECTURA DE EVENTOS (PREVENCIÓN DE INTERFACES CONGELADAS) ---
            widgets_combo = [card_combo, head_combo, lbl_nom_combo, body_combo, lbl_pre_combo, lbl_stock_combo]
            
            # Congelamos los datos en memoria por cada iteración del bucle
            cmd_click = lambda e, n=nom, p=precio, c=costo, r=receta: \
                        self.agregar_combo_al_carrito(n, p, c, r)

            for w in widgets_combo:
                w.config(cursor="hand2")
                w.bind("<Button-1>", cmd_click) # Universalidad de clic
                
                # Efecto Hover dinámico sobre la tarjeta del combo
                w.bind("<Enter>", lambda e, cp=card_combo, hc=head_combo, ln=lbl_nom_combo, lp=lbl_pre_combo: [
                    cp.config(highlightbackground="#f1c40f", bg="#fdfaf2"),
                    hc.config(bg="#f1c40f"),
                    ln.config(bg="#f1c40f", fg="white"),
                    lp.config(bg="#fdfaf2")
                ])
                w.bind("<Leave>", lambda e, cp=card_combo, hc=head_combo, ln=lbl_nom_combo, lp=lbl_pre_combo: [
                    cp.config(highlightbackground="#dcdde1", bg="white"),
                    hc.config(bg="#ffeaa7"),
                    ln.config(bg="#ffeaa7", fg="#b38f00"),
                    lp.config(bg="white")
                ])

        # 5. Forzado del Canvas de Scroll
        self.grid_productos.update_idletasks()
        try:
            canvas_parent = self.grid_productos.master
            canvas_parent.configure(scrollregion=canvas_parent.bbox("all"), bg="#f5f7f8")
        except:
            pass





    def agregar_combo_al_carrito(self, nombre, precio, costo, receta):
        """
        Agrega un combo al carrito validando stock y habilitando edición inmediata.
        """
        # 1. VALIDACIÓN DE STOCK Y CÁLCULO DE LÍMITE
        cantidades_maximas = []
        
        for cod_p, cant_p in receta:
            # Consultamos stock actual y nombre de cada pieza en inventario_motos.db
            res = consulta("SELECT cantidad, nombre FROM repuestos WHERE codigo=?", (cod_p,))
             
            if not res or len(res) == 0:
                messagebox.showerror("Error", f"El componente [{cod_p}] ya no existe en el inventario.")
                return
            
            # Extraemos los valores de la tupla (cantidad, nombre)
            stock_actual = int(res[0][0])
            nombre_pieza = res[0][1]

            if stock_actual < cant_p:
                messagebox.showerror("Stock Insuficiente", 
                                   f"No se puede armar el combo.\n\n"
                                   f"Pieza: {nombre_pieza}\n"
                                   f"Requerido: {cant_p} | Disponible: {stock_actual}")
                return
            
            # Cálculo de cuántos combos permite fabricar esta pieza específica
            cantidades_maximas.append(stock_actual // cant_p)

        # El stock máximo real del combo es el valor más bajo de la lista
        stock_max_combo = min(cantidades_maximas)

        # 2. CONSTRUCCIÓN DEL OBJETO
        item_combo = {
            'codigo': f"CB-{nombre[:5].upper()}",
            'nombre': f"🎁 {nombre.upper()}",
            'marca': "PROMO",
            'modelo': "VARIOS",
            'precio': float(precio),
            'costo': float(costo),
            'cantidad': 1,
            'stock_max': stock_max_combo, # <--- Clave para modificar_cantidad_carrito
            'es_combo': True,
            'receta': receta
        }

        # 3. EVITAR DUPLICADOS
        for item in self.carrito:
            if item['codigo'] == item_combo['codigo']:
                messagebox.showinfo("Aviso", "Este combo ya está en el carrito. Haga doble clic para cambiar cantidad.")
                return

        # 4. INSERCIÓN Y REFRESCO
        self.carrito.append(item_combo)
        self.actualizar_tabla_pos()
        
        # --- ACTIVACIÓN QUIRÚRGICA DEL EVENTO ---
        # Aseguramos que el doble clic esté activo para el nuevo ítem
        self.tree_pos.bind("<Double-1>", self.modificar_cantidad_carrito)
        
        print(f"DEBUG: Combo {nombre} agregado con stock_max: {stock_max_combo}")

    def mostrar_inspeccion_producto(self, codigo, nombre, modelo, marca, precio_venta, stock_actual):
        """Despliega una ventana modal con la ficha técnica y costos del repuesto."""
        # 1. Consulta quirúrgica para traer el precio de costo en tiempo real
        res = consulta("SELECT precio_costo FROM repuestos WHERE codigo = ?", (codigo,))
        precio_costo = float(res[0][0]) if res and res[0][0] is not None else 0.0

        # Cálculos de rendimiento para la auditoría del operario
        ganancia_usd = precio_venta - precio_costo
        porc_utilidad = (ganancia_usd / precio_venta * 100) if precio_venta > 0 else 0.0

        # 2. Arquitectura de la Ventana Emergente
        win = tk.Toplevel(self.root)
        win.title(f"Ficha Técnica: {codigo}")
        win.geometry("460x520")
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()
        win.resizable(False, False)

        # Centrado milimétrico en pantalla
        win.update_idletasks()
        x = (win.winfo_screenwidth() // 2) - (460 // 2)
        y = (win.winfo_screenheight() // 2) - (520 // 2)
        win.geometry(f"+{x}+{y}")

        # Cabecera Institucional
        tk.Label(win, text="📋 DETALLE COMPLETO DEL PRODUCTO", font=("Segoe UI", 12, "bold"), 
                 fg="#3498db", bg="#2c3e50", pady=15).pack()

        # Ficha Contenedora Estilizada (Corta la monotonía del fondo)
        ficha = tk.Frame(win, bg="#34495e", padx=20, pady=15, highlightthickness=1, highlightbackground="#455a64")
        ficha.pack(fill="both", expand=True, padx=30, pady=10)

        # Inyección jerárquica de datos técnicos
        def agregar_fila_info(label, valor, color_val="white", es_bold=False):
            f = tk.Frame(ficha, bg="#34495e", pady=4)
            f.pack(fill="x")
            tk.Label(f, text=label, font=("Segoe UI", 9), fg="#bdc3c7", bg="#34495e", width=18, anchor="w").pack(side="left")
            tk.Label(f, text=valor, font=("Segoe UI", 10, "bold" if es_bold else "normal"), fg=color_val, bg="#34495e", anchor="w").pack(side="left")

        agregar_fila_info("Código de Parte:", codigo, "#f1c40f", es_bold=True)
        agregar_fila_info("Descripción:", nombre.upper(), "white", es_bold=True)
        agregar_fila_info("Marca de Fábrica:", marca.upper())
        agregar_fila_info("Compatibilidad:", modelo.upper())
        
        # Separador visual de costos
        ttk.Separator(ficha, orient="horizontal").pack(fill="x", pady=10)

        # Bloque Financiero Auditable
        agregar_fila_info("Precio de Costo:", f"${precio_costo:.2f}", "#e74c3c", es_bold=True)
        agregar_fila_info("Precio de Venta:", f"${precio_venta:.2f}", "#2ecc71", es_bold=True)
        agregar_fila_info("Utilidad Neta:", f"${ganancia_usd:.2f} ({porc_utilidad:.1f}%)", "#3498db", es_bold=True)
        
        # Separador visual de inventario
        ttk.Separator(ficha, orient="horizontal").pack(fill="x", pady=10)
        
        color_stock = "#2ecc71" if int(stock_actual) > 5 else ("#f39c12" if int(stock_actual) > 0 else "#e74c3c")
        agregar_fila_info("Stock Disponible:", f"{stock_actual} Unidades", color_stock, es_bold=True)

        # --- BOTONERA EN MATRIZ SIMÉTRICA DE ALTO IMPACTO (GRID) ---
        btns_frame = tk.Frame(win, bg="#2c3e50")
        btns_frame.pack(side="bottom", fill="x", padx=30, pady=20)
        btns_frame.columnconfigure(0, weight=1)
        btns_frame.columnconfigure(1, weight=1)

        estilo_btn = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}

        # Acción 1: Añadir al carrito y cerrar de una vez
        cmd_vender = lambda: [self.agregar_al_carrito(codigo, nombre, modelo, marca, precio_venta, stock_actual), win.destroy()]
        
        btn_add = tk.Button(btns_frame, text="🛒 AÑADIR A VENTA", bg="#27ae60", fg="white", 
                            activebackground="#2ecc71", activeforeground="white", command=cmd_vender, **estilo_btn)
        btn_add.grid(row=0, column=0, padx=(0, 5), sticky="nsew")

        # Acción 2: Cancelar / Salir de la ficha
        btn_close = tk.Button(btns_frame, text="❌ CERRAR", bg="#c0392b", fg="white", 
                              activebackground="#e74c3c", activeforeground="white", command=win.destroy, **estilo_btn)
        btn_close.grid(row=0, column=1, padx=(5, 0), sticky="nsew")



    

    def calcular_stock_disponible_combo(self, receta):
        """Calcula el stock máximo de un combo según sus componentes."""
        cantidades_maximas = []
        for cod_p, cant_necesaria in receta:
            res = consulta("SELECT cantidad FROM repuestos WHERE codigo=?", (cod_p,))
            stock_real = res[0][0] if res else 0
            # Si necesito 2 y tengo 5, puedo hacer 2.5 combos -> floor es 2.
            cantidades_maximas.append(stock_real // cant_necesaria)
        
        return min(cantidades_maximas) if cantidades_maximas else 0
    
    def procesar_nota_credito_cambio(self):
        """Paso 1 de la Logística: Extrae automáticamente la cedula y nombre de la factura e inicializa el canje."""
        sel = self.tree_finanzas.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione la factura de la cual se cambiarán los productos.")
            return

        valores = self.tree_finanzas.item(sel, "values")
        id_venta = valores[0]
        tags = self.tree_finanzas.item(sel, "tags")
        detalle_full = tags[0] if tags else ""

        # --- MOTOR AUTOMÁTICO DE EXTRACCIÓN DE IDENTIDAD ---
        cedula_detectada = "GENERICO"
        nombre_detectado = "CLIENTE GENERICO"

        if "CEDULA: " in detalle_full:
            try:
                # Cortamos para aislar la cédula guardada en el sello de la venta
                # Ejemplo: "... | CÉDULA: 20123456]" -> extrae "20123456"
                cedula_raw = detalle_full.split("CEDULA: ")[1].split("]")[0].strip()
                if cedula_raw and cedula_raw != "GENERICO":
                    cedula_detectada = cedula_raw
                    # Buscamos quirúrgicamente el nombre real en la DB central de clientes
                    res_cliente = consulta("SELECT nombre FROM clientes WHERE cedula = ?", (cedula_detectada,), db_path=DB_CLIENTES)
                    if res_cliente and len(res_cliente) > 0:
                        nombre_detectado = res_cliente[0][0]
            except Exception as err_identidad:
                print(f"DEBUG IDENTIDAD: Fallo al procesar metadatos del cliente: {err_identidad}")

        # Preparación de la glosa descriptiva para la cabecera
        cliente_label_final = f"{cedula_detectada} - {nombre_detectado}"

        # Clasificación de artículos reales vendidos
        lista_cruda = detalle_full.split(", ")
        items_factura_reales = []

        for item_str in lista_cruda:
            if "MÉTODO" not in item_str and "CANJE" not in item_str and "INFO-NC" not in item_str:
                if "[" in item_str and "]" in item_str and " x" in item_str:
                    try:
                        codigo = item_str[item_str.find("[")+1 : item_str.find("]")]
                        desc_completa = item_str[item_str.find("] ")+2 : item_str.find(" x")]
                        parte_cant = item_str.split(" x")[1].split(" ")[0]
                        cant_facturada = int(parte_cant)
                        
                        items_factura_reales.append({
                            'cod': codigo,
                            'nom': desc_completa,
                            'cant_max': cant_facturada,
                            'texto_original': item_str
                        })
                    except:
                        continue

        if not items_factura_reales:
            messagebox.showwarning("Atención", "Esta factura no contiene repuestos físicos legibles para procesar un canje.")
            return

        # Interfaz Maestra de Selección Multi-Ítem Rediseñada
        win = tk.Toplevel(self.root)
        win.title("Gestor Modular de Canjes Multi-Ítem")
        win.geometry("540x550")
        win.configure(bg="#2c3e50")
        win.transient(self.root)
        win.grab_set()
        win.resizable(False, False)

        # Centrado asíncrono corporativo
        win.withdraw()
        def _centrar_gestor_canjes_real():
            win.update_idletasks()
            x = (win.winfo_screenwidth() // 2) - (540 // 2)
            y = (win.winfo_screenheight() // 2) - (550 // 2)
            win.geometry(f"540x550+{x}+{y}")
            win.deiconify()
        win.after(10, _centrar_gestor_canjes_real)

        tk.Label(win, text="🔄 LOGÍSTICA DE DEVOLUCIÓN MULTI-ÍTEM", font=("Segoe UI", 12, "bold"), fg="#f39c12", bg="#2c3e50", pady=15).pack()
        
        # --- CASILLA AUTOMATIZADA DE LECTURA EXCLUSIVA ---
        id_frame = tk.Frame(win, bg="#2c3e50")
        id_frame.pack(fill="x", padx=40, pady=(0, 10))
        tk.Label(id_frame, text="Cliente Beneficiario del Vale:", font=("Segoe UI", 9, "bold"), fg="#bdc3c7", bg="#2c3e50").pack(side="left")
        
        # El entry se llena solo y cambia a color de lectura de sistema (#ebd4d4)
        ent_cliente = tk.Entry(id_frame, font=("Segoe UI", 10, "bold"), justify="center", width=32, bg="#ebd4d4", fg="#2c3e50")
        ent_cliente.pack(side="left", padx=10)
        ent_cliente.insert(0, cliente_label_final)
        ent_cliente.config(state="disabled") # CORRECCIÓN: Bloqueado contra modificaciones manuales

        # Sistema de rejilla deslizable (Canvas) para componentes checks
        scroll_frame = tk.Frame(win, bg="#2c3e50")
        scroll_frame.pack(fill="both", expand=True, padx=35, pady=5)

        canvas = tk.Canvas(scroll_frame, bg="#2c3e50", highlightthickness=0)
        v_scroll = ttk.Scrollbar(scroll_frame, orient="vertical", command=canvas.yview)
        frame_checklist = tk.Frame(canvas, bg="#2c3e50")
        
        canvas_window = canvas.create_window((0, 0), window=frame_checklist, anchor="nw", width=450)
        canvas.configure(yscrollcommand=v_scroll.set)
        
        canvas.pack(side="left", fill="both", expand=True)
        v_scroll.pack(side="right", fill="y")
        
        frame_checklist.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        win.bind("<MouseWheel>", lambda e: canvas.yview_scroll(int(-1*(e.delta/120)), "units"))

        referencias_ui_canje = []

        for index, item in enumerate(items_factura_reales):
            row_frame = tk.Frame(frame_checklist, bg="#34495e", pady=8, padx=15, highlightthickness=1, highlightbackground="#455a64")
            row_frame.pack(fill="x", pady=4)

            var_check = tk.BooleanVar(value=False)
            cb = tk.Checkbutton(row_frame, variable=var_check, bg="#34495e", activebackground="#34495e", selectcolor="#2c3e50")
            cb.pack(side="left")

            descripcion_corta = (item['nom'][:28] + '..') if len(item['nom']) > 28 else item['nom']
            lbl_desc = tk.Label(row_frame, text=f"[{item['cod']}] {descripcion_corta}", font=("Segoe UI", 9, "bold"), fg="white", bg="#34495e")
            lbl_desc.pack(side="left", padx=5)

            tk.Label(row_frame, text="Cant. a Dev:", font=("Segoe UI", 8), fg="#bdc3c7", bg="#34495e").pack(side="left", padx=(10, 2))
            ent_cant_dev = tk.Entry(row_frame, font=("Segoe UI", 10, "bold"), justify="center", width=4, bg="white", fg="#2c3e50")
            ent_cant_dev.pack(side="left")
            ent_cant_dev.insert(0, str(item['cant_max']))

            tk.Label(row_frame, text=f"Max: {item['cant_max']}", font=("Segoe UI", 8, "italic"), fg="#2ecc71", bg="#34495e").pack(side="left", padx=5)

            referencias_ui_canje.append({
                'item_data': item,
                'check_var': var_check,
                'entry_widget': ent_cant_dev
            })

        # Motor de procesamiento consolidado
        def ejecutar_consolidacion_canje_masivo():
            monto_acumulado_vale = 0.0
            costo_acumulado_reingreso = 0.0
            glosas_de_devolucion_log = []
            se_selecciono_algo = False

            for ref in referencias_ui_canje:
                if ref['check_var'].get() == True:
                    se_selecciono_algo = True
                    datos_pieza = ref['item_data']
                    
                    try:
                        cantidad_devolver = int(ref['entry_widget'].get().strip())
                    except ValueError:
                        messagebox.showerror("Error de Formato", f"La cantidad para [{datos_pieza['cod']}] debe ser un número entero.", parent=win)
                        return

                    if cantidad_devolver <= 0 or cantidad_devolver > datos_pieza['cant_max']:
                        messagebox.showerror("Límites Excedidos", f"Monto inválido para [{datos_pieza['cod']}]. Permitido entre 1 y {datos_pieza['cant_max']}.", parent=win)
                        return

                    res_valores = consulta("SELECT precio_costo, precio_venta FROM repuestos WHERE codigo=?", (datos_pieza['cod'],), db_path=DB_NAME)
                    if res_valores and len(res_valores) > 0:
                        costo_u, precio_u = res_valores[0]
                    else:
                        costo_u, precio_u = 0.0, 0.0

                    monto_acumulado_vale += (precio_u * cantidad_devolver)
                    costo_acumulado_reingreso += (costo_u * cantidad_devolver)

                    consulta("UPDATE repuestos SET cantidad = cantidad + ? WHERE codigo = ?", (cantidad_devolver, datos_pieza['cod']), fetch=False, db_path=DB_NAME)
                    glosas_de_devolucion_log.append(f"[{datos_pieza['cod']}] {datos_pieza['nom']} x{cantidad_devolver}")

            if not se_selecciono_algo:
                messagebox.showwarning("Falta Selección", "Marque las casillas de verificación de los productos que el cliente va a cambiar.", parent=win)
                return

            try:
                num_nc = f"NC-{random.randint(1000, 9999)}"
                
                # Guardamos la Nota de Crédito usando el nombre e identidad que el sistema auto-detectó
                consulta(
                    """INSERT INTO notas_credito 
                       (codigo_nota, factura_origen_id, cliente, monto_a_favor, costo_devuelto, estado) 
                       VALUES (?, ?, ?, ?, ?, 'VIGENTE')""",
                    (num_nc, id_venta, nombre_detectado, round(monto_acumulado_vale, 2), round(costo_acumulado_reingreso, 2),),
                    fetch=False, db_path=DB_VENTAS
                )

                # --- COMPROBANTE TICKET EMITIDO ---
                win_nc_ticket = tk.Toplevel(self.root)
                win_nc_ticket.title("Comprobante de Canje")
                win_nc_ticket.geometry("450x460")
                win_nc_ticket.configure(bg="#2c3e50")
                win_nc_ticket.transient(self.root)
                win_nc_ticket.grab_set()
                win_nc_ticket.resizable(False, False)

                win_nc_ticket.withdraw()
                win_nc_ticket.after(10, lambda: [
                    win_nc_ticket.update_idletasks(),
                    win_nc_ticket.geometry(f"450x440+{(win_nc_ticket.winfo_screenwidth()//2)-(450//2)}+{(win_nc_ticket.winfo_screenheight()//2)-(440//2)}"),
                    win_nc_ticket.deiconify()
                ])

                tk.Label(win_nc_ticket, text="🎟️ VALE CONSOLIDADO GENERADO", font=("Segoe UI", 12, "bold"), fg="#2ecc71", bg="#2c3e50", pady=15).pack()

                ticket_frame = tk.Frame(win_nc_ticket, bg="#1e272e", padx=25, pady=15, highlightthickness=1, highlightbackground="#2f3640")
                ticket_frame.pack(fill="both", expand=True, padx=35, pady=5)

                tk.Label(ticket_frame, text="CÓDIGO DE CANJE ÚNICO:", font=("Segoe UI", 9, "bold"), fg="#7f8c8d", bg="#1e272e").pack(anchor="w")
                tk.Label(ticket_frame, text=num_nc, font=("Consolas", 18, "bold"), fg="#f1c40f", bg="#1e272e").pack(anchor="w", pady=(2, 6))

                tk.Label(ticket_frame, text="CLIENTE BENEFICIARIO:", font=("Segoe UI", 9, "bold"), fg="#7f8c8d", bg="#1e272e").pack(anchor="w")
                tk.Label(ticket_frame, text=f"V-{cedula_detectada} | {nombre_detectado}", font=("Segoe UI", 10, "bold"), fg="white", bg="#1e272e").pack(anchor="w", pady=(2, 6))

                tk.Label(ticket_frame, text="MONTO TOTAL ACUMULADO A FAVOR:", font=("Segoe UI", 9, "bold"), fg="#7f8c8d", bg="#1e272e").pack(anchor="w")
                tk.Label(ticket_frame, text=f"${monto_acumulado_vale:.2f}", font=("Segoe UI", 20, "bold"), fg="#2ecc71", bg="#1e272e").pack(anchor="w", pady=(2, 6))

                tk.Label(ticket_frame, text="PIEZAS CANJEADAS:", font=("Segoe UI", 8, "bold"), fg="#bdc3c7", bg="#1e272e").pack(anchor="w")
                lbl_res_piezas = tk.Label(ticket_frame, text=", ".join(glosas_de_devolucion_log), font=("Segoe UI", 8, "italic"), fg="#819090", bg="#1e272e", wraplength=340, justify="left")
                lbl_res_piezas.pack(anchor="w")

                tk.Label(win_nc_ticket, text="📌 Introduzca este código en el Punto de Venta\nal finalizar la nueva compra para restar el saldo consolidado.", 
                         font=("Segoe UI", 8, "italic"), fg="#bdc3c7", bg="#2c3e50", justify="center", pady=10).pack()

                tk.Button(win_nc_ticket, text="✅ ENTENDIDO / CERRAR", bg="#27ae60", fg="white", activebackground="#2ecc71",
                          font=("Segoe UI", 10, "bold"), relief="flat", height=2, cursor="hand2", command=win_nc_ticket.destroy).pack(fill="x", padx=35, pady=(5, 15))

                win.destroy()
                self.actualizar_tabla_finanzas()
                self.cargar_tabla_gestion()
                self.actualizar_estadisticas()

            except Exception as err_db:
                messagebox.showerror("Error Crítico", f"Fallo al registrar la transacción en disco: {err_db}", parent=win)

        # --- BOTONERA PRINCIPAL EN REJILLA DE REFUERZO ---
        btns_frame = tk.Frame(win, bg="#2c3e50")
        btns_frame.pack(side="bottom", fill="x", padx=40, pady=20)
        btns_frame.columnconfigure(0, weight=1)
        btns_frame.columnconfigure(1, weight=1)

        estilo_btn = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}
        tk.Button(btns_frame, text="✅ PROCESAR CANJES", bg="#27ae60", fg="white", command=ejecutar_consolidacion_canje_masivo, **estilo_btn).grid(row=0, column=0, padx=(0,5), sticky="nsew")
        tk.Button(btns_frame, text="❌ ABORTAR", bg="#c0392b", fg="white", command=win.destroy, **estilo_btn).grid(row=0, column=1, padx=(5,0), sticky="nsew")






    def abrir_visor_vales_canje(self):
        """Despliega el panel corporativo independiente para auditar Notas de Crédito e historiales por Cedula."""
        win_vales = tk.Toplevel(self.root)
        win_vales.title("Auditoría de Notas de Crédito y Canjes")
        win_vales.geometry("650x580") # Dimensiones holgadas para la grilla expandida
        win_vales.configure(bg="#2c3e50")
        win_vales.transient(self.root)
        win_vales.grab_set()
        win_vales.resizable(False, False)

        # Centrado Absoluto Real Asíncrono sin parpadeos en esquinas
        win_vales.withdraw()
        def _centrar_vales_panel_real():
            win_vales.update_idletasks()
            ancho_p = win_vales.winfo_screenwidth()
            alto_p = win_vales.winfo_screenheight()
            x = (ancho_p // 2) - (650 // 2)
            y = (alto_p // 2) - (580 // 2)
            win_vales.geometry(f"650x580+{x}+{y}")
            win_vales.deiconify()
        win_vales.after(10, _centrar_vales_panel_real)

        # Cabecera Institucional
        tk.Label(win_vales, text="🎟️ CONTROL DE VALES DE CANJE E HISTORIALES", font=("Segoe UI", 13, "bold"), fg="#9b59b6", bg="#2c3e50", pady=12).pack()

        # --- CONTENEDOR DE BÚSQUEDA ANALÍTICA POR CÉDULA ---
        search_frame = tk.Frame(win_vales, bg="white", pady=8, highlightthickness=1, highlightbackground="#dcdde1")
        search_frame.pack(fill="x", padx=30, pady=5)
        
        tk.Label(search_frame, text="🪪 Filtrar por Cedula del Cliente:", font=("Segoe UI", 9, "bold"), bg="white", fg="#7f8c8d").pack(side="left", padx=(15, 5))
        ent_buscar_cedula_nc = tk.Entry(search_frame, font=("Segoe UI", 10, "bold"), width=18, justify="center", relief="flat", highlightthickness=1, highlightbackground="#bdc3c7")
        ent_buscar_cedula_nc.pack(side="left", padx=5)
        ent_buscar_cedula_nc.focus()

        # --- TABLA DE AUDITORÍA MULTIPROPÓSITO ---
        tabla_frame = tk.Frame(win_vales, bg="white")
        tabla_frame.pack(fill="both", expand=True, padx=30, pady=10)

        cols = ("code", "factura", "cli", "monto")
        tree_nc = ttk.Treeview(tabla_frame, columns=cols, show="headings")
        tree_nc.pack(side="left", fill="both", expand=True)

        scroll = ttk.Scrollbar(tabla_frame, orient="vertical", command=tree_nc.yview)
        scroll.pack(side="right", fill="y")
        tree_nc.configure(yscrollcommand=scroll.set)

        # --- MOTOR DE CARGA Y EXPLOSIÓN HISTÓRICA EN CALIENTE ---
                # --- MOTOR DE CARGA Y EXPLOSIÓN HISTÓRICA EN CALIENTE (CORREGIDO) ---
        def filtrar_vales_por_cedula(event=None):
            tree_nc.delete(*tree_nc.get_children())
            
            # Limpiamos rigurosamente la entrada eliminando letras V/E al inicio, espacios, puntos y guiones
            entrada_cruda = ent_buscar_cedula_nc.get().strip().replace(".", "").replace("-", "").upper()
            
            # Si el usuario escribió por ejemplo "V-123456" o "V123456", aislamos solo el número
            ced_filtro = "".join([char for char in entrada_cruda if char.isdigit()])
            
            # MODO A: Si digitan una cédula válida (Muestra su historial de compras para iniciar canje)
            if len(ced_filtro) >= 4:
                tree_nc.heading("code", text="Nro Factura")
                tree_nc.heading("factura", text="Fecha Venta")
                tree_nc.heading("cli", text="Detalle Histórico de Repuestos Comprados")
                tree_nc.heading("monto", text="Total ($)")
                
                tree_nc.column("code", width=95, anchor="center")
                tree_nc.column("factura", width=105, anchor="center")
                tree_nc.column("cli", width=290, anchor="w")
                tree_nc.column("monto", width=70, anchor="center")

                # CLAVE: Buscamos solo el número de cédula en cualquier parte de la cadena 'productos'
                query = "SELECT id, date(fecha_hora), productos, total FROM ventas_log WHERE productos LIKE ? ORDER BY id DESC"
                facturas_cliente = consulta(query, (f"%{ced_filtro}%",), db_path=DB_VENTAS)
                
                if facturas_cliente:
                    for f in facturas_cliente:
                        folio = f"FACT-{str(f[0]).zfill(5)}"
                        # Reemplazo seguro de caracteres para visualización premium
                        articulos_limpios = str(f[2]).replace("[MÉTODO:", "📌 [MÉTODO:")
                        tree_nc.insert("", "end", values=(folio, f[1], articulos_limpios, f"${f[3]:.2f}"), tags=(f[2], f[0]))
                else:
                    # Si no encuentra facturas, buscamos si el texto coincide con el nombre del deudor
                    query_backup = "SELECT id, date(fecha_hora), productos, total FROM ventas_log WHERE productos LIKE ? ORDER BY id DESC"
                    facturas_nombre = consulta(query_backup, (f"%{entrada_cruda}%",), db_path=DB_VENTAS)
                    for f in facturas_nombre:
                        folio = f"FACT-{str(f[0]).zfill(5)}"
                        articulos_limpios = str(f[2]).replace("[MÉTODO:", "📌 [MÉTODO:")
                        tree_nc.insert("", "end", values=(folio, f[1], articulos_limpios, f"${f[3]:.2f}"), tags=(f[2], f[0]))
            
            # MODO B: Si el buscador está vacío o tiene menos de 4 dígitos (Muestra Notas de Crédito NC- activas)
            else:
                tree_nc.heading("code", text="Código Vale")
                tree_nc.heading("factura", text="Factura Origen")
                tree_nc.heading("cli", text="Cliente Beneficiario")
                tree_nc.heading("monto", text="Saldo ($)")
                
                tree_nc.column("code", width=90, anchor="center")
                tree_nc.column("factura", width=95, anchor="center")
                tree_nc.column("cli", width=220, anchor="w")
                tree_nc.column("monto", width=65, anchor="center")

                query_nc = "SELECT codigo_nota, factura_origen_id, cliente, monto_a_favor FROM notas_credito WHERE estado='VIGENTE' ORDER BY codigo_nota ASC"
                registros = consulta(query_nc, db_path=DB_VENTAS)
                for r in registros:
                    folio_origen = f"FACT-{str(r[1]).zfill(5)}"
                    tree_nc.insert("", "end", values=(r[0], folio_origen, str(r[2]).upper(), f"${r[3]:.2f}"), tags=("VALE_ACTIVO", r[0]))


        # --- PASARELA DE DOBLE CLIC INTELIGENTE ---
        def pasarela_doble_clic_vales(event):
            sel = tree_nc.selection()
            if not sel: return
            valores_fila = tree_nc.item(sel)["values"]
            tags_fila = tree_nc.item(sel)["tags"]
            
            if "FACT-" in str(valores_fila[0]):
                # Doble Clic en Historial: Abre directo el gestor de devoluciones multi-ítem
                win_vales.destroy()
                self.procesar_nota_credito_cambio() 
            else:
                # Doble Clic en NC Activo: Copia automática del cupón a la RAM
                codigo_nc_copiar = valores_fila[0]
                self.root.clipboard_clear()
                self.root.clipboard_append(codigo_nc_copiar)
                messagebox.showinfo("Copiado", f"📋 Código '{codigo_nc_copiar}' copiado al portapapeles.\n\nPéguelo directamente en el POS (Ctrl + V).", parent=win_vales)

        tree_nc.bind("<Double-1>", pasarela_doble_clic_vales)

        # --- MOTOR DE ANULACIÓN Y REVERSIÓN DE VALES NC ---
        def ejecutar_anulacion_vale_nc():
            sel_nc = tree_nc.selection()
            if not sel_nc:
                messagebox.showwarning("Atención", "Seleccione el vale activo que desea anular de la lista.", parent=win_vales)
                return

            valores_nc = tree_nc.item(sel_nc, "values")
            tags_nc = tree_nc.item(sel_nc, "tags")
            
            if tags_nc[0] != "VALE_ACTIVO":
                messagebox.showwarning("Acción Inválida", "Solo se pueden anular Vales NC vigentes. Las facturas históricas se procesan desde Finanzas.", parent=win_vales)
                return

            codigo_nc = valores_nc[0]
            cliente_nc = valores_nc[2]

            # Confirmación Premium Modal
            win_conf_anular = tk.Toplevel(win_vales)
            win_conf_anular.title("Confirmar Anulación")
            win_conf_anular.geometry("450x320")
            win_conf_anular.configure(bg="#2c3e50")
            win_conf_anular.transient(win_vales)
            win_conf_anular.grab_set()
            
            win_conf_anular.withdraw()
            win_conf_anular.after(10, lambda: [
                win_conf_anular.update_idletasks(),
                win_conf_anular.geometry(f"450x320+{(win_conf_anular.winfo_screenwidth()//2)-(450//2)}+{(win_conf_anular.winfo_screenheight()//2)-(320//2)}"),
                win_conf_anular.deiconify()
            ])

            self.confirmacion_anulacion_valida = False

            tk.Label(win_conf_anular, text="⚠️ ADVERTENCIA DE ANULACIÓN", font=("Segoe UI", 12, "bold"), fg="#e74c3c", bg="#2c3e50", pady=15).pack()
            alert_frame = tk.Frame(win_conf_anular, bg="#1e272e", padx=20, pady=15, highlightthickness=1, highlightbackground="#3d566e")
            alert_frame.pack(fill="both", expand=True, padx=30, pady=5)

            tk.Label(alert_frame, text=f"¿Está seguro de anular el vale {codigo_nc} de '{cliente_nc}'?", font=("Segoe UI", 10, "bold"), fg="white", bg="#1e272e", wraplength=350, justify="center").pack(pady=(0, 10))
            tk.Label(alert_frame, text="El cliente conservará su producto original. El stock físico se volverá a descontar de forma automática en el almacén y el vale quedará INHABILITADO permanentemente.", font=("Segoe UI", 9, "italic"), fg="#f1c40f", bg="#1e272e", wraplength=350, justify="center").pack()

            btns_conf_frame = tk.Frame(win_conf_anular, bg="#2c3e50")
            btns_conf_frame.pack(side="bottom", fill="x", padx=30, pady=(15, 20))
            btns_conf_frame.columnconfigure(0, weight=1); btns_conf_frame.columnconfigure(1, weight=1)

            estilo_conf_btn = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}

            tk.Button(btns_conf_frame, text="🛑 SÍ, ANULAR VALE", bg="#c0392b", fg="white", activebackground="#e74c3c", command=lambda: [setattr(self, 'confirmacion_anulacion_valida', True), win_conf_anular.destroy()], **estilo_conf_btn).grid(row=0, column=0, padx=(0, 4), sticky="nsew")
            tk.Button(btns_conf_frame, text="🔄 REVERTIR ACCIÓN", bg="#7f8c8d", fg="white", activebackground="#95a5a6", command=win_conf_anular.destroy, **estilo_conf_btn).grid(row=0, column=1, padx=(4, 0), sticky="nsew")

            win_conf_anular.wait_window()
            if not getattr(self, 'confirmacion_anulacion_valida', False): return

            # --- RUTA DE REVERSIÓN LOGÍSTICA BLINDADA ANTI-CRASHES ---
            try:
                res_detalles_nc = consulta("SELECT monto_a_favor, costo_devuelto, factura_origen_id FROM notas_credito WHERE codigo_nota=?", (codigo_nc,), db_path=DB_VENTAS)
                if not res_detalles_nc:
                    messagebox.showerror("Error", "No se encontraron los metadatos contables del vale.", parent=win_vales)
                    return
                monto_nc, costo_nc, id_origen = res_detalles_nc[0]

                res_factura_string = consulta("SELECT productos FROM ventas_log WHERE id=?", (id_origen,), db_path=DB_VENTAS)
                if res_factura_string and len(res_factura_string) > 0:
                    factura_full_txt = res_factura_string[0][0]
                    items_factura = factura_full_txt.split(", ")
                    
                    for item_f in items_factura:
                        if "[" in item_f and "]" in item_f and "MÉTODO" not in item_f and "CANJE" not in item_f:
                            cod_pieza_revertir = item_f[item_f.find("[")+1 : item_f.find("]")]
                            if " x" in item_f:
                                parte_cant = item_f.split(" x")[1].split(" ")[0]
                                cant_pieza_revertir = int(parte_cant)

                                # Redirección explícita a la DB de Inventario (DB_NAME) para restar existencias
                                consulta("UPDATE repuestos SET cantidad = cantidad - ? WHERE codigo = ?", 
                                         (cant_pieza_revertir, cod_pieza_revertir), fetch=False, db_path=DB_NAME)

                # Inhabilitación del cupón en DB_VENTAS
                consulta("UPDATE notas_credito SET estado='ANULADA' WHERE codigo_nota=?", (codigo_nc,), fetch=False, db_path=DB_VENTAS)
                messagebox.showinfo("Vale Anulado", f"✅ El vale {codigo_nc} ha sido inhabilitado.\n\nEl stock de los repuestos ha sido corregido en el almacén.", parent=win_vales)
                
                filtrar_vales_por_cedula()
                self.cargar_tabla_gestion()
                self.actualizar_estadisticas()
            except Exception as e:
                messagebox.showerror("Error Crítico", f"No se pudo revertir el canje en disco: {e}", parent=win_vales)

        # --- BOTONERA PRINCIPAL EN REJILLA SIMÉTRICA DE CONTROL (GRID) ---
        btns_vales_frame = tk.Frame(win_vales, bg="#2c3e50")
        btns_vales_frame.pack(side="bottom", fill="x", padx=30, pady=(5, 20))
        btns_vales_frame.columnconfigure(0, weight=1); btns_vales_frame.columnconfigure(1, weight=1)

        estilo_vales_btn = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}

        tk.Button(btns_vales_frame, text="🗑️ ANULAR VALE NC", bg="#c0392b", fg="white", activebackground="#e74c3c", command=ejecutar_anulacion_vale_nc, **estilo_vales_btn).grid(row=0, column=0, padx=(0, 6), sticky="nsew")
        tk.Button(btns_vales_frame, text="❌ CERRAR PANEL", bg="#34495e", fg="white", activebackground="#455a64", command=win_vales.destroy, **estilo_vales_btn).grid(row=0, column=1, padx=(6, 0), sticky="nsew")

        filtrar_vales_por_cedula() # Carga por defecto inicial


    def verificar_o_registrar_cliente_pos(self):
        """Verifica la existencia del cliente. Si es nuevo, abre la pasarela de registro reducida."""
        cedula = self.ent_cedula_pos.get().strip().replace(".", "").replace("-", "").upper()
        if not cedula: return

        # Consultar la DB central de clientes
        res = consulta("SELECT nombre FROM clientes WHERE cedula = ?", (cedula,), db_path=DB_CLIENTES)
        
        if res:
            # Cliente existente: Seteamos su nombre en la barra del POS
            nombre_cliente = res[0][0]
            self.lbl_info_cliente_pos.config(text=f"{nombre_cliente}", fg="#2ecc71")
            messagebox.showinfo("Cliente Encontrado", f"🏍️ ¡Bienvenido de nuevo, {nombre_cliente}!\nSu historial ha sido vinculado a esta orden.")
        else:
            # Cliente nuevo: Abrir subventana modal premium compacta
            win_reg_c = tk.Toplevel(self.root)
            win_reg_c.title("Alta de Nuevo Cliente")
            win_reg_c.geometry("400x280") # Altura reducida para perfecta simetría
            win_reg_c.configure(bg="#2c3e50")
            win_reg_c.transient(self.root)
            win_reg_c.grab_set()
            win_reg_c.resizable(False, False)

            # Centrado absoluto real anti-parpadeo
            win_reg_c.withdraw()
            win_reg_c.after(10, lambda: [
                win_reg_c.update_idletasks(),
                win_reg_c.geometry(f"400x280+{(win_reg_c.winfo_screenwidth()//2)-(400//2)}+{(win_reg_c.winfo_screenheight()//2)-(280//2)}"),
                win_reg_c.deiconify()
            ])

            tk.Label(win_reg_c, text="🪪 REGISTRAR NUEVO CLIENTE", font=("Segoe UI", 12, "bold"), fg="#f39c12", bg="#2c3e50", pady=15).pack()

            form_frame = tk.Frame(win_reg_c, bg="#2c3e50")
            form_frame.pack(fill="x", padx=40, pady=10)

            def crear_campo_c(label, default_val=""):
                f = tk.Frame(form_frame, bg="#2c3e50", pady=8)
                f.pack(fill="x")
                lbl = tk.Label(
                    f, text=label, font=("Segoe UI", 10, "bold"), 
                    fg="#bdc3c7", bg="#2c3e50", 
                    width=18, anchor="e"
                )
                lbl.pack(side="left", padx=(0, 15)) # Añade un colchón de 15 píxeles antes de la caja blanca

                ent = tk.Entry(f, font=("Segoe UI", 11), bg="white", fg="#2c3e50", relief="flat")
                ent.pack(side="left", fill="x", expand=True)
                
                if default_val: 
                    ent.insert(0, default_val)
                return ent

            ent_c_ced = crear_campo_c("Cedula / ID:", cedula)
            ent_c_nom = crear_campo_c("Nombre Completo  :",)
            ent_c_nom.focus()

            def guardar_nuevo_cliente_db():
                c_ced = ent_c_ced.get().strip().upper()
                c_nom = ent_c_nom.get().strip().upper()
                
                if not c_ced or not c_nom:
                    messagebox.showwarning("Atención", "Cedula y Nombre son campos obligatorios.", parent=win_reg_c)
                    return
                
                try:
                    fecha_hoy = datetime.now().strftime("%Y-%m-%d")
                    # Inserción limpia con las 3 columnas solicitadas
                    consulta("INSERT INTO clientes (cedula, nombre, fecha_registro) VALUES (?, ?, ?)",
                             (c_ced, c_nom, fecha_hoy), fetch=False, db_path=DB_CLIENTES)
                    
                    self.lbl_info_cliente_pos.config(text=f"{c_nom}", fg="#2ecc71")
                    messagebox.showinfo("Éxito", f"Cliente '{c_nom}' registrado correctamente en el ERP.", parent=win_reg_c)
                    win_reg_c.destroy()
                except sqlite3.IntegrityError:
                    messagebox.showerror("Error", "Esta cedula ya se encuentra registrada.", parent=win_reg_c)

            # Botonera simétrica de bloque ejecutivo
            btns_frame = tk.Frame(win_reg_c, bg="#2c3e50")
            btns_frame.pack(side="bottom", fill="x", padx=40, pady=20)
            btns_frame.columnconfigure(0, weight=1); btns_frame.columnconfigure(1, weight=1)
            estilo_btn = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}

            tk.Button(btns_frame, text="GUARDAR", bg="#27ae60", fg="white", command=guardar_nuevo_cliente_db, **estilo_btn).grid(row=0, column=0, padx=(0,4), sticky="nsew")
            tk.Button(btns_frame, text="CANCELAR", bg="#c0392b", fg="white", command=win_reg_c.destroy, **estilo_btn).grid(row=0, column=1, padx=(4,0), sticky="nsew")


    def crear_pantalla_clientes(self):
        """Crea la interfaz completa para auditar, buscar, editar y eliminar clientes."""
        frame = tk.Frame(self.contenedor, bg="#2c3e50")
        self.frames["clientes"] = frame

        # --- BARRA SUPERIOR INSTITUCIONAL ---
        top = tk.Frame(frame, bg="#2c3e50")
        top.pack(fill="x", pady=10, padx=15)

        tk.Label(
            top, text="👥 PANEL DE GESTIÓN DE CLIENTES", font=("Segoe UI", 20, "bold"), 
            bg="#2c3e50", fg="#ffffff"
        ).pack(side="left", padx=(0, 20))

        tk.Button(
            top, text="⬅ Volver", font=("Segoe UI", 10, "bold"), bg="#7f8c8d", fg="white",
            bd=0, relief="flat", padx=15, pady=5, cursor="hand2",
            command=lambda: self.mostrar_pantalla("menu", direccion="left")
        ).pack(side="right")

        # --- BARRA DE BÚSQUEDA EN CALIENTE ---
        search_frame = tk.Frame(frame, bg="white", pady=8, highlightthickness=1, highlightbackground="#dcdde1")
        search_frame.pack(fill="x", padx=15, pady=(5, 10))

        tk.Label(search_frame, text="🔍 Buscar por Cedula o Nombre:", font=("Segoe UI", 9, "bold"), bg="white", fg="#7f8c8d").pack(side="left", padx=(15, 5))
        self.entry_buscar_cliente = tk.Entry(search_frame, font=("Segoe UI", 11), width=35, relief="flat", highlightthickness=1, highlightbackground="#bdc3c7")
        self.entry_buscar_cliente.pack(side="left", padx=5)
        self.entry_buscar_cliente.bind("<KeyRelease>", lambda e: self.cargar_tabla_clientes())

        # --- TABLA DE DATOS (TREEVIEW) ---
        tabla_frame = tk.Frame(frame, bg="#f4f6f7")
        tabla_frame.pack(fill="both", expand=True, padx=15, pady=5)

        cols = ("cedula", "nombre", "fecha")
        self.tree_clientes = ttk.Treeview(tabla_frame, columns=cols, show="headings", selectmode="browse")
        
        self.tree_clientes.heading("cedula", text="Cedula de Identidad")
        self.tree_clientes.heading("nombre", text="Nombre Completo del Cliente")
        self.tree_clientes.heading("fecha", text="Fecha de Registro en ERP")
        
        self.tree_clientes.column("cedula", width=150, anchor="center")
        self.tree_clientes.column("nombre", width=350, anchor="w")
        self.tree_clientes.column("fecha", width=150, anchor="center")

        self.tree_clientes.pack(side="left", fill="both", expand=True)
        
        scroll = ttk.Scrollbar(tabla_frame, orient="vertical", command=self.tree_clientes.yview)
        scroll.pack(side="right", fill="y")
        self.tree_clientes.configure(yscrollcommand=scroll.set)

        # Enlace rápido: doble clic sobre una fila abre la ventana de edición de forma automatizada
        self.tree_clientes.bind("<Double-1>", lambda e: self.editar_cliente_perfil())

        # --- BARRA DE ACCIONES INFERIOR EN REJILLA SIMÉTRICA (GRID) ---
        panel_inferior = tk.Frame(frame, bg="#2c3e50", pady=15)
        panel_inferior.pack(fill="x", side="bottom")

        btns_grid = tk.Frame(panel_inferior, bg="#2c3e50")
        btns_grid.pack(side="bottom", pady=5)
        
        # Configuramos 2 columnas equilibradas con peso proporcional para simetría exacta
        btns_grid.columnconfigure(0, weight=1)
        btns_grid.columnconfigure(1, weight=1)

        estilo_bloque = {"relief": "flat", "width": 22, "height": 2, "font": ("Segoe UI", 10, "bold"), "cursor": "hand2"}

        tk.Button(btns_grid, text="✏️ EDITAR CLIENTE", bg="#3498db", fg="white",
                  activebackground="#2980b9", activeforeground="white",
                  command=self.editar_cliente_perfil, **estilo_bloque).grid(row=0, column=0, padx=5, sticky="nsew")

        tk.Button(btns_grid, text="🗑️ BORRAR CLIENTE", bg="#e74c3c", fg="white",
                  activebackground="#c0392b", activeforeground="white",
                  command=self.eliminar_cliente_perfil, **estilo_bloque).grid(row=0, column=1, padx=5, sticky="nsew")


    def cargar_tabla_clientes(self):
        """Filtra y renderiza en caliente los perfiles registrados en la base de datos central."""
        self.tree_clientes.delete(*self.tree_clientes.get_children())
        filtro = self.entry_buscar_cliente.get().strip().upper() if hasattr(self, 'entry_buscar_cliente') else ""
        
        if filtro:
            query = "SELECT cedula, nombre, fecha_registro FROM clientes WHERE cedula LIKE ? OR nombre LIKE ? ORDER BY nombre ASC"
            datos = consulta(query, (f"%{filtro}%", f"%{filtro}%"), db_path=DB_CLIENTES)
        else:
            query = "SELECT cedula, nombre, fecha_registro FROM clientes ORDER BY nombre ASC"
            datos = consulta(query, db_path=DB_CLIENTES)

        for d in datos:
            self.tree_clientes.insert("", "end", values=(d[0], d[1], d[2]))

    def editar_cliente_perfil(self):
        """Abre una ventana modal premium y centrada para reajustar los datos del deudor."""
        sel = self.tree_clientes.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione un cliente de la lista para editar.", parent=self.frames["clientes"])
            return
        
        valores = self.tree_clientes.item(sel, "values")
        cedula_original = valores[0]
        nombre_original = valores[1]

        win_edit = tk.Toplevel(self.root)
        win_edit.title(f"Modificar Perfil: {cedula_original}")
        win_edit.geometry("420x300")
        win_edit.configure(bg="#2c3e50")
        win_edit.transient(self.root)
        win_edit.grab_set()
        win_edit.resizable(False, False)

        # Centrado Absoluto Asíncrono
        win_edit.withdraw()
        win_edit.after(10, lambda: [
            win_edit.update_idletasks(),
            win_edit.geometry(f"420x300+{(win_edit.winfo_screenwidth()//2)-(420//2)}+{(win_edit.winfo_screenheight()//2)-(300//2)}"),
            win_edit.deiconify()
        ])

        tk.Label(win_edit, text="✏️ MODIFICAR DATOS DEL CLIENTE", font=("Segoe UI", 12, "bold"), fg="#3498db", bg="#2c3e50", pady=15).pack()

        form_f = tk.Frame(win_edit, bg="#2c3e50")
        form_f.pack(fill="x", padx=40, pady=10)

        def _campo(lbl_txt, val_init):
            f = tk.Frame(form_f, bg="#2c3e50", pady=6)
            f.pack(fill="x")
            tk.Label(f, text=lbl_txt, font=("Segoe UI", 10, "bold"), fg="#bdc3c7", bg="#2c3e50", width=18, anchor="e").pack(side="left", padx=(0, 15))
            ent = tk.Entry(f, font=("Segoe UI", 11), bg="white", fg="#2c3e50", relief="flat")
            ent.pack(side="left", fill="x", expand=True)
            ent.insert(0, val_init)
            return ent

        # --- NUEVA CORRECCIÓN QUIRÚRGICA: CAMPO HABILITADO ---
        ent_ced = _campo("Cédula de Identidad:", cedula_original) 
        ent_nom = _campo("Nombre Completo:", nombre_original)
        ent_ced.focus()

        def procesar_cambio_cliente_db():
            nueva_cedula = ent_ced.get().strip().replace(".", "").replace("-", "").upper()
            nuevo_nombre = ent_nom.get().strip().upper()
            
            if not nueva_cedula or not nuevo_nombre:
                messagebox.showwarning("Atención", "Ninguno de los campos puede estar vacío.", parent=win_edit)
                return

            # Si el usuario cambió la cédula por una que ya le pertenece a otra persona por error
            if nueva_cedula != cedula_original:
                val_duplicado = consulta("SELECT nombre FROM clientes WHERE cedula = ?", (nueva_cedula,), db_path=DB_CLIENTES)
                if val_duplicado and len(val_duplicado) > 0:
                    nombre_duplicado = val_duplicado[0][0]
                    messagebox.showerror("Cédula Duplicada", f"❌ ERROR: La cédula [{nueva_cedula}] ya le pertenece al cliente '{nombre_duplicado}'.", parent=win_edit)
                    return

            try:
                # --- PASO 1: TRANSACCIÓN RELACIONAL EN LA DB DE CLIENTES ---
                if nueva_cedula != cedula_original:
                    # 1. Recuperamos de forma segura la fecha de registro original en texto plano
                    res_fecha = consulta("SELECT fecha_registro FROM clientes WHERE cedula = ?", (cedula_original,), db_path=DB_CLIENTES)
                    
                    # CORRECCIÓN QUIRÚRGICA: Acceso seguro e indexado al valor string puro de la tupla
                    if res_fecha and len(res_fecha) > 0:
                        fecha_origen_txt = res_fecha[0][0]
                    else:
                        fecha_origen_txt = datetime.now().strftime("%Y-%m-%d") # Fallback si no existía fecha

                    # 2. Insertamos el nuevo perfil con la cédula corregida y la fecha histórica real
                    consulta("INSERT INTO clientes (cedula, nombre, fecha_registro) VALUES (?, ?, ?)", 
                             (nueva_cedula, nuevo_nombre, fecha_origen_txt), fetch=False, db_path=DB_CLIENTES)
                    
                    # 3. Purgamos el registro con la cédula obsoleta para mantener la llave primaria única
                    consulta("DELETE FROM clientes WHERE cedula = ?", (cedula_original,), fetch=False, db_path=DB_CLIENTES)
                else:
                    # Si la cédula es la misma, solo se actualiza el nombre de pila de forma regular
                    consulta("UPDATE clientes SET nombre = ? WHERE cedula = ?", (nuevo_nombre, cedula_original), fetch=False, db_path=DB_CLIENTES)

                # --- PASO 2: ACTUALIZACIÓN EN CASCADA EN CRÉDITOS ---
                if nueva_cedula != cedula_original:
                    query_creditos_viejos = "SELECT id, cliente FROM cuentas_por_cobrar WHERE cliente LIKE ?"
                    creditos_activos = consulta(query_creditos_viejos, (f"%{cedula_original}%",), db_path=DB_CREDITOS)
                    
                    if creditos_activos and len(creditos_activos) > 0:
                        for cred in creditos_activos:
                            id_credito = cred[0]
                            nuevo_label_deudor_db = f"{nuevo_nombre} (C.I: {nueva_cedula})"
                            consulta("UPDATE cuentas_por_cobrar SET cliente = ? WHERE id = ?", (nuevo_label_deudor_db, id_credito), fetch=False, db_path=DB_CREDITOS)

                messagebox.showinfo("Perfil Actualizado", f"Perfil modificado correctamente.\n\nCédula: {nueva_cedula}\nCliente: {nuevo_nombre}\n\nLos saldos y créditos en cascada fueron sincronizados.", parent=win_edit)
                win_edit.destroy()
                
                # Refresco coordinado de interfaces de fondo
                self.cargar_tabla_clientes()
                if "creditos" in self.frames: 
                    self.cargar_tabla_creditos()

            except Exception as e:
                messagebox.showerror("Error Crítico", f"Fallo de escritura en base de datos: {e}", parent=win_edit)


        btns_f = tk.Frame(win_edit, bg="#2c3e50")
        btns_f.pack(side="bottom", fill="x", padx=40, pady=20)
        btns_f.columnconfigure(0, weight=1); btns_f.columnconfigure(1, weight=1)
        estilo_b = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}

        tk.Button(btns_f, text="💾 GUARDAR", bg="#27ae60", fg="white", command=procesar_cambio_cliente_db, **estilo_b).grid(row=0, column=0, padx=(0,4), sticky="nsew")
        tk.Button(btns_f, text="❌ CANCELAR", bg="#c0392b", fg="white", command=win_edit.destroy, **estilo_b).grid(row=0, column=1, padx=(4,0), sticky="nsew")
        ent_nom.bind("<Return>", lambda e: procesar_cambio_cliente_db())
        ent_ced.bind("<Return>", lambda e: procesar_cambio_cliente_db())
        ent_nom.bind("<Return>", lambda e: procesar_cambio_cliente_db())


    def eliminar_cliente_perfil(self):
        """Purga un cliente de la DB central tras pasar por una confirmación modal premium."""
        sel = self.tree_clientes.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione el cliente que desea eliminar de la lista.", parent=self.frames["clientes"])
            return

        valores = self.tree_clientes.item(sel, "values")
        cedula = valores[0]
        nombre = valores[1]

        # Cuadro de confirmación premium unificado
        win_conf = tk.Toplevel(self.frames["clientes"])
        win_conf.title("Confirmar Eliminación")
        win_conf.geometry("450x260")
        win_conf.configure(bg="#2c3e50")
        win_conf.transient(self.frames["clientes"])
        win_conf.grab_set()
        win_conf.resizable(False, False)

        win_conf.withdraw()
        win_conf.after(10, lambda: [
            win_conf.update_idletasks(),
            win_conf.geometry(f"450x260+{(win_conf.winfo_screenwidth()//2)-(450//2)}+{(win_conf.winfo_screenheight()//2)-(260//2)}"),
            win_conf.deiconify()
        ])

        self.conf_delete_cliente = False

        tk.Label(win_conf, text="⚠️ ELIMINACIÓN DE REGISTRO", font=("Segoe UI", 11, "bold"), fg="#e74c3c", bg="#2c3e50", pady=15).pack()
        alert_f = tk.Frame(win_conf, bg="#1e272e", padx=20, pady=15, highlightthickness=1, highlightbackground="#3d566e")
        alert_f.pack(fill="both", expand=True, padx=30, pady=5)

        tk.Label(alert_f, text=f"¿Eliminar permanentemente a {nombre} ({cedula}) del ERP?", font=("Segoe UI", 10, "bold"), fg="white", bg="#1e272e", wraplength=350, justify="center").pack()

        btns_c_f = tk.Frame(win_conf, bg="#2c3e50")
        btns_c_f.pack(side="bottom", fill="x", padx=30, pady=(15, 20))
        btns_c_f.columnconfigure(0, weight=1); btns_c_f.columnconfigure(1, weight=1)

        estilo_c_b = {"relief": "flat", "height": 2, "font": ("Segoe UI", 9, "bold"), "cursor": "hand2"}

        tk.Button(btns_c_f, text="SÍ, ELIMINAR", bg="#c0392b", fg="white", command=lambda: [setattr(self, 'conf_delete_cliente', True), win_conf.destroy()], **estilo_c_b).grid(row=0, column=0, padx=(0,4), sticky="nsew")
        tk.Button(btns_c_f, text="CANCELAR", bg="#7f8c8d", fg="white", command=win_conf.destroy, **estilo_c_b).grid(row=0, column=1, padx=(4,0), sticky="nsew")

        win_conf.wait_window()
        if not self.conf_delete_cliente: return

        # Ejecución del borrado físico en el archivo centralizado
        consulta("DELETE FROM clientes WHERE cedula = ?", (cedula,), fetch=False, db_path=DB_CLIENTES)
        messagebox.showinfo("Éxito", "Registro purgado. El historial de facturas se preserva por motivos contables.")
        self.cargar_tabla_clientes()






    def inyectar_500_productos():
        conn = sqlite3.connect(DB_NAME)
        cur = conn.cursor()
        
        # Limpiamos la tabla para la prueba pura
        cur.execute("DELETE FROM repuestos")
        
        piezas = ["Pistón", "Anillos", "Válvula", "Empacadura", "Carburador", "Bujía", "Filtro Aire", 
                "Filtro Aceite", "Cadena", "Piñón", "Corona", "Pastilla Freno", "Disco Freno", 
                "Caucho", "Batería", "Bombillo LED", "Guaya Freno", "Guaya Clutch", "Amortiguador", 
                "CDI", "Bobina", "Regulador", "Estopera", "Rodamiento", "Manubrio"]
        
        marcas = ["Yamaha", "Honda", "Suzuki", "Kawasaki", "Bajaj", "Empire", "Bera", "Pirelli", "Motul", "NGK"]
        
        modelos = ["GN125", "FZ16", "Pulsar 200", "KLR650", "V-Strom", "Horse", "SBR", "DT125", "CG150", "CBR600"]

        fecha_hoy = datetime.now().strftime("%Y-%m-%d")
        productos_masivos = []

        for i in range(1, 800):
            codigo = f"EXP-{str(i).zfill(4)}"
            nombre = f"{random.choice(piezas)} {random.choice(['Pro', 'Std', 'Elite', 'HD', 'Racing'])}"
            marca = random.choice(marcas)
            modelo = random.choice(modelos)
            
            # Precios aleatorios coherentes
            costo = round(random.uniform(2.0, 50.0), 2)
            venta = round(costo * random.uniform(1.4, 2.5), 2)
            stock = random.randint(0, 40)
            
            productos_masivos.append((codigo, nombre, modelo, marca, costo, venta, stock, fecha_hoy))

        cur.executemany("""
            INSERT INTO repuestos (codigo, nombre, modelo, marca, precio_costo, precio_venta, cantidad, fecha)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        """, productos_masivos)

        conn.commit()
        conn.close()
        print("--- CIRUGÍA EXITOSA: 500 PRODUCTOS INYECTADOS ---")
        
    pass # Llamada a la función para llenar la base de datos con 500 productos de prueba
        
        

# =================================================
# MAIN
# =================================================


if __name__ == "__main__":
    root = tk.Tk()
    app = ERPRepuestosApp(root)
    root.mainloop()

    
   
