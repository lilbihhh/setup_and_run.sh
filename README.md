# setup_and_run.sh
#!/usr/bin/env bash
set -e

mkdir -p teamos/agents teamos/benchmarks teamos/tests keys

cat > requirements.txt <<'EOF'
pydantic>=1.10.0
pytest>=8.0.0
cryptography>=42.0.0
sqlalchemy>=2.0.0
openai>=1.0.0
fastapi>=0.110.0
uvicorn>=0.29.0
psycopg2-binary>=2.9.9
jinja2>=3.1.4
EOF

cat > .env.example <<'EOF'
LLM_PROVIDER=mock
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4o-mini
DATABASE_URL=sqlite:///teamos.db
TEAMOS_SIGNING_KEY_PATH=keys/private.pem
TEAMOS_VERIFY_KEY_PATH=keys/public.pem
DASHBOARD_HOST=0.0.0.0
DASHBOARD_PORT=8000
EOF

cat > README.md <<'EOF'
# TeamOS

Governed epistemic machine with append-only ledger semantics, signed message routing, verification, and benchmark evaluation.

## Quick start

1. python -m venv .venv
2. source .venv/bin/activate
3. pip install -r requirements.txt
4. python - <<'PY'
from teamos.signing import generate_keypair
generate_keypair("keys/private.pem", "keys/public.pem")
PY
5. PYTHONPATH=. pytest -q
6. PYTHONPATH=. python app.py

Open:
- http://localhost:8000/
- http://localhost:8000/benchmarks
EOF

cat > app.py <<'EOF'
import uvicorn
from fastapi import FastAPI
from fastapi.responses import HTMLResponse

from teamos.config import settings
from teamos.ui import build_dashboard_html, build_benchmark_html
from teamos.benchmark import run_all_benchmarks

app = FastAPI(title="TeamOS")

@app.get("/", response_class=HTMLResponse)
async def dashboard():
    return HTMLResponse(content=build_dashboard_html())

@app.get("/benchmarks", response_class=HTMLResponse)
async def benchmark_page():
    result = await run_all_benchmarks()
    return HTMLResponse(content=build_benchmark_html(result))

if __name__ == "__main__":
    uvicorn.run(app, host=settings.dashboard_host, port=settings.dashboard_port)
EOF

cat > run_benchmarks.py <<'EOF'
import asyncio
from teamos.benchmark import run_all_benchmarks

async def main():
    result = await run_all_benchmarks()
    print(result["metrics"])
    with open("benchmark_report.html", "w", encoding="utf-8") as f:
        f.write(result["html"])

if __name__ == "__main__":
    asyncio.run(main())
EOF

cat > teamos/__init__.py <<'EOF'
__all__ = ["Ledger", "Router", "Message"]
EOF

cat > teamos/config.py <<'EOF'
import os
from dataclasses import dataclass
from typing import Optional

@dataclass
class Settings:
    llm_provider: str = os.getenv("LLM_PROVIDER", "mock")
    openai_api_key: Optional[str] = os.getenv("OPENAI_API_KEY")
    openai_model: str = os.getenv("OPENAI_MODEL", "gpt-4o-mini")
    signing_key_path: str = os.getenv("TEAMOS_SIGNING_KEY_PATH", "keys/private.pem")
    verify_key_path: str = os.getenv("TEAMOS_VERIFY_KEY_PATH", "keys/public.pem")
    database_url: str = os.getenv("DATABASE_URL", "sqlite:///teamos.db")
    dashboard_host: str = os.getenv("DASHBOARD_HOST", "0.0.0.0")
    dashboard_port: int = int(os.getenv("DASHBOARD_PORT", "8000"))

settings = Settings()
EOF

cat > teamos/schema.py <<'EOF'
import uuid
from datetime import datetime, timezone
from typing import Any, Dict, List, Optional

from pydantic import BaseModel, Field, root_validator, validator

class EvidenceItem(BaseModel):
    id: Optional[str] = None
    type: Optional[str] = None
    payload: Dict[str, Any] = Field(default_factory=dict)

