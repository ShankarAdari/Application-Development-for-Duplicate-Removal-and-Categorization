# Implementation Guide: Job Portal with AI-based Resume Screening & Career Counselling

## 1. Introduction
This guide provides a practical implementation approach for a Job Portal with AI-based Resume Screening and Career Counselling functionalities. We will focus on key AI/ML components using Python, including resume parsing, information extraction, and basic matching. For the backend, we will use Flask, and for the AI/ML parts, we will leverage libraries like `spaCy` and `scikit-learn`. This guide will provide foundational code snippets for these complex functionalities.

## 2. Setting Up the Environment

### 2.1 Install Required Libraries
Open your terminal or command prompt and run the following commands to install the necessary Python libraries:

```bash
pip install Flask Flask-SQLAlchemy psycopg2-binary spacy scikit-learn numpy pandas
python -m spacy download en_core_web_sm # Download a small English language model for spaCy
```

*   `Flask`: Web framework for the backend.
*   `Flask-SQLAlchemy`: ORM for database interaction.
*   `psycopg2-binary`: PostgreSQL adapter.
*   `spaCy`: For advanced NLP tasks like Named Entity Recognition (NER) and tokenization.
*   `scikit-learn`: For machine learning utilities, including vectorization and similarity calculations.
*   `numpy`, `pandas`: For data manipulation.

### 2.2 Database Setup (PostgreSQL)

1.  **Install and run PostgreSQL.**
2.  **Create a database for the Job Portal:**
    ```sql
    CREATE DATABASE job_portal;
    ```
3.  **Define tables (simplified for this guide):**
    ```sql
    CREATE TABLE job_postings (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        title VARCHAR(255) NOT NULL,
        description TEXT NOT NULL,
        requirements TEXT NOT NULL,
        location VARCHAR(100),
        posted_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    CREATE TABLE candidates (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name VARCHAR(255) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL,
        skills TEXT, -- Comma-separated for simplicity
        experience TEXT, -- Simplified text field
        education TEXT, -- Simplified text field
        resume_text TEXT -- Extracted text from resume
    );

    CREATE TABLE applications (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        job_id UUID REFERENCES job_postings(id),
        candidate_id UUID REFERENCES candidates(id),
        applied_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        resume_score FLOAT
    );
    ```

## 3. Core Components Implementation

### 3.1 Resume Parsing and Information Extraction
This is a complex task. For this guide, we will simulate resume parsing by assuming we have the resume content as plain text. We will use `spaCy` to extract skills and other entities.

```python
import spacy

nlp = spacy.load("en_core_web_sm")

def extract_info_from_resume(resume_text):
    doc = nlp(resume_text)

    # Extracting skills (simplified: looking for common skill keywords)
    # In a real system, you'd use a more sophisticated approach, e.g., rule-based matching
    # against a predefined skill taxonomy or a custom NER model.
    skills = []
    skill_keywords = ["python", "java", "sql", "machine learning", "data analysis", "flask", "django", "react", "aws", "azure", "gcp"]
    for token in doc:
        if token.text.lower() in skill_keywords:
            skills.append(token.text.lower())
    skills = list(set(skills)) # Remove duplicates

    # Extracting entities like names, organizations, GPE (geopolitical entities - locations)
    name = ""
    email = ""
    for ent in doc.ents:
        if ent.label_ == "PERSON":
            name = ent.text
        elif ent.label_ == "ORG" and "@" in resume_text: # Simple heuristic for email
            # This is a very basic way to find email, a regex would be better
            import re
            email_match = re.search(r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b", resume_text)
            if email_match:
                email = email_match.group(0)

    # In a real system, you'd extract experience, education, etc. more robustly
    experience = "Extracted experience summary..."
    education = "Extracted education summary..."

    return {
        "name": name,
        "email": email,
        "skills": ", ".join(skills),
        "experience": experience,
        "education": education,
        "resume_text": resume_text
    }

# Example Usage:
# sample_resume_text = """
# John Doe
# johndoe@example.com
# Experienced Python Developer with a strong background in Machine Learning and Data Analysis.
# Proficient in Flask and Django. Worked on AWS cloud platforms.
# """
# extracted_data = extract_info_from_resume(sample_resume_text)
# print(extracted_data)
```

