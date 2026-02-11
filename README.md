# CRISPR Data Collection System

Automated data collection system for CRISPR guide RNA sequences. Includes a one-time bulk importer for established libraries and a 24/7 continuous monitor for new publications.

---

## Overview

This repository contains two complementary systems:

1. **Library Importer** - One-time bulk import of 100,000+ validated guides from five major CRISPR libraries
2. **24/7 Monitor** - Continuous scraper that monitors PubMed, GitHub, Addgene, bioRxiv, and Broad Institute for new CRISPR data

The collected data powers the CRISPR Guide Designer tool at [sciencecore.in](https://sciencecore.in).

---

## Components

### 1. Library Importer (library_importer.py)

One-time extraction of large-scale CRISPR guide data from established libraries.

| Specification | Details |
|---------------|---------|
| Total Guides | 485,000+ |
| Libraries | 5 (Brunello, GeCKO v2, TKOv3, Brie, Sabatini) |
| Execution Time | 2-3 hours |
| Run Frequency | One-time (with optional quarterly refresh) |

#### Libraries Included

| Library | Guides | Genes | Year | Journal |
|---------|--------|-------|------|---------|
| Brunello (Broad Institute) | 76,441 | 19,114 | 2016 | Nature |
| GeCKO v2 (Feng Zhang Lab) | 123,411 | 19,050 | 2014 | Science |
| TKOv3 (Toronto) | 71,090 | 18,053 | 2018 | Nature Biotechnology |
| Brie (Broad Institute) | 125,000 | 20,000 | 2019 | Cell |
| Sabatini Lab | 90,000 | 18,000 | 2017 | Cell |

### 2. 24/7 Monitor (crispr_monitor_24_7.py)

Production-grade continuous scraper with automatic error recovery.

| Specification | Details |
|---------------|---------|
| Cycle Interval | Every 6 hours |
| Sources Monitored | 5 (PubMed, GitHub, Addgene, bioRxiv, Broad) |
| Heartbeat Interval | 15 minutes |
| Health Check Interval | 30 minutes |
| Error Recovery | Automatic with exponential backoff |

#### Data Sources

| Source | Type | Update Frequency |
|--------|------|------------------|
| PubMed | Scientific publications | Real-time |
| GitHub | Open datasets | Daily |
| Addgene | Plasmid repository | Weekly |
| bioRxiv | Preprints | Daily |
| Broad Institute | Validated libraries | Monthly |

---

## Requirements

### System Requirements

- Python 3.8 or higher
- MySQL 5.7 or higher
- Linux server (recommended for 24/7 operation)
- screen or tmux (for background execution)

### Python Dependencies

```
mysql-connector-python>=8.0.0
requests>=2.28.0
psutil>=5.9.0
```

---

## Installation

```bash
# Clone repository
git clone https://github.com/YOUR_USERNAME/crispr-scraper.git
cd crispr-scraper

# Install dependencies
pip install -r requirements.txt

# Create log directory (for 24/7 monitor)
sudo mkdir -p /var/log
sudo chmod 755 /var/log
```

---

## Database Setup

### Create Table

```sql
CREATE TABLE crispr_guides_mega (
    id INT AUTO_INCREMENT PRIMARY KEY,
    guide_hash VARCHAR(32) UNIQUE,
    guide_sequence VARCHAR(25) NOT NULL,
    gene_symbol VARCHAR(50),
    efficiency DECIMAL(5,2),
    gc_content DECIMAL(5,2),
    off_target_score DECIMAL(5,2),
    validation_status VARCHAR(20),
    source_database VARCHAR(100),
    paper_title VARCHAR(255),
    publication_date DATE,
    cell_line VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_gene (gene_symbol),
    INDEX idx_efficiency (efficiency),
    INDEX idx_source (source_database),
    INDEX idx_created (created_at)
);
```

### Configuration

Update database credentials in both scripts:

```python
DB_CONFIG = {
    'host': 'your_host',
    'port': 3306,
    'user': 'your_username',
    'password': 'your_password',
    'database': 'your_database'
}
```

**Important:** Never commit real credentials to version control.

---

## Usage

### Running the Library Importer

One-time bulk import:

```bash
python library_importer.py
```

Expected output:

```
IMPORTING: Brunello (Broad Institute)
Target: 76,441 guides across 19,114 genes

  Progress: 500/19,114 genes (2.6%) - 1,847 added
  Progress: 1,000/19,114 genes (5.2%) - 3,692 added
  ...

Brunello (Broad Institute) COMPLETE!
   Added: 76,441 guides
   Duplicates: 0
```

### Running the 24/7 Monitor

#### Start in Background (Recommended)

```bash
# Using screen
screen -dmS crispr_monitor python3 crispr_monitor_24_7.py

# Attach to monitor output
screen -r crispr_monitor

# Detach: Press Ctrl+A, then D
```

#### Alternative: Using systemd

Create service file `/etc/systemd/system/crispr-monitor.service`:

```ini
[Unit]
Description=CRISPR 24/7 Monitor
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/path/to/scraper
ExecStart=/usr/bin/python3 /path/to/crispr_monitor_24_7.py
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl enable crispr-monitor
sudo systemctl start crispr-monitor
sudo systemctl status crispr-monitor
```

#### Monitor Logs

```bash
# View live logs
tail -f /var/log/crispr_monitor.log

# View statistics
cat /var/log/crispr_stats.json
```

---

## Configuration Options

### Library Importer

| Parameter | Default | Description |
|-----------|---------|-------------|
| batch_size | 500 | Guides per database insert |

### 24/7 Monitor

| Parameter | Default | Description |
|-----------|---------|-------------|
| cycle_hours | 6 | Hours between scraping cycles |
| heartbeat_minutes | 15 | Minutes between heartbeat logs |
| health_check_minutes | 30 | Minutes between database health checks |
| batch_size | 200 | Guides per database insert |
| max_retries | 5 | Maximum retry attempts on error |
| retry_delay | 300 | Seconds to wait between retries |
| rate_limit_delay | 0.5 | Seconds between API calls |

---

## Output Data Structure

Each guide record contains:

| Field | Type | Description |
|-------|------|-------------|
| guide_hash | VARCHAR(32) | MD5 hash for duplicate detection |
| guide_sequence | VARCHAR(25) | 20-mer guide sequence |
| gene_symbol | VARCHAR(50) | Target gene name |
| efficiency | DECIMAL(5,2) | Predicted knockout efficiency (0-100) |
| gc_content | DECIMAL(5,2) | GC percentage |
| off_target_score | DECIMAL(5,2) | Off-target activity score |
| validation_status | VARCHAR(20) | validated / published / predicted |
| source_database | VARCHAR(100) | Data source identifier |
| paper_title | VARCHAR(255) | Source publication title |
| publication_date | DATE | Publication date |
| cell_line | VARCHAR(50) | Cell line used for validation |

---

## Gene Categories

Both scrapers target genes across multiple categories:

| Category | Examples |
|----------|----------|
| Cancer Genes | TP53, KRAS, EGFR, BRCA1, BRCA2, MYC, PTEN |
| Essential Genes | POLR2A, PSMC1, RPL3, SF3B1, MCM2, ACTB |
| Cell Cycle | CCNA1, CCNB1, CCND1, CDK1, CDK2, CDC25A |
| DNA Repair | XRCC1-6, LIG1-4, ERCC1, RAD51, ATM, ATR |
| Kinases | AKT1-3, GSK3B, MAPK family, JAK family |
| Transcription Factors | MYC, JUN, FOS, STAT3, NFKB1, E2F family |
| Metabolism | HK1, HK2, PFKM, GAPDH, LDHA, G6PD |
| Apoptosis | BCL2, BAX, CASP3, CASP8, CASP9, XIAP |
| Chromatin Modifiers | EZH2, KMT2A, KDM1A, SMARCA4, ARID1A |

---

## Cell Lines

Guides are associated with validation data from:

- HEK293T
- HeLa
- MCF-7
- A549
- HCT116
- U2OS
- K562
- HAP1
- RPE1
- Jurkat
- A375
- H1299

---

## Monitoring and Maintenance

### Log Files

| File | Description |
|------|-------------|
| /var/log/crispr_monitor.log | Detailed execution log |
| /var/log/crispr_stats.json | JSON statistics file |

### Statistics JSON Format

```json
{
  "total_cycles": 100,
  "successful_cycles": 98,
  "failed_cycles": 2,
  "total_guides_added": 45000,
  "total_duplicates": 3200,
  "uptime_seconds": 604800,
  "database_health": "healthy",
  "memory_usage_mb": 125.4,
  "cpu_usage_percent": 2.3,
  "last_update": "2026-02-11T14:30:00"
}
```

### Health Checks

The 24/7 monitor performs automatic health checks:

- Database connectivity test
- Table existence verification
- Record count monitoring
- Memory usage tracking
- CPU usage tracking

### Graceful Shutdown

Send SIGINT or SIGTERM to trigger graceful shutdown:

```bash
# Find process
ps aux | grep crispr_monitor

# Send shutdown signal
kill -SIGTERM <PID>
```

The monitor will:
1. Complete current operation
2. Save final statistics
3. Log shutdown summary
4. Exit cleanly

---

## Error Handling

### Library Importer

- Database connection retry with exponential backoff
- Batch insert error logging (continues with next batch)
- Duplicate detection via INSERT IGNORE

### 24/7 Monitor

- Automatic reconnection on database failure
- Per-source error isolation (one source failing does not stop others)
- Exponential backoff on repeated failures
- Detailed error logging with stack traces
- Automatic recovery after transient errors

---

## API Rate Limits

The scrapers respect rate limits:

| Source | Rate Limit | Implementation |
|--------|------------|----------------|
| PubMed (NCBI) | 3 requests/second | 0.5s delay |
| GitHub API | 60 requests/hour (unauthenticated) | 0.5s delay |
| bioRxiv | 1 request/second | 0.5s delay |

---

## Project Structure

```
crispr-scraper/
├── library_importer.py       # One-time bulk import
├── crispr_monitor_24_7.py    # 24/7 continuous monitor
├── requirements.txt          # Python dependencies
├── LICENSE                   # MIT License
├── README.md                 # This file
└── sql/
    └── create_table.sql      # Database schema
```

---

## Performance

### Library Importer

| Metric | Value |
|--------|-------|
| Total import time | 2-3 hours |
| Guides imported | 485,000+ |
| Memory usage | Less than 500 MB |

### 24/7 Monitor

| Metric | Value |
|--------|-------|
| Cycle duration | 2-5 minutes |
| Guides per cycle | 200-500 |
| Memory usage | Less than 150 MB |
| CPU usage | Less than 5% |

---

## Recommended Workflow

1. Run Library Importer first to populate database with 100,000+ validated guides
2. Start 24/7 Monitor to continuously add new guides from publications
3. Monitor logs and statistics periodically
4. Re-run Library Importer quarterly to refresh base data

---

## Troubleshooting

### Database Connection Failed

- Verify credentials in DB_CONFIG
- Check network connectivity to database server
- Ensure MySQL server is running
- Check firewall rules

### No Guides Being Added

- Check source API availability
- Review rate limit settings
- Verify internet connectivity
- Check /var/log/crispr_monitor.log for errors

### High Memory Usage

- Reduce batch_size in configuration
- Restart monitor to clear memory
- Check for memory leaks in logs

### Monitor Not Starting

- Verify Python 3.8+ is installed
- Check all dependencies are installed
- Ensure /var/log is writable
- Review error output in terminal

---

## License

This project is licensed under the MIT License.

---

## Author

Fazil Firdous

Computational Biology Developer
Kashmir, India

- Website: [sciencecore.in](https://sciencecore.in)
- GitHub: [@eleosvanberg](https://github.com/eleosvanberg)

---

## Related Projects

- [ScienceCore](https://github.com/YOUR_USERNAME/sciencecore) - Computational biology platform
- [CRISPR Guide Designer](https://sciencecore.in) - Web tool using this data

---

## Changelog

### Version 1.0.0 (February 2026)

- Initial release
- Library Importer: 5 major CRISPR libraries supported
- 24/7 Monitor: 5 data sources monitored
- Automatic error recovery and health checks
- Comprehensive logging and statistics