class Message(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    type: str
    role: str
    status: str
    content: Dict[str, Any] = Field(default_factory=dict)
    requires_verification: bool = True
    verification_level: str = "required"
    derived_from: List[str] = Field(default_factory=list)
    supersedes: List[str] = Field(default_factory=list)
    created_at: str = Field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
    model: Optional[str] = None
    evidence: List[EvidenceItem] = Field(default_factory=list)
    signature: Optional[str] = None

    @validator("created_at", pre=True)
    def parse_created_at(cls, value):
        if isinstance(value, datetime):
            return value.astimezone(timezone.utc).isoformat()
        return value

    @root_validator
    def enforce_roles(cls, values):
        msg_type = values.get("type")
        role = values.get("role")
        if msg_type == "VERIFICATION" and role != "VERIFIER":
            raise ValueError("VERIFICATION must be from VERIFIER")
        if msg_type == "CRITIQUE" and role != "CRITIC":
            raise ValueError("CRITIQUE must be from CRITIC")
        if msg_type == "MERGE" and role != "SYNTHESIZER":
            raise ValueError("MERGE must be from SYNTHESIZER")
        return values
EOF

cat > teamos/ledger.py <<'EOF'
import json
from typing import Dict, List, Optional

from sqlalchemy import create_engine, text
from sqlalchemy.exc import IntegrityError

from teamos.config import settings
from teamos.schema import Message

class Ledger:
    def __init__(self):
        self.backend = "postgres" if "postgres" in settings.database_url.lower() else "sqlite"
        if self.backend == "postgres":
            self.engine = create_engine(settings.database_url)
            self._init_postgres()
        else:
            self.engine = None
            self._messages: Dict[str, Message] = {}

    def _init_postgres(self):
        with self.engine.begin() as conn:
            conn.execute(text("""
                CREATE TABLE IF NOT EXISTS messages (
                    id TEXT PRIMARY KEY,
                    payload JSONB NOT NULL,
                    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
                )
            """))
            conn.execute(text("""
                CREATE INDEX IF NOT EXISTS idx_messages_created_at
                ON messages (created_at)
            """))

    def append(self, message: Message):
        if self.backend == "postgres":
            payload = json.loads(message.json())
            with self.engine.begin() as conn:
                try:
                    conn.execute(
                        text("""
                            INSERT INTO messages (id, payload, created_at)
                            VALUES (:id, CAST(:payload AS JSONB), :created_at)
                        """),
                        {"id": str(message.id), "payload": json.dumps(payload), "created_at": message.created_at},
                    )
                except IntegrityError as exc:
                    raise ValueError(f"Message {message.id} already exists (append-only)") from exc
            return

        if message.id in self._messages:
            raise ValueError(f"Message {message.id} already exists (append-only)")
        self._messages[message.id] = message

    def exists(self, message_id: str) -> bool:
        if self.backend == "postgres":
            with self.engine.begin() as conn:
                row = conn.execute(text("SELECT 1 FROM messages WHERE id = :id"), {"id": message_id}).fetchone()
            return bool(row)
        return message_id in self._messages

    def get(self, message_id: str) -> Optional[Message]:
        if self.backend == "postgres":
            with self.engine.begin() as conn:
                row = conn.execute(text("SELECT payload FROM messages WHERE id = :id"), {"id": message_id}).fetchone()
            if not row:
                return None
            return Message.parse_obj(row[0])
        return self._messages.get(message_id)

    def list_all(self) -> List[Message]:
        if self.backend == "postgres":
            with self.engine.begin() as conn:
                rows = conn.execute(text("SELECT payload FROM messages ORDER BY created_at ASC")).fetchall()
            return [Message.parse_obj(r[0]) for r in rows]
        return list(self._messages.values())

    def size(self) -> int:
        if self.backend == "postgres":
            with self.engine.begin() as conn:
                row = conn.execute(text("SELECT COUNT(*) FROM messages")).fetchone()
            return int(row[0])
        return len(self._messages)
EOF

cat > teamos/signing.py <<'EOF'
import json
from pathlib import Path

from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import ed25519

def generate_keypair(private_path: str, public_path: str):
    private_key = ed25519.Ed25519PrivateKey.generate()
    public_key = private_key.public_key()

    Path(private_path).parent.mkdir(parents=True, exist_ok=True)
    Path(public_path).parent.mkdir(parents=True, exist_ok=True)

    private_bytes = private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption(),
    )
    public_bytes = public_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo,
    )

    Path(private_path).write_bytes(private_bytes)
    Path(public_path).write_bytes(public_bytes)

def load_private_key(path: str):
    return serialization.load_pem_private_key(Path(path).read_bytes(), password=None)

def load_public_key(path: str):
    return serialization.load_pem_public_key(Path(path).read_bytes())

def canonical_payload(message: dict) -> bytes:
    return json.dumps(message, sort_keys=True, separators=(",", ":")).encode("utf-8")

def sign_message(message: dict, private_key_path: str) -> str:
    private_key = load_private_key(private_key_path)
    sig = private_key.sign(canonical_payload(message))
    return sig.hex()

def verify_signature(message: dict, signature_hex: str, public_key_path: str) -> bool:
    public_key = load_public_key(public_key_path)
    if not signature_hex:
        return False
    try:
        public_key.verify(bytes.fromhex(signature_hex), canonical_payload(message))
        return True
    except Exception:
        return False
EOF

cat > teamos/router.py <<'EOF'
from typing import Any, Dict

from teamos.config import settings
from teamos.ledger import Ledger
from teamos.signing import verify_signature

