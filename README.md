# MLS Flash Report

What the project is and where things live.

## Project Structure

```
mls-flash-report/
├── README.md                    # what the project is, where things live
├── docs/
│   ├── PROJECT_BRIEF.md         # the v3 brief
│   ├── infosec_review.md        # the one-pager
│   ├── decisions/               # ADR-style decision log
│   │   ├── 001-streamlit-over-cron-headless.md
│   │   ├── 002-render-hosting.md
│   │   ├── 003-google-oauth.md
│   │   └── 004-defer-content-data-to-v1.1.md
│   └── conversations/
│       └── 2026-05-21-planning-with-claude.md  # this chat, exported
├── infrastructure/
│   └── snowflake_setup.sql
└── .gitignore                   # node_modules, .env, *.p8, etc.
```

## Documentation

See the `docs/` directory for project documentation, decision records, and conversation logs.

## Infrastructure

Database and infrastructure setup scripts are located in `infrastructure/`.
