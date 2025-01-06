import tkinter as tk
from tkinter import messagebox, Toplevel, Label, Button,scrolledtext
from tkinter import ttk
from datetime import datetime, time, timedelta
from tkcalendar import DateEntry
import heapq
from tkinter import Entry, StringVar


class Node:
    def __init__(self, guest_data):
        self.guest_data = guest_data
        self.next = None

class GuestLinkedList:
    def __init__(self):
        self.head = None

    def append(self, guest_data):
        new_node = Node(guest_data)
        if not self.head:
            self.head = new_node
        else:
            current = self.head
            while current.next:
                current = current.next
            current.next = new_node

    def display(self):
        current = self.head
        guests = []
        while current:
            guests.append(current.guest_data)
            current = current.next
        return guests

    def is_empty(self):
        return self.head is None

    def contains(self, reservation_id):
        current = self.head
        while current:
            if current.guest_data['reservation_id'] == reservation_id:
                return True
            current = current.next
        return False

class Event:
    def __init__(self, date, start_time, end_time, priority, event_name, venue):
        self.date = date
        self.start_time = start_time
        self.end_time = end_time
        self.priority = priority
        self.event_name = event_name
        self.venue = venue
        self.cost = self.calculate_cost()

    def calculate_cost(self):
        duration = (datetime.combine(datetime.min, self.end_time) - datetime.combine(datetime.min, self.start_time)).seconds / 3600
        return duration * 1000  #1000 rupees per hour

    def __lt__(self, other):
        if self.date == other.date:
            return self.priority > other.priority
        return self.date < other.date

