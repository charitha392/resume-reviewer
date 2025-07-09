import os
import io
import base64
import streamlit as st
from dotenv import load_dotenv
from PIL import Image
import pdf2image
import google.generativeai as genai

load_dotenv()
genai.configure(api_key=os.getenv("GOOGLE_API_KEY"))

st.set_page_config(page_title="ATS Resume Expert", layout="centered")

def set_local_background(image_path):
    with open(image_path, "rb") as image:
        encoded = base64.b64encode(image.read()).decode()
        css = f"""
        <style>
        .stApp {{
            background-image: url("data:image/jpg;base64,{encoded}");
            background-size: cover;
            background-attachment: fixed;
            background-repeat: no-repeat;
        }}
        .block-container {{
            background-color: rgba(255, 255, 255, 0.0);
            padding: 2rem;
        }}
        h1, h2, h3, h4, h5 {{
            color: white !important;
        }}
        label {{
            color: black !important;
            font-weight: 600;
        }}
        .stTextArea textarea {{
            background-color: #f0f4f8;
            color: black;
            border-radius: 10px;
        }}
        .stFileUploader {{
            background-color: #e8eef7;
            border: 2px dashed #4682b4;
            padding: 12px;
            border-radius: 12px;
            color: black;
        }}
        .stButton>button {{
            background-color: #1e3a8a;
            color: white;
            font-weight: bold;
            border-radius: 10px;
        }}
        .stButton>button:hover {{
            background-color: #3b82f6;
        }}
        .filename {{
            color: black;
            font-size: 1rem;
            font-weight: 600;
            margin-bottom: 1rem;
        }}
        .translucent-output {{
            background-color: rgba(255, 255, 255, 0.4);
            color: black;
            padding: 1.5rem;
            border-radius: 12px;
            font-size: 1rem;
            backdrop-filter: blur(6px);
        }}
        .centered-buttons .element-container {{
            display: flex;
            justify-content: center;
            gap: 1rem;
            flex-wrap: wrap;
        }}
        </style>
        """
        st.markdown(css, unsafe_allow_html=True)

set_local_background("image.jpg")

def input_pdf_setup(uploaded_file):
    if uploaded_file:
        images = pdf2image.convert_from_bytes(uploaded_file.read())
        first_page = images[0]
        img_byte_arr = io.BytesIO()
        first_page.save(img_byte_arr, format='JPEG')
        encoded_image = base64.b64encode(img_byte_arr.getvalue()).decode("utf-8")
        return {"inline_data": {"mime_type": "image/jpeg", "data": encoded_image}}
    raise FileNotFoundError("No file uploaded")

def get_gemini_response(prompt, resume_part, job_desc):
    model = genai.GenerativeModel('models/gemini-1.5-flash')
    parts = []
    parts.append({"text": prompt})
    if resume_part:
        parts.append(resume_part)
    if job_desc:
        parts.append({"text": job_desc})
    response = model.generate_content(parts)
    return response.text

st.markdown("<h1 style='text-align:center;'>ATS Resume Expert</h1>", unsafe_allow_html=True)
st.markdown("<h4 style='text-align:center; color:white;'>Upload your resume and paste a job description to get a professional analysis and matching insights.</h4>", unsafe_allow_html=True)

input_text = st.text_area("Paste Job Description Here:")
uploaded_file = st.file_uploader("Upload Resume (PDF only)", type=["pdf"])

if uploaded_file:
    st.markdown(f"<p class='filename'>📄 {uploaded_file.name}</p>", unsafe_allow_html=True)

st.markdown("""
<div class="centered-buttons">
""", unsafe_allow_html=True)

col1, col2, col3 = st.columns(3)
with col1:
    submit_match = st.button("Check ATS Match")
with col2:
    submit_questions = st.button("Top Interview Questions")
with col3:
    submit_grammar = st.button("Check Grammar")

st.markdown("""
</div>
""", unsafe_allow_html=True)

input_prompt3 = """
You are a skilled ATS (Applicant Tracking System) scanner with a deep understanding of data science and ATS functionality. 
Your task is to evaluate the resume against the provided job description. First, give a percentage match, 
then list the missing keywords, and finally provide suggestions to improve the resume.
"""

input_prompt_questions = """
You are an expert technical interviewer. Based on the following job description, generate a list of the top 10 technical and behavioral interview questions likely to be asked:
"""

input_prompt_grammar = """
You are a professional proofreader and resume expert. Analyze the following resume content and list all grammatical errors, typos, and suggestions to improve language clarity:
"""

if submit_match:
    if uploaded_file and input_text:
        with st.spinner("Calculating match percentage..."):
            pdf_content = input_pdf_setup(uploaded_file)
            result = get_gemini_response(input_prompt3, pdf_content, input_text)
            st.markdown(f"<div class='translucent-output'>{result}</div>", unsafe_allow_html=True)
    else:
        st.warning("⚠️ Please upload a resume and paste a job description.")

if submit_questions:
    if input_text:
        with st.spinner("Fetching top interview questions..."):
            result = get_gemini_response(input_prompt_questions, {}, input_text)
            st.markdown(f"<div class='translucent-output'>{result}</div>", unsafe_allow_html=True)
    else:
        st.warning("⚠️ Please paste a job description.")

if submit_grammar:
    if uploaded_file:
        with st.spinner("Analyzing grammar in resume..."):
            pdf_content = input_pdf_setup(uploaded_file)
            result = get_gemini_response(input_prompt_grammar, pdf_content, "")
            st.markdown(f"<div class='translucent-output'>{result}</div>", unsafe_allow_html=True)
    else:
        st.warning("⚠️ Please upload a resume.")
