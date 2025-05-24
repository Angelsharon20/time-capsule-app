import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import mysql.connector
import pandas as pd

# ---------------------- Configuration ----------------------
# MySQL database connection parameters — update with your credentials
DB_CONFIG = {
    'host': 'localhost',
    'user': 'root',               # Replace with your MySQL username
    'password': 'your_password', # Replace with your MySQL password
    'database': 'time_capsule_ai'
}

# CSV filename where time capsule entries will be appended
CSV_FILE = "time_capsule_entries.csv"
# -----------------------------------------------------------

def get_db_connection():
    """
    Establishes and returns a new connection to the MySQL database using DB_CONFIG.
    """
    return mysql.connector.connect(**DB_CONFIG)

def save_capsule_to_db_and_csv(username, message, release_date):
    """
    Saves a new time capsule entry to both MySQL database and CSV file.
    
    Args:
        username (str): Name of the user creating the capsule.
        message (str): The time capsule message.
        release_date (datetime): Date and time when the message can be released.
    """
    timestamp = datetime.now()  # Current time when capsule is created
    summary = "No summary (AI not used)"  # Placeholder for future AI-generated summary

    # Save to MySQL database
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        query = """
        INSERT INTO capsules (username, message, summary, timestamp, release_date)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(query, (username, message, summary, timestamp, release_date))
        conn.commit()
        cursor.close()
        conn.close()
    except Exception as e:
        # Show error message if database operation fails
        messagebox.showerror("Database Error", str(e))
        return

    # Prepare data for CSV export
    data = {
        'Username': username,
        'Message': message,
        'Summary': summary,
        'Timestamp': timestamp,
        'Release Date': release_date
    }
    df = pd.DataFrame([data])

    # Append to CSV file; create the file if it doesn't exist yet
    try:
        with open(CSV_FILE, "x") as f:  # Create file if not exists
            df.to_csv(f, index=False)
    except FileExistsError:
        df.to_csv(CSV_FILE, mode='a', index=False, header=False)

    # Confirm success to the user
    messagebox.showinfo("Success", "Capsule saved to database and CSV!")

def retrieve_capsules():
    """
    Fetches and displays all capsules whose release date is now or has passed.
    Displays the capsules in a popup message box.
    """
    try:
        conn = get_db_connection()
        cursor = conn.cursor(dictionary=True)
        now = datetime.now()
        # Select capsules ready to be opened (release_date <= current time)
        query = "SELECT * FROM capsules WHERE release_date <= %s"
        cursor.execute(query, (now,))
        results = cursor.fetchall()
        cursor.close()
        conn.close()
        
        if results:
            msg = ""
            # Format each capsule's details for display
            for row in results:
                msg += (
                    f"From: {row['username']}\n"
                    f"Created: {row['timestamp']}\n"
                    f"Release: {row['release_date']}\n"
                    f"Message: {row['message']}\n"
                    f"{'-'*40}\n"
                )
            messagebox.showinfo("Released Capsules", msg)
        else:
            messagebox.showinfo("No Capsules", "No capsules are ready to be opened yet.")
    except Exception as e:
        # Show error if retrieval fails
        messagebox.showerror("Error", str(e))

# ---------------------- GUI Section ----------------------

def submit_data():
    """
    Handles the event when the user clicks the 'Save Capsule' button.
    Retrieves user inputs, validates the release date format,
    and saves the capsule data.
    """
    username = entry_name.get()
    message = text_message.get("1.0", tk.END).strip()
    release_date_str = entry_date.get()

    try:
        # Convert release date string to datetime object
        release_date = datetime.strptime(release_date_str, "%Y-%m-%d %H:%M:%S")
        save_capsule_to_db_and_csv(username, message, release_date)
        # Clear input fields after successful save
        entry_name.delete(0, tk.END)
        entry_date.delete(0, tk.END)
        text_message.delete("1.0", tk.END)
    except ValueError:
        # Inform user if date format is incorrect
        messagebox.showerror("Invalid Date", "Please use format: YYYY-MM-DD HH:MM:SS")

# Create main Tkinter window
window = tk.Tk()
window.title("Time Capsule")
window.geometry("500x500")

# Username input
tk.Label(window, text="Your Name:").pack()
entry_name = tk.Entry(window, width=50)
entry_name.pack()

# Message input
tk.Label(window, text="Your Message:").pack()
text_message = tk.Text(window, height=10, width=60)
text_message.pack()

# Release date input
tk.Label(window, text="Release Date (YYYY-MM-DD HH:MM:SS):").pack()
entry_date = tk.Entry(window, width=50)
entry_date.pack()

# Buttons for saving and retrieving capsules
tk.Button(window, text="Save Capsule", command=submit_data).pack(pady=10)
tk.Button(window, text="Retrieve Released Capsules", command=retrieve_capsules).pack()

# Start the Tkinter event loop
window.mainloop()
