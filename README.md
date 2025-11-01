"""
chat_app.py
Combined Chat App (Desktop + Mobile)
Author: GPT-5 Build
--------------------------------------------------
- Option 1: Desktop GUI (Tkinter) – default
- Option 2: Mobile Preview (Kivy) – install kivy first using:
      pip install kivy
--------------------------------------------------
Features:
• Add / delete contacts
• Save chats & contacts in contacts.json
• Auto-reply simulation (toggle in Settings)
• Colorful Tkinter UI
"""

import os, sys, json
from datetime import datetime

DB_PATH = "contacts.json"

# ---------------- DB Helpers ----------------
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

# ---------------- Tkinter Desktop App ----------------
def run_tkinter():
    import tkinter as tk
    from tkinter import messagebox, simpledialog

    class ChatApp:
        def __init__(self, root):
            self.root = root
            self.root.title("CHAT 💬")
            self.root.geometry("820x540")
            self.root.configure(bg="#f4f6f9")
            self.db = load_db()
            self.current_number = None

            title = tk.Label(root, text="💬 CHAT", font=("Segoe UI", 20, "bold"), bg="#0078D7", fg="white")
            title.pack(fill=tk.X)

            main = tk.Frame(root, bg="#f4f6f9")
            main.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

            # Left Panel - Contacts
            left = tk.Frame(main, bg="#eaf0f6", bd=1, relief=tk.SOLID)
            left.pack(side=tk.LEFT, fill=tk.Y)
            tk.Label(left, text="Contacts", bg="#eaf0f6", font=("Segoe UI", 11, "bold")).pack(pady=5)
            self.contact_list = tk.Listbox(left, font=("Segoe UI", 10))
            self.contact_list.pack(fill=tk.BOTH, expand=True, padx=6, pady=4)
            self.contact_list.bind("<<ListboxSelect>>", self.on_select)

            btns = tk.Frame(left, bg="#eaf0f6")
            btns.pack(pady=4)
            tk.Button(btns, text="Add", command=self.add_contact, bg="#4CAF50", fg="white", width=8).pack(side=tk.LEFT, padx=3)
            tk.Button(btns, text="Delete", command=self.delete_contact, bg="#E53935", fg="white", width=8).pack(side=tk.LEFT, padx=3)
            tk.Button(btns, text="⚙ Settings", command=self.open_settings, bg="#03A9F4", fg="white", width=10).pack(side=tk.LEFT, padx=3)

            # Right Panel - Chat
            right = tk.Frame(main, bg="white", bd=1, relief=tk.SOLID)
            right.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=(10, 0))
            self.chat_header = tk.Label(right, text="Select a contact", font=("Segoe UI", 12, "bold"), bg="#f0f0f0", anchor="w")
            self.chat_header.pack(fill=tk.X)
            self.chat_box = tk.Text(right, state=tk.DISABLED, wrap=tk.WORD, font=("Segoe UI", 10))
            self.chat_box.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)

            entry_frame = tk.Frame(right, bg="#f9f9f9")
            entry_frame.pack(fill=tk.X, pady=4)
            self.msg_entry = tk.Entry(entry_frame, font=("Segoe UI", 10))
            self.msg_entry.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=6, pady=6)
            tk.Button(entry_frame, text="Send ➤", bg="#0078D7", fg="white", command=self.send_message).pack(side=tk.LEFT, padx=6)

            self.reload_contacts()

        def reload_contacts(self):
            self.contact_list.delete(0, tk.END)
            for num, info in sorted(self.db["contacts"].items(), key=lambda x: x[1]["name"]):
                self.contact_list.insert(tk.END, f"{info['name']} ({num})")

        def add_contact(self):
            name = simpledialog.askstring("Add Contact", "Enter Name:")
            if not name: return
            num = simpledialog.askstring("Add Contact", "Enter Number:")
            if not num: return
            if num in self.db["contacts"]:
                messagebox.showinfo("Exists", "This contact already exists!")
                return
            self.db["contacts"][num] = {"name": name, "chats": []}
            save_db(self.db)
            self.reload_contacts()

        def delete_contact(self):
            sel = self.contact_list.curselection()
            if not sel: return
            disp = self.contact_list.get(sel)
            num = disp.split("(")[-1].strip(")")
            if messagebox.askyesno("Delete", f"Delete {disp}?"):
                del self.db["contacts"][num]
                save_db(self.db)
                self.reload_contacts()
                self.clear_chat()

        def on_select(self, e):
            sel = self.contact_list.curselection()
            if not sel: return
            disp = self.contact_list.get(sel)
            num = disp.split("(")[-1].strip(")")
            self.select_contact(num)

        def select_contact(self, num):
            self.current_number = num
            name = self.db["contacts"][num]["name"]
            self.chat_header.config(text=f"Chat with {name} ({num})")
            self.reload_chat()

        def reload_chat(self):
            self.chat_box.config(state=tk.NORMAL)
            self.chat_box.delete(1.0, tk.END)
            for msg in self.db["contacts"][self.current_number]["chats"]:
                self.chat_box.insert(tk.END, f"[{msg['time']}] {msg['sender']}: {msg['text']}\n")
            self.chat_box.config(state=tk.DISABLED)
            self.chat_box.see(tk.END)

        def send_message(self):
            if not self.current_number:
                messagebox.showwarning("Select Contact", "Please select a contact first.")
                return
            text = self.msg_entry.get().strip()
            if not text: return
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            self.db["contacts"][self.current_number]["chats"].append({"time": now, "sender": "You", "text": text})
            save_db(self.db)
            self.msg_entry.delete(0, tk.END)
            self.reload_chat()
            if self.db["settings"]["notifications"]:
                self.root.after(800, self.auto_reply)

        def auto_reply(self):
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            self.db["contacts"][self.current_number]["chats"].append({"time": now, "sender": "Contact", "text": "Auto-reply: got your message!"})
            save_db(self.db)
            self.reload_chat()

        def clear_chat(self):
            self.chat_box.config(state=tk.NORMAL)
            self.chat_box.delete(1.0, tk.END)
            self.chat_box.insert(tk.END, "Select a contact to start chatting.")
            self.chat_box.config(state=tk.DISABLED)
            self.chat_header.config(text="Select a contact")

        def open_settings(self):
            s = tk.Toplevel(self.root)
            s.title("Settings ⚙")
            s.geometry("300x150")
            s.configure(bg="#f4f6f9")
            notif_var = tk.BooleanVar(value=self.db["settings"]["notifications"])
            tk.Checkbutton(s, text="Enable Auto-Reply", variable=notif_var, bg="#f4f6f9", font=("Segoe UI", 10)).pack(pady=20)
            tk.Button(s, text="Save", bg="#0078D7", fg="white", command=lambda: self.save_settings(notif_var, s)).pack()

        def save_settings(self, notif_var, s):
            self.db["settings"]["notifications"] = notif_var.get()
            save_db(self.db)
            s.destroy()

    root = tk.Tk()
    ChatApp(root)
    root.mainloop()

