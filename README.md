# DAAK

DAAK is a human-in-the-loop fraud detection system for identifying suspicious mobile money scam messages in Pakistan, with a focus on Easypaisa, JazzCash, and UPaisa transactions.

## Overview

The platform analyzes incoming messages and screenshots, scores them across multiple detection layers, and classifies them as safe, suspicious, or requiring human review. It is designed to reduce scam exposure before money is transferred while keeping review costs and latency under control.

## Key Features

- Multi-stage detection pipeline: metadata, content, and local LLM analysis
- Adaptive scoring to prevent single-signal overreliance
- Human review queue with priority ranking and review budget controls
- Local-first architecture using Ollama, ChromaDB, and SQLite
- FastAPI-based dashboard for monitoring and review workflows

## Tech Stack

- Python
- FastAPI
- SQLite
- ChromaDB
- Ollama
- Jinja2, Pydantic, SQLAlchemy

## Getting Started

1. Open the project directory.
2. Create and activate a virtual environment.
3. Install dependencies:

   pip install -r app/requirements.txt

4. Ensure Ollama is installed and the local model is available:

   ollama pull gemma2:2b

5. Start the API server:

   cd app && uvicorn main:app --reload

6. Open the dashboard in your browser:

   http://localhost:8000/dashboard

## Project Structure

- app/main.py — FastAPI application entry point
- app/routers — API routes for messages, reviews, patterns, analytics, and status
- app/services — analysis and scam-detection logic
- app/models — database models and initialization
- app/templates — dashboard UI
- app/data — seed scam patterns and related datasets

## Note

This project is intended as a practical prototype for scam detection and review automation in local deployment environments.

Developed by [@ch-yousufali](https://github.com/ch-yousufali) and [@farhann-saleem](https://github.com/farhann-saleem).
