# app.py
from flask import Flask, render_template, request, redirect, url_for, flash
import sqlite3
import os
from datetime import date

app = Flask(__name__)
app.secret_key = "vegetable-market-secret-key"

DATABASE = "market.db"
UPLOAD_FOLDER = "uploads"

app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER

os.makedirs(UPLOAD_FOLDER, exist_ok=True)


# ---------------------------------------------------------
# DATABASE CONNECTION
# ---------------------------------------------------------

def get_db():
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    return conn


# ---------------------------------------------------------
# CREATE DATABASE TABLES
# ---------------------------------------------------------

def init_db():
    conn = get_db()

    conn.execute("""
        CREATE TABLE IF NOT EXISTS hotels (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            owner_name TEXT,
            phone TEXT,
            address TEXT,
            gst_number TEXT,
            created_at TEXT DEFAULT CURRENT_TIMESTAMP
        )
    """)

    conn.execute("""
        CREATE TABLE IF NOT EXISTS bills (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            hotel_id INTEGER NOT NULL,
            bill_number TEXT NOT NULL UNIQUE,
            bill_date TEXT NOT NULL,
            notes TEXT,
            total_amount REAL DEFAULT 0,
            created_at TEXT DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (hotel_id) REFERENCES hotels(id)
        )
    """)

    conn.execute("""
        CREATE TABLE IF NOT EXISTS bill_items (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            bill_id INTEGER NOT NULL,
            vegetable_name TEXT NOT NULL,
            quantity REAL NOT NULL,
            unit TEXT NOT NULL,
            rate REAL NOT NULL,
            amount REAL NOT NULL,
            FOREIGN KEY (bill_id) REFERENCES bills(id) ON DELETE CASCADE
        )
    """)

    conn.commit()
    conn.close()


# ---------------------------------------------------------
# DASHBOARD
# ---------------------------------------------------------

@app.route("/")
def dashboard():

    conn = get_db()

    total_hotels = conn.execute(
        "SELECT COUNT(*) FROM hotels"
    ).fetchone()[0]

    total_bills = conn.execute(
        "SELECT COUNT(*) FROM bills"
    ).fetchone()[0]

    total_amount = conn.execute(
        "SELECT COALESCE(SUM(total_amount), 0) FROM bills"
    ).fetchone()[0]

    recent_bills = conn.execute("""
        SELECT bills.*, hotels.name AS hotel_name
        FROM bills
        JOIN hotels ON bills.hotel_id = hotels.id
        ORDER BY bills.bill_date DESC, bills.id DESC
        LIMIT 10
    """).fetchall()

    conn.close()

    return render_template(
        "dashboard.html",
        total_hotels=total_hotels,
        total_bills=total_bills,
        total_amount=total_amount,
        recent_bills=recent_bills
    )


# ---------------------------------------------------------
# HOTELS
# ---------------------------------------------------------

@app.route("/hotels")
def hotels():

    search = request.args.get("search", "").strip()

    conn = get_db()

    if search:
        hotels = conn.execute("""
            SELECT * FROM hotels
            WHERE name LIKE ?
               OR owner_name LIKE ?
               OR phone LIKE ?
            ORDER BY name
        """, (
            f"%{search}%",
            f"%{search}%",
            f"%{search}%"
        )).fetchall()
    else:
        hotels = conn.execute("""
            SELECT * FROM hotels
            ORDER BY name
        """).fetchall()

    conn.close()

    return render_template(
        "hotels.html",
        hotels=hotels,
        search=search
    )


# ---------------------------------------------------------
# ADD HOTEL
# ---------------------------------------------------------

@app.route("/hotels/add", methods=["GET", "POST"])
def add_hotel():

    if request.method == "POST":

        name = request.form["name"].strip()
        owner_name = request.form["owner_name"].strip()
        phone = request.form["phone"].strip()
        address = request.form["address"].strip()
        gst_number = request.form["gst_number"].strip()

        if not name:
            flash("Hotel name is required.", "danger")
            return redirect(url_for("add_hotel"))

        conn = get_db()

        conn.execute("""
            INSERT INTO hotels
            (name, owner_name, phone, address, gst_number)
            VALUES (?, ?, ?, ?, ?)
        """, (
            name,
            owner_name,
            phone,
            address,
            gst_number
        ))

        conn.commit()
        conn.close()

        flash("Hotel added successfully.", "success")

        return redirect(url_for("hotels"))

    return render_template("edit_hotel.html", hotel=None)


