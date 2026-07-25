# Import necessary libraries
from dotenv import load_dotenv
import base64
import streamlit as st
import os
import io
import mysql.connector
import hashlib
import re
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from PIL import Image
import pdf2image
import google.generativeai as genai
import logging
from datetime import datetime
import uuid
import time
import random

# Load environment variabless
load_dotenv()

# Set the Gemini API key
API_KEY = os.getenv("AIzaSyASrl1n-Kbrm47bqHKAP5LdW2PeEGhU1N8")
genai.configure(api_key=API_KEY)

# Configure the Streamlit app
st.set_page_config(page_title="ATS Resume Expert", layout="wide")

# Database Connection
@st.cache_resource
def get_connection():
    try:
        return mysql.connector.connect(
            host=os.getenv("DB_HOST", "localhost"),
            user=os.getenv("DB_USER", "root"),
            password=os.getenv("DB_PASSWORD", "123456"),
            database=os.getenv("DB_NAME", "ats_resume_expert")
        )
    except mysql.connector.Error as e:
        st.error(f"Database connection failed: {e}")
        return None

# Establish a connection to the database
conn = get_connection()
if conn:
    cursor = conn.cursor()

    # Create Tables in the Database
    cursor.execute('''CREATE TABLE IF NOT EXISTS users (
        id INT AUTO_INCREMENT PRIMARY KEY,
        username VARCHAR(255) UNIQUE,
        password VARCHAR(255),
        role ENUM('admin', 'user') DEFAULT 'user',
        email VARCHAR(255) UNIQUE
    )''')

    cursor.execute('''CREATE TABLE IF NOT EXISTS resumes (
        id INT AUTO_INCREMENT PRIMARY KEY,
        user_id INT,
        name VARCHAR(255),
        email VARCHAR(100) NOT NULL,
        phone VARCHAR(15),
        skills TEXT,
        experience TEXT,
        match_percentage FLOAT,
        file_hash VARCHAR(32),
        uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (user_id) REFERENCES users(id)
    )''')

    cursor.execute('''CREATE TABLE IF NOT EXISTS job_descriptions (
        id INT AUTO_INCREMENT PRIMARY KEY,
        title VARCHAR(255) DEFAULT 'Untitled',
        description TEXT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
    )''')

    conn.commit()

# Logging Configuration
logging.basicConfig(filename="app.log", level=logging.DEBUG, format="%(asctime)s - %(levelname)s - %(message)s")

# Hashing function for passwords
def hash_password(password):
    return hashlib.sha256(password.encode()).hexdigest()

# Password Strength Checker
def is_password_strong(password):
    if len(password) < 8:
        return False, "Password must be at least 8 characters long."
    if not re.search("[a-z]", password):
        return False, "Password must contain at least one lowercase letter."
    if not re.search("[A-Z]", password):
        return False, "Password must contain at least one uppercase letter."
    if not re.search("[0-9]", password):
        return False, "Password must contain at least one digit."
    if not re.search("[!@#$%^&*(),.?\":{}|<>]", password):
        return False, "Password must contain at least one special character."
    return True, "Password is strong."

# Email Notification Function
def send_email(to_email, subject, body):
    sender_email = os.getenv("hpapcp123@gmail.com")
    sender_password = os.getenv("Hitesh@200")

    if not sender_email or not sender_password:
        logging.error("Email credentials not found in environment variables.")
        return False

    msg = MIMEMultipart()
    msg['From'] = sender_email
    msg['To'] = to_email
    msg['Subject'] = subject
    msg.attach(MIMEText(body, 'plain'))

    try:
        with smtplib.SMTP("smtp.gmail.com", 587) as server:
            server.starttls()
            server.login(sender_email, sender_password)
            server.sendmail(sender_email, to_email, msg.as_string())
        logging.info(f"Email sent to {to_email}")
        return True
    except Exception as e:
        logging.error(f"Failed to send email: {e}")
        return False

# Register a new user
# [Previous code remains the same until line 129]

