# Heartbeat checklist

Read `heartbeat-state.json`. Run whichever check is most overdue.

- Email: every 30 min (9 AM - 9 PM)
- Calendar: every 2 hours (8 AM - 10 PM)
- Tasks: every 30 min
- Git: every 24 hours
- System: every 24 hours (3 AM)

Process:
1. Load timestamps from `heartbeat-state.json`
2. Calculate which check is most overdue
3. Run only that check
4. Update its timestamp
5. Report only if something is actionable
6. Otherwise reply `HEARTBEAT_OK`
