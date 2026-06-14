# Implementation Guide: Complaint Management System with Chatbot Integration
## 1. Introduction
This guide provides a step-by-step approach to implementing a Complaint Management System (CMS) with chatbot integration and automated ticket generation. We will use a Python-based backend with Flask, a PostgreSQL database, and Google Dialogflow for the chatbot. This combination offers a powerful and flexible platform for building a modern customer support system.
## 2. Setting Up the Environment
### 2.1 Backend Setup (Python & Flask)

1.  **Install Python and pip:** Ensure you have Python 3.x and pip installed.
2.  **Create a virtual environment:**
    ```bash
    python -m venv venv
    source venv/bin/activate  # On Windows, use `venv\Scripts\activate`
    ```
3.  **Install required libraries:**
    ```bash
    pip install Flask Flask-SQLAlchemy Flask-Migrate psycopg2-binary requests google-cloud-dialogflow
    ```
    *   `Flask`: A lightweight web framework for Python.
    *   `Flask-SQLAlchemy`: An extension for Flask that adds support for SQLAlchemy.
    *   `Flask-Migrate`: An extension that handles SQLAlchemy database migrations for Flask applications using Alembic.
    *   `psycopg2-binary`: PostgreSQL adapter for Python.
    *   `requests`: For making HTTP requests to external APIs.
    *   `google-cloud-dialogflow`: The official Google Cloud Dialogflow API client library for Python.

### 2.2 Database Setup (PostgreSQL)

1.  **Install and run PostgreSQL.**
2.  **Create a database for the CMS:**
    ```sql
    CREATE DATABASE complaint_management_system;
    ```

### 2.3 Chatbot Setup (Google Dialogflow)

