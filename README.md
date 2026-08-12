import tkinter as tk
from tkinter import ttk, messagebox
import heapq
from datetime import datetime, timedelta


class EmergencyRoomSimulator:
    def __init__(self, root):
        self.root = root
        self.root.title("Emergency Room Simulator")
        self.root.geometry("1050x720")

        self.patients = []
        self.completed = []
        self.priority_queue = []
        self.current_patient = None
        self.remaining_time = 0
        self.current_time = None
        self.next_patient = 0
        self.sequence = 0
        self.running = False

        self.create_gui()

    def create_gui(self):

        tk.Label(
            self.root,
            text="EMERGENCY ROOM SIMULATOR",
            font=("Arial", 22, "bold")
        ).pack(pady=15)

        input_frame = tk.LabelFrame(
            self.root,
            text="Patient Information",
            font=("Arial", 12, "bold"),
            padx=10,
            pady=10
        )
        input_frame.pack(fill="x", padx=20, pady=5)

        tk.Label(input_frame, text="Patient Name:").grid(
            row=0, column=0, padx=5, pady=5
        )

        self.name_entry = tk.Entry(input_frame, width=20)
        self.name_entry.grid(row=0, column=1, padx=5, pady=5)

        tk.Label(input_frame, text="Priority:").grid(
            row=0, column=2, padx=5, pady=5
        )

        self.priority_combo = ttk.Combobox(
            input_frame,
            width=35,
            state="readonly",
            values=[
                "1 - Critical / Highest Priority",
                "2 - Very Urgent",
                "3 - Urgent",
                "4 - Moderate",
                "5 - Low Priority"
            ]
        )
        self.priority_combo.grid(row=0, column=3, padx=5, pady=5)
        self.priority_combo.current(0)

        tk.Label(input_frame, text="Processing Time:").grid(
            row=1, column=0, padx=5, pady=5
        )

        self.processing_entry = tk.Entry(input_frame, width=20)
        self.processing_entry.grid(row=1, column=1, padx=5, pady=5)

        tk.Label(input_frame, text="Arrival Time:").grid(
            row=1, column=2, padx=5, pady=5
        )

        self.arrival_entry = tk.Entry(input_frame, width=15)
        self.arrival_entry.grid(row=1, column=3, padx=5, pady=5)
        self.arrival_entry.insert(0, "08:00")

        tk.Button(
            input_frame,
            text="Add Patient",
            command=self.add_patient,
            width=15
        ).grid(row=0, column=4, rowspan=2, padx=15)

        priority_frame = tk.LabelFrame(
            self.root,
            text="Priority Levels",
            font=("Arial", 11, "bold"),
            padx=10,
            pady=5
        )
        priority_frame.pack(fill="x", padx=20, pady=5)

        priority_text = (
            "1 - Critical / Highest Priority     "
            "2 - Very Urgent     "
            "3 - Urgent     "
            "4 - Moderate     "
            "5 - Low Priority"
        )

        tk.Label(
            priority_frame,
            text=priority_text,
            font=("Arial", 10)
        ).pack()

        button_frame = tk.Frame(self.root)
        button_frame.pack(pady=10)

        self.start_button = tk.Button(
            button_frame,
            text="Start Simulation",
            command=self.start_simulation,
            width=18,
            bg="green",
            fg="white"
        )
        self.start_button.grid(row=0, column=0, padx=5)

        self.clear_button = tk.Button(
            button_frame,
            text="Clear All",
            command=self.clear_all,
            width=15
        )
        self.clear_button.grid(row=0, column=1, padx=5)

        tk.Button(
            button_frame,
            text="Exit",
            command=self.root.destroy,
            width=15
        ).grid(row=0, column=2, padx=5)

        main_frame = tk.Frame(self.root)
        main_frame.pack(fill="both", expand=True, padx=20, pady=5)

        table_frame = tk.LabelFrame(
            main_frame,
            text="Patient Records",
            font=("Arial", 11, "bold")
        )
        table_frame.pack(side="left", fill="both", expand=True, padx=(0, 5))

        columns = (
            "ID",
            "Name",
            "Priority",
            "Arrival",
            "Start",
            "Wait",
            "Process",
            "Finish"
        )

        self.tree = ttk.Treeview(
            table_frame,
            columns=columns,
            show="headings"
        )

        for column in columns:
            self.tree.heading(column, text=column)
            self.tree.column(column, width=80, anchor="center")

        self.tree.column("Name", width=130)

        scrollbar = ttk.Scrollbar(
            table_frame,
            orient="vertical",
            command=self.tree.yview
        )

        self.tree.configure(yscrollcommand=scrollbar.set)

        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        status_frame = tk.LabelFrame(
            main_frame,
            text="Simulation Status",
            font=("Arial", 11, "bold"),
            width=300
        )
        status_frame.pack(side="right", fill="y", padx=(5, 0))
        status_frame.pack_propagate(False)

        self.status_text = tk.Text(
            status_frame,
            font=("Consolas", 10),
            state="disabled"
        )
        self.status_text.pack(
            fill="both",
            expand=True,
            padx=5,
            pady=5
        )

        self.summary_label = tk.Label(
            self.root,
            text=(
                "Patients Served: 0 | "
                "Average Wait: 0.00 minute(s) | "
                "Average Process: 0.00 minute(s)"
            ),
            font=("Arial", 11, "bold")
        )
        self.summary_label.pack(pady=10)

        self.show_status(
            "System ready.\n\n"
            "Add patient information above.\n"
            "Then click Start Simulation."
        )

    def show_status(self, text):
        self.status_text.config(state="normal")
        self.status_text.delete("1.0", tk.END)
        self.status_text.insert(tk.END, text)
        self.status_text.config(state="disabled")

    def add_patient(self):
        if self.running:
            messagebox.showwarning(
                "Simulation Running",
                "Please wait until the simulation is finished."
            )
            return

        name = self.name_entry.get().strip()
        processing = self.processing_entry.get().strip()
        arrival = self.arrival_entry.get().strip()

        if not name:
            messagebox.showerror(
                "Error",
                "Name cannot be empty."
            )
            return

        if not all(
            character.isalpha() or character.isspace()
            for character in name
        ):
            messagebox.showerror(
                "Error",
                "Letters and spaces only."
            )
            return

        try:
            processing_time = int(processing)

            if processing_time < 1 or processing_time > 60:
                raise ValueError

        except ValueError:
            messagebox.showerror(
                "Error",
                "Processing time must be 1-60 minutes."
            )
            return

        try:
            arrival_time = datetime.strptime(
                arrival,
                "%H:%M"
            )

        except ValueError:
            messagebox.showerror(
                "Error",
                "Use a valid time such as 08:30 or 14:45."
            )
            return

        priority = int(
            self.priority_combo.get().split(" - ")[0]
        )

        number = len(self.patients) + 1

        patient = {
            "id": f"P{number}",
            "name": name,
            "priority": priority,
            "arrival": arrival_time,
            "processing": processing_time,
            "start": None,
            "finish": None,
            "waiting": None
        }

        self.patients.append(patient)

        self.tree.insert(
            "",
            "end",
            iid=patient["id"],
            values=(
                patient["id"],
                patient["name"],
                f"P{patient['priority']}",
                self.format_time(patient["arrival"]),
                "",
                "",
                f"{patient['processing']} min",
                ""
            )
        )

        self.name_entry.delete(0, tk.END)
        self.processing_entry.delete(0, tk.END)

        self.show_status(
            f"PATIENT ADDED\n"
            f"====================\n\n"
            f"Patient: {patient['id']} - {patient['name']}\n"
            f"Priority: {patient['priority']}\n"
            f"Arrival: {self.format_time(patient['arrival'])}\n"
            f"Processing: {patient['processing']} minute(s)"
        )

    def start_simulation(self):
        if not self.patients:
            messagebox.showwarning(
                "No Patients",
                "Please add at least one patient."
            )
            return

        self.running = True

        self.start_button.config(state="disabled")
        self.clear_button.config(state="disabled")

        self.patients.sort(
            key=lambda patient: patient["arrival"]
        )

        self.completed = []
        self.priority_queue = []
        self.current_patient = None
        self.remaining_time = 0
        self.next_patient = 0
        self.sequence = 0

        self.current_time = self.patients[0]["arrival"]

        for patient in self.patients:
            patient["start"] = None
            patient["finish"] = None
            patient["waiting"] = None

        self.run_simulation_step()

    def run_simulation_step(self):
        if not self.running:
            return

        while (
            self.next_patient < len(self.patients)
            and self.patients[self.next_patient]["arrival"]
            <= self.current_time
        ):
            patient = self.patients[self.next_patient]

            self.sequence += 1

            heapq.heappush(
                self.priority_queue,
                (
                    patient["priority"],
                    self.sequence,
                    patient
                )
            )

            self.next_patient += 1

        if (
            self.current_patient is not None
            and self.remaining_time <= 0
        ):
            self.current_patient["finish"] = self.current_time

            self.completed.append(
                self.current_patient
            )

            self.current_patient = None

        if (
            self.current_patient is None
            and self.priority_queue
        ):
            selected = heapq.heappop(
                self.priority_queue
            )

            patient = selected[2]

            patient["start"] = self.current_time

            patient["waiting"] = int(
                (
                    self.current_time
                    - patient["arrival"]
                ).total_seconds() / 60
            )

            self.current_patient = patient

            self.remaining_time = (
                patient["processing"]
            )

        self.update_table()
        self.update_status()

        if (
            self.next_patient >= len(self.patients)
            and not self.priority_queue
            and self.current_patient is None
        ):
            self.finish_simulation()
            return

        if self.current_patient is not None:
            self.remaining_time -= 1

        self.current_time += timedelta(
            minutes=1
        )

        self.root.after(
            300,
            self.run_simulation_step
        )

    def update_table(self):
        for patient in self.patients:

            self.tree.item(
                patient["id"],
                values=(
                    patient["id"],
                    patient["name"],
                    f"P{patient['priority']}",
                    self.format_time(
                        patient["arrival"]
                    ),
                    self.format_time(
                        patient["start"]
                    ) if patient["start"] else "",
                    f"{patient['waiting']} min"
                    if patient["waiting"] is not None
                    else "",
                    f"{patient['processing']} min",
                    self.format_time(
                        patient["finish"]
                    ) if patient["finish"] else ""
                )
            )

    def update_status(self):
        if self.priority_queue:
            queue = sorted(
                self.priority_queue,
                key=lambda item: (
                    item[0],
                    item[1]
                )
            )

            queue_text = " -> ".join(
                f"{item[2]['id']}(P{item[2]['priority']})"
                for item in queue
            )

        else:
            queue_text = "Empty"

        if self.current_patient:
            current = (
                f"{self.current_patient['id']} - "
                f"{self.current_patient['name']}"
            )
        else:
            current = "None"

        status = (
            "SIMULATION RUNNING\n"
            "========================\n\n"
            f"Current Time:\n"
            f"{self.format_time(self.current_time)}\n\n"
            f"Now Processing:\n"
            f"{current}\n\n"
        )

        if self.current_patient:
            status += (
                f"Priority: "
                f"{self.current_patient['priority']}\n"
                f"Remaining: "
                f"{self.remaining_time} minute(s)\n"
                f"Processing: "
                f"{self.current_patient['processing']} minute(s)\n\n"
            )

        status += (
            f"Waiting Queue:\n"
            f"{queue_text}\n\n"
            f"Completed: "
            f"{len(self.completed)} / "
            f"{len(self.patients)}"
        )

        self.show_status(status)

    def finish_simulation(self):
        self.running = False

        self.update_table()

        total_waiting = sum(
            patient["waiting"]
            for patient in self.completed
        )

        total_processing = sum(
            patient["processing"]
            for patient in self.completed
        )

        average_waiting = (
            total_waiting / len(self.completed)
            if self.completed else 0
        )

        average_processing = (
            total_processing / len(self.completed)
            if self.completed else 0
        )

        simulation_start = min(
            patient["arrival"]
            for patient in self.completed
        )

        simulation_finish = max(
            patient["finish"]
            for patient in self.completed
        )

        self.summary_label.config(
            text=(
                f"Patients Served: {len(self.completed)} | "
                f"Average Wait: {average_waiting:.2f} minute(s) | "
                f"Average Process: "
                f"{average_processing:.2f} minute(s)"
            )
        )

        self.show_status(
            "FINAL REPORT\n"
            "========================\n\n"
            f"Patients Served: "
            f"{len(self.completed)}\n\n"
            f"Average Wait: "
            f"{average_waiting:.2f} minute(s)\n\n"
            f"Average Process: "
            f"{average_processing:.2f} minute(s)\n\n"
            f"Simulation Start:\n"
            f"{self.format_time(simulation_start)}\n\n"
            f"Simulation End:\n"
            f"{self.format_time(simulation_finish)}"
        )

        self.start_button.config(state="normal")
        self.clear_button.config(state="normal")

        messagebox.showinfo(
            "Simulation Complete",
            "Emergency Room Simulator finished!"
        )

    def clear_all(self):
        if self.running:
            messagebox.showwarning(
                "Simulation Running",
                "Wait until the simulation finishes."
            )
            return

        self.patients.clear()
        self.completed.clear()
        self.priority_queue.clear()

        for item in self.tree.get_children():
            self.tree.delete(item)

        self.summary_label.config(
            text=(
                "Patients Served: 0 | "
                "Average Wait: 0.00 minute(s) | "
                "Average Process: 0.00 minute(s)"
            )
        )

        self.show_status(
            "System ready.\n\n"
            "Add patient information above.\n"
            "Then click Start Simulation."
        )

    @staticmethod
    def format_time(value):
        return value.strftime("%I:%M %p")


def main():
    root = tk.Tk()
    EmergencyRoomSimulator(root)
    root.mainloop()


if __name__ == "__main__":
    main()