# ---------------------------------------------------------
# EDIT HOTEL
# ---------------------------------------------------------

@app.route("/hotels/<int:hotel_id>/edit", methods=["GET", "POST"])
def edit_hotel(hotel_id):

    conn = get_db()

    hotel = conn.execute(
        "SELECT * FROM hotels WHERE id = ?",
        (hotel_id,)
    ).fetchone()

    if hotel is None:
        conn.close()
        flash("Hotel not found.", "danger")
        return redirect(url_for("hotels"))

    if request.method == "POST":

        name = request.form["name"].strip()
        owner_name = request.form["owner_name"].strip()
        phone = request.form["phone"].strip()
        address = request.form["address"].strip()
        gst_number = request.form["gst_number"].strip()

        conn.execute("""
            UPDATE hotels
            SET name = ?,
                owner_name = ?,
                phone = ?,
                address = ?,
                gst_number = ?
            WHERE id = ?
        """, (
            name,
            owner_name,
            phone,
            address,
            gst_number,
            hotel_id
        ))

        conn.commit()
        conn.close()

        flash("Hotel information updated.", "success")

        return redirect(
            url_for("hotel_detail", hotel_id=hotel_id)
        )

    conn.close()

    return render_template(
        "edit_hotel.html",
        hotel=hotel
    )


# ---------------------------------------------------------
# HOTEL DETAILS
# ---------------------------------------------------------

@app.route("/hotels/<int:hotel_id>")
def hotel_detail(hotel_id):

    conn = get_db()

    hotel = conn.execute(
        "SELECT * FROM hotels WHERE id = ?",
        (hotel_id,)
    ).fetchone()

    if hotel is None:
        conn.close()
        flash("Hotel not found.", "danger")
        return redirect(url_for("hotels"))

    bills = conn.execute("""
        SELECT *
        FROM bills
        WHERE hotel_id = ?
        ORDER BY bill_date DESC, id DESC
    """, (hotel_id,)).fetchall()

    hotel_total = conn.execute("""
        SELECT COALESCE(SUM(total_amount), 0)
        FROM bills
        WHERE hotel_id = ?
    """, (hotel_id,)).fetchone()[0]

    conn.close()

    return render_template(
        "hotel_detail.html",
        hotel=hotel,
        bills=bills,
        hotel_total=hotel_total
    )


# ---------------------------------------------------------
# ADD BILL
# ---------------------------------------------------------