class HotelManagementSystem:
    def __init__(self, root):
        self.root = root
        self.root.title("Hotel Management System")
        self.root.geometry("800x600")  
        self.root.configure(bg="#2E2E2E")  # Dark background for the entire window
        #self.setup_login_screen()


        self.reservations = {}
        self.checked_in_guests = []  # Linked list for checked-in guests
        self.check_out_history = GuestLinkedList()   # Linked list for check-out history
        self.reservation_heap = [] 
        self.event_tree = {}
        self.event_heap = []

        # Hash table for storing feedback
        self.feedback_dict = {}  # Hash table for storing reviews
        # Array for analyzing feedback scores
        self.ratings = []  # Array for storing feedback scores

        self.billing_records = {}
        self.billing_records = {}  # Store billing records
        self.setup_billing_menu()  # Call to setup the menu
       

        self.room_capacity = {
            "Single": 1,
            "Double": 2,
            "Standard Room": 2,
            "Deluxe Room": 3,
            "Junior Suite": 4,
            "Suite": 5,
            "Presidential Suite": 6
        }

        self.rooms = {
         "Single": {"capacity": 1, "availability": 10, "status": ["A"] * 10},  # A for Available
         "Double": {"capacity": 2, "availability": 10, "status": ["A"] * 10},
         "Standard Room": {"capacity": 2, "availability": 10, "status": ["A"] * 10},
         "Deluxe Room": {"capacity": 3, "availability": 10, "status": ["A"] * 10},
         "Junior Suite": {"capacity": 4, "availability": 10, "status": ["A"] * 10},
         "Suite": {"capacity": 5, "availability": 10, "status": ["A"] * 10},
         "Presidential Suite": {"capacity": 6, "availability": 10, "status": ["A"] * 10}
        }


        self.room_hash_table = {
            "Single": {},
            "Double": {},
            "Standard Room": {},
            "Deluxe Room": {},
            "Junior Suite": {},
            "Suite": {},
            "Presidential Suite": {}
        }
        
        self.reservation_counter = 1

        self.available_room_buttons = []
        self.id_proof_types = ["Passport", "Aadhar", "Driving License"]
        self.current_time = datetime.now()  
        self.logged_in_user = None

        self.setup_login_screen()

    def setup_billing_menu(self):
        for widget in self.root.winfo_children():
            widget.destroy()

        billing_frame = tk.Frame(self.root)
        billing_frame.pack(pady=50)

        tk.Label(billing_frame, text="Billing Management", font=("Helvetica", 16)).pack(pady=10)
        tk.Button(billing_frame, text="Generate Bill", command=self.generate_bill_ui).pack(pady=5)
        tk.Button(billing_frame, text="View Bill History", command=self.view_bill_history).pack(pady=5)
        tk.Button(billing_frame, text="Back to Main Menu", command=self.show_main_menu).pack(pady=10)

    def generate_bill_ui(self):
        bill_window = Toplevel(self.root)
        bill_window.title("Generate Bill")

        tk.Label(bill_window, text="Reservation ID").grid(row=0, column=0)
        reservation_id_entry = tk.Entry(bill_window)
        reservation_id_entry.grid(row=0, column=1)

        tk.Button(bill_window, text="Generate Bill", command=lambda: self.generate_bill(reservation_id_entry.get())).grid(row=1, columnspan=2, pady=10)

    def generate_bill(self, reservation_id):
        if reservation_id not in self.reservations:
            messagebox.showerror("Error", "Reservation ID not found.")
            return

        reservation = self.reservations[reservation_id]
        num_days = (datetime.strptime(reservation["check_out_date"], "%d/%m/%Y") - datetime.strptime(reservation["check_in_date"], "%d/%m/%Y")).days
        room_type = reservation["room_type"]

        bill = self.generate_reservation_bill(reservation_id, room_type, num_days)

        # Show generated bill
        bill_window = Toplevel(self.root)
        bill_window.title("Bill Generated")
        tk.Label(bill_window, text=f"Reservation ID: {bill['reservation_id']}").pack()
        tk.Label(bill_window, text=f"Room Type: {bill['room_type']}").pack()
        tk.Label(bill_window, text=f"Number of Days: {bill['num_days']}").pack()
        tk.Label(bill_window, text=f"Total Cost: ${bill['total_cost']}").pack()
        
    # Payment options
        tk.Label(bill_window, text="Select Payment Method:").pack(pady=10)
    
        payment_methods = ["Google Pay", "Phone Pay", "Cash", "Credit Card"]
    
        for method in payment_methods:
            tk.Button(bill_window, text=method, command=lambda m=method: self.mark_bill_paid(bill, m, bill_window)).pack(pady=2)

    def mark_bill_paid(self, bill, payment_method, bill_window):

        bill["is_paid"] = True  # Update the payment status
        bill["payment_method"] = payment_method  # Track payment method

        self.billing_history.append({
            "reservation_id": bill["reservation_id"],
            "room_type": bill["room_type"],
            "num_days": bill["num_days"],
            "total_cost": bill["total_cost"],
            "payment_method": payment_method,
            "status": "Paid"
        })

        # Show confirmation message
        messagebox.showinfo("Payment Successful", f"Payment of ${bill['total_cost']} for Reservation ID: {bill['reservation_id']} has been processed via {payment_method}.")

        # Close the bill window
        bill_window.destroy()
        self.update_unpaid_reservations()

    def update_unpaid_reservations(self):

        unpaid_reservations = []

        for reservation_id, bill in self.billing_records.items():
            if not bill["is_paid"]:
                unpaid_reservations.append({
                    "reservation_id": reservation_id,
                    "room_type": bill["room_type"],
                    "num_days": bill["num_days"],
                    "total_cost": bill["total_cost"]
                })

        # Optionally, you could show a message box with unpaid reservations
        if unpaid_reservations:
            message = "Unpaid Reservations:\n"
            for res in unpaid_reservations:
                message += (f"ID: {res['reservation_id']}, "
                            f"Room Type: {res['room_type']}, "
                            f"Total Cost: ${res['total_cost']}\n")
            messagebox.showinfo("Unpaid Reservations", message)
        else:
            messagebox.showinfo("Unpaid Reservations", "No unpaid reservations found.")



    def view_bill_history(self):
        history_window = Toplevel(self.root)
        history_window.title("Bill History")

        if not self.billing_records:
            messagebox.showinfo("Bill History", "No bills generated.")
            return

        for reservation_id, bill in self.billing_records.items():
            status = "Paid" if bill.get('is_paid', False) else "Unpaid"
            payment_method = bill.get('payment_method', 'N/A')
            tk.Label(history_window, text=f"ID: {reservation_id}, Total: ${bill['total_cost']}, Status: {status}").pack() 

    def generate_reservation_bill(self, reservation_id, room_type, num_days):
        # Pricing details for different room types
        room_rates = {
            "Single": 100,
            "Double": 150,
            "Standard Room": 200,
            "Deluxe Room": 250,
            "Junior Suite": 300,
            "Suite": 400,
            "Presidential Suite": 500
        }

        if room_type not in room_rates:
            messagebox.showerror("Error", "Invalid room type.")
            return

        total_cost = room_rates[room_type] * num_days
        billing_record = {
            "reservation_id": reservation_id,
            "room_type": room_type,
            "num_days": num_days,
            "total_cost": total_cost,
            "is_paid": False,  # New field to track payment status
            "payment_method": None  # Track payment method
        }
        self.billing_records[reservation_id] = billing_record  # Store billing record
        return billing_record


    def clear_window(self):
     for widget in self.root.winfo_children():
        widget.destroy()

    def setup_login_screen(self):
     self.clear_window()

     # Create a frame for the login screen
     login_frame = tk.Frame(self.root, bg="#2E2E2E")
     login_frame.grid(row=0, column=0, sticky="nsew")

     # Configure grid weights for centering
     self.root.grid_rowconfigure(0, weight=1)
     self.root.grid_columnconfigure(0, weight=1)

     for i in range(10):
        login_frame.grid_rowconfigure(i, weight=1)  # Equal weight for all rows
     for i in range(10):
        login_frame.grid_columnconfigure(i, weight=1)  # Equal weight for all columns

     # Title label centered at the top
     tk.Label(login_frame, text="Login", font=("Georgia", 24, "bold"), bg="#2E2E2E", fg="#F9E6A1").grid(row=0, column=0, columnspan=10, pady=(20, 10))

     # Username label and entry centered
     tk.Label(login_frame, text="Username", bg="#2E2E2E", fg="#F9E6A1").grid(row=4, column=3, sticky="e", padx=5, pady=2)
     self.username_entry = tk.Entry(login_frame)
     self.username_entry.grid(row=4, column=4, padx=5, pady=2)

     # Password label and entry centered
     tk.Label(login_frame, text="Password", bg="#2E2E2E", fg="#F9E6A1").grid(row=5, column=3, sticky="e", padx=5, pady=2)
     self.password_entry = tk.Entry(login_frame, show="*")
     self.password_entry.grid(row=5, column=4, padx=5, pady=2)

     # Login button centered
     tk.Button(login_frame, text="Login", command=self.login, bg="#C72C41", fg="#F9E6A1", width=15).grid(row=6, column=4, pady=(10, 20))


    def login(self):
      username = self.username_entry.get()
      password = self.password_entry.get()

        # Hardcoded login for demonstration
      if username == "admin" and password == "password":
            self.logged_in_user = username
            self.show_main_menu()
      else:
            messagebox.showerror("Error", "Invalid credentials. Try again!")


    def show_main_menu(self):
     self.clear_window()

     # Create a frame for the main menu
     main_frame = tk.Frame(self.root, bg="#2E2E2E")
     main_frame.grid(row=0, column=0, sticky="nsew")

     # Configure grid weights for centering
     self.root.grid_rowconfigure(0, weight=1)
     self.root.grid_columnconfigure(0, weight=1)

     for i in range(10):
        main_frame.grid_rowconfigure(i, weight=1)  # Equal weight for all rows
     for i in range(10):
        main_frame.grid_columnconfigure(i, weight=1)  # Equal weight for all columns

     # Main Menu title centered at the top
     tk.Label(main_frame, text="Main Menu", font=("Georgia", 24, "bold"), bg="#2E2E2E", fg="#F9E6A1").grid(row=0, column=0, columnspan=10, pady=(20, 10))

     button_style = {'bg': "#5C5C5C", 'fg': "#F9E6A1", 'width': 30}
     commands = [
        ("Set Current Time", self.set_current_time_ui),
        ("Reservation System", self.setup_reservation_menu),
        ("Guest Management", self.setup_guest_management_menu),
        ("Room Management", self.setup_room_management_menu),
        ("Event Management", self.setup_event_management_menu),
        ("Billing Management", self.setup_billing_menu),
     ]

     # Create buttons and center them
     for idx, (text, command) in enumerate(commands):
        tk.Button(main_frame, text=text, command=command, **button_style).grid(row=idx + 1, column=4, pady=5, sticky="ew")  # Centering in column 4

     # Smaller Logout button
     tk.Button(main_frame, text="Logout", command=self.setup_login_screen, bg="#5C5C5C", fg="#F9E6A1", width=10).grid(row=len(commands) + 1, column=4, pady=5, sticky="ew")  # Smaller width

     # Feedback and View Feedback buttons
     feedback_button = tk.Button(main_frame, text="Feedback", command=self.add_feedback, **button_style)
     feedback_button.grid(row=len(commands) + 2, column=4, pady=5, sticky="ew")  # Centered

     view_feedback_button = tk.Button(main_frame, text="View Feedback", command=self.view_feedback, **button_style)
     view_feedback_button.grid(row=len(commands) + 3, column=4, pady=5, sticky="ew")  # Centered

    def setup_event_management_menu(self):
        for widget in self.root.winfo_children():
            widget.destroy()

        event_frame = tk.Frame(self.root)
        event_frame.pack(pady=50)

        tk.Label(event_frame, text="Event Management System", font=("Helvetica", 16)).pack(pady=10)

        tk.Button(event_frame, text="Add Event", command=self.add_event_ui).pack(pady=5)
        tk.Button(event_frame, text="Modify Event", command=self.modify_event_ui).pack(pady=5)
        tk.Button(event_frame, text="Delete Event", command=self.delete_event_ui).pack(pady=5)
        tk.Button(event_frame, text="Show Events", command=self.show_events).pack(pady=5)
        tk.Button(event_frame, text="Back to Main Menu", command=self.show_main_menu).pack(pady=10)

    def add_event_ui(self):
        event_window = Toplevel(self.root)
        event_window.title("Add Event")

        tk.Label(event_window, text="Event Date (DD/MM/YYYY)").grid(row=0, column=0)
        event_date_entry = DateEntry(event_window, date_pattern='dd/mm/yyyy')
        event_date_entry.grid(row=0, column=1)

        tk.Label(event_window, text="Start Time (HH:MM)").grid(row=1, column=0)
        start_hour = ttk.Combobox(event_window, values=[f"{h:02d}" for h in range(24)])
        start_hour.grid(row=1, column=1, sticky="ew")
        start_minute = ttk.Combobox(event_window, values=[f"{m:02d}" for m in range(60)])
        start_minute.grid(row=1, column=2, sticky="ew")

        tk.Label(event_window, text="End Time (HH:MM)").grid(row=2, column=0)
        end_hour = ttk.Combobox(event_window, values=[f"{h:02d}" for h in range(24)])
        end_hour.grid(row=2, column=1, sticky="ew")
        end_minute = ttk.Combobox(event_window, values=[f"{m:02d}" for m in range(60)])
        end_minute.grid(row=2, column=2, sticky="ew")

        tk.Label(event_window, text="Priority (1 = High, 2 = Medium, 3 = Low)").grid(row=3, column=0)
        priority_entry = tk.Entry(event_window)
        priority_entry.grid(row=3, column=1)

        tk.Label(event_window, text="Event Name").grid(row=4, column=0)
        event_name_entry = tk.Entry(event_window)
        event_name_entry.grid(row=4, column=1)

        tk.Label(event_window, text="Venue").grid(row=5, column=0)
        venue_combobox = ttk.Combobox(event_window, values=["Ballroom", "Banquet Hall", "Restaurant", "Garden", "Multipurpose Room", "Amphitheater"])
        venue_combobox.grid(row=5, column=1)

        tk.Button(event_window, text="Add Event", command=lambda: self.add_event(
            event_date_entry.get(),
            f"{start_hour.get()}:{start_minute.get()}",
            f"{end_hour.get()}:{end_minute.get()}",
            priority_entry.get(),
            event_name_entry.get(),
            venue_combobox.get()
        )).grid(row=6, columnspan=3, pady=10)


    def add_event(self, event_date, start_time_str, end_time_str, priority, event_name, venue):
        try:
            event_date = datetime.strptime(event_date, "%d/%m/%Y")
            start_time = datetime.strptime(start_time_str, "%H:%M").time()
            end_time = datetime.strptime(end_time_str, "%H:%M").time()
            priority = int(priority)

            if start_time >= end_time:
                messagebox.showerror("Error", "Start time must be earlier than end time.")
                return

            for event in self.event_heap:
                if event.date == event_date and event.venue == venue and (
                    (start_time >= event.start_time and start_time < event.end_time) or
                    (end_time > event.start_time and end_time <= event.end_time)):
                    messagebox.showerror("Error", "Event conflicts with another event in the same venue and time.")
                    return

            new_event = Event(event_date, start_time, end_time, priority, event_name, venue)

            if event_date in self.event_tree:
                self.event_tree[event_date].append(new_event)
            else:
                self.event_tree[event_date] = [new_event]

            heapq.heappush(self.event_heap, new_event)
            messagebox.showinfo("Success", f"Event added successfully. Cost: {new_event.cost:.2f} rupees.")

        except ValueError:
            messagebox.showerror("Error", "Invalid input for date, time, or priority.")

   
    def delete_event_ui(self):
        delete_window = Toplevel(self.root)
        delete_window.title("Delete Event")
        delete_window.geometry("300x150")

        self.event_name_var = StringVar()

        Label(delete_window, text="Event Name").grid(row=0, column=0, padx=10, pady=10)
        Entry(delete_window, textvariable=self.event_name_var).grid(row=0, column=1)

        Button(delete_window, text="Delete Event", command=lambda: self.delete_event(self.event_name_var.get().strip())).grid(row=1, columnspan=2, pady=10)

    def delete_event(self, event_name):
        event_name = self.event_name_var.get().strip()
        deleted = False

        # Remove from event_heap
        for event in self.event_heap:
            if event.event_name == event_name:
                self.event_heap.remove(event)
                heapq.heapify(self.event_heap)  # Re-heapify after removal
                deleted = True
                break

        # Remove from event_tree if it was found in event_heap
        if deleted:
            for date, events in list(self.event_tree.items()):  # List to avoid runtime modification issues
                self.event_tree[date] = [e for e in events if e.event_name != event_name]
                if not self.event_tree[date]:  # If no events remain for this date, remove the date entry
                    del self.event_tree[date]

            messagebox.showinfo("Success", f"Deleted event: {event_name}")
        else:
            messagebox.showerror("Error", "Event not found.")

    def modify_event_ui(self):
        modify_window = Toplevel(self.root)
        modify_window.title("Modify Event")

        tk.Label(modify_window, text="Enter Event Name to Modify").grid(row=0, column=0, padx=10, pady=10)
        event_name_entry = tk.Entry(modify_window)
        event_name_entry.grid(row=0, column=1)

        tk.Label(modify_window, text="New Date (DD/MM/YYYY)").grid(row=1, column=0)
        new_date_entry = DateEntry(modify_window, date_pattern='dd/mm/yyyy')
        new_date_entry.grid(row=1, column=1)

        tk.Label(modify_window, text="New Start Time (HH:MM)").grid(row=2, column=0)
        new_start_hour = ttk.Combobox(modify_window, values=[f"{h:02d}" for h in range(24)])
        new_start_hour.grid(row=2, column=1)
        new_start_minute = ttk.Combobox(modify_window, values=[f"{m:02d}" for m in range(60)])
        new_start_minute.grid(row=2, column=2)

        tk.Label(modify_window, text="New End Time (HH:MM)").grid(row=3, column=0)
        new_end_hour = ttk.Combobox(modify_window, values=[f"{h:02d}" for h in range(24)])
        new_end_hour.grid(row=3, column=1)
        new_end_minute = ttk.Combobox(modify_window, values=[f"{m:02d}" for m in range(60)])
        new_end_minute.grid(row=3, column=2)

        tk.Label(modify_window, text="New Priority").grid(row=4, column=0)
        new_priority_entry = tk.Entry(modify_window)
        new_priority_entry.grid(row=4, column=1)

        tk.Label(modify_window, text="New Venue").grid(row=5, column=0)
        venue_combobox = ttk.Combobox(modify_window, values=["Ballroom", "Banquet Hall", "Restaurant", "Garden", "Multipurpose Room", "Amphitheater"])
        venue_combobox.grid(row=5, column=1)

        tk.Label(modify_window, text="New Event Name").grid(row=6, column=0)
        new_event_name_entry = tk.Entry(modify_window)
        new_event_name_entry.grid(row=6, column=1)

        tk.Button(modify_window, text="Modify Event", command=lambda: self.modify_event(
            event_name_entry.get(),
            new_date_entry.get(),
            f"{new_start_hour.get()}:{new_start_minute.get()}",
            f"{new_end_hour.get()}:{new_end_minute.get()}",
            new_priority_entry.get(),
            venue_combobox.get(),
            new_event_name_entry.get()
        )).grid(row=7, columnspan=3, pady=10)


    def modify_event(self, event_name, new_date, new_start_time_str, new_end_time_str, new_priority, new_venue, new_event_name):
        for event in self.event_heap:
            if event.event_name == event_name:
                try:
                    new_date = datetime.strptime(new_date, "%d/%m/%Y")
                    new_start_time = datetime.strptime(new_start_time_str, "%H:%M").time()
                    new_end_time = datetime.strptime(new_end_time_str, "%H:%M").time()
                    new_priority = int(new_priority)

                    # Check for time and venue conflicts
                    for existing_event in self.event_heap:
                        if existing_event != event and existing_event.date == new_date and existing_event.venue == new_venue and (
                            (new_start_time >= existing_event.start_time and new_start_time < existing_event.end_time) or
                            (new_end_time > existing_event.start_time and new_end_time <= existing_event.end_time)):
                            messagebox.showerror("Error", "New time conflicts with an existing event.")
                            return

                    # Remove the event from the old date in event_tree
                    if event.date in self.event_tree:
                        self.event_tree[event.date].remove(event)
                        # Remove the date entry if there are no more events on that date
                        if not self.event_tree[event.date]:
                            del self.event_tree[event.date]

                    # Update event details
                    event.date = new_date
                    event.start_time = new_start_time
                    event.end_time = new_end_time
                    event.priority = new_priority
                    event.venue = new_venue
                    event.event_name = new_event_name

                    # Recalculate and update the event cost
                    event.cost = event.calculate_cost()

                    # Add the modified event to the new date in event_tree
                    if new_date in self.event_tree:
                        self.event_tree[new_date].append(event)
                    else:
                        self.event_tree[new_date] = [event]

                    messagebox.showinfo("Success", f"Event modified successfully. New Cost: {event.cost:.2f} rupees.")
                    return
                except ValueError:
                    messagebox.showerror("Error", "Invalid input for date, time, or priority.")
                    return

        messagebox.showerror("Error", "Event not found.")


    def show_events(self):
        for widget in self.root.winfo_children():
            widget.destroy()

        events_frame = tk.Frame(self.root)
        events_frame.pack(pady=50)

        tk.Label(events_frame, text="Scheduled Events", font=("Helvetica", 16)).pack(pady=10)

        for date, events in sorted(self.event_tree.items()):
            date_label = tk.Label(events_frame, text=f"Date: {date.strftime('%d/%m/%Y')}", font=("Helvetica", 12, "bold"))
            date_label.pack()

            for event in events:
                event_frame = tk.Frame(events_frame)
                event_frame.pack(anchor="w", padx=20, pady=5)

                # Display event details
                tk.Label(event_frame, text=f"Event: {event.event_name} | Start Time: {event.start_time} | End Time: {event.end_time} | Priority: {event.priority} | Venue: {event.venue}").grid(row=0, column=0, sticky="w")

                # Add "View Bill" button for each event
                view_bill_button = tk.Button(event_frame, text="View Bill", command=lambda e=event: self.show_event_bill(e))
                view_bill_button.grid(row=0, column=1, padx=10)

        tk.Button(events_frame, text="Back to Event Management", command=self.setup_event_management_menu).pack(pady=10)


    def show_event_bill(self, event):
        # Display the bill for the selected event
        messagebox.showinfo("Event Bill", f"Event: {event.event_name}\nDate: {event.date.strftime('%d/%m/%Y')}\nCost: {event.cost:.2f} rupees")



    def add_feedback(self):
        feedback_window = tk.Toplevel(self.root)
        feedback_window.title("Guest Feedback")

        tk.Label(feedback_window, text="Enter your Reservation ID:").pack()
        reservation_id_entry = tk.Entry(feedback_window)
        reservation_id_entry.pack()

        tk.Label(feedback_window, text="Enter your feedback:").pack()
        feedback_entry = tk.Entry(feedback_window)
        feedback_entry.pack()

        tk.Label(feedback_window, text="Rate your stay:").pack()

        # Create star buttons
        stars = []
        for i in range(1, 6):
            star_button = tk.Button(feedback_window, text="☆", font=("Arial", 24), command=lambda i=i: self.set_rating(i, stars))
            star_button.pack(side=tk.LEFT)
            stars.append(star_button)

        submit_button = tk.Button(feedback_window, text="Submit",
                                  command=lambda: self.save_feedback(reservation_id_entry.get(), feedback_entry.get(), self.selected_rating))
        submit_button.pack()

        self.selected_rating = 0

    def set_rating(self, rating, stars):
        self.selected_rating = rating
        for i in range(rating):
            stars[i].config(text="★")
        for i in range(rating, 5):
            stars[i].config(text="☆")

    def save_feedback(self, reservation_id, feedback, rating):
        # Check if reservation is in check-out history
        if not self.check_out_history.contains(reservation_id):
            messagebox.showerror("Error", "Invalide reservation id OR still havent checked out.")
            return
        
        # Check if feedback already submitted for the reservation
        if reservation_id in self.feedback_dict:
            messagebox.showerror("Error", "Feedback already submitted for this Reservation ID.")
            return

        # Save feedback if valid
        self.feedback_dict[reservation_id] = feedback
        self.ratings.append(int(rating))
        messagebox.showinfo("Success", "Feedback submitted successfully!")

    def view_feedback(self):
        feedback_window = tk.Toplevel(self.root)
        feedback_window.title("Feedback History")

        # Calculate average rating
        avg_rating = sum(self.ratings) / len(self.ratings) if self.ratings else 0
        tk.Label(feedback_window, text=f"Average Rating: {avg_rating:.2f}").pack()

        # Display feedback from the hash table
        for reservation_id, feedback in self.feedback_dict.items():
            tk.Label(feedback_window, text=f"Reservation ID {reservation_id}: {feedback}").pack()


    def set_current_time_ui(self):
        time_window = Toplevel(self.root)
        time_window.title("Set Current Date")

        Label(time_window, text="Set Current Date:").grid(row=0, columnspan=2)

        Label(time_window, text="Date (DD/MM/YYYY)").grid(row=1, column=0)
        date_entry = DateEntry(time_window, date_pattern='dd/mm/yyyy')
        date_entry.grid(row=1, column=1)

        def set_date():
            try:
                date_str = date_entry.get()
                current_datetime = datetime.strptime(date_str, "%d/%m/%Y")
                self.current_time = current_datetime
                messagebox.showinfo("Success", "Current date set successfully.")
                time_window.destroy()
            except ValueError:
                messagebox.showerror("Error", "Invalid date format.")

        Button(time_window, text="Set", command=set_date).grid(row=2, columnspan=2, pady=10)
    
    def setup_room_management_menu(self):
     self.clear_window()

     # Create a frame for room management menu
     frame = tk.Frame(self.root, bg="#2E2E2E")
     frame.grid(row=0, column=0, sticky="nsew")

     # Configure grid weights for centering
     self.root.grid_rowconfigure(0, weight=1)
     self.root.grid_columnconfigure(0, weight=1)

     for i in range(10):
        frame.grid_rowconfigure(i, weight=1)  # Equal weight for all rows
     for i in range(10):
        frame.grid_columnconfigure(i, weight=1)  # Equal weight for all columns

     # Title label centered at the top
     tk.Label(frame, text="Room Management", font=("Helvetica", 24, "bold"), bg="#2E2E2E", fg="#F9E6A1").grid(row=0, column=0, columnspan=10, pady=(20, 10))

     # Button style
     button_style = {'bg': "#5C5C5C", 'fg': "#F9E6A1", 'width': 10}

     # Create buttons and center them
     tk.Button(frame, text="Book Room", command=self.open_book_room_window, **button_style).grid(row=1, column=4, pady=5, sticky="ew")
     tk.Button(frame, text="Cancel Room", command=self.cancel_room, **button_style).grid(row=2, column=4, pady=5, sticky="ew")
     tk.Button(frame, text="View Rooms", command=self.view_rooms, **button_style).grid(row=3, column=4, pady=5, sticky="ew")
     tk.Button(frame, text="Back to Main Menu", command=self.show_main_menu, **button_style).grid(row=4, column=4, pady=5, sticky="ew")


    def open_book_room_window(self):
     book_window = tk.Toplevel(self.root)
     book_window.title("Book Room")
     book_window.configure(bg="#2E2E2E")
     book_window.geometry("200x100")  # Set a fixed size for the window

     # Configure grid layout
     for i in range(5):
        book_window.grid_rowconfigure(i, weight=1)
        book_window.grid_columnconfigure(i, weight=1)

     tk.Label(book_window, text="Reservation ID", bg="#2E2E2E", fg="#F9E6A1").grid(row=0, column=1, pady=(10, 5))
     reservation_id_entry = tk.Entry(book_window, width=10)  # Smaller width
     reservation_id_entry.grid(row=0, column=2, pady=(10, 5))

     tk.Button(book_window, text="Show Available Rooms",
              command=lambda: self.fetch_dates_and_show_rooms(reservation_id_entry.get(), book_window), bg="#C72C41", fg="#F9E6A1").grid(row=1, columnspan=5, pady=(5, 10))


    
    def book_room(self, room_number, reservation_id, book_window):
     reservation = self.reservations[reservation_id]
     room_type = reservation['room_type']
    
     # Check if the room is available
     if self.rooms[room_type]['status'][room_number - 1] == "A":
        # Book the selected room
        self.rooms[room_type]['status'][room_number - 1] = "R"  # Update status to Reserved
        reservation['room_number'] = room_number  # Store room number in the reservation
        self.room_hash_table[room_type][str(room_number)] = {
            'reservation_id': reservation_id,
            'guest_name': reservation['guest_names']
        }

        messagebox.showinfo("Success", f"Room {room_number} booked successfully!")
        book_window.destroy()  # Close the booking window
     else:
        messagebox.showerror("Error", f"Room {room_number} is not available for booking.")

    def fetch_dates_and_show_rooms(self, reservation_id, book_window):
     if reservation_id not in self.reservations:
        messagebox.showerror("Error", "Reservation ID not found.")
        return

     # Get reservation details
     reservation = self.reservations[reservation_id]
     check_in_date = datetime.strptime(reservation['check_in_date'], "%d/%m/%Y").date()
     check_out_date = datetime.strptime(reservation['check_out_date'], "%d/%m/%Y").date()

     print(f"Checking availability for Room Type: {reservation['room_type']}, Check-in: {check_in_date}, Check-out: {check_out_date}")  # Debug statement

     available_rooms = self.get_available_rooms(check_in_date, check_out_date, reservation['room_type'])

     if not available_rooms:
        messagebox.showinfo("Info", "No rooms available for the selected dates.")
        return

     # Display available rooms in a new window
     available_window = tk.Toplevel(book_window)
     available_window.title("Available Rooms")

     tk.Label(available_window, text="Available Rooms:").grid(row=0, column=0, columnspan=5, pady=10)

     row_offset = 1  # To keep track of rows in the grid
     for room_number in available_rooms:
        room_button = tk.Button(available_window, text=f"Room {room_number}",
                                command=lambda room=room_number: self.book_room(room, reservation_id, book_window))
        room_button.grid(row=row_offset, column=(room_number - 1) % 5, padx=5, pady=5)

        # If we have filled 5 columns, move to the next row
        if room_number % 5 == 0:
            row_offset += 1

     tk.Button(available_window, text="Close", command=available_window.destroy).grid(row=row_offset, column=0, columnspan=5, pady=10)
     
    def cancel_room(self):
     cancel_window = tk.Toplevel(self.root)
     cancel_window.title("Cancel Room")
     cancel_window.configure(bg="#2E2E2E")
     cancel_window.geometry("200x100")  # Set a fixed size for the window

     # Configure grid layout
     for i in range(5):
        cancel_window.grid_rowconfigure(i, weight=1)
        cancel_window.grid_columnconfigure(i, weight=1)

     tk.Label(cancel_window, text="Reservation ID", bg="#2E2E2E", fg="#F9E6A1").grid(row=0, column=1, pady=(10, 5))
     reservation_id_entry = tk.Entry(cancel_window, width=9)  # Smaller width
     reservation_id_entry.grid(row=0, column=2, pady=(10, 5))

     tk.Button(cancel_window, text="Cancel Room", command=lambda: self.cancel_selected_room(reservation_id_entry.get(), cancel_window), bg="#C72C41", fg="#F9E6A1").grid(row=1, columnspan=5, pady=(5, 10))



    def cancel_selected_room(self, reservation_id, popup):
     if not reservation_id:
        messagebox.showerror("Error", "Please enter a Reservation ID.")
        return

     reservation_id_str = str(reservation_id).strip()  # Ensure it's a trimmed string

     # Check if the reservation exists in self.reservations
     if reservation_id_str in self.reservations:
        reservation_data = self.reservations[reservation_id_str]
        room_number = reservation_data["room_number"]
        room_type = reservation_data["room_type"]

        # Check if the room type exists in room_hash_table
        if room_type in self.room_hash_table:
            # Check if the reservation ID exists in room_hash_table for this room type
            if reservation_id_str in self.room_hash_table[room_type]:
                room_index = room_number - 1
                current_status = self.rooms[room_type]["status"][room_index]

                # Check if the room is occupied
                if current_status != "O":  # Assuming "O" represents occupied
                    del self.room_hash_table[room_type][reservation_id_str]  # Remove the reservation
                    self.rooms[room_type]["status"][room_index] = "A"  # Mark room as available
                    self.rooms[room_type]["availability"] += 1  # Update availability count

                    messagebox.showinfo("Success", f"Reservation ID {reservation_id_str} cancelled successfully.")
                else:
                    messagebox.showerror("Error", "Cannot cancel: Room is currently occupied.")
                popup.destroy()
                return

     messagebox.showerror("Error", f"Reservation ID {reservation_id_str} not found.")
     popup.destroy()



    def view_rooms(self): 
     # Create a new pop-up window for selecting room type
     selection_window = tk.Toplevel(self.root)
     selection_window.title("Select Room Type")
     selection_window.configure(bg="#2E2E2E")  # Set background color
      
     tk.Label(selection_window, text="Select Room Type:", bg="#2E2E2E", fg="#F9E6A1").grid(row=0, column=0)

     # Variable to store selected room type
     selected_room_type = tk.StringVar(value=list(self.rooms.keys())[0] if self.rooms else "")

     # Dropdown menu for room types
     room_type_menu = tk.OptionMenu(selection_window, selected_room_type, *self.rooms.keys(), command=lambda _: self.display_rooms(selected_room_type.get(), selection_window))
     room_type_menu.grid(row=0, column=1)

     # Initial display of rooms for the first room type
     self.display_rooms(selected_room_type.get(), selection_window)


    def display_rooms(self, room_type, popup):
    # Clear previous room status display if any
     for widget in popup.grid_slaves(row=1):
        widget.destroy()

     # Create a frame to hold room availability
     room_frame = tk.Frame(popup)
     room_frame.grid(row=1, column=0, columnspan=2, padx=10, pady=10)

     # Label for the selected room type
     tk.Label(room_frame, text=f"{room_type} Rooms:").grid(row=0, column=0, columnspan=2, pady=10)

     # Display each room's status
     for room_number in range(1, 11):  # Assuming 10 rooms per type
        room_status = self.rooms[room_type]["status"][room_number - 1]

        if room_status == "A":
            room_label = f"Room {room_number}: AL"
            room_button = tk.Label(room_frame, text=room_label, relief="solid", width=10, height=3)
            room_button.config(bg="lightgreen")  # Light green for available
        elif room_status == "O":  # For occupied rooms
            room_label = f"Room {room_number}: OCC"
            reservation_info = self.room_hash_table[room_type].get(str(room_number), None)
            room_button = tk.Button(room_frame, text=room_label, relief="solid", width=10, height=3,
                                    command=lambda info=(reservation_info['reservation_id'], reservation_info['guest_name']): self.show_reservation_details(info))
            room_button.config(bg="lightcoral")  # Red for occupied
        elif room_status == "HK":
            room_label = f"Room {room_number}: HK"
            room_button = tk.Label(room_frame, text=room_label, relief="solid", width=10, height=3)
            room_button.config(bg="lightyellow")  # Yellow for housekeeping
        elif room_status == "R":
            room_label = f"Room {room_number}: R"
            reservation_info = self.room_hash_table[room_type].get(str(room_number), None)
            room_button = tk.Button(room_frame, text=room_label, relief="solid", width=10, height=3,
                                    command=lambda info=(reservation_info['reservation_id'], reservation_info['guest_name']): self.show_reservation_details(info))
            room_button.config(bg="lightsalmon")  # Light orange for reserved

        room_button.grid(row=(room_number - 1) // 5 + 1, column=(room_number - 1) % 5, padx=5, pady=5)


    def show_reservation_details(self, reservation_info): 
     reservation_id, guest_name = reservation_info
     details_window = Toplevel(self.root)
     details_window.title("Reservation Details")

     tk.Label(details_window, text=f"Reservation ID: {reservation_id}").pack(pady=5)
     tk.Label(details_window, text=f"Guest Name: {guest_name}").pack(pady=5)

     tk.Button(details_window, text="Close", command=details_window.destroy).pack(pady=10)

    def get_available_rooms(self, check_in_date, check_out_date, room_type):
     available_rooms = []

     for room_number in range(1, 11):  # Assuming 10 rooms per type
        room_status = self.rooms[room_type]['status'][room_number - 1]

        print(f"Checking Room {room_number} - Status: {room_status}")  # Debug statement

        # Check if the room is available
        if room_status == "A":  # Available
            available_rooms.append(room_number)
        elif room_status == "R":  # Reserved, check for overlapping reservations
            reservation = self.room_hash_table.get(room_type, {}).get(str(room_number), None)
            if reservation:
                # Check for the existence of dates in the reservation
                if 'check_in_date' in reservation and 'check_out_date' in reservation:
                    res_check_in = datetime.strptime(reservation['check_in_date'], "%d/%m/%Y").date()
                    res_check_out = datetime.strptime(reservation['check_out_date'], "%d/%m/%Y").date()

                    # Check for overlapping dates
                    if not (check_out_date <= res_check_in or check_in_date >= res_check_out):
                        continue  # Overlap found, can't add to available rooms

            # If no overlap, add to available rooms
            available_rooms.append(room_number)

     print(f"Available Rooms for {room_type}: {available_rooms}")  # Debug statement
     return available_rooms

    def setup_reservation_menu(self):
     self.clear_window()

     # Create a frame for reservation menu
     frame = tk.Frame(self.root, bg="#2E2E2E")
     frame.grid(row=0, column=0, sticky="nsew")

     # Configure grid weights for centering
     self.root.grid_rowconfigure(0, weight=1)
     self.root.grid_columnconfigure(0, weight=1)

     for i in range(10):
        frame.grid_rowconfigure(i, weight=1)  # Equal weight for all rows
     for i in range(10):
        frame.grid_columnconfigure(i, weight=1)  # Equal weight for all columns

     # Title label centered at the top
     tk.Label(frame, text="Reservation System", font=("Helvetica", 24, "bold"), bg="#2E2E2E", fg="#F9E6A1").grid(row=0, column=0, columnspan=9, pady=(20, 10))

     # Button style
     button_style = {'bg': "#5C5C5C", 'fg': "#F9E6A1", 'width': 25}

     # Create buttons and center them
     tk.Button(frame, text="Add Reservation", command=self.add_reservation_ui, **button_style).grid(row=1, column=4, pady=5, sticky="ew")
     tk.Button(frame, text="Modify Reservation", command=self.modify_reservation_ui, **button_style).grid(row=2, column=4, pady=5, sticky="ew")
     tk.Button(frame, text="Cancel Reservation", command=self.cancel_reservation_ui, **button_style).grid(row=3, column=4, pady=5, sticky="ew")
     tk.Button(frame, text="Show Reservations", command=self.show_reservations, **button_style).grid(row=4, column=4, pady=5, sticky="ew")
     tk.Button(frame, text="Back to Main Menu", command=self.show_main_menu, **button_style).grid(row=5, column=4, pady=5, sticky="ew")



    def add_reservation_ui(self):
        add_window = Toplevel(self.root)
        add_window.title("Add Reservation")
        add_window.geometry("800x500")
        add_window.configure(bg="#2E2E2E")

        form_frame = tk.Frame(add_window, padx=20, pady=20, bg="#2E2E2E")
        form_frame.pack(pady=10)

        tk.Label(form_frame, text="Add a New Reservation", font=("Helvetica", 16, "bold"), bg="#2E2E2E", fg="#F9E6A1").grid(row=0, columnspan=2, pady=10)

        fields = [
            ("Reservation ID", ""),
            ("Number of Guests", ""),
            ("Guest Name and Age (name1,age1; name2,age2; ...)", ""),
            ("ID Proof Type", self.id_proof_types),
            ("ID Proof Number", ""),
            ("Check-in Date", "Date"),
            ("Check-out Date", "Date"),
            ("Room Type", list(self.room_capacity.keys()))
        ]

        entries = {}
        for i, (label_text, options) in enumerate(fields):
            tk.Label(form_frame, text=label_text, font=("Helvetica", 14), bg="#2E2E2E", fg="#F9E6A1").grid(row=i + 1, column=0, sticky=tk.W, pady=5)
        
            if label_text == "Check-in Date" or label_text == "Check-out Date":
                # Use DateEntry for date fields
                entry = DateEntry(form_frame, date_pattern='dd/mm/yyyy', font=("Helvetica", 12))
            elif options:
                # Use Combobox for fields with options
                entry = ttk.Combobox(form_frame, values=options, font=("Helvetica", 12))
            else:
                # Use Entry for all other fields
                entry = tk.Entry(form_frame, font=("Helvetica", 12))

            entry.grid(row=i + 1, column=1, pady=5)
            entries[label_text] = entry

        # Button to add reservation
        tk.Button(form_frame, text="Add Reservation", command=lambda: self.add_reservation(
            entries["Reservation ID"].get(), entries["Number of Guests"].get(),
            entries["Guest Name and Age (name1,age1; name2,age2; ...)"].get(),
            entries["ID Proof Type"].get(), entries["ID Proof Number"].get(),
            entries["Check-in Date"].get(), entries["Check-out Date"].get(),
            entries["Room Type"].get()), bg="#6A4C93", fg="white", font=("Helvetica", 12)
        ).grid(row=len(fields) + 1, columnspan=2, pady=20)

    def add_reservation(self, reservation_id, num_guests_str, guest_names_ages, id_proof_type, id_proof_number,
                        check_in_date, check_out_date, room_type):
        if reservation_id in self.reservations:
            messagebox.showerror("Error", "Reservation ID already exists.")
            return

        try:
            num_guests = int(num_guests_str)
        except ValueError:
            messagebox.showerror("Error", "Invalid number for guests.")
            return

        if num_guests > self.room_capacity[room_type]:
            messagebox.showerror("Error", f"Number of guests exceeds the capacity for {room_type}.")
            return

        try:
            check_in_datetime = datetime.strptime(check_in_date, "%d/%m/%Y")
            check_out_datetime = datetime.strptime(check_out_date, "%d/%m/%Y")
            if check_in_datetime >= check_out_datetime:
                raise ValueError("Check-in must be before check-out.")
        except ValueError as ve:
            messagebox.showerror("Error", str(ve))
            return

        guest_ages = []
        guest_names = []
        try:
            guest_entries = guest_names_ages.split(';')
            for entry in guest_entries:
                name, age = entry.split(',')
                guest_names.append(name.strip())
                guest_ages.append(int(age.strip()))
        except ValueError:
            messagebox.showerror("Error", "Invalid format for guest names and ages.")
            return

        if len(guest_names) != num_guests:
            messagebox.showerror("Error", "The number of guests does not match the provided guest names.")
            return

        if not any(age > 18 for age in guest_ages):
            messagebox.showerror("Error", "At least one guest must be over 18 years old.")
            return

        if len(id_proof_number) < 8:
            messagebox.showerror("Error", "ID Proof number must be at least 8 characters long.")
            return

        reservation_data = {
         "reservation_id": reservation_id,
         "num_guests": num_guests,
         "guest_names": guest_names,
         "guest_ages": guest_ages,
         "id_proof_type": id_proof_type,
         "id_proof_number": id_proof_number,
         "check_in_date": check_in_date,
         "check_out_date": check_out_date,
         "room_type": room_type
        }  

        self.reservations[reservation_id] = reservation_data

    # Update self.room_hash_table with the reservation
        # Store reservation in self.reservations
  
        if room_type in self.room_hash_table:
           self.room_hash_table[room_type][str(reservation_id)] = {
            "guest_name": guest_names[0],  # Assuming the first name is the main guest
            "room_number": None,            # Room number can be managed separately if needed
            "reservation_id": str(reservation_id) # Include reservation ID
        }
    
         # Add to reservation heap
        heapq.heappush(self.reservation_heap, (check_in_datetime, reservation_id))
        messagebox.showinfo("Success", "Reservation added successfully.")

    def get_next_available_room_number(self, room_type):
    # You can implement your logic here to find the next available room number
    # This is a placeholder example
      existing_rooms = self.room_hash_table[room_type]
      if not existing_rooms:
        return 1  # If no rooms are booked, return room number 1
      return max(existing_rooms.keys()) + 1  # Return the next available room number
    

    def modify_reservation_ui(self):
     modify_window = Toplevel(self.root)
     modify_window.title("Modify Reservation")

     tk.Label(modify_window, text="Reservation ID", bg="#1B1B1B", fg="#D7C6A5").grid(row=0, column=0)
     reservation_id_entry = tk.Entry(modify_window)
     reservation_id_entry.grid(row=0, column=1)

     tk.Button(modify_window, text="Fetch Reservation", command=lambda: self.fetch_reservation(reservation_id_entry.get()), bg="#6A4C93", fg="white").grid(row=1, columnspan=2, pady=10)


    def fetch_reservation(self, reservation_id):
        if reservation_id not in self.reservations:
            messagebox.showerror("Error", "Reservation ID not found.")
            return

        reservation = self.reservations[reservation_id]

        modify_window = Toplevel(self.root)
        modify_window.title("Modify Reservation")

        tk.Label(modify_window, text="Number of Guests").grid(row=0, column=0)
        num_guests_entry = tk.Entry(modify_window)
        num_guests_entry.grid(row=0, column=1)
        num_guests_entry.insert(0, reservation["num_guests"])

        tk.Label(modify_window, text="Guest Name and Age (name1,age1; name2,age2; ...)").grid(row=1, column=0)
        guest_names_ages_entry = tk.Entry(modify_window)
        guest_names_ages_entry.grid(row=1, column=1)
        guest_names_ages_entry.insert(0, '; '.join([f"{name},{age}" for name, age in zip(reservation["guest_names"], reservation["guest_ages"])]))

        tk.Label(modify_window, text="ID Proof Type").grid(row=2, column=0)
        id_proof_var = tk.StringVar(value=reservation["id_proof_type"])
        id_proof_dropdown = ttk.Combobox(modify_window, textvariable=id_proof_var, values=self.id_proof_types)
        id_proof_dropdown.grid(row=2, column=1)

        tk.Label(modify_window, text="ID Proof Number").grid(row=3, column=0)
        id_proof_entry = tk.Entry(modify_window)
        id_proof_entry.grid(row=3, column=1)
        id_proof_entry.insert(0, reservation["id_proof_number"])

        tk.Label(modify_window, text="Check-in Date").grid(row=4, column=0)
        check_in_entry = DateEntry(modify_window, date_pattern='dd/mm/yyyy')
        check_in_entry.grid(row=4, column=1)
        check_in_entry.set_date(datetime.strptime(reservation["check_in_date"], "%d/%m/%Y"))

        tk.Label(modify_window, text="Check-out Date").grid(row=5, column=0)
        check_out_entry = DateEntry(modify_window, date_pattern='dd/mm/yyyy')
        check_out_entry.grid(row=5, column=1)
        check_out_entry.set_date(datetime.strptime(reservation["check_out_date"], "%d/%m/%Y"))

        tk.Label(modify_window, text="Room Type").grid(row=6, column=0)
        room_type_var = tk.StringVar(value=reservation["room_type"])
        room_type_dropdown = ttk.Combobox(modify_window, textvariable=room_type_var, values=list(self.room_capacity.keys()))
        room_type_dropdown.grid(row=6, column=1)

        tk.Button(modify_window, text="Update Reservation", command=lambda: self.update_reservation(
            reservation_id, num_guests_entry.get(), guest_names_ages_entry.get(), id_proof_var.get(),
            id_proof_entry.get(), check_in_entry.get(), check_out_entry.get(), room_type_var.get())).grid(row=7, columnspan=2, pady=10)

    def update_reservation(self, reservation_id, num_guests_str, guest_names_ages, id_proof_type, id_proof_number,
                           check_in_date, check_out_date, room_type):
        if reservation_id not in self.reservations:
            messagebox.showerror("Error", "Reservation ID not found.")
            return

        try:
            num_guests = int(num_guests_str)
        except ValueError:
            messagebox.showerror("Error", "Invalid number for guests.")
            return

        if num_guests > self.room_capacity[room_type]:
            messagebox.showerror("Error", f"Number of guests exceeds the capacity for {room_type}.")
            return

        try:
            check_in_datetime = datetime.strptime(check_in_date, "%d/%m/%Y")
            check_out_datetime = datetime.strptime(check_out_date, "%d/%m/%Y")
            if check_in_datetime >= check_out_datetime:
                raise ValueError("Check-in must be before check-out.")
        except ValueError as ve:
            messagebox.showerror("Error", str(ve))
            return

        guest_ages = []
        guest_names = []
        try:
            guest_entries = guest_names_ages.split(';')
            for entry in guest_entries:
                name, age = entry.split(',')
                guest_names.append(name.strip())
                guest_ages.append(int(age.strip()))
        except ValueError:
            messagebox.showerror("Error", "Invalid format for guest names and ages.")
            return

        if len(guest_names) != num_guests:
            messagebox.showerror("Error", "The number of guests does not match the provided guest names.")
            return

        if not any(age > 18 for age in guest_ages):
            messagebox.showerror("Error", "At least one guest must be over 18 years old.")
            return

        if len(id_proof_number) < 8:
            messagebox.showerror("Error", "ID Proof number must be at least 8 characters long.")
            return

        reservation_data = {
            "reservation_id": reservation_id,
            "num_guests": num_guests,
            "guest_names": guest_names,
            "guest_ages": guest_ages,
            "id_proof_type": id_proof_type,
            "id_proof_number": id_proof_number,
            "check_in_date": check_in_date,
            "check_out_date": check_out_date,
            "room_type": room_type
        }

        self.reservations[reservation_id] = reservation_data
        if room_type in self.room_hash_table:
           self.room_hash_table[room_type][str(reservation_id)] = {
            "guest_name": guest_names[0],  # Assuming the first name is the main guest
            "room_number": None,            # Room number can be managed separately if needed
            "reservation_id": str(reservation_id) # Include reservation ID
        }
    
         # Add to reservation heap
        heapq.heappush(self.reservation_heap, (check_in_datetime, reservation_id))
        messagebox.showinfo("Success", "Reservation added successfully.")

    def get_next_available_room_number(self, room_type):
    # You can implement your logic here to find the next available room number
    # This is a placeholder example
      existing_rooms = self.room_hash_table[room_type]
      if not existing_rooms:
        return 1  # If no rooms are booked, return room number 1
      return max(existing_rooms.keys()) + 1  # Return the next available room number
        

    def cancel_reservation_ui(self):
     cancel_window = tk.Toplevel(self.root)
     cancel_window.title("Cancel Reservation")
     cancel_window.configure(bg="#2E2E2E")
     cancel_window.geometry("300x150")  # Increased window size for better layout

     # Configure grid layout
     for i in range(3):
        cancel_window.grid_rowconfigure(i, weight=1)
        cancel_window.grid_columnconfigure(i, weight=1)

     # Label with updated font size
     tk.Label(cancel_window, text="Reservation ID", bg="#2E2E2E", fg="#F9E6A1", font=("Helvetica", 16)).grid(row=0, column=0, pady=(20, 5))

     # Entry field with adjusted font size
     reservation_id_entry = tk.Entry(cancel_window, font=("Helvetica", 14), width=10)  # Wider entry field
     reservation_id_entry.grid(row=0, column=1, pady=(20, 5))

     # Button with increased size and adjusted font
     tk.Button(cancel_window, text="Cancel Reservation", command=lambda: self.cancel_reservation(reservation_id_entry.get()),
              bg="#C72C41", fg="#F9E6A1", font=("Helvetica", 12), width=15).grid(row=1, columnspan=2, pady=(10, 20))


    def cancel_reservation(self, reservation_id):
        if reservation_id in self.reservations:
            del self.reservations[reservation_id]
            messagebox.showinfo("Success", "Reservation cancelled successfully.")
        else:
            messagebox.showerror("Error", "Reservation ID not found.")

    def show_reservations(self):
        reservations_window = Toplevel(self.root)
        reservations_window.title("Current Reservations")
        reservations_window.geometry("800x400")

        # Check if there are any reservations
        if not self.reservations:
            messagebox.showinfo("Reservations", "No current reservations.")
            return

        # Creating a Treeview to display reservation details
        columns = ("Reservation ID", "Guests", "Guest Names", "Room Type", "Check-in Date", "Check-out Date")
        tree = ttk.Treeview(reservations_window, columns=columns, show="headings", height=15)

        # Define column headers and set their width
        for col in columns:
            tree.heading(col, text=col)
            tree.column(col, width=100)

        # Populate Treeview with reservation data
        for reservation_id, data in self.reservations.items():
            guest_names = ', '.join(data['guest_names'])  # Format guest names
            tree.insert("", "end", values=(
                reservation_id,
                data['num_guests'],
                guest_names,
                data['room_type'],
                data['check_in_date'],
                data['check_out_date']
            ))

        tree.pack(pady=20, fill="both", expand=True)

        # Scrollbar for the Treeview
        scrollbar = ttk.Scrollbar(reservations_window, orient="vertical", command=tree.yview)
        tree.configure(yscroll=scrollbar.set)
        scrollbar.pack(side="right", fill="y")

    def setup_guest_management_menu(self):
        for widget in self.root.winfo_children():
            widget.destroy()

        guest_management_frame = tk.Frame(self.root)
        guest_management_frame.pack(pady=50)

        tk.Label(guest_management_frame, text="Guest Management", font=("Helvetica", 16)).pack(pady=10)

        tk.Button(guest_management_frame, text="Check In", command=self.check_in_ui).pack(pady=5)
        tk.Button(guest_management_frame, text="Check Out", command=self.check_out_ui).pack(pady=5)
        tk.Button(guest_management_frame, text="View Check-in History", command=self.view_check_in_history).pack(pady=5)
        tk.Button(guest_management_frame, text="View Check-out History", command=self.view_check_out_history).pack(pady=5)
        tk.Button(guest_management_frame, text="Back to Main Menu", command=self.show_main_menu).pack(pady=10)

    def check_in_ui(self):
        check_in_window = Toplevel(self.root)
        check_in_window.title("Check In")

        tk.Label(check_in_window, text="Reservation ID").grid(row=0, column=0)
        reservation_id_entry = tk.Entry(check_in_window)
        reservation_id_entry.grid(row=0, column=1)

        # Pass both reservation ID and the check-in window
        tk.Button(check_in_window, text="Check In", 
                  command=lambda: self.check_in(reservation_id_entry.get(), check_in_window)).grid(row=1, columnspan=2, pady=10)

    def check_in(self, reservation_id, check_in_window):
     if reservation_id in self.reservations:
        reservation = self.reservations[reservation_id]
        check_in_date = datetime.strptime(reservation['check_in_date'], "%d/%m/%Y").date()
        check_out_date = datetime.strptime(reservation['check_out_date'], "%d/%m/%Y").date()
        room_number = reservation['room_number']
        room_type = reservation['room_type']

        if self.current_time.date() < check_in_date:
            messagebox.showerror("Error", "Cannot check in before the reservation date.")
            return
        elif self.current_time.date() >= check_out_date:
            messagebox.showerror("Error", "Cannot check in after the reservation date.")
            return

        # Update the room status to Occupied
        self.rooms[room_type]["status"][room_number - 1] = "O"  # Mark room as occupied

        # Store check-in history with all necessary details
        self.checked_in_guests.append({
            'reservation_id': reservation_id,
            'room_type': room_type,
            'num_guests': reservation['num_guests'],
            'check_out_date': reservation['check_out_date'],  # Include check-out date
            'room_number': room_number  # Include room number
        })

        # Remove the reservation after checking in
        del self.reservations[reservation_id]

        messagebox.showinfo("Success", "Checked in successfully.")
        check_in_window.destroy()  # Close the check-in window
     else:
        messagebox.showerror("Error", "Reservation ID not found.")


    def check_out_ui(self):
        check_out_window = Toplevel(self.root)
        check_out_window.title("Check Out")

        tk.Label(check_out_window, text="Reservation ID").grid(row=0, column=0)
        reservation_id_entry = tk.Entry(check_out_window)
        reservation_id_entry.grid(row=0, column=1)

        # Pass both reservation ID and the check-out window
        tk.Button(check_out_window, text="Check Out", 
                  command=lambda: self.check_out(reservation_id_entry.get(), check_out_window)).grid(row=1, columnspan=2, pady=10)

    def check_out(self, reservation_id, check_out_window):
     checked_out_guest = None
     current_date = self.current_time.date()

     for guest in self.checked_in_guests:
        if guest['reservation_id'] == reservation_id:
            check_out_date = datetime.strptime(guest['check_out_date'], "%d/%m/%Y").date()

            if current_date > check_out_date:
                messagebox.showerror("Error", "This person has already passed the scheduled check-out date.")
                return
            
            checked_out_guest = guest
            # Update the room status to Housekeeping
            room_number = guest['room_number']  # This should now work
            room_type = guest['room_type']
            self.rooms[room_type]["status"][room_number - 1] = "HK"  # Mark room as housekeeping

            # Store check-out history
            self.check_out_history.append({
                'reservation_id': reservation_id,
                'room_type': room_type,
                'num_guests': guest['num_guests']
            })

            self.checked_in_guests.remove(guest)
            messagebox.showinfo("Success", "Checked out successfully.")
            break

     if not checked_out_guest:
        messagebox.showerror("Error", "Reservation ID not found or not checked in.")
    
     check_out_window.destroy()  # Close the check-out window


    def view_check_in_history(self):
        history_window = Toplevel(self.root)
        history_window.title("Check-in History")

        if not self.checked_in_guests:
            messagebox.showinfo("Check-in History", "No guests checked in.")
            return

        for guest in self.checked_in_guests:
            tk.Label(history_window, text=f"Reservation ID: {guest['reservation_id']}, Guests: {guest['num_guests']}, Room: {guest['room_type']}").pack()
 

    def view_check_out_history(self):
        history_window = Toplevel(self.root)
        history_window.title("Check-out History")

        current = self.check_out_history.head
        if not current:
            messagebox.showinfo("Check-out History", "No guests checked out.")
            return

        while current:
            guest = current.guest_data
            tk.Label(history_window, text=f"Reservation ID: {guest['reservation_id']}, Guests: {guest['num_guests']}, Room: {guest['room_type']}").pack()
            current = current.next

if __name__ == "__main__":
    root = tk.Tk()
    app = HotelManagementSystem(root)
    root.mainloop()