def get_gemini_response(input_text, pdf_content, prompt, existing_percentages=None):
    try:
        model = genai.GenerativeModel("gemini-1.5-flash")
        response = model.generate_content([input_text, pdf_content[0], prompt])
        logging.info(f"Gemini API Response: {response.text}")
        
        # Extract base percentage from Gemini's response
        try:
            base_percentage = float(re.search(r"(\d+(\.\d+)?)%", response.text).group(1))
        except (AttributeError, ValueError):
            base_percentage = 0
        
        # Adjust percentage to ensure uniqueness
        if existing_percentages is not None:
            adjusted_percentage = base_percentage
            while adjusted_percentage in existing_percentages:
                variation = random.uniform(0.1, 0.9)  # Add small variation
                adjusted_percentage = round(base_percentage + variation, 2)
                adjusted_percentage = max(0, min(100, adjusted_percentage))  # Keep within 0-100
            percentage = adjusted_percentage
        else:
            percentage = base_percentage
        
        # Ensure all instances of base percentage are replaced
        modified_response = re.sub(rf"\b{base_percentage}%", f"{percentage:.2f}%", response.text)

        return modified_response, percentage

    except Exception as e:
        logging.error(f"Error in get_gemini_response: {e}")
        st.error(f"Gemini API Error: {e}")
        return f"Error: {e}", 0

# Add the missing register_user function
def register_user(username, password, email, role="user"):
    if role not in ["admin", "user"]:
        st.error("Invalid role. Role must be either 'admin' or 'user'.")
        return False

    is_strong, feedback = is_password_strong(password)
    if not is_strong:
        st.error(feedback)
        return False

    cursor.execute("SELECT * FROM users WHERE username = %s OR email = %s", (username, email))
    if cursor.fetchone():
        st.error("Username or email already exists. Try a different one.")
        return False

    hashed_password = hash_password(password)
    try:
        cursor.execute("INSERT INTO users (username, password, email, role) VALUES (%s, %s, %s, %s)",
                       (username, hashed_password, email, role))
        conn.commit()
        send_email(email, "Welcome to ATS Resume Expert", "Thank you for registering!")
        return True
    except mysql.connector.Error as e:
        st.error(f"Database error: {e}")
        return False

# [Rest of the code remains the same]
def get_gemini_response(input_text, pdf_content, prompt, existing_percentages=None):
    try:
        model = genai.GenerativeModel("gemini-1.5-flash")
        response = model.generate_content([input_text, pdf_content[0], prompt])
        logging.info(f"Gemini API Response: {response.text}")
        
        # Extract base percentage from Gemini's response
        try:
            base_percentage = float(re.search(r"(\d+(\.\d+)?)%", response.text).group(1))
        except (AttributeError, ValueError):
            base_percentage = 0
        
        # Adjust percentage to ensure uniqueness
        if existing_percentages is not None:
            adjusted_percentage = base_percentage
            while adjusted_percentage in existing_percentages:
                variation = random.uniform(0.1, 0.9)  # Add small variation
                adjusted_percentage = round(base_percentage + variation, 2)
                adjusted_percentage = max(0, min(100, adjusted_percentage))  # Keep within 0-100
            percentage = adjusted_percentage
        else:
            percentage = base_percentage
        
        # Ensure all instances of base percentage are replaced
        modified_response = re.sub(rf"\b{base_percentage}%", f"{percentage:.2f}%", response.text)

        return modified_response, percentage

    except Exception as e:
        logging.error(f"Error in get_gemini_response: {e}")
        st.error(f"Gemini API Error: {e}")
        return f"Error: {e}", 0
# Authenticate a user
def authenticate_user(username, password, role):
    hashed_password = hash_password(password)
    cursor.execute("SELECT * FROM users WHERE username = %s AND password = %s AND role = %s",
                   (username, hashed_password, role))
    return cursor.fetchone()

# Function to calculate file hash for duplicate detection
def calculate_file_hash(uploaded_file):
    file_content = uploaded_file.read()
    uploaded_file.seek(0)  # Reset file pointer
    return hashlib.md5(file_content).hexdigest()

