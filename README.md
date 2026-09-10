# BUS-RESERVATION#!/usr/bin/env python3
"""
Flask Bus Reservation App (prototype)
- SQLite (app.db) + SQLAlchemy
- Flask-Login for authentication
- Werkzeug password hashing
- Seed demo data with --seed / --force-seed
"""

import os
import argparse
from datetime import datetime
from flask import Flask, render_template, request, redirect, url_for, flash, abort
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, login_user, login_required, logout_user, current_user, UserMixin
from werkzeug.security import generate_password_hash, check_password_hash
from sqlalchemy.exc import IntegrityError

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DB_PATH = os.path.join(BASE_DIR, "app.db")

app = Flask(__name__)
app.config["SECRET_KEY"] = os.environ.get("FLASK_SECRET", "change-me-in-production")
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///" + DB_PATH
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

db = SQLAlchemy(app)
login_manager = LoginManager(app)
login_manager.login_view = "login"

# Models
class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password_hash = db.Column(db.String(256), nullable=False)
    is_admin = db.Column(db.Boolean, default=False)

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)


class Bus(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    bus_name = db.Column(db.String(128), nullable=False)
    route = db.Column(db.String(256), nullable=False)
    date = db.Column(db.Date, nullable=False)
    time = db.Column(db.Time, nullable=False)
    price = db.Column(db.Integer, nullable=False)  # simple integer price
    seats = db.Column(db.Integer, nullable=False, default=10)

    def __repr__(self):
        return f"<Bus {self.id} {self.bus_name}>"


class Booking(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    bus_id = db.Column(db.Integer, db.ForeignKey("bus.id"), nullable=False)
    seat = db.Column(db.Integer, nullable=False)
    status = db.Column(db.String(20), nullable=False, default="Booked")
    booked_at = db.Column(db.DateTime, default=datetime.utcnow)

    user = db.relationship("User", backref="bookings")
    bus = db.relationship("Bus", backref="bookings")

    __table_args__ = (
        # prevent duplicate active bookings for same bus + seat + status
        db.UniqueConstraint("bus_id", "seat", "status", name="uix_bus_seat_status"),
    )


@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))


# Helpers
def init_db():
    # Ensure database tables are created inside application context
    with app.app_context():
        db.create_all()


