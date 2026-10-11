# Data Dictionary

## events.csv

- `event_id`: unique event identifier.
- `user_id`: synthetic user identifier.
- `session_id`: session identifier linking events and session metadata.
- `event_timestamp`: ISO-like timestamp string; events are ordered within each session.
- `event_name`: recorded action (e.g. page view, cart action, payment outcome).
- `page_name`: page associated with the event.
- `product_id`: synthetic product identifier, blank when not applicable.
- `click_count`: click count recorded for selected interaction events; zero otherwise.
- `duration_seconds`: page duration signal when available; blank otherwise.
- `error_type`: error category when an error is recorded.
- `payment_status`: payment outcome when relevant.
- `navigation_action`: navigation behavior when relevant.
- `device_type`: mobile, desktop, or tablet.
- `acquisition_channel`: synthetic acquisition channel.

## sessions.csv

- `session_id`, `user_id`: identifiers.
- `scenario`: simulator scenario class (`typical`, friction subtypes, or suspicious-pattern subtypes).
- `ground_truth_friction`: 1 for injected friction scenarios.
- `ground_truth_suspicious`: 1 for injected suspicious-pattern scenarios.
- `purchased`: whether a purchase event occurred.
- `reached_cart`, `reached_checkout`: journey milestones.
- `device_type`, `acquisition_channel`: session context.
- `session_duration_seconds`: elapsed time between first and last event.
- `event_count`: number of events in the session.

`ground_truth_suspicious` is a synthetic scenario label, not verified fraud. Do not use `scenario` or ground-truth labels as model input features.
