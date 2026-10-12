# DoomScroll Guard - Quick Setup Guide

## What You Need to Provide

### 1. Training Data CSV
Place your training dataset at:
```
doomscroll-guard/data/training_data.csv
```

**Required columns:**
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

(Optionally: RiskScore)

### 2. Warning Sound MP3 Files (Optional)
Place your MP3 files at:
```
extension/sounds/stage3.mp3
extension/sounds/stage4.mp3
```

---

## Installation Steps

### Step 1: Install Python Dependencies
```bash
cd doomscroll-guard
pip install -r requirements.txt
```

### Step 2: Train the ML Models
```bash
cd ml
python train.py
```

This will:
- Load your training data
- Train 3 models (Random Forest, Gradient Boosting, Extra Trees)
- Save models to `ml/models/`
- Save evaluation metrics

### Step 3: Start the Python Server
```bash
cd server
python app.py
```

Keep this terminal open while using the extension.

### Step 4: Install Chrome Extension
1. Open Chrome and go to `chrome://extensions/`
2. Enable "Developer mode" (top-right toggle)
3. Click "Load unpacked"
4. Select the `doomscroll-guard/extension/` directory

### Step 5: Test the Extension
1. Open YouTube (youtube.com)
2. You should see a live counter in the top-right corner
3. Click the extension icon to open the dashboard
4. Verify "Server Online" is shown

---

## File Structure

```
doomscroll-guard/
├── extension/              # Chrome Extension
│   ├── manifest.json      # Extension configuration
│   ├── background.js      # Service worker
│   ├── content.js         # YouTube tracking
│   ├── popup.html         # Dashboard UI
│   ├── popup.js           # Dashboard logic
│   ├── styles.css         # All styling
│   ├── stage-config.js    # Stage quotes & config
│   ├── assets/
│   │   ├── icons/         # 12 generated SVG icons
│   │   └── stages/        # 5 generated stage illustrations
│   └── sounds/            # Place your MP3s here
│
├── server/                # Python API
│   ├── app.py            # Flask server
│   ├── predictor.py      # ML prediction
│   └── session_manager.py # Session state
│
├── ml/                    # Machine Learning
│   ├── train.py          # Training script
│   ├── preprocessing.py  # Data cleaning
│   ├── evaluate.py       # Evaluation
│   └── models/           # Saved models (after training)
│
├── data/                  # Data
│   └── training_data.csv # Your training data
│
├── requirements.txt       # Python packages
└── README.md             # Full documentation
```

---

## What Has Been Generated

### Icons (SVG)
- icon16.svg, icon48.svg, icon128.svg (Extension icons)
- dashboard.svg, timer.svg, stage.svg
- scroll.svg, video.svg, shorts.svg
- break.svg, warning.svg

### Stage Illustrations (SVG)
- stage0.svg - Calm, balanced usage
- stage1.svg - Beginning of prolonged scrolling
- stage2.svg - Increased engagement
- stage3.svg - High intensity
- stage4.svg - Strong intervention reminder

### Stage Quotes
- Stage 0: "Balance in digital life brings clarity to real life."
- Stage 1: "Awareness is the first step to mindful browsing."
- Stage 2: "Your attention deserves a break."
- Stage 3: "Your attention deserves a pause."
- Stage 4: "Step away. Your mind needs space to breathe."

---

## Key Features Implemented

✅ Three-model ensemble (Random Forest, Gradient Boosting, Extra Trees)
✅ Group-based data splitting by SessionID (prevents data leakage)
✅ Monotonic stage progression (stage never decreases during session)
✅ Stage-transition data saving (only on new stage, not every second)
✅ CSV appending (never overwrites existing data)
✅ Live YouTube counter with real-time updates
✅ Stage popups with quotes and images
✅ Warning sound support for stages 3 and 4
✅ Polished dashboard with model metrics
✅ Error handling for missing server/models
✅ Privacy-focused (local processing only)

---

## Troubleshooting

**Server Offline:**
- Ensure Python server is running: `cd server && python app.py`
- Check for errors in server terminal

**Models Not Found:**
- Train models first: `cd ml && python train.py`
- Verify `ml/models/` contains .joblib files

**Extension Not Working:**
- Refresh YouTube page after loading extension
- Check browser console (F12) for errors
- Verify extension has YouTube permissions

**Training Data Issues:**
- Ensure CSV has all required columns
- Check file is named exactly `training_data.csv`
- Place in `data/` directory

---

## Next Steps

1. Replace the sample `training_data.csv` with your actual dataset
2. Run `python ml/train.py` to train models on your data
3. (Optional) Add your MP3 warning sounds to `extension/sounds/`
4. Start the server and test the extension
5. Use YouTube and observe the system in action

---

## System Overview

The system implements a complete AI pipeline:

1. **Data Collection** - Training CSV you provide
2. **Preprocessing** - Cleaning and feature engineering
3. **Model Training** - 3 different classifiers
4. **Evaluation** - Accuracy, Precision, Recall, F1
5. **Real-Time Inference** - Live prediction every second
6. **Ensemble** - Average of 3 model predictions
7. **Monotonic Progression** - Stage only increases
8. **Intervention** - Popups with quotes and images
9. **Data Collection** - New stage transitions saved
10. **Retraining** - Models can be retrained with new data

---

For full documentation, see README.md
