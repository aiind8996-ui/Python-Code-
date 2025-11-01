# Python-Code-
"""
chat_app.py
Combined Chat app (single-file)
- Option 1: Desktop GUI (Tkinter) -- works with standard Python (no extra install)
- Option 2: Mobile/Preview GUI (Kivy) -- requires `pip install kivy` beforehand

Features:
- Top label "CHAT"
- Add / delete contacts (contact = name + number)
- Save contacts + per-contact chat history in local JSON (contacts.json)
- Settings: toggle notifications (auto-reply simulation)
"""

import os
import json
import sys
from datetime import datetime
from functools import partial

DB_PATH = "contacts.json"

# ---------------- DB helpers ----------------
def ensure_db(path):
    if not os.path.exists(path):
        with open(path, "w", encoding="utf-8") as f:
            json.dump({"contacts": {}, "settings": {"notifications": True}}, f, indent=2, ensure_ascii=False)

def load_db(path=DB_PATH):
    ensure_db(path)
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)

def save_db(data, path=DB_PATH):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

# ---------------- Tkinter desktop app ----------------
def run_tkinter():
    try:
        import tkinter as tk
        from tkinter import messagebox, simpledialog, filedialog
    except Exception as e:
        print("Tkinter import failed:", e)
        return

    APP_TITLE = "CHAT"

    class TkChat:
        def __init__(self, root):
            self.root = root
            root.title(APP_TITLE)
            root.geometry("800x520")
            root.minsize(600, 420)

            # load db
            self.db = load_db()
            self.db_path = DB_PATH

            # Top label
            top = tk.Label(root, text=APP_TITLE, font=("Helvetica", 20, "bold"))
            top.pack(side=tk.TOP, fill=tk.X, pady=6)

            # main layout
            main = tk.Frame(root)
            main.pack(fill=tk.BOTH, expand=True, padx=8, pady=6)

            # left: contacts
            left = tk.Frame(main, width=260)
            left.pack(side=tk.LEFT, fill=tk.Y)
            tk.Label(left, text="Contacts", anchor="w").pack(fill=tk.X)
            self.contact_listbox = tk.Listbox(left)
            self.contact_listbox.pack(fill=tk.BOTH, expand=True, pady=4)
            self.contact_listbox.bind("<<ListboxSelect>>", self.on_select)

            left_btns = tk.Frame(left)
            left_btns.pack(fill=tk.X, pady=4)
            tk.Button(left_btns, text="Add", command=self.add_contact).pack(side=tk.LEFT, padx=3)
            tk.Button(left_btns, text="Delete", command=self.delete_contact).pack(side=tk.LEFT, padx=3)
            tk.Button(left_btns, text="Settings", command=self.open_settings).pack(side=tk.LEFT, padx=3)

            # right: chat
            right = tk.Frame(main)
            right.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

            self.chat_header = tk.Label(right, text="Select a contact", font=("Helvetica", 14), anchor="w")
            self.chat_header.pack(fill=tk.X)

            self.chat_text = tk.Text(right, state=tk.DISABLED, wrap=tk.WORD)
            self.chat_text.pack(fill=tk.BOTH, expand=True, pady=6)

            entry_frame = tk.Frame(right)
            entry_frame.pack(fill=tk.X)
            self.msg_entry = tk.Entry(entry_frame)
            self.msg_entry.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=6, pady=6)
            self.msg_entry.bind("<Return>", lambda e: self.send_message())
            tk.Button(entry_frame, text="Send", command=self.send_message).pack(side=tk.LEFT, padx=4)

            self.current_number = None
            self.reload_contacts()
            self.clear_chat_area()

        # contacts UI
        def reload_contacts(self):
            self.contact_listbox.delete(0, tk.END)
            for number, info in sorted(self.db.get("contacts", {}).items(), key=lambda x: x[1].get("name","")):
                display = f"{info.get('name','-')} ({number})"
                self.contact_listbox.insert(tk.END, display)

        def add_contact(self):
            name = simpledialog.askstring("Add Contact", "Enter contact name:")
            if not name:
                return
            number = simpledialog.askstring("Add Contact", "Enter contact number:")
            if not number:
                return
            contacts = self.db.setdefault("contacts", {})
            if number in contacts:
                messagebox.showinfo("Exists", "Yeh number pehle se saved hai.")
                return
            contacts[number] = {"name": name, "chats": []}
            save_db(self.db, self.db_path)
            self.reload_contacts()

        def delete_contact(self):
            sel = self.contact_listbox.curselection()
            if not sel:
                messagebox.showwarning("Select", "Please select a contact to delete.")
                return
            idx = sel[0]
            display = self.contact_listbox.get(idx)
            num = display.split("(")[-1].strip(")")
            if messagebox.askyesno("Delete", f"Delete {display}?"):
                self.db["contacts"].pop(num, None)
                save_db(self.db, self.db_path)
                self.reload_contacts()
                self.clear_chat_area()

        def on_select(self, event):
            sel = self.contact_listbox.curselection()
            if not sel:
                return
            idx = sel[0]
            display = self.contact_listbox.get(idx)
            num = display.split("(")[-1].strip(")")
            self.select_contact(num)

        def select_contact(self, number):
            self.current_number = number
            info = self.db["contacts"].get(number, {})
            self.chat_header.config(text=f"Chat with {info.get('name','-')} ({number})")
            self.reload_chat()

        def reload_chat(self):
            self.chat_text.config(state=tk.NORMAL)
            self.chat_text.delete(1.0, tk.END)
            if not self.current_number:
                self.chat_text.insert(tk.END, "Select a contact to start chatting.\n")
            else:
                chats = self.db["contacts"][self.current_number].get("chats", [])
                for m in chats:
                    t = m.get("time", "")
                    s = m.get("sender", "")
                    txt = m.get("text", "")
                    self.chat_text.insert(tk.END, f"[{t}] {s}: {txt}\n")
            self.chat_text.config(state=tk.DISABLED)
            self.chat_text.see(tk.END)

        def send_message(self):
            if not self.current_number:
                messagebox.showwarning("No contact", "Pehle contact select karo.")
                return
            text = self.msg_entry.get().strip()
            if not text:
                return
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            msg = {"time": now, "sender": "You", "text": text}
            self.db["contacts"][self.current_number].setdefault("chats", []).append(msg)
            save_db(self.db, self.db_path)
            self.msg_entry.delete(0, tk.END)
            self.reload_chat()
            if self.db.get("settings", {}).get("notifications", True):
                # simulate short auto-reply
                self.root.after(700, self.auto_reply)

        def auto_reply(self):
            if not self.current_number:
                return
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            reply = {"time": now, "sender": "Contact", "text": "Auto-reply: message received."}
            self.db["contacts"][self.current_number].setdefault("chats", []).append(reply)
            save_db(self.db, self.db_path)
            self.reload_chat()

        def clear_chat_area(self):
            self.current_number = None
            self.chat_header.config(text="Select a contact")
            self.chat_text.config(state=tk.NORMAL)
            self.chat_text.delete(1.0, tk.END)
            self.chat_text.insert(tk.END, "Select a contact to start chatting.\n")
            self.chat_text.config(state=tk.DISABLED)

        def open_settings(self):
            s = tk.Toplevel(self.root)
            s.title("Settings")
            s.geometry("360x140")
            notif_var = tk.BooleanVar(value=self.db.get("settings", {}).get("notifications", True))
            tk.Checkbutton(s, text="Enable notifications (auto-reply)", variable=notif_var).pack(anchor="w", pady=8, padx=10)
            def save_and_close():
                self.db.setdefault("settings", {})["notifications"] = notif_var.get()
                save_db(self.db, self.db_path)
                s.destroy()
            tk.Button(s, text="Save", command=save_and_close).pack(pady=8)

    root = tk.Tk()
    app = TkChat(root)
    root.mainloop()

