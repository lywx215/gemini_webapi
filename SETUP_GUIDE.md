# WebAI-to-API Setup Guide

## Overview
This guide documents the setup and configuration of the WebAI-to-API server with Gemini 2.5 Pro support.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Custom Modifications](#custom-modifications)
- [Running the Server](#running-the-server)
- [API Usage](#api-usage)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software
- **Python 3.13** or higher
- **pip** (Python package installer)
- **Chrome browser** (for cookie authentication)

### Required Accounts
- Google account with Gemini access
- Gemini Advanced/Pro subscription (recommended for 2.5 Pro access)

---

## Installation

### 1. Clone the Repository
```bash
git clone https://github.com/Amm1rr/WebAI-to-API.git
cd WebAI-to-API
```

### 2. Install Dependencies
```bash
pip install -r requirements.txt
```

### 3. Install Additional Crypto Libraries
```bash
pip install pycryptodome
```

---

## Configuration

### 1. Create Configuration File
```bash
cp config.conf.example config.conf
```

### 2. Get Gemini Cookies

#### Method 1: Manual Cookie Extraction
1. Open Chrome and navigate to `https://gemini.google.com`
2. Login to your Google account
3. Press `F12` to open Developer Tools
4. Go to **Application** tab → **Cookies** → `https://gemini.google.com`
5. Find and copy these cookies:
   - `__Secure-1PSID`
   - `__Secure-1PSIDTS`

#### Method 2: Automatic Cookie Retrieval
Leave the cookie fields empty in `config.conf` and the application will automatically retrieve them from your browser.

### 3. Configure `config.conf`

```ini
[Browser]
name = chrome

[AI]
default_ai = gemini
default_model_gemini = gemini-2.5-pro

[Cookies]
# Option 1: Manual cookies (recommended)
gemini_cookie_1psid = YOUR_1PSID_COOKIE_HERE
gemini_cookie_1psidts = YOUR_1PSIDTS_COOKIE_HERE

# Option 2: Leave empty for automatic retrieval
# gemini_cookie_1psid =
# gemini_cookie_1psidts =

[EnabledAI]
gemini = true

[Proxy]
# Optional: Add proxy if needed
http_proxy =
```

---

## Custom Modifications

### 1. Added Gemini 2.5 Pro Support

**File Modified**: `C:\Users\{username}\AppData\Roaming\Python\Python313\site-packages\gemini_webapi\constants.py`

**Changes Made**:
Added new model definition in the `Model` enum:

```python
G_2_5_PRO = (
    "gemini-2.5-pro",
    {"x-goog-ext-525001261-jspb": '[1,null,null,null,"4af6c7f5da75d65d",null,null,0,[4]]'},
    True,
)
```

**How to Find the Header ID** (if Google changes it):
1. Open Chrome DevTools (`F12`)
2. Go to **Network** tab
3. Filter by `StreamGenerate`
4. Select Gemini 2.5 Pro model in Gemini
5. Send a test message
6. Click on the `StreamGenerate` request
7. Look for header `x-goog-ext-525001261-jspb`
8. Copy the value and update the model definition

### 2. Fixed Unicode Issues for Windows

**File Modified**: `WebAI-to-API/src/run.py`

**Changes**: Replaced emoji characters with ASCII equivalents to prevent encoding errors on Windows:
- `✅` → `[OK]`
- `⚠️` → `[!!]`
- `🚀` → `[*]`
- `✨` → `[+]`
- `⚙️` → `[=]`
- `🔗` → `[>]`
- `🔍` → `[?]`

### 3. Fixed Multiprocessing Client Initialization

**File Modified**: `WebAI-to-API/src/app/main.py`

**Changes**: Updated the lifespan manager to initialize the Gemini client within the FastAPI subprocess:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Import here to avoid circular dependency
    from app.services.gemini_client import init_gemini_client

    # Initialize the Gemini client in this process
    client_initialized = await init_gemini_client()
    if client_initialized:
        init_session_managers()
        logger.info("Gemini client and session managers initialized for WebAI-to-API.")
    else:
        logger.warning("Gemini client could not be initialized. API endpoints may not work.")

    yield

    logger.info("Application shutdown complete.")
```

---

## Running the Server

### Start the Server
```bash
python src/run.py
```

### Expected Output
```
INFO:     Checking availability of server modes...
INFO:     [OK] WebAI-to-API mode is available (Gemini client initialized).
WARN:     [!!] gpt4free mode is not available ('g4f' library not found).

================================================================================
                              Webai To Api v0.4.0
...
INFO:     Uvicorn running on http://localhost:6969 (Press CTRL+C to quit)
```

### Server Information
- **URL**: `http://localhost:6969`
- **Documentation**: `http://localhost:6969/docs`
- **Default Port**: 6969

---

## API Usage

### Available Endpoints

#### 1. `/gemini` - Stateless Conversation
Creates a new session for each request.

```bash
curl -X POST http://localhost:6969/gemini \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What is 2+2?",
    "model": "gemini-2.5-pro"
  }'
```

**Response**:
```json
{
  "response": "Four."
}
```

#### 2. `/gemini-chat` - Persistent Conversation
Maintains context across requests.

```bash
curl -X POST http://localhost:6969/gemini-chat \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hello, my name is John"
  }'
```

#### 3. `/v1/chat/completions` - OpenAI-Compatible
OpenAI API format for easy integration.

```bash
curl -X POST http://localhost:6969/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-2.5-pro",
    "messages": [
      {"role": "user", "content": "Hello!"}
    ]
  }'
```

**Response**:
```json
{
  "id": "chatcmpl-12345",
  "object": "chat.completion",
  "created": 1693417200,
  "model": "gemini-2.5-pro",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Hello! How can I help you today?"
      },
      "finish_reason": "stop",
      "index": 0
    }
  ],
  "usage": {
    "prompt_tokens": 0,
    "completion_tokens": 0,
    "total_tokens": 0
  }
}
```

#### 4. `/translate` - Translation Endpoint
Integration with browser extension.

```bash
curl -X POST http://localhost:6969/translate \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Translate this to French: Hello World"
  }'
```

### Available Models

| Model | Description | Status |
|-------|-------------|--------|
| `gemini-2.5-pro` | Latest Gemini Pro model | ✅ Supported (Custom) |
| `gemini-1.5-pro` | Gemini 1.5 Pro | ✅ Supported |
| `gemini-1.5-flash` | Fast Gemini model | ✅ Supported |
| `gemini-1.5-pro-research` | Research variant | ✅ Supported |
| `gemini-2.0-flash-exp` | Experimental Flash | ✅ Supported |
| `gemini-2.0-exp-advanced` | Experimental Advanced | ✅ Supported |

---

## Troubleshooting

### Issue: "SECURE_1PSIDTS could get expired frequently"

**Solution**: Refresh your cookies
1. Go to `https://gemini.google.com`
2. Start a new chat to refresh session
3. Extract fresh `__Secure-1PSIDTS` cookie
4. Update `config.conf`
5. Restart the server

### Issue: "Gemini client is not initialized"

**Causes**:
- Expired cookies
- Invalid cookie format
- No Gemini access on your account

**Solution**:
1. Verify you're logged into Gemini in Chrome
2. Get fresh cookies
3. Ensure cookies are complete (not truncated)
4. Restart the server

### Issue: Unicode encoding errors

**Solution**: Already fixed in `src/run.py`. If still occurring:
```bash
set PYTHONIOENCODING=utf-8
python src/run.py
```

### Issue: "Unknown model name: gemini-2.5-pro"

**Solution**: The custom modification in `gemini_webapi/constants.py` may need to be reapplied. Check the [Custom Modifications](#custom-modifications) section.

### Issue: Port 6969 already in use

**Solution**: Change the port in `src/run.py` or kill the existing process:
```bash
# Windows
netstat -ano | findstr :6969
taskkill /PID <PID> /F

# Linux/Mac
lsof -ti:6969 | xargs kill -9
```

---

## Cookie Lifetime

- **`__Secure-1PSID`**: Long-lived (months to a year)
- **`__Secure-1PSIDTS`**: Short-lived (hours to days)

**Recommendation**: Update `__Secure-1PSIDTS` daily or when authentication fails.

---

## Security Notes

⚠️ **Important Security Considerations**:

1. **Keep cookies private**: Never commit `config.conf` to version control
2. **Cookie access**: Cookies provide full access to your Gemini account
3. **Usage limits**: Respects your Gemini account's rate limits
4. **Terms of Service**: This automation may violate Google's ToS
5. **Educational use**: Intended for research and educational purposes only

---

## Additional Resources

- **Original Repository**: [WebAI-to-API](https://github.com/Amm1rr/WebAI-to-API)
- **gemini-webapi Library**: [gemini-webapi on PyPI](https://pypi.org/project/gemini-webapi/)
- **API Documentation**: `http://localhost:6969/docs` (when server is running)

---

## Maintenance

### Updating Dependencies
```bash
pip install -r requirements.txt --upgrade
```

### Backing Up Custom Modifications
The custom modifications to `gemini_webapi/constants.py` may be lost when updating the library. Save this modification separately:

```python
# Add to Model enum in constants.py
G_2_5_PRO = (
    "gemini-2.5-pro",
    {"x-goog-ext-525001261-jspb": '[1,null,null,null,"4af6c7f5da75d65d",null,null,0,[4]]'},
    True,
)
```

---

## Version Information

- **WebAI-to-API**: v0.4.0
- **Python**: 3.13
- **gemini-webapi**: 1.8.3 (with custom modifications)
- **FastAPI**: 0.115.7
- **Last Updated**: November 2025

---

## Support

For issues related to:
- **WebAI-to-API**: [GitHub Issues](https://github.com/Amm1rr/WebAI-to-API/issues)
- **Gemini 2.5 Pro custom support**: Refer to this guide
- **Cookie issues**: See [Troubleshooting](#troubleshooting) section

---

## License

This project follows the original WebAI-to-API license (MIT).

**Disclaimer**: This is for educational and research purposes only. Use responsibly and be aware of Google's Terms of Service.
