# HR Leave Policy Assistant

A simple AI-powered assistant that answers employee questions **exclusively** from a provided leave policy document. Every answer cites the relevant policy section, anything not covered gets a fixed "I don't know" reply and requests for other employees' personal information are politely refused.

## Features

| Requirement | Behavior |
|---|---|
| **1. Accurate, cited answers** | Answers use only the policy's rules and numbers and every answer shows the policy section it relied on. Nothing is added or inferred. |
| **2. Admits what it doesn't know** | If the policy doesn't cover the question, the assistant replies exactly: *"I don't know, please contact HR."* It does not guess or use general knowledge about leave laws or company practice. |
| **3. Protects personal data** | Requests for another employee's personal information (e.g. leave taken, reasons for leave) are politely declined and redirected to HR. |

## How it works

1. On startup, the app reads the policy PDF from.
2. For every question, the full policy text is sent to the LLM together with strict rules.
3. The model returns a structured result: an outcome (`answered`, `not_covered` or `privacy_refusal`), the answer and the cited sections.
4. The app's own code decides what the user sees:
   - **`not_covered`**: the code outputs the fixed sentence *"I don't know, please contact HR."* (the model does not write it, so the wording is guaranteed)
   - **`answered`**: shown only if every cited section actually exists in the policy text (citation-exists check); otherwise the fixed fallback sentence is used
   - **`privacy_refusal`**: a courteous refusal is shown

The user never uploads anything. The pdf sits at the same folder as the app. The policy is loaded once and sent automatically with each question.

## Design decisions

- **Full-context instead of RAG.** The policy is only six pages (a few thousand tokens) so it fits entirely in the model's context. Sending the whole document avoids retrieval errors (a search step missing a relevant section), handles questions that span several sections and keeps citations simple. RAG would add an embedding model, a vector store and tuning without any benefit at this size. I would switch to RAG if the policy grew into a large multi-document library.
- **Single file.** The whole assistant lives in `app.py` to keep it easy to run, read and review.
- **No database.** Nothing is stored.
- **No chat memory beyond the current session.** Each question is answered independently. The chat history shown on screen exists only for display and is cleared when the page is reloaded.
- **No extra frameworks.** Only Streamlit (UI), the Google Gen AI SDK, `pypdf` and `python-dotenv`.
- **Citation-exists check.** A few lines of code verify that every section the model cites really appears in the policy text, guarding against made-up citations.
- **Temperature 0** for consistent, repeatable answers.

## Tech stack

- Python 3.10+
- [Streamlit](https://streamlit.io/): chat interface
- [Google Gen AI SDK](https://pypi.org/project/google-genai/) with a Gemini Flash model (free tier)
- [pypdf](https://pypi.org/project/pypdf/): PDF text extraction
- [python-dotenv](https://pypi.org/project/python-dotenv/): loading settings from `.env`

## Project structure

```
app.py                # the entire assistant
requirements.txt      # Python dependencies
test_questions.txt    # questions used to test the assistant
.env.example          # template for your API key and model name
.gitignore            # keeps .env and the policy PDF out of Git
README.md
policy.pdf            # not included: you add the policy file here           
```

## Setup

### 1. Prerequisites
- Python 3.10 or newer
- Any LLM API Key (OpenAI/Gemini/Claude)

### 2. Clone the repo and open it in an IDE (VSCode)
```bash
git clone https://github.com/RaianRashidRimon/HR-Leave-Policy-Assistant.git
```

### 3. Create a virtual environment and install dependencies using the IDE terminal

**Windows**
```bash
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

**macOS / Linux**
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 4. Configure your API key and model
Rename the `.env.example` to `.env`

Open `.env` and fill in both values:

```
API_KEY=your_api_key_here
MODEL=model_id
```

### 5. Add the policy file
The sample leave policy is confidential and not included in this repository. Place your provided copy at the same folder as the repo clone

### 6. Run the app
```bash
streamlit run app.py
```

Streamlit opens the chat interface in your browser.

## Troubleshooting

- **"Set API_KEY and MODEL in your .env file"**: the `.env` file is missing, misnamed (e.g. `.env.txt`), in the wrong folder or has empty values. Restart Streamlit after editing it.
- **"The AI service is busy right now"**: the free tier API can return temporary "high demand" (503) or rate-limit errors. The app retries automatically a few times, so waiting a few seconds and asking again usually works. If it keeps happening, try a different family model in `.env`.
- **First answer is slower than the rest**: the first request includes connection warm-up and possibly a retry. Later requests are faster.
- **Persistent errors**: check the terminal where Streamlit is running, which prints the underlying API error.

## Limitations and future improvements

- **Testing was manual** on a small set of questions, so unusual phrasings or edge cases may still slip through. The citation check confirms a cited section exists in the policy, not that it is the *right* section for the question.
- **With more time**, I would add an automated test suite of 30-50 questions (covered, not covered, privacy, prompt-injection and multi-section cases) with expected outcomes, run after every prompt or model change, to give a measurable accuracy figure.
- Free-tier availability and rate limits are outside the app's control.