# Modified get_gemini_response function with unique percentages
def get_gemini_response(input_text, pdf_content, prompt, existing_percentages=None):
    try:
        model = genai.GenerativeModel("gemini-1.5-flash")
        response = model.generate_content([input_text, pdf_content[0], prompt])
        logging.info(f"Gemini API Response: {response.text}")
        
        # Extract base percentage
        try:
            base_percentage = float(response.text.split('%')[0].split()[-1])
        except (IndexError, ValueError):
            base_percentage = 0
        
        # Adjust percentage to ensure uniqueness
        if existing_percentages is not None:
            adjusted_percentage = base_percentage
            while adjusted_percentage in existing_percentages:
                # Add small random variation (0.1 to 0.9)
                variation = random.uniform(0.1, 0.9)
                adjusted_percentage = round(base_percentage + variation, 2)
                # Ensure it stays within 0-100 range
                adjusted_percentage = max(0, min(100, adjusted_percentage))
            percentage = adjusted_percentage
        else:
            percentage = base_percentage
        
        # Update response text with adjusted percentage
        modified_response = response.text.replace(
            f"{base_percentage}%", 
            f"{percentage:.2f}%"
        )
        
        return modified_response, percentage
    
    except Exception as e:
        logging.error(f"Error in get_gemini_response: {e}")
        st.error(f"Gemini API Error: {e}")
        return f"Error: {e}", 0

# Role-Based Access Control
ROLES = {
    "admin": ["upload_resume", "view_all_resumes", "manage_job_description", "delete_resume"],
    "user": ["upload_resume", "view_own_resumes"]
}

# Check if a user has permission to perform an action
def has_permission(user_role, action):
    return action in ROLES.get(user_role, [])

# Define domain-specific keywords with weights
DOMAIN_KEYWORDS = {
    "Software Engineer": {
        "Python": 10,
        "Java": 8,
        "C++": 7,
        "JavaScript": 9,
        "Data Structures": 9,
        "Algorithms": 9,
        "System Design": 8,
        "Git": 7,
        "CI/CD": 7,
        "Docker": 7,
        "Kubernetes": 7,
        "Microservices": 8,
        "REST APIs": 8,
        "Cloud Computing": 8,
        "AWS": 8,
        "Azure": 7,
        "GCP": 7,
        "SQL": 7,
        "NoSQL": 7,
        "DevOps": 7,
        "Agile": 6,
        "Scrum": 6,
        "Test-Driven Development (TDD)": 6
    },
    "Java Developer": {
        "Java": 10,
        "Spring Boot": 9,
        "Spring MVC": 8,
        "Spring Security": 8,
        "Hibernate": 7,
        "JPA": 7,
        "Microservices": 8,
        "J2EE": 7,
        "Servlets": 6,
        "JSP": 6,
        "Multithreading": 7,
        "Concurrency": 7,
        "Design Patterns": 8,
        "Maven": 7,
        "Gradle": 7,
        "JUnit": 7,
        "Mockito": 7,
        "Kafka": 7,
        "RabbitMQ": 7,
        "Docker": 7,
        "Kubernetes": 7,
        "AWS Lambda": 7,
        "Cloud Functions": 7,
        "SQL": 7,
        "MySQL": 7,
        "PostgreSQL": 7,
        "MongoDB": 7
    },
    "Data Scientist and Analytics": {
        "Python": 10,
        "R": 8,
        "SQL": 9,
        "Pandas": 9,
        "NumPy": 9,
        "Matplotlib": 8,
        "Seaborn": 8,
        "Machine Learning": 10,
        "Deep Learning": 9,
        "NLP": 8,
        "Computer Vision": 8,
        "Scikit-learn": 9,
        "TensorFlow": 9,
        "Keras": 8,
        "PyTorch": 8,
        "Big Data": 8,
        "Hadoop": 7,
        "Spark": 8,
        "Hive": 7,
        "ETL": 8,
        "Data Pipelines": 8,
        "Data Cleaning": 8,
        "Feature Engineering": 9,
        "Predictive Modeling": 9,
        "Tableau": 8,
        "Power BI": 8,
        "Google Data Studio": 7,
        "Statistics": 8,
        "Probability": 8,
        "A/B Testing": 7,
        "Hypothesis Testing": 7
    },
    "Project Manager": {
        "Project Management": 10,
        "Agile": 9,
        "Scrum": 9,
        "Kanban": 8,
        "Waterfall": 7,
        "Risk Management": 8,
        "Stakeholder Management": 8,
        "Resource Allocation": 8,
        "Budgeting": 8,
        "Scheduling": 8,
        "Gantt Charts": 7,
        "JIRA": 8,
        "Trello": 7,
        "Asana": 7,
        "Microsoft Project": 7,
        "Leadership": 9,
        "Team Management": 9,
        "Communication Skills": 9,
        "Negotiation": 8,
        "Conflict Resolution": 8,
        "Quality Assurance": 8,
        "Six Sigma": 7,
        "Lean Management": 7,
        "Strategic Planning": 8,
        "Business Analysis": 8,
        "Requirements Gathering": 8,
        "Feasibility Studies": 7,
        "Procurement": 7,
        "Contract Management": 7
    }
}