def seed_demo_data(force=False):
    """Seed users, buses, bookings. If force=True, recreate tables."""
    # Perform all DB operations inside application context
    with app.app_context():
        if force:
            # Ensure active sessions/engines are closed first (prevents Windows file-lock)
            try:
                db.session.remove()
            except Exception:
                pass
            try:
                db.engine.dispose()
            except Exception:
                pass

            if os.path.exists(DB_PATH):
                os.remove(DB_PATH)
            db.create_all()

        # Seed only if no buses exist
        if Bus.query.first():
            print("Seed skipped: buses already exist.")
            return

        # Users
        demo_users = [
            ("yetunde", "yetunde33", True),
            ("timmy", "timmy43", False),
            ("loner", "loner43", False),
            ("basit", "basit43", False),
        ]
        extra_usernames = [
            "seun", "bola", "chinedu", "uche", "amaka", "emeka", "ayodele", "toyin",
            "ifeoma", "babatunde", "tolu", "funke", "abubakar", "ibrahim", "halima", "fatima", "kunle", "oluchi",
            "hannah", "efe", "oluwaseun", "ebuka", "salihu", "bukola", "ayomide", "ruth", "timothy", "esther",
            "joseph", "stephen", "favour", "raphael", "precious", "miracle", "desmond", "david", "sophia",
            "anthony", "kelvin", "charity", "grace", "patience", "isaac", "cynthia", "kenneth", "sunday"
        ]
        for u, p, a in demo_users:
            if not User.query.filter_by(username=u).first():
                user = User(username=u, is_admin=a)
                user.set_password(p)
                db.session.add(user)
        # extra users with password <name>33
        for uname in extra_usernames:
            if not User.query.filter_by(username=uname).first():
                u = User(username=uname, is_admin=False)
                u.set_password(f"{uname}33")
                db.session.add(u)
        db.session.commit()

        # Buses (10)
        buses = [
            (1, "Toyota Coaster", "Lekki to Ikeja", "2025-11-11", "07:00", 5000, 10),
            (2, "Hyundai County", "Ikeja to Maryland", "2025-11-11", "08:00", 5000, 10),
            (3, "Ford Transit", "Maryland to Ketu", "2025-11-11", "09:00", 5000, 10),
            (4, "Mercedes Sprinter", "Ketu to Magodo", "2025-11-11", "10:00", 5000, 10),
            (5, "Nissan Civilian", "Magodo to Onipan", "2025-11-11", "11:00", 5000, 10),
            (6, "Isuzu Journey", "Onipan to Yaba", "2025-11-11", "14:00", 3500, 10),
            (7, "Volkswagen Crafter", "Yaba to Fadeyi", "2025-11-11", "15:00", 3500, 10),
            (8, "Renault Master", "Fadeyi to Palmgroove", "2025-11-11", "16:00", 3500, 10),
            (9, "Fiat Ducato", "Palmgroove to Eko", "2025-11-11", "17:00", 3500, 10),
            (10, "Peugeot Boxer", "Eko to Lekki", "2025-11-11", "18:00", 3500, 10),
        ]
        for _, name, route, dstr, tstr, price, seats in buses:
            d = datetime.strptime(dstr, "%Y-%m-%d").date()
            t = datetime.strptime(tstr, "%H:%M").time()
            b = Bus(bus_name=name, route=route, date=d, time=t, price=price, seats=seats)
            db.session.add(b)
        db.session.commit()

        # Demo bookings
        now = datetime.utcnow()
        # timmy seats=1 on buses 1-4
        timmy = User.query.filter_by(username="timmy").first()
        for bus_id in [1, 2, 3, 4]:
            bk = Booking(user_id=timmy.id, bus_id=bus_id, seat=1, status="Booked", booked_at=now)
            db.session.add(bk)
        # loner seats=2 on buses 1-3
        loner = User.query.filter_by(username="loner").first()
        for bus_id in [1, 2, 3]:
            bk = Booking(user_id=loner.id, bus_id=bus_id, seat=2, status="Booked", booked_at=now)
            db.session.add(bk)
        # basit seats=3 on buses 1-3
        basit = User.query.filter_by(username="basit").first()
        for bus_id in [1, 2, 3]:
            bk = Booking(user_id=basit.id, bus_id=bus_id, seat=3, status="Booked", booked_at=now)
            db.session.add(bk)

        db.session.commit()
        print("Seed data written.")

        # Seed only if no buses exist
        if Bus.query.first():
            print("Seed skipped: buses already exist.")
            return

        # Users
        demo_users = [
            ("yetunde", "yetunde33", True),
            ("timmy", "timmy43", False),
            ("loner", "loner43", False),
            ("basit", "basit43", False),
        ]
        extra_usernames = [
            "seun", "bola", "chinedu", "uche", "amaka", "emeka", "ayodele", "toyin",
            "ifeoma", "babatunde", "tolu", "funke", "abubakar", "ibrahim", "halima", "fatima", "kunle", "oluchi",
            "hannah", "efe", "oluwaseun", "ebuka", "salihu", "bukola", "ayomide", "ruth", "timothy", "esther",
            "joseph", "stephen", "favour", "raphael", "precious", "miracle", "desmond", "david", "sophia",
            "anthony", "kelvin", "charity", "grace", "patience", "isaac", "cynthia", "kenneth", "sunday"
        ]
        for u, p, a in demo_users:
            if not User.query.filter_by(username=u).first():
                user = User(username=u, is_admin=a)
                user.set_password(p)
                db.session.add(user)
        # extra users with password <name>33
        for uname in extra_usernames:
            if not User.query.filter_by(username=uname).first():
                u = User(username=uname, is_admin=False)
                u.set_password(f"{uname}33")
                db.session.add(u)
        db.session.commit()

        # Buses (10)
        buses = [
            (1, "Toyota Coaster", "Lekki to Ikeja", "2025-11-11", "07:00", 5000, 10),
            (2, "Hyundai County", "Ikeja to Maryland", "2025-11-11", "08:00", 5000, 10),
            (3, "Ford Transit", "Maryland to Ketu", "2025-11-11", "09:00", 5000, 10),
            (4, "Mercedes Sprinter", "Ketu to Magodo", "2025-11-11", "10:00", 5000, 10),
            (5, "Nissan Civilian", "Magodo to Onipan", "2025-11-11", "11:00", 5000, 10),
            (6, "Isuzu Journey", "Onipan to Yaba", "2025-11-11", "14:00", 3500, 10),
            (7, "Volkswagen Crafter", "Yaba to Fadeyi", "2025-11-11", "15:00", 3500, 10),
            (8, "Renault Master", "Fadeyi to Palmgroove", "2025-11-11", "16:00", 3500, 10),
            (9, "Fiat Ducato", "Palmgroove to Eko", "2025-11-11", "17:00", 3500, 10),
            (10, "Peugeot Boxer", "Eko to Lekki", "2025-11-11", "18:00", 3500, 10),
        ]
        for _, name, route, dstr, tstr, price, seats in buses:
            d = datetime.strptime(dstr, "%Y-%m-%d").date()
            t = datetime.strptime(tstr, "%H:%M").time()
            b = Bus(bus_name=name, route=route, date=d, time=t, price=price, seats=seats)
            db.session.add(b)
        db.session.commit()

        # Demo bookings
        now = datetime.utcnow()
        # timmy seats=1 on buses 1-4
        timmy = User.query.filter_by(username="timmy").first()
        for bus_id in [1, 2, 3, 4]:
            bk = Booking(user_id=timmy.id, bus_id=bus_id, seat=1, status="Booked", booked_at=now)
            db.session.add(bk)
        # loner seats=2 on buses 1-3
        loner = User.query.filter_by(username="loner").first()
        for bus_id in [1, 2, 3]:
            bk = Booking(user_id=loner.id, bus_id=bus_id, seat=2, status="Booked", booked_at=now)
            db.session.add(bk)
        # basit seats=3 on buses 1-3
        basit = User.query.filter_by(username="basit").first()
        for bus_id in [1, 2, 3]:
            bk = Booking(user_id=basit.id, bus_id=bus_id, seat=3, status="Booked", booked_at=now)
            db.session.add(bk)

        db.session.commit()
        print("Seed data written.")


