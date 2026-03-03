# Procurement Spec Workspace

Cross-project specification workspace for the AKDITAL Procurement system.

## Layout

- `specs/` — specifications (may span both API and frontend)
- `.specify/` — Specify tooling (scripts, templates, memory)
- `procurement-api` → symlink to `/home/aelfannir/dev/procurement-api` (Laravel API)
- `procurement-app` → symlink to `/home/aelfannir/dev/procurement-app` (React app)

## Purpose

This workspace is used to author and manage specs that can require changes across both the API and the frontend. The symlinks provide direct access to both project codebases.
