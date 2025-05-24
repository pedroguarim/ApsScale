import pandas as pd  # Biblioteca para manipulação de dados
import matplotlib.pyplot as plt  # Biblioteca para visualização de gráficos
from sqlalchemy import create_engine  # Biblioteca para conexão com o banco de dados SQL Server
import tkinter as tk  # Biblioteca para criar interfaces gráficas
from tkinter import ttk, messagebox, filedialog  # Componentes adicionais do Tkinter
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.lib.pagesizes import A4
from reportlab.lib import colors
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib.enums import TA_LEFT
# Classe responsável pelo cálculo de emergia

class EmergyCalculator: #calculador de emergia
    def __init__(self, connection_string):
        self.engine = create_engine(connection_string)  # Cria a conexão com o banco de dados
    
    def fetch_lci_data(self): # os dados do lci
        query = """
    SELECT 
        lci.Processo,
        lci.EntradaSaida,
        lci.Item,
        lci.Quantidade,
        lci.Unidade,
        t.Transformidade,
        t.Unidade AS UnidadeTransformidade,
        (lci.Quantidade * t.Transformidade) AS Emergy
    FROM InventarioCicloVida AS lci
    JOIN Transformidades AS t ON lci.Item = t.Material
    """
        df = pd.read_sql(query, self.engine)
        return df

    def apply_emergy_algebra(self, lci_data, transformities): #algebra de emergia - o calculo se faz aqui
        # Calcula a emergia multiplicando a quantidade do item pelo fator de transformidade correspondente
        lci_data['Emergy'] = lci_data.apply(
            lambda row: row['Quantidade'] * transformities.get(row['Item'], 1), axis=1)
        return lci_data
    def fetch_transformities(self): # as transformidades
        query = "SELECT Material, Transformidade FROM Transformidades"
        df = pd.read_sql(query, self.engine)
        return dict(zip(df['Material'], df['Transformidade'])) # retorna as transformidades


    def calculate_total_emergy(self, lci_data): # o calculador total das emegias
        return lci_data['Emergy'].sum()  # Retorna a soma total da emergia


