# Standard Operating Procedure: Storage Management

**SAIAB HPC Server - lab417.saiab.ac.za**

| Document Information ||
|----------------------|----------------|
| **Version** | 1.0 |
| **Date** | 2026-02-10 |
| **Author** | SAIAB HPC Administration |
| **Contact** | EP.deVilliers@saiab.nrf.ac.za |

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [Prerequisites](#2-prerequisites)
3. [Storage System Overview](#3-storage-system-overview)
4. [Procedure: Running Disk Monitoring System](#4-procedure-running-disk-monitoring-system)
5. [Procedure: Responding to Low Disk Space Alerts](#5-procedure-responding-to-low-disk-space-alerts)
6. [Procedure: Assisting Users with Storage Optimization](#6-procedure-assisting-users-with-storage-optimization)
7. [Scheduled Maintenance](#7-scheduled-maintenance)
8. [Troubleshooting](#8-troubleshooting)
9. [Best Practices](#9-best-practices)
10. [Appendix A: Disk Monitoring System Details](#appendix-a-disk-monitoring-system-details)
11. [Appendix B: Common Storage Optimization Commands](#appendix-b-common-storage-optimization-commands)
12. [Appendix C: Emergency Procedures](#appendix-c-emergency-procedures)
13. [Appendix D: Report Interpretation Guide](#appendix-d-report-interpretation-guide)

---

## 1. Purpose and Scope

### 1.1 Purpose
This Standard Operating Procedure (SOP) provides comprehensive instructions for managing storage resources on the SAIAB HPC bioinformatics server (lab417.saiab.ac.za). It ensures consistent monitoring, optimization, and maintenance of disk space across all filesystems.

### 1.2 Scope
This SOP covers:
- Automated disk monitoring and reporting system operations
- Manual storage assessments and interventions
- User support for storage optimization
- Emergency procedures for critical disk space situations
- Storage best practices for bioinformatics data

### 1.3 Intended Audience
- System administrators with sudo/root privileges
- HPC support staff
- Anyone responsible for maintaining server storage health

---

## 2. Prerequisites

### 2.1 Required Privileges
- Root or sudo access to lab417.saiab.ac.za
- SSH access to the server
- Permission to read user directories for scanning

### 2.2 System Requirements
- Disk monitoring system installed at: `/home/evilliers/work/sysadmin/`
- Mail server configured for email notifications
- Sufficient access to all monitored filesystems:
  - `/home` (user home directories)
  - `/data` (shared data storage)
  - `/mnt/agrp/lab417` (NFS-mounted research storage)

### 2.3 Knowledge Requirements
- Basic Linux system administration
- Understanding of filesystem concepts (disk usage, inodes, permissions)
- Familiarity with bioinformatics file formats (FASTQ, SAM/BAM)
- SLURM job scheduler basics (for storage-intensive operations)

---

## 3. Storage System Overview

### 3.1 Filesystem Layout

| Mount Point | Type | Purpose | Monitored |
|------------|------|---------|-----------|
| `/home` | Local | User home directories | Yes |
| `/data` | Local | Shared bioinformatics data | Yes |
| `/mnt/agrp/lab417` | NFS | Research group storage | Yes |
| `/tmp` | tmpfs | Temporary files | No |

### 3.2 Current Status (Baseline)
Based on typical operations:
- **Disk Usage**: 86-94% utilization across filesystems
- **User Count**: ~38 active users
- **Potential Savings**: ~796GB identified per scan
  - FASTQ compression: ~632GB (70-80% savings)
  - SAM to BAM conversion: ~164GB (50-80% savings)

### 3.3 Automated Monitoring
The disk monitoring system runs automatically every Sunday at 2:00 AM via cron job. It:
- Scans all user directories in monitored filesystems
- Identifies optimization opportunities
- Generates individual user reports
- Emails reports to users and administrators
- Creates backup reports in home directories

### 3.4 Alert Thresholds
- **Large File**: Files >100MB flagged for review
- **Old File**: Files >365 days flagged for archival
- **User Reporting**: Only users with >1GB total usage receive reports
- **Critical Space**: <10% free space requires immediate action

---

## 4. Procedure: Running Disk Monitoring System

### 4.1 Testing and Dry Runs

#### Step 1: Test on Single User
Before running full system scans, test on a single user account:

```bash
# SSH to server
ssh admin@lab417.saiab.ac.za

# Navigate to sysadmin directory
cd /home/evilliers/work/sysadmin

# Run dry-run test on single user (no emails sent)
sudo ./bin/disk_monitor.sh --dry-run --user evilliers --verbose
```

#### Step 2: Review Test Output
Check the output for:
- Files correctly identified (FASTQ, SAM, large files, old files)
- Accurate size calculations
- Proper space savings estimates
- Report formatting
- No errors or warnings

#### Step 3: Test Full System (Dry Run)
Once single-user test succeeds:

```bash
# Test full scan without NFS (faster for testing)
sudo ./bin/disk_monitor.sh --dry-run --skip-nfs --verbose
```

#### Step 4: Verify Report Generation
Check that reports are created but not sent:

```bash
# List generated reports
ls -lt reports/admin_summary_*.txt | head -1

# Review latest admin summary
cat reports/admin_summary_*.txt | less
```

### 4.2 Manual Production Execution

#### Step 1: Full System Scan with Notifications
When ready to run in production:

```bash
# Full scan including NFS, with email notifications
sudo ./bin/disk_monitor.sh
```

**Expected Duration**: 30-120 minutes depending on:
- Number of files to scan
- NFS mount responsiveness
- System load
- Number of users

#### Step 2: Monitor Progress
Open a separate terminal and watch the logs:

```bash
# Watch main execution log
tail -f logs/disk_monitor_*.log

# Watch notification log
tail -f logs/notifications_$(date +%Y%m%d).log
```

#### Step 3: Verify Completion
After execution completes, verify:

```bash
# Check for successful completion in log
grep "COMPLETED" logs/disk_monitor_*.log | tail -1

# Verify admin summary was created
ls -lt reports/admin_summary_*.txt | head -1

# Check notification delivery status
grep "SUCCESS\|FAILED" logs/notifications_*.log | tail -20
```

#### Step 4: Review Admin Summary
```bash
# Read the admin summary
cat reports/admin_summary_*.txt
```

Review:
- Filesystem usage percentages
- Top 10 users by disk consumption
- Total potential space savings
- Any failed notifications
- Performance metrics

### 4.3 Targeted User Scans

For investigating specific user issues:

```bash
# Scan single user with full reporting
sudo ./bin/disk_monitor.sh --user username

# Check user's report
sudo cat /home/username/DISK_USAGE_REPORT.txt

# Verify email was sent
grep "username" logs/notifications_$(date +%Y%m%d).log
```

### 4.4 Quick Scans (Skip NFS)

For faster execution during testing or urgent checks:

```bash
# Skip NFS scanning
sudo ./bin/disk_monitor.sh --skip-nfs
```

**Use when**:
- NFS mount is slow or unresponsive
- Need quick local filesystem check
- Testing configuration changes
- Troubleshooting issues

---

## 5. Procedure: Responding to Low Disk Space Alerts

### 5.1 Immediate Assessment

#### Step 1: Check Current Disk Usage
```bash
# Check all filesystem usage
df -h

# Check specific filesystems
df -h /home /data /mnt/agrp/lab417

# Check inode usage (can fill up independently)
df -i
```

#### Step 2: Identify Severity Level

| Free Space | Severity | Action Required |
|-----------|----------|-----------------|
| >20% | Normal | Routine monitoring |
| 10-20% | Warning | Schedule cleanup within 1 week |
| 5-10% | Critical | Immediate action required |
| <5% | Emergency | Follow emergency procedures |

#### Step 3: Identify Top Consumers
```bash
# Find largest directories in /home
sudo du -h --max-depth=1 /home | sort -hr | head -20

# Find largest directories in /data
sudo du -h --max-depth=1 /data | sort -hr | head -20

# Find largest files across filesystem
sudo find /home -type f -size +1G -exec ls -lh {} \; | sort -k5 -hr | head -20
```

### 5.2 Run Targeted Analysis

#### Step 1: Run Immediate Disk Scan
```bash
# Run disk monitor for quick assessment
sudo ./bin/disk_monitor.sh --skip-nfs
```

#### Step 2: Review Admin Summary
```bash
# Read latest admin summary
cat reports/admin_summary_*.txt

# Focus on:
# - Top users by consumption
# - Total potential savings
# - Quick win opportunities (uncompressed FASTQ)
```

### 5.3 Contact High-Usage Users

#### Step 1: Prioritize User Contacts
Based on admin summary, identify users with:
1. Largest disk usage
2. Most uncompressed FASTQ files
3. SAM files that should be BAM
4. Large old files

#### Step 2: Send Personalized Communication
```bash
# Check if user already received automated report
grep "username" logs/notifications_*.log

# If not recent, run targeted scan
sudo ./bin/disk_monitor.sh --user username
```

Email template:
```
Subject: Urgent: Storage Optimization Required on HPC Server

Dear [username],

Our HPC server is currently experiencing limited disk space. Your account has been
identified as a significant storage consumer.

Current usage: [X] GB

We have identified opportunities to reduce your storage footprint:
- [X] GB in uncompressed FASTQ files
- [X] GB in SAM files that should be BAM format
- [X] GB in files older than 1 year

Please review the detailed report in your home directory:
  ~/DISK_USAGE_REPORT.txt

Priority actions (within 48 hours):
1. Compress all uncompressed FASTQ files
2. Convert SAM files to BAM format
3. Archive or delete old files

Need assistance? Contact: evilliers@saiab.ac.za

Thank you for your prompt attention to this matter.
```

### 5.4 Monitor Improvement

#### Step 1: Track Disk Space Changes
```bash
# Record current usage
df -h > /tmp/disk_before.txt

# After users complete cleanup (wait 24-48 hours)
df -h > /tmp/disk_after.txt

# Compare
diff /tmp/disk_before.txt /tmp/disk_after.txt
```

#### Step 2: Re-scan to Verify
```bash
# Run follow-up scan
sudo ./bin/disk_monitor.sh

# Compare new admin summary to previous
diff reports/admin_summary_[previous].txt reports/admin_summary_[latest].txt
```

---

## 6. Procedure: Assisting Users with Storage Optimization

### 6.1 Compress FASTQ Files

Users should be guided through this process:

#### Interactive Compression
```bash
# Navigate to directory with FASTQ files
cd /path/to/fastq/files

# Compress single file
gzip filename.fastq
# Result: filename.fastq.gz (70-80% space savings)

# Compress all FASTQ files in directory
gzip *.fastq
gzip *.fq

# Verify compression worked
ls -lh *.gz
```

#### Batch Compression with SLURM
For large numbers of files:

```bash
# Create compression script
cat > compress_fastq.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=compress_fastq
#SBATCH --output=compress_%j.log
#SBATCH --partition=agrp
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G

# Compress FASTQ files with parallel gzip
find . -name "*.fastq" -o -name "*.fq" | \
  parallel -j 4 'gzip {}'
EOF

# Submit job
sbatch compress_fastq.sh
```

#### Safety Checks
```bash
# Test on one file first
gzip -t filename.fastq.gz  # Verify integrity

# If verification succeeds, original can be deleted
# (gzip automatically removes original after compression)
```

### 6.2 Convert SAM to BAM

#### Single File Conversion
```bash
# Install samtools (if not already in conda environment)
conda activate bioinformatics
conda install samtools

# Convert SAM to sorted BAM
samtools view -bS input.sam | samtools sort -o output.bam

# Index BAM file
samtools index output.bam

# Verify BAM file integrity
samtools quickcheck output.bam
echo $?  # Should return 0 if OK

# If successful, remove SAM file
rm input.sam
```

#### Batch Conversion with SLURM
```bash
# Create conversion script
cat > sam_to_bam.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=sam2bam
#SBATCH --output=sam2bam_%j.log
#SBATCH --partition=agrp
#SBATCH --cpus-per-task=8
#SBATCH --mem=16G

# Load conda environment
source ~/miniconda3/etc/profile.d/conda.sh
conda activate bioinformatics

# Convert all SAM files to BAM
for sam in *.sam; do
  bam="${sam%.sam}.bam"
  echo "Converting $sam to $bam"

  samtools view -@ 4 -bS "$sam" | samtools sort -@ 4 -o "$bam"
  samtools index "$bam"

  # Verify and remove SAM if successful
  if samtools quickcheck "$bam"; then
    echo "Success: $bam verified"
    rm "$sam"
  else
    echo "ERROR: $bam verification failed, keeping $sam"
  fi
done
EOF

# Submit job
sbatch sam_to_bam.sh
```

### 6.3 Archive Old Files

#### Identify Old Files
```bash
# Find files older than 1 year
find /home/username -type f -mtime +365 -size +100M

# Count old files and total size
find /home/username -type f -mtime +365 -exec du -ch {} + | tail -1
```

#### Create Archive
```bash
# Create dated archive
tar czf archive_$(date +%Y%m%d).tar.gz /path/to/old/files/

# Verify archive integrity
tar tzf archive_$(date +%Y%m%d).tar.gz > /dev/null
echo $?  # Should return 0 if OK

# Move to archive location
mv archive_*.tar.gz /mnt/agrp/lab417/archives/username/

# Only remove originals after verifying archive
# rm -rf /path/to/old/files/
```

#### Best Practices for Archival
1. **Always verify archives before deleting originals**
2. **Use dated filenames** for easy identification
3. **Document archive contents** with README files
4. **Store archives on long-term storage** (NFS, not /home)
5. **Test extraction** periodically to ensure archives remain readable

---

## 7. Scheduled Maintenance

### 7.1 Weekly Automated Scan

#### Verify Cron Job is Active
```bash
# Check if cron job is installed
sudo ls -l /etc/cron.d/disk-monitor

# View cron schedule
sudo cat /etc/cron.d/disk-monitor
```

Expected content:
```
# Disk monitoring system - runs every Sunday at 2:00 AM
0 2 * * 0 root /home/evilliers/work/sysadmin/bin/disk_monitor.sh >> /home/evilliers/work/sysadmin/logs/cron.log 2>&1
```

#### Review Weekly Execution
Every Monday morning:

```bash
# Check Sunday's cron execution
tail -100 logs/cron.log

# Review latest admin summary
cat reports/admin_summary_*.txt | less

# Check for any notification failures
grep "FAILED" logs/notifications_*.log | tail -20
```

### 7.2 Monthly Review

Perform these checks on the first Monday of each month:

#### Step 1: Trend Analysis
```bash
# Compare disk usage over time
df -h > /tmp/disk_usage_$(date +%Y%m).txt

# Compare to previous month
diff /tmp/disk_usage_[prev_month].txt /tmp/disk_usage_$(date +%Y%m).txt
```

#### Step 2: Review Top Users
```bash
# Check if same users appear in top 10 consistently
grep "TOP 10 USERS" reports/admin_summary_*.txt | tail -5
```

#### Step 3: Effectiveness Metrics
- Total space reclaimed this month
- Number of users who compressed FASTQ files
- Number of SAM files converted to BAM
- Reduction in old file count

#### Step 4: Configuration Review
```bash
# Review configuration settings
cat config/disk_monitor.conf | grep -E "THRESHOLD|DAYS|MIN_"
```

Consider adjusting if:
- Too many false positives in reports
- Missing significant storage issues
- Performance issues with scanning

### 7.3 Quarterly Maintenance

Every 3 months:

#### Clean Up Old Logs
```bash
# Logs are auto-cleaned after 90 days, verify:
find logs/ -name "*.log" -mtime +90 -ls

# Manually clean if needed
find logs/ -name "*.log" -mtime +90 -delete

# Clean old reports
find reports/ -name "admin_summary_*.txt" -mtime +90 -delete
```

#### Audit User Accounts
```bash
# Find users who haven't logged in recently
sudo lastlog | grep "Never logged in\|200 days"

# Check disk usage of inactive accounts
for user in $(sudo lastlog | grep "Never" | awk '{print $1}'); do
  echo -n "$user: "
  sudo du -sh /home/$user
done
```

#### Review System Capacity
- Assess if additional storage is needed
- Plan for capacity expansion if consistently >80% usage
- Consider implementing quotas if needed

---

## 8. Troubleshooting

### 8.1 Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **Disk monitor not running** | No recent logs or reports | Check cron job, verify symlink, review permissions |
| **Email notifications failing** | Users not receiving reports | Test mail server, check SMTP config, verify email addresses |
| **NFS scanning hangs** | Script runs >3 hours | Use `--skip-nfs`, check NFS mount health, adjust timeout |
| **Lock file error** | "Another instance running" | Verify no process running, remove stale lock file |
| **Permission denied errors** | Scan fails for some users | Ensure running as root, check directory permissions |
| **Reports show 0 bytes savings** | No optimization opportunities found | May be false negative, manually verify with `find` |

### 8.2 Detailed Troubleshooting Steps

#### Issue: No Email Notifications Received

**Step 1: Verify email is enabled**
```bash
grep ENABLE_EMAIL config/disk_monitor.conf
# Should show: ENABLE_EMAIL=true
```

**Step 2: Test mail system**
```bash
echo "Test email" | mail -s "Test from HPC" evilliers@saiab.ac.za

# Check mail logs
sudo tail -f /var/log/mail.log
```

**Step 3: Check notification logs**
```bash
grep "FAILED\|ERROR" logs/notifications_*.log
```

**Step 4: Verify email addresses**
```bash
# Check if users have valid email format
cat config/disk_monitor.conf | grep EMAIL_DOMAIN
```

**Note**: Users always receive file notifications in home directory even if email fails.

#### Issue: Disk Monitor Runs Too Slowly

**Step 1: Check current performance**
```bash
# Review last execution time
grep "Duration:" reports/admin_summary_*.txt | tail -1
```

**Step 2: Disable NFS if slow**
```bash
# Temporary: use --skip-nfs flag
sudo ./bin/disk_monitor.sh --skip-nfs

# Permanent: edit config
vim config/disk_monitor.conf
# Set: ENABLE_NFS_SCAN=false
```

**Step 3: Reduce parallel scans**
```bash
vim config/disk_monitor.conf
# Reduce: MAX_PARALLEL_SCANS=2  (default: 4)
```

**Step 4: Limit scan depth**
```bash
vim config/disk_monitor.conf
# Reduce: MAX_FIND_DEPTH=10  (default: 20)
```

**Step 5: Add exclusion patterns**
```bash
vim config/disk_monitor.conf
# Add paths to EXCLUDE_PATTERNS
# Example: EXCLUDE_PATTERNS="*.tmp *.bak .cache* .singularity*"
```

#### Issue: Lock File Prevents Execution

**Step 1: Check if process actually running**
```bash
ps aux | grep disk_monitor.sh
```

**Step 2: If no process found, remove stale lock**
```bash
sudo rm /var/run/disk_monitor.lock
```

**Step 3: Re-run script**
```bash
sudo ./bin/disk_monitor.sh
```

#### Issue: Permission Denied Errors

**Step 1: Verify running as root**
```bash
whoami  # Should show: root
sudo ./bin/disk_monitor.sh  # Use sudo
```

**Step 2: Check log file permissions**
```bash
ls -la logs/
# Should be writable by root/admin
sudo chown -R evilliers:evilliers logs/
```

**Step 3: Verify scan path permissions**
```bash
ls -ld /home /data /mnt/agrp/lab417
```

### 8.3 Emergency Recovery

#### Disk Full Scenario

If filesystem reaches 100% capacity:

**Step 1: Immediate space recovery**
```bash
# Clear system temporary files
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*

# Clear old log files
sudo find /var/log -name "*.gz" -delete
sudo find /var/log -name "*.old" -delete

# Truncate large log files
sudo truncate -s 0 /var/log/syslog
```

**Step 2: Identify largest consumers**
```bash
# Find massive files quickly
sudo find /home -xdev -type f -size +10G -exec ls -lh {} \;

# Check for core dumps
sudo find /home -name "core.*" -exec rm {} \;
```

**Step 3: Force user cleanup**
```bash
# Temporarily move largest files to archive
sudo mkdir -p /mnt/agrp/lab417/emergency_archive/$(date +%Y%m%d)
sudo mv [large_files] /mnt/agrp/lab417/emergency_archive/$(date +%Y%m%d)/

# Document what was moved
echo "Moved files from emergency cleanup" > /mnt/agrp/lab417/emergency_archive/$(date +%Y%m%d)/README.txt
```

**Step 4: Notify affected users**
Send immediate email explaining emergency action and where files were moved.

---

## 9. Best Practices

### 9.1 Proactive Storage Management

**1. Regular Monitoring**
- Review weekly admin summaries every Monday
- Track disk usage trends monthly
- Set up alerts for >85% utilization

**2. User Education**
- Include storage best practices in onboarding
- Reference training materials in automated reports
- Provide examples of good file organization

**3. Preventive Actions**
- Encourage compression of FASTQ files immediately after generation
- Recommend BAM format over SAM from the start
- Promote archival of completed project data

### 9.2 Bioinformatics File Management

**File Format Best Practices**

| File Type | Recommended Storage | Space Savings | Notes |
|-----------|-------------------|---------------|-------|
| FASTQ | Always compress (.gz) | 70-80% | Use `pigz` for faster compression |
| SAM | Convert to BAM immediately | 50-80% | Smaller and faster to process |
| BAM | Already compressed, keep | N/A | Always index (.bai) |
| VCF | Compress to VCF.gz | 60-75% | Use bgzip for compatibility |
| BED | Keep small files uncompressed | Variable | Compress large files |
| FASTA | Compress unless frequently accessed | 60-70% | Reference genomes can stay uncompressed |

**Directory Organization**
```
~/project_name/
├── raw_data/           # Original FASTQ (compressed)
├── alignments/         # BAM files (sorted and indexed)
├── variants/           # VCF files
├── scripts/            # Analysis scripts
├── results/            # Final outputs
└── README.md          # Project documentation
```

### 9.3 Archive Strategy

**When to Archive**
- Project completed and published
- Data not accessed in >1 year
- Reference data superseded by newer version
- Intermediate files after final results generated

**Where to Archive**
1. **Short-term**: `/mnt/agrp/lab417/archives/` (1-2 years)
2. **Long-term**: External backup systems or cold storage
3. **Published data**: Deposit to public repositories (SRA, ENA, etc.)

**Archive Naming Convention**
```
YYYY-MM-DD_ProjectName_Description.tar.gz

Examples:
2026-02-10_SeabreamGenome_RawFASTQ.tar.gz
2026-02-10_SeabreamGenome_Alignments.tar.gz
2026-02-10_SeabreamGenome_FinalResults.tar.gz
```

### 9.4 Quota Considerations

**If implementing quotas** (future enhancement):

Recommended quotas by user type:
- **Students/Course participants**: 50GB soft, 75GB hard
- **Researchers**: 200GB soft, 300GB hard
- **Project groups**: 1TB soft, 1.5TB hard

Grace period: 7 days for soft quota

### 9.5 Data Retention Policy

Establish clear policies:

| Data Type | Retention | Action After Retention |
|-----------|-----------|----------------------|
| Raw sequencing data | 3 years | Archive or delete if published |
| Intermediate files | 1 year | Delete after results verified |
| Final results | Permanent | Keep in archived projects |
| Temporary analysis | 30 days | Auto-delete or user confirms |
| Published data | Permanent | Move to public repositories |

---

## Appendix A: Disk Monitoring System Details

### A.1 System Architecture

```
/home/evilliers/work/sysadmin/
├── bin/
│   └── disk_monitor.sh          # Main orchestrator script
├── lib/
│   ├── file_scanner.sh           # Filesystem scanning functions
│   ├── report_generator.sh       # Report generation functions
│   └── notification_handler.sh   # Email and file notification functions
├── config/
│   ├── disk_monitor.conf         # Central configuration file
│   └── disk-monitor.cron         # Cron schedule
├── logs/
│   ├── disk_monitor_*.log        # Execution logs
│   ├── notifications_*.log       # Email delivery logs
│   └── cron.log                  # Cron execution logs
├── reports/
│   └── admin_summary_*.txt       # Admin summary archives
└── templates/
    └── (Future: email templates)
```

### A.2 Configuration Parameters

Key settings in `config/disk_monitor.conf`:

**Scan Locations**
```bash
SCAN_PATHS=(
    "/home"
    "/data"
    "/mnt/agrp/lab417"
)
```

**Thresholds**
```bash
LARGE_FILE_THRESHOLD="100M"       # Flag files larger than this
OLD_FILE_AGE_DAYS=365             # Flag files older than this
MIN_DISK_USAGE="1G"               # Minimum usage to generate report
```

**Performance**
```bash
NICE_LEVEL=19                     # Process priority (19 = lowest)
IONICE_CLASS=3                    # I/O priority (3 = idle)
MAX_PARALLEL_SCANS=4              # Concurrent user scans
```

**Email Settings**
```bash
ENABLE_EMAIL=true
ADMIN_EMAIL="evilliers@saiab.ac.za"
EMAIL_DOMAIN="saiab.ac.za"
```

### A.3 Report Outputs

**User Reports** (`~/DISK_USAGE_REPORT.txt`):
- Current disk usage by filesystem
- List of uncompressed FASTQ files with compression commands
- List of SAM files with BAM conversion commands
- Large files (>100MB)
- Old files (>1 year)
- Total potential space savings
- Prioritized action recommendations

**Admin Summary** (`reports/admin_summary_YYYYMMDD_HHMMSS.txt`):
- System-wide filesystem status with usage graphs
- Top 10 users by disk consumption
- System-wide statistics (total files, potential savings)
- Notification delivery status
- Performance metrics

### A.4 Execution Logs

**Main Log** (`logs/disk_monitor_YYYYMMDD_HHMMSS.log`):
- Script execution start/end times
- Users scanned
- Errors encountered
- Summary statistics

**Notification Log** (`logs/notifications_YYYYMMDD.log`):
- Email delivery attempts
- Success/failure status
- Error messages

**Cron Log** (`logs/cron.log`):
- Automated execution records
- Output from cron-triggered scans

---

## Appendix B: Common Storage Optimization Commands

### B.1 Finding Files

**Find large files**
```bash
# Files larger than 1GB
find /home/username -type f -size +1G -exec ls -lh {} \;

# Top 20 largest files
find /home/username -type f -exec ls -s {} \; | sort -n -r | head -20
```

**Find old files**
```bash
# Files not accessed in >1 year
find /home/username -type f -atime +365

# Files not modified in >1 year
find /home/username -type f -mtime +365
```

**Find specific file types**
```bash
# All FASTQ files (uncompressed)
find /home/username -name "*.fastq" -o -name "*.fq"

# All SAM files
find /home/username -name "*.sam"

# Count and size
find /home/username -name "*.fastq" -exec du -ch {} + | tail -1
```

### B.2 Compression Commands

**gzip (single-threaded)**
```bash
# Compress single file
gzip file.fastq

# Compress with best compression (slower)
gzip -9 file.fastq

# Keep original file
gzip -k file.fastq

# Decompress
gunzip file.fastq.gz
```

**pigz (parallel gzip, faster)**
```bash
# Install if not available
conda install pigz

# Compress with 4 threads
pigz -p 4 file.fastq

# Compress all FASTQ files in parallel
find . -name "*.fastq" | parallel -j 4 pigz
```

**Test compressed file integrity**
```bash
# Test gzip integrity
gzip -t file.fastq.gz

# Will exit with error if corrupt
echo $?  # 0 = OK, non-zero = error
```

### B.3 SAM/BAM Operations

**Convert SAM to BAM**
```bash
# Basic conversion
samtools view -bS input.sam > output.bam

# Convert and sort in one step
samtools view -bS input.sam | samtools sort -o output.sorted.bam

# With multiple threads (faster)
samtools view -@ 4 -bS input.sam | samtools sort -@ 4 -o output.sorted.bam
```

**Index BAM files**
```bash
# Create BAI index
samtools index input.bam

# Results in input.bam.bai
```

**Verify BAM integrity**
```bash
samtools quickcheck input.bam
echo $?  # 0 = OK, 1 = corrupt
```

**Get BAM statistics**
```bash
samtools flagstat input.bam
samtools stats input.bam
```

### B.4 Archive Creation

**Create compressed archive**
```bash
# Create tar.gz archive
tar czf archive.tar.gz /path/to/files/

# Create with progress display
tar czf archive.tar.gz /path/to/files/ --verbose

# Exclude certain patterns
tar czf archive.tar.gz --exclude="*.tmp" --exclude="*.log" /path/to/files/
```

**Verify archive integrity**
```bash
# List contents without extracting
tar tzf archive.tar.gz

# Verify integrity
tar tzf archive.tar.gz > /dev/null
echo $?  # 0 = OK
```

**Extract archive**
```bash
# Extract all
tar xzf archive.tar.gz

# Extract to specific directory
tar xzf archive.tar.gz -C /destination/path/

# Extract single file
tar xzf archive.tar.gz path/to/specific/file
```

### B.5 Disk Usage Analysis

**Directory sizes**
```bash
# Current directory subdirectories
du -h --max-depth=1 . | sort -hr

# Specific user's home
sudo du -h --max-depth=1 /home/username | sort -hr

# All users in /home
sudo du -sh /home/* | sort -hr
```

**Find disk usage by file type**
```bash
# All FASTQ files
find /home/username -name "*.fastq*" -exec du -ch {} + | tail -1

# All BAM files
find /home/username -name "*.bam" -exec du -ch {} + | tail -1
```

**Filesystem usage**
```bash
# Human-readable format
df -h

# Inode usage
df -i

# Specific filesystem
df -h /home
```

---

## Appendix C: Emergency Procedures

### C.1 Critical Disk Space (<5% Free)

**Immediate Actions (within 30 minutes)**

1. **Alert team**
   ```bash
   echo "URGENT: Disk space critical on lab417" | \
     mail -s "URGENT: Disk Space Critical" evilliers@saiab.ac.za
   ```

2. **Prevent new writes** (if absolutely necessary)
   ```bash
   # Make filesystem read-only (EXTREME - only if disk full)
   # This will prevent all new writes
   # sudo mount -o remount,ro /home

   # Better: set quota for all users temporarily
   # Requires quota support
   ```

3. **Clear immediate space**
   ```bash
   # Remove temp files
   sudo rm -rf /tmp/*

   # Clear old logs
   sudo find /var/log -name "*.gz" -delete
   sudo find /var/log -name "*.old" -delete

   # Check for core dumps
   sudo find /home -name "core.*" -size +100M -delete
   ```

4. **Identify top consumers**
   ```bash
   # Fastest way to find space hogs
   sudo du -sh /home/* /data/* | sort -hr | head -10
   ```

5. **Contact top users immediately**
   - Phone call or instant message
   - Request immediate cleanup
   - Provide specific file locations to remove

**Short-term Actions (within 24 hours)**

1. **Run disk monitor**
   ```bash
   sudo ./bin/disk_monitor.sh --skip-nfs
   ```

2. **Force cleanup of top offenders**
   - Work with users to compress FASTQ
   - Convert SAM to BAM
   - Move old files to archive

3. **Temporary space expansion**
   - Mount additional volumes if available
   - Move data to NFS temporarily

### C.2 NFS Mount Failure

**Symptoms**
- Disk monitor hangs
- Users can't access `/mnt/agrp/lab417`
- Long delays on filesystem access

**Diagnosis**
```bash
# Check mount status
mount | grep agrp

# Check NFS server connectivity
ping [NFS_SERVER_IP]

# Try to access mount
ls -la /mnt/agrp/lab417

# Check NFS services
sudo systemctl status nfs-client.target
```

**Resolution**

1. **Remount NFS**
   ```bash
   sudo umount -f /mnt/agrp/lab417
   sudo mount -a
   ```

2. **If unmount fails**
   ```bash
   # Force unmount
   sudo umount -l /mnt/agrp/lab417
   sudo mount -a
   ```

3. **Skip NFS in monitoring**
   ```bash
   # Temporarily disable NFS scanning
   sudo ./bin/disk_monitor.sh --skip-nfs
   ```

4. **Contact NFS administrator**
   - Report issue with NFS server
   - Provide error messages
   - Check /var/log/syslog for NFS errors

### C.3 Disk Monitor Failure

**Symptoms**
- No reports generated
- Script exits with errors
- No recent logs

**Diagnosis**
```bash
# Check last execution
ls -lt logs/disk_monitor_*.log | head -1

# Review error messages
tail -100 logs/disk_monitor_*.log | grep -i error

# Check for lock file
ls -la /var/run/disk_monitor.lock

# Verify script exists and is executable
ls -la bin/disk_monitor.sh
```

**Resolution**

1. **Remove stale locks**
   ```bash
   sudo rm /var/run/disk_monitor.lock
   ```

2. **Test script manually**
   ```bash
   sudo ./bin/disk_monitor.sh --dry-run --user evilliers --verbose
   ```

3. **Check permissions**
   ```bash
   sudo chown -R evilliers:evilliers /home/evilliers/work/sysadmin/
   chmod +x bin/disk_monitor.sh
   ```

4. **Review configuration**
   ```bash
   cat config/disk_monitor.conf | less
   # Look for syntax errors
   ```

5. **Manual scan alternative**
   ```bash
   # Generate basic disk usage report
   df -h > /tmp/disk_report.txt
   sudo du -sh /home/* >> /tmp/disk_report.txt
   mail -s "Manual Disk Report" evilliers@saiab.ac.za < /tmp/disk_report.txt
   ```

---

## Appendix D: Report Interpretation Guide

### D.1 Understanding User Reports

**Sample User Report Structure**
```
==========================================
DISK USAGE REPORT for username
Generated: 2026-02-10 03:15:22
==========================================

CURRENT DISK USAGE:
  /home/username:        45.2 GB
  /data:                 102.8 GB
  /mnt/agrp/lab417:      356.4 GB
  TOTAL:                 504.4 GB

OPTIMIZATION OPPORTUNITIES:

1. UNCOMPRESSED FASTQ FILES (Priority: HIGH)
   Found: 45 files
   Current size: 180.5 GB
   After compression: ~45.1 GB (estimated)
   Potential savings: ~135.4 GB

   Example files:
   /home/username/project1/sample1_R1.fastq (12.3 GB)
   /home/username/project1/sample1_R2.fastq (12.1 GB)
   ...

   Recommended command:
   gzip *.fastq

2. SAM FILES (Priority: MEDIUM)
   Found: 12 files
   Current size: 89.2 GB
   After BAM conversion: ~26.8 GB (estimated)
   Potential savings: ~62.4 GB

   Example files:
   /data/alignments/sample1.sam (15.4 GB)
   ...

   Recommended command:
   samtools view -bS input.sam | samtools sort -o output.bam

3. OLD FILES (Priority: LOW)
   Found: 234 files older than 1 year
   Total size: 67.3 GB

TOTAL POTENTIAL SAVINGS: ~265.1 GB (53% reduction)

ACTION ITEMS:
1. [HIGH] Compress 45 FASTQ files (saves ~135 GB)
2. [MEDIUM] Convert 12 SAM files to BAM (saves ~62 GB)
3. [LOW] Review 234 old files for archival (saves ~67 GB)
```

**Interpretation Guidelines**

- **Priority HIGH**: Action will save significant space (>100GB or >50% of user total)
- **Priority MEDIUM**: Moderate savings (10-100GB or 10-50% of user total)
- **Priority LOW**: Good practice but smaller impact (<10GB or <10% of user total)

### D.2 Understanding Admin Summary

**Sample Admin Summary Sections**

**1. Filesystem Status**
```
FILESYSTEM USAGE:
  /home         : [##########          ] 86% (1.2 TB / 1.4 TB)
  /data         : [############        ] 94% (4.7 TB / 5.0 TB)
  /mnt/agrp/lab417: [########            ] 67% (6.7 TB / 10.0 TB)
```

**Status Indicators**:
- `<70%`: Normal (keep monitoring)
- `70-85%`: Attention needed (plan cleanup)
- `85-95%`: Warning (initiate user contact)
- `>95%`: Critical (immediate action required)

**2. Top Users**
```
TOP 10 USERS BY DISK CONSUMPTION:
1. researcher1    : 1,234 GB  [****************************]
2. researcher2    :   892 GB  [********************        ]
3. student1       :   567 GB  [*************               ]
...
```

**What to do**:
- Users at top of list are priority contacts
- Check if their usage is justified (active projects)
- Verify they received automated reports
- Contact directly if usage seems excessive

**3. System-wide Statistics**
```
SYSTEM-WIDE STATISTICS:
  Total files scanned    : 1,234,567
  Total users scanned    : 38
  Users reported         : 28
  Uncompressed FASTQ     : 632 GB (potential savings)
  SAM files             : 164 GB (potential savings)
  Total potential savings: 796 GB

NOTIFICATION DELIVERY:
  Email sent            : 26
  Email failed          : 2
  File notifications    : 28
  Success rate          : 93%
```

**Key metrics**:
- **Success rate <90%**: Investigate email delivery issues
- **Potential savings >500GB**: High-priority cleanup campaign needed
- **Users reported vs scanned**: Shows how many have significant usage

**4. Performance Metrics**
```
PERFORMANCE:
  Scan duration         : 87 minutes
  Average per user      : 2.3 minutes
  Files processed/sec   : 237
```

**Benchmarks**:
- Scan duration should be <120 minutes
- If >180 minutes, consider optimization (skip NFS, reduce depth)

### D.3 Trend Analysis

**Week-over-Week Comparison**

Compare multiple admin summaries:
```bash
# Extract key metrics from last 4 weeks
for f in reports/admin_summary_*.txt; do
  echo "=== $f ==="
  grep "TOTAL:" $f
  grep "potential savings" $f
done
```

**Red flags**:
- Disk usage increasing >5% per week
- Same users in top 10 week after week
- Potential savings increasing (users ignoring reports)
- Email failure rate increasing

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | SAIAB HPC Admin | Initial release |

---

## Document Approval

**Prepared by:** SAIAB HPC Administration Team
**Contact:** evilliers@saiab.ac.za
**Server:** lab417.saiab.ac.za

---

*End of Standard Operating Procedure*