class Router:
    def __init__(self, ledger: Ledger):
        self.ledger = ledger

    def validate(self, payload: Dict[str, Any]):
        from teamos.schema import Message
        return Message.parse_obj(payload)

    def enforce_references(self, message):
        for ref in list(message.derived_from) + list(message.supersedes):
            if not self.ledger.exists(ref):
                raise ValueError(f"Reference {ref} not found in ledger")

    def enforce_signature(self, message):
        sig = getattr(message, "signature", None)
        if not sig:
            raise ValueError(f"Message {message.id} is missing a signature")
        payload = message.dict(exclude={"signature"})
        if not verify_signature(payload, sig, settings.verify_key_path):
            raise ValueError(f"Invalid signature for message {message.id}")

    def route(self, payload: Dict[str, Any]):
        msg = self.validate(payload)
        if self.ledger.exists(msg.id):
            raise ValueError(f"Duplicate message id: {msg.id}")
        self.enforce_references(msg)
        self.enforce_signature(msg)
        self.ledger.append(msg)
        return msg
EOF

cat > teamos/verifier.py <<'EOF'
from typing import Any, Dict

def verify_value_claim(claim: Dict[str, Any]) -> Dict[str, Any]:
    content = claim.get("content", {})
    actual = content.get("actual")
    expected = content.get("expected")
    if actual is not None and expected is not None:
        passed = actual == expected
        return {
            "passed": passed,
            "summary": f"Expected {expected}, got {actual}.",
            "actual": actual,
            "expected": expected,
        }
    return {
        "passed": False,
        "summary": "No deterministic expected-value check available.",
    }
EOF

cat > teamos/agents/__init__.py <<'EOF'
EOF

cat > teamos/agents/llm_client.py <<'EOF'
import json
from dataclasses import dataclass
from typing import Any, Dict, Optional

from teamos.config import settings

@dataclass
class LLMResponse:
    text: str
    parsed: Optional[Dict[str, Any]] = None
    model: str = ""
    provider: str = ""
    raw: Optional[Dict[str, Any]] = None

class BaseLLMAdapter:
    async def generate(self, system_prompt: str, user_prompt: str, **kwargs) -> LLMResponse:
        raise NotImplementedError

class MockLLMAdapter(BaseLLMAdapter):
    async def generate(self, system_prompt: str, user_prompt: str, **kwargs) -> LLMResponse:
        payload = {
            "ok": True,
            "message": "mock generation successful",
            "prompt_preview": user_prompt[:200],
            "confidence": 0.92,
        }
        return LLMResponse(
            text=json.dumps(payload),
            parsed=payload,
            model="mock-model",
            provider="mock",
            raw={"source": "mock"},
        )

class OpenAIAdapter(BaseLLMAdapter):
    provider_name = "openai"

    def __init__(self):
        import openai
        self.client = openai.OpenAI(api_key=settings.openai_api_key)

    async def generate(self, system_prompt: str, user_prompt: str, **kwargs) -> LLMResponse:
        response = self.client.chat.completions.create(
            model=settings.openai_model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt},
            ],
            temperature=0.0,
            response_format={"type": "json_object"},
        )
        content = response.choices[0].message.content or "{}"
        try:
            parsed = json.loads(content)
        except Exception:
            parsed = None
        return LLMResponse(
            text=content,
            parsed=parsed,
            model=response.model,
            provider="openai",
            raw={"id": response.id},
        )

class LLMClient:
    def __init__(self, adapter: BaseLLMAdapter):
        self.adapter = adapter

    async def generate_structured(self, system_prompt: str, user_prompt: str, **kwargs) -> Dict[str, Any]:
        response = await self.adapter.generate(system_prompt, user_prompt, **kwargs)
        if response.parsed is not None:
            return response.parsed
        try:
            return json.loads(response.text)
        except Exception as exc:
            raise ValueError(f"Invalid JSON from LLM: {response.text[:200]}") from exc

def get_default_llm_client() -> LLMClient:
    if settings.llm_provider == "openai":
        if not settings.openai_api_key:
            raise RuntimeError("OPENAI_API_KEY is required when LLM_PROVIDER=openai")
        return LLMClient(OpenAIAdapter())
    return LLMClient(MockLLMAdapter())
EOF

cat > teamos/agents/specialist.py <<'EOF'
import uuid
from datetime import datetime, timezone

SPECIALIST_SYSTEM_PROMPT = """
You are a specialist in TeamOS.
Return valid JSON only.
Expected structure:
{
  "content": {"value": ..., "subject": "..."},
  "status": "pending",
  "requires_verification": true,
  "verification_level": "required",
  "evidence": []
}
"""

