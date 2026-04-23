import tkinter as tk
from tkinter import ttk, messagebox
import json
import os
from datetime import datetime

class ExpenseTrackerApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Expense Tracker / Трекер расходов")
        self.root.geometry("850x600")
        self.root.resizable(False, False)

        self.expenses = []
        self.json_file = "expenses.json"

        self._create_widgets()

        # Автозагрузка, если файл существует
        if os.path.exists(self.json_file):
            self.load_data(silent=True)

    def _create_widgets(self):
        # --- Блок добавления расхода ---
        add_frame = ttk.LabelFrame(self.root, text="➕ Добавить расход")
        add_frame.pack(padx=10, pady=5, fill="x")

        ttk.Label(add_frame, text="Сумма:").grid(row=0, column=0, padx=5, pady=8, sticky="w")
        self.amount_entry = ttk.Entry(add_frame, width=12)
        self.amount_entry.grid(row=0, column=1, padx=5, pady=8)

        ttk.Label(add_frame, text="Категория:").grid(row=0, column=2, padx=5, pady=8, sticky="w")
        self.category_var = tk.StringVar()
        self.category_combo = ttk.Combobox(add_frame, textvariable=self.category_var,
                                           values=["Еда", "Транспорт", "Развлечения", "Жильё", "Здоровье", "Другое"],
                                           state="readonly", width=14)
        self.category_combo.grid(row=0, column=3, padx=5, pady=8)
        self.category_combo.current(0)

        ttk.Label(add_frame, text="Дата (ГГГГ-ММ-ДД):").grid(row=0, column=4, padx=5, pady=8, sticky="w")
        self.date_entry = ttk.Entry(add_frame, width=12)
        self.date_entry.grid(row=0, column=5, padx=5, pady=8)
        self.date_entry.insert(0, datetime.now().strftime("%Y-%m-%d"))

        ttk.Button(add_frame, text="Добавить", command=self.add_expense).grid(row=0, column=6, padx=10, pady=8)

        # --- Блок фильтрации и подсчёта ---
        filter_frame = ttk.LabelFrame(self.root, text="🔍 Фильтр и подсчёт за период")
        filter_frame.pack(padx=10, pady=5, fill="x")

        ttk.Label(filter_frame, text="Категория:").grid(row=0, column=0, padx=5, pady=8, sticky="w")
        self.filter_cat_var = tk.StringVar(value="Все")
        self.filter_cat_combo = ttk.Combobox(filter_frame, textvariable=self.filter_cat_var,
                                             values=["Все"] + ["Еда", "Транспорт", "Развлечения", "Жильё", "Здоровье", "Другое"],
                                             state="readonly", width=12)
        self.filter_cat_combo.grid(row=0, column=1, padx=5, pady=8)

        ttk.Label(filter_frame, text="С:").grid(row=0, column=2, padx=5, pady=8, sticky="w")
        self.start_date_entry = ttk.Entry(filter_frame, width=11)
        self.start_date_entry.grid(row=0, column=3, padx=5, pady=8)

        ttk.Label(filter_frame, text="По:").grid(row=0, column=4, padx=5, pady=8, sticky="w")
        self.end_date_entry = ttk.Entry(filter_frame, width=11)
        self.end_date_entry.grid(row=0, column=5, padx=5, pady=8)

        ttk.Button(filter_frame, text="Применить", command=self.apply_filter).grid(row=0, column=6, padx=5, pady=8)
        
        self.total_label = ttk.Label(filter_frame, text="💰 Итого: 0.00", font=("Segoe UI", 11, "bold"), foreground="#2e7d32")
        self.total_label.grid(row=0, column=7, padx=15, pady=8)

        # --- Таблица ---
        table_frame = ttk.Frame(self.root)
        table_frame.pack(padx=10, pady=5, fill="both", expand=True)

        columns = ("amount", "category", "date")
        self.tree = ttk.Treeview(table_frame, columns=columns, show="headings", height=15)
        self.tree.heading("amount", text="Сумма (₽)")
        self.tree.heading("category", text="Категория")
        self.tree.heading("date", text="Дата")
        self.tree.column("amount", width=120, anchor="center")
        self.tree.column("category", width=180, anchor="center")
        self.tree.column("date", width=120, anchor="center")

        scrollbar = ttk.Scrollbar(table_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # --- Кнопки сохранения/загрузки ---
        btn_frame = ttk.Frame(self.root)
        btn_frame.pack(padx=10, pady=5, fill="x", side="bottom")
        ttk.Button(btn_frame, text="💾 Сохранить в JSON", command=self.save_data).pack(side="left", padx=5, expand=True)
        ttk.Button(btn_frame, text="📂 Загрузить из JSON", command=self.load_data).pack(side="left", padx=5, expand=True)

    # ---------------- ЛОГИКА ----------------

    def _validate_inputs(self, amount_str, date_str):
        try:
            amount = float(amount_str.replace(",", "."))
            if amount <= 0:
                messagebox.showerror("Ошибка ввода", "Сумма должна быть строго больше 0.")
                return None
        except ValueError:
            messagebox.showerror("Ошибка ввода", "Неверный формат суммы. Введите число.")
            return None

        try:
            datetime.strptime(date_str, "%Y-%m-%d")
        except ValueError:
            messagebox.showerror("Ошибка ввода", "Неверный формат даты. Используйте ГГГГ-ММ-ДД.")
            return None

        return amount

    def add_expense(self):
        amount = self._validate_inputs(self.amount_entry.get(), self.date_entry.get())
        if amount is None:
            return

        expense = {
            "amount": amount,
            "category": self.category_var.get(),
            "date": self.date_entry.get()
        }
        self.expenses.append(expense)
        self.apply_filter()  # Обновляем таблицу и сумму
        self.amount_entry.delete(0, tk.END)
        self.date_entry.delete(0, tk.END)
        self.date_entry.insert(0, datetime.now().strftime("%Y-%m-%d"))
        messagebox.showinfo("Успех", "Расход успешно добавлен!")

    def apply_filter(self):
        filtered = self.expenses.copy()
        cat = self.filter_cat_var.get()
        start_str = self.start_date_entry.get().strip()
        end_str = self.end_date_entry.get().strip()

        if cat != "Все":
            filtered = [e for e in filtered if e["category"] == cat]

        if start_str:
            try:
                start_date = datetime.strptime(start_str, "%Y-%m-%d")
                filtered = [e for e in filtered if datetime.strptime(e["date"], "%Y-%m-%d") >= start_date]
            except ValueError:
                messagebox.showwarning("Внимание", "Неверный формат даты начала.")
                return

        if end_str:
            try:
                end_date = datetime.strptime(end_str, "%Y-%m-%d")
                filtered = [e for e in filtered if datetime.strptime(e["date"], "%Y-%m-%d") <= end_date]
            except ValueError:
                messagebox.showwarning("Внимание", "Неверный формат даты конца.")
                return

        self._update_table(filtered)
        self.calculate_total(filtered)

    def _update_table(self, data):
        for item in self.tree.get_children():
            self.tree.delete(item)
        for exp in sorted(data, key=lambda x: x["date"], reverse=True):
            self.tree.insert("", "end", values=(f"{exp['amount']:.2f}", exp["category"], exp["date"]))

    def calculate_total(self, data=None):
        if data is None:
            data = self.expenses
        total = sum(e["amount"] for e in data)
        self.total_label.config(text=f"💰 Итого: {total:.2f} ₽")

    def save_data(self):
        try:
            with open(self.json_file, "w", encoding="utf-8") as f:
                json.dump(self.expenses, f, ensure_ascii=False, indent=4)
            messagebox.showinfo("Сохранение", f"Данные сохранены в {self.json_file}")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось сохранить: {e}")

    def load_data(self, silent=False):
        try:
            with open(self.json_file, "r", encoding="utf-8") as f:
                self.expenses = json.load(f)
            self.apply_filter()
            if not silent:
                messagebox.showinfo("Загрузка", f"Загружено {len(self.expenses)} записей.")
        except FileNotFoundError:
            if not silent:
                messagebox.showwarning("Файл не найден", "expenses.json отсутствует. Начните с добавления расходов.")
        except json.JSONDecodeError:
            if not silent:
                messagebox.showerror("Ошибка", "Файл повреждён или содержит неверный JSON.")
        except Exception as e:
            if not silent:
                messagebox.showerror("Ошибка", f"Не удалось загрузить: {e}")

if __name__ == "__main__":
    root = tk.Tk()
    app = ExpenseTrackerApp(root)
    root.mainloop()