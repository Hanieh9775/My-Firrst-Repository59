"""
Async Job Queue with Monitoring Dashboard
Single-file FastAPI application

Features:
- Background task execution
- Job queue with status tracking
- REST API for job management
- Real-time job monitoring dashboard
- Async & production-style structure

Run:
pip install fastapi uvicorn
uvicorn app:app --reload
"""

import uuid
import asyncio
from datetime import datetime
from typing import Dict

from fastapi import FastAPI, BackgroundTasks, HTTPException
from fastapi.responses import HTMLResponse
from pydantic import BaseModel

app = FastAPI(title="Async Job Queue")

# ------------------------
# In-memory job storage
# ------------------------
jobs: Dict[str, dict] = {}

# ------------------------
# Models
# ------------------------
class JobCreate(BaseModel):
    payload: str
    duration: int = 5  # seconds

# ------------------------
# Job Worker
# ------------------------
async def run_job(job_id: str, payload: str, duration: int):
    try:
        jobs[job_id]["status"] = "RUNNING"
        await asyncio.sleep(duration)
        jobs[job_id]["status"] = "DONE"
        jobs[job_id]["result"] = payload.upper()
        jobs[job_id]["finished_at"] = datetime.utcnow().isoformat()
    except Exception as e:
        jobs[job_id]["status"] = "FAILED"
        jobs[job_id]["error"] = str(e)

# ------------------------
# API Routes
# ------------------------
@app.post("/jobs")
async def create_job(data: JobCreate, background: BackgroundTasks):
    job_id = str(uuid.uuid4())

    jobs[job_id] = {
        "id": job_id,
        "payload": data.payload,
        "status": "PENDING",
        "created_at": datetime.utcnow().isoformat(),
        "finished_at": None,
        "result": None
    }

    background.add_task(run_job, job_id, data.payload, data.duration)
    return {"job_id": job_id}

@app.get("/jobs/{job_id}")
async def get_job(job_id: str):
    job = jobs.get(job_id)
    if not job:
        raise HTTPException(status_code=404, detail="Job not found")
    return job

@app.get("/jobs")
async def list_jobs():
    return list(jobs.values())

# ------------------------
# Dashboard
# ------------------------
@app.get("/")
async def dashboard():
    return HTMLResponse(HTML)

HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>Job Queue Dashboard</title>
    <style>
        body { font-family: Arial; background:#eef2f5; padding:40px; }
        .container { max-width:900px; margin:auto; }
        .card {
            background:#fff; padding:20px; margin-bottom:15px;
            border-radius:10px; box-shadow:0 4px 10px rgba(0,0,0,0.1);
        }
        .status { font-weight:bold; }
        .PENDING { color:orange; }
        .RUNNING { color:blue; }
        .DONE { color:green; }
        .FAILED { color:red; }
        input, button {
            padding:10px; margin-top:10px; width:100%;
        }
        button {
            background:#0069d9; color:white; border:none;
            border-radius:6px; cursor:pointer;
        }
    </style>
</head>
<body>

<div class="container">
    <h2>Async Job Queue Dashboard</h2>

    <div class="card">
        <input id="payload" placeholder="Job payload">
        <input id="duration" type="number" placeholder="Duration (seconds)" value="5">
        <button onclick="createJob()">Create Job</button>
    </div>

    <div id="jobs"></div>
</div>

<script>
async function createJob() {
    const payload = document.getElementById("payload").value;
    const duration = document.getElementById("duration").value;

    await fetch("/jobs", {
        method: "POST",
        headers: {"Content-Type": "application/json"},
        body: JSON.stringify({ payload, duration })
    });

    loadJobs();
}

async function loadJobs() {
    const res = await fetch("/jobs");
    const jobs = await res.json();
    const container = document.getElementById("jobs");
    container.innerHTML = "";

    jobs.reverse().forEach(job => {
        container.innerHTML += `
            <div class="card">
                <div>ID: ${job.id}</div>
                <div class="status ${job.status}">Status: ${job.status}</div>
                <div>Result: ${job.result || "-"}</div>
            </div>
        `;
    });
}

setInterval(loadJobs, 2000);
loadJobs();
</script>

</body>
</html>
"""
