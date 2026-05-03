# Marker Format Notes

Canonical marker:

```text
<bubble:800>
```

Rejected alternatives:

```text
<|bubble:800|>
```

Reason: OpenClaw special-token sanitization removes `<|...|>` patterns.

```text
⟦bubble:800⟧
```

Reason: Low collision risk, but uncommon and harder for models to learn consistently.
