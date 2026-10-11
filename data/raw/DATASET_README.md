# Synthetic E-commerce Behavioral Dataset

This dataset was generated for the Funnel Friction & Fake Engagement Detection portfolio project.

## Files
- `events.csv`: event-level user behavior; one row per recorded event.
- `sessions.csv`: session-level scenario metadata and synthetic ground-truth labels.

## Important limitations
All records are synthetic. They are designed to provide coherent e-commerce journeys and varied behavioral patterns, not to reproduce empirically validated industry distributions or real users. The scenario labels are known because they were injected by the simulator. They are suitable for testing analytical code and demonstrating model evaluation mechanics, but model performance on this dataset does not establish real-world fraud-detection accuracy.

The simulator deliberately includes overlap and variability: some friction sessions purchase, some typical sessions abandon, and suspicious sessions can have varying journey depth. The labels describe generated scenarios, not a definitive judgment about user intent.

## Reproducibility
Generated with NumPy random seed 42. The generation details are documented in the accompanying project specification.