async def generate_specialist_claim(llm_client, prompt: str):
    payload = await llm_client.generate_structured(SPECIALIST_SYSTEM_PROMPT, prompt, temperature=0.0)
    return {
        "id": str(uuid.uuid4()),
        "type": "CLAIM",
        "role": "SPECIALIST",
        "status": payload.get("status", "pending"),
        "content": payload.get("content", {}),
        "requires_verification": payload.get("requires_verification", True),
        "verification_level": payload.get("verification_level", "required"),
        "derived_from": [],
        "supersedes": [],
        "created_at": datetime.now(timezone.utc).isoformat(),
        "model": "specialist",
        "evidence": payload.get("evidence", []),
        "signature": None,
    }
EOF

cat > teamos/agents/critic.py <<'EOF'
import uuid
from datetime import datetime, timezone

CRITIC_SYSTEM_PROMPT = """
You are a critic in TeamOS.
Return valid JSON only.
Expected structure:
{
  "content": {"issues": ["..."], "summary": "..."},
  "status": "open"
}
"""

async def generate_critique(llm_client, claim: dict):
    payload = await llm_client.generate_structured(CRITIC_SYSTEM_PROMPT, str(claim), temperature=0.0)
    return {
        "id": str(uuid.uuid4()),
        "type": "CRITIQUE",
        "role": "CRITIC",
        "status": payload.get("status", "open"),
        "content": payload.get("content", {}),
        "requires_verification": True,
        "verification_level": "required",
        "derived_from": [claim["id"]],
        "supersedes": [],
        "created_at": datetime.now(timezone.utc).isoformat(),
        "model": "critic",
        "evidence": [],
        "signature": None,
    }
EOF

cat > teamos/agents/verifier.py <<'EOF'
import uuid
from datetime import datetime, timezone

from teamos.verifier import verify_value_claim

VERIFIER_SYSTEM_PROMPT = """
You are a verifier in TeamOS.
Return valid JSON only.
Expected structure:
{
  \"content\": {\"passed\": true, \"summary\": \"...\", \"actual\": ..., \"expected\": ...},
  \"status\": \"passed\"
}
"""

async def generate_verification(llm_client, claim: dict):
    payload = await llm_client.generate_structured(VERIFIER_SYSTEM_PROMPT, str(claim), temperature=0.0)
    verification_content = payload.get("content", {})
    result = verify_value_claim({"content": verification_content})
    return {
        "id": str(uuid.uuid4()),
        "type": "VERIFICATION",
        "role": "VERIFIER",
        "status": \"passed\" if result[\"passed\"] else \"failed\",\n        \"content\": verification_content,\n        \"requires_verification\": True,\n        \"verification_level\": \"required\",\n        \"derived_from\": [claim[\"id\"]],\n        \"supersedes\": [],\n        \"created_at\": datetime.now(timezone.utc).isoformat(),\n        \"model\": \"verifier\",\n        \"evidence\": [],\n        \"signature\": None,\n    }\nEOF

cat > teamos/pipeline.py <<'EOF'
from datetime import datetime, timezone
from typing import Dict, List

from teamos.agents.critic import generate_critique
from teamos.agents.llm_client import get_default_llm_client
from teamos.agents.specialist import generate_specialist_claim
from teamos.agents.verifier import generate_verification
from teamos.config import settings
from teamos.ledger import Ledger
from teamos.router import Router
from teamos.signing import sign_message

async def run_pipeline(goal: str, specialists: List[Dict[str, str]]):
    ledger = Ledger()
    router = Router(ledger)
    llm_client = get_default_llm_client()

    task = {
        "id": "task-001",
        "type": "TASK",
        "role": "PLANNER",
        "status": "open",
        "content": {"goal": goal},
        "requires_verification": True,
        "verification_level": "required",
        "derived_from": [],
        "supersedes": [],
        "created_at": datetime.now(timezone.utc).isoformat(),
        "model": "planner",
        "evidence": [],
        "signature": None,
    }
    task["signature"] = sign_message(task, settings.signing_key_path)
    router.route(task)

    claims = []
    for spec in specialists:
        claim = await generate_specialist_claim(llm_client, spec["prompt"])
        claim["signature"] = sign_message(claim, settings.signing_key_path)
        router.route(claim)
        claims.append(claim)

    critiques = []
    for claim in claims:
        crit = await generate_critique(llm_client, claim)
        crit["signature"] = sign_message(crit, settings.signing_key_path)
        router.route(crit)
        critiques.append(crit)

    verifications = []
    for claim in claims:
        ver = await generate_verification(llm_client, claim)
        ver["signature"] = sign_message(ver, settings.signing_key_path)
        router.route(ver)
        verifications.append(ver)

    accepted = [c["id"] for c, v in zip(claims, verifications) if v["status"] == "passed"]
    rejected = [c["id"] for c, v in zip(claims, verifications) if v["status"] != "passed"]

    merge = {
        "id": "merge-001",
        "type": "MERGE"
