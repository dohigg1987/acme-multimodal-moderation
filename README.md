# ACME Multimodal Content Moderation

**Author:** Dennis O'Higgins

A customer service training application for the fictional company ACME Enterprise. A trainee agent chats with a simulated, initially angry customer (played by a Gemini model) whose ACME Power Widget Pro has stopped working. Every message and file the trainee sends is moderated before it reaches the customer: text, images, video and audio are checked for personally identifiable information (PII), unfriendly or unprofessional tone, disturbing content and low quality. Flagged content is blocked, never sent to the customer, and the trainee sees the moderation rationale.

## Architecture

| Component | File | Role |
|---|---|---|
| Moderation result models | `multimodal_moderation/types/moderation_result.py` | Pydantic output schemas. The base `ModerationResult` holds only the required `rationale`; each modality subclass adds its own required boolean flags. |
| Text agent | `multimodal_moderation/agents/text_agent.py` | Pydantic AI agent returning `TextModerationResult` (`contains_pii`, `is_unfriendly`, `is_unprofessional`). |
| Image agent | `multimodal_moderation/agents/image_agent.py` | Wraps the uploaded bytes in `BinaryContent` and returns `ImageModerationResult` (`contains_pii`, `is_disturbing`, `is_low_quality`). |
| Video and audio agents | `multimodal_moderation/agents/video_agent.py`, `audio_agent.py` | Provided agents for video and audio (audio also returns a transcription). |
| Customer agent | `multimodal_moderation/agents/customer_agent.py` | The LLM that plays the ACME customer. |
| Backend API | `multimodal_moderation/fastapi_app.py` | FastAPI endpoints that wrap the moderation agents, protected by `USER_API_KEY`. |
| Chat front end | `multimodal_moderation/gradio_app.py` | Multimodal `gr.ChatInterface`. Each turn is moderated through the backend; safe content goes to the customer agent with the full message history. |
| Observability | `multimodal_moderation/tracing.py` | OpenTelemetry tracing exported to Arize Phoenix. |
| Evals | `evals/` | Pydantic Evals suites for text, image, video and audio, with rule-based flag checks and an LLM judge for the rationale. |

### Moderation flow in the chat app

1. The trainee sends text and/or files.
2. `check_content_safety` sends each item to the matching backend endpoint inside a `moderate_*` span.
3. If any unsafe flag is set, the turn stops: the message is not sent to the customer, the rationale is shown in the **Moderation Agent Feedback** panel, and the `chat_turn` span records the `feedback` attribute.
4. If everything is safe, the text and files (as `BinaryContent`) are sent to the customer agent with `message_history=past_messages`, so the customer keeps the context of the conversation. The updated history is stored in `past_messages_state`.

### Tracing spans

| Span | Created in | Attributes |
|---|---|---|
| `conversation` | `ChatSessionWithTracing.__init__` (`tracer.start_span`, kept open until **End Conversation**) | `session.id` |
| `chat_turn` | `chat_with_gemini`, child of `conversation` | `session.id`, `input.value`, `input.file_count`, `history.message_count`, `moderation.flagged`, `feedback`, `output.value` |
| `moderate_text` (renamed `moderate_image`, `moderate_video` or `moderate_audio` for media) | `check_content_safety` | input text or media metadata, and every moderation output field |
| `llm_customer` | `chat_with_gemini` | Customer agent call, with the Pydantic AI spans nested underneath |

All spans use the tracer from `tracing.py` (`get_tracer`).

## Setup (Udacity workspace)

```bash
cd /voc/work/code/project/starter/
pip install uv
python -m uv sync
source .venv/bin/activate
uv pip install -e .
cp env.example .env   # then set GEMINI_API_KEY (your voc- key) and USER_API_KEY (a password you invent)
```

On a local machine use `uv sync --dev`, a Google AI Studio key, and remove the `GOOGLE_GEMINI_BASE_URL` line from `.env`. Never commit `.env`.

## Running

```bash
# Unit tests (no Gemini calls)
uv run pytest tests/ -vv --ignore=tests/test_env_setup.py

# Evals (call Gemini; EVAL_NUM_REPEATS in .env sets the number of repeats)
uv run evals/text/test_cases.py
uv run evals/image/test_cases.py
uv run evals/audio/test_cases.py
uv run evals/video/test_cases.py

# App: chat UI on 7860, backend API on 8000, Phoenix on 6006
export PHOENIX_HOST_ROOT_PATH=/proxy/6006   # workspace only
uv run multimodal-moderation
```

## Evals

The text suite covers one acceptable case (`professional_text`) and two unacceptable cases (`text_with_pii`, `unfriendly_text`). The image suite covers one acceptable case (`professional_image`) and two unacceptable cases (`image_with_person`, `low_quality_image`). Each case checks the expected flags with a rule-based evaluator and grades the rationale with an `LLMJudge`; every case also checks the output type and that the rationale is not empty. Because both the agent and the judge are language models, a score below 100% is expected and varies between runs.

## Results and evidence

All runs were made in the Udacity workspace (Python 3.12.12).

| Check | Result |
|---|---|
| Unit tests (`uv run pytest tests/ -vv --ignore=tests/test_env_setup.py`) | 45 passed |
| Text eval (`uv run evals/text/test_cases.py`, 1 repeat) | 91.7% of assertions passed (flags correct on all three cases; the judge rejected one short rationale) |
| Image eval (`uv run evals/image/test_cases.py`, 1 repeat) | 100% |
| Chat app | A rude message was flagged as unfriendly and unprofessional, blocked from the customer agent, and its rationale shown in the feedback panel |
| Phoenix | `conversation` (with `session.id`), `chat_turn` (with `feedback`), `moderate_text` and `llm_customer` spans received |

`RUN_EVIDENCE.pdf` contains the unit test output, the eval output, a screenshot of the flagged conversation with its rationale, and screenshots of the Phoenix traces. The raw outputs and screenshots are also in the `evidence/` folder.