### 3.2 AI-based Resume Scoring
We will use TF-IDF (Term Frequency-Inverse Document Frequency) to vectorize job requirements and resume content, then calculate cosine similarity to get a match score.

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

def calculate_resume_score(job_requirements_text, resume_text):
    # Combine texts for TF-IDF vectorization
    documents = [job_requirements_text, resume_text]

    # Initialize TF-IDF Vectorizer
    # You might want to add stop words, n-grams, etc. for better results
    vectorizer = TfidfVectorizer()

    # Fit and transform the documents
    tfidf_matrix = vectorizer.fit_transform(documents)

    # Calculate cosine similarity between the job requirements and the resume
    # The similarity is between the first document (job_requirements) and the second (resume)
    cosine_sim = cosine_similarity(tfidf_matrix[0:1], tfidf_matrix[1:2])[0][0]

    # You can scale the similarity score to a 0-100 range if desired
    score = cosine_sim * 100

    return score

# Example Usage:
# job_req = "Requires strong Python, machine learning, and data analysis skills."
# candidate_resume = "Proficient in Python, SQL, and data analysis. Some experience with ML."
# score = calculate_resume_score(job_req, candidate_resume)
# print(f"Resume Match Score: {score:.2f}")
```

### 3.3 Flask Backend for Job Portal

Create `app.py` for the Flask application:

```python
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
import uuid
import datetime

# Assuming extract_info_from_resume and calculate_resume_score are defined above

app = Flask(__name__)
app.config["SQLALCHEMY_DATABASE_URI"] = "postgresql://user:password@host:port/job_portal"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False
db = SQLAlchemy(app)

# Database Models (simplified for brevity, match your SQL schema)
class JobPosting(db.Model):
    __tablename__ = "job_postings"
    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    title = db.Column(db.String(255), nullable=False)
    description = db.Column(db.Text, nullable=False)
    requirements = db.Column(db.Text, nullable=False)
    location = db.Column(db.String(100))
    posted_date = db.Column(db.DateTime, default=datetime.datetime.utcnow)

class Candidate(db.Model):
    __tablename__ = "candidates"
    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    name = db.Column(db.String(255), nullable=False)
    email = db.Column(db.String(255), unique=True, nullable=False)
    skills = db.Column(db.Text)
    experience = db.Column(db.Text)
    education = db.Column(db.Text)
    resume_text = db.Column(db.Text)

class Application(db.Model):
    __tablename__ = "applications"
    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    job_id = db.Column(db.String(36), db.ForeignKey("job_postings.id"))
    candidate_id = db.Column(db.String(36), db.ForeignKey("candidates.id"))
    applied_date = db.Column(db.DateTime, default=datetime.datetime.utcnow)
    resume_score = db.Column(db.Float)

# Create tables (run once, or use Flask-Migrate)
# with app.app_context():
#     db.create_all()

@app.route("/job_postings", methods=["POST"])
def create_job_posting():
    data = request.get_json()
    new_job = JobPosting(
        title=data["title"],
        description=data["description"],
        requirements=data["requirements"],
        location=data.get("location")
    )
    db.session.add(new_job)
    db.session.commit()
    return jsonify({"message": "Job posting created", "id": new_job.id}), 201

@app.route("/candidates", methods=["POST"])
def create_candidate_profile():
    data = request.get_json()
    resume_content = data.get("resume_content", "") # Assume plain text resume content
    extracted_info = extract_info_from_resume(resume_content)

    new_candidate = Candidate(
        name=extracted_info.get("name", data.get("name", "Unknown")),
        email=extracted_info.get("email", data["email"]),
        skills=extracted_info.get("skills"),
        experience=extracted_info.get("experience"),
        education=extracted_info.get("education"),
        resume_text=extracted_info.get("resume_text")
    )
    db.session.add(new_candidate)
    db.session.commit()
    return jsonify({"message": "Candidate profile created", "id": new_candidate.id}), 201

