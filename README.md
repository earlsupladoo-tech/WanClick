# WanClick
main.py
import datetime
import json
import os

from kivy.app import App
from kivy.graphics import Color, Rectangle
from kivy.metrics import dp, sp
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.button import Button
from kivy.uix.dropdown import DropDown
from kivy.uix.label import Label
from kivy.uix.popup import Popup
from kivy.uix.scrollview import ScrollView
from kivy.uix.textinput import TextInput

DATA_FILE = "ledger_data.json"
BG_FILE = "background.png"


class ReceiptLedgerApp(App):

    def build(self):
        self.title = "Payment & Receipt Tracker (PHP)"
        self.records = self.load_records()
        self.current_filter = "All"

        self.root = BoxLayout(
            orientation="vertical", padding=dp(12), spacing=dp(8)
        )

        with self.root.canvas.before:
            if os.path.exists(BG_FILE):
                self.bg_rect = Rectangle(
                    source=BG_FILE,
                    pos=self.root.pos,
                    size=self.root.size,
                )
            else:
                Color(0.12, 0.12, 0.15, 1)
                self.bg_rect = Rectangle(
                    pos=self.root.pos,
                    size=self.root.size,
                )
        self.root.bind(pos=self._update_bg, size=self._update_bg)

        header = Label(
            text="WanClick",
            font_size=sp(20),
            bold=True,
            size_hint_y=None,
            height=dp(40),
            color=(1, 1, 1, 1),
        )
        self.root.add_widget(header)

        input_box = BoxLayout(
            orientation="vertical",
            spacing=dp(8),
            size_hint_y=None,
            height=dp(220),
        )

        self.payer_input = TextInput(
            hint_text="Customer / Payer Name",
            multiline=False,
            font_size=sp(15),
            size_hint_y=None,
            height=dp(45),
        )
        self.amount_input = TextInput(
            hint_text="Amount (₱)",
            input_filter="float",
            multiline=False,
            font_size=sp(15),
            size_hint_y=None,
            height=dp(45),
        )
        self.desc_input = TextInput(
            hint_text="Item / Description",
            multiline=False,
            font_size=sp(15),
            size_hint_y=None,
            height=dp(45),
        )

        self.status_btn = Button(
            text="Status: Paid",
            font_size=sp(15),
            size_hint_y=None,
            height=dp(45),
            background_color=(0.2, 0.7, 0.3, 1),
        )
        self.status = "Paid"

        dropdown = DropDown()
        btn_paid = Button(
            text="Paid",
            font_size=sp(15),
            size_hint_y=None,
            height=dp(45),
            background_color=(0.2, 0.7, 0.3, 1),
        )
        btn_paid.bind(
            on_release=lambda btn: self.select_status("Paid", dropdown)
        )
        dropdown.add_widget(btn_paid)

        btn_unpaid = Button(
            text="Unpaid",
            font_size=sp(15),
            size_hint_y=None,
            height=dp(45),
            background_color=(0.8, 0.2, 0.2, 1),
        )
        btn_unpaid.bind(
            on_release=lambda btn: self.select_status("Unpaid", dropdown)
        )
        dropdown.add_widget(btn_unpaid)

        self.status_btn.bind(on_release=dropdown.open)

        input_box.add_widget(self.payer_input)
        input_box.add_widget(self.amount_input)
        input_box.add_widget(self.desc_input)
        input_box.add_widget(self.status_btn)
        self.root.add_widget(input_box)

        add_btn = Button(
            text="Save Payment Record",
            font_size=sp(16),
            bold=True,
            size_hint_y=None,
            height=dp(48),
            on_press=self.add_entry,
            background_color=(0.1, 0.5, 0.8, 1),
        )
        self.root.add_widget(add_btn)

        menu_label = Label(
            text="Filter Records:",
            font_size=sp(14),
            bold=True,
            size_hint_y=None,
            height=dp(25),
            color=(0.9, 0.9, 0.9, 1),
            halign="left",
        )
        menu_label.bind(size=menu_label.setter("text_size"))
        self.root.add_widget(menu_label)

        menu_box = BoxLayout(size_hint_y=None, height=dp(45), spacing=dp(5))

        btn_all = Button(
            text="All",
            font_size=sp(13),
            on_press=lambda x: self.apply_filter("All"),
        )
        btn_paid_filter = Button(
            text="Paid Only",
            font_size=sp(13),
            on_press=lambda x: self.apply_filter("Paid"),
            background_color=(0.2, 0.7, 0.3, 1),
        )
        btn_unpaid_filter = Button(
            text="Unpaid Only",
            font_size=sp(13),
            on_press=lambda x: self.apply_filter("Unpaid"),
            background_color=(0.8, 0.2, 0.2, 1),
        )
        btn_month_filter = Button(
            text="This Month",
            font_size=sp(13),
            on_press=lambda x: self.apply_filter("Month"),
            background_color=(0.5, 0.3, 0.7, 1),
        )

        menu_box.add_widget(btn_all)
        menu_box.add_widget(btn_paid_filter)
        menu_box.add_widget(btn_unpaid_filter)
        menu_box.add_widget(btn_month_filter)
        self.root.add_widget(menu_box)

        self.ledger_list = BoxLayout(
            orientation="vertical", spacing=dp(8), size_hint_y=None
        )
        self.ledger_list.bind(minimum_height=self.ledger_list.setter("height"))

        scroll = ScrollView(size_hint=(1, 1))
        scroll.add_widget(self.ledger_list)
        self.root.add_widget(scroll)

        self.apply_filter("All")

        return self.root

    def _update_bg(self, instance, value):
        self.bg_rect.pos = instance.pos
        self.bg_rect.size = instance.size

    def select_status(self, status, dropdown):
        self.status = status
        self.status_btn.text = f"Status: {status}"
        if status == "Paid":
            self.status_btn.background_color = (0.2, 0.7, 0.3, 1)
        else:
            self.status_btn.background_color = (0.8, 0.2, 0.2, 1)
        dropdown.dismiss()

    def add_entry(self, instance):
        name = self.payer_input.text.strip()
        amount = self.amount_input.text.strip()
        desc = self.desc_input.text.strip() or "General Payment"

        if not name or not amount:
            self.show_popup("Error", "Please enter customer name and amount.")
            return

        date_str = datetime.datetime.now().strftime("%Y-%m-%d %H:%M")
        month_str = datetime.datetime.now().strftime("%Y-%m")

        entry = {
            "name": name,
            "amount": float(amount),
            "description": desc,
            "status": self.status,
            "date": date_str,
            "month": month_str,
        }

        self.records.append(entry)
        self.save_records()
        self.apply_filter(self.current_filter)

        self.payer_input.text = ""
        self.amount_input.text = ""
        self.desc_input.text = ""

    def apply_filter(self, filter_type):
        self.current_filter = filter_type
        current_month = datetime.datetime.now().strftime("%Y-%m")

        if filter_type == "Paid":
            filtered = [r for r in self.records if r["status"] == "Paid"]
            title = "Paid Customers"
        elif filter_type == "Unpaid":
            filtered = [r for r in self.records if r["status"] == "Unpaid"]
            title = "Unpaid Balance / Due"
        elif filter_type == "Month":
            filtered = [
                r for r in self.records if r.get("month") == current_month
            ]
            title = f"This Month ({current_month})"
        else:
            filtered = self.records
            title = "All Records"

        self.refresh_list(filtered, title)

    def refresh_list(self, record_set, title_prefix="All Records"):
        self.ledger_list.clear_widgets()

        total_sum = sum(r["amount"] for r in record_set)

        header_label = Label(
            text=f"--- {title_prefix} ({len(record_set)}) | Total: ₱{total_sum:,.2f} ---",
            size_hint_y=None,
            height=dp(30),
            font_size=sp(14),
            bold=True,
            color=(1, 1, 1, 1),
        )
        self.ledger_list.add_widget(header_label)

        if not record_set:
            empty_lbl = Label(
                text="No records found.",
                size_hint_y=None,
                height=dp(40),
                font_size=sp(14),
                color=(0.7, 0.7, 0.7, 1),
            )
            self.ledger_list.add_widget(empty_lbl)
            return

        for rec in reversed(record_set):
            status_color = (
                (0.2, 0.7, 0.3, 1)
                if rec["status"] == "Paid"
                else (0.8, 0.2, 0.2, 1)
            )

            card = BoxLayout(
                orientation="vertical",
                padding=dp(8),
                spacing=dp(3),
                size_hint_y=None,
                height=dp(85),
            )

            with card.canvas.before:
                Color(0.18, 0.18, 0.22, 0.9)
                Rectangle(pos=card.pos, size=card.size)
            card.bind(
                pos=lambda instance, val: self._update_card_bg(instance),
                size=lambda instance, val: self._update_card_bg(instance),
            )

            title_text = (
                f"{rec['name']} - ₱{rec['amount']:,.2f} [{rec['status']}]"
            )
            lbl_title = Label(
                text=title_text,
                bold=True,
                font_size=sp(15),
                color=status_color,
                size_hint_y=None,
                height=dp(30),
                halign="left",
                valign="middle",
            )
            lbl_title.bind(size=lbl_title.setter("text_size"))

            detail_text = f"Date: {rec['date']} | Note: {rec['description']}"
            lbl_detail = Label(
                text=detail_text,
                font_size=sp(12),
                color=(0.85, 0.85, 0.85, 1),
                size_hint_y=None,
                height=dp(25),
                halign="left",
                valign="middle",
            )
            lbl_detail.bind(size=lbl_detail.setter("text_size"))

            card.add_widget(lbl_title)
            card.add_widget(lbl_detail)
            self.ledger_list.add_widget(card)

    def _update_card_bg(self, instance):
        instance.canvas.before.clear()
        with instance.canvas.before:
            Color(0.18, 0.18, 0.22, 0.9)
            Rectangle(pos=instance.pos, size=instance.size)

    def load_records(self):
        if os.path.exists(DATA_FILE):
            with open(DATA_FILE, "r") as f:
                try:
                    return json.load(f)
                except json.JSONDecodeError:
                    return []
        return []

    def save_records(self):
        with open(DATA_FILE, "w") as f:
            json.dump(self.records, f, indent=2)

    def show_popup(self, title, message):
        popup = Popup(
            title=title,
            content=Label(text=message, font_size=sp(15)),
            size_hint=(0.85, 0.35),
        )
        popup.open()


if __name__ == "__main__":
    ReceiptLedgerApp().run()
