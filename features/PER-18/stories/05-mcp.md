# Story 5: README HTTP setup and migration guide

**Repo:** `mcp` (adlc-mcp-server)  
**Story ID:** 5  
**Story Title:** Update README with HTTP setup and migration guide  
**Parent Epic:** PER-19  

---

## 1. Acceptance Criteria

**AC4:** README documents FastAPI + FastMCP HTTP as primary path (uvicorn, env vars, MCP URL).  
**Exception 2:** Migration notes for stdio → HTTP clients.  
**Exception 3:** Auth called out as follow-up / out of scope for v1.

---

## 2. File Plan

### Update
1. **`README.md`** — replace stub with Quick start + migration

Must include:
1. Env: `GRAPHQL_ENDPOINT`, optional `HOST`/`PORT`
2. Run: `uvicorn server:app --host 0.0.0.0 --port 8000` (or `python main.py`)
3. MCP URL: `http://localhost:8000/sse`
4. Short stdio vs HTTP migration section
5. Auth deferred note

No product code changes required unless README references wrong module paths.

## 3. Tests

No new automated tests required; keep existing pytest green.

## 4. Definition of done

README complete; tests still pass; commit on `feat/PER-18`.
