# Ticket: The "Model Slot" Architecture
**Label**: `wayfinder:prototype`
**Issue**: https://github.com/ConnectedEdge/connectededge/issues/3
**Status**: Resolved

## Question
To solve the "Last Mile," we need a mechanism to update ML models without full firmware updates.

We need to decide:
1. **Memory Layout**: Where does the model live? (Flash partition vs. internal SRAM).
2. **Update Trigger**: How does the app know a new model has arrived? (Signal from Gateway via MQTT $\rightarrow$ Write to Flash $\rightarrow$ Soft Reset or Dynamic Reload).
3. **Validation**: How do we ensure the model is compatible with the current firmware version before loading it?

The output of this ticket should be a rough memory map and a sequence diagram of the update flow.
