# Tableau Full Backup to S3

This script automates the process of creating a full backup of your Tableau Server, uploading it to an AWS S3 bucket for secure storage, and optionally sending monitoring data to a Zabbix server.

## Features

- **Automated Backups**: Runs the `tsm maintenance backup` command to create a full Tableau Server backup.
- **S3 Upload**: Securely uploads the generated backup file (`.tsbak`) to a specified AWS S3 bucket.
- **Data Integrity**: Calculates the MD5 checksum of the backup file and uploads it as a separate `.md5sum.txt` file for verification.
- **Cleanup**: Automatically removes the local backup file after a successful upload to save disk space.
- **Monitoring**: Integrates with Zabbix to send metrics and status updates for backup duration, file size, and success/failure.
- **Flexible Configuration**: All settings are managed through a simple `config.ini` file.
- **Timestamping**: Option to automatically append a timestamp to backup filenames to avoid overwriting previous backups.
- **Logging**: Provides detailed logging to both the console and a rotating log file.

## Prerequisites

- Python 3
- Tableau Server with access to the `tsm` command-line utility.
- An AWS account and an S3 bucket.
- AWS credentials configured for `boto3`. You can configure them using environment variables, a credentials file, or an IAM role. See the [boto3 documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) for more details.
- (Optional) A Zabbix server and the `zabbix_sender` command-line utility installed on the Tableau Server for monitoring.

## Installation

1. **Install the required Python library:**
    ```bash
    pip install -r requirements.txt
    ```

## Configuration

1.  **Create the config file:**
    Copy the example configuration file:
    ```bash
    cp config.ini-example config.ini
    ```

2.  **Edit the configuration:**
    Open `config.ini` and customize the settings for your environment.

    ```ini
    [Logging]
    # https://docs.python.org/3/library/logging.handlers.html#logging.handlers.RotatingFileHandler
    level = info
    debug = False
    filename = full-backup2s3.log
    max_bytes = 52428800
    backup_count = 6

    [Backup]
    # The base name for the backup file.
    backup_file = ts_backup
    # If True, a timestamp (YYYYMMDD-HHMMSS) will be appended to the filename.
    append_timestamp = True
    # If True, uses the --multithreaded option for the tsm backup command.
    multithreaded = True
    # The directory where Tableau Server stores its backups.
    backup_dir = /var/opt/tableau/tableau_server/data/tabsvc/files/backups

    [AWS]
    # The name of your S3 bucket.
    bucket_name = your-s3-bucket-name

    [Zabbix]
    # The path to your Zabbix agent configuration file.
    # This is only needed if you want to send monitoring data to Zabbix.
    config_file = /etc/zabbix/zabbix_agentd.conf
    ```

## Usage

The script has two main commands:

- `backup`: Creates a new Tableau Server backup and then uploads all `.tsbak` files found in the `backup_dir` to S3.
- `upload`: Skips the backup creation and only uploads existing `.tsbak` files from the `backup_dir` to S3.

**Examples:**

- **To run a full backup and upload to S3:**
  ```bash
  python3 full_backup2s3.py backup
  ```

- **To upload existing backups only:**
  ```bash
  python3 full_backup2s3.py upload
  ```

- **For more detailed logs, use the `-d` flag:**
  ```bash
  python3 full_backup2s3.py backup -d
  ```

## S3 Bucket Policy

Your S3 bucket needs a policy that allows the script to upload objects. Here is an example of a minimal IAM role policy that grants `s3:PutObject` permission. 

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "S3Access",
			"Effect": "Allow",
			"Action": "s3:PutObject",
			"Resource": [
				"arn:aws:s3:::__BACKET_NAME__/*"
			]
		}
	]
}
```

## Troubleshooting

Here are some common outputs from the `tsm` command that can help you diagnose issues:

- **Success:**
  You will see progress percentages and the final message will be `Backup written to ...`.

- **Error: Cannot overwrite existing file:**
  ```
  (1,
  "\nAn error occurred on the server generating the backup.\n\nSee '/var/opt/tableau/tableau_server/data/tabsvc/logs/tabadmincontroller/tabadmincontroller_*.log' on Tableau Server nodes running the Administration Controller process for server log information.\n\nResource Conflict: Cannot overwrite the existing file at '/var/opt/tableau/tableau_server/data/tabsvc/files/backups/ts_backup-2024-04-08.tsbak'\n",
  '')
  ```
  **Solution:** Set `append_timestamp = True` in your `config.ini` to create uniquely named backups.

- **Error: Could not connect to server:**
  ```
  (1,
  'Could not connect to server. Make sure that Tableau Server is running and try again.\n',
  '')
  ```
  **Solution:** Ensure that the Tableau Server is running and that the user executing the script has the necessary permissions.

## How It Works

1.  **Initialization**: The script starts, reads the `config.ini` file, and sets up logging.
2.  **Backup (if commanded)**:
    - It constructs and executes a `tsm maintenance backup` command with the options specified in the config file.
    - It monitors the exit code and output of the `tsm` command to determine success or failure.
3.  **Upload**:
    - The script scans the `backup_dir` for any files ending in `.tsbak`.
    - For each backup file found, it does the following:
        - Starts uploading the file to the specified S3 bucket.
        - Calculates the MD5 checksum of the file.
        - After the upload is complete, it uploads the checksum in a separate file (`<backup_filename>.md5sum.txt`).
        - If the upload was successful, it deletes the local `.tsbak` file.
4.  **Monitoring**:
    - If Zabbix is configured, the script sends a heartbeat and various metrics throughout the process:
        - `full-backup2s3.heartbeat`: A signal that the script is running.
        - `full-backup2s3.tsm.backup_duration`: The time taken for the `tsm` backup to complete.
        - `full-backup2s3.tsm.backup_result_code`: The result code of the backup (0 for success, 1 for failure).
        - `full-backup2s3.backup_file_size`: The size of the uploaded backup file in bytes.
        - `full-backup2s3.upload_duration`: The time taken to upload the file to S3.
        - `full-backup2s3.upload_result_code`: The result code of the upload (0 for success, 1 for failure).
