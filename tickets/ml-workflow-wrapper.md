# Ticket: The ML Workflow Wrapper
**Label**: `wayfinder:grilling`

## Question
The goal is to make the platform "intuitive." Users shouldn't be writing C code to deploy a TFLite Micro model.

We need to define the "wrapper":
1. **Input**: What does the user provide? (e.g., a `.tflite` file or a JSON config).
2. **Transformation**: How do we turn that into the C arrays/headers required by the firmware? (Automated build step in the Backend).
3. **Abstraction**: Does the firmware have a "Generic Model Runner" that just executes whatever is in the Model Slot, or do we generate a small "shim" of code for every model?