# Routes
@app.route("/")
def index():
    buses = Bus.query.order_by(Bus.date, Bus.time).all()
    return render_template("index.html", buses=buses)


@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        username = request.form["username"].strip()
        password = request.form["password"].strip()
        if not username or not password:
            flash("Username and password are required.", "danger")
            return redirect(url_for("register"))
        if User.query.filter_by(username=username).first():
            flash("Username already exists.", "warning")
            return redirect(url_for("register"))
        user = User(username=username)
        user.set_password(password)
        db.session.add(user)
        db.session.commit()
        flash("Registration successful. Please log in.", "success")
        return redirect(url_for("login"))
    return render_template("register.html")


@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = request.form["username"].strip()
        password = request.form["password"].strip()
        user = User.query.filter_by(username=username).first()
        if user and user.check_password(password):
            login_user(user)
            flash("Login successful.", "success")
            return redirect(url_for("index"))
        flash("Invalid credentials.", "danger")
    return render_template("login.html")


@app.route("/logout")
@login_required
def logout():
    logout_user()
    flash("Logged out.", "info")
    return redirect(url_for("index"))


@app.route("/bus/<int:bus_id>", methods=["GET", "POST"])
def bus_detail(bus_id):
    bus = Bus.query.get_or_404(bus_id)
    taken = [b.seat for b in Booking.query.filter_by(bus_id=bus.id, status="Booked").all()]
    if request.method == "POST":
        if not current_user.is_authenticated:
            flash("You must log in to book.", "warning")
            return redirect(url_for("login"))
        try:
            seat = int(request.form["seat"])
        except (KeyError, ValueError):
            flash("Invalid seat number.", "danger")
            return redirect(url_for("bus_detail", bus_id=bus_id))
        if seat < 1 or seat > bus.seats:
            flash("Seat number out of range.", "danger")
            return redirect(url_for("bus_detail", bus_id=bus_id))
        # attempt to create booking
        booking = Booking(user_id=current_user.id, bus_id=bus.id, seat=seat, status="Booked")
        db.session.add(booking)
        try:
            db.session.commit()
            flash(f"Seat {seat} booked.", "success")
            return redirect(url_for("my_bookings"))
        except IntegrityError:
            db.session.rollback()
            flash("Seat already booked. Choose another seat.", "danger")
            return redirect(url_for("bus_detail", bus_id=bus_id))
    return render_template("bus_detail.html", bus=bus, taken=taken)


