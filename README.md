import yfinance as yf
import pandas as pd
import ta  # Librería de análisis técnico
import tkinter as tk
from tkinter import ttk
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import matplotlib.pyplot as plt

# 📌 Lista de acciones a analizar
stocks = [
    # Empresas de tecnología
    "AAPL", "TSLA", "GOOGL", "AMZN", "NVDA", "MSFT", "FB", "INTC", "AMD", "IBM",
    "ORCL", "CSCO", "ADBE", "NFLX", "CRM", "TXN", "QCOM", "AVGO", "MU", "HPQ",
    
    # Empresas de materias primas
    "BHP", "RIO", "VALE", "FCX", "NEM", "GOLD", "AA", "SCCO", "CLF", "X",
    
    # Divisas
    "EURUSD=X", "JPY=X", "GBPUSD=X", "AUDUSD=X", "USDCAD=X", "USDCHF=X", "NZDUSD=X", "USDCNY=X", "USDHKD=X", "USDSGD=X",
    "USDINR=X", "USDMXN=X", "USDZAR=X", "USDTRY=X", "USDBRL=X", "USDRUB=X", "USDKRW=X", "USDPHP=X", "USDIDR=X", "USDTHB=X"
]

# 🚀 Descargar datos de las últimas 3 meses
def get_stock_data(symbol):
    data = yf.download(symbol, period="3mo", interval="1d", auto_adjust=True)
    return data

# 📈 Función para calcular indicadores técnicos
def analyze_stock(symbol):
    df = get_stock_data(symbol)
    
    if df.empty:
        print(f"⚠️ No hay datos para {symbol}. Saltando...")
        return None  # Si no hay datos, saltamos esta acción
    
    # 📌 Asegurar que la columna Close sea tipo float
    df["Close"] = df["Close"].astype(float)

    # 📌 Extraer la columna Close como una Serie 1-dimensional
    close_series = df["Close"].squeeze()  # Esto asegura que sea 1-dimensional

    # 📌 Calcular Media Móvil (SMA)
    df["SMA_50"] = ta.trend.sma_indicator(close_series, window=50)
    df["SMA_200"] = ta.trend.sma_indicator(close_series, window=200)

    # 📌 Calcular RSI (Relative Strength Index)
    df["RSI"] = ta.momentum.rsi(close_series, window=14)

    # 📌 Calcular MACD (Moving Average Convergence Divergence)
    macd = ta.trend.MACD(close_series)
    df["MACD"] = macd.macd()  # Línea MACD
    df["MACD_Signal"] = macd.macd_signal()  # Línea de señal
    df["MACD_Histogram"] = macd.macd_diff()  # Histograma

    # 📊 Análisis de tendencia
    last_price = df["Close"].iloc[-1].item()  # Convertir a valor escalar
    last_sma50 = df["SMA_50"].iloc[-1].item()  # Convertir a valor escalar
    last_sma200 = df["SMA_200"].iloc[-1].item()  # Convertir a valor escalar
    last_rsi = df["RSI"].iloc[-1].item()  # Convertir a valor escalar
    last_macd = df["MACD"].iloc[-1].item()  # Convertir a valor escalar
    last_macd_hist = df["MACD_Histogram"].iloc[-1].item()  # Convertir a valor escalar

    # 🚀 Reglas para clasificar la acción
    signal = "Neutral"
    
    if (last_price > last_sma50 and last_sma50 > last_sma200 and 
        last_rsi > 50 and last_macd > 0 and last_macd_hist > 0):
        signal = "📈 Alta probabilidad de SUBIR"
    elif (last_price < last_sma50 and last_sma50 < last_sma200 and 
          last_rsi < 50 and last_macd < 0 and last_macd_hist < 0):
        signal = "📉 Alta probabilidad de BAJAR"

    return {"Stock": symbol, "Último Precio": last_price, "RSI": last_rsi, "MACD": last_macd, "Tendencia": signal, "DataFrame": df}

# 🔥 Analizar todas las acciones
results = []
for stock in stocks:
    result = analyze_stock(stock)
    if result:
        results.append(result)

# 📊 Crear un DataFrame con las recomendaciones
df_results = pd.DataFrame(results)

# 🖥️ Crear la interfaz gráfica
root = tk.Tk()
root.title("Recomendaciones del Mercado")

# Crear el Treeview
tree = ttk.Treeview(root, columns=("Stock", "Último Precio", "RSI", "MACD", "Tendencia"), show="headings")
tree.heading("Stock", text="Stock")
tree.heading("Último Precio", text="Último Precio")
tree.heading("RSI", text="RSI")
tree.heading("MACD", text="MACD")
tree.heading("Tendencia", text="Tendencia")

# Insertar los datos en el Treeview
for index, row in df_results.iterrows():
    tree.insert("", tk.END, values=(row["Stock"], row["Último Precio"], row["RSI"], row["MACD"], row["Tendencia"]))

# Empaquetar el Treeview
tree.pack(expand=True, fill=tk.BOTH)

# Función para mostrar la ventana de detalles
def show_details(event):
    selected_item = tree.selection()[0]
    selected_stock = tree.item(selected_item, "values")[0]
    selected_row = df_results[df_results["Stock"] == selected_stock].iloc[0]
    
    detail_window = tk.Toplevel(root)
    detail_window.title(f"Detalles de {selected_stock}")
    
    fig, ax = plt.subplots(figsize=(6, 4))
    df = selected_row["DataFrame"]
    df["Close"].plot(ax=ax, title=selected_stock)
    ax.plot(df["SMA_50"], label="SMA 50")
    ax.plot(df["SMA_200"], label="SMA 200")
    ax.legend()
    
    # Agregar información adicional y tendencia
    ax.text(0.02, 0.95, f"Último Precio: {selected_row['Último Precio']:.2f}", transform=ax.transAxes, fontsize=10, verticalalignment='top')
    ax.text(0.02, 0.90, f"RSI: {selected_row['RSI']:.2f}", transform=ax.transAxes, fontsize=10, verticalalignment='top')
    ax.text(0.02, 0.85, f"MACD: {selected_row['MACD']:.2f}", transform=ax.transAxes, fontsize=10, verticalalignment='top')
    ax.text(0.02, 0.80, f"Tendencia: {selected_row['Tendencia']}", transform=ax.transAxes, fontsize=10, verticalalignment='top')
    
    canvas = FigureCanvasTkAgg(fig, master=detail_window)
    canvas.draw()
    canvas.get_tk_widget().pack(side=tk.TOP, fill=tk.BOTH, expand=True)

# Vincular la función show_details al evento de selección del Treeview
tree.bind("<Double-1>", show_details)

# Ejecutar la aplicación
root.mainloop()
