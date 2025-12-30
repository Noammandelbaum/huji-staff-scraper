# CLAUDE.md

> Think carefully and implement the most concise solution that changes as little code as possible.

## Project: HUJI Staff Scraper

Scraper for Hebrew University staff directory - exports to CSV with monthly AWS updates.

## Tech Stack

- **Language**: Python 3.11+
- **Scraping**: BeautifulSoup / Scrapy
- **Cloud**: AWS (Lambda/ECS, S3, SES, EventBridge)
- **IaC**: Terraform

## Project Structure

```
src/
  scraper/      # Scraping logic
  parsers/      # HTML parsers by page type
  exporters/    # CSV export + email
aws/
  terraform/    # Infrastructure as Code
data/           # Output files
tests/          # Unit tests
```

## Code Style

- Follow PEP 8
- Type hints required
- Docstrings for public functions

## Testing

Run tests before committing:
```bash
pytest tests/
```
