import streamlit as st
import requests
import json
from ddgs import DDGS



#CONFIG
HF_TOKEN = ""


HF_API_URL = "https://api-inference.huggingface.co/models/meta-llama/Llama-3.1-8B-Instruct"



HEADERS = {
    "Authorization": f"Bearer {"key"}",
    "Content-Type": "application/json"
}

# Search tool
def search_web(query):
    results = []
    with DDGS() as ddgs:
        for r in ddgs.text(query, max_results=3):
            results.append(r["body"])
    return "\n".join(results)
# LLM call

def call_llm(prompt):
    payload = {
        "model": "meta-llama/Llama-3.1-8B-Instruct",
        "messages": [
            {"role": "system", "content": "You are a helpful travel planning assistant."},
            {"role": "user", "content": prompt}
        ],
        "temperature": 0.7,
        "max_tokens": 500
    }

    try:
        response = requests.post(
            "https://router.huggingface.co/v1/chat/completions",
            headers={
                "Authorization": f"Bearer {HF_TOKEN}",
                "Content-Type": "application/json"
            },
            json=payload,
            timeout=60
        )
    except requests.exceptions.RequestException as e:
        return f" Network error: {e}"

    if response.status_code != 200:
        return f"HTTP {response.status_code}: {response.text}"

    try:
        data = response.json()
        return data["choices"][0]["message"]["content"]
    except Exception as e:
        return f"Parse error: {e}\nRaw: {response.text}"


#                  STREAMLIT UI 
st.set_page_config(page_title="AI Travel Agent", layout="centered")

st.title("🧳 AI Travel Agent")
st.write("Plan trips with live web search support")

user_query = st.text_input(
    "Where do you want to go?",
    placeholder="Example: 3-day budget trip to Manali under ₹20,000"
)

use_search = st.checkbox("Use live web search (weather, places, tips)")

if st.button("Generate Plan"):
    if not user_query:
        st.warning("Please enter a travel query.")
    else:
        with st.spinner("Thinking..."):
            context = ""

            if use_search:
                search_results = search_web(user_query)
                context = f"\n\nHere is recent web information:\n{search_results}\n'

            final_prompt = f"""
You are a smart travel planning assistant.

User request:
{user_query}

{context}

Respond with:
- Day-wise itinerary
- Budget estimate
- Transport tips
- Local food suggestions
"""

            answer = call_llm(final_prompt)

        st.success("Here’s your travel plan:")
        st.write(answer)