class EmergyApp: # Classe responsável pela interface gráfica
    def __init__(self, root, calculator):
        self.calculator = calculator  # Instância do cálculo de emergia
        self.root = root  # Janela principal
        self.root.title("Sistema de Cálculo de Emergia")  # Define o título da janela
        self.root.geometry("900x600")  # Define o tamanho da janela

        self.frame = ttk.Frame(root, padding="10")
        self.frame.pack(fill=tk.BOTH, expand=True)

        ttk.Label(self.frame, text="Fluxo de Emergia", font=("Arial", 14, "bold")).pack()

        # Criando a tabela para exibição dos dados
        self.tree = ttk.Treeview(self.frame, columns=("Processo", "EntradaSaida", "Item", "Quantidade", "Unidade", "Transformidade", "UnidadeTransformidade", "Emergy"), show="headings")

        for col in self.tree['columns']:
            self.tree.heading(col, text=col)  # Define os cabeçalhos da tabela
            self.tree.column(col, width=120)  # Define a largura das colunas
        self.tree.pack(fill=tk.BOTH, expand=True)

        # Criando os botões da interface
        btn_frame = ttk.Frame(self.frame)
        btn_frame.pack(pady=10)
        
        ttk.Button(btn_frame, text="Calcular Emergia", command=self.calculate, width=20).grid(row=0, column=0, padx=5)
        ttk.Button(btn_frame, text="Visualizar Fluxo", command=self.plot_emergy_distribution, width=20).grid(row=0, column=2, padx=5)
        ttk.Button(btn_frame, text="Exportar Relatório", command=self.export_pdf, width=20).grid(row=0, column=3, padx=5)
        
        self.label_total = ttk.Label(self.frame, text="Total de Emergia: -", font=("Arial", 12, "bold"))
        self.label_total.pack(pady=5)
        
    def export_pdf(self):
        caminho = filedialog.asksaveasfilename(defaultextension=".pdf", filetypes=[("PDF files", "*.pdf")])
        if not caminho:
            return

        doc = SimpleDocTemplate(caminho, pagesize=A4)
        elementos = []

        styles = getSampleStyleSheet()
        estilo_esquerda = styles["Normal"]
        estilo_esquerda.alignment = 0  # Alinhamento da interface - parte mais chata

        dados = [[
            Paragraph("<b>Processo</b>", estilo_esquerda),
            Paragraph("<b>EntradaSaida</b>", estilo_esquerda),
            Paragraph("<b>Item</b>", estilo_esquerda),
            Paragraph("<b>Quantidade</b>", estilo_esquerda),
            Paragraph("<b>Unidade</b>", estilo_esquerda),
            Paragraph("<b>Transformidade</b>", estilo_esquerda),
            Paragraph("<b>UnidadeTransformid.</b>", estilo_esquerda),
            Paragraph("<b>Emergia</b>", estilo_esquerda)
        ]]

        total_emergy = 0

        for item in self.tree.get_children():
            valores = self.tree.item(item)['values']
            try:
                total_emergy += float(valores[-1])
            except:
                pass

            linha = []
            for i, valor in enumerate(valores):
                texto = str(valor)
                if i in [0, 1, 2]:  # Aplica Paragraph nas 3 primeiras colunas
                    linha.append(Paragraph(texto, estilo_esquerda))
                else:
                    linha.append(texto)
            dados.append(linha)

        tabela = Table(dados, colWidths=[90, 60, 90, 60, 50, 80, 70, 80])
        tabela.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.lightgrey),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.black),
            ('ALIGN', (3, 1), (-1, -1), 'CENTER'),
            ('VALIGN', (0, 0), (-1, -1), 'TOP'),
            ('INNERGRID', (0, 0), (-1, -1), 0.25, colors.black),
            ('BOX', (0, 0), (-1, -1), 0.25, colors.black),
        ]))

        elementos.append(Paragraph("<b>Relatório de Emergia</b>", styles["Heading1"]))
        elementos.append(tabela)
        elementos.append(Paragraph(f"<b>Total de Emergia:</b> {total_emergy:,.2f} sej", estilo_esquerda))

        doc.build(elementos)
        # Dicionário com as transformidades dos materiais
        # Puxa os dados de transformidade diretamente do banco
        self.transformities = self.calculator.fetch_transformities()
    

    def calculate(self):
        try:
            # Busca os dados do banco
            lci_data = self.calculator.fetch_lci_data()
            # Aplica os cálculos de emergia
            updated_lci = lci_data  # Não precisa mais aplicar cálculo em Python, pois já vem pronto do SQL, achei esse metodo mais facil e mais dificl de conter erros
            # Calcula o total de emergia
            total_emergy = self.calculator.calculate_total_emergy(updated_lci)

            # Limpa os dados existentes na tabela
            for row in self.tree.get_children():
                self.tree.delete(row)

            # Adiciona os novos dados calculados na tabela
            for _, row in updated_lci.iterrows():
                self.tree.insert("", tk.END, 
                    values=(
                    row.Processo,
                    row.EntradaSaida,
                    row.Item,
                    row.Quantidade,
                    row.Unidade,
                    row.Transformidade,
                    row.UnidadeTransformidade,
                    row.Emergy
))
            self.total_emergy = total_emergy


            # Atualiza o total de emergia na interface
            self.label_total.config(text=f"Total de Emergia: {total_emergy:.2f} ")
            self.lci_data = updated_lci  # Armazena os dados para exportação
        except Exception as e:
            messagebox.showerror("Erro", f"Erro ao calcular emergia: {e}")
            

    def plot_emergy_distribution(self):
        try:
            self.lci_data.sort_values(by="Emergy", ascending=False, inplace=True)
            plt.figure(figsize=(12, 6))
            bars = plt.bar(self.lci_data['Item'], self.lci_data['Emergy'], color='royalblue')

            plt.yscale('log')  # Aplica escala logarítmica
            plt.xlabel("Item", fontsize=12)
            plt.ylabel("Emergia (sej, escala log)", fontsize=12)
            plt.title("Distribuição de Emergia por Insumo (Escala Logarítmica)", fontsize=14)

            plt.xticks(rotation=45, ha="right", fontsize=10)
            plt.yticks(fontsize=10)

            plt.tight_layout()
            plt.show()
        except Exception as e:
            messagebox.showerror("Erro", f"Erro ao gerar gráfico: {e}")
# Configuração da conexão ao SQL Server
server = "MATHEUS_SANTANA\\SQLEXPRESS" # caso venha a testar professor utilize sua base de dados, deixarei um arquivo sql com os inserts e as bases ja criadas
database = "Dados de LCI" # nome da base
connection_string = f"mssql+pyodbc://@{server}/{database}?driver=ODBC+Driver+17+for+SQL+Server&Trusted_Connection=yes" # conexão

# Inicializa o sistema e a interface gráfica
calculator = EmergyCalculator(connection_string)
root = tk.Tk()
app = EmergyApp(root, calculator)
root.mainloop()
# fim de codigo