@app.route("/bills/add", methods=["GET", "POST"])
def add_bill():

    conn = get_db()

    hotels = conn.execute("""
        SELECT * FROM hotels
        ORDER BY name
    """).fetchall()

    if request.method == "POST":

        hotel_id = request.form["hotel_id"]
        bill_number = request.form["bill_number"].strip()
        bill_date = request.form["bill_date"]
        notes = request.form["notes"].strip()

        vegetable_names = request.form.getlist("vegetable_name[]")
        quantities = request.form.getlist("quantity[]")
        units = request.form.getlist("unit[]")
        rates = request.form.getlist("rate[]")

        if not hotel_id:
            flash("Please select a hotel.", "danger")
            conn.close()
            return redirect(url_for("add_bill"))

        if not bill_number:
            flash("Bill number is required.", "danger")
            conn.close()
            return redirect(url_for("add_bill"))

        if not vegetable_names:
            flash("Please add at least one vegetable.", "danger")
            conn.close()
            return redirect(url_for("add_bill"))

        try:

            # Calculate total
            total_amount = 0

            items = []

            for i in range(len(vegetable_names)):

                name = vegetable_names[i].strip()

                if not name:
                    continue

                quantity = float(quantities[i])
                rate = float(rates[i])
                unit = units[i]

                amount = quantity * rate

                total_amount += amount

                items.append(
                    (
                        name,
                        quantity,
                        unit,
                        rate,
                        amount
                    )
                )

            if not items:
                flash("Please add valid vegetable items.", "danger")
                conn.close()
                return redirect(url_for("add_bill"))

            # Insert bill
            cursor = conn.execute("""
                INSERT INTO bills
                (hotel_id, bill_number, bill_date, notes, total_amount)
                VALUES (?, ?, ?, ?, ?)
            """, (
                hotel_id,
                bill_number,
                bill_date,
                notes,
                total_amount
            ))

            bill_id = cursor.lastrowid

            # Insert bill items
            for item in items:

                conn.execute("""
                    INSERT INTO bill_items
                    (bill_id, vegetable_name, quantity, unit, rate, amount)
                    VALUES (?, ?, ?, ?, ?, ?)
                """, (
                    bill_id,
                    item[0],
                    item[1],
                    item[2],
                    item[3],
                    item[4]
                ))

            conn.commit()

            flash("Bill saved successfully.", "success")

            return redirect(
                url_for("bill_detail", bill_id=bill_id)
            )

        except ValueError:

            conn.rollback()

            flash(
                "Please enter valid quantity and rate values.",
                "danger"
            )

        except sqlite3.IntegrityError:

            conn.rollback()

            flash(
                "Bill number already exists.",
                "danger"
            )

    conn.close()

    return render_template(
        "add_bill.html",
        hotels=hotels,
        today=date.today().isoformat()
    )


# ---------------------------------------------------------
# BILL DETAIL
# ---------------------------------------------------------

@app.route("/bills/<int:bill_id>")
def bill_detail(bill_id):

    conn = get_db()

    bill = conn.execute("""
        SELECT bills.*, hotels.name AS hotel_name,
               hotels.owner_name,
               hotels.phone,
               hotels.address,
               hotels.gst_number
        FROM bills
        JOIN hotels ON bills.hotel_id = hotels.id
        WHERE bills.id = ?
    """, (bill_id,)).fetchone()

    if bill is None:
        conn.close()
        flash("Bill not found.", "danger")
        return redirect(url_for("all_bills"))

    items = conn.execute("""
        SELECT *
        FROM bill_items
        WHERE bill_id = ?
        ORDER BY id
    """, (bill_id,)).fetchall()

    conn.close()

    return render_template(
        "bill_detail.html",
        bill=bill,
        items=items
    )


# ---------------------------------------------------------
# ALL BILLS
# ---------------------------------------------------------

@app.route("/bills")
def all_bills():

    search = request.args.get("search", "").strip()

    conn = get_db()

    if search:

        bills = conn.execute("""
            SELECT bills.*, hotels.name AS hotel_name
            FROM bills
            JOIN hotels ON bills.hotel_id = hotels.id
            WHERE bills.bill_number LIKE ?
               OR hotels.name LIKE ?
            ORDER BY bills.bill_date DESC, bills.id DESC
        """, (
            f"%{search}%",
            f"%{search}%"
        )).fetchall()

    else:

        bills = conn.execute("""
            SELECT bills.*, hotels.name AS hotel_name
            FROM bills
            JOIN hotels ON bills.hotel_id = hotels.id
            ORDER BY bills.bill_date DESC, bills.id DESC
        """).fetchall()

    conn.close()

    return render_template(
        "all_bills.html",
        bills=bills,
        search=search
    )


# ---------------------------------------------------------
# DELETE BILL
# ---------------------------------------------------------

@app.route("/bills/<int:bill_id>/delete", methods=["POST"])
def delete_bill(bill_id):

    conn = get_db()

    conn.execute(
        "DELETE FROM bill_items WHERE bill_id = ?",
        (bill_id,)
    )

    conn.execute(
        "DELETE FROM bills WHERE id = ?",
        (bill_id,)
    )

    conn.commit()
    conn.close()

    flash("Bill deleted successfully.", "success")

    return redirect(url_for("all_bills"))


# ---------------------------------------------------------
# RUN APPLICATION
# ---------------------------------------------------------

if __name__ == "__main__":

    init_db()

    app.run(
        debug=True,
        host="127.0.0.1",
        port=5000
    )
