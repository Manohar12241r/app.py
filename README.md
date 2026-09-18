# app.py
# 🥕 Vegetable Market Bill Manager

A web-based Bill Record Management System developed for a vegetable market that supplies vegetables to multiple hotels.

## 📌 Project Overview

The Vegetable Market Bill Manager is a Flask and SQLite based web application used to maintain and manage vegetable supply bills for hotels.

The system is designed for a vegetable market supplying approximately 40 hotels.

It is NOT a point-of-sale billing system.

The main purpose is to maintain records of bills supplied to hotels.

## ✨ Features

- Dashboard
- Hotel management
- Add hotels
- Edit hotel information
- Store owner details
- Store phone numbers
- Store addresses
- Store GST numbers
- Add vegetable supply bills
- Multiple vegetables per bill
- Quantity management
- Unit selection
- Rate management
- Automatic amount calculation
- Automatic total calculation
- Bill history
- Hotel-wise bill records
- Search hotels
- Search bills
- View individual bills
- Print bills
- Delete bills
- SQLite database
- Responsive web interface

## 🛠 Technologies Used

- Python
- Flask
- SQLite
- HTML
- CSS
- JavaScript
- Jinja2
- Git
- GitHub

## 📁 Project Structure

```text
vegetable_market_bill_manager/
│
├── app.py
├── requirements.txt
├── README.md
├── .gitignore
│
├── market.db
│
├── templates/
│   ├── base.html
│   ├── dashboard.html
│   ├── hotels.html
│   ├── hotel_detail.html
│   ├── add_bill.html
│   ├── bill_detail.html
│   ├── edit_hotel.html
│   └── all_bills.html
│
├── static/
│   └── style.css
│
└── uploads/
    └── .gitkeep