1.  **Create a Google Cloud Platform (GCP) project:** Go to the [GCP Console](https://console.cloud.google.com/) and create a new project.
2.  **Enable the Dialogflow API:** In your GCP project, go to the API Library and enable the "Dialogflow API".
3.  **Create a Dialogflow agent:** Go to the [Dialogflow Console](https://dialogflow.cloud.google.com/) and create a new agent, linking it to your GCP project.
4.  **Create a service account:** In the GCP IAM & Admin section, create a service account with the "Dialogflow API Client" role. Download the JSON key file for this service account; you will need it for authentication.
5.  **Set up environment variable for authentication:**
    ```bash
    export GOOGLE_APPLICATION_CREDENTIALS="/path/to/your/keyfile.json"
    ```
6.  **Create Intents in Dialogflow:**
    *   **`Welcome Intent`:** To greet the user.
    *   **`FileComplaint Intent`:** To handle the user's request to file a complaint. Define training phrases like "I want to file a complaint", "I have an issue", etc. Define parameters to collect information like `subject`, `description`, and `customer_email`.
    *   **`CheckStatus Intent`:** To check the status of a complaint. Define parameters like `ticket_id`.
    *   **`Fallback Intent`:** To handle unrecognized user inputs.

## 3. Backend Implementation (Flask)

### 3.1 Initial Flask Application Setup

Create a file named `app.py`:

```python
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
import os

app = Flask(__name__)

# Database Configuration
app.config["SQLALCHEMY_DATABASE_URI"] = "postgresql://user:password@host:port/complaint_management_system"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

db = SQLAlchemy(app)
migrate = Migrate(app, db)

# Import models after db is initialized
from models import Complaint, Ticket, Customer

@app.route("/")
def index():
    return "Complaint Management System API is running!"

if __name__ == "__main__":
    app.run(debug=True)
```

### 3.2 Database Models

Create a file named `models.py`:

```python
from app import db
import uuid

class Customer(db.Model):
    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    name = db.Column(db.String(100))
    email = db.Column(db.String(100), unique=True, nullable=False)

class Complaint(db.Model):
    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    customer_id = db.Column(db.String(36), db.ForeignKey("customer.id"), nullable=False)
    subject = db.Column(db.String(200), nullable=False)
    description = db.Column(db.Text, nullable=False)
    status = db.Column(db.String(50), default="New")
    created_at = db.Column(db.DateTime, server_default=db.func.now())

    customer = db.relationship("Customer", backref=db.backref("complaints", lazy=True))

class Ticket(db.Model):
    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    complaint_id = db.Column(db.String(36), db.ForeignKey("complaint.id"), nullable=False)
    status = db.Column(db.String(50), default="Open")
    created_at = db.Column(db.DateTime, server_default=db.func.now())

    complaint = db.relationship("Complaint", backref=db.backref("ticket", uselist=False))
```

After defining your models, run the database migrations:

```bash
flask db init
flask db migrate -m "Initial migration."
flask db upgrade
```

### 3.3 API Endpoints

Add the following endpoints to `app.py`:

```python
# ... (imports and app setup)

@app.route("/complaints", methods=["POST"])
def create_complaint():
    data = request.get_json()
    customer = Customer.query.filter_by(email=data["email"]).first()
    if not customer:
        customer = Customer(name=data.get("name"), email=data["email"])
        db.session.add(customer)
        db.session.commit()

    new_complaint = Complaint(
        customer_id=customer.id,
        subject=data["subject"],
        description=data["description"]
    )
    db.session.add(new_complaint)
    db.session.commit()

    # Automatically generate a ticket
    new_ticket = Ticket(complaint_id=new_complaint.id)
    db.session.add(new_ticket)
    db.session.commit()

    return jsonify({"message": "Complaint filed successfully", "ticket_id": new_ticket.id}), 201

@app.route("/complaints/<ticket_id>/status", methods=["GET"])
def get_complaint_status(ticket_id):
    ticket = Ticket.query.get(ticket_id)
    if not ticket:
        return jsonify({"error": "Ticket not found"}), 404
    return jsonify({"status": ticket.complaint.status})

# ... (rest of the file)
```

### 3.4 Dialogflow Webhook

Now, let's create a webhook endpoint in `app.py` that Dialogflow will call to fulfill intents.

```python
# ... (imports and app setup)

@app.route("/webhook", methods=["POST"])
def webhook():
    req = request.get_json(force=True)
    intent_name = req.get("queryResult").get("intent").get("displayName")

    if intent_name == "FileComplaint Intent":
        # Extract parameters from Dialogflow request
        params = req.get("queryResult").get("parameters")
        customer_email = params.get("customer_email")
        subject = params.get("subject")
        description = params.get("description")

        # Find or create customer
        customer = Customer.query.filter_by(email=customer_email).first()
        if not customer:
            customer = Customer(email=customer_email)
            db.session.add(customer)
            db.session.commit()

        # Create complaint and ticket
        new_complaint = Complaint(customer_id=customer.id, subject=subject, description=description)
        db.session.add(new_complaint)
        db.session.commit()

        new_ticket = Ticket(complaint_id=new_complaint.id)
        db.session.add(new_ticket)
        db.session.commit()

        response_text = f"Thank you for filing a complaint. Your ticket ID is {new_ticket.id}."

    elif intent_name == "CheckStatus Intent":
        params = req.get("queryResult").get("parameters")
        ticket_id = params.get("ticket_id")
        ticket = Ticket.query.get(ticket_id)
        if ticket:
            response_text = f"The status of your complaint (Ticket ID: {ticket_id}) is: {ticket.complaint.status}."
        else:
            response_text = f"Sorry, I couldn't find a ticket with the ID {ticket_id}."

    else:
        response_text = "I'm sorry, I don't understand. Can you please rephrase?"

    return jsonify({"fulfillmentText": response_text})

# ... (rest of the file)
```

## 4. Deploying and Connecting the Webhook

### 4.1 Deploying the Flask App
To make your webhook accessible to Dialogflow, you need to deploy it to a public server. For development, you can use a tool like `ngrok` to expose your local server to the internet.

1.  **Install ngrok:** [https://ngrok.com/download](https://ngrok.com/download)
2.  **Run your Flask app:** `python app.py` (it will run on `http://127.0.0.1:5000`)
3.  **Expose your local server with ngrok:**
    ```bash
    ngrok http 5000
    ```
    `ngrok` will give you a public HTTPS URL (e.g., `https://xxxxxxxx.ngrok.io`).

### 4.2 Connecting Dialogflow to the Webhook

1.  In the Dialogflow Console, go to the **Fulfillment** section.
2.  Enable the **Webhook**.
3.  In the **URL** field, enter your ngrok URL followed by `/webhook` (e.g., `https://xxxxxxxx.ngrok.io/webhook`).
4.  Save the changes.
5.  For each intent that needs to be fulfilled by your webhook (e.g., `FileComplaint Intent`, `CheckStatus Intent`), go to the intent's settings, expand the **Fulfillment** section, and enable **Enable webhook call for this intent**.

## 5. Frontend (Optional but Recommended)

A simple frontend can be built using HTML, CSS, and JavaScript to provide a chat interface.

### 5.1 HTML (`index.html`)

```html
<!DOCTYPE html>
<html>
<head>
    <title>Complaint Management Chatbot</title>
    <style>
        /* Add some basic styling for the chat window */
    </style>
</head>
<body>
    <div id="chat-container">
        <div id="chat-log"></div>
        <input type="text" id="user-input" placeholder="Type your message...">
        <button onclick="sendMessage()">Send</button>
    </div>

    <script>
        // JavaScript to handle sending messages to Dialogflow
        // and displaying the conversation
    </script>
</body>
</html>
```

### 5.2 JavaScript

You would need to use the Dialogflow V2 API (or a library that wraps it) to send user messages from your frontend to your Dialogflow agent. The response from Dialogflow (which may come from your webhook) would then be displayed in the chat log.

## 6. Further Enhancements

*   **Agent Dashboard:** Build a separate web interface for support agents to view and manage complaints and tickets.
*   **Real-time Updates:** Use WebSockets (e.g., with Flask-SocketIO) to provide real-time updates on ticket status to both customers and agents.
*   **Knowledge Base Integration:** Create a knowledge base and integrate it with the chatbot to answer frequently asked questions automatically.
*   **Authentication:** Implement user authentication for customers and agents.
*   **Advanced NLP:** For more complex scenarios, you could use more advanced NLP models (e.g., with Rasa or custom models) for intent recognition and entity extraction.

This guide provides a solid starting point for building a Complaint Management System with chatbot integration. By following these steps, you can create a functional and modern customer support solution.