# Function to extract keywords from text using basic string matching
def extract_keywords(text):
    words = re.findall(r'\b\w+\b', text.lower())
    stop_words = set(["and", "the", "of", "in", "to", "a", "for", "on", "with", "as", "by", "at", "an", "be", "is", "are", "was", "were", "it", "that", "this", "have", "has", "had", "will", "would", "should", "can", "could", "may", "might", "must", "shall", "which", "what", "when", "where", "who", "whom", "whose", "how", "why", "if", "then", "else", "while", "until", "because", "since", "although", "though", "unless", "whether", "either", "neither", "both", "each", "every", "any", "all", "some", "none", "no", "not", "only", "just", "also", "even", "still", "already", "yet", "so", "such", "than", "too", "very", "much", "many", "few", "more", "most", "less", "least", "own", "other", "another", "same", "different", "like", "as", "about", "after", "before", "between", "into", "through", "during", "above", "below", "from", "up", "down", "out", "over", "under", "again", "further", "once", "here", "there", "where", "why", "how", "all", "any", "both", "each", "few", "more", "most", "other", "some", "such", "no", "nor", "not", "only", "own", "same", "so", "than", "too", "very", "s", "t", "can", "will", "just", "don", "should", "now"])
    keywords = [word for word in words if word not in stop_words]
    return set(keywords)

# Function to extract email ID from resume text
def extract_email(resume_text):
    email_pattern = r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
    emails = re.findall(email_pattern, resume_text)
    if emails:
        return emails[0]
    return "Not specified"

# Function to extract experience from resume text
def extract_experience(resume_text):
    experience_patterns = re.findall(r'\b(\d+\+?\s*(?:years?|yrs?))\b', resume_text, re.IGNORECASE)
    if experience_patterns:
        return experience_patterns[0]
    return "Not specified"

# Function to extract skills from resume text
def extract_skills(resume_text):
    skills_match = re.search(r"Skills:?\s*(.*?)(?:\n|$)", resume_text, re.IGNORECASE)
    if skills_match:
        return skills_match.group(1)
    return "Not specified"

# Function to process PDF files and extract text
def input_pdf_setup(uploaded_file):
    try:
        if uploaded_file.size == 0:
            raise ValueError("Uploaded file is empty.")

        file_content = uploaded_file.read()

        if not file_content:
            raise ValueError("File content is empty.")

        images = pdf2image.convert_from_bytes(file_content)

        if not images:
            raise ValueError("No pages found in the PDF.")

        first_page = images[0]
        img_byte_arr = io.BytesIO()
        first_page.save(img_byte_arr, format='JPEG')
        img_byte_arr = img_byte_arr.getvalue()

        pdf_parts = [{
            "mime_type": "image/jpeg",
            "data": base64.b64encode(img_byte_arr).decode()
        }]

        logging.info("PDF processed successfully.")
        return pdf_parts

    except Exception as e:
        logging.error(f"Error in input_pdf_setup: {e}")
        st.error(f"Failed to process PDF: {e}")
        return None

# Function to handle duplicate names
def handle_duplicate_names(name, existing_names):
    if name not in existing_names:
        return name
    count = 1
    while f"{count}-{name}" in existing_names:
        count += 1
    return f"{count}-{name}"

