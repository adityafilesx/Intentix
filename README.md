Intentix
Adaptive Motor-Accessible Human–Computer Interaction System
I. Overview

Intentix is an adaptive assistive interaction system that enables individuals with motor disabilities to control digital devices using eye movement, intentional blinks, and voice commands.

It is designed for users whose cognitive intent remains intact but whose motor execution is degraded due to conditions such as:

Amyotrophic Lateral Sclerosis (ALS)

Parkinson’s disease

Stroke

Spinal cord injuries

Carpal Tunnel Syndrome (CTS)

Repetitive Strain Injuries (RSI)

Intentix stabilizes noisy motor signals and introduces structured confirmation mechanisms to ensure safe, reliable, and fatigue-aware interaction.

II. Problem Context

Modern digital systems assume precise and sustained motor control. As motor ability declines, cursor positioning, clicking, and typing become unreliable.

Motor disability exists on a spectrum of signal degradation. Most assistive technologies either require specialized hardware or fail under real-world instability and tremor.

Intentix addresses this by redesigning interaction around degraded motor signals rather than ideal precision.

III. Core Capabilities

Real-time eye-gaze tracking

Stabilized cursor mapping with tremor filtering

Multi-stage blink confirmation model

Voice-based daily task automation

Fatigue-aware blink monitoring

Safety-first execution workflow

IV. Technology Stack
A. Computer Vision

MediaPipe Face Mesh

OpenCV

B. Backend Processing

Python

Real-time processing loop

C. System Automation

PyAutoGUI

D. Voice Processing

Speech-to-Text engine

Intent parsing logic

E. Frontend

Flask

HTML / CSS / JavaScript

F. Hardware Requirements

Standard webcam

Microphone

V. System Architecture

Intentix follows a modular, layered architecture optimized for real-time assistive interaction.

A. High-Level Architecture Flow
flowchart TD

    A[User] --> B[Webcam Input]
    A --> C[Microphone Input]

    B --> D[Video Frame Capture]
    C --> E[Audio Capture]

    D --> F[Face & Eye Landmark Detection (MediaPipe)]
    F --> G[Gaze Estimation]
    G --> H[Cursor Stabilization (EMA + Dead Zone)]

    F --> I[Blink Detection (EAR + Threshold Logic)]

    E --> J[Speech-to-Text Engine]
    J --> K[Command Parsing]

    H --> L[Intent Interpretation Engine]
    I --> L
    K --> L

    L --> M[Action Executor]
    M --> N[OS-Level Control (PyAutoGUI)]
    M --> O[Automation Services (Email / Alarm / Search)]

    N --> P[Visual Feedback Layer]
    O --> P
    P --> A

B. Layered Architectural Model
flowchart TB

    subgraph Input Layer
        A1[Webcam]
        A2[Microphone]
    end

    subgraph Vision & Audio Processing
        B1[Landmark Detection]
        B2[Gaze Estimation]
        B3[Blink Detection]
        B4[Speech Processing]
        B5[Stabilization Engine]
    end

    subgraph Intent Layer
        C1[Intent State Machine]
        C2[Three-Step Lock-In]
    end

    subgraph Execution Layer
        D1[Cursor Control]
        D2[Keyboard Automation]
        D3[Task Automation]
    end

    subgraph Feedback Layer
        E1[Focus Highlight]
        E2[Execution Confirmation]
        E3[Fatigue Monitoring]
    end

    A1 --> B1
    A2 --> B4
    B1 --> B2
    B2 --> B5
    B3 --> C1
    B4 --> C1
    B5 --> C1
    C1 --> C2
    C2 --> D1
    C2 --> D2
    C2 --> D3
    D1 --> E1
    D2 --> E2
    D3 --> E3

C. Architectural Layer Explanation
1. Input Layer

Captures raw video and audio signals from consumer-grade hardware.

2. Vision & Audio Processing Layer

Extracts facial and iris landmarks

Computes Eye Aspect Ratio (EAR)

Applies smoothing and tremor filtering

Converts speech into structured commands

3. Intent Layer

Combines gaze coordinates, blink states, and parsed voice commands

Implements a three-stage confirmation state machine

Produces validated intent objects

4. Execution Layer

Converts intent into OS-level actions

Handles cursor movement, clicks, typing, and automation

Applies confirmation gates for safety

5. Feedback Layer

Displays focus and lock states

Confirms action execution

Monitors blink fatigue patterns

VI. Real-Time Processing Pipeline

Capture frame

Detect landmarks

Estimate gaze

Stabilize cursor

Detect blink

Update state machine

Interpret intent

Execute action

Provide feedback

Repeat

Latency is optimized for smooth assistive interaction.

VII. Installation Guide
A. Prerequisites

Python 3.8+

Webcam

Microphone

Windows / macOS / Linux

B. Clone Repository
git clone https://github.com/your-username/intentix.git
cd intentix

C. Create Virtual Environment
python -m venv venv


Activate:

Windows:

venv\Scripts\activate


macOS/Linux:

source venv/bin/activate

D. Install Dependencies
pip install -r requirements.txt


Typical dependencies:

opencv-python

mediapipe

pyautogui

flask

speechrecognition

numpy

E. Run Application
python app.py


Open:

http://localhost:5000


Grant camera and microphone permissions if prompted.

VIII. Configuration

Configurable parameters:

Blink threshold

Blink cooldown duration

Cursor smoothing factor

Dead-zone threshold

Emergency contact details

Stored in config.json.

IX. Security & Safety

No raw video storage

Fully local processing

Confirmation required for critical actions

Isolated emergency execution flow

X. Future Scope

Adaptive learning of motor thresholds

Android Accessibility Service implementation

AI-driven predictive intent modeling

User profile persistence

Multilingual voice automation
