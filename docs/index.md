# Fasiri Documentation

## What is Fasiri?

**Fasiri** (Swahili: *to interpret*) is a unified API and Python SDK for African language translation, speech-to-text, and text-to-speech. It brings together the best African language AI providers - Sunbird AI, Khaya AI, and HuggingFace - behind a single consistent interface.

Fasiri supports two modes of operation:

**Fasiri Cloud** - use the hosted API with a single key. Fasiri handles provider routing, fallback, and reliability.

**Direct mode** - bring your own provider keys and call Sunbird AI, Khaya AI, or HuggingFace directly from your application. No Fasiri account needed. You handle your own provider billing. Fasiri is the routing layer.

---

## Capabilities

**Translation**

Translate text between English and 19+ African languages. Supports single requests and batch translation with automatic provider routing.

[Get started with translation](guides/translation.md)

**Speech-to-Text**

Transcribe audio in Luganda, Acholi, Ateso, Runyankore, Lugbara, and Swahili using Sunbird AI.

[Get started with STT](guides/speech-workflows.md)

**Text-to-Speech**

Synthesise natural-sounding speech in Ugandan languages using Sunbird AI voice models.

[Get started with TTS](guides/speech-workflows.md)

**Python SDK**

Sync and async Python client with full type hints. Supports both cloud and direct mode with the same interface.

[Install the SDK](getting-started/installation.md)

---

## Quick example

**Cloud mode:**

```python
from fasiri import Fasiri

client = Fasiri(api_key="fsri_...")

result = client.translate("Good morning", target="lug")
print(result)          # Wasuze otya
print(result.provider) # sunbird
```

**Direct mode (bring your own keys):**

```python
from fasiri import Fasiri
from fasiri.providers import SunbirdProvider, KhayaProvider

client = Fasiri(
    providers=[
        SunbirdProvider(api_key="eyJ..."),
        KhayaProvider(api_key="your-khaya-key"),
    ]
)

result = client.translate("Good morning", target="lug")
print(result)          # Wasuze otya
print(result.provider) # sunbird
```

Same interface. Same results. You choose where requests go.

---

## How provider routing works

```
Request: en -> lug   ->  Sunbird AI  (specialised for Ugandan languages)
Request: en -> yo    ->  Khaya AI    (specialised for West African languages)
Request: en -> sw    ->  HuggingFace (Helsinki-NLP opus-mt-en-sw)

If primary fails     ->  next provider in chain
If all fail          ->  ProviderError raised
```

---

## Next steps

- [Installation](getting-started/installation.md)
- [Authentication](getting-started/authentication.md)
- [Quick start](getting-started/quickstart.md)
- [Direct mode guide](guides/direct-mode.md)
- [Supported languages](getting-started/languages.md)
