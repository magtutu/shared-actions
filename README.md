# Shared Action

This repository contains reusable GitHub Actions workflows that can be called from other repositories.

## About Reusable Workflows

Reusable workflows allow you to define a workflow once and call it from multiple repositories or within the same repository. This helps you follow the DRY (Don't Repeat Yourself) principle in your CI/CD pipelines.

## Usage

Reusable workflows are stored in the `.github/workflows/` directory and can be called from other workflows using the `workflow_call` trigger.