# Function to generate a unique resume name
def generate_unique_resume_name(user_id, uploaded_file, existing_names):
    base_name = uploaded_file.name.replace(".pdf", "").replace(" ", "_")
    
    if base_name not in existing_names:
        return base_name
    count = 1
    while f"{count}-{base_name}" in existing_names:
        count += 1
    return f"{count}-{base_name}"

# Main App Interface
st.title("ATS Tracking System")

# Initialize session state for authentication
if "authenticated" not in st.session_state:
    st.session_state["authenticated"] = False
    st.session_state["user_role"] = None
    st.session_state["user_id"] = None
    st.session_state["username"] = None

# Login/Signup Selection - Using Radio Buttons
auth_option = st.sidebar.radio("Select Option", ["Login", "Sign Up"])

# Sign Up Page
if auth_option == "Sign Up":
    st.subheader("Create a New Account")
    new_username = st.text_input("Username")
    new_password = st.text_input("Password", type="password")
    new_email = st.text_input("Email")
    role = st.selectbox("Select Role", ["user", "admin"])
    if new_password:
        is_strong, feedback = is_password_strong(new_password)
        st.write(feedback)
        if not is_strong:
            st.error("Please choose a stronger password.")
    if st.button("Register"):
        if register_user(new_username, new_password, new_email, role):
            st.success("Account created successfully! Please login.")
        else:
            st.error("Registration failed. Please try again.")

# Login Page
elif auth_option == "Login":
    st.subheader("Login to Your Account")
    username = st.text_input("Username")
    password = st.text_input("Password", type="password")
    
    # Using radio buttons for role selection
    role = st.radio("Select Role", ["User", "Admin"], index=0)
    
    if st.button("Login"):
        user = authenticate_user(username, password, role.lower())
        if user:
            st.session_state["authenticated"] = True
            st.session_state["username"] = username
            st.session_state["user_role"] = role.lower()
            st.session_state["user_id"] = user[0]
            st.success(f"{role} login successful!")
        else:
            st.error("Invalid credentials or incorrect role selected.")

# Logout Function
def logout():
    st.session_state.clear()
    st.success("Logged out successfully!")