@app.route("/apply", methods=["POST"])
def apply_for_job():
    data = request.get_json()
    job_id = data["job_id"]
    candidate_id = data["candidate_id"]

    job = JobPosting.query.get(job_id)
    candidate = Candidate.query.get(candidate_id)

    if not job or not candidate:
        return jsonify({"error": "Job or Candidate not found"}), 404

    # Calculate resume score
    resume_score = calculate_resume_score(job.requirements, candidate.resume_text)

    new_application = Application(
        job_id=job_id,
        candidate_id=candidate_id,
        resume_score=resume_score
    )
    db.session.add(new_application)
    db.session.commit()

    return jsonify({"message": "Application submitted", "resume_score": f"{resume_score:.2f}"}), 201

@app.route("/job_postings", methods=["GET"])
def get_job_postings():
    jobs = JobPosting.query.all()
    return jsonify([{"id": job.id, "title": job.title, "location": job.location} for job in jobs])

@app.route("/job_postings/<job_id>/applicants", methods=["GET"])
def get_job_applicants(job_id):
    applications = Application.query.filter_by(job_id=job_id).order_by(Application.resume_score.desc()).all()
    results = []
    for app in applications:
        candidate = Candidate.query.get(app.candidate_id)
        results.append({
            "candidate_name": candidate.name,
            "candidate_email": candidate.email,
            "resume_score": f"{app.resume_score:.2f}"
        })
    return jsonify(results)

if __name__ == "__main__":
    app.run(debug=True)
```

## 4. Career Counselling (Conceptual)

Career counselling involves more complex AI models, often leveraging large language models (LLMs) or specialized knowledge graphs. Here's a conceptual outline and a simple example of skill gap analysis.

### 4.1 Skill Gap Analysis (Simple Example)

```python
def analyze_skill_gap(candidate_skills_text, target_job_requirements_text):
    # Convert skills and requirements to sets for easy comparison
    candidate_skills = set([s.strip().lower() for s in candidate_skills_text.split(",") if s.strip()])
    target_requirements = set([s.strip().lower() for s in target_job_requirements_text.split(",") if s.strip()])

    missing_skills = target_requirements - candidate_skills
    present_skills = candidate_skills.intersection(target_requirements)

    recommendations = []
    if missing_skills:
        recommendations.append(f"Consider acquiring these skills: {\", \".join(missing_skills)}.")
        # In a real system, you'd suggest courses/resources for these skills

    if present_skills:
        recommendations.append(f"You already have these relevant skills: {\", \".join(present_skills)}.")

    return " ".join(recommendations) if recommendations else "No specific skill gaps identified based on the provided information."

# Example Usage:
# candidate_skills = "Python, SQL, Data Analysis"
# desired_job_req = "Python, Machine Learning, Data Analysis, Cloud (AWS)"
# gap_analysis = analyze_skill_gap(candidate_skills, desired_job_req)
# print(f"\nSkill Gap Analysis: {gap_analysis}")
```

### 4.2 Advanced Career Counselling

For a full-fledged career counseling feature, you would integrate:
*   **Large Language Models (LLMs):** To provide conversational career advice, generate personalized learning paths, and simulate interview scenarios.
*   **Knowledge Graphs:** To map skills to job roles, industries, and learning resources, enabling more intelligent recommendations.
*   **Job Market Data:** Integrate with APIs providing real-time data on job trends, salary benchmarks, and in-demand skills.

## 5. Further Enhancements

*   **Robust Resume Parsing:** Integrate with dedicated resume parsing libraries (e.g., `python-docx`, `pdfminer.six`) or commercial APIs for more accurate and comprehensive extraction.
*   **Advanced AI Matching:** Explore more sophisticated NLP models (e.g., fine-tuned BERT models) for semantic matching of resumes and job descriptions.
*   **Recommendation Engine:** Implement a full-fledged recommendation system (collaborative filtering, content-based, or hybrid) for job suggestions.
*   **User Authentication:** Implement secure user registration and login for candidates and recruiters.
*   **Frontend Development:** Build a rich, interactive frontend using React, Angular, or Vue.js to consume the Flask APIs.
*   **Deployment:** Deploy the Flask application and database to a cloud platform (e.g., AWS, GCP, Azure) for production use.
*   **Asynchronous Tasks:** Use a task queue (e.g., Celery with RabbitMQ) for long-running tasks like resume processing to avoid blocking the main web server.

This implementation guide provides a foundational understanding and practical code examples for building a Job Portal with AI-based resume screening and career counseling. The provided snippets can be expanded and integrated into a complete, production-ready system.