# ---------------- Kivy Mobile Preview ----------------
def run_kivy():
    try:
        from kivy.app import App
        from kivy.lang import Builder
        from kivy.uix.popup import Popup
        from kivy.uix.label import Label
        from kivy.uix.boxlayout import BoxLayout
        from kivy.uix.textinput import TextInput
        from kivy.uix.button import Button
    except:
        print("\nKivy not installed! Run: pip install kivy\nLaunching Tkinter instead...")
        run_tkinter()
        return

    KV = '''
BoxLayout:
    orientation: 'vertical'
    Label:
        text: '💬 CHAT'
        size_hint_y: None
        height: '48dp'
        font_size: '22sp'
        color: 1,1,1,1
        canvas.before:
            Color:
                rgba: 0.1, 0.5, 0.9, 1
            Rectangle:
                pos: self.pos
                size: self.size
    BoxLayout:
        orientation: 'horizontal'
        padding: 6
        spacing: 6
        BoxLayout:
            orientation: 'vertical'
            size_hint_x: 0.35
            Label:
                text: 'Contacts'
                size_hint_y: None
                height: '30dp'
            ScrollView:
                GridLayout:
                    id: contacts_grid
                    cols: 1
                    size_hint_y: None
                    height: self.minimum_height
        BoxLayout:
            orientation: 'vertical'
            Label:
                id: hdr
                text: 'Select a contact'
                size_hint_y: None
                height: '28dp'
            ScrollView:
                GridLayout:
                    id: chat_grid
                    cols: 1
                    size_hint_y: None
                    height: self.minimum_height
            BoxLayout:
                size_hint_y: None
                height: '46dp'
                TextInput:
                    id: msg_input
                    multiline: False
                Button:
                    text: 'Send'
                    on_release: app.send_message()
'''

    class KivyChat(App):
        def build(self):
            self.db = load_db()
            self.current = None
            self.root = Builder.load_string(KV)
            self.refresh_contacts()
            return self.root

        def refresh_contacts(self):
            grid = self.root.ids.contacts_grid
            grid.clear_widgets()
            for num, info in self.db["contacts"].items():
                btn = Button(text=f"{info['name']}\\n({num})", size_hint_y=None, height=40)
                btn.bind(on_release=lambda inst, n=num: self.select_contact(n))
                grid.add_widget(btn)

        def select_contact(self, num):
            self.current = num
            self.root.ids.hdr.text = f"Chat with {self.db['contacts'][num]['name']}"
            self.refresh_chat()

        def refresh_chat(self):
            g = self.root.ids.chat_grid
            g.clear_widgets()
            for m in self.db["contacts"][self.current]["chats"]:
                g.add_widget(Label(text=f"[{m['time']}] {m['sender']}: {m['text']}", size_hint_y=None, height=30))

        def send_message(self):
            if not self.current: return
            txt = self.root.ids.msg_input.text.strip()
            if not txt: return
            now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            self.db["contacts"][self.current]["chats"].append({"time": now, "sender": "You", "text": txt})
            save_db(self.db)
            self.root.ids.msg_input.text = ""
            self.refresh_chat()

    KivyChat().run()

# ---------------- Main Entry ----------------
def main():
    ensure_db(DB_PATH)
    if len(sys.argv) > 1 and sys.argv[1].lower() == "kivy":
        run_kivy()
    else:
        run_tkinter()

if __name__ == "__main__":
    main()
