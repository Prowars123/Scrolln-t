# Scrolln't

An AI-powered behavioral stage estimation system for YouTube viewing habits. This is an educational AI Capstone Project that monitors viewing behavior in real-time and estimates scrolling intensity using machine learning.

**IMPORTANT:** This is an educational project and NOT a medical or psychological diagnostic system. It estimates behavioral stages related to prolonged viewing patterns, not addiction or any medical condition.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [AI Pipeline](#ai-pipeline)
- [Installation](#installation)
- [Training the Models](#training-the-models)
- [Running the Application](#running-the-application)
- [Chrome Extension Setup](#chrome-extension-setup)
- [How It Works](#how-it-works)
- [Stage System](#stage-system)
- [Break Mode](#break-mode)
- [Data Collection](#data-collection)
- [Troubleshooting](#troubleshooting)
- [Privacy](#privacy)

---

## Project Overview

Scrolln't uses a three-model ensemble to estimate behavioral stages based on YouTube viewing metrics. The system:

- Tracks YouTube viewing behavior in real-time
- Collects metrics like duration, videos watched, scrolls, and viewing patterns
- Uses three ML models (Random Forest, Gradient Boosting, Extra Trees) for prediction
- Enforces monotonic stage progression during sessions
- Saves stage-transition data for future model retraining
- Provides visual interventions when stages increase

---

## Architecture

```
YouTube
    ↓
Chrome Extension (Content Script)
    ↓
Live Session Metrics (every second)
    ↓
Local Python API Server (Flask)
    ↓
3 Trained ML Models
    ↓
3 Stage Predictions
    ↓
Average Predictions
    ↓
Round Average (half-up rounding)
    ↓
Enforce Monotonic Stage Progression
    ↓
Final Current Stage
    ↓
Chrome Extension UI + Warnings
    ↓
Stage-Transition Data Saved to CSV
```

### Project Structure

```
doomscroll-guard/
│
├── extension/
│   ├── manifest.json          # Chrome Extension manifest
│   ├── background.js          # Service worker for API communication
│   ├── content.js             # YouTube tracking and live counter
│   ├── popup.html             # Dashboard UI
│   ├── popup.js               # Dashboard logic
│   ├── styles.css             # All styling
│   ├── stage-config.js        # Stage quotes and configuration
│   ├── assets/
│   │   ├── icons/             # Generated SVG icons
│   │   └── stages/            # Generated stage illustrations
│   └── sounds/                # Place for MP3 warning sounds
│
├── server/
│   ├── app.py                 # Flask API server
│   ├── predictor.py           # ML prediction logic
│   └── session_manager.py     # Session state and CSV saving
│
├── ml/
│   ├── train.py               # Model training script
│   ├── preprocessing.py       # Data cleaning and preparation
│   ├── evaluate.py            # Model evaluation utilities
│   └── models/                # Saved trained models
│
├── data/
│   └── training_data.csv      # Training dataset (you provide this)
│
├── requirements.txt           # Python dependencies
└── README.md                  # This file
```

---

## AI Pipeline

This project implements a complete supervised machine learning pipeline:

### 1. Problem Identification
Monitor YouTube viewing behavior to estimate scrolling intensity and provide behavioral interventions.

### 2. Data Collection
Training data is collected from user sessions, including:
- Session duration
- Number of unique videos watched
- Number of unique Shorts watched
- Scroll actions
- Late-night viewing flag
- Active viewing time
- Breaks taken
- Time since last break
- Derived rates (scroll rate, short rate)
- Behavioral stage (target variable)

### 3. Data Preprocessing
- Loading and validating CSV data
- Handling missing values
- Ensuring numeric types
- Removing invalid entries

### 4. Feature Engineering
The following features are used for ML training:
- Duration
- Videos
- Shorts
- Scrolls
- LateNight
- ActiveViewingTime
- Breaks
- TimeSinceLastBreak
- ScrollRate
- ShortRate

**Excluded features (to prevent data leakage):**
- SessionID (used only for grouped splitting)
- Timestamp (used only for tracking)
- RiskScore (if present, to avoid target leakage)

### 5. Supervised Machine Learning
Three different classification models are trained:
1. **Random Forest Classifier** - Ensemble of decision trees
2. **Gradient Boosting Classifier** - Sequential ensemble with boosting
3. **Extra Trees Classifier** - Extremely randomized trees

### 6. Model Training
- **Data Splitting:** Uses `GroupShuffleSplit` to split by SessionID
  - **CRITICAL:** This prevents data leakage by ensuring all rows from the same session stay in either train or test set
  - Random row splitting would leak information across sessions
- **Training:** Each model is trained on the training set
- **Evaluation:** Models are evaluated on the held-out test set

### 7. Model Evaluation
For each model, we calculate:
- Accuracy
- Precision (weighted average)
- Recall (weighted average)
- F1 Score (weighted average)
- Confusion Matrix
- Training/test sample counts

### 8. Ensemble Prediction
During real-time inference:
1. All three models make independent predictions
2. Predictions are averaged
3. Average is rounded using half-up rounding (not banker's rounding)
4. Example: (3 + 3 + 2) / 3 = 2.67 → Stage 3

### 9. Real-Time Inference
- Chrome extension collects metrics every second
- Metrics are sent to the Python API
- API returns ensemble prediction
- Current stage is updated with monotonic progression

### 10. Behavioral Intervention
- When stage increases, a popup appears with:
  - Stage number and title
  - Original anti-doomscrolling quote
  - Stage-specific illustration
  - Explanation of the stage
  - Action buttons

### 11. Stage-Transition Data Collection
- **CRITICAL:** Data is saved ONLY when entering a NEW stage
- Stage 0 baseline is saved at session start
- Subsequent stages save one row each
- Same stage for extended time does NOT create duplicate rows
- Data is appended to training CSV for future retraining

### 12. Future Retraining
- Saved stage-transition data becomes part of training set
- Models can be retrained with `python ml/train.py`
- New models replace old models in the `ml/models/` directory

---

## Installation

### Prerequisites

- Python 3.8 or higher
- Google Chrome browser
- Your training data CSV file
- Your MP3 warning sound files (optional)

### Step 1: Clone or Download Project

Download the project to your local machine.

### Step 2: Install Python Dependencies

Navigate to the project directory and install required packages:

```bash
cd doomscroll-guard
pip install -r requirements.txt
```

### Step 3: Place Your Training Data

Place your `training_data.csv` file in the `data/` directory:

```
doomscroll-guard/data/training_data.csv
```

**Expected CSV columns:**
- SessionID
- Timestamp
- Duration
- Videos
- Shorts
- Scrolls
- LateNight
- ActiveViewingTime
- Breaks
- TimeSinceLastBreak
- ScrollRate
- ShortRate
- Stage

(Optionally: RiskScore - will be preserved but not used as a feature)

### Step 4: Place Warning Sounds (Optional)

If you have MP3 warning sounds, place them in the `extension/sounds/` directory:

```
extension/sounds/stage2.mp3
extension/sounds/stage3.mp3
extension/sounds/stage4.mp3
```

**Note:** Stages 3 and 4 will play warning sounds when entered. Stage 2 has visual warning only.

---

## Training the Models

### Train All Three Models

Run the training script:

```bash
cd ml
python train.py
```

This will:
1. Load `training_data.csv`
2. Validate and clean the data
3. Split data by SessionID (preventing data leakage)
4. Train Random Forest, Gradient Boosting, and Extra Trees models
5. Evaluate all models
6. Save trained models to `ml/models/`
7. Save evaluation metrics to `ml/models/model_metrics.json`

### View Model Evaluation

To see detailed evaluation metrics:

```bash
cd ml
python evaluate.py
```

This displays:
- Accuracy, Precision, Recall, F1 for each model
- Confusion matrices
- Feature importance (for models that support it)

---

## Running the Application

### Start the Python ML Server

The Flask API server must be running for the extension to work:

```bash
cd server
python app.py
```

You should see:
```
============================================================
DOOMSCROLL GUARD - ML API SERVER
============================================================
Loading ML models...
  Loaded random_forest
  Loaded gradient_boosting
  Loaded extra_trees
  Loaded feature names: [...]
All models loaded successfully!
Starting Flask server on http://127.0.0.1:5000
```

**Keep this terminal open** while using the extension.

---

## Chrome Extension Setup

### Step 1: Open Chrome Extensions

1. Open Google Chrome
2. Navigate to `chrome://extensions/`
3. Enable "Developer mode" (toggle in top-right corner)

### Step 2: Load the Extension

1. Click "Load unpacked"
2. Navigate to and select the `doomscroll-guard/extension/` directory
3. Click "Select Folder"

The extension should now appear in your extensions list.

### Step 3: Verify Installation

1. Click the extension icon in your Chrome toolbar
2. The dashboard should open
3. Check that "Server Online" is shown in the header

If it shows "Server Offline", ensure the Python server is running.

---

## How It Works

### Real-Time Tracking

When you visit YouTube:

1. **Session Start:** Extension starts a new session and saves Stage 0 baseline
2. **Metric Collection:** Every second, the extension updates:
   - Duration (elapsed time)
   - Unique videos watched (using video IDs)
   - Unique Shorts watched (using short IDs)
   - Total scroll actions
   - Late-night flag (based on time of day)
   - Active viewing time (time with user activity)
   - Breaks (periods of inactivity > 30 seconds)
   - Time since last break
   - Scroll rate (scrolls per second)
   - Short rate (shorts per second)

3. **Prediction:** Metrics are sent to the Python API
4. **Ensemble:** Three models predict independently, results are averaged
5. **Monotonic Progression:** Stage can only increase, never decrease during a session
6. **Intervention:** If stage increases, popup appears with quote and image
7. **Data Saving:** New stage triggers one CSV row append

### Unique Content Counting

- **Videos:** Counted once per unique video ID
- **Shorts:** Counted once per unique short ID
- **Scrolls:** Every scroll action counts (even revisiting old content)

This distinction is important: scrolling back to an old Short still counts as a scroll, but doesn't increment the unique Shorts count.

### Late Night Detection

Late night is defined as:
- 10 PM (22:00) to 6 AM (06:00)

Late-night viewing is a feature that may contribute to faster stage progression through the trained model.

### Break Detection

A break is counted when:
- User is inactive for > 30 seconds
- Mouse, keyboard, or scroll activity resets the timer

Breaks are meaningful inactivity periods, not tiny pauses.

---

## Stage System

There are 6 behavioral stages (0-5):

### Stage 0: Normal / Low-Intensity Viewing
- Balanced digital usage
- Quote: "Balance in digital life brings clarity to real life."
- Visual: Calm, balanced scene

### Stage 1: Mild Prolonged Scrolling
- Beginning of extended viewing
- Quote: "Awareness is the first step to mindful browsing."
- Visual: Person starting to spend more time

### Stage 2: Elevated Scrolling Behavior
- Increased engagement
- Quote: "Your attention deserves a break."
- Visual: More intense scrolling
- Warning: Visual only

### Stage 3: High Scrolling Intensity
- Very high screen engagement
- Quote: "Your attention deserves a pause."
- Visual: Strong engagement
- Warning: Visual + Sound (stage3.mp3)

### Stage 4: Very High/Prolonged Scrolling Intensity
- Extended viewing period
- Quote: "Step away. Your mind needs space to breathe."
- Visual: Strong intervention reminder
- Warning: Visual + Sound (stage4.mp3)

### Stage 5: Extreme Doomscrolling
- Critical stage requiring immediate action
- Quote: "The screen can wait. Your wellbeing cannot."
- Visual: Intense warning with "STOP NOW" message
- Warning: Visual + Sound (stage4.mp3)

### Monotonic Progression

**CRITICAL:** During a single session, the stage can ONLY increase.

Example:
- Previous Stage: 2
- New ensemble prediction: 1
- **Current Stage remains: 2** (does not decrease)

- Previous Stage: 2
- New ensemble prediction: 3
- **Current Stage becomes: 3** (increases)

When a new session starts, stage resets to 0.

---

## Break Mode

Break Mode is a dedicated calming interface with four selectable themes to help users take meaningful breaks from YouTube.

### Entry Points

There are three ways to enter Break Mode:

1. **Live Counter Button** - Click "Take a Break" in the live counter overlay on YouTube
2. **Stage Popup Button** - Click "Take a Break" in any stage-warning popup
3. **Automatic Inactivity Prompt** - After 2 minutes of inactivity, a gentle prompt appears offering to start Break Mode

### Four Calm Themes

1. **Sage Garden** - Sage green, muted botanical greens, peaceful foliage and nature details
2. **Lavender Dream** - Lavender, lilac, soft gradients, delicate glowing particles
3. **Ocean Breeze** - Muted aqua, seafoam, soft teal, subtle ocean waves
4. **Warm Sand** - Cream, beige, muted sand, gentle sunlight effects

### Features

- **Countdown Timer** - Choose 1, 3, 5, or 10 minute breaks with pause/resume/add time controls
- **Virtual Plant** - A plant that grows as your break progresses
- **Activity Lounge** - 12 offline activities with descriptions (Mindful Breathing, Eye Rest, Quick Stretch, etc.)
- **Surprise Me** - Randomly selects an activity
- **Ambient Audio** - Procedural ambient sounds matching each theme (optional)
- **Theme Persistence** - Your chosen theme is remembered across sessions

### Limitations

- Ambient audio is generated procedurally using Web Audio API and may not perfectly replicate natural sounds
- Inactivity detection tracks user activity (mouse, keyboard, scroll) but may not account for video playback perfectly
- Break Mode does not block YouTube or force users to stay; users can exit at any time

---

## Data Collection

### Stage-Transition Saving

**IMPORTANT:** Data is saved ONLY when entering a NEW stage.

Example session:
```
Session starts → Stage 0 → Save row
...time passes...
Stage 0 → Stage 1 → Save row
...time passes...
Stage 1 → Stage 2 → Save row
Stage remains 2 for 10 minutes → NO save
Stage 2 → Stage 3 → Save row
...time passes...
Stage 3 → Stage 4 → Save row
```

This results in exactly 5 rows for this session, one per stage transition.

### CSV Structure

Each saved row contains:
- SessionID (unique UUID)
- Timestamp (ISO format)
- Duration
- Videos
- Shorts
- Scrolls
- LateNight
- ActiveViewingTime
- Breaks
- TimeSinceLastBreak
- ScrollRate
- ShortRate
- Stage

### Safe CSV Writing

The system uses thread-safe file writing to prevent corruption when multiple operations occur simultaneously.

### Retraining

To retrain models with new data:

1. Ensure `training_data.csv` contains your latest data
2. Run: `python ml/train.py`
3. New models are saved to `ml/models/`
4. Restart the Python server to load new models

---

## Troubleshooting

### Server Shows "Offline"

**Problem:** Extension shows "Server Offline" in dashboard

**Solutions:**
1. Ensure Python server is running: `cd server && python app.py`
2. Check that server shows "Starting Flask server on http://127.0.0.1:5000"
3. Verify no firewall is blocking localhost:5000
4. Try restarting the server

### Models Not Found

**Problem:** Server error "Model not found"

**Solutions:**
1. Train models first: `cd ml && python train.py`
2. Verify `ml/models/` directory contains:
   - `random_forest.joblib`
   - `gradient_boosting.joblib`
   - `extra_trees.joblib`
   - `model_metrics.json`

### Training Data Not Found

**Problem:** "Training data not found" error

**Solutions:**
1. Place your `training_data.csv` in `data/` directory
2. Verify filename is exactly `training_data.csv`
3. Check CSV has required columns (see Installation section)

### Extension Not Loading on YouTube

**Problem:** Live counter doesn't appear on YouTube

**Solutions:**
1. Refresh the YouTube page after loading extension
2. Check extension is enabled in `chrome://extensions/`
3. Open browser console (F12) to check for errors
4. Verify extension has permission for YouTube URLs

### Prediction Errors

**Problem:** Prediction fails or returns errors

**Solutions:**
1. Check server logs for detailed error messages
2. Verify all required features are being sent
3. Ensure metrics are numeric values
4. Check that SessionID is properly initialized

### Sound Not Playing

**Problem:** Warning sound doesn't play on stage 3/4

**Solutions:**
1. Place MP3 files in `extension/sounds/`
2. Verify filenames: `stage3.mp3`, `stage4.mp3`
3. Check browser console for audio loading errors
4. Ensure browser allows audio autoplay
5. Sounds are optional - system works without them

### CSV Corruption

**Problem:** Training CSV becomes corrupted

**Solutions:**
1. Keep a backup of your original CSV
2. The system uses thread-safe writing, but manual edits can cause issues
3. Ensure only one instance of the server is running
4. Validate CSV structure after manual edits

---

## Privacy

This application is designed with privacy in mind:

### Local Processing
- All ML inference happens on your local machine
- Python server runs on localhost (127.0.0.1)
- No data is sent to external servers

### Data Collection
- Only collects metrics required for behavioral estimation
- Does NOT collect:
  - Passwords
  - Unrelated browsing history
  - Personal information
  - Video titles or content
  - User account information

### YouTube-Specific
- Only tracks on YouTube pages
- Uses video IDs for uniqueness, not content
- Does not access watch history or personal data

### Training Data
- Training CSV is stored locally
- You control the data
- No cloud synchronization

---

## Educational Purpose

This project is designed as a Class 12 AI Capstone Project to demonstrate:

- Supervised machine learning
- Ensemble methods
- Real-time inference
- Feature engineering
- Data preprocessing
- Model evaluation
- Chrome Extension development
- Full-stack application architecture

**This is NOT a medical or psychological diagnostic tool.** It estimates behavioral patterns for educational purposes only.

---

## License

This project is created for educational purposes.

---

## Credits

Created as an AI Capstone Project demonstrating end-to-end machine learning application development.
