# AI Invoice & Document Processing System

Invoices come in by email or upload, get read by AI, validated against business rules, and routed to the right approval tier — automatically.

## Overview

This system eliminates manual invoice data entry. Any PDF invoice — arriving by email or submitted through an upload endpoint — gets analyzed by GPT-4o Vision, which extracts every relevant field: vendor details, line items, totals, tax, due dates. A validation engine then checks the extraction for completeness and correctness, and routes the invoice to auto-approval, manager review, executive review, or rejection based on configurable business rules.

## The Problem

Someone on the finance team spends real hours every week opening invoice PDFs, typing numbers into a spreadsheet, checking whether the math adds up, and deciding who needs to sign off. It's repetitive, error-prone at 4pm on a Friday, and doesn't scale — more invoices just means more hours, not a better process.

## The Solution

Every invoice gets read the same way, every time, in seconds instead of minutes. Low-risk invoices that pass validation with high confidence get auto-approved with zero human involvement. Everything else — high-value, low-confidence, or flagged with validation errors — gets routed to the right reviewer with full context already extracted, instead of a raw PDF someone has to open and interpret from scratch.

## How It Works

Invoice arrives (Gmail) or is uploaded (webhook)
↓
Normalize input (unify email + manual upload into one shape)
↓
Has PDF attachment? → No → logged as rejected, stops here
↓ Yes
GPT-4o Vision extracts structured data
(vendor, dates, line items, totals, confidence score)
↓
Parse extraction result
↓
Validation Engine
(required fields · math check · overdue flag · duplicate placeholder)
↓
Approval Router
↓
Auto-Approved / Manager Review / Executive Review / Rejected
↓
Notification built → Finance team emailed → Logged to sheet
↓
Non-approved cases also → Pending Review queue + vendor acknowledgement


## Features

- **Dual entry point** — accepts invoices from a monitored Gmail inbox or a direct API upload, normalized into one consistent shape before processing
- **Vision-based extraction** — reads the invoice image directly with GPT-4o, no OCR pipeline or template matching required
- **Structured output** — every invoice becomes a consistent JSON object: vendor, line items, subtotal, tax, total, currency, payment terms, and more
- **Validation engine** — flags missing critical fields, checks subtotal + tax = total, detects overdue due dates
- **Tiered approval routing** — auto-approve under a configurable threshold with high confidence, otherwise route to manager or executive review based on invoice value
- **Finance team notification** — a clean, readable summary email for every invoice processed, not just a raw JSON dump
- **Vendor acknowledgement** — automatic reply confirming receipt when an invoice needs review, so vendors aren't left wondering
- **Full audit trail** — every invoice logged with extraction data, validation results, and final status; anything pending review is also tracked separately in its own queue

## Tech Stack

- **n8n** — workflow orchestration
- **Gmail API** — invoice intake + notifications
- **OpenAI GPT-4o** — vision-based document extraction (via direct HTTP request)
- **Google Sheets** — invoice log and pending review queue

## Setup Guide

See [`docs/setup-guide.md`](./docs/setup-guide.md) for full step-by-step instructions, including Google Sheet structure, credential setup, and business rule tuning.

Quick summary:
- Required accounts: Gmail, OpenAI (GPT-4o vision access), a Google Sheet with two tabs
- Approval thresholds (`AUTO_APPROVE_LIMIT`, `REVIEW_LIMIT`) are set inside the Validation Engine code node and should be adjusted to match your company's actual policy
- The OpenAI API key is currently set directly in the HTTP Request node's header — move this to an n8n credential or environment variable before using in production

## Usage

**Email invoices:** send any PDF invoice to the monitored inbox. It's picked up within a minute and processed automatically.

**Manual upload:**
```bash
curl -X POST https://your-n8n-instance/webhook/invoice-upload \
  -H "Content-Type: application/json" \
  -d '{
    "sender_email": "vendor@example.com",
    "file_name": "invoice_0234.pdf",
    "file_type": "application/pdf",
    "file_data": "<base64-encoded PDF>"
  }'
```

**Reviewing pending invoices:** check the `Pending_Review` tab in the Google Sheet — anything not auto-approved lands there with `Reviewer_Action` set to `PENDING`.