# Main App Interface
if st.session_state.get("authenticated", False):
    st.subheader(f"Welcome, {st.session_state['username']} ({st.session_state['user_role']})")
    if st.button("Logout"):
        logout()
        st.rerun()

    # Admin Section
    if st.session_state["user_role"] == "admin":
        st.subheader("Admin Dashboard")

        # Upload Job Description
        st.write("### Manage Job Description")
        allowed_domains = ["Software Engineer", "Java Developer", "Data Scientist and Analytics", "Project Manager"]
        job_title = st.selectbox("Select Job Title", allowed_domains, key="job_title")
        job_description = st.text_area("Job Description", key="job_description")

        if st.button("Save Job Description"):
            if job_description:
                cursor.execute("INSERT INTO job_descriptions (title, description) VALUES (%s, %s)", (job_title, job_description))
                conn.commit()
                st.success("Job Description Saved!")
            else:
                st.error("Please provide a job description.")

        # View and Delete Job Descriptions
        st.write("### View/Delete Job Descriptions")
        cursor.execute("SELECT * FROM job_descriptions WHERE title IN (%s, %s, %s, %s)", tuple(allowed_domains))
        job_descriptions = cursor.fetchall()

        for job in job_descriptions:
            st.write(f"**Title:** {job[1]}")
            st.write(f"**Description:** {job[2]}")
            st.write(f"**Created At:** {job[3]}")

            if st.button(f"Delete Job Description {job[0]}"):
                cursor.execute("DELETE FROM job_descriptions WHERE id = %s", (job[0],))
                conn.commit()
                st.success(f"Job Description {job[0]} Deleted!")

        # Upload Multiple Resumes
        st.write("### Upload Multiple Resumes")
        uploaded_files = st.file_uploader("Upload Resumes (PDF)...", type=["pdf"], accept_multiple_files=True)

        # Evaluate Resumes with Progress Bar
        if st.button("Evaluate Resumes"):
            if uploaded_files:
                # Track existing percentages and file hashes
                existing_percentages = set()
                existing_hashes = set()
                
                # Get existing file hashes from database
                cursor.execute("SELECT file_hash FROM resumes")
                for row in cursor.fetchall():
                    existing_hashes.add(row[0])
                
                # Filter out invalid and duplicate files
                valid_files = []
                duplicate_files = []
                invalid_files = []
                
                for file in uploaded_files:
                    if file.type != "application/pdf" or file.size == 0:
                        invalid_files.append(file)
                        continue
                    
                    file_hash = calculate_file_hash(file)
                    if file_hash in existing_hashes:
                        duplicate_files.append(file)
                    else:
                        valid_files.append(file)
                        existing_hashes.add(file_hash)
                
                # Display warnings
                for file in invalid_files:
                    st.warning(f"Invalid file skipped: {file.name} (Type: {file.type}, Size: {file.size} bytes)")
                
                for file in duplicate_files:
                    st.warning(f"Duplicate file skipped: {file.name} (Already exists in system)")
                
                if valid_files:
                    # Fetch the latest job description
                    cursor.execute("SELECT * FROM job_descriptions ORDER BY created_at DESC LIMIT 1")
                    job_description = cursor.fetchone()
                    if job_description:
                        # Create progress elements
                        progress_bar = st.progress(0)
                        status_text = st.empty()
                        results = []
                        existing_names = set()
                        
                        total_files = len(valid_files)
                        processed_files = 0
                        
                        for uploaded_file in valid_files:
                            # Update progress
                            processed_files += 1
                            progress_percentage = int((processed_files / total_files) * 100)
                            progress_bar.progress(progress_percentage)
                            status_text.text(f"Processing {processed_files} of {total_files} resumes... ({progress_percentage}%)")
                            
                            # Generate a unique resume name
                            unique_resume_name = generate_unique_resume_name(st.session_state["user_id"], uploaded_file, existing_names)
                            existing_names.add(unique_resume_name)

                            pdf_content = input_pdf_setup(uploaded_file)
                            if pdf_content:
                                # Use the job title to select the correct domain keywords
                                selected_domain = job_description[1]
                                domain_keywords = DOMAIN_KEYWORDS.get(selected_domain, {})
                                updated_prompt = f"""
                                You are an ATS system. Evaluate the resume based on the job description and provide a match percentage, 
                                missing keywords, and a concluding summary. Focus on the following keywords for the selected domain:
                                {', '.join(domain_keywords.keys())}
                                """
                                
                                # Get response with unique percentage
                                response, percentage = get_gemini_response(
                                    job_description[2], 
                                    pdf_content, 
                                    updated_prompt,
                                    existing_percentages
                                )
                                existing_percentages.add(percentage)
                                
                                # Extract details from response
                                email = extract_email(response)
                                experience = extract_experience(response)
                                skills = extract_skills(response)
                                file_hash = calculate_file_hash(uploaded_file)
                                
                                # Save to database
                                cursor.execute("""
                                    INSERT INTO resumes 
                                    (user_id, name, email, phone, skills, experience, match_percentage, file_hash) 
                                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
                                """, (
                                    st.session_state["user_id"], 
                                    unique_resume_name, 
                                    email, 
                                    "Not specified", 
                                    skills, 
                                    experience, 
                                    percentage,
                                    file_hash
                                ))
                                conn.commit()
                                results.append((unique_resume_name, percentage, response))
                                
                                # Add 10-second delay between processing each resume
                                if processed_files < total_files:
                                    for i in range(10, 0, -1):
                                        status_text.text(f"Processing {processed_files} of {total_files} resumes... ({progress_percentage}%) - Next resume in {i} seconds")
                                        time.sleep(1)
                        
                        # Complete the progress
                        progress_bar.progress(100)
                        status_text.text("Evaluation complete!")
                        
                        # Display ranked results
                        ranked_results = sorted(results, key=lambda x: x[1], reverse=True)
                        st.subheader("Ranked Resumes")
                        for rank, (name, percentage, response) in enumerate(ranked_results, 1):
                            st.write(f"<span style='font-size:24px; font-weight:bold; color:blue;'>Rank {rank}</span>: {name} - Match: {percentage:.2f}%", unsafe_allow_html=True)
                            st.write("### Detailed Feedback:")
                            st.write(response)
                            if rank == 1:
                                st.success(f"🎉 Congratulations! {name} is eligible for this job.")
                    else:
                        st.error("No job description found. Please add a job description first.")
                else:
                    st.error("No valid PDF files found. Please upload at least one valid PDF.")
            else:
                st.error("Please upload at least one resume.")

    # User Section
    elif st.session_state["user_role"] == "user":
        st.subheader("User Dashboard")

        # Allow the user to upload a resume
        st.write("### Upload Your Resume")
        uploaded_file = st.file_uploader("Choose a PDF file", type=["pdf"])

        if uploaded_file is not None:
            if uploaded_file.type != "application/pdf":
                st.error("Please upload a valid PDF file.")
            else:
                st.write(f"Selected file: {uploaded_file.name}")

                # Check for duplicate resume
                file_hash = calculate_file_hash(uploaded_file)
                cursor.execute("SELECT * FROM resumes WHERE file_hash = %s AND user_id = %s", 
                             (file_hash, st.session_state["user_id"]))
                if cursor.fetchone():
                    st.warning("This resume has already been uploaded and evaluated.")
                else:
                    # Generate a unique resume name
                    existing_names = set()
                    unique_resume_name = generate_unique_resume_name(st.session_state["user_id"], uploaded_file, existing_names)

                    if st.button("Evaluate Resume"):
                        with st.spinner("Analyzing your resume..."):
                            pdf_content = input_pdf_setup(uploaded_file)
                            if pdf_content:
                                # Fetch the latest job description
                                cursor.execute("SELECT * FROM job_descriptions ORDER BY created_at DESC LIMIT 1")
                                job_description = cursor.fetchone()
                                if job_description:
                                    # Use the job title to select the correct domain keywords
                                    selected_domain = job_description[1]
                                    domain_keywords = DOMAIN_KEYWORDS.get(selected_domain, {})
                                    updated_prompt = f"""
                                    You are an ATS system. Evaluate the resume based on the job description and provide:
                                    1. A match percentage (0-100%)
                                    2. Missing keywords from the resume
                                    3. A detailed summary of strengths and weaknesses
                                    4. Suggestions for improvement
                                    
                                    Focus on these domain-specific keywords:
                                    {', '.join(domain_keywords.keys())}
                                    """
                                    
                                    # Get existing percentages for this user
                                    cursor.execute("SELECT match_percentage FROM resumes WHERE user_id = %s", 
                                                 (st.session_state["user_id"],))
                                    existing_percentages = {row[0] for row in cursor.fetchall()}
                                    
                                    response, percentage = get_gemini_response(
                                        job_description[2], 
                                        pdf_content, 
                                        updated_prompt,
                                        existing_percentages
                                    )
                                    
                                    # Extract details from response
                                    email = extract_email(response)
                                    experience = extract_experience(response)
                                    skills = extract_skills(response)

                                    # Clear previous results
                                    st.empty()
                                    
                                    # Display current evaluation results
                                    st.subheader("Evaluation Results")
                                    st.write(f"**Match Score:** {percentage:.2f}%")
                                    st.write("**Detailed Analysis:**")
                                    st.write(response)

                                    # Save the results to database
                                    cursor.execute("""
                                        INSERT INTO resumes 
                                        (user_id, name, email, phone, skills, experience, match_percentage, file_hash) 
                                        VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
                                    """, (
                                        st.session_state["user_id"], 
                                        unique_resume_name, 
                                        email, 
                                        "Not specified", 
                                        skills, 
                                        experience, 
                                        percentage,
                                        file_hash
                                    ))
                                    conn.commit()
                                    
                                    st.success("Evaluation completed successfully!")
                                else:
                                    st.error("No job description found. Please contact admin.")
                            else:
                                st.error("Failed to process the PDF file.")

else:
    st.warning("Please log in to access the system.")