# ---------------- Kivy mobile/preview app ----------------
def run_kivy():
    try:
        from kivy.app import App
        from kivy.lang import Builder
        from kivy.uix.popup import Popup
        from kivy.uix.boxlayout import BoxLayout
        from kivy.uix.label import Label
        from kivy.uix.button import Button
        from kivy.uix.textinput import TextInput
    except Exception as e:
        print("Kivy is not installed or failed to import. To use mobile mode install kivy:")
        print("    pip install kivy")
        print("Falling back to Tkinter mode.")
        run_tkinter()
        return

    KV = """
BoxLayout:
    orientation: 'vertical'
    Label:
        text: 'CHAT'
        size_hint_y: None
        height: '48dp'
        font_size: '22sp'
    BoxLayout:
        orientation: 'horizontal'
        padding: 6
        spacing: 6
        BoxLayout:
            orientation: 'vertical'
            size_hint_x: 0.36
            Label:
                text: 'Contacts'
                size_hint_y: None
                height: '28dp'
            ScrollView:
                do_scroll_x: False
                GridLayout:
                    id: contacts_grid
                    cols: 1
                    size_hint_y: None
                    height: self.minimum_height
                    row_default_height: '56dp'
                    row_force_default: False
            BoxLayout:
                size_hint_y: None
                height: '40dp'
                spacing: 6
                Button:
                    text: 'Add'
                    on_release: app.open_add_contact()
                Button:
                    text: 'Settings'
                    on_release: app.open_settings()
        BoxLayout:
            orientation: 'vertical'
            Label:
                id: hdr
                text: 'Select a contact'
                size_hint_y: None
                height: '28dp'
            ScrollView:
                do_scroll_x: False
                GridLayout:
                    id: chat_grid
                    cols: 1
                    size_hint_y: None
                    height: self.minimum_height
                    row_default_height: '28dp'
                    row_force_default: False
            BoxLayout:
                size_hint_y: None
                height: '44dp'
                spacing: 6
                TextInput:
                    id: msg_input
                    multiline: False
                    on_text_validate: app.send_message()
                Button:
                    text: 'Send'
                    size_hint_x: None
                    width: '90dp'
                    on_release: app.send_message()
"""

    class KivyChatApp(App):
        def build(self):
            self.db = load_db()
            self.current = None
            self.root = Builder.load_string(KV)
            self.refresh_contacts()
            return self.root

        def refresh_contacts(self):
            grid = self.root.ids.contacts_grid
            grid.clear_widgets()
            for number, info in sorted(self.db.get("contacts", {}).items(), key=lambda x: x[1].get("name","")):
                btn = Button(text=f"{info.get('name','-')}\\n({number})", size_hint_y=None, height=56)
                btn.bind(on_release=partial(self.select_contact, number))
                grid.add_widget(btn)

        def open_add_contact(self):
            content = BoxLayout(orientation="vertical", spacing=6, padding=6)
            name = TextInput(hint_text="Name", multiline=False)
            num = TextInput(hint_text="Number", multiline=False)
            btns = BoxLayout(size_hint_y=None, height=40, spacing=6)
            save_btn = Button(text="Save")
            cancel_btn = Button(text="Cancel")
            btns.add_widget(save_btn); btns.add_widget(cancel_btn)
            content.add_widget(name); content.add_widget(num); content.add_widget(btns)
            popup = Popup(title="Add Contact", content=content, size_hint=(0.9, 0.5))
            def do_save(instance):
                n = name.text.strip()
                ph = num.text.strip()
                if not n or not ph:
                    return
                self.db.setdefault("contacts", {})[ph] = {"name": n, "chats": []}
                save_db(self.db)
                popup.dismiss()
                self.refresh_contacts()
            save_btn.bind(on_release=do_save)
            cancel_btn.bind(on_release=lambda x: popup.dismiss())
            popup.open()

        def select_contact(self, number, *l):
            self.current = number
            info = self.db["contacts"].get(number, {})
            self.root.ids.hdr.text = f"Chat with {info.get('name','-')}"
            self.refresh_chat()

        def refresh_chat(self):
            grid = self.root.ids.chat_grid
            grid.clear_widgets()
            if not self.current:
                return
            chats = self.db["contacts"][self.current].get("chats", [])
            for m in chats:
                # show simple label-like button
                grid.add_widget(Label(text=f"[{m.get('time','')}] {m.get('sender','')}: {m.get('text','')}", size_hint_y=None, height=28))

        def send_message(self, *a):
            if not self.current:
                return
            txt = self.root.ids.msg_input.text.strip()
            if not txt:
                return
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            self.db["contacts"][self.current].setdefault("chats", []).append({"time": now, "sender": "You", "text": txt})
            save_db(self.db)
            self.root.ids.msg_input.text = ""
            self.refresh_chat()
            if self.db.get("settings", {}).get("notifications", True):
                # schedule a simple popup auto-reply (not using scheduling; simulate quickly)
                popup = Popup(title="Auto-reply", content=Label(text="Auto-reply: got your message."), size_hint=(0.6, 0.3))
                popup.open()
                # add to history
                self.db["contacts"][self.current].setdefault("chats", []).append({"time": datetime.now().strftime("%Y-%m-%d %H:%M:%S"), "sender": "Contact", "text": "Auto-reply: got your message."})
                save_db(self.db)
                self.refresh_chat()

        def open_settings(self):
            notif = self.db.get("settings", {}).get("notifications", True)
            content = BoxLayout(orientation="vertical", padding=8, spacing=6)
            label = Label(text=f"Notifications (auto-reply) = {notif}")
            btns = BoxLayout(size_hint_y=None, height=40, spacing=6)
            toggle = Button(text="Toggle")
            close = Button(text="Close")
            btns.add_widget(toggle); btns.add_widget(close)
            content.add_widget(label); content.add_widget(btns)
            popup = Popup(title="Settings", content=content, size_hint=(0.8, 0.35))
            def do_toggle(instance):
                current = self.db.setdefault("settings", {}).get("notifications", True)
                self.db["settings"]["notifications"] = not current
                save_db(self.db)
                label.text = f"Notifications (auto-reply) = {not current}"
            toggle.bind(on_release=do_toggle)
            close.bind(on_release=lambda x: popup.dismiss())
            popup.open()

    KivyChatApp().run()

# ---------------- Main entry ----------------
def main():
    # ensure db exists
    ensure_db(DB_PATH)
    # If user passed mode via CLI, use it
    if len(sys.argv) > 1:
        arg = sys.argv[1].lower()
        if arg in ("tk", "tkinter", "desktop", "1"):
            run_tkinter()
            return
        if arg in ("kivy", "mobile", "2"):
            run_kivy()
            return

    # interactive choice
    print("Choose mode:")
    print("1 - Desktop (Tkinter)  (recommended)")
    print("2 - Mobile/Preview (Kivy)  (requires `pip install kivy`)")
    choice = input("Enter 1 or 2: ").strip()
    if choice == "1":
        run_tkinter()
    elif choice == "2":
        run_kivy()
    else:
        print("Invalid choice, launching Tkinter by default.")
        run_tkinter()

if __name__ == "__main__":
    main()