@app.route("/bookings")
@login_required
def my_bookings():
    bookings = Booking.query.filter_by(user_id=current_user.id).join(Bus).order_by(Booking.booked_at.desc()).all()
    return render_template("my_bookings.html", bookings=bookings)


@app.route("/cancel/<int:booking_id>")
@login_required
def cancel_booking(booking_id):
    booking = Booking.query.filter_by(id=booking_id, user_id=current_user.id).first_or_404()
    if booking.status == "Booked":
        booking.status = "Cancelled"
        db.session.commit()
        flash("Booking cancelled.", "info")
    return redirect(url_for("my_bookings"))


# Simple admin page (requires is_admin)
@app.route("/admin", methods=["GET", "POST"])
@login_required
def admin():
    # Only allow admin users
    if not current_user.is_admin:
        abort(403)

    if request.method == "POST":
        # Handle delete requests first
        if "delete_bus_id" in request.form:
            try:
                delete_id = int(request.form.get("delete_bus_id"))
                bus = Bus.query.get(delete_id)
                if not bus:
                    flash("Bus not found.", "warning")
                else:
                    db.session.delete(bus)
                    db.session.commit()
                    flash("Bus deleted.", "info")
            except Exception as e:
                db.session.rollback()
                app.logger.exception("Error deleting bus")
                flash("Error deleting bus.", "danger")
            return redirect(url_for("admin"))

        # Otherwise assume this is the Add Bus submission
        name = request.form.get("bus_name", "").strip()
        route = request.form.get("route", "").strip()
        date_str = request.form.get("date", "").strip()
        time_str = request.form.get("time", "").strip()
        try:
            price = int(request.form.get("price", "0"))
            seats = int(request.form.get("seats", "10"))
        except ValueError:
            flash("Invalid price or seats", "danger")
            return redirect(url_for("admin"))
        try:
            d = datetime.strptime(date_str, "%Y-%m-%d").date()
            t = datetime.strptime(time_str, "%H:%M").time()
        except ValueError:
            flash("Invalid date or time format", "danger")
            return redirect(url_for("admin"))
        if not name or not route:
            flash("Bus name and route are required.", "warning")
            return redirect(url_for("admin"))

        bus = Bus(bus_name=name, route=route, date=d, time=t, price=price, seats=seats)
        db.session.add(bus)
        db.session.commit()
        flash("Bus added.", "success")
        return redirect(url_for("admin"))

    # GET: show page
    buses = Bus.query.order_by(Bus.date, Bus.time).all()
    return render_template("admin.html", buses=buses)

# CLI entry
if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Run Flask Bus Reservation app")
    parser.add_argument("--seed", action="store_true", help="Seed demo data (creates DB if missing)")
    parser.add_argument("--force-seed", action="store_true", help="Force reseed (delete DB and recreate)")
    args = parser.parse_args()

    # Initialize DB and optionally seed inside app context
    init_db()
    if args.force_seed:
        seed_demo_data(force=True)
    elif args.seed:
        seed_demo_data(force=False)

    # Run dev server
    app.run(debug=True)
