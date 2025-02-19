# sed-Logs

## How to Use `sed` to Extract Errors and Insert Them into MySQL

### Issue
During the workday, some database tables were locked and later unlocked as per the socket. Our customers needed to export daily data, but during the issue, we found that some tables did not record the required data.

After deep investigation into the logs, I decided to delete the incorrect data from MySQL and reconstruct it using the latest event logs from the CTI system. Since log analysis is complex, I used the following strategy:

1. Extract multiple samples to identify the relevant search patterns (table names and errors).
2. Process logs efficiently to extract the required data.

---

### Steps to Extract and Process Logs

#### 1. Move the logs for the day
```bash
cp /var/log/Path/to-your/cti_2025-02-18-*.log /home/islam/ctilog/
```

#### 2. Extract agent IDs associated with `isInCall[false]`
```bash
grep "isInCall\[false\]" cti_2025-02-18-*.log | while read -r line; do
    AGENT_ID=$(echo "$line" | sed -n 's/.*Agent\[\([0-9]\+\)\].*/\1/p');
    if [[ -n "$AGENT_ID" ]]; then
        grep "INSERT INTO DATAMART_AGENT_DETAILS" cti_2025-02-18-*.log | grep "'$AGENT_ID'"
    fi
done > /home/islam/result1.log
```

#### 3. Extract the `INSERT` statements from the logs
```bash
sed -n "s/.*\(INSERT INTO DATAMART_AGENT_DETAILS.*\)/\1/p" /home/islam/result1.log > /home/islam/result2.log
```

#### 4. Clean up the extracted data
```bash
sed -i 's/\]$//' /home/islam/result2.log
sed -i 's/\]\.$//' /home/islam/result2.log
sed -i 's/$/;/' /home/islam/result2.log
```

---

### Expected Output
After running these steps, the processed data will be formatted correctly, ready for MySQL insertion:
```sql
INSERT INTO DATAMART_AGENT_DETAILS(EVENT_TIME,AGENT_ID,GRP_DBID,EVENT,EVENTDETAILS,END_CALL_REASON,TRACKNUM,SESSION_ID,SKILL_LIST,QUEUE_LIST,TEAM_DBID)
VALUES ('2025.02.18 10:26:29','9571', '174', 'WAITING_FOR_NEXT_CALL', '', '', '', '2357897',',',',','');
```

---

### Last Step
Once inside MySQL, execute the following command:
```bash
mysql -u name -p -A
SOURCE /home/islam/result2.log;
```

---

### Notes
- Ensure you have proper permissions to access logs and modify database records.
- Always backup the database before making bulk insertions.
- Modify the script as needed to fit your database schema and log format.

- Here Last update SCRIPT
- #!/bin/bash

LOG_DIR="/var/log/PATH-LOGSHERE"
DEST_DIR="/home/islam/ctilog"
RESULT1="/home/islam/result1.log"
RESULT2="/home/islam/result2.log"
DB_USER="YOURDBUSER"
DB_PASS="YOUDBPASS"
DB_NAME="YOURSCHEMA NAME"

read -p "Enter the date (YYYY-MM-DD): " LOG_DATE

cp $LOG_DIR/cti_${LOG_DATE}-*.log $DEST_DIR/
echo "Log files copied to $DEST_DIR."

for LOG_FILE in $DEST_DIR/cti_${LOG_DATE}-*.log; do
    echo "Processing $LOG_FILE..."
    read -p "Continue processing this file? (y/n): " CONFIRM
    if [[ "$CONFIRM" != "y" ]]; then
        continue
    fi
    
    grep "isInCall\[false\]" "$LOG_FILE" | while read -r line; do
        AGENT_ID=$(echo "$line" | sed -n 's/.*Agent\[\([0-9]\+\)\].*/\1/p');
        if [[ -n "$AGENT_ID" ]]; then
            grep "INSERT INTO DATAMART_AGENT_DETAILS" "$LOG_FILE" | grep "'$AGENT_ID'"
        fi
    done > "$RESULT1"
    
    echo "Extracted relevant log data to $RESULT1."
    cat "$RESULT1"
    read -p "Proceed with extraction to SQL format? (y/n): " CONFIRM
    if [[ "$CONFIRM" != "y" ]]; then
        continue
    fi
    
    sed -n "s/.*\(INSERT INTO DATAMART_AGENT_DETAILS.*\)/\1/p" "$RESULT1" > "$RESULT2"
    
    echo "Extracted SQL insert statements to $RESULT2."
    cat "$RESULT2"
    read -p "Proceed with SQL cleaning? (y/n): " CONFIRM
    if [[ "$CONFIRM" != "y" ]]; then
        continue
    fi
    
    sed -i 's/\]$//' "$RESULT2"
    sed -i 's/\]\.$//' "$RESULT2"
    sed -i 's/$/;/' "$RESULT2"
    
    echo "Cleaned SQL statements in $RESULT2:"
    cat "$RESULT2"
    read -p "Proceed with database insertion? (y/n): " CONFIRM
    if [[ "$CONFIRM" != "y" ]]; then
        continue
    fi

    mysql -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" -A -e "SOURCE $RESULT2;"
    echo "Database updated successfully."
done

echo "All logs processed."

# Bash Script 

## Description
This script automates the process of extracting relevant data from log files and inserting it into a MySQL database. It performs the following steps:

1. Copies log files from a specified directory to a destination folder.
2. Extracts specific log entries related to agent call statuses.
3. Formats the extracted data into SQL `INSERT` statements.
4. Cleans and structures the SQL statements properly.
5. Inserts the cleaned SQL data into a MySQL database.

## Prerequisites
- Linux environment with Bash support.
- MySQL installed and accessible.
- Necessary permissions to access log files and execute MySQL commands.

## Installation
1. Clone the repository:
   ```sh
   git clone <repository_url>
   cd <repository_directory>
   ```
2. Ensure the script is executable:
   ```sh
   chmod +x script.sh
   ```

## Usage
1. Run the script:
   ```sh
   ./script.sh
   ```
2. Enter the required date when prompted (format: `YYYY-MM-DD`).
3. Follow the interactive prompts to process logs and update the database.

## Configuration
Update the script variables as needed:
- `LOG_DIR`: Path to the log files.
- `DEST_DIR`: Destination directory for copied logs.
- `DB_USER`: MySQL database user.
- `DB_PASS`: MySQL database password.
- `DB_NAME`: MySQL database schema name.

## Example
```sh
Enter the date (YYYY-MM-DD): 2025-02-18
Log files copied to /home/islam/ctilog.
Processing /home/islam/ctilog/cti_2025-02-18-001.log...
Extracted SQL insert statements to /home/islam/result2.log.
Database updated successfully.
```

## Notes
- Ensure MySQL credentials are correctly set.
- Run the script with appropriate permissions.
- Modify `LOG_DIR` and `DEST_DIR` paths based on your environment.






