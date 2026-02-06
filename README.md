# End-to-End-API-Testing-of-DemoQA-BookStore-APIs-using-Postman
# DemoQA Postman API Tests

## Overview
Postman-based API automation for DemoQA BookStore APIs.
Includes:
- Positive & negative functional tests
- E2E user book lifecycle
- CI/CD execution via GitHub Actions

## How to Run Locally
```bash
newman run postman/collections/demoqa.postman_collection.json \
  -e postman/environments/DemoQA-env.postman_environment.json
