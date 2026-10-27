# Ticket: Front-End Labeling UI
**Label**: `wayfinder:grilling`
**Status**: Resolved

## Question
We need to design the UX for time-series data labeling in the web UI. Users will need to:
1. View time-series sensor data
2. Select time ranges to label
3. Assign labels to those ranges
4. Export labeled datasets for ML training

We need to decide on the interaction model, UI components, and data flow.

## Resolution

### Labeling Paradigm
**Brush selection** on the time-series chart. Users click and drag to select a time range, then assign a label class to that range. Zoom controls allow fine-grained selection.

### Label Types
**Multi-class** labels. Users can define an arbitrary set of classes (e.g., "normal", "warning", "critical"). Binary classification is a special case with two classes.

### Label Class Management
**Dynamic** class management. Users can add new classes at any time. Removing a class that's already in use requires confirmation. Classes are defined per-project.

### Time Scale
**Single scale at current zoom**. Labels are created at the granularity of the current chart view. Users zoom in/out to control label precision. No multi-scale hierarchy.

### Data Flow
**Direct DB write with API-mediated training trigger**. Labels are stored directly in the database as first-class data. Training is explicitly triggered by the user via an API call — not automatic. This gives users control over when to retrain.

### Dataset Versioning
**Versioned snapshots**. Each training run creates a versioned snapshot of the labeled dataset at that moment. This enables reproducibility and comparison across training runs.

### Review Workflow
**Overlay + confusion matrix**. The UI shows:
1. Raw sensor data (line chart)
2. User labels (colored regions overlaid)
3. Model predictions (dashed lines or different opacity overlaid)
4. Confusion matrix for aggregate accuracy metrics

Users can click on specific misclassifications to inspect them in detail.

### Temporal Alignment
**Window-based alignment** with configurable tolerance. Predictions are matched to labels within a time window (default: 1 second, configurable). This accounts for inference delay and different sampling rates between labeling and prediction.

### Interaction Flow
1. User opens a time-series chart for a device/sensor
2. User selects a label class from a dropdown
3. User brushes (click-drag) to select a time range
4. Label is applied and shown as a colored region
5. User repeats for all relevant segments
6. User clicks "Train Model" to trigger training on the current labeled dataset
7. After training, user views predictions overlaid on labels
8. User inspects confusion matrix and specific misclassifications
9. User refines labels and re-trains as needed

## Decisions Made
- Brush selection for time range labeling
- Multi-class labels (not binary-only or regression)
- Dynamic class management (add anytime, confirm on remove)
- Single scale at current zoom (no multi-scale hierarchy)
- Direct DB write with API-mediated training trigger
- Versioned dataset snapshots for each training run
- Overlay + confusion matrix for review
- Window-based temporal alignment (configurable tolerance)
