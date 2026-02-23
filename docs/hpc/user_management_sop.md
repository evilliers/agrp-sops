# Standard Operating Procedure: User Account Management

**SAIAB HPC Server**

| Document Information ||
|----------------------|----------------|
| **Version** | 1.0 |
| **Date** | 2026-02-10 |
| **Author** | SAIAB HPC Administration |
| **Contact** | evilliers@saiab.ac.za |

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [Prerequisites](#2-prerequisites)
3. [Procedure: Creating Individual Users](#3-procedure-creating-individual-users)
4. [Procedure: Creating Course Participant Accounts (Batch)](#4-procedure-creating-course-participant-accounts-batch)
5. [Verification and Testing](#5-verification-and-testing)
6. [Troubleshooting](#6-troubleshooting)
7. [Security Considerations](#7-security-considerations)
8. [Appendix A: Script Documentation](#appendix-a-script-documentation)
9. [Appendix B: Email Template](#appendix-b-email-template)
10. [Appendix C: Batch File Format](#appendix-c-batch-file-format)
11. [Appendix D: .bashrc Configuration Details](#appendix-d-bashrc-configuration-details)

---

## 1. Purpose and Scope

### 1.1 Purpose
This Standard Operating Procedure (SOP) provides step-by-step instructions for creating and configuring user accounts on the SAIAB HPC server. It ensures consistent account setup with proper SLURM and conda environment configurations.

### 1.2 Scope
This SOP covers:
- Individual user account creation (on-demand)
- Batch user account creation for course participants
- Account configuration with SLURM aliases and conda initialization
- Credential generation and distribution
- New user onboarding communications

### 1.3 Intended Audience
- System administrators with sudo/root privileges
- HPC support staff
- Course organizers

### 1.4 Conda Environment Architecture
This user creation process implements a **user-specific conda installation model**:

- **Each user installs conda in their own home directory** (typically `~/miniconda3`)
- **No system-wide conda installation** is used or configured
- Users have complete control over their conda environments and packages
- This approach provides:
  - **Isolation**: Package conflicts between users are impossible
  - **Independence**: Users can install/update packages without affecting others
  - **Flexibility**: Each user can choose different conda versions or distributions
  - **No permission issues**: Users own their conda installation completely

The .bashrc configuration automatically detects conda installations in standard home directory locations (`~/miniconda3`, `~/anaconda3`, or `~/.conda`).

---

## 2. Prerequisites

### 2.1 Required Privileges
- Root or sudo access to the HPC server
- SSH access to lab417.saiab.ac.za

### 2.2 Required Information

**For Individual Users:**
- Username (lowercase, alphanumeric with hyphens/underscores)
- Email address

**For Course Participants:**
- Text file containing usernames and email addresses (CSV format)

### 2.3 Software Requirements
- Bash shell
- Standard Linux user management tools (useradd, chpasswd, chage)
- User creation script: `/home/evilliers/work/sysadmin/create_users/create_new_user.sh`

---

## 3. Procedure: Creating Individual Users

### 3.1 Overview
Use this procedure when creating accounts for individual users on-demand (e.g., new staff, researchers, or collaborators).

### 3.2 Step-by-Step Instructions

#### Step 1: Access the Server
```bash
ssh your_admin_account@lab417.saiab.ac.za
```

#### Step 2: Navigate to Script Directory
```bash
cd /home/evilliers/work/sysadmin/create_users
```

#### Step 3: Run the User Creation Script
```bash
sudo ./create_new_user.sh
```

#### Step 4: Select Interactive Mode
When the menu appears, select option **1** (Create single user).

```
========================================
HPC USER CREATION TOOL
========================================

1) Create single user (interactive)
2) Create multiple users from file (batch)
3) Exit

Select option [1-3]: 1
```

#### Step 5: Provide User Information
- Enter the username when prompted
  - Must start with a lowercase letter
  - Can contain lowercase letters, numbers, hyphens, underscores
  - Maximum 32 characters

- Enter the email address when prompted
  - Must be a valid email format

```
Enter username: jsmith
Enter email address: jsmith@saiab.ac.za
```

#### Step 6: Confirm Creation
Review the displayed information and confirm by typing `y`:

```
Creating user with the following details:
  Username: jsmith
  Email:    jsmith@saiab.ac.za

Proceed? (y/n): y
```

#### Step 7: Review Output
The script will:
- Create the user account
- Generate a secure random password
- Configure .bashrc with SLURM and conda settings
- Save credentials to a dated file
- Generate an email template

#### Step 8: Send Welcome Email
Locate the generated email file:
```bash
ls -lt email_*.txt | head -1
```

Open the email template and send it to the user:
```bash
cat email_jsmith_YYYYMMDD_HHMMSS.txt
```

Copy the email content and send it to the user via your email client.

#### Step 9: Secure Credential Storage
After sending the email:
```bash
# View credentials file
cat credentials_YYYYMMDD_HHMMSS.txt

# Securely delete after confirming email was sent
shred -u credentials_YYYYMMDD_HHMMSS.txt
```

### 3.3 Expected Results
- User account created with home directory at `/home/username`
- User assigned to their own primary group
- .bashrc configured with SLURM aliases and conda initialization for **user-specific conda installations**
- Temporary password set (must be changed on first login)
- Welcome email generated with all necessary information, including conda installation instructions

**Note:** Each user will have their own isolated conda installation in their home directory (e.g., `~/miniconda3`). This ensures package management independence and prevents conflicts between users.

---

## 4. Procedure: Creating Course Participant Accounts (Batch)

### 4.1 Overview
Use this procedure when creating multiple accounts for course participants or workshops.

### 4.2 Step-by-Step Instructions

#### Step 1: Prepare User List File
Create a CSV file with usernames and email addresses. See [Appendix C](#appendix-c-batch-file-format) for format details.

Example (`bioinformatics_course_2026.txt`):
```
# Bioinformatics Course - February 2026
# Format: username,email
student1,student1@university.ac.za
student2,student2@university.ac.za
student3,student3@university.ac.za
```

#### Step 2: Upload File to Server
If the file is on your local machine, upload it:
```bash
scp bioinformatics_course_2026.txt admin@lab417.saiab.ac.za:~/
```

#### Step 3: Run the User Creation Script
```bash
cd /home/evilliers/work/sysadmin/create_users
sudo ./create_new_user.sh
```

#### Step 4: Select Batch Mode
Select option **2** (Create multiple users from file).

```
Select option [1-3]: 2
```

#### Step 5: Provide File Path
Enter the full path to your user list file:
```
Enter path to user list file: /home/admin/bioinformatics_course_2026.txt
```

#### Step 6: Review and Confirm
The script will count the users and ask for confirmation:
```
Found 3 users in file

Proceed with creating 3 users? (y/n): y
```

#### Step 7: Monitor Progress
The script will process each user and display status:
```
Processing user: student1
  ✓ User 'student1' created successfully
  ✓ Password set
  ✓ Password change required on first login
  ✓ Home directory: /home/student1
  ✓ Configured .bashrc with SLURM and conda settings
  ✓ User 'student1' setup complete
```

#### Step 8: Review Summary
After completion, review the summary:
```
========================================
Batch creation complete!
Created: 3 users
Skipped: 0 users
========================================
```

#### Step 9: Distribute Credentials
Individual email files are created for each user:
```bash
# List all email files
ls -lt email_*.txt

# Send emails to participants
# Option 1: Copy each email manually
cat email_student1_*.txt
# Copy and send via email client

# Option 2: Use automated email script (if available)
```

#### Step 10: Secure Cleanup
After distributing all credentials:
```bash
# Securely delete credentials file
shred -u credentials_YYYYMMDD_HHMMSS.txt

# Optionally archive email templates
mkdir -p ~/course_archives/bioinformatics_2026/
mv email_*.txt ~/course_archives/bioinformatics_2026/
chmod 700 ~/course_archives/bioinformatics_2026/
```

---

## 5. Verification and Testing

### 5.1 Verify User Creation
```bash
# Check user exists
id username

# Verify home directory
ls -la /home/username

# Check password expiration (should show password change required)
sudo chage -l username | grep "Last password change"
```

### 5.2 Test User Login
```bash
# Test SSH login
ssh username@lab417.saiab.ac.za

# On first login, user will be prompted to change password
```

### 5.3 Verify .bashrc Configuration
After logging in as the user:
```bash
# Test aliases
ll
sq
si

# Check conda initialization (after user installs conda in their home directory)
# Users should install conda at: ~/miniconda3 or ~/anaconda3
conda --version

# Verify prompt shows conda environment
conda activate base
# Prompt should show: (base) username@hostname:~$

# Test SLURM interactive login
slogin
# Prompt should show: [SLURM:JOBID] username@hostname:~$
```

**Note:** Conda will not be available until the user installs it in their home directory following the instructions in the welcome email. The .bashrc is pre-configured to detect conda installations at:
- `~/miniconda3/etc/profile.d/conda.sh`
- `~/anaconda3/etc/profile.d/conda.sh`
- `~/.conda/etc/profile.d/conda.sh`

---

## 6. Troubleshooting

### 6.1 Common Issues and Solutions

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| "Permission denied" when running script | Not running as root/sudo | Use `sudo ./create_new_user.sh` |
| "User already exists" | Username conflict | Choose a different username or check if user was previously created |
| "Invalid username" | Username doesn't meet requirements | Use lowercase letters, numbers, hyphens, underscores only; must start with letter |
| "Invalid email address" | Email format incorrect | Verify email contains @ and valid domain |
| User cannot log in | Network/firewall issue | Check SSH connectivity; verify user created with `id username` |
| Aliases not working | .bashrc not sourced | Have user log out and log back in, or run `source ~/.bashrc` |
| Conda not found | Conda not installed in user's home | User needs to install miniconda/anaconda in their **home directory** following the welcome email instructions. Should be at `~/miniconda3` or `~/anaconda3` |
| SLURM commands fail | SLURM not configured | Contact HPC administrator; verify SLURM is running |

### 6.2 Script Errors

**Error: "File not found"**
- Verify the batch file path is correct
- Check file permissions are readable

**Error: "Cannot create home directory"**
- Check disk space: `df -h /home`
- Verify /home partition is writable

**Error: "Password change failed"**
- Check chage command availability
- Verify PAM configuration

### 6.3 Log File Review
Check the activity log for detailed error messages:
```bash
tail -f /home/evilliers/work/sysadmin/create_users/user_creation.log
```

---

## 7. Security Considerations

### 7.1 Password Security
- Passwords are randomly generated with 12 characters including uppercase, lowercase, numbers, and special characters
- Users are forced to change password on first login
- Temporary passwords should never be reused
- Credentials files are created with 600 permissions (owner read/write only)

### 7.2 Credential Distribution
- **DO NOT** email passwords in plain text
- Use secure methods:
  - In-person delivery for sensitive accounts
  - Encrypted email (GPG/PGP)
  - Secure messaging platforms
  - Two-channel delivery (username via email, password via SMS/phone)

### 7.3 File Cleanup
- Always securely delete credential files after distribution
- Use `shred -u` instead of `rm` to prevent recovery
- Archive email templates in a secure location with restricted permissions

### 7.4 Audit Trail
- All user creation activities are logged with timestamps
- Review logs periodically: `/home/evilliers/work/sysadmin/create_users/user_creation.log`
- Maintain records of when users were created and by whom

### 7.5 Account Review
- Periodically review active accounts
- Disable or remove accounts for departed users
- Check for unused accounts:
```bash
sudo lastlog | grep "Never logged in"
```

---

## Appendix A: Script Documentation

### Location
```
/home/evilliers/work/sysadmin/create_users/create_new_user.sh
```

### Features
- **Interactive Mode**: Create single users with prompts
- **Batch Mode**: Create multiple users from CSV file
- **Validation**: Username and email validation
- **Password Generation**: Secure 12-character random passwords
- **Configuration**: Automatic .bashrc setup with SLURM and conda settings
- **Logging**: Complete activity logging with timestamps
- **Email Templates**: Automatic generation of welcome emails

### Script Components

#### User Creation
```bash
useradd -m -s /bin/bash -U username
```
- `-m`: Create home directory
- `-s /bin/bash`: Set bash as default shell
- `-U`: Create user's own primary group

#### Password Management
```bash
echo "username:password" | chpasswd
chage -d 0 username
```
- Sets initial password
- Forces password change on first login

#### .bashrc Configuration
Automatically adds:
- SLURM aliases (sq, si, slogin, slogin-x11)
- Helpful aliases (ll, cls, gs)
- Conda initialization
- Custom prompt function

### File Permissions
```bash
chmod 750 create_new_user.sh  # Owner and group can execute
chmod 600 credentials_*.txt    # Only owner can read/write
chmod 600 email_*.txt          # Only owner can read/write
```

---

## Appendix B: Email Template

Below is the email template automatically generated by the script. Customize as needed for your organization.

---

**Subject:** Welcome to SAIAB HPC Server - Your Account Details

Dear [username],

Welcome to the SAIAB HPC (High Performance Computing) server! Your user account has been created and is ready to use.

**LOGIN CREDENTIALS**

- **Username:** [username]
- **Temporary Password:** [auto-generated]
- **Server:** lab417.saiab.ac.za

**IMPORTANT:** You will be required to change your password on first login.

**HOW TO CONNECT**

From a terminal or SSH client, use the following command:

```
ssh [username]@lab417.saiab.ac.za
```

On first login, you will be prompted to change your password. Choose a strong password that:
- Is at least 8 characters long
- Contains uppercase and lowercase letters
- Contains numbers
- Contains special characters

**YOUR ENVIRONMENT**

Your account has been pre-configured with helpful tools and shortcuts:

**SLURM Aliases (for job management):**
- `sq` - View the SLURM queue (see running jobs)
- `si` - View SLURM node information
- `slogin` - Start an interactive session on a compute node
- `slogin-x11` - Start an interactive session with X11 forwarding (for GUI apps)

**General Aliases:**
- `ll` - Detailed file listing (ls -la)
- `cls` - Clear screen
- `gs` - Git status

Your shell prompt will automatically show:
- Active conda environment name (when activated)
- SLURM job ID (when running in a SLURM allocation)

**GETTING STARTED**

1. Connect to the server using SSH
2. Change your password when prompted
3. Install Miniconda in your home directory (detailed instructions provided in email)
4. Complete the online training courses (see below)

**NOTE:** The actual email sent to users includes detailed step-by-step instructions for installing Miniconda in their home directory at `~/miniconda3`. This ensures each user has an isolated conda environment.

**TRAINING COURSES**

We strongly recommend completing these courses to get the most out of the HPC system:

**Shell for Bioinformatics:**
https://lab417.saiab.ac.za/unix_course/Shell-for-bioinformatics/index.html

**Conda Package Management:**
https://lab417.saiab.ac.za/conda_course/conda_website.html

**BASIC WORKFLOW**

1. Log in to the server
2. Use SLURM to request computational resources
3. Load your conda environments for software
4. Run your analyses
5. Transfer results back to your local machine

**Example SLURM interactive session:**
```
slogin
# This starts an interactive bash session on a compute node
```

**Example batch job submission:**
```
sbatch my_script.sh
```

**NEED HELP?**

If you have questions or need assistance, please contact:

**HPC Administrator:** evilliers@saiab.ac.za

We also recommend reading the SLURM documentation and conda documentation available through the training courses above.

---

Welcome aboard, and happy computing!

SAIAB HPC Administration Team

---

## Appendix C: Batch File Format

### File Format Specification

**Format:** CSV (Comma-Separated Values)

**Structure:**
```
username,email
```

### Requirements
- One user per line
- Fields separated by commas
- No spaces around commas (will be trimmed automatically)
- Comments start with `#`
- Blank lines are ignored

### Example File

```
# Bioinformatics Course - Spring 2026
# Instructor: Dr. Smith
# Format: username,email

# Group 1 - Genomics
student1,student1@university.ac.za
jdoe,john.doe@institution.org
mwilliams,mary.williams@research.edu

# Group 2 - Transcriptomics
student2,student2@university.ac.za
asmith,alice.smith@lab.com

# Group 3 - Proteomics
rbrown,robert.brown@institute.ac.za
student3,student3@university.ac.za
```

### Creating a Batch File

**Using a text editor:**
```bash
nano course_participants.txt
```

**From a spreadsheet:**
1. Create columns: username, email
2. Export as CSV
3. Upload to server

**Validation before use:**
```bash
# Check file format
cat course_participants.txt

# Count users (excluding comments)
grep -v '^#' course_participants.txt | grep -v '^[[:space:]]*$' | wc -l
```

### Best Practices

1. **Naming Convention:** Use descriptive filenames
   - `bioinformatics_course_feb2026.txt`
   - `workshop_participants_2026.txt`

2. **Documentation:** Include comments with:
   - Course/workshop name
   - Date
   - Instructor/organizer
   - File format reminder

3. **Validation:** Review file before running script
   - Check for typos in usernames
   - Verify email addresses are correct
   - Ensure no duplicate usernames

4. **Backup:** Keep a copy of the participant list
   ```bash
   cp course_participants.txt ~/course_archives/
   ```

---

## Appendix D: .bashrc Configuration Details

### Overview
The user creation script automatically configures each user's `.bashrc` file with optimizations for SLURM and conda usage.

### Configuration Components

#### 1. Helpful Aliases
```bash
alias ll='ls -la'
alias cls='clear'
alias gs='git status'
```

**Purpose:** Common convenience aliases for file listing, screen clearing, and git operations.

#### 2. SLURM Aliases

```bash
# View queue with formatted output
alias sq='squeue -o "%8i%12j%4t%10u%20q%20a%20P%5D%R"'

# View node information
alias si='sinfo -o "%20P%8D%16F%8z%10m%N"'

# Interactive login to compute node
alias slogin='srun -p agrp --cpus-per-task=1 --nodes=1 --mem=4G --pty bash -i'

# Interactive login with X11 forwarding
alias slogin-x11='srun -p agrp --cpus-per-task=1 --nodes=1 --mem=4G --pty --x11 bash -i'
```

**Purpose:**
- `sq` - Quick view of job queue with relevant columns
- `si` - View available compute resources
- `slogin` - Fast access to compute nodes for interactive work
- `slogin-x11` - For running GUI applications remotely

**Default partition:** `agrp` (adjust if your setup uses different partitions)

#### 3. Conda Initialization

```bash
# Initialize conda - check for user-specific installations in home directory
# NOTE: Each user should install conda in their own home directory
# This ensures isolation and allows users to manage their own environments
if [ -f "$HOME/miniconda3/etc/profile.d/conda.sh" ]; then
    source "$HOME/miniconda3/etc/profile.d/conda.sh"
elif [ -f "$HOME/anaconda3/etc/profile.d/conda.sh" ]; then
    source "$HOME/anaconda3/etc/profile.d/conda.sh"
elif [ -f "$HOME/.conda/etc/profile.d/conda.sh" ]; then
    source "$HOME/.conda/etc/profile.d/conda.sh"
fi
```

**Purpose:** Automatically initialize conda if installed in the user's home directory, checking standard installation locations.

**Important:** This configuration supports **user-specific conda installations only**. Each user should install conda in their own home directory (e.g., `~/miniconda3`). This approach:
- Ensures package isolation between users
- Prevents permission conflicts
- Allows users to manage their own environments independently
- Avoids dependency conflicts between different users' projects

#### 4. Custom Prompt Function

```bash
set_custom_prompt() {
    local conda_env=""
    local slurm_info=""

    # Show active conda environment
    if [[ -n "$CONDA_DEFAULT_ENV" ]]; then
        conda_env="(\e[1;32m$CONDA_DEFAULT_ENV\e[0m) "
    fi

    # Show SLURM job ID if in allocation
    if [[ -n "$SLURM_JOB_ID" ]]; then
        slurm_info="[SLURM:$SLURM_JOB_ID] "
    fi

    export PS1="${slurm_info}${conda_env}\u@\h:\w\$ "
}
PROMPT_COMMAND=set_custom_prompt
```

**Purpose:** Dynamic prompt that shows:
- Active conda environment in green (e.g., `(base)` or `(myenv)`)
- SLURM job ID when running in an allocation (e.g., `[SLURM:12345]`)

**Example prompts:**
```bash
# Default (no conda, no SLURM)
user@lab417:~$

# With conda environment activated
(base) user@lab417:~$

# In SLURM job allocation
[SLURM:45678] user@lab417:~$

# Both conda and SLURM
[SLURM:45678] (myenv) user@lab417:~$
```

### Manual Configuration

If you need to manually add these configurations to an existing user:

```bash
# Append to user's .bashrc
sudo cat >> /home/username/.bashrc << 'EOF'
# HPC Server Custom Configuration
[paste configuration here]
EOF

# Apply changes
sudo chown username:username /home/username/.bashrc
```

### Customization Options

Administrators can customize the script's .bashrc template by editing:
```
/home/evilliers/work/sysadmin/create_users/create_new_user.sh
```

Look for the `configure_bashrc()` function around line 100.

**Common customizations:**
- Change SLURM partition from `agrp` to your default partition
- Add organization-specific aliases
- Modify prompt colors or format
- Add module loading commands
- Include custom environment variables

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-10 | SAIAB HPC Admin | Initial release |
| 1.1 | 2026-02-10 | SAIAB HPC Admin | Enhanced documentation to clarify user-specific conda installation model; added detailed conda installation instructions in welcome email; updated .bashrc comments to emphasize home directory conda installations |

---

## Document Approval

**Prepared by:** SAIAB HPC Administration Team
**Contact:** evilliers@saiab.ac.za
**Server:** lab417.saiab.ac.za

---

*End of Standard Operating Procedure*
